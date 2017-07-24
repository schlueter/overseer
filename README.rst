========
Overseer
========
Make sure your jobs get executed.
---------------------------------

Features
--------
- Git backed. All jobs are defined in a repository which Overseer is configured to read its own job-related configuration from.
- Provides a library of common actions and behaviours with which to build jobs.
- Jobs may be defined in JSON or YAML and may execute any shell commands.
- Stderr and Stdout from each job are accessible via GETs to the api.
- Execution status may be reported via a webhook at the conclusion of the job run, or at an point during job execution.

Git configuration repository
----------------------------

The simplest git configuration repository (that does anything) would consist of a single job, though typically, one would also have a bin directory which would be utilized in the jobs. The bin directory of the configured repository will be added to the path of all jobs in the repo. The directory layout would look similar to:

::

    ├── bin
    │   └── example_command
    └── example_job.yml

API
---

`/jobs`

- GET                returns a list of all jobs

`/jobs/<job name>`

- GET                returns job definition and metadata
- POST               executes the job with provided parameters
- DELETE             remove job from list of active jobs
- PUT                add job to list of active jobs

`/jobs/<job name>/<execution id>`

- GET                returns metadata on the execution
- POST               repeat execution
- PUT                send a signal specified in the query string parameter to the job using `kill` (the default is whatever kill's default is on the executor, POSIX defines this as `SIGTERM`__)

`/jobs/<job name>/<execution id>/stderr`

- GET                returns stderr of execution

`/jobs/<job name>/<execution id>/stdout`

- GET                returns stdout of execution

Job Definition
--------------

Job definitions may either be in json or in yaml. The job will do nothing without a value specified in command or script, though it will "execute" and create an execution entry if scheduled or triggered by other means. The basic structure is:

.. code-block:: yaml

    ---
    - name: A job # the job endpoint would be /job/A%20job (so maybe avoid non ascii symbols)
      description: # an optional description
      cmdspec: my_command positional_var --some-opt some_opt
      env:
        SOME_ENV_VAR: SOME_VALUE
      user: nobody # an optional user to execute as
      group: nogroup # an optiona group to as

Execution Status
----------------
Execution status events will be created automatically at certain points during execution, those being: start, and either complete, fail, or time-out. Execution status may be set to any value by calling the `event` method provided to jobs during execution. This will trigger hooks to all endpoints configured for the set status. The contents of the post data for the hooks will be similar to:

.. code-block:: json

  {
      "job": "example-job",
      "execution": 1489895624.840211,
      "status": "complete",
      "metadata": "http://overseer-host/example-job/1489888092.0"
  }

Library
-------
The paths to the library programs will be prepended to the PATH variable available, behind only the configuration repository's bin directory.

:Author: Brandon Schlueter <overseer@schlueter.blue>
:Copyright: Brandon Schlueter 2017
:License: Affero General Public License v3 or newer

.. _POSIX_Kill: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/kill.html
__ _POSIX_Kill
