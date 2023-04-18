# MySQL/MariaDB Replication

Another possibility is to connect to a MySQL/MariaDB database instance to consume a
transaction-based replication feed using global transaction identifiers (GTIDs). This is primarily
intended for migration use-cases, in which it is desirable to have a minimum- or zero-downtime
migration from MySQL to CockroachDB. For an overview of MySQL replication with GTIDs, refer
to
[MySQL Replication with Global Transaction Identifiers](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids.html).
For an overview of MariaDB replication refer to
[MariaDB Replication Overview](https://mariadb.com/kb/en/replication-overview/).

```text
start a mySQL replication feed

Usage:
  cdc-sink mylogical [flags]

Flags:
      --applyTimeout duration         the maximum amount of time to wait for an update to be applied (default 30s)
      --backfillWindow duration       use a high-throughput, but non-transactional mode if replication is this far behind
      --bytesInFlight int             apply backpressure when amount of in-flight mutation data reaches this limit (default 10485760)
      --defaultGTIDSet string         default GTIDSet. Used if no state is persisted
      --fanShards int                 the number of concurrent connections to use when writing data in fan mode (default 16)
      --foreignKeys                   re-order updates to satisfy foreign key constraints
  -h, --help                          help for mylogical
      --immediate                     apply data without waiting for transaction boundaries
      --loopName string               identify the replication loop in metrics (default "mylogical")
      --metricsAddr string            a host:port to serve metrics from at /_/varz
      --replicationProcessID uint32   the replication process id to report to the source database (default 10)
      --retryDelay duration           the amount of time to sleep between replication retries (default 10s)
      --sourceConn string             the source database's connection string
      --stagingDB ident               a SQL database to store metadata in (default _cdc_sink)
      --standbyTimeout duration       how often to commit the consistent point (default 5s)
      --targetConn string             the target cluster's connection string
      --targetDB ident                the SQL database in the target cluster to update
      --targetDBConns int             the maximum pool size to the target cluster (default 1024)
      --userscript string             the path to a configuration script, see userscript subcommand

Global Flags:
      --logDestination string   write logs to a file, instead of stdout
      --logFormat string        choose log output format [ fluent, text ] (default "text")
  -v, --verbose count           increase logging verbosity to debug; repeat for trace
```

The theory of operation is similar to the standard use case, the only difference is that `cdc-sink`
connects to the source database to receive a replication feed, rather than act as the target for a
webhook.

## MySQL/MariaDB Replication Setup

- The MySQL server should have the following settings:

```text
      --gtid-mode=on
      --enforce-gtid-consistency=on
      --binlog-row-metadata=full
```

- If server is MariaDB, it should have the following settings:

```text
      --log-bin
      --server_id=1
      --log-basename=master1
      --binlog-format=row
      --binlog-row-metadata=full
```

- Verify the master status, on the MySQL/MariaDB server

```text
    show master status;
```

- Perform a backup of the database. Note: Starting with MariaDB 10.0.13, mysqldump automatically
  includes the GTID position as a comment in the backup file if either the --master-data or
  --dump-slave option is used.

```bash
   mysqldump -p db_name > backup-file.sql
```

- Note the GTID state at the beginning of the backup, as reported in the backup file. For instance:

```SQL
-- MySQL:
--
-- GTID state at the beginning of the backup
--

SET @@GLOBAL.GTID_PURGED=/*!80000 '+'*/ '6fa7e6ef-c49a-11ec-950a-0242ac120002:1-8';
```

```SQL
-- MariaDB:
--
-- GTID to start replication from
--

SET GLOBAL gtid_slave_pos='0-1-1';
```

- Import the database into Cockroach DB, following the instructions
  at [Migrate from MySQL](https://www.cockroachlabs.com/docs/stable/migrate-from-mysql.html).
- Run `cdc-sink mylogical` with at least the `--sourceConn`, `--targetConn`
  , `--defaultGTIDSet` and `--targetDB`. Set `--defaultGTIDSet` to the GTID state shown above.
