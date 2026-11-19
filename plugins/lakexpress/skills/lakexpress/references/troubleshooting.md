# LakeXpress – Troubleshooting and known 0.6.8 behaviours

## Known behaviours to plan around

| Behaviour | What to do |
|---|---|
| `config create` accepts a configuration with no `--output_dir` / `--target_storage_id`; every `sync` then fails (exit 10, `No output destination specified`) | Always give one of them; otherwise `config delete --confirm` and recreate |
| Publish patterns are checked by `sync`, not by `config create` (exit 10, `Table pattern must include {table}` / `Invalid tokens … {table}`) | Table pattern contains `{table}`; schema pattern never does (all targets, not only Glue) |
| An invalid `--incremental_table` (e.g. type `timestamp`) exits 10 but the configuration is already saved, without incremental tables | Fix the spec, `config delete --sync_id … --confirm`, create again |
| DuckDB LakeXpress DB before 0.6.2: `--incremental_table` entries are not saved (`Failed to create incremental table config: … 'lastrowid'`), `config create` still exits 0; the tables are then fully exported each run | Fixed in 0.6.2; before, use a PostgreSQL, SQL Server or SQLite LakeXpress DB for incremental syncs |
| After a run with failed tables (exit 40), `sync` publishes the **latest completed run**, i.e. an earlier run if there is one (or nothing: `No completed export run found for publish`) | `sync --resume` first (it publishes once the run completes), or publish a given run: `'sync[publish]' --run_id <RUN_ID>` |
| A run that crashed stays `running` and blocks the next ones (`… is already running`, exit 10) | `cleanup --sync_id … --status running` (try `--dry-run` first). Before 0.6.7 `cleanup` failed with `RUN_DELETE_FAILED` on a run that had exported at least one file: upgrade |
| Incremental locks left by a crash block the tables | `lxdb locks`, then `lxdb release-locks --confirm` (`--max_age_hours`, `--table_id`) |
| `--error_action fail` (default) stops at the first failed table | `--error_action continue` to export the rest, then `--resume` |
| No table matches the schema / table filters: exit 44 and the run is recorded as `failed` (exit 0 before 0.6.2; in 0.6.0 the run also stayed `running` and blocked the next ones until `cleanup`) | Check `--source_schema_name` (case, patterns) and the source permissions |
| A configuration cannot be changed | `config delete --confirm` + `config create`; incremental watermarks start again |
| `--sub_path "daily/$(date +%Y%m%d)"` on `config create` is evaluated once and stored: every later sync writes to that same folder | For dated publish names use `{date}` / `{timestamp}` in the publish patterns; for a new storage folder per day, create a configuration per day |
| `--fastbcp_table_config` entries without a valid integer degree are silently ignored | Always end with `:<degree>` (`orders:Ctid::4`) |
| `--fastbcp_table_config` with an empty method (`orders:::4`) exported the table sequentially before 0.6.2 | 0.6.2+ keeps the automatic method with the given degree; before, name the method |
| SQL Server, PostgreSQL and Teradata are extracted through ADBC since 0.6.3: some Parquet types change (timestamps with time zone become UTC timestamps, microseconds are kept, `jsonb` is text) | Expected (export.md); `"use_adbc": false` on the source keeps the native driver |
| Oracle → Snowflake column types changed in 0.6.5 (`DATE` → `TIMESTAMP_NTZ(0)`, `TIMESTAMP(n)` → `TIMESTAMP_NTZ(n)`, `NUMBER` without precision → `NUMBER(38,18)`…). A full publish recreates its tables; a table kept across runs (incremental or row-level strategies) keeps its old types | Drop the table, or publish to a new schema, before the first run with 0.6.5 |
| Oracle `NUMBER` without precision published to Snowflake: digits beyond the 18th decimal place are rounded, and values of 10^20 or more are rejected | `--unbounded_number_type varchar` or `decimal_strict` with `"use_adbc": true` (publishing.md) |
| Incremental sync with a publish target: a table that was not published keeps its watermark (0.6.8+), so the next run exports the same rows again; two `'sync[export]'` without a publish in between write overlapping folders | Expected: nothing is lost. Publish the latest run (`'sync[publish]'` default). Before 0.6.8 the watermark moved at export and the rows of an unpublished run were lost: upgrade, then reload the affected tables with a new baseline (incremental.md) |
| Empty table with an `int` / `bigint` / `auto` watermark: skipped for the run, exported once it has rows (0.6.8+). Before: `d_new must be set before getting folder name`, the tables after it in the same schema were not exported, and the run exited 0 | Upgrade to 0.6.8+ |
| Incremental tables published before 0.6.8: `upsert` / `delete_insert` stopped the publish with exit 46 (`requires primary key columns`) even when the table had a primary key (every target). On Snowflake, an `append` table held only the rows of the last run, and with `--publish_method internal` its load failed with `unexpected 'BINARY_AS_TEXT'` | Upgrade to 0.6.8+, then reload the affected tables with a new baseline (incremental.md, "Inspecting and resetting watermarks") |
| Snowflake before 0.6.6: values loaded with `COPY INTO` were shifted by the session time zone (Oracle `DATE` `1992-01-01 00:00:00` loaded as `1991-12-31 16:00:00` with `America/Los_Angeles`) | 0.6.6+ opens every session in UTC. Publish again to correct the values: a full publish recreates its tables; drop an incremental table and reload it with a new baseline (incremental.md, "Inspecting and resetting watermarks"). Before 0.6.6, `ALTER USER … SET TIMEZONE = 'UTC'` |
| With `--incremental_safety_lag`, before 0.6.2 the lag was only applied to empty tables: rows committed late with a value near `MAX(column)` could be lost | Upgrade to 0.6.2+ |
| SQL Server behind an Always On listener with read-only routing, before 0.6.2: the export read a secondary while the watermark came from the primary, and rows could be skipped | 0.6.2+: both use `application_intent` (default `ReadWrite` = primary) |
| `$env{VAR}` not set anywhere in the file stops every command (exit 10), even for unused entries | Set all referenced variables, or split the file |
| SQLite / DuckDB LakeXpress DB with a relative `filepath` | Use an absolute path; otherwise each working directory gets its own empty DB |
| SQL Server LakeXpress DB or source with a self-signed certificate: `pyodbc` TLS fails and LakeXpress falls back to `pymssql` (classic auth only) | Add `"trust_server_certificate": true` to `info` |
| Relative `--output_dir` before 0.4.9 was resolved from FastBCP's `engine/` folder | 0.4.9+ saves it as an absolute path; with older versions pass an absolute path |
| Fabric entry without `"fabric_target": "lakehouse"` uses the Fabric **Warehouse** publisher | Add it for the Lakehouse |
| Snowflake external tables don't see new files | `ALTER EXTERNAL TABLE … REFRESH;`. Not for an incremental `append` table (0.6.8+): LakeXpress registers the files of each published run, and a full refresh would also register runs that were exported but not published |
| Snowflake publishing stops before creating any table, with an error naming the stage URL and the storage `directory` (0.6.2+) | Make them the same folder (`DESC STAGE …`); a `directory` that is a parent of the stage URL is not accepted. `gcs://` / `azure://` on the Snowflake side match `gs://` / `abfss://` |
| `--generate_metadata` with OneLake storage can write wrong paths in the CDM files | Check them, or omit the flag |
| DuckLake publishes tables one at a time, whatever `--n_jobs` | Expected |
| Incremental (query) export of a large table with `--fastbcp_p` > 1, 0.5.0 to 0.6.8: a physical method (`Ctid`, `Rowid`, `Physloc`) is replaced by `None` with a warning, so the table is read by a single thread on every run, the baseline included (before 0.5.0 the export failed) | Use a key-based method (`Ntile`, `RangeId`) in `--fastbcp_table_config`. Versions after 0.6.8 keep the physical method (they pass `--sourceschema` / `--sourcetable` with `--query`); on Oracle the incremental table then needs `SELECT_CATALOG_ROLE` like a full export |
| Oracle, large table: `Error in retrieving RowID Chunks. The user must have SELECT_CATALOG_ROLE` with `ORA-00942` on `SYS.DBA_EXTENTS`; the table fails (exit 40 / 41) | The `Rowid` method reads the table's extents from `DBA_EXTENTS`, `DBA_SEGMENTS` and `DBA_OBJECTS` to build its `ROWID` ranges. `GRANT SELECT_CATALOG_ROLE TO <user>` (or `SELECT` on these three views), or avoid `Rowid`: `--fastbcp_table_config "SCHEMA.TABLE:None::1"`, a key-based method, or `--fastbcp_p 1` |

