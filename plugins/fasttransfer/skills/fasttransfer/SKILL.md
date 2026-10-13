---
name: fasttransfer
description: Build, fix, explain or run FastTransfer command lines. FastTransfer (by arpe.io) is a high-performance bulk transfer tool that copies data in parallel between databases (SQL Server, PostgreSQL, Oracle, MySQL, ClickHouse, SAP HANA, Teradata, Netezza, DuckDB, ODBC/OLE DB sources) and imports CSV, Parquet and JSON files into a database through DuckDB. Use when the user mentions FastTransfer, or wants to copy / migrate / load a table or query from one database to another, bulk load files into a database, choose a parallel transfer method, or troubleshoot a FastTransfer run.
---

# FastTransfer

FastTransfer reads a table or query from a **source** and bulk loads it into an **existing table** on a
**target**, optionally in parallel. One run = one source (table, query or files) → one target table.

Goal: produce **one correct, copy-pasteable command** for the user's OS and shell, built only from
documented parameters. Never invent hosts, credentials, table or column names, or paths: ask.

Facts below reflect FastTransfer **0.17** (docs + CLI source). Detailed references (load only what you need):

| Need | File |
|---|---|
| Source and target connection types, server formats, auth, connection strings | [references/connections.md](references/connections.md) |
| Every CLI flag with defaults and allowed values | [references/parameters.md](references/parameters.md) |
| Parallel methods, distribute key, degree | [references/parallelism.md](references/parallelism.md) |
| Importing CSV, Parquet, JSON files (DuckDB `read_*`) | [references/files.md](references/files.md) |
| Install, license, Docker, env vars, logging | [references/environment.md](references/environment.md) |
| Complete commands from the docs | [references/examples.md](references/examples.md) |
| Pitfalls, version notes | [references/troubleshooting.md](references/troubleshooting.md) |

If something is not covered there, read the official docs instead of guessing:
https://fasttransfer-docs.arpe.io/latest/llms-full.txt

## Check the installed version

When you can run commands, run `./FastTransfer --version` (Windows: `.\FastTransfer.exe --version`)
before building a command; otherwise ask for the version when it matters (a rejected flag, an unexpected
error).

- **0.17** → use this skill as is.
- **Older** → options added since may be missing (see the version notes in troubleshooting.md): check
  the flag in `./FastTransfer '-?'`, or suggest upgrading.
- **Newer** → this skill may miss new options or changed defaults: confirm anything you rely on in
  the official docs above, and say that you did.

## Workflow

1. **Source** – engine → `--sourceconnectiontype`; `--sourceserver`, `--sourcedatabase`, auth
   (`--sourcetrusted`, `--sourceuser`/`--sourcepassword`, or `--sourceconnectstring`).
   For files: `--sourceconnectiontype duckdbstream --sourceserver ":memory:"` + a `read_csv` /
   `read_parquet` / `read_json` query (see files.md).
2. **What to read** – `--sourceschema` + `--sourcetable`, or `--query` / `--fileinput`. If both a table
   and a query are given, the query is used (keep schema/table when the parallel method needs them).
3. **Target** – engine → `--targetconnectiontype` (bulk loader, table below); `--targetserver`,
   `--targetdatabase`, auth; **`--targetschema` + `--targettable` are mandatory**.
4. **Load behaviour** – `--loadmode Append` (default) or `Truncate`; `--mapmethod Position` (default)
   or `Name`; optional `--batchsize`.
5. **Parallelism** – only for large volumes (docs: more than about 1M cells). Pick the method below.
6. **Ops** – `--runid`, `--loglevel`, `--nobanner` for scripts, license location.
7. **Output** – one fenced code block for the right shell, then a one-line summary and any assumption made.

## Choosing the target connection type

| Target | `--targetconnectiontype` |
|---|---|
| SQL Server | `msbulk` |
| PostgreSQL | `pgcopy` (COPY; binary when the source is also `pgcopy`, text otherwise) or `pgsql` |
| Oracle | `orabulk` (bulk) or `oradirect` |
| MySQL / MariaDB | `mysqlbulk` |
| ClickHouse | `clickhousebulk` |
| SAP HANA | `hanabulk` |
| Teradata | `teradata` |
| Netezza | `nzbulk` |
| DuckDB file | `duckdb` |

There is no ODBC or OLE DB target.

## Choosing a parallel method

