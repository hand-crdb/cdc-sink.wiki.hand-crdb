cdc-sink can be configured to allow one CockroachDB cluster ingest CDC feeds from another CockroachDB cluster.

A thorough understanding of [CockroachDB CDC features](https://www.cockroachlabs.com/docs/stable/change-data-capture-overview.html)
is a prerequisite for any operators deploying cdc-sink in a CockroachDB-to-CockroachDB configuration.

## Overview

- A source CRDB cluster emits changes in near-real time via the enterprise `CHANGEFEED` feature.
- `cdc-sink` accepts the changes via an HTTP end point.
- `cdc-sink` applies the changes to the target cluster in one of several modes.

```text
+------------------------+
|     source CRDB        |
|                        |
| CDC {source_table(s)}  |
+------------------------+
          |
          V
   http://ip:26258
+---------------------+
|       cdc-sink      |
|                     |
+---------------------+
          |
          V
 postgresql://ip:26257
+------------------------+
|    target CRDB         |
| {destination_table(s)} |
+------------------------+

```

## Theory of operation

- `cdc-sink` receives a stream of partially-ordered row mutations from one of more nodes in the
  source cluster.
- The two leading path segments from the URL,
  `/target_database/target_schema`, are combined with the table name from the mutation payload to
  determine the target table:
  `target_database.target_schema.source_table_name`.
- The incoming mutations are staged on a per-table basis into the target cluster, which is ordered
  by the `updated` MVCC timestamps.
- Upon receipt of a `resolved` timestamp, mutations whose timestamps are less that the new, resolved
  timestamp are dequeued.
- The dequeued mutations are applied to the target tables, either as
  `UPSERT` or `DELETE` operations.
- The `resolved` timestamp is stored for future use.

There should be, at most, one source `CHANGEFEED` per target `SCHEMA`. Multiple instances
of `cdc-sink` should be used in production scenarios.

The behavior described above, staging and application as two phases, ensures that the target tables
will be in a transactionally-consistent state with respect to the source database. This is desirable
in a steady-state configuration, however it imposes limitations on the maximum throughput or
transaction size that `cdc-sink` is able to achieve. In situations where a large volume or a high
rate of data must be transferred, e.g.: an initial CDC `CHANGEFEED` backfill, the server may be
started with the `--immediate` command-line parameter in order to apply all mutations to the target
table without staging them.

`cdc-sink` relies heavily upon the delivery guarantees provided by a CockroachDB `CHANGEFEED`. When
errors are encountered while staging or applying mutations, `cdc-sink` will return them to the
incoming changefeed request. Errors will then become visible in the output of `SHOW JOBS` or in the
administrative UI on the source cluster.

## Instructions

1. In the source cluster, choose the table(s) that you would like to replicate.
1. In the destination cluster, re-create those tables within a
   single [SQL database schema](https://www.cockroachlabs.com/docs/stable/sql-name-resolution#naming-hierarchy)
   and match the table definitions exactly:
    - Don't create any new constraints on any destination table(s).
    - Don't have any foreign key relationships with the destination table(s).
    - It's imperative that the columns are named exactly the same in the destination table.
1. Create the staging database `_cdc_sink` in the destination cluster.
    - It is recommended that you reduce the default GC time to five minutes, since these tables have a high volume of deletes.
    - `ALTER DATABASE _cdc_sink CONFIGURE ZONE USING gc.ttlseconds=300`
1. Start `cdc-sink` either on
    - all nodes of the destination cluster
    - one or more servers that can reach the destination cluster
1. Set the cluster setting of the source cluster to enable range feeds:
   `SET CLUSTER SETTING kv.rangefeed.enabled = true`
1. Once it starts up, enable a cdc feed from the source cluster
    - `CREATE CHANGEFEED FOR TABLE [source_table]`
      `INTO 'http://[cdc-sink-host:port]/target_database/target_schema' WITH updated,resolved=1s,min_checkpoint_frequency=1s`
    - The `target_database` path element is the name passed to the `CREATE DATABASE` command.
    - The `target_schema` element is the name of
      the [user-defined schema](https://www.cockroachlabs.com/docs/stable/create-schema.html) in the
      target database. By default, every SQL `DATABASE` has a `SCHEMA` whose name is "`public`".
    - Be sure to always use the options `updated`, `resolved`, `min_checkpoint_frequency` as these are required for timely replication.
    - The `protect_data_from_gc_on_pause` option is not required, but can be used in situations
      where a `cdc-sink` instance may be unavailable for periods longer than the source's
      garbage-collection window (25 hours by default).


### Table schema

cdc-sink will automatically create a number of staging and metadata tables in a database named `_cdc_sink`.
This database must be manually created, using `CREATE DATABASE _cdc_sink`.

```sql
CREATE TABLE _targetDB_targetSchema_targetTable
(
    nanos   INT    NOT NULL,
    logical INT    NOT NULL,
    key     STRING NOT NULL,
    mut     JSONB  NOT NULL,
    PRIMARY KEY (nanos, logical, key)
)
```

There is also a timestamp-tracking table.

```sql
CREATE TABLE _timestamps_
(
    key     STRING NOT NULL PRIMARY KEY,
    nanos   INT8   NOT NULL,
    logical INT8   NOT NULL
)
```

## Starting Changefeed Replication

```text
Usage:
  cdc-sink start [flags]

Flags:
      --applyTimeout duration     the maximum amount of time to wait for an update to be applied (default 30s)
      --backfillWindow duration   use a high-throughput, but non-transactional mode if replication is this far behind
      --bindAddr string           the network address to bind to (default ":26258")
      --bytesInFlight int         apply backpressure when amount of in-flight mutation data reaches this limit (default 10485760)
      --disableAuthentication     disable authentication of incoming cdc-sink requests; not recommended for production.
      --fanShards int             the number of concurrent connections to use when writing data in fan mode (default 16)
      --foreignKeys               re-order updates to satisfy foreign key constraints
  -h, --help                      help for start
      --idealFlushBatchSize int   try to apply at least this many mutations per resolved-timestamp window (default 1000)
      --immediate                 apply data without waiting for transaction boundaries
      --metaTable ident           the name of the table in which to store resolved timestamps (default resolved_timestamps)
      --ndjsonBufferSize int      the maximum amount of data to buffer while reading a single line of ndjson input; increase when source cluster has large blob values (default 65536)
      --retryDelay duration       the amount of time to sleep between replication retries (default 10s)
      --selectBatchSize int       the number of rows to select at once when reading staged data (default 10000)
      --stagingDB ident           a SQL database to store metadata in (default _cdc_sink)
      --standbyTimeout duration   how often to commit the consistent point (default 5s)
      --targetConn string         the target cluster's connection string
      --targetDBConns int         the maximum pool size to the target cluster (default 1024)
      --tlsCertificate string     a path to a PEM-encoded TLS certificate chain
      --tlsPrivateKey string      a path to a PEM-encoded TLS private key
      --tlsSelfSigned             if true, generate a self-signed TLS certificate valid for 'localhost'
      --userscript string         the path to a configuration script, see userscript subcommand

Global Flags:
      --logDestination string   write logs to a file, instead of stdout
      --logFormat string        choose log output format [ fluent, text ] (default "text")
  -v, --verbose count           increase logging verbosity to debug; repeat for trace
```

### Example

```bash
# install cdc-sink
go install github.com/cockroachdb/cdc-sink@latest

# source CRDB is single node
cockroach start-single-node --listen-addr :30000 --http-addr :30001 --store cockroach-data/30000 --insecure --background

# target CRDB is single node as well
cockroach start-single-node --listen-addr :30002 --http-addr :30003 --store cockroach-data/30002 --insecure --background

# source ycsb.usertable is populated with 10 rows
cockroach workload init ycsb 'postgresql://root@localhost:30000/ycsb?sslmode=disable' --families=false --insert-count=10

# target ycsb.usertable is empty
cockroach workload init ycsb 'postgresql://root@localhost:30002/ycsb?sslmode=disable' --families=false --insert-count=0

# source has rows, target is empty
cockroach sql --port 30000 --insecure -e "select * from ycsb.usertable"
cockroach sql --port 30002 --insecure -e "select * from ycsb.usertable"

# create staging database for cdc-sink
cockroach sql --port 30002 --insecure -e "CREATE DATABASE _cdc_sink"
# cdc-sink started as a background task. Remove the tls flag for CockroachDB <= v21.1
cdc-sink start --bindAddr :30004 --tlsSelfSigned --disableAuthentication --targetConn 'postgresql://root@localhost:30002/?sslmode=disable' &

# start the CDC that will send across the initial data snapshot
# Versions of CRDB before v21.2 should use the experimental-http:// URL scheme
cockroach sql --insecure --port 30000 <<EOF
-- add enterprise license
-- SET CLUSTER SETTING cluster.organization = 'Acme Company';
-- SET CLUSTER SETTING enterprise.license = 'xxxxxxxxxxxx';
SET CLUSTER SETTING kv.rangefeed.enabled = true;
CREATE CHANGEFEED FOR TABLE YCSB.USERTABLE
  INTO 'webhook-https://127.0.0.1:30004/ycsb/public?insecure_tls_skip_verify=true'
  WITH updated, resolved='10s',
       webhook_sink_config='{"Flush":{"Messages":1000,"Frequency":"1s"}}';
EOF

# source has rows, target is the same as the source
cockroach sql --port 30000 --insecure -e "select * from ycsb.usertable"
cockroach sql --port 30002 --insecure -e "select * from ycsb.usertable"

# start YCSB workload and the incremental updates happens automagically
cockroach workload run ycsb 'postgresql://root@localhost:30000/ycsb?sslmode=disable' --families=false --insert-count=100 --concurrency=1 &

# source updates are applied to the target
cockroach sql --port 30000 --insecure -e "select * from ycsb.usertable"
cockroach sql --port 30002 --insecure -e "select * from ycsb.usertable"
```

### Backfill mode

The `--backfillWindow` flag can be used with logical-replication modes to ignore source transaction
boundaries if the replication lag exceeds a certain threshold or when first populating data into the
target database. This flag is useful when `cdc-sink` is not expected to be run continuously and will
need to catch up to the current state in the source database.

### Immediate mode

Immediate mode writes incoming mutations to the target schema as soon as they are received, instead
of waiting for a resolved timestamp. Transaction boundaries from the source database will not be
preserved, but overall load on the destination database will be reduced. This sacrifices transaction
atomicity for performance, and may be appropriate in eventually-consistency use cases.

Immediate mode is enabled by passing the `--immediate` flag.

## Limitations

Note that while limitation exist, cdc-sink makes an effort to detect these cases and will push back on the source changefeed if there is a chance that incoming data cannot be entirely applied to the destination.

- all the limitations from CDC hold true.
  - See <https://www.cockroachlabs.com/docs/dev/change-data-capture.html#known-limitations>
- schema changes do not work,
  - in order to perform a schema change
    1. stop the change feed
    2. stop cdc-sink
    3. make the schema changes to both tables
    4. start cdc-sink
    5. restart the change feed
- constraints on the destination table
  - foreign keys
    - there is no guarantee that foreign keys between two tables will arrive in the correct
      order so please only use them on the source table
    - different table constraints
      - anything that has a tighter constraint than the original table may break the streaming
- the schema of the destination table must match the primary table exactly

### JSONB columns

Due to [limitations](https://github.com/cockroachdb/cockroach/issues/103289) in how JSONB values
are encoded by CockroachDB, it is not possible to distinguish the SQL `NULL` value from a
JSONB `null` token. We do not recommend the use of nullable JSONB column if the `null` JSONB token
may be used as a value.  Instead, it is preferable to declare the destination column as
`JSONB NOT NULL` and use a [substutite expression](https://github.com/cockroachdb/cdc-sink/wiki/Data-Behaviors#substitute-expressions)
to replace a SQL `NULL` value with the JSONB `null` token.

```
UPSERT
INTO _cdc_sink.apply_config (target_db, target_schema, target_table, target_column, expr)
VALUES ('my_database', 'public', 'my_table', 'the_jsonb_column', 'COALESCE( $0::JSONB, ''null''::JSONB)');
```

### Schema Expansions

While the schema of the secondary table must match that of the primary table, specifically the
primary index. There are some other changes than can be made on the destination side.

- Different and new secondary indexes are allowed.
- Different zone configs are allowed.
- Adding new computed columns, that cannot reject any row, should work.