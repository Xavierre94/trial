##############################
Guidelines for test developers
##############################

There are several workflows for developing xPAD platform and component
tests that should be executed in the CI system. This section describes such workflows.

*******************
Developer workflows
*******************

Simple shell scripts executed on target
=======================================

To integrate existing test cases / test suites into the xPAD test
execution environment, a small helper class is provided. Test cases and
test suites should be installed during the build process according to
the `Test suite conventions`_ below. This currently only supports the system
tests type.

Test suite conventions
----------------------

A test suite consists of one folder containing a set of
binaries/scripts.

-  setup\_suite: is executed once before the test suite is started
-  setup: is executed before every test case is started
-  teardown\_suite: is executed once after the complete test suite is
   finished
-  teardown: is executed once after each test case
-  N test case binaries / scripts

All binaries need to have an exit status of 0 on successful execution.
All other exit statuses are treated as failure. If the execution of
setup\_suite indicates failure, the execution of the test suite is
skipped and a failure is recorded. If teardown\_suite indicates failure,
only the failure is recorded and the results of the test cases remain
unaffected. If the setup or teardown of a test case fails, the test case
is marked as failed.

Google Test suite conventions
-----------------------------

Google Tests based test suites can conform to the conventions mentioned above by
wrapping the test suite with a Python module. In case of *system tests*, instead
of plain binaries, it is also possible to use the test suite support along with
Google Test binaries.

In addition, the Google Tests suite folder can also provide the following
*optional* text files

- An `ENVIRONMENT-<gtest_bin>` file which specifies the environment for the test
  case named `<gtest_bin>`
- A file named `ENVIRONMENT`, which contains the global environment for the tests

These files would specify the environment using the format::

    ENV_VAR1=foo
    ENV_VAR2=bar

There exists a helper `create_gtest_support_script` bbclass which automates the
creation of the wrapper module and so helps ease the packaging and deployment of
Google Test based test suites. This is described below.

You may however also choose to customize the packaging and deployment of your
Google Test based test suite as documented under `Integration tests created with Google Test`_


File locations and test suite name
----------------------------------

On the target device the test suites need to be installed to:

::

    ${SYSTEM_TESTS_TARGET_FILES_PATH}/<testsuite>
               # (currently set to: ${libdir}/${PN}/tests/<testsuite>/)

Each package needs at least one test suite called BAT (Build Acceptance
Tests). This test suite will be executed on all CI instances. All other
test suites are only executed on the bci.Team for the package.

In the package recipe the packager needs to ensure that after the
do\_install step the needed test suite files are in the correct target
location. Additionally, the package must inherit the
create\_test\_support\_script bbclass or the create\_gtest_support_script
bbclass depending on the type of the test suite.

:Example:

::

    # Load automatic test suite generation support
    inherit create_test_support_script

    # Provide one test suite named "BAT"
    do_install_append() {
        # Create the test suite folder
        install -d "${D}${SYSTEM_TESTS_TARGET_FILES_PATH}/BAT"
        # Install the test scripts into the folder
        tar --no-same-owner --exclude-vcs -cpf - -C "${S}/BAT" . \
            | tar --no-same-owner -xpf - -C "${D}${SYSTEM_TESTS_TARGET_FILES_PATH}/BAT"
    }

:Google Test based bbclass example:

::

    # Load automatic test suite generation support
    inherit create_gtest_support_script

    # Provide one test suite named "BAT"
    do_install_append() {
        # Create the test suite folder
        install -d "${D}${SYSTEM_TESTS_TARGET_FILES_PATH}/BAT"
        # Install the test scripts into the folder
        tar --no-same-owner --exclude-vcs -cpf - -C "${S}/BAT" . \
            | tar --no-same-owner -xpf - -C "${D}${SYSTEM_TESTS_TARGET_FILES_PATH}/BAT"
    }


Standard xTEE based tests
===========================================

Tests are Python scripts that either contain test classes or test
functions. As the xPAD test execution environment is based on
`python-nose`_ almost all features of `python-nose`_ can be used in
the test scripts (see the examples ``test_uname()`` in chapter `System tests`_ or
``test_sanity_001_run_command_rc_check()`` in `SDK tests`_ below).

System tests
------------

:Example:

