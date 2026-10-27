---
name: migratorxpress
description: Build, fix, explain or run MigratorXpress command lines and credentials files. MigratorXpress (by arpe.io) migrates whole database schemas between platforms (Oracle, PostgreSQL, SQL Server, Netezza sources; SQL Server or PostgreSQL target) - it translates data types, creates the target tables, transfers the data in parallel with FastTransfer, compares source and target, and copies constraints, with a tracking database that allows resuming. Use when the user mentions MigratorXpress, or wants to migrate a database or schemas from Oracle / PostgreSQL / SQL Server / Netezza to SQL Server or PostgreSQL, resume a failed migration, choose a task list, or troubleshoot a migration run.
---

# MigratorXpress

MigratorXpress migrates **whole schemas**: it discovers the source tables, translates DDL and data types,
creates target tables, moves the rows with FastTransfer, verifies the result and copies constraints. Every
step and table is tracked in a **migration tracking database**, which makes runs resumable.

Goal: produce a **credentials file** (if needed) and **one correct, copy-pasteable command** for the user's
OS and shell. Never invent hosts, credentials, schema or database names: ask.

Facts below reflect MigratorXpress **0.8.0** (checked against the v0.8.0 code).
References (load only what you need):

| Need | File |
|---|---|
| Credentials file format (`--auth`), auth modes | [references/auth-file.md](references/auth-file.md) |
| Every CLI flag, defaults, schema/table selection | [references/parameters.md](references/parameters.md) |
| Tasks, accepted task combinations, constraint support, resume | [references/tasks.md](references/tasks.md) |
| Install, license, FastTransfer, tracking DB, logging, exit codes | [references/environment.md](references/environment.md) |
| Complete commands | [references/examples.md](references/examples.md) |
| Known behaviours, pitfalls, common errors | [references/troubleshooting.md](references/troubleshooting.md) |

Official docs: https://migratorxpress-docs.arpe.io/latest/ (full text, for agents:
https://migratorxpress-docs.arpe.io/latest/llms-full.txt)

## Three databases, one credentials file

| Role | Flags | Supported (`ds_type`) |
|---|---|---|
| Source | `--source_db_auth_id`, `--source_schema_name` | `oracle`, `postgres`, `mssql`, `netezza` |
| Target | `--target_db_auth_id`, `--target_schema_name` (`--target_db_name` optional) | `mssql`, `postgres` |
| Tracking DB | `--migration_db_auth_id` | `mssql`, `postgres` |

Supported source → target pairs: Oracle, PostgreSQL, SQL Server, Netezza → SQL Server; Oracle, PostgreSQL,
SQL Server → PostgreSQL. **Netezza → PostgreSQL is rejected** at startup (exit 10).

Each `*_auth_id` is a key in the JSON credentials file passed with `--auth`. Auth modes (auth-file.md):

| `auth_mode` | For |
|---|---|
| `classic` (server, port, database, username, password) | all database types |
| `connection_string` (ADO.NET string, passed to FastTransfer as is) | PostgreSQL, SQL Server |
| `trusted` (Kerberos on Linux / SSPI on Windows) | SQL Server |
| `odbc` (DSN) | SQL Server **source or tracking DB only**, not target |

No environment-variable substitution in the file.

## Check the installed version

When you can run commands, run `./MigratorXpress --version` (Windows: `.\MigratorXpress.exe --version`)
before building a command; otherwise ask for the version when it matters (a rejected flag, an unexpected
exit code).

- **0.8.0** → use this skill as is.
- **Older** → several hard rules below changed: before 0.8.0 the binary ran 30 days from its build date
  without a license, came as a Trial (FastTransfer bundled) and a Full package, and skipped unchanged
  Oracle / Netezza tables by default (`--without_xid` to reload); 0.7.2 / 0.7.3 changed exit codes 40/41,
  the PostgreSQL target, `--drop_tables_if_exists false` and startup validation. Check
  `./MigratorXpress --help` and the version notes in environment.md and troubleshooting.md, or suggest
  upgrading.
- **Newer** → this skill may miss new options or changed behaviour: confirm anything you rely on in
  `./MigratorXpress --help` and the official docs above, and say that you did.

## Workflow

