# LakeXpress – Incremental syncs

A table listed with `--incremental_table` exports only the rows past its **watermark**; every other table
of the configuration is fully exported on each sync. Watermarks live in the LakeXpress DB.

## Spec

```text
--incremental_table "schema.table:column:type[:i|:e][@start_value][!strategy]"
```

Repeat the flag for each table (`config create` only).

| Part | Values |
|---|---|
| `schema.table` | Source table |
| `column` | Watermark column (index it; ideally set on insert and never updated) |
| `type` | `date`, `datetime` (also for timestamp columns), `int`, `bigint`, `auto` (numeric, from `MAX(column)`). **Not** `timestamp` or `integer` |
| `:e` / `:i` | Lower bound `column > watermark` (`:e`, default) or `column >= watermark` (`:i`). Not an include/exclude switch |
| `@start_value` | Initial watermark instead of a full first export, e.g. `@2025-01-01` |
| `!strategy` | `append` (default), `upsert`, `delete_insert` |

Examples:

```bash
--incremental_table "sales.orders:o_orderdate:date"
--incremental_table "sales.orders:created_date:date:i@2025-01-01!upsert"
--incremental_table "events.pageviews:event_time:datetime!append"
--incremental_table "sales.orders:order_id:bigint"
--incremental_safety_lag 3600
```

An invalid spec (`Invalid incremental type …`) makes `config create` exit 10 **after** the configuration
was saved, without its incremental tables: `config delete --sync_id … --confirm` and create it again.

## How the bounds are computed

| Column type | Upper bound `d_new` |
|---|---|
| `date`, `datetime` | run start time − `--incremental_safety_lag` seconds (not `MAX(column)`) |
| `int`, `bigint`, `auto` | `MAX(column)` in the source |

- First run (no watermark): `WHERE column <= d_new` (all existing rows).
- Next runs: `WHERE column > d_old AND column <= d_new` (`>=` with `:i`); `d_old` is the previous `d_new`.
- The safety lag holds back the most recent rows until the next run (late-arriving data); it does not
  look further back. Rows arriving later with an older timestamp than `d_old` are missed.
- The watermark is updated only for tables that exported successfully.

## Loading strategies (what the publish step does with the new rows)

| Strategy | Does | Primary key |
|---|---|---|
| `append` | inserts the new rows | not needed |
| `upsert` | `MERGE`: updates matching rows, inserts the others (MotherDuck, DuckLake: delete by key + insert) | required |
| `delete_insert` | deletes target rows whose key is in the batch, inserts the batch | required |

Support: `upsert` / `delete_insert` on Snowflake, Redshift, MotherDuck, DuckLake (as persistent internal
tables whatever `--publish_method`), Databricks and BigQuery (with `--publish_method internal`), Fabric
Warehouse. Not on Fabric Lakehouse or AWS Glue. A pre-flight check fails the publish with the table name
when a strategy is unsupported or the table has no primary key. Without publishing, the strategy has no
effect: each run writes the new rows' files.

| Scenario | Strategy |
|---|---|
| Append-only events, logs, metrics, transactions | `append` |
| Master data, order status, CDC on `updated_at` | `upsert` |
| Rows rewritten as a whole by the source | `delete_insert` |

## Locks

Incremental syncs lock their tables in the LakeXpress DB so that two runs don't move the same watermark.
A crashed or killed sync leaves locks that block the next runs:

```bash
./LakeXpress lxdb locks --auth_file credentials.json --lxdb_auth_id lxdb_postgres
./LakeXpress lxdb release-locks --auth_file credentials.json --lxdb_auth_id lxdb_postgres --max_age_hours 2 --confirm
./LakeXpress lxdb release-locks --auth_file credentials.json --lxdb_auth_id lxdb_postgres --table_id 42 --confirm
```

## Inspecting and resetting watermarks

Tables in the LakeXpress DB: `incremental_tables` (one row per configured table), `incremental_watermarks`
(current `last_exported_value` = next `d_old`, `d_new`, `last_export_at`, `rows_exported`),
`incremental_export_history` (one row per run: `d_old`, `d_new`, `export_mode` `baseline` /
`incremental`, `status`).

```sql
SELECT t.source_schema, t.source_table, t.incremental_column, t.incremental_type, t.strategy,
       w.last_exported_value, w.d_new, w.last_export_at, w.rows_exported
FROM incremental_tables t
LEFT JOIN incremental_watermarks w ON w.incremental_table_id = t.incremental_table_id
WHERE t.sync_id = 'YOUR_SYNC_ID';
```

No `--reset-watermarks` flag. To reload everything: `config delete --confirm` and recreate the
configuration (new watermarks). For one table (manual): delete its row from `incremental_watermarks`; the
next sync exports it as a new baseline.

## Tips

- Use a PostgreSQL, SQL Server or SQLite LakeXpress DB (with DuckDB, 0.4.10 does not save the incremental
  tables).
- Duplicates after re-running: use `!upsert` (where supported) or deduplicate at query time.
- To track both inserts and updates on different columns, use two configurations.
- `--log_level DEBUG` on `sync` shows the bounds used per table.
