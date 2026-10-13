# LakeXpress – Install, license, FastBCP, LakeXpress DB, logging, exit codes

## Install

- Native binaries: `LakeXpress` (Linux x64, `chmod +x LakeXpress`) and `LakeXpress.exe` (Windows x64).
  Trial: https://arpe.io/get-trial?product=LakeXpress. Unzip anywhere.
- `./LakeXpress --version` → `LakeXpress 0.4.10 (build 2026-09-24T12:04:58Z)` (build date shown since
  0.4.9). `./LakeXpress --help`, `./LakeXpress config create --help`.
- Archive: the binary, `README.txt`, SBOM files (CycloneDX JSON/XML + SHA-256), and in the **trial**
  edition `engine/FastBCP[.exe]`.

## FastBCP

- Trial edition: FastBCP bundled in `engine/`, found automatically.
- Full edition: FastBCP is not bundled. Install it, pass its folder with `config create
  --fastbcp_dir_path <folder>` (saved in the configuration; `sync --fastbcp_dir_path` overrides it). Its
  license must be named exactly `FastBCP.lic` next to the FastBCP binary (rename the downloaded
  `SUBSCRIPTION-FastBCP-….lic`).
- No FastBCP found: `FastBCP directory path is required for export operations` (exit 10).
- `--quiet_fbcp` keeps FastBCP output out of the LakeXpress log.

## License

Lookup order:

1. env `LAKEXPRESS_LICENSE` (license text; for CI/containers)
2. `--license "<text>"`
3. `--license_file <path>`
4. `~/.lakexpress/license.lic` (Linux/macOS), `%APPDATA%/LakeXpress/license.lic` (Windows), `./license.lic`
5. Trial: 30 days from the **build date** of the binary. Trial downloads are rebuilt weekly (≥ 23 days left).

Invalid or expired license: exit 13.

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
  `lx_YYYYMMDD_HHMMSS_<run_id>.yml` (copy of the configuration, usable with `run`).
- Format: `[YYYY-MM-DD HH:MM:SS.fff+TZ :: LEVEL :: PID :: process] message`.
- `--log_level DEBUG` adds SQL queries, file operations, watermark bounds.
- The run id is printed at start (`Created export run: <run_id>`); the end shows `RUN SUMMARY` (or
  `RESUME SUMMARY`) `Export: n/m tables succeeded`, the failed tables and their errors.

## Exit codes (0.4.10)

| Code | Meaning |
|---|---|
| 0 | Success. Also: no table matched the filters (warning), `'sync[publish]'` with no publish target, `lxdb drop` / `truncate` / `release-locks` / `config delete` without `--confirm` (dry run) |
| 1 | Unexpected error, or `'sync[publish]'` failed |
| 2 | Command-line error (unknown option, invalid choice, missing required flag) |
| 10 | Configuration error: credentials file missing or invalid for `sync`, unknown id, unset `$env{}`, sync not found or already existing, invalid `sync_id`, invalid incremental spec, no output destination, FastBCP not found, invalid publish pattern, resume problems, run already running, `status` without `--sync_id` / `--run_id`, `cleanup` duration invalid |
| 12 | Connection failed: source (`sync`), or LakeXpress DB (`lxdb`, `config create`; with `sync` a LakeXpress DB connection failure exits 10) |
| 13 | License invalid or expired |
| 21 | Credentials file not found (`lxdb`, `config`, `status`, `cleanup`; `sync` reports 10) |
| 31 | `lxdb init` / `drop` / `truncate` failed |
| 40 | Some tables failed (export or publish): fix, then `sync --resume` |

Codes 11, 20, 22, 30, 32–33, 41–47 and 50 are defined in the product but not returned by 0.4.10
commands (a run where every table fails also exits 40). Schedulers: 10–29 need a fix; 40 can be retried
with `--resume`.

## Scheduling

Run `sync` from cron, Task Scheduler, Airflow… with the same command every time; check the exit code.
Only one run of a configuration can run at a time. With SQLite / DuckDB LakeXpress DB, run all syncs on
the same machine, one at a time.
