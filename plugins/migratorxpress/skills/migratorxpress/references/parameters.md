# MigratorXpress – CLI parameter reference (0.8.0)

Generate long flags. Short forms exist only for a few flags (`-a`, `-r`, `-p`, `-i`, `-e`, `-min`, `-max`, `-f`).

## Required (every migration command)

| Flag | Notes |
|---|---|
| `--auth PATH` | Credentials JSON ([auth-file.md](auth-file.md)) |
| `--source_db_auth_id ID` | Source entry |
| `--target_db_auth_id ID` | Target entry |
| `--migration_db_auth_id ID` | Tracking DB entry |
| `--task_list TASKS…` **or** `--resume RUN_ID` | Exactly one ([tasks.md](tasks.md)) |

Exception: `--upgrade_migdb` needs only `--auth` and `--migration_db_auth_id`.

## Database names (optional)

| Flag | Notes |
|---|---|
| `--source_db_name NAME` | **Deprecated, ignored**: the source database is the source entry's. A different value is logged as a warning |
| `--target_db_name NAME` | Defaults to the target entry's database. When given, must match it (case-insensitive for SQL Server, exact for PostgreSQL), otherwise exit 20 |

## Scope: schemas and tables

| Flag | Default | Notes |
|---|---|---|
| `--source_schema_name PATTERN` | all user schemas | One schema (`SALES`), SQL `LIKE` patterns (`"app_%"`, `"APP_%, TENANT_%"`), or an exact comma list (`"tpch_1, tpch_2"`). Schema patterns are matched **case-insensitively**; `_` matches any single character |
| `--target_schema_name NAME` | source schema name(s) | Plain name, never a pattern. With several source schemas, **all** of them are loaded into this one schema; omit it to keep one target schema per source schema. Created if missing |
| `--include_tables PATTERN` | `%` | Comma-separated `LIKE` patterns, quoted as one argument: `"fact_%,dim_%"` |
| `--exclude_tables PATTERN` | none | Applied after include |
| `--min_rows N` | `-1` (off) | Exclude tables with strictly fewer rows |
| `--max_rows N` | `-1` (off) | Exclude tables with at least N rows |

## Tracking DB

| Flag | Default | Notes |
|---|---|---|
| `--migration_db_mode` | `preserve` | `preserve` keeps all tracking data (needed for `--resume`); `truncate` empties all tracking tables; `drop` drops and recreates them |
| `--upgrade_migdb` | – | Upgrade a 0.6.x tracking DB to the 0.7 schema, then exit |
| `--force` | off | Delete existing tracking rows for the source database/schema before the run |
| `--project NAME` | none | Tag stored with the run (`[A-Za-z0-9_-]`, 1–64 chars). Ignored on resume |

## Load behaviour

| Flag | Default | Notes |
|---|---|---|
| `--load_mode` | `truncate` | `truncate` empties each target table before loading; `append` adds rows |
| `--drop_tables_if_exists [BOOL]` | `false` | Drops and recreates existing target tables in `create`. `true` / `yes` / `1` or the bare flag enable it, `false` / `no` / `0` disable it, other values are rejected (exit 2). When enabled, `--with_xid` is ignored (warning). A table whose `translate` failed is neither created nor dropped |
| `--basic_diff` | off | `diff` compares only row and column counts (skips per-column statistics) |
| `--compute_nbrows true\|false` | `false` | Exact `COUNT(*)` instead of catalog estimates |
| `--with_xid` | off | Oracle / Netezza sources: skip a table whose `MAX(ROWID)` / `MAX(CREATEXID)` is unchanged since the last `--with_xid` run and whose target holds rows (log: `skipped (--with_xid): …`). No effect (warning) for PostgreSQL / SQL Server sources. Before 0.8.0 this skip was on by default |
| `--without_xid` | – | Deprecated, ignored with a warning (the skip is off by default since 0.8.0) |

## Performance and FastTransfer

| Flag | Default | Notes |
|---|---|---|
| `--n_jobs N` | `1` | Tables processed in parallel |
| `--fasttransfer_p N` | `1` | Parallel streams per **large** table |
| `--ft_large_table_th N` | `250000000` | Tables with at least this many rows (estimated) use the source's parallel method when `--fasttransfer_p` > 1: Ctid (PostgreSQL), Rowid (Oracle), NZDataSlice (Netezza). SQL Server sources: no intra-table parallelism |
| `--fasttransfer_dir_path PATH` | `engine/` next to the binary | Directory containing the FastTransfer binary (≥ 0.15.0) |

## SQL Server target options

Ignored with a PostgreSQL target (`--cci_threshold` / `--aci_threshold` are logged as ignored; `copy_fk` is
rejected there).

| Flag | Default | Notes |
|---|---|---|
| `--cci_threshold N` | `1000000` | Tables ≥ N rows get a clustered columnstore index |
| `--aci_threshold N` | `100000000` | Tables ≥ N rows get `COLUMNSTORE_ARCHIVE` compression |
| `--fk_mode` | `disabled` | For `copy_fk`: `disabled` (WITH NOCHECK then disabled), `untrusted` (WITH NOCHECK, enabled), `trusted` (fully validated). Any other value exits with code 20 |

## Oracle source options

| Flag | Default | Notes |
|---|---|---|
| `--forced_int_id_prefixes P…` | none | Column name prefixes forced to integer types (skip profiling), case-insensitive, space-separated |
| `--forced_int_id_suffixes S…` | none | Same for suffixes, e.g. `_ID _KEY` |
| `--profiling_sample_pc N` | `10` | Sample percentage for DATE/NUMBER profiling (`100` = no sampling) |
| `--min_sample_pc_profile N` | `100000` | Tables below this row count are profiled in full |
| `--p_query N` | `6` | `PARALLEL` hint degree for profiling queries |

## Logging and licence

| Flag | Default | Notes |
|---|---|---|
| `--log_level` | `INFO` | `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` |
| `--log_dir PATH` | current directory | Created if missing |
| `--no_banner`, `--no_progress`, `--quiet_ft` | off | Hide banner / progress display / FastTransfer console output (still in the log file) |
| `--license TEXT` | – | License content |
| `--license_file PATH` | – | License file (default places: environment.md) |
| `--version`, `--help` | – | |
