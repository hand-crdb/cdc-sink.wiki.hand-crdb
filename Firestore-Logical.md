# Google Firestore replication

`cdc-sink` supports [Google Cloud Firestore](https://cloud.google.com/firestore) as a source of
logical-replication data. It is very likely that a data model which is appropriate for a document
store will need signification revisions to be useful in a relational, tabular database. This
transformation can be accomplished within cdc-sink with a userscript (see the output
of `cdc-sink userscript` for details).

```text
start a Google Cloud Firestore logical replication feed

Usage:
  cdc-sink fslogical [flags]

Flags:
      --applyTimeout duration               the maximum amount of time to wait for an update to be applied (default 30s)
      --backfillBatchSize int               the number of documents to load when backfilling (default 10000)
      --bytesInFlight int                   apply backpressure when amount of in-flight mutation data reaches this limit (default 10485760)
      --credentials string                  a file containing JSON service credentials.
      --docID ident                         the column name (likely the primary key) to populate with the document id (default id)
      --fanShards int                       the number of concurrent connections to use when writing data in fan mode (default 16)
      --foreignKeys                         re-order updates to satisfy foreign key constraints
  -h, --help                                help for fslogical
      --loopName string                     identifies the logical replication loops in metrics (default "fslogical")
      --metricsAddr string                  a host:port to serve metrics from at /_/varz
      --projectID string                    override the project id contained in the credentials file
      --retryDelay duration                 the amount of time to sleep between replication retries (default 10s)
      --stagingDB ident                     a SQL database to store metadata in (default _cdc_sink)
      --standbyTimeout duration             how often to commit the consistent point (default 5s)
      --targetConn string                   the target cluster's connection string
      --targetDB ident                      the SQL database in the target cluster to update
      --targetDBConns int                   the maximum pool size to the target cluster (default 1024)
      --tombstoneCollection string          the name of a collection that contains document Tombstones
      --tombstoneCollectionProperty ident   the property name in a tombstone document that contains the original collection name (default collection)
      --tombstoneIgnoreUnmapped             skip, rather than reject, any tombstone documents that do not map to a target table
      --updatedAt ident                     the name of a document property used for high-water marks (default updated_at)
      --userscript string                   the path to a configuration script, see userscript subcommand

Global Flags:
      --logDestination string   write logs to a file, instead of stdout
      --logFormat string        choose log output format [ fluent, text ] (default "text")
  -v, --verbose count           increase logging verbosity to debug; repeat for trace
```

Source document collections are mapped onto target tables within the destination database. A
document is the unit of atomicity when applying updates. Updates to collections are applied to their
target tables via independent replication loops. This is to say that code which consumes replicated
data from the target database may encounter skew, since there are no synchronization guarantees
provided by Firestore.

The use of an `extras` column will allow source documents with variable schemas to be stored in
a `JSONB` column, to facilitate future schema changes.

## Document requirements

All documents to be replicated must have a timestamp property which is set
to `FieldValue.serverTimestamp()` or its equivalent in your SDK of choice. By default, this property
is named `updated_at`, but the specific property name can be changed with the `--updatedAt` flag.
This server-assigned timestamp allows `cdc-sink` to behave in a resumable manner.

The source document ID will be provided as a property specified by the `--docID` flag, which
defaults to `id`. This property must be used as the primary key of the destination table. Future
re-keying of the replicated data is possible by adding a unique secondary index on columns with a
reasonable `DEFAULT` value, such as `gen_random_uuid()`.

## Configuring source collections

The use of Firestore as a replication source is configured through cdc-sink's userscript mechanism.
Top-level collections are configured by declaring a source and a target table or a dispatch
function. [Collection group queries](https://firebase.google.com/docs/firestore/query-data/queries#collection-group-query)
can be used by using a `group:` prefix with the collection name. If documents have sub-collections
with dynamic names, the `recurse` source option will iterate over each document's sub-collections
when the document is updated.

```typescript
// This module name is recognized by the userscript loader. A corresponding .d.ts file
// can be retrieved by running "cdc-sink userscript --api"
import * as api from "cdc-sink@v1";

// This call will map a top-level collection onto a single table. A simple
// mapping like this can work for "flat" documents that translate directly
// to SQL rows.
api.configureSource("my-collection", {target: "my_table"});

// This call creates a collection-group query. This is useful when
// documents contain a sub-collection with a common name.
api.configureSource("group:subcollection", {target: "other_table"});

// More complex mappings of documents to rows can be accomplished by using
// a dispatch function. This will recive the source document and return a
// mapping of destintation tables to, potentially, multiple rows.
api.configureSource("complex-collection", {
    dispatch: (doc, meta) => {
        return {
            "parent_table": [{"some": "data"}],
            "child_table": [{"child": 1}, {"child": 2}],
            "join_table": [{}, {}, {}]
        }
    },
    deletesTo: "parent_table",
})
```

## Hard deletion

It is not possible to receive notification of deleted documents when `cdc-sink` is not running. If
users require a "hard delete" solution, it is necessary to write tombstone documents to a separate
collection.

The structure of a tombstone is as follows:

```json
{
  "id": "AABBCCDD",
  "collection": "my-collection",
  "updated_at": "2022-08-11T13:01:59Z"
}
```

The specific property names used in the tombstone document can be configured by the `--docID`
, `--tombstoneCollectionProperty`, and `--updatedAt` flags. The `--tombstoneCollection` flag enables
the behavior.
