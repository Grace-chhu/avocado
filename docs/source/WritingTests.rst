.. _writing-tests:

=====================
Writing Avocado Tests
=====================

We are going to write an Avocado test in Python and we are going to inherit from
:class:`avocado.Test`. This makes this test a so-called instrumented test.

Basic example
=============

Let's re-create an old time favorite, ``sleeptest`` [#f1]_.  It is so simple, it
does nothing besides sleeping for a while::

        import time

        from avocado import Test

        class SleepTest(Test):

            def test(self):
                sleep_length = self.params.get('sleep_length', default=1)
                self.log.debug("Sleeping for %.2f seconds", sleep_length)
                time.sleep(sleep_length)

This is about the simplest test you can write for Avocado, while still
leveraging its API power.

What is an Avocado Test
-----------------------

As can be seen in the example above, an Avocado test is a method that
starts with ``test`` in a class that inherits from :mod:`avocado.Test`.

Multiple tests and naming conventions
-------------------------------------

You can have multiple tests in a single class.

To do so, just give the methods names that start with ``test``, say
``test_foo``, ``test_bar`` and so on. We recommend you follow this naming
style, as defined in the `PEP8 Function Names`_ section.

For the class name, you can pick any name you like, but we also recommend
that it follows the CamelCase convention, also known as CapWords, defined
in the PEP 8 document under `Class Names`_.

Convenience Attributes
----------------------

Note that the test class provides you with a number of convenience attributes:

* A ready to use log mechanism for your test, that can be accessed by means
  of ``self.log``. It lets you log debug, info, error and warning messages.
* A parameter passing system (and fetching system) that can be accessed by
  means of ``self.params``. This is hooked to the Varianter, about which
  you can find that more information at :doc:`TestParameters`.
* And many more (see :mod:`avocado.core.test.Test`)

To minimize the accidental clashes we define the public ones as properties
so if you see something like ``AttributeError: can't set attribute`` double
you are not overriding these.

.. _Test statuses:

Test statuses
=============

Avocado supports the most common exit statuses:

* ``PASS`` - test passed, there were no untreated exceptions
* ``WARN`` - a variant of ``PASS`` that keeps track of noteworthy events
  that ultimately do not affect the test outcome. An example could be
  ``soft lockup`` present in the ``dmesg`` output. It's not related to the
  test results and unless there are failures in the test it means the feature
  probably works as expected, but there were certain condition which might
  be nice to review. (some result plugins does not support this and report
  ``PASS`` instead)
* ``SKIP`` - the test's pre-requisites were not satisfied and the test's
  body was not executed (nor its ``setUp()`` and ``tearDown``).
* ``CANCEL`` - the test was canceled somewhere during the ``setUp()``, the
  test method or the ``tearDown()``. The ``setUp()`` and ``tearDown``
  methods are executed.
* ``FAIL`` - test did not result in the expected outcome. A failure points
  at a (possible) bug in the tested subject, and not in the test itself.
  When the test (and its) execution breaks, an ``ERROR`` and not a ``FAIL``
  is reported."
* ``ERROR`` - this points (probably) at a bug in the test itself, and not
  in the subject being tested.It is usually caused by uncaught exception
  and such failures needs to be thoroughly explored and should lead to
  test modification to avoid this failure or to use ``self.fail`` along
  with description how the subject under testing failed to perform it's
  task.
* ``INTERRUPTED`` - this result can't be set by the test writer, it is
  only possible when the timeout is reached or when the user hits
  ``CTRL+C`` while executing this test.
* other - there are some other internal test statuses, but you should not
  ever face them.

As you can see the ``FAIL`` is a neat status, if tests are developed
correctly. When writing tests always think about what its ``setUp``
should be, what the ``test body`` and is expected to go wrong in the
test. To support you Avocado supports several methods:

Test methods
------------

The simplest way to set the status is to use ``self.fail``,
``self.error`` or ``self.cancel`` directly from test.

To remember a warning, one simply writes to ``self.log.warning``
logger. This won't interrupt the test execution, but it will
remember the condition and, if there are no failures, will
report the test as ``WARN``.

Turning errors into failures
----------------------------

Errors on Python code are commonly signaled in the form of exceptions
being thrown.  When Avocado runs a test, any unhandled exception will
be seen as a test ``ERROR``, and not as a ``FAIL``.

Still, it's common to rely on libraries, which usually raise custom
(or builtin) exceptions. Those exceptions would normally result in
``ERROR`` but if you are certain this is an odd behavior of the
object under testing, you should catch the exception and explain
the failure in ``self.fail`` method::

    try:
        process.run("stress_my_feature")
    except process.CmdError as details:
        self.fail("The stress comamnd failed: %s" % details)

If your test compounds of many executions and you can't get this exception
in other case then expected failure, you can simplify the code by using
``fail_on`` decorator::

    @avocado.fail_on(process.CmdError)
    def test(self):
        process.run("first cmd")
        process.run("second cmd")
        process.run("third cmd")

Once again, keeping your tests up-to-date and distinguishing between
``FAIL`` and ``ERROR`` will save you a lot of time while reviewing the
test results.

Saving test generated (custom) data
===================================

Each test instance provides a so called ``whiteboard``. It can be accessed
through ``self.whiteboard``. This whiteboard is simply a string that will be
automatically saved to test results after the test finishes (it's not synced
during the execution so when the machine or python crashes badly it might
not be present and one should use direct io to the ``outputdir`` for
critical data). If you choose to save binary data to the whiteboard,
it's your responsibility to encode it first (base64 is the obvious choice).

Building on the previously demonstrated ``sleeptest``, suppose that you want to save the
sleep length to be used by some other script or data analysis tool::

        def test(self):
            sleep_length = self.params.get('sleep_length', default=1)
            self.log.debug("Sleeping for %.2f seconds", sleep_length)
            time.sleep(sleep_length)
            self.whiteboard = "%.2f" % sleep_length

The whiteboard can and should be exposed by files generated by the available test result
plugins. The ``results.json`` file already includes the whiteboard for each test.
Additionally, we'll save a raw copy of the whiteboard contents on a file named
``whiteboard``, in the same level as the ``results.json`` file, for your convenience
(maybe you want to use the result of a benchmark directly with your custom made scripts
to analyze that particular benchmark result).

If you need to attach several output files, you can also use
``self.outputdir``, which points to the
``$RESULTS/test-results/$TEST_ID/data`` location and is reserved for
arbitrary test result data.

.. _accessing-test-data-files:

Accessing test data files
=========================

Some tests can depend on data files, external to the test file itself.
Avocado provides a test API that makes it really easy to access such
files: :meth:`get_data() <avocado.core.test.TestData.get_data>`.

For Avocado tests (that is, ``INSTRUMENTED`` tests)
:meth:`get_data() <avocado.core.test.TestData.get_data>` allows test data files
to be accessed from up to three sources:

 * **file** level data directory: a directory named after the test file, but
   ending with ``.data``.  For a test file ``/home/user/test.py``, the file level
   data directory is ``/home/user/test.py.data/``.

 * **test** level data directory: a directory named after the test file and the
   specific test name.  These are useful when different tests part of the
   same file need different data files (with the same name or not).  Considering
   the previous example of ``/home/user/test.py``, and supposing it contains two
   tests, ``MyTest.test_foo`` and ``MyTest.test_bar``, the test level data
   directories will be, ``/home/user/test.py.data/MyTest.test_foo/`` and
   ``home/user/test.py.data/MyTest.test_bar/`` respectively.

 * **variant** level data directory: if variants are being used during the test
   execution, a directory named after the variant will also be considered when
   looking for test data files.  For test file ``/home/user/test.py``, and test
   ``MyTest.test_foo``, with variant ``debug-ffff``, the data directory path
   will be ``/home/user/test.py.data/MyTest.test_foo/debug-ffff/``.

.. note:: Unlike INSTRUMENTED tests, SIMPLE tests only define ``file``
          and ``variant`` data_dirs, therefore the most-specific data-dir
          might look like ``/bin/echo.data/debug-ffff/``.

Avocado looks for data files in the order defined at
:attr:`DATA_SOURCES <avocado.core.test.TestData.DATA_SOURCES>`, which are
from most specific one, to most generic one.  That means that, if a variant
is being used, the **variant** directory is used first.  Then the **test**
level directory is attempted, and finally the **file** level directory.
Additionally you can use ``get_data(filename, must_exist=False)`` to get
expected location of a possibly non-existing file, which is useful when
you intend to create it.

.. tip:: When running tests you can use the ``--log-test-data-directories``
         command line option log the test data directories that will be used
         for that specific test and execution conditions (such as with or
         without variants).  Look for "Test data directories" in the test logs.