| Situation | `--parallelmethod` | Also needs |
|---|---|---|
| Small table / file | `None` (default) | – |
| PostgreSQL source (`pgsql`, `pgcopy`) | `Ctid` | `--sourceschema`, `--sourcetable` |
| SQL Server source (`mssql`), **read-only** table / DB / snapshot | `Physloc` | – |
| Oracle source (`oraodp`) | `Rowid` | `--sourceschema`, `--sourcetable` |
| Netezza source (`nzsql`) | `NZDataSlice` | `--sourceschema`, `--sourcetable` |
| Any source, sortable key (numeric, date, string) | `Ntile` | `--distributekeycolumn` |
| Any source, numeric key without gaps (identity) | `RangeId` | `--distributekeycolumn` |
| Any source, few distinct values, an expression, or one chunk per file | `DataDriven` (+ optional `--datadrivenquery`) | `--distributekeycolumn` |
| Any source, numeric key, nothing better | `Random` | `--distributekeycolumn` (numeric) |

Then set `--degree` (default `-4` = cores/4; `0` = all cores; `N>0` = N threads; `-N` = cores/N).

## Hard rules

- **Use long flags only.** Never generate short flags (`-c`, `-i`, `-s`…).
- **Exact flag names** (the parser rejects near-misses):
  `--sourceconnectstring` / `--targetconnectstring` (not `…connectionstring`), `--degree` (not
  `--paralleldegree`), `--distributekeycolumn` (lowercase; camelCase is a legacy alias), `--fileinput`,
  `--sourcedsn`. Use `--parallelmethod` (`--method` is an accepted alias).
- **The target table must already exist** with a compatible structure. FastTransfer never creates it.
  If unsure, tell the user to create it first.
- Column mapping is by **position** unless `--mapmethod Name` is set. Use `Name` when the source column
  order differs from the target, and for file imports.
- Methods `Random`, `DataDriven`, `RangeId`, `Ntile` require `--distributekeycolumn`; FastTransfer stops
  with an error otherwise. Ask for the column if unknown; never guess one.
- `Ctid`, `Rowid`, `NZDataSlice` require `--sourceschema` and `--sourcetable`, even when a `--query` is given.
- `Ctid` only with `pgsql`/`pgcopy` sources; `Rowid` only with `oraodp`; `Physloc` only with `mssql`/`odbc`
  (SQL Server), and only on data that is not being written.
- `--loadmode Truncate` empties the target table first. Confirm with the user before suggesting it on a
  table that may hold data they want to keep.
- `--fileinput` points to a file containing the SQL query; the file must exist.
- Quote values with spaces, backslashes, `$`, `&`, `|`. Inside `--query`, double a `"` to escape it
  (`"select * from ""tpch_10"".orders"`). In PowerShell escape `$` with a backtick (`` "`$PARTITION.PF(col)" ``).
- FastTransfer copies between databases and imports files. To **export** a database to files or to cloud
  storage, the arpe.io tool is FastBCP, not FastTransfer.

## Secrets

- Never put a real password or license content in a command you show. Prefer `--sourcetrusted` /
  `--targettrusted`; otherwise reference shell variables set beforehand, e.g.
  `--sourcepassword "$FT_SOURCE_PASSWORD"` (bash) or `--sourcepassword $env:FT_SOURCE_PASSWORD`
  (PowerShell). FastTransfer does not read such variables itself; the shell expands them.
- FastTransfer masks passwords in its logs, but not in shell history.

## Shell conventions

| OS / shell | Executable | Line continuation |
|---|---|---|
| Linux / macOS (bash, zsh) | `./FastTransfer` | `\` |
| Windows PowerShell | `.\FastTransfer.exe` | `` ` `` |
| Windows cmd.exe | `FastTransfer.exe` | `^` |
| Docker | `docker run --rm arpeio/fasttransfer:latest` | `\` |

Order arguments: source → target → what to transfer → load behaviour → parallelism → advanced.

## Example

PostgreSQL table to SQL Server on Linux, parallel with Ctid, target emptied first:

```bash
./FastTransfer \
  --sourceconnectiontype pgcopy \
  --sourceserver "pg01:5432" \
  --sourcedatabase "sales" \
  --sourceuser "$FT_SOURCE_USER" \
  --sourcepassword "$FT_SOURCE_PASSWORD" \
  --sourceschema "public" \
  --sourcetable "orders" \
  --targetconnectiontype msbulk \
  --targetserver "sqlprod01" \
  --targetdatabase "dwh" \
  --targettrusted \
  --targetschema "stg" \
  --targettable "orders" \
  --loadmode Truncate \
  --mapmethod Name \
  --parallelmethod Ctid \
  --degree -2 \
  --runid "pg_to_mssql_orders" \
  --nobanner
```

## Running and checking

- Check the version first (see above). `./FastTransfer '-?'` prints the help (quote `-?`: `?` is a shell
  wildcard).
- Exit code 0 = success, non-zero = failure. Read the log on failure.
- Use `--loglevel debug` to diagnose; reuse the scheduler's run id in `--runid`.
- On a large table, try a filtered `--query` first to validate mapping and types.
