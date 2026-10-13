---
name: lakexpress
description: Build, fix, explain or run LakeXpress commands and credentials files. LakeXpress (by arpe.io) exports database tables (Oracle, PostgreSQL, SQL Server, MySQL, MariaDB, SAP HANA, Teradata) to Parquet files with FastBCP, on local disk or S3 / GCS / Azure / OneLake, and optionally publishes them as tables in Snowflake, Databricks, Microsoft Fabric, BigQuery, Amazon Redshift, MotherDuck, AWS Glue or DuckLake. Runs are stored as sync configurations in a LakeXpress DB, with incremental (watermark) syncs and resume. Use when the user mentions LakeXpress, or wants to export a database or schemas to Parquet / a data lake / a lakehouse, publish them to one of those platforms, set up incremental syncs, resume a failed export, or troubleshoot a LakeXpress run.
---

# LakeXpress

LakeXpress exports **whole schemas** to Parquet, in parallel, then optionally **publishes** them to a
data platform. It works in two steps: `config create` stores a **sync configuration** (what to export,
where, where to publish) in the **LakeXpress DB**, then `sync` runs it, as often as needed.

Goal: produce a **credentials file** (if needed) and **correct, copy-pasteable commands** for the
user's OS and shell. Never invent hosts, credentials, schema, bucket or account names: ask.

Facts below reflect LakeXpress **0.4.10** (docs checked against the v0.4.10 code). References (load only
what you need):

| Need | File |
|---|---|
| Credentials file: sources, LakeXpress DB, storage, publish targets, `$env{}` | [references/credentials.md](references/credentials.md) |
| Every command and flag, defaults | [references/commands.md](references/commands.md) |
| Publishing: per-platform setup, external / internal, naming patterns | [references/publishing.md](references/publishing.md) |
| Incremental syncs, watermarks, loading strategies, locks | [references/incremental.md](references/incremental.md) |
| Parallel export methods, per-table FastBCP config, output layout | [references/export.md](references/export.md) |
| Install, license, FastBCP, LakeXpress DB tables, logging, exit codes | [references/environment.md](references/environment.md) |
| Complete commands | [references/examples.md](references/examples.md) |
| Known behaviours, doc errors, common errors | [references/troubleshooting.md](references/troubleshooting.md) |

Official docs: https://lakexpress-docs.arpe.io/latest/

## The pieces (all in one credentials file)

| Role | Flag | Supported (`ds_type`) |
|---|---|---|
| LakeXpress DB (configs, runs, watermarks) | `--lxdb_auth_id` (every command) | `postgres` (recommended), `mssql`, `mysql`, `sqlite`, `duckdb` |
| Source | `--source_db_auth_id` | `oracle`, `postgres`, `mssql`, `mysql`, `mariadb`, `saphana`, `teradata` |
| Storage | `--output_dir PATH` (local) or `--target_storage_id` | `s3` (also MinIO, OVH…), `gcs`, `azure`, `onelake` |
| Publish target (optional) | `--publish_target` | `snowflake`, `databricks`, `fabric`, `bigquery`, `redshift`, `motherduck`, `aws_glue` / `glue`, DuckLake catalog (`sqlite`, `duckdb`, `postgres`, `mysql`) |

Each `*_id` is a top-level key of the JSON file passed with `--auth_file`. Any string can be
`$env{VAR}` (credentials.md).

## Check the installed version

When you can run commands, run `./LakeXpress --version` (Windows: `.\LakeXpress.exe --version`); it
prints e.g. `LakeXpress 0.4.10 (build 2026-09-24T12:04:58Z)`. Otherwise ask for the version when it
matters (a rejected option, an unexpected exit code).

- **0.4.10** → use this skill as is.
- **Older** → some behaviours differ (relative `--output_dir` before 0.4.9, SQL Server `pymssql` fix in
  0.4.8, Linux OpenSSL crash in 0.4.5–0.4.6: see troubleshooting.md); check `./LakeXpress <command>
  --help`, or suggest upgrading.
- **Newer** → this skill may miss new options or changed behaviour: confirm anything you rely on in
  `--help` and https://lakexpress-docs.arpe.io/latest/llms-full.txt (whole documentation in one file),
  and say that you did.

## Workflow

1. **LakeXpress DB** – once: an empty database owned by its user (PostgreSQL, SQL Server, MySQL) or a
   file path (SQLite, DuckDB), then `lxdb init` (optional, checks the connection).
2. **Credentials file** – one entry per LakeXpress DB, source, storage and publish target.
3. **`config create`** – required: `--auth_file`, `--lxdb_auth_id`, `--source_db_auth_id`, and
   `--output_dir` or `--target_storage_id`. Scope: `--source_schema_name` (comma list, `LIKE` patterns,
   default all schemas), `--include` / `--exclude` (table `LIKE` patterns), `--min_rows` / `--max_rows`.
   Performance: `--n_jobs` (tables in parallel) × `--fastbcp_p` (processes per table ≥
   `--large_table_threshold` rows). Publishing: `--publish_target`, `--publish_method`, naming patterns.
   Incremental: `--incremental_table` (repeat). Name it with `--sync_id` for scripts.