1. **Credentials file** – one entry per database (source, target, tracking). See auth-file.md.
2. **Required flags** – `--auth`, `--source_db_auth_id`, `--target_db_auth_id`, `--migration_db_auth_id`.
   Database names come from the auth entries: don't pass `--source_db_name` (deprecated, ignored) and omit
   `--target_db_name` (if given, it must match the target entry's database, otherwise exit 20).
3. **Scope** – `--source_schema_name` (one schema, `LIKE` patterns, a comma list, or omitted = all user schemas);
   `--target_schema_name` (plain name; omit it for multi-schema runs); table filters `--include_tables`,
   `--exclude_tables`, `--min_rows`, `--max_rows`.
4. **Tasks** – exactly one of `--task_list …` (new run) or `--resume RUN_ID`. Typical full run:
   `--task_list translate create transfer diff`, then constraints `copy_pk copy_ak copy_fk` (SQL Server target)
   or `copy_pk copy_ak` (PostgreSQL target). Only some task combinations are accepted (tasks.md).
5. **Performance** – `--n_jobs` (tables in parallel) × `--fasttransfer_p` (streams per large table, for tables
   ≥ `--ft_large_table_th` rows).
6. **Ops** – `--log_dir`, `--log_level`, `--project` tag, `--no_banner`, `--no_progress`, `--quiet_ft`.
7. **Output** – the credentials JSON (placeholders for secrets) and one fenced command, then a short summary
   with assumptions and what to check afterwards (exit code, RUN SUMMARY, tracking tables).

## Hard rules (0.8.0)

- **A license file is required**, also for the free trial (it is a 30-day license file emailed by
  https://arpe.io/get-trial?product=MigratorXpress). Without one every run exits 13. Default place:
  `~/.migratorxpress/license.lic` (Linux), `%APPDATA%\MigratorXpress\license.lic` (Windows); other ways
  in environment.md.

- **Exactly one of `--task_list` / `--resume RUN_ID`.** `--resume` needs the run id (`YYYYMMDD-<uuid4>`) and
  still needs `--auth` and the three auth ids; schema flags are not needed on resume. Keep
  `--migration_db_mode preserve` (default) when you may resume.
- **Tasks always run in the order** translate → create → transfer → diff → copy_pk → copy_ak → copy_fk, and
  only listed combinations are accepted (e.g. `translate transfer` is rejected: include `create`). Check
  tasks.md before proposing a non-standard list. An invalid list exits with code 20.
- **Constraint copy**: `copy_pk` / `copy_ak` from Oracle, PostgreSQL and SQL Server sources; not from Netezza
  (exit 10). `copy_fk` and `--fk_mode` only with a **SQL Server target** – with a PostgreSQL target, `copy_fk`
  and therefore `--task_list all` are rejected (exit 10): use an explicit list such as
  `translate create transfer diff copy_pk copy_ak`.
- **`--drop_tables_if_exists`** drops and recreates existing target tables in `create` (and ignores
  `--with_xid`). Default off; `true` / bare flag enables, `false` disables. Use it only when the user wants
  tables recreated, and confirm with them first.
- `--load_mode truncate` (default) empties each target table before loading; `append` adds rows.
- **Every table is transferred on every run.** `--with_xid` (Oracle, Netezza sources only) skips a table
  whose source `MAX(ROWID)` / `MAX(CREATEXID)` is unchanged since the last `--with_xid` run and whose
  target holds rows. `--without_xid` is deprecated and ignored (warning).
- **PostgreSQL target**: schema, table and column names are created **lower case** (`Sales.Orders` →
  `sales.orders`); `--cci_threshold` / `--aci_threshold` are ignored. Columns of type SQL Server
  `GEOGRAPHY` / `GEOMETRY` or PostgreSQL `ARRAY` / user-defined fail `translate` (that table fails, others go on).
  Oracle → PostgreSQL data exactness was not verified by the 0.7.3 release: suggest a `diff` and spot checks.
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

The package bundles FastTransfer in `engine/`, found automatically, and the MigratorXpress license also
licenses it: don't add `--fasttransfer_dir_path` or a `FastTransfer.lic` unless the user wants another
FastTransfer.

## Example

Oracle schema SALES to SQL Server, tables, data and validation, 4 tables in parallel:

```bash
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id oracle_prod \
  --source_schema_name SALES \
  --target_db_auth_id mssql_target \
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

- **Exit code**: `0` all tables succeeded, `40` some tables failed a task, `41` all tables failed; the log
  then prints the `--resume <run_id>` command. Differences found by `diff` are reported but don't fail a table.
  Config errors exit `10`, validation errors `20`, connection errors `12` (environment.md).
- Details: the `RUN SUMMARY: n/m tables succeeded` block at the end of the log
  (`mig_YYYYMMDD_HHMMSS_<run_id>.log`), or `sourcetargetloads` / `eventlog` in the tracking DB.
- To finish a run after failures: `--auth`, the three auth ids and `--resume <run_id>`.
- Upgrading from 0.6.x: the tracking DB is upgraded automatically on the first 0.7 run, or explicitly with
  `--auth … --migration_db_auth_id … --upgrade_migdb`. Back it up first. Don't `--resume` a PostgreSQL-target
  run started before 0.7.2 (names were not lower-cased then): start a new run.
