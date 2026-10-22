# MigratorXpress – Troubleshooting and known 0.7.3 behaviours

## Known behaviours to plan around

| Behaviour | What to do |
|---|---|
| Failed tables make the run exit 40 (some) or 41 (all) since 0.7.3; before, it exited 0 | Schedulers: treat 40/41 as retryable with `--resume <run_id>` (the log prints the command) |
| Differences found by `diff` don't fail a table and don't change the exit code | Read the diff results (`v_sourcetargetdiff`) or the log |
| PostgreSQL target: schema, table and column names are lower-cased | Expect `sales.orders` for `Sales.Orders`; don't `--resume` a PostgreSQL-target run started before 0.7.2 |
| PostgreSQL target: `copy_fk`, `--fk_mode`, `--task_list all` not supported; `--cci_threshold` / `--aci_threshold` ignored | Use `translate create transfer diff copy_pk copy_ak`; create foreign keys manually |
| Netezza source: no `copy_pk` / `copy_ak` / `copy_fk`; no PostgreSQL target | Create constraints manually; target SQL Server |
| Foreign keys to a table in another schema (PostgreSQL / SQL Server sources) are skipped with a warning | Create them manually |
| SQL Server `GEOGRAPHY` / `GEOMETRY` (to PostgreSQL) and PostgreSQL `ARRAY` / user-defined columns fail `translate` | Exclude those tables (`--exclude_tables`) and migrate them separately; the failed table is not created |
| Oracle → PostgreSQL data exactness not verified by the 0.7.3 release | Run `diff`, spot-check dates, timestamps and binary columns |
| `--source_db_name` is ignored; `--target_db_name` must match the target entry | Omit both |
| PostgreSQL source columns keep `NOT NULL` on the target since 0.7.3 (they were all nullable before) | Tables created by an older version differ; recreate them (`--drop_tables_if_exists true`) to get the source nullability |

## Common errors

| Symptom | Cause / fix |
|---|---|
| `the following arguments are required: …` (exit 2) | Add the missing required flag (`--auth`, the three auth ids) |
| `argument --drop_tables_if_exists: … Boolean value expected` (exit 2) | Use `true` / `false` (or the bare flag) |
| `incorrect task sequence` (exit 20) | Use an accepted combination ([tasks.md](tasks.md)); e.g. add `create` between `translate` and `transfer` |
| `Either 'task_list' or 'resume' should be given …` (exit 20) | Pass exactly one of `--task_list` / `--resume` |
| Exit 20, target database name mismatch | `--target_db_name` differs from the target entry's database: remove it or fix the entry |
| Exit 10, auth entry error (`INVALID_AUTH_ENTRY`, `AUTH_MODE_NOT_SUPPORTED`, `INVALID_CONNECTION_STRING`) | Fix the named entry/key in the credentials file ([auth-file.md](auth-file.md)); `classic` accepts only `username`, `password`, `server`, `port`, `database`, `lib_dir` |
| Exit 10, auth id not found | The `*_auth_id` is not a key of the file; the log lists the available names |
| Exit 10, `UNSUPPORTED_SOURCE_TARGET_PAIR` / `COPY_FK_UNSUPPORTED_TARGET` / `CONSTRAINT_COPY_UNSUPPORTED_SOURCE` | See the known behaviours above |
| Exit 10 with `odbc` | Install unixODBC (Linux) and Microsoft ODBC Driver 18, declare the DSN; never `odbc` for the target |
| `trusted` fails on Linux | `klist` shows a valid ticket? `server` is the FQDN? SPN `MSSQLSvc/<fqdn>:<port>` registered? `/etc/krb5.conf` realm/KDC? `libgssapi_krb5` installed for FastTransfer? |
| `--resume` complains or does nothing useful | Pass the run id; don't add `--task_list`; the tracking DB must still contain the run (`--migration_db_mode preserve`) |
| SQL Server `Cannot define PRIMARY KEY constraint on nullable column` | The target column is nullable: tables created from a PostgreSQL source before 0.7.3 were all nullable; recreate them (`--drop_tables_if_exists true`) |
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
