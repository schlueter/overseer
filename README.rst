========
Overseer
========
A job runner.
-------------

Features
--------
- Git backed. All jobs are defined in a repository which Overseer is configured to read its own job-related configuration from.
- Provides a library of common actions and behaviours with which to build jobs.
- Jobs may be defined in JSON, or YAML and may execute any shell commands.
- Jobs may be triggered via POSTs to the api.
- Jobs may be triggered via configured schedules.
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
- DELETE (Maybe?)    remove job from list of active jobs
- PUT    (Maybe?)    add job to list of active jobs

`/jobs/<job name>/<execution id>`

- GET                returns metadata on the execution
- POST               repeat execution
- PUT                create an execution event. If the event is one of "stop", \*"continue","terminate" or "kill" and the execution is currently executing (\* or stopped), the execution process will be sent the respective signal.

`/jobs/<job name>/<execution id>/stderr`

- GET                returns stderr of execution

`/jobs/<job name>/<execution id>/stdout`

- GET                returns stdout of execution

`/jobs/<job name>/<execution id>/output`

- GET                returns combined stdout and stderr of execution

`/jobs/<job name>/<execution id>/<execution event>`

- GET                returns metadata about the execution event

Job Definition
--------------

Job definitions may either be in json or in yaml. The job will do nothing without a value specified in command or script, though it will "exucute" and create an execution entry if scheduled or triggered by other means. The basic structure is:

.. code-block:: yaml

    ---
    - command: my_command {{ positional_var }} --some-opt {{ some_opt }}
      vars:
        - name: positional_var
          type: str
        - name: positional_var
          type: str
        - name: some_opt
          type: enum
          option: yes
          values: [ a, b, c ]
      schedule:
        - time:
          priority: '0'
          frequency: 3600
      ...

As no options are required, there are no system defaults.

+----------+-----------------------------------------------------------------------+
| Option   |   Description                                                         |
+==========+=======================================================================+
| command  | A command to be executed.*                                            |
+----------+-----------------------------------------------------------------------+
| script   | The path to a script relative to the scripts directory of the         |
|          | scripts directory of the configured git repository.*                  |
+----------+-----------------------------------------------------------------------+
| timeout  | The maximum amount of time in seconds which the job may execute for.  |
+----------+-----------------------------------------------------------------------+
| vars     | Variables which will be passed to the command or                      |
|          | script. Variable definitions consist of a name                        |
|          | and either a type or options. Valid types                             |
|          | include any python type. The value of type and                        |
|          | the value of any options may include values of                        |
|          | any of the standard yaml types or enum:                               |
|          |                                                                       |
|          | ===========  =======================================                  |
|          |  Type         Python Type                                             |
|          | ===========  =======================================                  |
|          |  null         None                                                    |
|          |  bool         bool                                                    |
|          |  int          int or long (int in Python 3)                           |
|          |  float        float                                                   |
|          |  binary       str (bytes in Python 3)                                 |
|          |  timestamp    datetime.datetime                                       |
|          |  pairs        list of pairs                                           |
|          |  omap         --                                                      |
|          |  set          set                                                     |
|          |  str          str or unicode (str in Python 3)                        |
|          |  seq          list                                                    |
|          |  map          dict                                                    |
|          |  enum         value must be one of the vars' values                   |
|          | ===========  =======================================                  |
|          |                                                                       |
|          | Defined vars which are passed in as post data                         |
|          | will be populated as Jinja2 variables or                              |
|          | appended to the command or script upon                                |
|          | execution.                                                            |
|          |                                                                       |
|          | =========  ======  ================================                   |
|          |  option     type    Description                                       |
|          | =========  ======  ================================                   |
|          |  name       str     the name of the variable                          |
|          |  type       str     type of the variable                              |
|          |  option     bool    is the variable an option                         |
|          |  values     list    posibble values for enum type                     |
|          | =========  ======  ================================                   |
|          |                                                                       |
+----------+-----------------------------------------------------------------------+
| schedule | Specification of a scheduled trigger to run the                       |
|          | job.                                                                  |
|          |                                                                       |
|          | +-----------+------+-----------------------------------------------+  |
|          | | option    | type | Description                                   |  |
|          | +===========+======+===============================================+  |
|          | | time      | str  | date and time of first run                    |  |
|          | +-----------+------+-----------------------------------------------+  |
|          | | frequency | str  | a crontab format frequency string or a period |  |
|          | |           |      | of time in seconds between executions         |  |
|          | +-----------+------+-----------------------------------------------+  |
|          | | priority  | int  | priority order relative to mutually           |  |
|          | |           |      | scheduled jobs                                |  |
|          | +-----------+------+-----------------------------------------------+  |
|          |                                                                       |
+----------+-----------------------------------------------------------------------+

\* *Both jobs defined with command and script have the overseer configuration repository's bin directory as well as the overseer library directories on the shell path.*

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
The paths to the library programs will be prepended to the PATH variable available, before the configration repository's bin directory.

Utilities provided from the library:

+--------------+------------------------------------------------------------------------------+
| Name         | Description                                                                  |
+==============+==============================================================================+
| event        | Create an execution event.                                                   |
|              |                                                                              |
|              | +------------+------+-----------------------------------------------+        |
|              | | argument   | type | Description                                   |        |
|              | +============+======+===============================================+        |
|              | | event name | str  | The name of event to create                   |        |
|              | +------------+------+-----------------------------------------------+        |
|              | | cause      | str  | The cause of the event                        |        |
|              | +------------+------+-----------------------------------------------+        |
|              |                                                                              |
+--------------+------------------------------------------------------------------------------+

:Author: Brandon Schlueter <overseer@schlueter.blue>
:Copyright: Brandon Schlueter
:License: Affero General Public License v3 or newer