.. code-block:: python

    # Copyright (C) 2015. BMW Car IT GmbH. All rights reserved.
    from nose.tools import assert_regexp_matches

    from mtee.testing.support.target_share import TargetShare
    from mtee.testing.tools import assert_process_returncode, metadata


    @metadata(testsuite="BAT")
    class TestTargetSystem(object):
        # test needs a target for execution
        target = TargetShare().target

        @classmethod
        def setup_class(cls):
            # Class level setup method.

            # This will be executed once before the first test method of the class
            # is executed.
            pass

        @classmethod
        def teardown_class(cls):
            # Class level teardown method.

            # This will be executed once after the last test method of the class
            # is executed.
            pass

        def setup(self):
            # Setup needed for all test cases in the class. Will be executed before
            # each test case is executed.
            pass

        def teardown(self):
            # teardown needed for all test cases in the class. Will be executed after
            # each test case is executed.
            pass

        def test_uname(self):
            # Verify that uname -a string is correct
            result = self.target.execute_command('uname -a')
            assert_process_returncode(0, result, "Executing 'uname -a' failed")
            assert_regexp_matches(result.stdout, "^Linux.*")

        def test_execute_command_bg(self):
            # Start a process in background
            ctx = self.target.execute_background_task("tail -f /etc/hostname", shell=True)
            # pid can be received from the ctx object
            pid = ctx.pid
            out = ctx.recv_stdout()
            err = ctx.recv_stderr()

            # DO SOMETHING HERE
            # Close the process
            ctx.stop()

        def test_execute_command_bg_as_ctx(self):
            # Start a process in background
            with self.target.execute_background_task("tail -f /etc/hostname", shell=True) as ctx:
                pid = ctx.pid
                out = ctx.recv_stdout()
                err = ctx.recv_stderr()
                # DO SOMETHING HERE
            # The process is automatically closed after leaving the context manager

SDK tests
---------

:Example:

.. code-block:: python

    from mtee.testing.tools import assert_process_returncode, run_command, metadata


    @metadata(testsuite="BAT")
    class TestsSanitySDK(object):

        def test_sanity_001_run_command_rc_check(self):
            # SANITY_001 Test expected rc value check of run_command
            result = run_command("exit 1", shell=True)
            assert_process_returncode(1, result, "Return code check for 1 failed ({actual})")

Advanced test features (generators, generator decorator)
--------------------------------------------------------

If generators are used for testing, mixing between **assert** statements
and **yield** statements should be avoided.

Either only have yielded/generated cases in a test (like in ``test\_generator`` example below),
or only assert statements. When both, single assert statements and generators, are used in
a single test, the normal tests will not appear in the results. Only the yielded test results will appear.

.. note::

    - To get a properly formatted result and to determine which generated test failed,
      the set_description(function, param1, param2, ...) method should be called before the yield statement
      is executed. With this method additional information can be set that is relevant in the result.

    - Once set_description() is called on a function, it is advised to call it before each yield  statement,
      otherwise the name will be the same for all yielded test which can cause problems in reporting.

    - If a parameter is a path which includes the word "tmp" the path will be replaced to the string "tmp".

        "/tmp/rnd/file.txt" -> "tmp"

        "/var/pref_tmp/file.txt" -> "tmp"

To ease the process of using test generators, a decorator was created, ``nose_parametrize``.
The decorator can not be used on classes, just on functions and methods.
If used on class methods, for a more pleasant test run output, the test class should
inherit from ``PrettyPrintParameterized``.

:Examples:

.. code-block:: python

    import os

    from nose.tools import assert_true, assert_process_returncode, assert_regexp_matches, metadata

    from mtee.testing.support.set_test_description import set_description


    @metadata(testsuite="BAT")
    class ExampleTest(object):

        def test_nativesdk_support(self):
            """
            Generator test if sdk supports needed features.
            """

            symlink_list = ["ash", "busybox", "cat", "chattr", "chgrp", "chmod", "chown",
                            "cp", "cpio", "date", "dd", "df", "dmesg", "dnsdomainname",
                            "dumpkmap", "echo", "egrep", "false", "fgrep", "grep", "gunzip",
                            "gzip", "hostname", "kill", "ln", "login", "ls", "mkdir", "mknod",
                            "mktemp", "more", "mount", "mv", "netstat", "pidof", "ping",
                            "ping6", "ps", "pwd", "rm", "rmdir", "sed", "sh",
                            "sleep", "stty", "su", "sync", "tar", "touch", "true",
                            "umount", "uname", "usleep", "vi", "zcat"]

            for cmd in symlink_list:
                set_description(file_is_symlink, "file_is_symlink", "/bin/{}".format(cmd))
                yield file_is_symlink, "/bin/{}".format(cmd)

    def file_is_symlink(linkname, linkdst=None):
        """
        Returns True if @linkname is a symbolic link and if it points
        to the @linkdst
        """
        # assert it with the sdk builtin method
        assert_true(os.path.islink(linkname))
        if linkdst:
            assert_equal(os.readlink(linkname), linkdst)

