---
name: migratorxpress
description: Build, fix, explain or run MigratorXpress command lines and credentials files. MigratorXpress (by arpe.io) migrates whole database schemas between platforms (Oracle, PostgreSQL, SQL Server, Netezza sources; SQL Server target, PostgreSQL target experimental in 0.7) - it translates data types, creates the target tables, transfers the data in parallel with FastTransfer, compares source and target, and copies constraints, with a tracking database that allows resuming. Use when the user mentions MigratorXpress, or wants to migrate a database or schemas from Oracle / PostgreSQL / SQL Server / Netezza to SQL Server, resume a failed migration, choose a task list, or troubleshoot a migration run.
---

# MigratorXpress

MigratorXpress migrates **whole schemas**: it discovers the source tables, translates DDL and data types,
creates target tables, moves the rows with FastTransfer, verifies the result and copies constraints. Every
step and table is tracked in a **migration tracking database**, which makes runs resumable.

Goal: produce a **credentials file** (if needed) and **one correct, copy-pasteable command** for the user's
OS and shell. Never invent hosts, credentials, schema or database names: ask.

Facts below reflect MigratorXpress **0.7.x** (v0.7.1 code + 0.7 docs). References (load only what you need):

| Need | File |
|---|---|
| Credentials file format (`--auth`) | [references/auth-file.md](references/auth-file.md) |
| Every CLI flag, defaults, schema/table selection | [references/parameters.md](references/parameters.md) |
| Tasks, accepted task combinations, resume | [references/tasks.md](references/tasks.md) |
| Install, license, FastTransfer, tracking DB, logging, exit codes | [references/environment.md](references/environment.md) |
| Complete commands | [references/examples.md](references/examples.md) |
| Known 0.7.x bugs and pitfalls | [references/troubleshooting.md](references/troubleshooting.md) |

Official docs: https://migratorxpress.arpe.io/latest/

## Three databases, one credentials file

| Role | Flags | Supported (`ds_type`) |
|---|---|---|
| Source | `--source_db_auth_id`, `--source_db_name`, `--source_schema_name` | `oracle`, `postgres`, `mssql`, `netezza` |
| Target | `--target_db_auth_id`, `--target_db_name`, `--target_schema_name` | `mssql` (PostgreSQL: experimental in 0.7.x, see below) |
| Tracking DB | `--migration_db_auth_id` | `mssql` or `postgres` |

Each `*_auth_id` is a key in the JSON credentials file passed with `--auth`. In 0.7.x every entry uses
`"auth_mode": "classic"` with `username`, `password`, `server`, `port`, `database` (and `lib_dir` for Oracle
thick mode). No ODBC, connection-string, trusted or env-var substitution.

## Workflow

1. **Credentials file** – one entry per database (source, target, tracking). See auth-file.md.
2. **Required flags** – `--auth`, `--source_db_auth_id`, `--source_db_name`, `--target_db_auth_id`,
   `--target_db_name`, `--migration_db_auth_id`. Database names = the `database` of the matching auth entry.
3. **Scope** – `--source_schema_name` (one schema, `LIKE` patterns, a comma list, or omitted = all user schemas);
   `--target_schema_name` (plain name; omit it for multi-schema runs); table filters `--include_tables`,
   `--exclude_tables`, `--min_rows`, `--max_rows`.
4. **Tasks** – exactly one of `--task_list …` (new run) or `--resume RUN_ID`. Typical full run:
   `--task_list translate create transfer diff`, then constraints `copy_pk copy_ak copy_fk` (Oracle → SQL Server
   only). Only some task combinations are accepted (tasks.md).
5. **Performance** – `--n_jobs` (tables in parallel) × `--fasttransfer_p` (streams per large table, for tables
   ≥ `--ft_large_table_th` rows).
6. **Ops** – `--log_dir`, `--log_level`, `--project` tag, `--no_banner`, `--no_progress`, `--quiet_ft`.
7. **Output** – the credentials JSON (placeholders for secrets) and one fenced command, then a short summary
   with assumptions and what to check afterwards (RUN SUMMARY, tracking tables).

## Hard rules (0.7.x)

- **Always pass `--source_db_name` and `--target_db_name`** (argparse-required). Use the `database` value of
  the corresponding auth entries.
- **Exactly one of `--task_list` / `--resume RUN_ID`.** `--resume` needs the run id (`YYYYMMDD-<uuid4>`) and
  still needs all the required flags above; schema flags are not needed on resume. Keep
  `--migration_db_mode preserve` (default) when you may resume.
- **Tasks always run in the order** translate → create → transfer → diff → copy_pk → copy_ak → copy_fk, and
  only listed combinations are accepted (e.g. `translate transfer` is rejected: include `create`). Check
  tasks.md before proposing a non-standard list.
- **`--drop_tables_if_exists`: any value enables it in 0.7.x, including `false`.** To keep existing tables,
  **omit the flag**. Use `--drop_tables_if_exists true` only when the user wants target tables dropped and
  recreated, and confirm with them first.
- `--load_mode truncate` (default) empties each target table before loading; `append` adds rows.
- **Target = SQL Server** for reliable migrations. A PostgreSQL target is experimental in 0.7.x (`translate`
  produces SQL Server types). If the user asks for PostgreSQL as a target, say so and suggest waiting for the
  release that completes it, or migrating to SQL Server.
- **Constraint copy** (`copy_pk`, `copy_ak`, `copy_fk`) works only for Oracle → SQL Server. `--fk_mode`
  (`disabled` default, `untrusted`, `trusted`) and `--cci_threshold` / `--aci_threshold` are SQL Server target
  options.
- `--target_schema_name` is never a pattern. Given with several source schemas, it merges all of them into
  that one target schema.
- Use **long flags** (`--auth`, `--resume`, `--include_tables`, …), not `-a`, `-r`, `-i`.
- Quote patterns as one argument: `--include_tables "fact_%,dim_%"`.
- Never put real passwords in what you show; use placeholders and tell the user to restrict the file's
  permissions.

## Shell conventions

| OS / shell | Executable | Line continuation |
|---|---|---|
| Linux (bash, zsh) | `./MigratorXpress` | `\` |
| Windows PowerShell | `.\MigratorXpress.exe` | `` ` `` |

## Example

Oracle schema SALES to SQL Server, tables, data and validation, 4 tables in parallel:

```bash
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id oracle_prod \
  --source_db_name ORCLPDB1 \
  --source_schema_name SALES \
  --target_db_auth_id mssql_target \
  --target_db_name target_db \
  --target_schema_name sales \
  --migration_db_auth_id ms_mig_log \
  --n_jobs 4 \
  --fasttransfer_p 2 \
  --task_list translate create transfer diff \
  --log_dir ./logs
```

Then constraints: same command with `--task_list copy_pk copy_ak copy_fk` (and without `--n_jobs`/FastTransfer
options if preferred).

## After a run

- **The exit code is 0 even if some tables failed.** Read the `RUN SUMMARY: n/m tables succeeded` block at the
  end of the log (`mig_YYYYMMDD_HHMMSS_<run_id>.log`) or query `sourcetargetloads` / `eventlog`.
- To finish a run after failures: same required flags + `--resume <run_id>`.
- Upgrading from 0.6.x: the tracking DB is upgraded automatically on the first 0.7 run, or explicitly with
  `--auth … --migration_db_auth_id … --upgrade_migdb`. Back it up first.
