# PostgreSQL logical replication

An alternate application of `cdc-sink` is to connect to a PostgreSQL-compatible database instance to
consume a logical replication feed. This is primarily intended for migration use-cases, in which it
is desirable to have a minimum- or zero-downtime migration from PostgreSQL to CockroachDB.

```text
start a pg logical replication feed

Usage:
  cdc-sink pglogical [flags]

Flags:
      --applyTimeout duration     the maximum amount of time to wait for an update to be applied (default 30s)
      --backfillWindow duration   use a high-throughput, but non-transactional mode if replication is this far behind
      --bytesInFlight int         apply backpressure when amount of in-flight mutation data reaches this limit (default 10485760)
      --fanShards int             the number of concurrent connections to use when writing data in fan mode (default 16)
      --foreignKeys               re-order updates to satisfy foreign key constraints
  -h, --help                      help for pglogical
      --immediate                 apply data without waiting for transaction boundaries
      --loopName string           identify the replication loop in metrics (default "pglogical")
      --metricsAddr string        a host:port to serve metrics from at /_/varz
      --publicationName string    the publication within the source database to replicate
      --retryDelay duration       the amount of time to sleep between replication retries (default 10s)
      --slotName string           the replication slot in the source database (default "cdc_sink")
      --sourceConn string         the source database's connection string
      --stagingDB ident           a SQL database to store metadata in (default _cdc_sink)
      --standbyTimeout duration   how often to commit the consistent point (default 5s)
      --targetConn string         the target cluster's connection string
      --targetDB ident            the SQL database in the target cluster to update
      --targetDBConns int         the maximum pool size to the target cluster (default 1024)
      --userscript string         the path to a configuration script, see userscript subcommand

Global Flags:
      --logDestination string   write logs to a file, instead of stdout
      --logFormat string        choose log output format [ fluent, text ] (default "text")
  -v, --verbose count           increase logging verbosity to debug; repeat for trace
```

The theory of operation is similar to the standard use case, the only difference is that `cdc-sink`
connects to the source database to receive a replication feed, rather than act as the target for a
webhook.

## Postgres Replication Setup

- A review of
  PostgreSQL [logical replication](https://www.postgresql.org/docs/current/logical-replication.html)
  will be useful to establish the limitations of logical replication.
- Run `CREATE PUBLICATION my_pub FOR ALL TABLES;` in the source database. The name `my_pub` can be
  changed as desired and must be provided to the `--publicationName` flag. Specifying only a subset
  of tables in the source database is possible.
- Run `SELECT pg_create_logical_replication_slot('cdc_sink', 'pgoutput');` in the source database.
  The value `cdc_sink` may be changed and should be passed to the `--slotName` flag.
- Run `SELECT pg_export_snapshot();` to create a consistent point for bulk data export and leave
  the source PosgreSQL session open. (Snapshot ids are valid only for the lifetime of the session
  which created them.)
  The value returned from this function can be passed
  to [`pg_dump --snapshot <snapshot_id>`](https://www.postgresql.org/docs/current/app-pgdump.html)
  that is subsequently
  [IMPORTed into CockroachDB](https://www.cockroachlabs.com/docs/stable/migrate-from-postgres.html)
  or used with
  [`SET TRANSACTION SNAPSHOT 'snapshot_id'`](https://www.postgresql.org/docs/current/sql-set-transaction.html)
  if migrating data using a SQL client.
- Complete the bulk data migration before continuing.
- Run `CREATE DATABASE IF NOT EXISTS _cdc_sink;` in the target cluster to create a staging arena.
- Run `cdc-sink pglogical` with at least the `--publicationName`, `--sourceConn`, `--targetConn`,
  and `--targetDB` flags after the bulk data migration has been completed. This will catch up with
  all database mutations that have occurred since the replication slot was created.

If you pass `--metricsAddr 127.0.0.1:13013`, a Prometheus-compatible HTTP endpoint will be available
at `/_/varz`. A trivial health-check endpoint is also available at `/_/healthz`.

To clean up from the above:

- `SELECT pg_drop_replication_slot('cdc_sink');`
- `DROP PUBLICATION my_pub;`
