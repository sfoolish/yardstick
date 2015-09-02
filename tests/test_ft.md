## Rally test

    vim tests/README.rst 
    apt-get install -y python-pip
    apt-get install -y libpq-dev python-dev
	apt-get install -y libxml2-dev libxslt1-dev
	pip install tox
    tox -e cover

---

Error: pg_config executable not found.

sudo apt-get install -y libpq-dev python-dev


/tmp/pip-build-ljQ3Fu/lxml/src/lxml/includes/etree_defs.h:14:31: fatal error: libxml/xmlversion.h: No such file or directory

apt-get install -y libxml2-dev libxslt1-dev python-dev



---

Functional tests
----------------

*Files: /tests/functional/**

The goal of `functional tests <https://en.wikipedia.org/wiki/Functional_testing>`_ is to check that everything works well together.
Fuctional tests use Rally API only and check responses without touching internal parts.

To run functional tests locally::

  $ source openrc
  $ rally deployment create --fromenv --name testing
  $ tox -e cli

  #NOTE: openrc file with OpenStack admin credentials

Output of every Rally execution will be collected under some reports root in
directory structure like: reports_root/ClassName/MethodName_suffix.extension
This functionality implemented in tests.functional.utils.Rally.__call__ method.
Use 'gen_report_path' method of 'Rally' class to get automatically generated file
path and name if you need. You can use it to publish html reports, generated
during tests.
Reports root can be passed through environment variable 'REPORTS_ROOT'. Default is
'rally-cli-output-files'.


rally ft 能正常测试，但有用例跑不通过，找不到镜像， glance 有问题，后续在定位

将 /rally/tests/functional/test_xx.py删除只留 test_cli_info.py 用于测试

apt-get install testrepository

pip install testrepository

https://www.howtoinstall.co/en/ubuntu/trusty/main/testrepository/
https://testrepository.readthedocs.org/en/latest/

