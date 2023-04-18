# Data application behaviors

`cdc-sink` supports a number of behaviors that modify how mutations are applied to the target
database. These behaviors can be enabled for both cluster-to-cluster and logical-replication uses
of `cdc-sink`. At present, the configuration is stored in the target CockroachDB cluster in
the `_cdc_sink.apply_config` table. The configuration table is polled once per minute for changes or
in response to a `SIGHUP` sent to the`cdc-sink` process. The active apply configuration is available
from a `/_/config/apply` endpoint.

## Compare-and-set

A compare-and-set (CAS) mode allows `cdc-sink` to discard some updates, based on version- or
time-like fields.

**The use of CAS mode intentionally discards mutations and may cause data inconsistencies.**

Consider a table with a `version INT` column. If `cdc-sink` receives a message to update some row in
this table, the update will only be applied if there is no pre-existing row in the destination
table, or if the update's `version` value is strictly greater than the existing row in the
destination table. That is, updates are applied only if they would increase the `version` column or
if they would insert a new row.

To opt into CAS mode for a feed, set the `cas_order` column in the `apply_config` table for one or
more columns in a table. The `cas_order` column must have unique, contiguous values within any
target table. The default value of zero indicates that a column is not subject to CAS behavior.

```sql
UPSERT
INTO _cdc_sink.apply_config (target_db, target_schema, target_table, target_column, cas_order)
VALUES ('some_db', 'public', 'my_table', 'major_version', 1),
    ('some_db', 'public', 'my_table', 'minor_version', 2),
    ('some_db', 'public', 'my_table', 'patch_version', 3);
```

When multiple CAS columns are present, they are compared as a tuple. That is, the second column is
only compared if the first column value is equal between the source and destination clusters. In
this multi-column case, the following pseudo-sql clause is applied to each incoming update:

```sql
WHERE existing IS NULL OR (
  (incoming.version_major, incoming.version_minor, incoming.version_patch) >
  (existing.version_major, existing.version_minor, existing.version_patch)
)
```

Deletes from the source cluster are always applied.

## Deadlines

A deadline mode allows `cdc-sink` to discard some updates, by comparing a time-like field to the
destination cluster's time. This is useful when a feed's data is advisory only or changefeed
downtime is expected.

**The use of deadlines intentionally discards mutations and may cause data inconsistencies.**

To opt into deadline mode for a table, set the `deadline` column in the `apply_config` table for one
or more columns in a table. The default value of zero indicates that the column is not subject to
deadline behavior.

```sql
UPSERT
INTO _cdc_sink.apply_config (target_db, target_schema, target_table, target_column, deadline)
VALUES ('some_db', 'public', 'my_table', 'updated_at', '1m');
```

Given the above configuration, each incoming update would behave as though it were filtered with a
clause such as `WHERE incoming.updated_at > now() - '1m'::INTERVAL`.

Deletes from the source cluster are always applied.

## Extras column

By default, `cdc-sink` will reject any incoming data that cannot be mapped onto a column in the
target table. Users may specify a JSONB column in a target table to receive otherwise-unmapped
properties. This is useful in logical-replication scenarios where the source data has a variable
schema (e.g. migrations from document stores). Values in the JSONB column can be extracted in
subsequent schema-change operations
using [computed columns](https://www.cockroachlabs.com/docs/stable/jsonb.html#create-a-table-with-a-jsonb-column-and-a-computed-column)
.

To enable extras mode for a table, set the `extras` column to `true` in the `apply_config` for
exactly one column in the target table. The name of the extras column should be chosen to avoid any
conflicts with properties in incoming mutations.

```sql
UPSERT
INTO _cdc_sink.apply_config (target_db, target_schema, target_table, target_column, extras)
VALUES ('some_db', 'public', 'my_table', 'overflow_data', true);
```

## Ignore columns

By default, `cdc-sink` will reject incoming mutations that have columns which do not map to a column
in the destination table. This behavior can be selectively disabled by setting the `ignore` column
in the `apply_config` table.

**The use of ignore mode intentionally discards columns and may cause data inconsistencies.**

```sql
UPSERT
INTO _cdc_sink.apply_config (target_db, target_schema, target_table, target_column, ignore)
VALUES ('some_db', 'public', 'my_table', 'dont_care', true);
```

Ignore mode is mostly useful when using logical replication modes and performing schema changes.

## Rename columns

By default, `cdc-sink` uses the column names in a target table when decoding incoming payloads. An
alternate column name can be provided by setting the `src_name` column in the `apply_config` table.

```sql
UPSERT
INTO _cdc_sink.apply_config (target_db, target_schema, target_table, target_column, src_name)
VALUES ('some_db', 'public', 'my_table', 'column_in_target_cluster', 'column_in_source_data');
```

Renaming columns is mostly useful when using logical replication modes and performing schema
changes.

## Substitute expressions

Substitute expressions allow incoming column data to be transformed with arbitrary SQL expressions
when being applied to the target table.

**The use of substitute expressions may cause data inconsistencies.**

Consider the following, contrived, case of multiplying a value by two before applying it:

```sql
UPSERT
INTO _cdc_sink.apply_config (target_db, target_schema, target_table, target_column, expr)
VALUES ('some_db', 'public', 'my_table', 'value', '2 * $0');
```

The substitution string `$0` will expand at runtime to the value of the incoming data. Expressions
may repeat the `$0` expression. For example `$0 || $0` would concatenate a value with itself.

In some logical-replication cases, it may be desirable to entirely discard the incoming mutation and
replace the value:

```sql
UPSERT
INTO _cdc_sink.apply_config (target_db, target_schema, target_table, target_column, expr)
VALUES ('some_db', 'public', 'my_table', 'did_migrate', 'true');
```

In cases where a column exists only in the target database, using a `DEFAULT` for the column is
preferable to using a substitute expression.
