|query-exporter logo|

Export Prometheus metrics from SQL queries
==========================================

|Latest Version| |Build Status| |Coverage Status| |Snap Status| |Docker Pulls|

``query-exporter`` is a Prometheus_ exporter which allows collecting metrics
from database queries, at specified time intervals.

It uses SQLAlchemy_ to connect to different database engines, including
PostgreSQL, MySQL, Oracle and Microsoft SQL Server.

Each query can be run on multiple databases, and update multiple metrics.

The application is simply run as::

  query-exporter config.yaml

where the passed configuration file contains the definitions of the databases
to connect and queries to perform to update metrics.


Configuration file format
-------------------------

A sample configuration file for the application looks like this:

.. code:: yaml

    databases:
      db1:
        dsn: sqlite://
        connect-sql:
          - PRAGMA application_id = 123
          - PRAGMA auto_vacuum = 1
        labels:
          region: us1
          app: app1
      db2:
        dsn: sqlite://
        keep-connected: false
        labels:
          region: us2
          app: app1

    metrics:
      metric1:
        type: gauge
        description: A sample gauge
      metric2:
        type: summary
        description: A sample summary
        labels: [l1, l2]
      metric3:
        type: histogram
        description: A sample histogram
        buckets: [10, 20, 50, 100, 1000]
      metric4:
        type: enum
        description: A sample enum
        states: [foo, bar, baz]

    queries:
      query1:
        interval: 5
        databases: [db1]
        metrics: [metric1]
        sql: SELECT random() / 1000000000000000 AS metric1
      query2:
        interval: 20
        databases: [db1, db2]
        metrics: [metric2, metric3]
        sql: |
          SELECT abs(random() / 1000000000000000) AS metric2,
                 abs(random() / 10000000000000000) AS metric3,
                 "value1" AS l1,
                 "value2" AS l2
      query3:
        interval: 10
        databases: [db2]
        metrics: [metric4]
        sql: |
          SELECT value FROM (
            SELECT "foo" AS metric4 UNION
            SELECT "bar" AS metric3 UNION
            SELECT "baz" AS metric4
          )
          ORDER BY random()
          LIMIT 1

``databases`` section
~~~~~~~~~~~~~~~~~~~~~

This section contains defintions for databases to connect to. Key names are
arbitrary and only used to reference databases in the ``queries`` section.

Each database defintions can have the following keys:

``dsn``:
  the connection string for the database, in the following format::

    dialect[+driver]://[username:password][@host:port]/database[?option=value&...]

  See `SQLAlchemy documentation`_ for details on available engines.

  It's also possible to get the connection string from an environment variable
  (e.g. ``$CONNECTION_STRING``) by setting ``dsn`` to::

    env:CONNECTION_STRING

``connect-sql``:
  An optional list of queries to run right after database connection. This can
  be used to set up connection-wise parameters and configurations.

``keep-connected``:
  whether to keep the connection open for the database between queries, or
  disconnect after each one. If not specified, defaults to ``true``.  Setting
  this option to ``false`` might be useful if queries on a database are run
  with very long interval, to avoid holding idle connections.

``autocommit``:
  whether to set autocommit for the database connection. If not specified,
  defaults to ``true``.  This should only be changed to ``false`` if specific
  queries require it.

``labels``:
  an optional mapping of label names and values to tag metrics collected from each database.
  When labels are used, all databases must define the same set of labels.

``metrics`` section
~~~~~~~~~~~~~~~~~~~

This section contains Prometheus_ metrics definitions. Keys are used as metric
names, and must therefore be valid metric identifiers.

Each metric definition can have the following keys:

``type``:
  the type of the metric, must be specified. The following metric types are
  supported:

  - counter
  - enum
  - gauge
  - histogram
  - summary

``description``:
  an optional description of the metric.

``labels``:
  an optional list of label names to apply to the metric.

  If specified, queries updating the metric must return rows that include
  values for each label in addition to the metric value.  Column names must
  match metric and labels names.

``buckets``:
  for ``histogram`` metrics, a list of buckets for the metrics.

  If not specified, default buckets are applied.

``states``:
  for ``enum`` metrics, a list of string values for possible states.

  Queries for updating the enum must return valid states.

