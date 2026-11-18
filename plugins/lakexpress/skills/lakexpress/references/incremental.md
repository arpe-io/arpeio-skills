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

Each export has a lower bound `d_old` (the previous watermark, none on the first run) and an upper
bound `d_new`, read from the source at the start of the run:

| Case | Upper bound `d_new` |
|---|---|
| Any type, no safety lag | `MAX(column)` |
| `date`, `datetime` with `--incremental_safety_lag` | `min(MAX(column), now − lag)` |
| Empty table, `date` / `datetime` | `now − lag` |
| Empty table, `int` / `bigint` / `auto` | none: the table is skipped for the run (watermark unchanged) and exported once it has rows. The run is not failed by it (exit 0 if the rest succeeds) and the other tables are exported and published; the target table is created by the first run that has rows |

- `now` is the **source's** clock (`CURRENT_TIMESTAMP`), since the column holds its values; UTC with a
  warning if it cannot be read.
- First run (no watermark): `WHERE column <= d_new` (all existing rows).
- Next runs: `WHERE column > d_old AND column <= d_new` (`>=` with `:i`); `d_old` is the previous `d_new`.
- Without a lag, a row committed later with a value lower than or equal to the watermark is never
  exported (long transactions, timestamp set before the commit). Set `--incremental_safety_lag` to the
  longest delay between a row's timestamp and its commit: rows newer than `now − lag` wait for the
  next run. For `int` / `bigint` columns, choose one that only grows as rows are written.
- The upper bound never goes below the stored watermark (e.g. after raising the lag): the table then
  exports no rows and its watermark stays.
- A `MAX(column)` query that fails (unknown column, missing privilege) is an error of that table,
  counted as a failed table (exit 40 or 41); it is not treated as an empty table.
- One table whose export job cannot be created does not drop the others; the run then exits 43.
- SQL Server behind an Always On listener: `MAX(column)` and the export use the same
  `application_intent`. With `"ReadOnly"` and several secondaries, add a lag larger than the replica
  lag, or point `server` at one secondary.
- Before 0.6.2 the lag was applied to empty tables only.
- Before 0.6.8 an empty table with an `int` / `bigint` / `auto` column stopped job creation
  (`d_new must be set before getting folder name`): the tables after it in the same schema were not
  exported and the run still exited 0.

## When the watermark moves

| Configuration | The watermark of a table moves… |
|---|---|
| No publish target | when the table is exported |
| With a publish target | when the table is **published** |

- With a publish target, a run that was exported but not delivered (publish error, publish skipped
  after an export error, interrupted run) leaves the watermark where it was: the next run reads the
  same rows again from the source. Nothing is lost and no reset is needed.
- After a failed publish, only the tables that were not published are read again on Snowflake. On the
  other targets, when some tables fail, every incremental table of the run is read again: with
  `append`, rows of the tables that were published can then be loaded twice (`upsert` and
  `delete_insert` are not affected).
- `'sync[export]'` then `'sync[publish]'`: each `'sync[export]'` starts from the last **published**
  point, so two exports without a publish in between write folders that overlap
  (`incremental_from_<d_old>_to_<d_new>` with the same `<d_old>` and a larger `<d_new>`). Publish the
  latest completed run, which `'sync[publish]'` does by default: it holds every row since the last
  publish. The folders of the runs that were not published stay in storage and are not used.
- A watermark that does not advance: no new rows (`SELECT MAX(column)`), or the table was not
  published (check the publish step and its failed tables).
- Before 0.6.8 the watermark moved at export: the rows of a run that was never published (publish
  error, two exports before one publish, crash) were lost until a new baseline.

## Loading strategies (what the publish step does with the new rows)

| Strategy | Does | Primary key |
|---|---|---|
| `append` | inserts the new rows | not needed |
| `upsert` | `MERGE`: updates matching rows, inserts the others (MotherDuck, DuckLake: delete by key + insert) | required |
| `delete_insert` | deletes target rows whose key is in the batch, inserts the batch | required |

- `append`: the baseline run replaces the target table; each later run adds its rows to it. On
  Snowflake (0.6.8+) this holds for both publish methods: `internal` loads the run into the existing
  table, `external` registers the files of each published run in the external table. Publishing the
  same run twice adds nothing. Before 0.6.8 a Snowflake `append` table held only the last run.
- `upsert` / `delete_insert`: the key is the primary key found on the source table at schema
  discovery. Before 0.6.8 the publish stopped with exit 46 (`requires primary key columns`) even when
  the table had one.

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

- Every LakeXpress DB backend works since 0.6.2; before, use PostgreSQL, SQL Server or SQLite (DuckDB
  did not save the incremental tables).
- Duplicates after re-running: use `!upsert` (where supported) or deduplicate at query time.
- To track both inserts and updates on different columns, use two configurations.
- `--log_level DEBUG` on `sync` shows the bounds used per table.
