# MigratorXpress – Install, license, FastTransfer, tracking DB, logging, exit codes

## Install

- Native binaries, no runtime needed: `MigratorXpress` (Linux x64) and `MigratorXpress.exe` (Windows x64).
  Download / trial from arpe.io (https://arpe.io/get-trial?product=MigratorXpress).
- Trial distributions ship FastTransfer in an `engine/` folder next to the binary.
- `./MigratorXpress --version`, `./MigratorXpress --help`.

## FastTransfer

All data movement is done by FastTransfer (≥ 0.15.0, version checked at startup).

- Found automatically in `engine/` next to the MigratorXpress binary; otherwise pass
  `--fasttransfer_dir_path <dir containing FastTransfer[.exe]>`.
- Parallelism: `--n_jobs` tables at once; inside a table ≥ `--ft_large_table_th` rows, `--fasttransfer_p`
  streams with Ctid (PostgreSQL source), Rowid (Oracle) or NZDataSlice (Netezza). SQL Server sources: one stream
  per table.
- `--quiet_ft` hides FastTransfer output from the console (still logged to the file).
- FastTransfer is started without a shell, so passwords with `"`, `$`, `` ` ``, `%` or spaces reach it unchanged;
  passwords and connection strings are masked in the logged command and output.
- PostgreSQL target: written with FastTransfer `pgcopy`. A PostgreSQL source is then read with `pgcopy` too, and
  SQL Server date/time and binary columns are read through a casting query, so values arrive exactly. Oracle →
  PostgreSQL exactness was not verified by the 0.7.3 release: run `diff` and spot-check.

Sizing profiles from the docs:

| Migration | Suggested flags |
|---|---|
| Small (< 10 tables, < 1M rows) | defaults |
| Medium (10–100 tables, up to 100M rows) | `--n_jobs 4 --fasttransfer_p 2` |
| Large (100+ tables, billions of rows) | `--n_jobs 8 --fasttransfer_p 4 --ft_large_table_th 500000` |

Oracle sources benefit from 4–8 streams on tables above 100M rows. Adjust to the source and target capacity.

## License

Lookup order (first found wins):

1. env `MIGRATORXPRESS_LICENSE` (license text)
2. `--license "<text>"`
3. `--license_file <path>`
4. `~/.migratorxpress/license.lic` (Linux/macOS) or `%APPDATA%\MigratorXpress\license.lic` (Windows)
5. `./license.lic` (current directory)
6. Trial: 30 days from the binary's build date

An invalid or expired license stops the run with exit code 13 (no fallback to trial).

## Tracking (migration) database

SQL Server (`ds_type: mssql`, tables in `dbo`) or PostgreSQL (`ds_type: postgres`, tables in `public`). Tables are
created on first use. Main tables:

| Table | Content |
|---|---|
| `sources`, `sourcecolumns`, `sourceconstraints` | Source catalog snapshot (per `run_id` since 0.7.0) |
| `targets`, `targetcolumns` | Target definitions (per `run_id`) |
| `sourcetargetloads` | One row per table job and task, with status (`STARTED`, `DONE`, `FAILED`), `run_id`, `project` |
| `eventlog` | Events: `eventtimestamp`, `eventrunid`, `eventstatus`, `eventmessage` (error text), elapsed time, `project` |
| `sourcetargetdiff` (+ view `v_sourcetargetdiff`) | `diff` results |

Upgrading from 0.6.x: the catalog tables gained `run_id` in their primary keys. The upgrade runs automatically on
the first 0.7 run, or explicitly:

```bash
./MigratorXpress --auth ./credentials.json --migration_db_auth_id ms_mig_log --upgrade_migdb
```

It is idempotent and resumable; existing rows get `run_id = 'legacy'`. Back up the tracking DB first; on large
SQL Server tracking DBs the primary-key rebuild is the slow part.

## Logging

- Console + file `mig_YYYYMMDD_HHMMSS_<run_id>.log` in `--log_dir` (default: current directory).
- Format: `[YYYY-MM-DD HH:MM:SS.fff+TZ :: LEVEL :: PID :: process] message`.
- `--log_level DEBUG` for troubleshooting.
- The logged `fasttransfer command` is shell-quoted and starts with the absolute FastTransfer path (since 0.7.2;
  update log parsers written for 0.7.1).
- At the end: `RUN SUMMARY: n/m tables succeeded` with the failed tables and tasks, and, when tables failed,
  the `--resume <run_id>` command to retry them.

## Exit codes (0.7.3)

| Code | Meaning |
|---|---|
| 0 | Every table succeeded every task (differences reported by `diff` don't fail a table) |
| 1 | Other error (migration error, invalid `--migration_db_mode`) |
| 2 | Command-line error (missing required flag, invalid choice or boolean, bad `--project`) |
| 10 | Configuration error, before any connection: invalid auth entry or mode, auth id not in the file, unsupported target / tracking DB type or source → target pair (Netezza → PostgreSQL), `copy_*` with a Netezza source, `copy_fk` with a PostgreSQL target, ODBC prerequisites missing |
| 12 | Connection to source, target or tracking DB failed |
| 13 | License invalid or expired |
| 20 | Validation error: task list missing / invalid / combined with `--resume`, invalid `--fk_mode`, `--target_db_name` not matching the target entry |
| 40 | Some tables failed a task (`PARTIAL_FAILURE`): fix and `--resume <run_id>` |
| 41 | Every table failed (`ALL_JOBS_FAILED`) |

Schedulers: 10–29 need a fix before retrying; 40–41 can be retried with `--resume`. Invalid task lists exited 0
before 0.7.2, and failed tables before 0.7.3; scripts relying on that now see non-zero codes.