``queries`` section
~~~~~~~~~~~~~~~~~~~

This section contains definitions for queries to perform. Key names are
arbitrary and only used to identify queries in logs.

Each query definition can have the following keys:la-

``databases``:
  the list of databases to run the query on.

  Names must match those defined in the ``databases`` section.

  Metrics are automatically tagged with the ``database`` label so that
  indipendent series are generated for each database that a query is run on.

``interval``:
  the time interval at which the query is run.

  The value is interpreted as seconds if no suffix is specified; valid suffixes
  are ``s``, ``m``, ``h``, ``d``. Only integer values are accepted.

  If no value is specified (or specified as ``null``), the query is only
  executed upon HTTP requests.

``metrics``:
  the list of metrics that the query updates.

  Names must match those defined in the ``metrics`` section.

``parameters``:
  an optional list of parameters sets to run the query with.

  If a query is specified with parameters in its ``sql``, it will be run once
  for every set of parameters specified in this list, for every interval.

  Each parameter set must be a dictionary where keys must match parameters
  names from the query SQL (e.g. ``:param``).

  As an example:

  .. code:: yaml

      query:
        databases: [db]
        metrics: [metric]
        sql: |
          SELECT COUNT(*) AS metric FROM table
          WHERE id > :param1 AND id < :param2
        parameters:
          - param1: 10
            param2: 20
          - param1: 30
            param2: 40

``sql``:
  the SQL text of the query.

  The query must return columns with names that match those of the metrics
  defined in ``metrics``, plus those of labels (if any) for all these metrics.

  .. code:: yaml

      query:
        databases: [db]
        metrics: [metric1, metric2]
        sql: SELECT 10.0 AS metric1, 20.0 AS metric2

  will update ``metric1`` to ``10.0`` and ``metric2`` to ``20.0``.

  **Note**:
   since ``:`` is used for parameter markers (see ``parameters`` above),
   literal single ``:`` at the beginning of a word must be escaped with
   backslash (e.g. ``SELECT '\:bar' FROM table``).  There's no need to escape
   when the colon occurs inside a word (e.g. ``SELECT 'foo:bar' FROM table``).


Metrics endpoint
----------------

The exporter listens on port ``9560`` providing the standard ``/metrics``
endpoint.

By default, the port is bound on ``localhost``. Note that if the name resolves
both IPv4 and IPv6 addressses, the exporter will bind on both.

For the configuration above, the endpoint would return something like this::

  # HELP database_errors_total Number of database errors
  # TYPE database_errors_total counter
  # HELP queries_total Number of database queries
  # TYPE queries_total counter
  queries_total{database="db2",status="success"} 2.0
  queries_total{database="db1",status="success"} 3.0
  # TYPE queries_created gauge
  queries_created{database="db2",status="success"} 1.558334663380845e+09
  queries_created{database="db1",status="success"} 1.558334663381175e+09
  # HELP metric1 A sample gauge
  # TYPE metric1 gauge
  metric1{database="db1"} 2580.0
  # HELP metric2 A sample summary
  # TYPE metric2 summary
  metric2_count{database="db2",l1="value1",l2="value2"} 1.0
  metric2_sum{database="db2",l1="value1",l2="value2"} 6476.0
  metric2_count{database="db1",l1="value1",l2="value2"} 1.0
  metric2_sum{database="db1",l1="value1",l2="value2"} 2340.0
  # TYPE metric2_created gauge
  metric2_created{database="db2",l1="value1",l2="value2"} 1.5583346633805697e+09
  metric2_created{database="db1",l1="value1",l2="value2"} 1.5583346633816812e+09
  # HELP metric3 A sample histogram
  # TYPE metric3 histogram
  metric3_bucket{database="db2",le="10.0"} 0.0
  metric3_bucket{database="db2",le="20.0"} 0.0
  metric3_bucket{database="db2",le="50.0"} 0.0
  metric3_bucket{database="db2",le="100.0"} 0.0
  metric3_bucket{database="db2",le="1000.0"} 1.0
  metric3_bucket{database="db2",le="+Inf"} 1.0
  metric3_count{database="db2"} 1.0
  metric3_sum{database="db2"} 135.0
  metric3_bucket{database="db1",le="10.0"} 0.0
  metric3_bucket{database="db1",le="20.0"} 0.0
  metric3_bucket{database="db1",le="50.0"} 0.0
  metric3_bucket{database="db1",le="100.0"} 0.0
  metric3_bucket{database="db1",le="1000.0"} 1.0
  metric3_bucket{database="db1",le="+Inf"} 1.0
  metric3_count{database="db1"} 1.0
  metric3_sum{database="db1"} 164.0
  # TYPE metric3_created gauge
  metric3_created{database="db2"} 1.5583346633807e+09
  metric3_created{database="db1"} 1.558334663381795e+09
  # HELP metric4 A sample enum
  # TYPE metric4 gauge
  metric4{database="db2",metric4="foo"} 0.0
  metric4{database="db2",metric4="bar"} 0.0
  metric4{database="db2",metric4="baz"} 1.0