## Doc errors (found in the 0.4 docs against the 0.4.10 code)

| Docs say | Actually |
|---|---|
| Resume recipe: `SELECT … FROM jobs WHERE status = 'failed'` (columns `status`, `error_message`, `finished_at`) | Table `export_jobs` (`job_status`, `completed_at`); errors are in `export_log.error_message` (environment.md) |
| `status`: `--sync_id` and `--run_id` both optional | One of them is required (exit 10) |
| `config create`: at least one of `--output_dir` / `--target_storage_id` "is required" | Only checked by `sync` (see above) |
| `--run_id` is the run to resume "with `--resume`" | Optional: without it, the latest incomplete run is resumed |
| Glue page: the `{table}` restriction on the schema pattern is Glue-specific | It applies to every target |
| Fabric: `fabric_target` "omitted or set to anything else" → Warehouse | Omitted or `"warehouse"` → Warehouse; use exactly `"lakehouse"` for the Lakehouse |
| Azure recipe: `"directory": "abfss://datalake.dfs.core.windows.net/exports"` | Needs the container: `abfss://<container>@<account>.dfs.core.windows.net/exports` |
| Daily snapshot recipes put `$(date +%Y%m%d)` in `--sub_path` of `config create` | The value is fixed when the configuration is created |
| 0.6 docs, `config create` page: `--pk_constraints` "not applied on BigQuery" | Applied since 0.6.2 (`PRIMARY KEY (...) NOT ENFORCED` on the native tables LakeXpress creates), as the BigQuery page says |

