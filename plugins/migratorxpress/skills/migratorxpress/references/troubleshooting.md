# MigratorXpress – Troubleshooting and known 0.7.x behaviours

## Known 0.7.x behaviours to plan around

| Behaviour | What to do |
|---|---|
| `--drop_tables_if_exists false` **enables** dropping (any value enables it) and forces `--without_xid` | Omit the flag to keep tables; pass `true` only on purpose |
| Exit code 0 even when some tables failed | Read `RUN SUMMARY: n/m tables succeeded` at the end of the log, or query `sourcetargetloads` / `eventlog` |
| Exit code 0 when the task list is invalid, missing, or combined with `--resume` | Check the log for CRITICAL/ERROR lines in scripts |
| `--source_db_name` is required but the source connection uses the auth entry's `database` | Always pass the same value as the auth entry |
| PostgreSQL as a target: `translate` produces SQL Server types | Use SQL Server as the target in 0.7.x |
| `copy_pk` / `copy_ak` / `copy_fk` only for Oracle → SQL Server | Create constraints manually for other paths |
| Only `auth_mode: "classic"` works | No ODBC, connection strings or trusted auth |

## Common errors

| Symptom | Cause / fix |
|---|---|
| `the following arguments are required: …` (exit 2) | Add the missing required flag (often `--source_db_name` / `--target_db_name`) |
| `incorrect task sequence` | Use an accepted combination ([tasks.md](tasks.md)); e.g. add `create` between `translate` and `transfer` |
| `--resume` complains or does nothing useful | Pass the run id; don't add `--task_list`; the tracking DB must still contain the run (`--migration_db_mode preserve`) |
| Connection error (exit 12) | Check host, `port` (JSON integer), `database` (Oracle: service name), firewall, credentials. `--log_level DEBUG` |
| Oracle `DPI-1047` (thick mode) | Install Oracle Instant Client and set `lib_dir` (and `LD_LIBRARY_PATH` on Linux, `libaio`); for Oracle 12.1+ remove `lib_dir` to use thin mode |
| Oracle `ORA-12154` / `ORA-28000` | Wrong service name / account locked |
| FastTransfer not found or too old | Put FastTransfer ≥ 0.15.0 in `engine/` next to the binary or pass `--fasttransfer_dir_path`; check `FastTransfer --version` |
| Large tables transfer as one stream | `--fasttransfer_p` must be > 1 and the estimated row count ≥ `--ft_large_table_th`; estimates may be stale (`--compute_nbrows true`). SQL Server sources never split a table |
| Oracle/Netezza tables skipped on a re-run | The transaction id / ROWID watermark skips unchanged tables; use `--without_xid` to force reloading |
| Existing target tables not recreated | `create` keeps existing tables; pass `--drop_tables_if_exists true` to drop and recreate them |
| Target tables emptied | `--load_mode truncate` is the default; use `--load_mode append` to keep rows |
| Tracking DB from 0.6.x | Upgraded automatically on the first 0.7 run, or with `--upgrade_migdb`; back it up first |
| License error (exit 13) | Check the lookup order ([environment.md](environment.md)); an invalid license does not fall back to trial |
| Tracking DB unreachable or not writable | The tracking DB is mandatory: check `--migration_db_auth_id` and CREATE TABLE rights there |

## Diagnostics

- `--log_level DEBUG --log_dir ./logs`, then read `mig_*_<run_id>.log`.
- `SELECT * FROM sourcetargetloads WHERE run_id = '<run_id>'` for per-table status.
- `SELECT * FROM eventlog WHERE eventrunid = '<run_id>' AND eventstatus = 'FAILED'` for error messages.
- After fixing the cause, `--resume <run_id>` processes only the jobs that are not `DONE`.