4. **`sync`** – `--auth_file`, `--lxdb_auth_id`, `--sync_id`: exports, then publishes if configured.
   `'sync[export]'` / `'sync[publish]'` run one step. A sync configuration cannot be edited: delete it
   (`config delete … --confirm`) and create it again (incremental watermarks restart from scratch).
5. **Output** – the credentials JSON (placeholders or `$env{}` for secrets), the `config create` command,
   the `sync` command with `<SYNC_ID>` (or the `--sync_id` chosen), then a short summary with
   assumptions and what to check (exit code, `status`).

## Hard rules (0.4.10)

- **Options go after the command** (`LakeXpress sync --log_level DEBUG …`); only `--version` and
  `--no_banner` may come before it. Quote `'sync[export]'` and `'sync[publish]'`.
- **Every command needs `--auth_file` and `--lxdb_auth_id`** (except legacy `run`). `sync` takes no
  source, storage or publish flags: they live in the configuration.
- **Without `--output_dir` / `--target_storage_id`, `config create` succeeds but every `sync` fails**
  (exit 10). Invalid publish patterns are also only caught by `sync` (exit 10).
- **Publish naming**: `--publish_table_pattern` must contain `{table}`; `--publish_schema_pattern` must
  not. Defaults: schema `EXT_{schema}` for `external` (the default method), `{schema}` for `internal`;
  table `{table}`. Glue uses the schema pattern as database name; BigQuery as dataset name.
- **External vs internal**: Glue and DuckLake are external only. Fabric needs `"fabric_target":
  "lakehouse"` in its entry for the Lakehouse; without it the Fabric **Warehouse** publisher is used.
- **Snowflake**: the storage `directory` must equal the Snowflake stage URL.
- **Incremental**: types `date`, `datetime`, `int`, `bigint`, `auto` only (not `timestamp`). An invalid
  spec still creates the configuration (without it): check the exit code, then `config delete` and
  recreate. `upsert` / `delete_insert` need a source primary key and are not supported on Glue or
  Fabric Lakehouse. Use a PostgreSQL, SQL Server or SQLite LakeXpress DB for incremental syncs (with
  DuckDB the incremental tables are not saved, exit 0).
- **SQLite / DuckDB LakeXpress DB**: absolute `filepath`, one machine, one sync at a time.
- **`--error_action fail`** (default) stops at the first failed table; `continue` exports the others.
- Use **long flags** in generated commands (`--auth_file`, `--include`, `--exclude`, `--fastbcp_p`), not
  `-a`, `-i`, `-e`, `-p`.
- Never put real secrets in what you show; prefer `$env{VAR}` and tell the user to `chmod 600` the file.

## Shell conventions

| OS / shell | Executable | Line continuation |
|---|---|---|
| Linux (bash, zsh) | `./LakeXpress` | `\` |
| Windows PowerShell | `.\LakeXpress.exe` | `` ` `` |

Trial archives bundle FastBCP in `engine/`. The full edition does not: add `--fastbcp_dir_path` to
`config create` (FastBCP needs its own `FastBCP.lic` next to its binary).

## Example

PostgreSQL schema `sales` to S3, published as Snowflake internal tables, 4 tables in parallel:

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres \
  --source_schema_name sales \
  --target_storage_id s3_01 \
  --publish_target snowflake_prod \
  --publish_method internal \
  --sync_id sales-to-snowflake \
  --n_jobs 4 \
  --fastbcp_p 2

./LakeXpress sync \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --sync_id sales-to-snowflake
```

## After a run

- **Exit code**: `0` success (also when no table matched: check the log), `40` some tables failed
  (export or publish), `10` configuration error, `12` connection error, `13` license (environment.md).
- `status --sync_id <SYNC_ID>` (add `--verbose` for every run); the log `lx_YYYYMMDD_HHMMSS_<run_id>.log`
  ends with `RUN SUMMARY` / `Export: n/m tables succeeded` and the failed tables.
- To retry failed tables: `sync --sync_id <SYNC_ID> --resume` (latest incomplete run, or add
  `--run_id <RUN_ID>`). It publishes when the resumed run completes.
- After a run with failures, `sync` publishes the **latest completed run**, which is an earlier run if
  there is one: resume first, or publish a given run with `'sync[publish]' --run_id`.
- A crashed run blocks the next ones (`already running`): `cleanup --sync_id <SYNC_ID>`; for incremental
  locks, `lxdb locks` then `lxdb release-locks --confirm`.
