# LakeXpress – Troubleshooting and known 0.6.0 behaviours

## Known behaviours to plan around

| Behaviour | What to do |
|---|---|
| `config create` accepts a configuration with no `--output_dir` / `--target_storage_id`; every `sync` then fails (exit 10, `No output destination specified`) | Always give one of them; otherwise `config delete --confirm` and recreate |
| Publish patterns are checked by `sync`, not by `config create` (exit 10, `Table pattern must include {table}` / `Invalid tokens … {table}`) | Table pattern contains `{table}`; schema pattern never does (all targets, not only Glue) |
| An invalid `--incremental_table` (e.g. type `timestamp`) exits 10 but the configuration is already saved, without incremental tables | Fix the spec, `config delete --sync_id … --confirm`, create again |
| DuckDB LakeXpress DB: `--incremental_table` entries are not saved (`Failed to create incremental table config: … 'lastrowid'`), `config create` still exits 0; the tables are then fully exported each run | Use a PostgreSQL, SQL Server or SQLite LakeXpress DB for incremental syncs |
| After a run with failed tables (exit 40), `sync` publishes the **latest completed run**, i.e. an earlier run if there is one (or nothing: `No completed export run found for publish`) | `sync --resume` first (it publishes once the run completes), or publish a given run: `'sync[publish]' --run_id <RUN_ID>` |
| A run that crashed stays `running` and blocks the next ones (`… is already running`, exit 10) | `cleanup --sync_id … --status running` (try `--dry-run` first) |
| Incremental locks left by a crash block the tables | `lxdb locks`, then `lxdb release-locks --confirm` (`--max_age_hours`, `--table_id`) |
| `--error_action fail` (default) stops at the first failed table | `--error_action continue` to export the rest, then `--resume` |
| No table matches the schema / table filters: exit 0, warning `No tables found to export` | Check `--source_schema_name` (case, patterns) and the source permissions |
| A configuration cannot be changed | `config delete --confirm` + `config create`; incremental watermarks start again |
| `--sub_path "daily/$(date +%Y%m%d)"` on `config create` is evaluated once and stored: every later sync writes to that same folder | For dated publish names use `{date}` / `{timestamp}` in the publish patterns; for a new storage folder per day, create a configuration per day |
| `--fastbcp_table_config` entries without a valid integer degree are silently ignored | Always end with `:<degree>` (`orders:Ctid::4`) |
| `$env{VAR}` not set anywhere in the file stops every command (exit 10), even for unused entries | Set all referenced variables, or split the file |
| SQLite / DuckDB LakeXpress DB with a relative `filepath` | Use an absolute path; otherwise each working directory gets its own empty DB |
| SQL Server LakeXpress DB or source with a self-signed certificate: `pyodbc` TLS fails and LakeXpress falls back to `pymssql` (classic auth only) | Add `"trust_server_certificate": true` to `info` |
| Relative `--output_dir` before 0.4.9 was resolved from FastBCP's `engine/` folder | 0.4.9+ saves it as an absolute path; with older versions pass an absolute path |
| Fabric entry without `"fabric_target": "lakehouse"` uses the Fabric **Warehouse** publisher | Add it for the Lakehouse |
| Snowflake external tables don't see new files | `ALTER EXTERNAL TABLE … REFRESH;`; the storage `directory` must equal the stage URL |
| `--generate_metadata` with OneLake storage can write wrong paths in the CDM files | Check them, or omit the flag |
| DuckLake publishes tables one at a time, whatever `--n_jobs` | Expected |
| Incremental (query) export of a large table with `--fastbcp_p` > 1: a physical method (`Ctid`, `Rowid`, `Physloc`) is replaced by `None` with a warning (0.5.0+; before, the export failed) | For parallel incremental exports use a key-based method (`Ntile`, `RangeId`) in `--fastbcp_table_config` |

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
| Exit 34 | FastBCP older than 0.30.4: use the bundled one |
| FastBCP license error in the job error (FastBCP exit 99/100) | License issued before 2026-10-26 (no FastBCP block) and no `FastBCP.lic`: request an updated license |
| ADBC source (`use_adbc: true`) stops with exit 13 | The license has no ADBC driver block: request an updated license, or set `ARPEIO_ADBC_LICENCE_FILE` |
| Every command exits 0 even on failure | LakeXpress before 0.6.0: read the log, or upgrade |
| `auth_mode: connection_string` source: FastBCP `Unknown argument found` (before 0.5.0) | Fixed in 0.5.0; or use `classic` |
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
| Snowflake: files not found | Storage `directory` ≠ stage URL |
| BigQuery `Dataset not found` | `location` of the entry vs bucket region |
| Databricks external tables fail | Missing storage credential / external location for the S3 path |
| MotherDuck `No files found that match the pattern` / `Missing or invalid credentials` | The storage entry used to create the `lakexpress_storage` secret is wrong (profile not resolvable…); test `SELECT count(*) FROM read_parquet('s3://…/*.parquet')` |
| Fabric `AADSTS7000215` | Client secret wrong or expired |
| `upsert` / `delete_insert` rejected at publish | Target not supported (Glue, Fabric Lakehouse), `--publish_method external` on Databricks / BigQuery, or no primary key on the source table |
| Data added to the source shows up one or more syncs later | `--incremental_safety_lag` too large |
| Duplicates after re-running an incremental sync | `append` adds rows: use `!upsert` where supported, or deduplicate in queries |