## Common errors

| Symptom | Cause / fix |
|---|---|
| `the following arguments are required: -a/--auth_file, --lxdb_auth_id` (exit 2) | Every command needs both, placed after the command |
| `unrecognized arguments` (exit 2) | Option before the command, or a `config create` option given to `sync` (`sync` only takes `--sync_id`, `--resume`, `--run_id`, `--fastbcp_dir_path` + common options) |
| `Connection 'x' not found in credentials file. Available connections: [...]` (exit 10) | Wrong id; the message lists the keys |
| `Missing required key 'server'` / `'database'` / `'filepath'` (exit 10) | LakeXpress DB entry incomplete (credentials.md) |
| `Sync 'x' already exists` (exit 10) | Choose another `--sync_id` or delete the old configuration |
| `Invalid sync_id format` (exit 10) | 1–64 letters, digits, `_`, `-` |
| `Invalid incremental type 'timestamp'` (exit 10) | Use `datetime` (or `date`, `int`, `bigint`, `auto`) |
| `No LakeXpress licence found. Get a free 30-day trial licence at https://arpe.io…` (exit 13) | Save the emailed license as `~/.lakexpress/license.lic` / `%APPDATA%\LakeXpress\license.lic`, or pass `--license_file` (environment.md) |
| License expired (exit 13) | Trial over (30 days) or subscription ended: a new license file from arpe.io, same place; no reinstall |
| `FastBCP directory path is required for export operations` / `FastBCP executable not found in …` (exit 30; 10 before 0.5.0) | Keep `engine/` next to the binary, or pass `--fastbcp_dir_path` (or `sync --fastbcp_dir_path`) |
| Exit 34 | FastBCP older than 1.4.0 (`--fastbcp_dir_path` pointing to an old FastBCP): use the bundled one |
| Exit 13 during the export, FastBCP license error (FastBCP exit 99/100) | License issued before 2026-10-26 (no FastBCP block) and no `FastBCP.lic`: request an updated license, then `sync --resume` |
| A SQL Server, PostgreSQL, Teradata (or Oracle with `use_adbc: true`) source stops with exit 13 before the export | The license has no ADBC driver block: request an updated license, set `ARPEIO_ADBC_LICENCE_FILE`, or set `"use_adbc": false` on the source |
| `use_adbc` rejected (exit 10) | It must be the JSON boolean `true` or `false`, next to `ds_type` |
| `--unbounded_number_type … is only supported with a Snowflake publish target` (exit 10) | `varchar` and `decimal_strict` need `--publish_target` to be a Snowflake entry |
| `--unbounded_number_type decimal_strict needs the ADBC driver` (exit 10) | Set `"use_adbc": true` on the Oracle source |
| Oracle export fails naming a column and a value, with `decimal_strict` | The value does not fit `NUMBER(38,18)` (e.g. `1/3`, `1E-20`): use `varchar`, or `decimal` to let Snowflake round |
| SQL Server export fails on the server certificate after upgrading to 0.6.2+ | The export now uses the entry's TLS settings: `"trust_server_certificate": true` for a self-signed certificate |
| Teradata export through ADBC cannot connect | No TLS on port 443 and no TTU client: install TTU (or set `ARPETERADATA_TTU_PATH`), or `"use_adbc": false` |
| Databricks `auth_mode: "oauth"` rejected | Use `"token"` (a service principal's token goes in `access_token`) |
| Redshift publishing stops: no `iam_role` | Add the role ARN to the Redshift entry (before 0.6.2 it created empty tables and reported success) |
| Exit 41 / 46 / 47 | Every table failed (nothing published) / publishing failed after a good export (`'sync[publish]'` again) / both had failures (`sync --resume`) |
| Exit 43 (0.6.8+) | The export job of at least one table could not be created (the log names it); the other tables were exported, the run is recorded as `failed` and nothing is published. Fix the cause and run `sync` again: incremental tables are exported again from their watermark, which did not move |
| `cleanup` fails with `RUN_DELETE_FAILED`, and the run keeps blocking the next ones (before 0.6.7) | Upgrade to 0.6.7+ |
| Exit 45 | `--resume` with no incomplete run or an unknown `--run_id`; `'sync[publish]'` with no completed run |
| Every command exits 0 even on failure | LakeXpress before 0.6.0: read the log, or upgrade |
| `auth_mode: connection_string` source: FastBCP `Unknown argument found` (before 0.5.0), or the export rejects the string (before 0.6.2) | 0.6.2+ converts the string for FastBCP (`info.fastbcp_connection_string` to set it yourself); or use `classic` |
| `--log_dir` ignored, logs in the current directory (before 0.6.2) | Fixed in 0.6.2 |
| Publishing with a PostgreSQL LakeXpress DB logs `column "table_created" is of type boolean but expression is of type integer` (0.6.0) | Fixed in 0.6.1; the data was published |
| Teradata schema pattern (`SALES%`) matches nothing (before 0.5.0) | Fixed in 0.5.0; or list the databases |
| SQLite LakeXpress DB: every table exported single-threaded whatever `--fastbcp_p` (before 0.5.0) | Fixed in 0.5.0 |
| `Failed to connect to source database` (exit 12) | Host, port, `database` (Oracle: service name), firewall, credentials; `--log_level DEBUG` |
| PostgreSQL LakeXpress DB: `permission denied for schema public` | Make the user owner of the database (`ALTER DATABASE lakexpress_log OWNER TO …`) or `GRANT CREATE ON SCHEMA public` |
| SQL Server LakeXpress DB: `User does not have permission to perform this action. (15247)` | Add the user to `db_owner` in that database |
| MySQL LakeXpress DB: `Access denied … to database` (1044) | Database missing, wrong name, or missing `GRANT` |
| SQL Server `Incorrect syntax near '?'` (0.4.7 and earlier, `pymssql` path) | Fixed in 0.4.8; or set `trust_server_certificate` to use ODBC |
| Linux binary: `libcrypto.so.3: version 'OPENSSL_3.3.0' not found` (0.4.5–0.4.6) | Upgrade to 0.4.7+ |
| Oracle `ORA-12154` / `ORA-28000` / thick-mode library not found | Service name / account locked (`ALTER USER … ACCOUNT UNLOCK`) / `lib_dir` and `LD_LIBRARY_PATH`; 12.1+: remove `lib_dir` (thin mode) |
| SAP HANA export fails in FastBCP | Set `HDBDOTNETCORE` to the runtime folder |
| Snowflake: `no file of '<folder>' is registered in external table …` (0.6.8+) | An external `append` table created by an earlier version points at a single run folder and cannot take another run: drop the table and run the sync again; export a new baseline to rebuild it in full |
| Incremental watermark does not advance | No new rows (`SELECT MAX(column)` equals the watermark), or, with a publish target (0.6.8+), the table was not published: check the publish step and its failed tables |
| An incremental table fails before its export with `Failed to query MAX(…)` (0.6.8+; counted in exit 40 / 41) | The watermark column does not exist on that table or cannot be read: fix the `--incremental_table` spec (`config delete --confirm`, create again). Before 0.6.8 the table was treated as empty |
| Windows: every table export fails with `[WinError 2]` (before 0.6.6) | 0.6.6+ launches FastBCP through its absolute path; before, start LakeXpress from inside its `engine\` folder |
| Snowflake: files not found (before 0.6.2) | Storage `directory` ≠ stage URL; 0.6.2+ stops with an error instead |
| BigQuery `Dataset not found` | `location` of the entry vs bucket region |
| Databricks external tables fail | Missing storage credential / external location for the S3 path |
| MotherDuck `No files found that match the pattern` / `Missing or invalid credentials` | The storage entry used to create the `lakexpress_storage` secret is wrong (profile not resolvable…); test `SELECT count(*) FROM read_parquet('s3://…/*.parquet')` |
| Fabric `AADSTS7000215` | Client secret wrong or expired |
| `upsert` / `delete_insert` rejected at publish | Target not supported (Glue, Fabric Lakehouse), `--publish_method external` on Databricks / BigQuery, or no primary key on the source table. Before 0.6.8 it was also rejected (`requires primary key columns`, exit 46) when the table had a primary key: upgrade |
| Data added to the source shows up one or more syncs later | `--incremental_safety_lag` too large |
| Duplicates after re-running an incremental sync | `append` adds rows: use `!upsert` where supported, or deduplicate in queries |
