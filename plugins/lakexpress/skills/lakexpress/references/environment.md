# LakeXpress – Install, license, FastBCP, LakeXpress DB, logging, exit codes

## Install

- Native binaries: `LakeXpress` (Linux x64, `chmod +x LakeXpress`) and `LakeXpress.exe` (Windows x64).
  One package per platform, `LakeXpress-Full-linux-x64.zip` / `LakeXpress-Full-win-x64.zip`, for the
  trial and for subscriptions (the license file makes the difference). Trial:
  https://arpe.io/get-trial?product=LakeXpress emails the download link and a 30-day license file.
  Unzip anywhere, then save the license (see License).
- `./LakeXpress --version` → e.g. `LakeXpress 0.6.4 (build 2026-11-11T08:27:18Z)`. `./LakeXpress --help`,
  `./LakeXpress config create --help`.
- Archive: the binary, `README.txt`, SBOM files (CycloneDX JSON/XML + SHA-256) and
  `engine/FastBCP[.exe]`.
- Install guide: https://lakexpress-docs.arpe.io/latest/installation

## FastBCP

- Bundled in `engine/` (FastBCP 1.6.8 in LakeXpress 0.6.5; 1.4.0 in 0.6.0 to 0.6.4), found automatically. The LakeXpress license
  licenses it: LakeXpress passes FastBCP its license blocks with `--license` (masked in the log, private
  temporary file removed at the end of the run). No `FastBCP.lic` needed.
- Another FastBCP: `config create --fastbcp_dir_path <folder>` (saved in the configuration; `sync
  --fastbcp_dir_path` overrides it).
- Checked before each export (`sync`, `'sync[export]'`, `run`; not `'sync[publish]'`): not configured or no
  executable in the folder → exit 30 (`FastBCP directory path is required for export operations` /
  `FastBCP executable not found in …`); older than 1.4.0 → exit 34 (0.30.4 before 0.6.2); another
  version than the bundled 1.6.8, older or newer, or an unreadable one → warning only.
- FastBCP before 1.4.0 exported `Timepartition` tables incorrectly (rows with a NULL date skipped, some
  rows exported twice with `week`), hence the minimum.
- `--unbounded_number_type decimal_strict` needs the bundled 1.6.8.
- `--quiet_fbcp` keeps FastBCP output out of the LakeXpress log.

## License

Lookup order:

1. env `LAKEXPRESS_LICENSE` (license text; for CI/containers)
2. `--license "<text>"`
3. `--license_file <path>`
4. `~/.lakexpress/license.lic` (Linux), `%APPDATA%\LakeXpress\license.lic` (Windows), `./license.lic`

No license, or an invalid or expired one: exit 13 (`No LakeXpress licence found. Get a free 30-day trial
licence at https://arpe.io…`). `--help`, `--version` and command help work without one. There is no
build-date trial since 0.6.0: the free trial is a 30-day license file.

The license file also carries the FastBCP and ADBC driver blocks: keep it whole, as delivered.

- License issued before 2026-10-26 (no FastBCP block): warning, FastBCP uses its own `FastBCP.lic`.
- Source extracted through ADBC (the default for SQL Server, PostgreSQL and Teradata; Oracle with
  `"use_adbc": true`) and no ADBC block: exit 13 before the export. Without an updated license file,
  give the driver license with `ARPEIO_ADBC_LICENCE` (text) or `ARPEIO_ADBC_LICENCE_FILE` (path), or
  with `arpeio.adbc.license_file=<path>` in `info.fastbcp_connection_string` (SQL Server, PostgreSQL;
  not in `info.connection_string`); or set `"use_adbc": false`.
- FastBCP rejects its license (FastBCP exit 99/100): exit 13; the run stops at the first job that
  fails this way, the incremental watermarks do not move, and `sync --resume` exports the remaining
  tables once the license is fixed. Ask for an updated license.

## LakeXpress DB

Created by `lxdb init` or on first use, in `public` (PostgreSQL) / `dbo` (SQL Server). Tables:

