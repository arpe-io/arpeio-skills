# MigratorXpress – Tasks and resume

## Tasks

| Task | Does |
|---|---|
| `translate` | Reads the source catalog and translates column data types for the target (Oracle DATE/NUMBER columns are profiled) |
| `create` | Creates target schemas and tables (keeps existing tables unless `--drop_tables_if_exists`) |
| `transfer` | Copies the rows with FastTransfer (`--load_mode` truncate/append) |
| `diff` | Compares source and target: row and column counts, then per-column statistics (min, max, distinct, nulls, mean) unless `--basic_diff` |
| `copy_pk` | Creates primary keys (Oracle → SQL Server only) |
| `copy_ak` | Creates unique constraints (Oracle → SQL Server only) |
| `copy_fk` | Creates foreign keys according to `--fk_mode` (Oracle → SQL Server only) |
| `all` | All seven |

Table discovery always runs first (internal step).

## Order and accepted combinations

Tasks always run in this order, whatever order is typed:
`translate → create → transfer → diff → copy_pk → copy_ak → copy_fk`. Duplicates are rejected.

Only these combinations are accepted (anything else stops with `incorrect task sequence`):

| Count | Accepted |
|---|---|
| 1 | any single task |
| 2 | `translate create`, `create transfer`, `transfer diff`, `transfer copy_pk`, `diff copy_pk`, `copy_pk copy_ak`, `copy_pk copy_fk`, `copy_ak copy_fk` |
| 3 | `translate create transfer`, `translate create copy_pk`, `create transfer diff`, `transfer diff copy_pk`, `transfer copy_pk copy_ak`, `diff copy_pk copy_ak`, `transfer copy_pk copy_fk`, `diff copy_pk copy_fk`, `copy_pk copy_ak copy_fk` |
| 4 | `translate create transfer diff`, `translate create transfer copy_pk`, `translate create transfer copy_ak`, `create transfer diff copy_pk`, `create transfer copy_pk copy_ak`, `transfer diff copy_pk copy_ak`, `create transfer copy_pk copy_fk`, `transfer diff copy_pk copy_fk`, `diff copy_pk copy_ak copy_fk` |
| 5 | `translate create transfer diff copy_pk`, `translate create transfer copy_pk copy_ak`, `translate create transfer copy_pk copy_fk`, `translate create transfer copy_ak copy_fk`, `create transfer diff copy_pk copy_ak`, `create transfer diff copy_pk copy_fk`, `create transfer diff copy_ak copy_fk` |
| 6 | `translate create transfer diff copy_pk copy_ak`, `translate create transfer diff copy_pk copy_fk`, `translate create transfer copy_pk copy_ak copy_fk`, `create transfer diff copy_pk copy_ak copy_fk` |
| 7 | `all` |

Examples: `translate transfer` is rejected (missing `create`); `transfer copy_ak` is rejected.

## Typical sequences

| Goal | `--task_list` |
|---|---|
| Full schema + data + validation | `translate create transfer diff` |
| Then constraints (Oracle → SQL Server) | `copy_pk copy_ak copy_fk` |
| Everything in one run | `all` (Oracle → SQL Server; with other sources `copy_*` fails) |
| Schema only | `translate create` |
| Reload data into existing tables | `transfer diff` |
| Validate only | `diff` |

The docs run constraints as a separate step after the data is loaded and validated.

## Resume

- `--resume RUN_ID` (`YYYYMMDD-<uuid4>`, printed at start, in the log file name and in the tracking DB) finishes a
  run: all steps are replayed and only table jobs not yet `DONE` are processed.
- Do not pass `--task_list` with `--resume`. Schema flags are not needed.
- All other required flags are still needed (`--auth`, both auth ids, both db names, `--migration_db_auth_id`).
- Performance and logging flags are taken from the resume command line.
- The tracking DB must still hold the run: keep `--migration_db_mode preserve` (the default).
- `--project` is ignored on resume (the original tag is kept).

## Validation exit code (0.7.x bug)

A missing, duplicated or invalid task list (or both `--task_list` and `--resume`) is logged as CRITICAL/ERROR but
the process exits **0**. In scripts, check the log as well as the exit code.
