###########################################################
Test suites and test management for xPAD CI test executions
###########################################################

***************
Test management
***************

Basics
======

Test management for xPAD is done in two levels.

- **on test script/test package level**:
    single test script or folders of test scripts can be added to a test suite or excluded from a test suite.
- **on test case level inside the test scripts**:
    inside a test script individual test cases or groups of test cases can be labeled for different
    test suites with attributes. If no attributes are added to a test case it does not belong to any
    test suite. For test management different combinations of attributes can be selected into a
    test suite or excluded from a test suite.

:Example:

    Test suite ``BAT``

    - All test cases that should be included need to have the attribute BAT added by the developers.
    - All the test scripts that contain test cases included in the BAT suite need to be included in the
      test suite definition file for BAT (BAT-systemtests)


Test suite definition
=====================

Each test suite is defined in a set of files. Currently the test runner expects the test suite definition in
a folder ``test-suites`` next to the test images.

- test suite filter definition file
    This file contains the filter definition for the attributes.
    Filename: ``<test_suite_name>``
- test suite test script/package file
    Filename: ``<test_suite_name>-<sdktests|systemtests>``
- one or more test suite target specific test script/package files
    Filename: ``<test_suite_name>-<sdktests|systemtests>-<targetid>``

Test suite filter definition file
---------------------------------
This file defines the attribute filter to be used for the test suite. It contains the attribute filter
arguments as required by the Attrib plugin for python-nose. See the
`documentation of the nose plugin <http://nose.readthedocs.org/en/latest/plugins/attrib.html>`_.

Example

.. code-block:: bash

    -a testsuite=BAT -a BAT

or

.. code-block:: bash

    -a '!duration=longterm'


Test suite test script/package file
-----------------------------------
Contains all the test scripts/packages included in the test suite, one script/package per line.

Test suite target specific test script/package file
---------------------------------------------------
Contains test scripts valid for a specific build target only. E.g. scripts that exist only for HW targets as
they are provided in layers that are not included when building the other targets.

Adding test cases to test suites
================================

When adding test cases to a test suite you need to add the correct attribute to it, so it can be picked up
by the filter. This is done by using the ``@metadata`` decorator, which will also put the metadata into the
test report.

.. code-block:: python

    from mtee.testing.tools import metadata

    @metadata(testsuite="BAT")
    def test_example():

If the test case/class should be assigned to more than one test suite you need to use a list. E.g.:
``@metadata(testsuite=["BAT", "domain"])``. Other attributes that can be used are specified here:
:ref:`attributes_best_practices`

***********
Test suites
***********

Test suites used in CI
======================

TODO

*********************************************
How to use Python modules with external tests
*********************************************

The directory given as the *--test-dir* option will be added to the
PYTHONPATH so that the tests can import modules living in this directory. In
the example below the test can make use of the Python package ``my_module``.

::

    |-- my_custom_folder
        |__ test_package_a
            |__ sdktests
            |   |__ example_a_sdk_test.py
            |   |__ [...]
            |__ systemtest
                |__ example_a_target_test.py
                |__ [...]
        |__ test_package_b
            |__ systemtest
                |__ example_b_target_test.py
                |__ [...]
        |__ my_module
            |__ __init__.py
            |__ foobar.py
            |__ sub_module
                |__ __init__.py
                |__ example.py
                |__ [...]

.. note::

    Please note that this is only recommended and possible to be used in case
    it's not planned to include the tests in the SDK. This mechanism can not
    be used for BAT tests since those need to be installed in the SDK.
    Please contact us in case there is a need for such functionality for
    BAT tests.

********
Glossary
********

**test script**
    one single python script file ending on _tests.py and containing one or more test cases/classes/methods

**test package**
    a folder containing one or more test scripts