.. code-block:: python

        from mtee.tools.nose_parametrize import nose_parametrize, PrettyPrintParameterized

        # Will generate two calls:
        # test_my_test_case(1, 2)
        # print message: test_my_test_case(1, 2) ... ok
        # test_my_test_case(3, 4)
        # print message: test_my_test_case(3, 4) ... ok
        @nose_parametrize((1, 2), (3, 4))
        def test_my_test_case(param1, param2):
            pass


        class TestWithPrettyTestNamePrint(PrettyPrintParameterized):

            # Will generate two calls:
            # test_my_test_case(1, 2)
            # print message: TestWithPrettyTestNamePrint.test_my_test_case_in_class(self, 1, 2) ... ok
            # test_my_test_case(3, 4)
            # print message: TestWithPrettyTestNamePrint.test_my_test_case_in_class(self, 3, 4) ... ok
            @nose_parametrize((1, 2), (3, 4))
            def test_my_test_case_in_class(param1, param2):
                pass


xTEE based tests that need test data on the device
==================================================

If the tests need additional test data on the device, it needs to be
supplied with the test scripts. Test data should be packaged in a sub
folder relative to the test script and use the supplied
``target.upload()`` method to upload it to the device under test in a
setup fixture. Of course the test script should take care that the data
is deleted again in the teardown fixture.

:Example:

.. code-block:: python

    import os
    import shutil

    from mtee.testing.tools import assert_process_returncode, run_command, metadata


    @metadata(testsuite="BAT")
    class TestsSdkDummyProject(object):

        sdk_destination = "/dummy_project"

        @classmethod
        def setupAll(cls):
            # DUMMY_PROJECT_001 Copy dummy project into the sysroot
            src_dir = os.path.join(os.path.dirname(os.path.realpath(__file__)), "test_data", "dummy_project")
            shutil.copytree(src_dir, cls.sdk_destination, symlinks=True)

        @classmethod
        def teardownAll(cls):
            if os.path.isdir(cls.sdk_destination):
                shutil.rmtree(cls.sdk_destination, True)

        def test_dummy_project_002_compile_hello_world(self):
            # DUMMY_PROJECT_002 Compile a Hello World application
            result = run_command(["/dummy_project/run_test_make.sh"])
            assert_process_returncode(0, result, "Building dummy project using make failed")

Extraction of files after test execution
========================================

It is possible to extract files from the target/SDK for later analysis. Extraction is
possible on test and context level. Contexts are standard nose contexts: modules,
classes and generator functions/methods.

-  **Test level extraction**
    Use the ``@target_extract("file1", "file2")`` or ``@sdk_extract("file1", "file2")`` decorator
    for test functions and methods

-  **Context level extraction**
    Set the variable/member ``__target_extract__`` or ``__sdk_extract__`` to a list of
    files you want to extract for modules and classes.

-  **Extraction for test generators**
    Functions and methods which generate tests are considered contexts in Nose. However, the extraction
    syntax is the same as if they were tests. Extraction happens only once, after all generated
    tests are executed.
    Use the ``@target_extract("file1", "file2")``  or ``@sdk_extract("file1", "file2")`` decorator.

-  **Extraction for generated (yielded) tests**
    To extract files for tests generated (yielded) by test generators use
    the ``@target_extract("file1", "file2")`` or ``@sdk_extract("file1", "file2")``
    decorator. The files will be extracted in each iteration.

:References:

    :ref:`@sdk_extract() <sdk_extract>`

    :ref:`@target_extract() <target_extract>`

:Examples:

.. literalinclude:: ../../tests/scripts/target_extract_tests.py

.. literalinclude:: ../../tests/scripts/sdk_extract_tests.py


Integration tests created with Google Test
==========================================

Tests based on Google Test usually have one or more binaries that need
to be executed on target and possibly some test data needed for target execution.

These can either be packaged and deployed as described in the `Google Test suite
conventions`_ or can be customized as described here using a small Python
script to support the execution of these tests in the CI system.

Workflow for system tests
-------------------------

Test developer develops the tests as usual using Google Test. In
addition the developer needs to make sure that the tests are installed
into ``${SYSTEM_TESTS_TARGET_FILES_PATH}`` during the installation
procedure. This can also be handled as an additional parameter for the
install procedure. See example below. In addition, the :ref:`recipe <recipe>` needs to
inherit the systemtest class to facilitate packaging the tests.

After that the developer needs to create a small python script to make
the tests visible to the CI system. A simple example can be seen below.
This can be extended to include more setup/teardown tasks, depending on
the tests.

At the end the created test packages need to be deployed to the correct
images. Therefore the developer needs to add the packages to:

-  ``packagegroup-systemtests.bb`` or a ``.bbappend`` file in the layer.
-  ``packagegroup-systemtests-targetfiles.bb`` or a ``.bbappend`` file
   in the layer.

Python script for system tests
------------------------------

