Creating a per-user MySQL database for Cromwell on HPC
=======================================================

Use case
--------

On a compute cluster that uses a resource scheduler such as SLURM, multiple
users can login to a login node and submit jobs to the cluster. The SLURM
resource scheduler does some job accouting based on the user's ID and the
user's department

A Cromwell server does not work well in this use case, because it runs under
a single user. This means the job accounting does not work properly.

Cromwell's `run` command is used in these cases, but without a backing database
there is no possibility for call-caching, which is vital for production
workflows.

There are several options to alleviate this problem:

1. Use a MySQL database and share a common database between all users.
    pros:
        + Highly efficient database calls
    cons:
        + Shared database can get locked by any of the users.
        + Security risk: confidential information under the GDPR such as
          patient IDs can end up in the database, where everyone can read it.
2. Use Cromwell's in-memory database with a persistence file. Documented `here <https://cromwell.readthedocs.io/en/stable/Configuring/#database>`_:
    pros:
        + Allows per-project databases to be set up easily.
        + Everyone can share the same configuration as the database can be
          set up in the cromwell executions directory.
        + Permissions handled by filesystem permissions. No security risks.
    cons:
        + More memory usage required by cromwell.
        + The database consumes enormous amounts of disk space. (Multiple
          gigabytes).
        + Cromwell's interaction with the database is slow and on very large
          projects timeouts may occur when reading in the database persistence
          file.
3. Create a separate Cromwell server for every user that asks for it.
    pros:
        + Solves the problem with user permissions, filesystem permissions and
          per-user job accounting.
    cons:
        + Enormous maintenance burden.

Currently we use option number 2 on our cluster. Unfortunately it does not
scale towards thousands of jobs.

Option 4: Allow creation of a per user MySQL database using command line scripts
--------------------------------------------------------------------------------

MySQL has a server-based architecture. Setting up a server is usuallly not
possible without root permissions, however there are some features that enable
setting up MySQL for non-root users.

+ Mysql is available trough the conda-forge channel and can be installed with
  miniconda.
+ Disabling of networking altogether
+ Communicating via file socket

This way a per user database can be set up by the users themselves without
intervention.

step 1: Installing mysql
++++++++++++++++++++++++++++++++++++++++++++++++++++
Set up miniconda with the conda-forge channel. Then:

    conda create -y -n mysql mysql
    conda activate mysql

step 2: Setting up a default configuration
++++++++++++++++++++++++++++++++++++++++++++++++++++
`mysqld` config values can be overwritten on the command line or can be set
in a personal config file: `~/.my.cnf`.

Set the following values::

    [mysqld]
    datadir=/path/to/mysqldb
    skip-networking=TRUE
    socket=/path/to/socket

step 3: Set up the cromwell configuration
++++++++++++++++++++++++++++++++++++++++++

.. code-block::

    database {
      profile = "slick.jdbc.MySQLProfile$"
      db {
        driver = "com.mysql.cj.jdbc.Driver"
        url = "jdbc:mysql:file:///path/to/socket/cromwell?rewriteBatchedStatements=true"
        user = "user"
        password = "pass"
        connectionTimeout = 5000
      }
    }


step 4: Start
+++++++++++++++++++++++++++++++++++++
+ On first start: ``mysqld --initialize``
+ Start the database ``mysqld -D``