Database engines
----------------

SQLAlchemy_ doesn't depend on specific Python database modules at
installation. This means additional modules might need to be installed for
engines in use. These can be installed as follows::

  pip install SQLAlchemy[postgresql] SQLAlchemy[mysql] ...

based on which database engines are needed.

See `supported databases`_ for details.


Install from Snap
-----------------

|Get it from the Snap Store|

``query-exporter`` can be installed from `Snap Store`_ on systems where Snaps
are supported, via::

  sudo snap install query-exporter

The snap provides both the ``query-exporter`` command and a deamon instance of
the command, managed via a Systemd service.

To configure the daemon:

- create or edit ``/var/snap/query-exporter/current/config.yaml`` with the
  configuration
- run ``sudo snap restart query-exporter``

The snap has support for connecting the following databases:

- PostgreSQL (``postgresql://``)
- MySQL (``mysql://``)
- SQLite (``sqlite://``)
- Microsoft SQL Server (``mssql+pyodbc://``)


Run in Docker
-------------

``query-exporter`` can be run inside Docker_ containers, and is availble from the `Docker Hub`_::

  docker run -p 9560:9560/tcp -v "$PWD/config.yaml:/config.yaml" --rm -it \
      adonato/query-exporter:latest -- /config.yaml

The image has support for connecting the following databases:

- PostgreSQL (``postgresql://``)
- MySQL (``mysql://``)
- SQLite (``sqlite://``)
- Microsoft SQL Server (``mssql+pyodbc://``)
- IBM DB2 (``db2+ibm_db://``)


.. _Prometheus: https://prometheus.io/
.. _SQLAlchemy: https://www.sqlalchemy.org/
.. _`SQLAlchemy documentation`:
   http://docs.sqlalchemy.org/en/latest/core/engines.html#database-urls
.. _`supported databases`:
   http://docs.sqlalchemy.org/en/latest/core/engines.html#supported-databases
.. _`Snap Store`: https://snapcraft.io
.. _Docker: http://docker.com/
.. _`Docker Hub`: https://hub.docker.com/r/adonato/query-exporter

.. |query-exporter logo| image:: ./logo.svg
   :alt: query-exporter logo
.. |Latest Version| image:: https://img.shields.io/pypi/v/query-exporter.svg
   :alt: Latest Version
   :target: https://pypi.python.org/pypi/query-exporter
.. |Build Status| image:: https://img.shields.io/travis/albertodonato/query-exporter.svg
   :alt: Build Status
   :target: https://travis-ci.com/albertodonato/query-exporter
.. |Coverage Status| image:: https://img.shields.io/codecov/c/github/albertodonato/query-exporter/master.svg
   :alt: Coverage Status
   :target: https://codecov.io/gh/albertodonato/query-exporter
.. |Snap Status| image:: https://build.snapcraft.io/badge/albertodonato/query-exporter.svg
   :alt: Snap Status
   :target: https://build.snapcraft.io/user/albertodonato/query-exporter
.. |Get it from the Snap Store| image:: https://snapcraft.io/static/images/badges/en/snap-store-black.svg
   :alt: Get it from the Snap Store
   :target: https://snapcraft.io/query-exporter
.. |Docker Pulls| image:: https://img.shields.io/docker/pulls/adonato/query-exporter
   :alt: Docker Pulls
   :target: https://hub.docker.com/r/adonato/query-exporter
