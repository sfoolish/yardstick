root@23c8a9d2e66b:~/yardstick# git diff c4c0e13e93dfa0aeef0a51dd0f4b00610baac592
diff --git a/yardstick/benchmark/runners/base.py b/yardstick/benchmark/runners/base.py
@@ -14,6 +14,8 @@ import multiprocessing
 import subprocess
 import time
+from yardstick.dispatcher.base import Base as DispatcherBase
 log = logging.getLogger(__name__)

 import yardstick.common.utils as utils
@@ -25,6 +27,9 @@ def _output_serializer_main(filename, queue):
     Use of this process enables multiple instances of a scenario without
     messing up the output file.
     '''
+    config = {}
+    config["type"] = "Http"
+    dispatcher = DispatcherBase.get(config)
     with open(filename, 'a+') as outfile:
         while True:
             # blocks until data becomes available
@@ -35,6 +40,7 @@ def _output_serializer_main(filename, queue):
             else:
                 json.dump(record, outfile)
                 outfile.write('\n')
+                dispatcher.record_metering_data({"title": record})
 def _single_action(seconds, command, queue):
diff --git a/yardstick/dispatcher/__init__.py b/yardstick/dispatcher/__init__.py
--- /dev/null
+++ b/yardstick/dispatcher/__init__.py
@@ -0,0 +1,3 @@
+import yardstick.common.utils as utils
+
+utils.import_modules_from_package("yardstick.dispatcher")
diff --git a/yardstick/dispatcher/base.py b/yardstick/dispatcher/base.py
+++ b/yardstick/dispatcher/base.py
@@ -0,0 +1,49 @@
+import abc
+import logging
+import six
+import yardstick.common.utils as utils
+
+LOG = logging.getLogger(__name__)
+
+@six.add_metaclass(abc.ABCMeta)
+class Base(object):
+
+    def __init__(self, conf):
+        self.conf = conf
+
+    @staticmethod
+    def get_cls(dispatcher_type):
+        '''Return class of specified type'''
+        for dispatcher in utils.itersubclasses(Base):
+            if dispatcher_type == dispatcher.__dispatcher_type__:
+                return dispatcher
+        raise RuntimeError("No such dispatcher_type %s" % dispatcher_type)
+
+    @staticmethod
+    def get(config):
+        """Returns instance of a dispatcher for dispatcher type.
+        """
+        return Base.get_cls(config["type"])(config)
+
+    @abc.abstractmethod
+    def record_metering_data(self, data):
+        """Recording metering data interface."""
+        pass
diff --git a/yardstick/dispatcher/file.py b/yardstick/dispatcher/file.py
+++ b/yardstick/dispatcher/file.py
@@ -0,0 +1,73 @@
+
+import logging
+import logging.handlers
+
+from yardstick.dispatcher.base import Base as DispatchBase
+
+
+class FileDispatcher(DispatchBase):
+    """Dispatcher class for recording metering data to a file.
+
+    The dispatcher class which logs each meter into a file configured in
+    ceilometer configuration file. An example configuration may look like the
+    following:
+
+    [dispatcher_file]
+    file_path = /tmp/meters
+
+    To enable this dispatcher, the following section needs to be present in
+    ceilometer.conf file
+
+    [DEFAULT]
+    dispatcher = file
+    """
+
+    __dispatcher_type__ = "File"
+
+    # Name and the location of the file to record result, currently just hard coded.
+    file_path = "/tmp/meters"
+    # The max size of the file, currently just hard coded.
+    max_bytes = 0
+    # The max number of the files to keep, current just hard codded.
+    backup_count = 0
+
+    def __init__(self, conf):
+        super(FileDispatcher, self).__init__(conf)
+        self.log = None
+
+        # if the directory and path are configured, then log to the file
+        if self.file_path:
+            dispatcher_logger = logging.Logger('dispatcher.file')
+            dispatcher_logger.setLevel(logging.INFO)
+            # create rotating file handler which logs meters
+            rfh = logging.handlers.RotatingFileHandler(
+                self.file_path,
+                maxBytes=self.max_bytes,
+                backupCount=self.backup_count,
+                encoding='utf8')
+
+            rfh.setLevel(logging.INFO)
+            # Only wanted the meters to be saved in the file, not the
+            # project root logger.
+            dispatcher_logger.propagate = False
+            dispatcher_logger.addHandler(rfh)
+            self.log = dispatcher_logger
+
+    def record_metering_data(self, data):
+        if self.log:
+            self.log.info(data)
+
diff --git a/yardstick/dispatcher/http.py b/yardstick/dispatcher/http.py
+++ b/yardstick/dispatcher/http.py
@@ -0,0 +1,90 @@
+
+import json
+import logging
+
+import requests
+
+from yardstick.dispatcher.base import Base as DispatchBase
+# from ceilometer.i18n import _, _LE
+# from ceilometer.publisher import utils as publisher_utils
+
+LOG = logging.getLogger(__name__)
+
+class HttpDispatcher(DispatchBase):
+    """Dispatcher class for posting metering data into a http target.
+
+    To enable this dispatcher, the following option needs to be present in
+    ceilometer.conf file::
+
+        [DEFAULT]
+        dispatcher = http
+
+    Dispatcher specific options can be added as follows::
+
+        [dispatcher_http]
+        target = www.example.com
+        timeout = 2
+    """
+
+    __dispatcher_type__ = "Http"
+
+    # The target where the http request will be sent.
+    # If this is not set, no data will be posted.
+    # For example: target = http://hostname:1234/path
+    target = "http://10.143.37.214:32770/todo/api/v1.0/tasks"
+    # The max time in seconds to wait for a request to timeout.
+    timeout = 5
+
+    def __init__(self, conf):
+        super(HttpDispatcher, self).__init__(conf)
+        self.headers = {'Content-type': 'application/json'}
+        self.timeout = self.timeout
+        self.target = self.target
+
+    def record_metering_data(self, data):
+        if self.target == '':
+            # if the target was not set, do not do anything
+            LOG.error(('Dispatcher target was not set, no meter will '
+                        'be posted. Set the target in the ceilometer.conf '
+                        'file'))
+            return
+
+        # We may have receive only one counter on the wire
+        if not isinstance(data, list):
+            data = [data]
+
+        for meter in data:
+            # LOG.debug(())
+            try:
+                res = requests.post(self.target,
+                                    data=json.dumps(meter),
+                                    headers=self.headers,
+                                    timeout=self.timeout)
+                LOG.debug(('Message posting finished with status code '
+                            '%d.') % res.status_code)
+            except Exception as err:
+                LOG.exception(('Failed to record metering data: %s'),
+                              err)