| Table | Content |
|---|---|
| `export_syncs` | Sync configurations (`config create`) |
| `export_runs` | One row per run: `run_id`, `sync_id`, status (`running`, `completed`, `failed`), totals |
| `export_jobs` | One row per table and run: `source_schema`, `source_table`, `job_status`, `num_rows_exported`, `num_files`, FastBCP method / degree / key, `d_old` / `d_new`, `started_at`, `completed_at` |
| `export_files` | Each Parquet file: path, size, rows |
| `export_columns` | Column metadata per job (source type, precision, Parquet name) |
| `export_log` | Events: `log_level`, `event_message`, `error_code`, `error_message`, source table |
| `export_steps` | Workflow steps |
| `partition_columns` | Persistent per-table FastBCP settings (export.md) |
| `incremental_tables`, `incremental_watermarks`, `incremental_export_history` | Incremental configuration and watermarks (incremental.md) |
| `publish_runs`, `publish_jobs` | Publishing history |

Failed tables of a run:

```sql
SELECT j.source_schema, j.source_table, j.job_status, l.error_message
FROM export_jobs j
LEFT JOIN export_log l ON l.job_id = j.job_id AND l.log_level = 'ERROR'
WHERE j.run_id = '<RUN_ID>' AND j.job_status = 'failed';
```

## Logging

- Console + file `lx_YYYYMMDD_HHMMSS_<run_id>.log` in `--log_dir` (default: current directory), and
  `lx_YYYYMMDD_HHMMSS_<run_id>.yml` (copy of the configuration, usable with `run`). `'sync[publish]'`
  writes a log file for the publish too (0.6.2+).
- Format: `[YYYY-MM-DD HH:MM:SS.fff+TZ :: LEVEL :: PID :: process] message`.
- `--log_level DEBUG` adds SQL queries, file operations, watermark bounds.
- The run id is printed at start (`Created export run: <run_id>`); the end shows `RUN SUMMARY` (or
  `RESUME SUMMARY`) `Export: n/m tables succeeded`, the failed tables and their errors.

## Exit codes (0.6.5)

Before 0.6.0 the binary always exited 0: check the log with older versions. Codes 41 and 44–47 are
returned since 0.6.2 (before: 40, 0, 10 or 1 in those situations).

| Code | Meaning |
|---|---|
| 0 | Success. Also: `'sync[publish]'` with no publish target, `lxdb drop` / `truncate` / `release-locks` / `config delete` without `--confirm` (dry run) |
| 1 | Unexpected error |
| 2 | Command-line error (unknown option, invalid choice, missing required flag) |
| 10 | Configuration error: credentials file missing or invalid for `sync`, unknown id, unset `$env{}`, sync not found or already existing, invalid `sync_id`, invalid incremental spec, no output destination, invalid publish pattern, run already running, `use_adbc` not a boolean, `--unbounded_number_type` that does not fit the source or target, `status` without `--sync_id` / `--run_id`, `cleanup` duration invalid |
| 11 | Authentication or authorization failure |
| 12 | Connection failed: source (`sync`), or LakeXpress DB (`lxdb`, `config create`; with `sync` a LakeXpress DB connection failure exits 10) |
| 13 | License missing, invalid or expired; source extracted through ADBC without an ADBC license block; FastBCP rejected its license during the export |
| 21 | Credentials file not found (`lxdb`, `config`, `status`, `cleanup`; `sync` reports 10) |
| 30 | FastBCP not configured, or no executable in the configured folder (10 before 0.5.0) |
| 31 | `lxdb init` / `drop` / `truncate` failed |
| 34 | FastBCP older than 1.4.0 |
| 40 | Some tables failed to export, others succeeded: fix, then `sync --resume` |
| 41 | Every table failed to export; `sync` does not publish |
| 44 | No table matched the source schemas and patterns (the run is recorded as `failed`) |
| 45 | `--resume` found no run to resume (or the `--run_id` does not exist); `'sync[publish]'` found no completed export run |
| 46 | Export succeeded but publishing had failures (`sync`), or publishing failed (`'sync[publish]'`) |
| 47 | Some tables failed to export and publishing had failures |
| 50 | Interrupted with Ctrl+C |

Codes 20, 22, 32, 33, 42 and 43 are reserved and not returned yet
(https://lakexpress-docs.arpe.io/latest/reference/exit-codes). Schedulers: 10–39 need a fix of the
configuration or the environment; 40 and 47 can be retried with `--resume`; after 46, publish again
with `'sync[publish]'`.

## Scheduling

Run `sync` from cron, Task Scheduler, Airflow… with the same command every time; check the exit code.
Only one run of a configuration can run at a time. With SQLite / DuckDB LakeXpress DB, run all syncs on
the same machine, one at a time.