.. note:: An older API, :attr:`avocado.core.test.Test.datadir`, allows access
          to the data directory based on the test file location only.  This API
          is limited, deprecated and will be removed.  All new users should rely
          on ``get_data()`` instead.

.. _accessing-test-parameters:

Accessing test parameters
=========================

Each test has a set of parameters that can be accessed through
``self.params.get($name, $path=None, $default=None)`` where:

* name - name of the parameter (key)
* path - where to look for this parameter (when not specified uses mux-path)
* default - what to return when param not found

The path is a bit tricky. Avocado uses tree to represent parameters. In simple
scenarios you don't need to worry and you'll find all your values in default
path, but eventually you might want to check-out :doc:`TestParameters` to understand
the details.

Let's say your test receives following params (you'll learn how to execute
them in the following section)::

    $ avocado variants -m examples/tests/sleeptenmin.py.data/sleeptenmin.yaml --variants 2
    ...
    Variant 1:    /run/sleeptenmin/builtin, /run/variants/one_cycle
        /run/sleeptenmin/builtin:sleep_method => builtin
        /run/variants/one_cycle:sleep_cycles  => 1
        /run/variants/one_cycle:sleep_length  => 600
    ...

In test you can access those params by:

.. code-block:: python

    self.params.get("sleep_method")    # returns "builtin"
    self.params.get("sleep_cycles", '*', 10)    # returns 1
    self.params.get("sleep_length", "/*/variants/*"  # returns 600

.. note:: The path is important in complex scenarios where clashes might
          occur, because when there are multiple values with the same
          key matching the query avocado raises an exception. As mentioned
          you can avoid those by using specific paths or by defining
          custom mux-path which allows specifying resolving hierarchy.
          More details can be found in :doc:`TestParameters`.


Running multiple variants of tests
==================================

In the previous section we described how parameters are handled.  Now,
let's have a look at how to produce them and execute your tests with
different parameters.

The variants subsystem is what allows the creation of multiple
variations of parameters, and the execution of tests with those
parameter variations.  This subsystem is pluggable, so you might use
custom plugins to produce variants.  To keep things simple, let's
use Avocado's primary implementation, called "yaml_to_mux".

The "yaml_to_mux" plugin accepts YAML files.  Those will create a
tree-like structure, store the variables as parameters and use custom
tags to mark locations as "multiplex" domains.

Let's use ``examples/tests/sleeptenmin.py.data/sleeptenmin.yaml`` file
as an example:

.. code-block:: yaml

   sleeptenmin: !mux
       builtin:
           sleep_method: builtin
       shell:
           sleep_method: shell
   variants: !mux
       one_cycle:
           sleep_cycles: 1
           sleep_length: 600
       six_cycles:
           sleep_cycles: 6
           sleep_length: 100
       one_hundred_cycles:
           sleep_cycles: 100
           sleep_length: 6
       six_hundred_cycles:
           sleep_cycles: 600
           sleep_length: 1

Which produces following structure and parameters::

  $ avocado variants -m examples/tests/sleeptenmin.py.data/sleeptenmin.yaml --summary 2 --variants 2
  Multiplex tree representation:
   ┗━━ run
        ┣━━ sleeptenmin
        ┃    ╠══ builtin
        ┃    ║     → sleep_method: builtin
        ┃    ╚══ shell
        ┃          → sleep_method: shell
        ┗━━ variants
             ╠══ one_cycle
             ║     → sleep_length: 600
             ║     → sleep_cycles: 1
             ╠══ six_cycles
             ║     → sleep_length: 100
             ║     → sleep_cycles: 6
             ╠══ one_hundred_cycles
             ║     → sleep_length: 6
             ║     → sleep_cycles: 100
             ╚══ six_hundred_cycles
                   → sleep_length: 1
                   → sleep_cycles: 600

  Multiplex variants (8):

  Variant builtin-one_cycle-f659:    /run/sleeptenmin/builtin, /run/variants/one_cycle
      /run/sleeptenmin/builtin:sleep_method => builtin
      /run/variants/one_cycle:sleep_cycles  => 1
      /run/variants/one_cycle:sleep_length  => 600

  Variant builtin-six_cycles-723b:    /run/sleeptenmin/builtin, /run/variants/six_cycles
      /run/sleeptenmin/builtin:sleep_method => builtin
      /run/variants/six_cycles:sleep_cycles => 6
      /run/variants/six_cycles:sleep_length => 100

  Variant builtin-one_hundred_cycles-633a:    /run/sleeptenmin/builtin, /run/variants/one_hundred_cycles
      /run/sleeptenmin/builtin:sleep_method         => builtin
      /run/variants/one_hundred_cycles:sleep_cycles => 100
      /run/variants/one_hundred_cycles:sleep_length => 6

  Variant builtin-six_hundred_cycles-a570:    /run/sleeptenmin/builtin, /run/variants/six_hundred_cycles
      /run/sleeptenmin/builtin:sleep_method         => builtin
      /run/variants/six_hundred_cycles:sleep_cycles => 600
      /run/variants/six_hundred_cycles:sleep_length => 1

  Variant shell-one_cycle-55f5:    /run/sleeptenmin/shell, /run/variants/one_cycle
      /run/sleeptenmin/shell:sleep_method  => shell
      /run/variants/one_cycle:sleep_cycles => 1
      /run/variants/one_cycle:sleep_length => 600

  Variant shell-six_cycles-9e23:    /run/sleeptenmin/shell, /run/variants/six_cycles
      /run/sleeptenmin/shell:sleep_method   => shell
      /run/variants/six_cycles:sleep_cycles => 6
      /run/variants/six_cycles:sleep_length => 100

  Variant shell-one_hundred_cycles-586f:    /run/sleeptenmin/shell, /run/variants/one_hundred_cycles
      /run/sleeptenmin/shell:sleep_method           => shell
      /run/variants/one_hundred_cycles:sleep_cycles => 100
      /run/variants/one_hundred_cycles:sleep_length => 6

  Variant shell-six_hundred_cycles-1e84:    /run/sleeptenmin/shell, /run/variants/six_hundred_cycles
      /run/sleeptenmin/shell:sleep_method           => shell
      /run/variants/six_hundred_cycles:sleep_cycles => 600
      /run/variants/six_hundred_cycles:sleep_length => 1

You can see that it creates all possible variants of each ``multiplex domain``,
which are defined by ``!mux`` tag in the YAML file and displayed as single
lines in tree view (compare to double lines which are individual nodes with
values). In total it'll produce 8 variants of each test::

      $ avocado run --mux-yaml examples/tests/sleeptenmin.py.data/sleeptenmin.yaml -- passtest.py
      JOB ID     : cc7ef22654c683b73174af6f97bc385da5a0f02f
      JOB LOG    : /home/medic/avocado/job-results/job-2017-01-22T11.26-cc7ef22/job.log
       (1/8) passtest.py:PassTest.test;builtin-one_cycle-f659: PASS (0.01 s)
       (2/8) passtest.py:PassTest.test;builtin-six_cycles-723b: PASS (0.01 s)
       (3/8) passtest.py:PassTest.test;builtin-one_hundred_cycles-633a: PASS (0.01 s)
       (4/8) passtest.py:PassTest.test;builtin-six_hundred_cycles-a570: PASS (0.01 s)
       (5/8) passtest.py:PassTest.test;shell-one_cycle-55f5: PASS (0.01 s)
       (6/8) passtest.py:PassTest.test;shell-six_cycles-9e23: PASS (0.01 s)
       (7/8) passtest.py:PassTest.test;shell-one_hundred_cycles-586f: PASS (0.01 s)
       (8/8) passtest.py:PassTest.test;shell-six_hundred_cycles-1e84: PASS (0.01 s)
      RESULTS    : PASS 8 | ERROR 0 | FAIL 0 | SKIP 0 | WARN 0 | INTERRUPT 0
      JOB TIME   : 0.16 s

There are other options to influence the params so please check out
``avocado run -h`` and for details use :doc:`TestParameters`.


Advanced logging capabilities
=============================

Avocado provides advanced logging capabilities at test run time.  These can
be combined with the standard Python library APIs on tests.

One common example is the need to follow specific progress on longer or more
complex tests. Let's look at a very simple test example, but one multiple
clear stages on a single test::

    import logging
    import time

    from avocado import Test

    progress_log = logging.getLogger("progress")

    class Plant(Test):

        def test_plant_organic(self):
            rows = self.params.get("rows", default=3)

            # Preparing soil
            for row in range(rows):
                progress_log.info("%s: preparing soil on row %s",
                                  self.name, row)

            # Letting soil rest
            progress_log.info("%s: letting soil rest before throwing seeds",
                              self.name)
            time.sleep(2)

            # Throwing seeds
            for row in range(rows):
                progress_log.info("%s: throwing seeds on row %s",
                                  self.name, row)

            # Let them grow
            progress_log.info("%s: waiting for Avocados to grow",
                              self.name)
            time.sleep(5)

            # Harvest them
            for row in range(rows):
                progress_log.info("%s: harvesting organic avocados on row %s",
                                  self.name, row)


From this point on, you can ask Avocado to show your logging stream, either
exclusively or in addition to other builtin streams::

    $ avocado --show app,progress run plant.py

The outcome should be similar to::

    JOB ID     : af786f86db530bff26cd6a92c36e99bedcdca95b
    JOB LOG    : /home/cleber/avocado/job-results/job-2016-03-18T10.29-af786f8/job.log
     (1/1) plant.py:Plant.test_plant_organic: progress: 1-plant.py:Plant.test_plant_organic: preparing soil on row 0
    progress: 1-plant.py:Plant.test_plant_organic: preparing soil on row 1
    progress: 1-plant.py:Plant.test_plant_organic: preparing soil on row 2
    progress: 1-plant.py:Plant.test_plant_organic: letting soil rest before throwing seeds
    -progress: 1-plant.py:Plant.test_plant_organic: throwing seeds on row 0
    progress: 1-plant.py:Plant.test_plant_organic: throwing seeds on row 1
    progress: 1-plant.py:Plant.test_plant_organic: throwing seeds on row 2
    progress: 1-plant.py:Plant.test_plant_organic: waiting for Avocados to grow
    \progress: 1-plant.py:Plant.test_plant_organic: harvesting organic avocados on row 0
    progress: 1-plant.py:Plant.test_plant_organic: harvesting organic avocados on row 1
    progress: 1-plant.py:Plant.test_plant_organic: harvesting organic avocados on row 2
    PASS (7.01 s)
    RESULTS    : PASS 1 | ERROR 0 | FAIL 0 | SKIP 0 | WARN 0 | INTERRUPT 0
    JOB TIME   : 7.11 s
    JOB HTML   : /home/cleber/avocado/job-results/job-2016-03-18T10.29-af786f8/html/results.html

The custom ``progress`` stream is combined with the application output, which
may or may not suit your needs or preferences. If you want the ``progress``
stream to be sent to a separate file, both for clarity and for persistence,
you can run Avocado like this::

    $ avocado run plant.py --store-logging-stream progress

The result is that, besides all the other log files commonly generated, there
will be another log file named ``progress.INFO`` at the job results
dir. During the test run, one could watch the progress with::

    $ tail -f ~/avocado/job-results/latest/progress.INFO
    10:36:59 INFO | 1-plant.py:Plant.test_plant_organic: preparing soil on row 0
    10:36:59 INFO | 1-plant.py:Plant.test_plant_organic: preparing soil on row 1
    10:36:59 INFO | 1-plant.py:Plant.test_plant_organic: preparing soil on row 2
    10:36:59 INFO | 1-plant.py:Plant.test_plant_organic: letting soil rest before throwing seeds
    10:37:01 INFO | 1-plant.py:Plant.test_plant_organic: throwing seeds on row 0
    10:37:01 INFO | 1-plant.py:Plant.test_plant_organic: throwing seeds on row 1
    10:37:01 INFO | 1-plant.py:Plant.test_plant_organic: throwing seeds on row 2
    10:37:01 INFO | 1-plant.py:Plant.test_plant_organic: waiting for Avocados to grow
    10:37:06 INFO | 1-plant.py:Plant.test_plant_organic: harvesting organic avocados on row 0
    10:37:06 INFO | 1-plant.py:Plant.test_plant_organic: harvesting organic avocados on row 1
    10:37:06 INFO | 1-plant.py:Plant.test_plant_organic: harvesting organic avocados on row 2

The very same ``progress`` logger, could be used across multiple test methods
and across multiple test modules.  In the example given, the test name is used
to give extra context.

:class:`unittest.TestCase` heritage
===================================

Since an Avocado test inherits from :class:`unittest.TestCase`, you
can use all the assertion methods that its parent.

The code example bellow uses :meth:`assertEqual
<unittest.TestCase.assertEqual>`, :meth:`assertTrue
<unittest.TestCase.assertTrue>` and :meth:`assertIsInstace
<unittest.TestCase.assertIsInstance>`::

    from avocado import Test

    class RandomExamples(Test):
        def test(self):
            self.log.debug("Verifying some random math...")
            four = 2 * 2
            four_ = 2 + 2
            self.assertEqual(four, four_, "something is very wrong here!")

            self.log.debug("Verifying if a variable is set to True...")
            variable = True
            self.assertTrue(variable)

            self.log.debug("Verifying if this test is an instance of test.Test")
            self.assertIsInstance(self, test.Test)

Running tests under other :mod:`unittest` runners
-------------------------------------------------

`nose <https://nose.readthedocs.org/>`__ is another Python testing framework
that is also compatible with :mod:`unittest`.

Because of that, you can run avocado tests with the ``nosetests`` application::

    $ nosetests examples/tests/sleeptest.py
    .
    ----------------------------------------------------------------------
    Ran 1 test in 1.004s

    OK

Conversely, you can also use the standard :func:`unittest.main` entry point to run an
Avocado test. Check out the following code, to be saved as ``dummy.py``::

   from avocado import Test
   from unittest import main

   class Dummy(Test):
       def test(self):
           self.assertTrue(True)

   if __name__ == '__main__':
       main()

It can be run by::

   $ python dummy.py
   .
   ----------------------------------------------------------------------
   Ran 1 test in 0.000s

   OK

But we'd still recommend using ``avocado.main`` instead which is our main entry point.

.. _Setup and cleanup methods:

Setup and cleanup methods
=========================

To perform setup actions before/after your test, you may use ``setUp``
and ``tearDown`` methods. The ``tearDown`` method is always executed
even on ``setUp`` failure so don't forget to initialize your variables
early in the ``setUp``. Example of usage is in the next section
`Running third party test suites`_.

Running third party test suites
===============================

It is very common in test automation workloads to use test suites developed
by third parties. By wrapping the execution code inside an Avocado test module,
you gain access to the facilities and API provided by the framework. Let's
say you want to pick up a test suite written in C that it is in a tarball,
uncompress it, compile the suite code, and then executing the test. Here's
an example that does that::

    #!/usr/bin/env python

    import os

    from avocado import Test
    from avocado import main
    from avocado.utils import archive
    from avocado.utils import build
    from avocado.utils import process


    class SyncTest(Test):

        """
        Execute the synctest test suite.
        """
        def setUp(self):
            """
            Set default params and build the synctest suite.
            """
            sync_tarball = self.params.get('sync_tarball',
                                           default='synctest.tar.bz2')
            self.sync_length = self.params.get('sync_length', default=100)
            self.sync_loop = self.params.get('sync_loop', default=10)
            # Build the synctest suite
            self.cwd = os.getcwd()
            tarball_path = self.get_data(sync_tarball)
            archive.extract(tarball_path, self.workdir)
            self.workdir = os.path.join(self.workdir, 'synctest')
            build.make(self.workdir)

        def test(self):
            """
            Execute synctest with the appropriate params.
            """
            os.chdir(self.workdir)
            cmd = ('./synctest %s %s' %
                   (self.sync_length, self.sync_loop))
            process.system(cmd)
            os.chdir(self.cwd)


    if __name__ == "__main__":
        main()

Here we have an example of the ``setUp`` method in action: Here we get the
location of the test suite code (tarball) through
:func:`avocado.Test.get_data`, then uncompress the tarball through
:func:`avocado.utils.archive.extract`, an API that will
decompress the suite tarball, followed by :func:`avocado.utils.build.make`, that will build
the suite.

In this example, the ``test`` method just gets into the base directory of
the compiled suite  and executes the ``./synctest`` command, with appropriate
parameters, using :func:`avocado.utils.process.system`.

Fetching asset files
====================

To run third party test suites as mentioned above, or for any other purpose,
we offer an asset fetcher as a method of Avocado Test class.
The asset method looks for a list of directories in the ``cache_dirs`` key,
inside the ``[datadir.paths]`` section from the configuration files. Read-only
directories are also supported. When the asset file is not present in any of
the provided directories, we will try to download the file from the provided
locations, copying it to the first writable cache directory. Example::

    cache_dirs = ['/usr/local/src/', '~/avocado/cache']

In the example above, ``/usr/local/src/`` is a read-only directory. In that
case, when we need to fetch the asset from the locations, it will be copied to
the ``~/avocado/cache`` directory.

If you don't provide a ``cache_dirs``, we will create a ``cache`` directory
inside the avocado ``data_dir`` location to put the fetched files in.

* Use case 1: no ``cache_dirs`` key in config files, only the asset name
  provided in the full url format::

    ...
        def setUp(self):
            stress = 'http://people.seas.harvard.edu/~apw/stress/stress-1.0.4.tar.gz'
            tarball = self.fetch_asset(stress)
            archive.extract(tarball, self.workdir)
    ...

  In this case, ``fetch_asset()`` will download the file from the url provided,
  copying it to the ``$data_dir/cache`` directory. ``tarball`` variable  will
  contains, for example, ``/home/user/avocado/data/cache/stress-1.0.4.tar.gz``.

* Use case 2: Read-only cache directory provided. ``cache_dirs = ['/mnt/files']``::

    ...
        def setUp(self):
            stress = 'http://people.seas.harvard.edu/~apw/stress/stress-1.0.4.tar.gz'
            tarball = self.fetch_asset(stress)
            archive.extract(tarball, self.workdir)
    ...

  In this case, we try to find ``stress-1.0.4.tar.gz`` file in ``/mnt/files``
  directory. If it's not there, since ``/mnt/files`` is read-only,  we will try
  to download the asset file to the ``$data_dir/cache`` directory.

* Use case 3: Writable cache directory provided, along with a list of
  locations. ``cache_dirs = ['~/avocado/cache']``::

    ...
        def setUp(self):
            st_name = 'stress-1.0.4.tar.gz'
            st_hash = 'e1533bc704928ba6e26a362452e6db8fd58b1f0b'
            st_loc = ['http://people.seas.harvard.edu/~apw/stress/stress-1.0.4.tar.gz',
                      'ftp://foo.bar/stress-1.0.4.tar.gz']
            tarball = self.fetch_asset(st_name, asset_hash=st_hash,
                                       locations=st_loc)
            archive.extract(tarball, self.workdir)
    ...

  In this case, we try to download ``stress-1.0.4.tar.gz`` from the provided
  locations list (if it's not already in ``~/avocado/cache``). The hash was
  also provided, so we will verify the hash. To do so, we first look for a
  hashfile named ``stress-1.0.4.tar.gz.sha1`` in the same directory. If the
  hashfile is not present we compute the hash and create the hashfile for
  further usage.

  The resulting ``tarball`` variable content will be
  ``~/avocado/cache/stress-1.0.4.tar.gz``.
  An exception will take place if we fail to download or to verify the file.


Detailing the ``fetch_asset()`` attributes:

* ``name:`` The name used to name the fetched file. It can also contains a full
  URL, that will be used as the first location to try (after serching into the
  cache directories).
* ``asset_hash:`` (optional) The expected file hash. If missing, we skip the
  check. If provided, before computing the hash, we look for a hashfile to
  verify the asset. If the hashfile is nor present, we compute the hash and
  create the hashfile in the same cache directory for further usage.
* ``algorithm:`` (optional) Provided hash algorithm format. Defaults to sha1.
* ``locations:`` (optional) List of locations that will be used to try to fetch
  the file from. The supported schemes are ``http://``, ``https://``,
  ``ftp://`` and ``file://``. You're required to inform the full url to the
  file, including the file name. The first success will skip the next
  locations. Notice that for ``file://`` we just create a symbolic link in the
  cache directory, pointing to the file original location.
* ``expire:`` (optional) time period that the cached file will be considered
  valid. After that period, the file will be dowloaded again. The value can
  be an integer or a string containing the time and the unit. Example: '10d'
  (ten days). Valid units are ``s`` (second), ``m`` (minute), ``h`` (hour) and
  ``d`` (day).

The expected ``return`` is the asset file path or an exception.

Test Output Check and Output Record Mode
========================================

In a lot of occasions, you want to go simpler: just check if the output of a
given test matches an expected output.  In order to help with this common
use case, Avocado provides the ``--output-check-record`` option:

.. code-block:: none

  --output-check-record {none,stdout,stderr,both,combined,all}
                        Record the output produced by each test (from stdout
                        and stderr) into both the current executing result and
                        into reference files. Reference files are used on
                        subsequent runs to determine if the test produced the
                        expected output or not, and the current executing
                        result is used to check against a previously recorded
                        reference file. Valid values: 'none' (to explicitly
                        disable all recording) 'stdout' (to record standard
                        output *only*), 'stderr' (to record standard error
                        *only*), 'both' (to record standard output and error
                        in separate files), 'combined' (for standard output
                        and error in a single file). 'all' is also a valid but
                        deprecated option that is a synonym of 'both'. This
                        option does not have a default value, but the Avocado
                        test runner will record the test under execution in
                        the most suitable way unless it's explicitly disabled
                        with value 'none'

If this option is used, Avocado will store the content generated by
the test in the standard (POSIX) streams, that is, ``STDOUT`` and
``STDERR``.  Depending on the option chosen, you may end up with different
files recorded (into what we call "reference files"):

 * ``stdout`` will produce a file named ``stdout.expected`` with the
   contents from the test process standard output stream (file
   descriptor 1)
 * ``stderr`` will produce a file named ``stderr.expected`` with the
   contents from the test process standard error stream (file
   descriptor 2)
 * ``both`` will produce both a file named ``stdout.expected`` and a
   file named ``stderr.expected``
 * ``combined``: will produce a single file named ``output.expected``,
   with the content from both test process standard output and error
   streams (file descriptors 1 and 2)
 * ``none`` will explicitly disable all recording of test generated
   output and the generation reference files with that content

The reference files will be recorded in the first (most specific)
test's data dir (:ref:`accessing-test-data-files`). Let's take as an
example the test ``synctest.py``.  In a fresh checkout of the Avocado
source code you can find the following reference files::

  examples/tests/synctest.py.data/stderr.expected
  examples/tests/synctest.py.data/stdout.expected

From those 2 files, only stdout.expected has some content::

  $ cat examples/tests/synctest.py.data/stdout.expected
  PAR : waiting
  PASS : sync interrupted

This means that during a previous test execution, output was recorded
with option ``--output-check-record both`` and content was generated
on the ``STDOUT`` stream only::

  $ avocado run --output-check-record both synctest.py
  JOB ID     : b6306504351b037fa304885c0baa923710f34f4a
  JOB LOG    : $JOB_RESULTS_DIR/job-2017-11-26T16.42-b630650/job.log
   (1/1) examples/tests/synctest.py:SyncTest.test: PASS (2.03 s)
  RESULTS    : PASS 1 | ERROR 0 | FAIL 0 | SKIP 0 | WARN 0 | INTERRUPT 0 | CANCEL 0
  JOB TIME   : 2.26 s

After the reference files are added, the check process is transparent,
in the sense that you do not need to provide special flags to the test
runner.  From this point on, after such as test (one with a reference
file recorded) has finished running, Avocado will check if the output
generated match the reference(s) file(s) content.  If they don't
match, the test will finish with a ``FAIL`` status.

You can disable this automatic check when a reference file exists by
passing ``--output-check=off`` to the test runner.

.. tip:: The :mod:`avocado.utils.process` APIs have a parameter called
         ``allow_output_check`` that let you individually select the
         output that will be part of the test output and recorded
         reference files.  Some other APIs built on top of
         :mod:`avocado.utils.process`, such as the ones in
         :mod:`avocado.utils.build` also provide the same parameter.

This process works fine also with simple tests, which are programs or shell scripts
that returns 0 (PASSed) or != 0 (FAILed). Let's consider our bogus example::

    $ cat output_record.sh
    #!/bin/bash
    echo "Hello, world!"

Let's record the output for this one::

    $ scripts/avocado run output_record.sh --output-check-record all
    JOB ID    : 25c4244dda71d0570b7f849319cd71fe1722be8b
    JOB LOG   : $HOME/avocado/job-results/job-2014-09-25T20.49-25c4244/job.log
     (1/1) output_record.sh: PASS (0.01 s)
    RESULTS    : PASS 1 | ERROR 0 | FAIL 0 | SKIP 0 | WARN 0 | INTERRUPT 0
    JOB TIME   : 0.11 s

After this is done, you'll notice that a the test data directory
appeared in the same level of our shell script, containing 2 files::

    $ ls output_record.sh.data/
    stderr.expected  stdout.expected

Let's look what's in each of them::

    $ cat output_record.sh.data/stdout.expected
    Hello, world!
    $ cat output_record.sh.data/stderr.expected
    $

Now, every time this test runs, it'll take into account the expected files that
were recorded, no need to do anything else but run the test. Let's see what
happens if we change the ``stdout.expected`` file contents to ``Hello, Avocado!``::

    $ scripts/avocado run output_record.sh
    JOB ID    : f0521e524face93019d7cb99c5765aedd933cb2e
    JOB LOG   : $HOME/avocado/job-results/job-2014-09-25T20.52-f0521e5/job.log
     (1/1) output_record.sh: FAIL (0.02 s)
    RESULTS    : PASS 0 | ERROR 0 | FAIL 1 | SKIP 0 | WARN 0 | INTERRUPT 0
    JOB TIME   : 0.12 s

Verifying the failure reason::

    $ cat $HOME/avocado/job-results/latest/job.log
	2017-10-16 14:23:02,567 test             L0381 INFO | START 1-output_record.sh
	2017-10-16 14:23:02,568 test             L0402 DEBUG| Test metadata:
	2017-10-16 14:23:02,568 test             L0403 DEBUG|   filename: $HOME/output_record.sh
	2017-10-16 14:23:02,596 process          L0389 INFO | Running '$HOME/output_record.sh'
	2017-10-16 14:23:02,603 process          L0499 INFO | Command '$HOME/output_record.sh' finished with 0 after 0.00131011009216s
	2017-10-16 14:23:02,602 process          L0479 DEBUG| [stdout] Hello, world!
	2017-10-16 14:23:02,603 test             L1084 INFO | Exit status: 0
	2017-10-16 14:23:02,604 test             L1085 INFO | Duration: 0.00131011009216
	2017-10-16 14:23:02,604 test             L0274 DEBUG| DATA (filename=stdout.expected) => $HOME/output_record.sh.data/stdout.expected (found at file source dir)
	2017-10-16 14:23:02,605 test             L0740 DEBUG| Stdout Diff:
	2017-10-16 14:23:02,605 test             L0742 DEBUG| --- $HOME/output_record.sh.data/stdout.expected
	2017-10-16 14:23:02,605 test             L0742 DEBUG| +++ $HOME/avocado/job-results/job-2017-10-16T14.23-8cba866/test-results/1-output_record.sh/stdout
	2017-10-16 14:23:02,605 test             L0742 DEBUG| @@ -1 +1 @@
	2017-10-16 14:23:02,605 test             L0742 DEBUG| -Hello, Avocado!
	2017-10-16 14:23:02,605 test             L0742 DEBUG| +Hello, world!
	2017-10-16 14:23:02,606 stacktrace       L0041 ERROR|
	2017-10-16 14:23:02,606 stacktrace       L0044 ERROR| Reproduced traceback from: $HOME/git/avocado/avocado/core/test.py:872
	2017-10-16 14:23:02,606 stacktrace       L0047 ERROR| Traceback (most recent call last):
	2017-10-16 14:23:02,606 stacktrace       L0047 ERROR|   File "$HOME/git/avocado/avocado/core/test.py", line 743, in _check_reference_stdout
	2017-10-16 14:23:02,606 stacktrace       L0047 ERROR|     self.fail('Actual test sdtout differs from expected one')
	2017-10-16 14:23:02,606 stacktrace       L0047 ERROR|   File "$HOME//git/avocado/avocado/core/test.py", line 983, in fail
	2017-10-16 14:23:02,607 stacktrace       L0047 ERROR|     raise exceptions.TestFail(message)
	2017-10-16 14:23:02,607 stacktrace       L0047 ERROR| TestFail: Actual test sdtout differs from expected one
	2017-10-16 14:23:02,607 stacktrace       L0048 ERROR|
	2017-10-16 14:23:02,607 test             L0274 DEBUG| DATA (filename=stderr.expected) => $HOME//output_record.sh.data/stderr.expected (found at file source dir)
	2017-10-16 14:23:02,608 test             L0965 ERROR| FAIL 1-output_record.sh -> TestFail: Actual test sdtout differs from expected one


As expected, the test failed because we changed its expectations, so an
unified diff was logged. The unified diffs are also present in the files
`stdout.diff` and `stderr.diff`, present in the test results directory::

	$ cat $HOME/avocado/job-results/latest/test-results/1-output_record.sh/stdout.diff
	--- $HOME/output_record.sh.data/stdout.expected
	+++ $HOME/avocado/job-results/job-2017-10-16T14.23-8cba866/test-results/1-output_record.sh/stdout
	@@ -1 +1 @@
	-Hello, Avocado!
	+Hello, world!

Test log, stdout and stderr in native Avocado modules
=====================================================

If needed, you can write directly to the expected stdout and stderr files
from the native test scope. It is important to make the distinction between
the following entities:

* The test logs
* The test expected stdout
* The test expected stderr

The first one is used for debugging and informational purposes. Additionally
writing to `self.log.warning` causes test to be marked as dirty and when
everything else goes well the test ends with WARN. This means that the test
passed but there were non-related unexpected situations described in warning
log.

You may log something into the test logs using the methods in
:mod:`avocado.Test.log` class attributes. Consider the example::

    class output_test(Test):

        def test(self):
            self.log.info('This goes to the log and it is only informational')
            self.log.warn('Oh, something unexpected, non-critical happened, '
                          'but we can continue.')
            self.log.error('Describe the error here and don't forget to raise '
                           'an exception yourself. Writing to self.log.error '
                           'won't do that for you.')
            self.log.debug('Everybody look, I had a good lunch today...')

If you need to write directly to the test stdout and stderr streams,
Avocado makes two preconfigured loggers available for that purpose,
named ``avocado.test.stdout`` and ``avocado.test.stderr``. You can use
Python's standard logging API to write to them. Example::

    import logging

    class output_test(Test):

        def test(self):
            stdout = logging.getLogger('avocado.test.stdout')
            stdout.info('Informational line that will go to stdout')
            ...
            stderr = logging.getLogger('avocado.test.stderr')
            stderr.info('Informational line that will go to stderr')

Avocado will automatically save anything a test generates on STDOUT
into a ``stdout`` file, to be found at the test results directory. The same
applies to anything a test generates on STDERR, that is, it will be saved
into a ``stderr`` file at the same location.

Additionally, when using the runner's output recording features,
namely the ``--output-check-record`` argument with values ``stdout``,
``stderr`` or ``all``, everything given to those loggers will be saved
to the files ``stdout.expected`` and ``stderr.expected`` at the test's
data directory (which is different from the job/test results directory).

Setting a Test Timeout
======================

Sometimes your test suite/test might get stuck forever, and this might
impact your test grid. You can account for that possibility and set up a
``timeout`` parameter for your test. The test timeout can be set through
the test parameters, as shown below.

::

    sleep_length: 5
    timeout: 3


::

    $ avocado run sleeptest.py --mux-yaml /tmp/sleeptest-example.yaml
    JOB ID     : c78464bde9072a0b5601157989a99f0ba32a288e
    JOB LOG    : $HOME/avocado/job-results/job-2016-11-02T11.13-c78464b/job.log
     (1/1) sleeptest.py:SleepTest.test: INTERRUPTED (3.04 s)
    RESULTS    : PASS 0 | ERROR 0 | FAIL 0 | SKIP 0 | WARN 0 | INTERRUPT 1
    JOB TIME   : 3.14 s
    JOB HTML   : $HOME/avocado/job-results/job-2016-11-02T11.13-c78464b/html/results.html


::

	$ cat $HOME/avocado/job-results/job-2016-11-02T11.13-c78464b/job.log
	2016-11-02 11:13:01,133 job              L0384 INFO | Multiplex tree representation:
	2016-11-02 11:13:01,133 job              L0386 INFO |  \-- run
	2016-11-02 11:13:01,133 job              L0386 INFO |         -> sleep_length: 5
	2016-11-02 11:13:01,133 job              L0386 INFO |         -> timeout: 3
	2016-11-02 11:13:01,133 job              L0387 INFO | 
	2016-11-02 11:13:01,134 job              L0391 INFO | Temporary dir: /var/tmp/avocado_PqDEyC
	2016-11-02 11:13:01,134 job              L0392 INFO | 
	2016-11-02 11:13:01,134 job              L0399 INFO | Variant 1:    /run
	2016-11-02 11:13:01,134 job              L0402 INFO | 
	2016-11-02 11:13:01,134 job              L0311 INFO | Job ID: c78464bde9072a0b5601157989a99f0ba32a288e
	2016-11-02 11:13:01,134 job              L0314 INFO | 
	2016-11-02 11:13:01,345 sysinfo          L0107 DEBUG| Not logging /proc/pci (file does not exist)
	2016-11-02 11:13:01,351 sysinfo          L0105 DEBUG| Not logging /proc/slabinfo (lack of permissions)
	2016-11-02 11:13:01,355 sysinfo          L0107 DEBUG| Not logging /sys/kernel/debug/sched_features (file does not exist)
	2016-11-02 11:13:01,388 sysinfo          L0388 INFO | Commands configured by file: /etc/avocado/sysinfo/commands
	2016-11-02 11:13:01,388 sysinfo          L0399 INFO | Files configured by file: /etc/avocado/sysinfo/files
	2016-11-02 11:13:01,388 sysinfo          L0419 INFO | Profilers configured by file: /etc/avocado/sysinfo/profilers
	2016-11-02 11:13:01,388 sysinfo          L0427 INFO | Profiler disabled
	2016-11-02 11:13:01,394 multiplexer      L0166 DEBUG| PARAMS (key=timeout, path=*, default=None) => 3
	2016-11-02 11:13:01,395 test             L0216 INFO | START 1-sleeptest.py:SleepTest.test
	2016-11-02 11:13:01,396 multiplexer      L0166 DEBUG| PARAMS (key=sleep_length, path=*, default=1) => 5
	2016-11-02 11:13:01,396 sleeptest        L0022 DEBUG| Sleeping for 5.00 seconds
	2016-11-02 11:13:04,411 stacktrace       L0038 ERROR| 
	2016-11-02 11:13:04,412 stacktrace       L0041 ERROR| Reproduced traceback from: $HOME/src/avocado/avocado/core/test.py:454
	2016-11-02 11:13:04,412 stacktrace       L0044 ERROR| Traceback (most recent call last):
	2016-11-02 11:13:04,413 stacktrace       L0044 ERROR|   File "/usr/share/avocado/tests/sleeptest.py", line 23, in test
	2016-11-02 11:13:04,413 stacktrace       L0044 ERROR|     time.sleep(sleep_length)
	2016-11-02 11:13:04,413 stacktrace       L0044 ERROR|   File "$HOME/src/avocado/avocado/core/runner.py", line 293, in sigterm_handler
	2016-11-02 11:13:04,413 stacktrace       L0044 ERROR|     raise SystemExit("Test interrupted by SIGTERM")
	2016-11-02 11:13:04,414 stacktrace       L0044 ERROR| SystemExit: Test interrupted by SIGTERM
	2016-11-02 11:13:04,414 stacktrace       L0045 ERROR| 
	2016-11-02 11:13:04,414 test             L0459 DEBUG| Local variables:
	2016-11-02 11:13:04,440 test             L0462 DEBUG|  -> self <class 'sleeptest.SleepTest'>: 1-sleeptest.py:SleepTest.test
	2016-11-02 11:13:04,440 test             L0462 DEBUG|  -> sleep_length <type 'int'>: 5
	2016-11-02 11:13:04,440 test             L0592 ERROR| ERROR 1-sleeptest.py:SleepTest.test -> TestError: SystemExit('Test interrupted by SIGTERM',): Test interrupted by SIGTERM


The YAML file defines a test parameter ``timeout`` which overrides
the default test timeout before the runner ends the test forcefully by
sending a class:`signal.SIGTERM` to the test, making it raise a
:class:`avocado.core.exceptions.TestTimeoutError`.


Skipping Tests
==============

To skip tests is in Avocado, you must use one of the Avocado skip
decorators:

- ``@avocado.skip(reason)``: Skips a test.
- ``@avocado.skipIf(condition, reason)``: Skips a test if the condition is
  ``True``.
- ``@avocado.skipUnless(condition, reason)``: Skips a test if the condition is
  ``False``

Those decorators can be used with both ``setUp()`` method and/or and in the
``test*()`` methods. The test below::

    import avocado

    class MyTest(avocado.Test):

        @avocado.skipIf(1 == 1, 'Skipping on True condition.')
        def test1(self):
            pass

        @avocado.skip("Don't want this test now.")
        def test2(self):
            pass

        @avocado.skipUnless(1 == 1, 'Skipping on False condition.')
        def test3(self):
            pass

Will produce the following result::

    $ avocado run  test_skip_decorators.py
    JOB ID     : 59c815f6a42269daeaf1e5b93e52269fb8a78119
    JOB LOG    : $HOME/avocado/job-results/job-2017-02-03T17.41-59c815f/job.log
     (1/3) test_skip_decorators.py:MyTest.test1: SKIP
     (2/3) test_skip_decorators.py:MyTest.test2: SKIP
     (3/3) test_skip_decorators.py:MyTest.test3: PASS (0.02 s)
    RESULTS    : PASS 1 | ERROR 0 | FAIL 0 | SKIP 2 | WARN 0 | INTERRUPT 0
    JOB TIME   : 0.13 s
    JOB HTML   : $HOME/avocado/job-results/job-2017-02-03T17.41-59c815f/html/results.html

Notice the ``test3`` was not skipped because the provided condition was
not ``False``.

Using the skip decorators, nothing is actually executed. We will skip
the  `setUp()` method, the test method and the `tearDown()` method.

.. note:: It's an erroneous condition, reported with test status
          ``ERROR``, to use any of the skip decorators on the
          ``tearDown()`` method.

Cancelling Tests
================

You can cancel a test calling `self.cancel()` at any phase of the test
(`setUp()`, test method or `tearDown()`). Test will finish with `CANCEL`
status and will not make the Job to exit with a non-0 status. Example::



    #!/usr/bin/env python

    from avocado import Test
    from avocado import main

    from avocado.utils.process import run
    from avocado.utils.software_manager import SoftwareManager


    class CancelTest(Test):

        """
        Example tests that cancel the current test from inside the test.
        """

        def setUp(self):
            sm = SoftwareManager()
            self.pkgs = sm.list_all(software_components=False)

        def test_iperf(self):
            if 'iperf-2.0.8-6.fc25.x86_64' not in self.pkgs:
                self.cancel('iperf is not installed or wrong version')
            self.assertIn('pthreads',
                          run('iperf -v', ignore_status=True).stderr)

        def test_gcc(self):
            if 'gcc-6.3.1-1.fc25.x86_64' not in self.pkgs:
                self.cancel('gcc is not installed or wrong version')
            self.assertIn('enable-gnu-indirect-function',
                          run('gcc -v', ignore_status=True).stderr)

    if __name__ == "__main__":
        main()

In a system missing the `iperf` package but with `gcc` installed in
the correct version, the result will be::

    JOB ID     : 39c1f120830b9769b42f5f70b6b7bad0b1b1f09f
    JOB LOG    : $HOME/avocado/job-results/job-2017-03-10T16.22-39c1f12/job.log
     (1/2) /home/apahim/avocado/tests/test_cancel.py:CancelTest.test_iperf: CANCEL (1.15 s)
     (2/2) /home/apahim/avocado/tests/test_cancel.py:CancelTest.test_gcc: PASS (1.13 s)
    RESULTS    : PASS 1 | ERROR 0 | FAIL 0 | SKIP 0 | WARN 0 | INTERRUPT 0 | CANCEL 1
    JOB TIME   : 2.38 s
    JOB HTML   : $HOME/avocado/job-results/job-2017-03-10T16.22-39c1f12/html/results.html

Notice that using the ``self.cancel()`` will cancel the rest of the test
from that point on, but the ``tearDown()`` will still be executed.

Depending on the result format you're referring to, the ``CANCEL`` status
is mapped to a corresponding valid status in that format. See the table
below:

+--------+----------------------+
| Format | Corresponding Status |
+========+======================+
| json   | cancel               |
+--------+----------------------+
| xunit  | skipped              |
+--------+----------------------+
| tap    | ok                   |
+--------+----------------------+
| html   | CANCEL (warning)     |
+--------+----------------------+

Docstring Directives
====================

Some Avocado features, usually only available to instrumented tests,
depend on setting directives on the test's class docstring.  A
docstring directive is composed of a marker (a literal ``:avocado:`` string),
followed by the custom content itself, such as ``:avocado: directive``.

This is similar to docstring directives such as ``:param my_param:
description`` and shouldn't be a surprise to most Python developers.

The reason Avocado uses those docstring directives (instead of real
Python code) is that the inspection done while looking for tests does
not involve any execution of code.

For a detailed explanation about what makes a docstring format valid
or not, please refer to our section on :ref:`docstring-directive-rules`.

Now let's follow with some docstring directives examples.

.. _docstring-directive-enable-disable:

Explicitly enabling or disabling tests
--------------------------------------

If your test is a method in a class that directly inherits from
:class:`avocado.Test`, then Avocado will find it as one would expect.

Now, the need may arise for more complex tests, to use more advanced
Python features such as inheritance.  For those tests that are written
in a class not directly inherting from :class:`avocado.Test`, Avocado
may need your help, because Avocado uses only static analysis to examine
the files.

For example, suppose that you define a new test class that inherits
from the Avocado base test class, that is, :class:`avocado.Test`, and
put it in ``mylibrary.py``::

    from avocado import Test


    class MyOwnDerivedTest(Test):
        def __init__(self, methodName='test', name=None, params=None,
                     base_logdir=None, job=None, runner_queue=None):
            super(MyOwnDerivedTest, self).__init__(methodName, name, params,
                                                   base_logdir, job,
                                                   runner_queue)
            self.log('Derived class example')


Then you implement your actual test using that derived class, in
``mytest.py``::

    import mylibrary


    class MyTest(mylibrary.MyOwnDerivedTest):

        def test1(self):
            self.log('Testing something important')

        def test2(self):
            self.log('Testing something even more important')


If you try to list the tests in that file, this is what you'll get:

.. code-block:: none

    scripts/avocado list mytest.py -V
    Type       Test      Tag(s)
    NOT_A_TEST mytest.py

    TEST TYPES SUMMARY
    ==================
    ACCESS_DENIED: 0
    BROKEN_SYMLINK: 0
    EXTERNAL: 0
    FILTERED: 0
    INSTRUMENTED: 0
    MISSING: 0
    NOT_A_TEST: 1
    SIMPLE: 0
    VT: 0

You need to give avocado a little help by adding a docstring
directive. That docstring directive is ``:avocado: enable``. It tells
the Avocado safe test detection code to consider it as an avocado
test, regardless of what the (admittedly simple) detection code thinks
of it. Let's see how that works out. Add the docstring, as you can see
the example below::

    import mylibrary


    class MyTest(mylibrary.MyOwnDerivedTest):
        """
        :avocado: enable
        """
        def test1(self):
            self.log('Testing something important')

        def test2(self):
            self.log('Testing something even more important')


Now, trying to list the tests on the ``mytest.py`` file again:

.. code-block:: none

    scripts/avocado list mytest.py -V
    Type         Test                   Tag(s)
    INSTRUMENTED mytest.py:MyTest.test1
    INSTRUMENTED mytest.py:MyTest.test2

    TEST TYPES SUMMARY
    ==================
    ACCESS_DENIED: 0
    BROKEN_SYMLINK: 0
    EXTERNAL: 0
    FILTERED: 0
    INSTRUMENTED: 2
    MISSING: 0
    NOT_A_TEST: 0
    SIMPLE: 0
    VT: 0

You can also use the ``:avocado: disable`` docstring directive, that
works the opposite way: something that would be considered an Avocado
test, but we force it to not be listed as one.

The docstring ``:avocado: disable`` is evaluated first by Avocado,
meaning that if both ``:avocado: disable`` and ``:avocado: enable`` are
present in the same docstring, the test will not be listed.

.. _docstring-directive-recursive:

Recursively Discovering Tests
-----------------------------

In addition to the ``:avocado: enable`` and ``:avocado: disable``
docstring directives, Avocado has support for the ``:avocado: recursive``
directive. It is intended to be used in inherited classes when you want
to tell Avocado to also discover the ancestor classes.

The ``:avocado: recursive`` directive will direct Avocado to evaluate all
the ancestors of the class until the base class, the one derived from
from ``avocado.Test``.

Example:

File `/usr/share/avocado/tests/test_base_class.py`::

    from avocado import Test


    class BaseClass(Test):

        def test_basic(self):
            pass


File `/usr/share/avocado/tests/test_first_child.py`::

    from test_base_class import BaseClass


    class FirstChild(BaseClass):

        def test_first_child(self):
            pass


File `/usr/share/avocado/tests/test_second_child.py`::

    from test_first_child import FirstChild


    class SecondChild(FirstChild):
        """
        :avocado: recursive
        """

        def test_second_child(self):
            pass

Using only `test_second_child.py` as a test reference will result in::

    $ avocado list test_second_child.py
    INSTRUMENTED test_second_child.py:SecondChild.test_second_child
    INSTRUMENTED test_second_child.py:SecondChild.test_first_child
    INSTRUMENTED test_second_child.py:SecondChild.test_basic

Notice that the ``:avocado: disable`` docstring will be ignored in
ancestors during the recursive discovery. What means that even if an
ancestor contains the docstring ``:avocado: disable``, that ancestor will
still be included in the results.

.. _categorizing-tests:

Categorizing tests
------------------

Avocado allows tests to be given tags, which can be used to create
test categories.  With tags set, users can select a subset of the
tests found by the test resolver (also known as test loader).

To make this feature easier to grasp, let's work with an example: a
single Python source code file, named ``perf.py``, that contains both
disk and network performance tests::

  from avocado import Test

  class Disk(Test):

      """
      Disk performance tests

      :avocado: tags=disk,slow,superuser,unsafe
      """

      def test_device(self):
          device = self.params.get('device', default='/dev/vdb')
          self.whiteboard = measure_write_to_disk(device)


  class Network(Test):

      """
      Network performance tests

      :avocado: tags=net,fast,safe
      """

      def test_latency(self):
          self.whiteboard = measure_latency()

      def test_throughput(self):
          self.whiteboard = measure_throughput()


  class Idle(Test):

      """
      Idle tests
      """

      def test_idle(self):
          self.whiteboard = "test achieved nothing"


.. warning:: All docstring directives in Avocado require a strict
             format, that is, ``:avocado:`` followed by one or
             more spaces, and then followed by a single value **with no
             white spaces in between**.  This means that an attempt to
             write a docstring directive like ``:avocado: tags=foo,
             bar`` will be interpreted as ``:avocado: tags=foo,``.


Usually, listing and executing tests with the Avocado test runner
would reveal all three tests::

  $ avocado list perf.py
  INSTRUMENTED perf.py:Disk.test_device
  INSTRUMENTED perf.py:Network.test_latency
  INSTRUMENTED perf.py:Network.test_throughput
  INSTRUMENTED perf.py:Idle.test_idle

If you want to list or run only the network based tests, you can do so
by requesting only tests that are tagged with ``net``::

  $ avocado list perf.py --filter-by-tags=net
  INSTRUMENTED perf.py:Network.test_latency
  INSTRUMENTED perf.py:Network.test_throughput

Now, suppose you're not in an environment where you're confortable
running a test that will write to your raw disk devices (such as your
development workstation).  You know that some tests are tagged
with ``safe`` while others are tagged with ``unsafe``.  To only
select the "safe" tests you can run::

  $ avocado list perf.py --filter-by-tags=safe
  INSTRUMENTED perf.py:Network.test_latency
  INSTRUMENTED perf.py:Network.test_throughput

But you could also say that you do **not** want the "unsafe" tests
(note the *minus* sign before the tag)::

  $ avocado list perf.py --filter-by-tags=-unsafe
  INSTRUMENTED perf.py:Network.test_latency
  INSTRUMENTED perf.py:Network.test_throughput


.. tip:: The ``-`` sign may cause issues with some shells.  One know
   error condition is to use spaces between ``--filter-by-tags`` and
   the negated tag, that is, ``--filter-by-tags -unsafe`` will most
   likely not work.  To be on the safe side, use
   ``--filter-by-tags=-tag``.


If you require tests to be tagged with **multiple** tags, just add
them separate by commas.  Example::

  $ avocado list perf.py --filter-by-tags=disk,slow,superuser,unsafe
  INSTRUMENTED perf.py:Disk.test_device

If no test contains all tags given on a single `--filter-by-tags`
parameter, no test will be included::

  $ avocado list perf.py --filter-by-tags=disk,slow,superuser,safe | wc -l
  0

.. _categorizing-tests-tags-on-methods:

Test tags can be applied to test classes and to test methods. Tags are
evaluated per method, meaning that the class tags will be inherited by
all methods, being merged with method local tags. Example::

    from avocado import Test

    class MyClass(Test):
        """
        :avocado: tags=furious
        """

        def test1(self):
            """
            :avocado: tags=fast
            """
            pass

        def test2(self):
            """
            :avocado: tags=slow
            """
            pass

If you use the tag ``furious``, all tests will be included::

    $ avocado list furious_tests.py --filter-by-tags=furious
    INSTRUMENTED test_tags.py:MyClass.test1
    INSTRUMENTED test_tags.py:MyClass.test2

But using ``fast`` and ``furious`` will include only ``test1``::

    $ avocado list furious_tests.py --filter-by-tags=fast,furious
    INSTRUMENTED test_tags.py:MyClass.test1

Multiple `--filter-by-tags`
~~~~~~~~~~~~~~~~~~~~~~~~~~~

While multiple tags in a single option will require tests with all the
given tags (effectively a logical AND operation), it's also possible
to use multiple ``--filter-by-tags`` (effectively a logical OR
operation).

For instance To include all tests that have the ``disk`` tag and all
tests that have the ``net`` tag, you can run::

  $ avocado list perf.py --filter-by-tags=disk --filter-by-tags=net
  INSTRUMENTED perf.py:Disk.test_device
  INSTRUMENTED perf.py:Network.test_latency
  INSTRUMENTED perf.py:Network.test_throughput

Including tests without tags
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The normal behavior when using ``--filter-by-tags`` is to require the
given tags on all tests.  In some situations, though, it may be
desirable to include tests that have no tags set.

For instance, you may want to include tests of certain types that do
not have support for tags (such as SIMPLE tests) or tests that have
not (yet) received tags.  Consider this command::

  $ avocado list perf.py /bin/true --filter-by-tags=disk
  INSTRUMENTED perf.py:Disk.test_device

Since it requires the ``disk`` tag, only one test was returned.  By
using the ``--filter-by-tags-include-empty`` option, you can force the
inclusion of tests without tags::

  $ avocado list perf.py /bin/true --filter-by-tags=disk --filter-by-tags-include-empty
  SIMPLE       /bin/true
  INSTRUMENTED perf.py:Idle.test_idle
  INSTRUMENTED perf.py:Disk.test_device

Python :mod:`unittest` Compatibility Limitations And Caveats
============================================================

When executing tests, Avocado uses different techniques than most
other Python unittest runners.  This brings some compatibility
limitations that Avocado users should be aware.

Execution Model
---------------

One of the main differences is a consequence of the Avocado design
decision that tests should be self contained and isolated from other
tests.  Additionally, the Avocado test runner runs each test in a
separate process.

If you have a unittest class with many test methods and run them
using most test runners, you'll find that all test methods run under
the same process.  To check that behavior you could add to your
:meth:`setUp <unittest.TestCase.setUp>` method::

   def setUp(self):
       print("PID: %s", os.getpid())

If you run the same test under Avocado, you'll find that each test
is run on a separate process.

Class Level :meth:`setUp <unittest.TestCase.setUpClass>` and :meth:`tearDown <unittest.TestCase.tearDownClass>`
---------------------------------------------------------------------------------------------------------------

Because of Avocado's test execution model (each test is run on a
separate process), it doesn't make sense to support unittest's
:meth:`unittest.TestCase.setUpClass` and
:meth:`unittest.TestCase.tearDownClass`.  Test classes are freshly
instantiated for each test, so it's pointless to run code in those
methods, since they're supposed to keep class state between tests.

The ``setUp`` method is the only place in avocado where you are allowed to
call the ``skip`` method, given that, if a test started to be executed, by
definition it can't be skipped anymore. Avocado will do its best to enforce
this boundary, so that if you use ``skip`` outside ``setUp``, the test upon
execution will be marked with the ``ERROR`` status, and the error message
will instruct you to fix your test's code.

If you require a common setup to a number of tests, the current
recommended approach is to to write regular :meth:`setUp
<unittest.TestCase.setUp>` and :meth:`tearDown
<unittest.TestCase.tearDown>` code that checks if a given state was
already set.  One example for such a test that requires a binary
installed by a package::

  from avocado import Test

  from avocado.utils import software_manager
  from avocado.utils import path as utils_path
  from avocado.utils import process


  class BinSleep(Test):

      """
      Sleeps using the /bin/sleep binary
      """
      def setUp(self):
          self.sleep = None
          try:
              self.sleep = utils_path.find_command('sleep')
          except utils_path.CmdNotFoundError:
              software_manager.install_distro_packages({'fedora': ['coreutils']})
              self.sleep = utils_path.find_command('sleep')

      def test(self):
          process.run("%s 1" % self.sleep)

If your test setup is some kind of action that will last accross
processes, like the installation of a software package given in the
previous example, you're pretty much covered here.

If you need to keep other type of data a class across test
executions, you'll have to resort to saving and restoring the data
from an outside source (say a "pickle" file).  Finding and using a
reliable and safe location for saving such data is currently not in
the Avocado supported use cases.

.. _environment-variables-for-tests:

Environment Variables for Tests
===============================

Avocado exports some information, including test parameters, as environment
variables to the running test.

While these variables are available to all tests, they are usually
more interesting to SIMPLE tests.  The reason is that SIMPLE tests can
not make direct use of Avocado API.  INSTRUMENTED tests will usually
have more powerful ways, to access the same information.

Here is a list of the variables that Avocado currently exports to
tests:

+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| Environemnt Variable        | Meaning                               | Example                                                                                             |
+=============================+=======================================+=====================================================================================================+
| AVOCADO_VERSION             | Version of Avocado test runner        | 0.12.0                                                                                              |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| AVOCADO_TEST_BASEDIR        | Base directory of Avocado tests       | $HOME/Downloads/avocado-source/avocado                                                              |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| AVOCADO_TEST_DATADIR        | Data directory for the test           | $AVOCADO_TEST_BASEDIR/my_test.sh.data                                                               |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| AVOCADO_TEST_WORKDIR        | Work directory for the test           | /var/tmp/avocado_Bjr_rd/my_test.sh                                                                  |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| AVOCADO_TESTS_COMMON_TMPDIR | Temporary directory created by the    | /var/tmp/avocado_XhEdo/                                                                             |
|                             | `teststmpdir` plugin. The directory   |                                                                                                     |
|                             | is persistent throughout the tests    |                                                                                                     |
|                             | in the same Job                       |                                                                                                     |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| AVOCADO_TEST_LOGDIR         | Log directory for the test            | $HOME/logs/job-results/job-2014-09-16T14.38-ac332e6/test-results/$HOME/my_test.sh.1                 |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| AVOCADO_TEST_LOGFILE        | Log file for the test                 | $HOME/logs/job-results/job-2014-09-16T14.38-ac332e6/test-results/$HOME/my_test.sh.1/debug.log       |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| AVOCADO_TEST_OUTPUTDIR      | Output directory for the test         | $HOME/logs/job-results/job-2014-09-16T14.38-ac332e6/test-results/$HOME/my_test.sh.1/data            |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| AVOCADO_TEST_SYSINFODIR     | The system information directory      | $HOME/logs/job-results/job-2014-09-16T14.38-ac332e6/test-results/$HOME/my_test.sh.1/sysinfo         |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| `***`                       | All variables from --mux-yaml         | TIMEOUT=60; IO_WORKERS=10; VM_BYTES=512M; ...                                                       |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+
| AVOCADO_TEST_SRCDIR         | Source directory for the test         | /var/tmp/avocado_Bjr_rd/my-test.sh/src                                                              |
+-----------------------------+---------------------------------------+-----------------------------------------------------------------------------------------------------+

.. warning:: ``AVOCADO_TEST_SRCDIR`` is deprecated and will be removed
             soon.  Please use ``AVOCADO_TEST_WORKDIR`` instead.


SIMPLE Tests BASH extensions
============================

SIMPLE tests written in shell can use a few Avocado utilities.  In your
shell code, check if the libraries are available with something like::

  AVOCADO_SHELL_EXTENSIONS_DIR=$(avocado exec-path 2>/dev/null)

And if available, injects that directory containing those utilities
into the PATH used by the shell, making those utilities readily
accessible::

  if [ $? == 0 ]; then
    PATH=$AVOCADO_SHELL_EXTENSIONS_DIR:$PATH
  fi

For a full list of utilities, take a look into at the directory return
by ``avocado exec-path`` (if any).  Also, the example test
``examples/tests/simplewarning.sh`` can serve as further inspiration.

.. tip:: These extensions may be available as a separate package.  For
         RPM packages, look for the ``bash`` sub-package.

.. _test_type_simple_status:

SIMPLE Tests Status
===================

With SIMPLE tests, Avocado checks the exit code of the test to determine
whether the test PASSed or FAILed.

If your test exits with exit code 0 but you still want to set a different test
status in some conditions, Avocado can search a given regular expression in
the test outputs and, based on that, set the status to WARN or SKIP.

To use that feature, you have to set the proper keys in the configuration
file. For instance, to set the test status to SKIP when the test outputs
a line like this: '11:08:24 Test Skipped'::

    [simpletests.output]
    skip_regex = ^\d\d:\d\d:\d\d Test Skipped$

That configuration will make avocado to search the
`Python Regular Expression <http://docs.python.org/2.7/howto/regex.html>`__
on  both stdout and stderr. If you want to limit the search for only one of
them, there's another key for that configuration, resulting in::

    [simpletests.output]
    skip_regex = ^\d\d:\d\d:\d\d Test Skipped$
    skip_location = stderr

The equivalent settings can be present for the WARN status. For instance,
if you want to set the test status to WARN when the test outputs a line
starting with string ``WARNING:``, the configuration file will look like this::

    [simpletests.output]
    skip_regex = ^\d\d:\d\d:\d\d Test Skipped$
    skip_location = stderr
    warn_regex = ^WARNING:
    warn_location = all

Wrap Up
=======

We recommend you take a look at the example tests present in the
``examples/tests`` directory, that contains a few samples to take some
inspiration from. That directory, besides containing examples, is also used by
the Avocado self test suite to do functional testing of Avocado itself.
Although one can inspire in `<https://github.com/avocado-framework-tests>`__
where people are allowed to share their basic system tests.

It is also recommended that you take a look at the :ref:`api-reference`.
for more possibilities.

.. [#f1] sleeptest is a functional test for Avocado. It's "old" because we
	 also have had such a test for `Autotest`_ for a long time.

.. _Autotest: http://autotest.github.io
.. _Class Names: https://www.python.org/dev/peps/pep-0008/
.. _PEP8 Function Names: https://www.python.org/dev/peps/pep-0008/#function-names
