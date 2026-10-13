# MigratorXpress – Tasks and resume

## Tasks

| Task | Does |
|---|---|
| `translate` | Reads the source catalog and translates column data types for the target (Oracle DATE/NUMBER columns are profiled) |
| `create` | Creates target schemas and tables (keeps existing tables unless `--drop_tables_if_exists`) |
| `transfer` | Copies the rows with FastTransfer (`--load_mode` truncate/append) |
| `diff` | Compares source and target: row and column counts, then per-column statistics (min, max, distinct, nulls, mean) unless `--basic_diff` |
| `copy_pk` | Creates primary keys |
| `copy_ak` | Creates unique constraints |
| `copy_fk` | Creates foreign keys according to `--fk_mode` (SQL Server target only) |
| `all` | All seven (so SQL Server target only, because of `copy_fk`) |

Table discovery always runs first (internal step).

## Constraint copy support (0.7.3)

| Source → target | `copy_pk`, `copy_ak` | `copy_fk` |
|---|---|---|
| Oracle, PostgreSQL, SQL Server → SQL Server | yes | yes |
| Oracle, PostgreSQL, SQL Server → PostgreSQL | yes | **no** – rejected at startup (exit 10) |
| Netezza → SQL Server | **no** – rejected at startup (exit 10) | **no** |

- Multi-column keys keep their key order. PostgreSQL and SQL Server sources: foreign keys that reference a
  table in **another schema** are logged as a warning and not copied (target foreign keys are created within the target schema).
- SQL Server refuses a primary key on a nullable column: the target columns keep the source `NOT NULL`
  (PostgreSQL sources included since 0.7.3).

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
| Then constraints, SQL Server target | `copy_pk copy_ak copy_fk` |
| Then constraints, PostgreSQL target | `copy_pk copy_ak` |
| Everything in one run, SQL Server target | `all` |
| Everything in one run, PostgreSQL target | `translate create transfer diff copy_pk copy_ak` |
| Netezza source | no `copy_*` task |
| Schema only | `translate create` |
| Reload data into existing tables | `transfer diff` |
| Validate only | `diff` |

The docs run constraints as a separate step after the data is loaded and validated.

## Resume

- `--resume RUN_ID` (`YYYYMMDD-<uuid4>`, printed at start, in the log file name and in the tracking DB) finishes a
  run: all steps are replayed and only table jobs not yet `DONE` are processed.
- Do not pass `--task_list` with `--resume`. Schema flags are not needed.
- `--auth` and the three auth ids (`--source_db_auth_id`, `--target_db_auth_id`, `--migration_db_auth_id`) are
  still needed.
- Performance and logging flags are taken from the resume command line.
- The tracking DB must still hold the run: keep `--migration_db_mode preserve` (the default).
- `--project` is ignored on resume (the original tag is kept).

- A run that ends with failed tables logs the exact `--resume <run_id>` command to use (exit 40 or 41).
- Don't resume a PostgreSQL-target run started before 0.7.2: start a new run.

## Validation errors (exit 20)

A missing task list (and no `--resume`), both `--task_list` and `--resume`, a duplicated or unknown task, or an
unaccepted combination stops the run with exit code 20 (`TASK_LIST_MISSING`, `TASK_LIST_AND_RESUME`,
`TASK_DUPLICATED`, `TASK_UNKNOWN`, `TASK_BAD_SEQUENCE`).