.. code-block:: python

    import os

    from nose.tools import assert_true, assert_regexp_matches

    from mtee.testing.support.target_share import TargetShare
    from mtee.testing.tools import gtests_run, metadata

    target = TargetShare().target

    @metadata(testsuite="BAT", teststage=["unit", "integration"])
    class TestGoogleTests(object):

        @classmethod
        def setup_class(cls):
            # Prepare test environment. Install needed packages.
            # Make sure the package was properly installed before starting any
            # further tests
            cls.deploy_dir = target.targetfiles_path(__file__)
            assert_true(target.isdir(cls.deploy_dir),
                        "Package directory %s not found" % cls.deploy_dir)

        def test_component_googletests(self):
            integration_path = os.path.join(self.deploy_dir, "integration")
            for test_file in target.listdir(integration_path):
                test_binary = os.path.join(integration_path, test_file)
                if not target.isfile(test_binary):
                    continue
                for test_case in gtests_run(target, test_binary):
                    yield test_case

        def test_uname(self):
            result = target.execute_command('uname -a')
            assert_process_returncode(0, result, "Executing 'uname -a' failed")
            assert_regexp_matches(result.stdout, "^Linux.*")

.. note::

    The local execution of new test cases will fail in ``target.targetfiles_path``
    as long as the test is not installed through a test package. In order to
    execute the test ``cls.deploy_dir`` has to be patched to point to the hard
    coded path where the necessary files (test binaries) are located (e.g.
    ``/usr/lib/mypackage/tests``).

.. _recipe:

Recipe additions for system tests
---------------------------------

::

    ...
    # add systemtest to the inherits
    inherit systemtest

    # make sure that the system test binaries are installed to the correct folder:
    EXTRA_OECMAKE += "\
                      -DINSTALL_TESTS=${SYSTEM_TESTS_TARGET_FILES_PATH} \
                     "

    # install the python test script and replace package names and deploy dir.
    do_install_systemtests() {
        install -m 0644 "${WORKDIR}/target_ramses_tests.py" "${D}${SYSTEM_TESTS_PATH}"
    }

Python script for sdk tests
---------------------------

.. code-block:: python

    import os

    from nose.tools import assert_true

    from mtee.testing.tools import gtests_run, metadata


    @metadata(testsuite="BAT", teststage=["unit", "integration"])
    class TestGoogleTests(object):
        deploy_dir = "<path to Google Test binaries>"

        @classmethod
        def setup_class(cls):
            # Prepare test environment. Install needed packages.
            # Make sure the package was properly installed before starting any
            # further tests
            assert_true(target.isdir(cls.deploy_dir),
                        "Package directory %s not found" % cls.deploy_dir)

        def test_component_googletests(self):
            integration_path = os.path.join(self.deploy_dir, "integration")
            for test_file in os.listdir(integration_path):
                test_binary = os.path.join(integration_path, test_file)
                if not os.path.isfile(test_binary):
                    continue
                for test_case in gtests_run(None, test_binary):
                    yield test_case


****************
Test environment
****************

Framework allows marking tests for execution only if environment
prerequisites are met. For example target is hardware, or supports CAN,
etc. On setup/teardown methods within classes, e.g. setup_class(cls) there is an
additional decorator required to skip execution.

Supported
=========

::

    "target":
        "hardware"
        "simulator"
    "target_type":
        "xpad-qemu",
        "xpad-hpad"


Example usage
=============

.. code-block:: python

    from mtee.testing.test_environment import require_environment, require_environment_setup
    from mtee.testing.test_environment import TEST_ENVIRONMENT as te

    target = TargetShare().target

    @require_environment(te.target.hardware)
    @metadata(testsuite="BAT")
    def test_valid_for_hardware_targets():
        pass

    @require_environment(te.target.simulator)
    @metadata(testsuite="BAT")
    def test_valid_for_simulated_targets():
        pass

    @require_environment(te.target.hardware, te.target_type.rse)
    @metadata(testsuite="BAT")
    def test_valid_only_for_hardware_RSE():
        pass

    @require_environment(te.target.hardware, te.target_type.hu, te.hardware_variant.mid)
    @metadata(testsuite="BAT")
    def test_valid_only_for_HU_mid():
        pass

    @require_environment(te.target.hardware, te.target_type.hu)
    class TestsClassWithSetupMethod(object)

        @classmethod
        @require_environment_setup(te.target.hardware, te.target_type.hu)
        def setup_class(cls):
            pass

        @classmethod
        @require_environment_setup(te.target.hardware, te.target_type.hu)
        def teardown_class(cls):
            pass

        def test_valid_only_for_hardware_HU(self):
            pass

.. _python-nose: https://nose.readthedocs.org/en/latest/
