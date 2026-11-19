---
name: fasttransfer
description: Build, fix, explain or run FastTransfer command lines. FastTransfer (by arpe.io) is a high-performance bulk transfer tool that copies data in parallel between databases (SQL Server, PostgreSQL, Oracle, MySQL, ClickHouse, SAP HANA, Teradata, Netezza, IBM Db2, DuckDB, ODBC/OLE DB sources), natively or through the bundled arpe.io ADBC drivers, and imports CSV, Parquet and JSON files into a database through DuckDB. Use when the user mentions FastTransfer, or wants to copy / migrate / load a table or query from one database to another, bulk load files into a database, choose a parallel transfer method, or troubleshoot a FastTransfer run.
---

# FastTransfer

FastTransfer reads a table or query from a **source** and bulk loads it into an **existing table** on a
**target**, optionally in parallel. One run = one source (table, query or files) → one target table.

Goal: produce **one correct, copy-pasteable command** for the user's OS and shell, built only from
documented parameters. Never invent hosts, credentials, table or column names, or paths: ask.

Facts below reflect FastTransfer **1.0.0**. Detailed references (load only what you need):

| Need | File |
|---|---|
| Source and target connection types, ADBC (`adbc_*`), server formats, auth, connection strings | [references/connections.md](references/connections.md) |
| Every CLI flag with defaults and allowed values | [references/parameters.md](references/parameters.md) |
| Parallel methods, distribute key, degree | [references/parallelism.md](references/parallelism.md) |
| Importing CSV, Parquet, JSON files (DuckDB `read_*`) | [references/files.md](references/files.md) |
| Install, license, Docker, env vars, logging | [references/environment.md](references/environment.md) |
| Complete commands from the docs | [references/examples.md](references/examples.md) |
| Pitfalls, known limitations, version notes | [references/troubleshooting.md](references/troubleshooting.md) |

If something is not covered there, read the official docs instead of guessing:
https://fasttransfer-docs.arpe.io/latest/llms-full.txt. When that site still describes 0.18, the
reference for the 1.0 additions (`adbc_*`, `Physical`) is the `README.md` shipped in the FastTransfer zip.

## Check the installed version

When you can run commands, run `./FastTransfer --version` (Windows: `.\FastTransfer.exe --version`)
before building a command; otherwise ask for the version when it matters (a rejected flag, an unexpected
error).

- **1.0** → use this skill as is.
- **0.18** → no `adbc_*` types (Db2 goes through `odbc`) and no `Physical`: write the engine name
  instead (`Ctid`, `Rowid`, `Physloc`, `NZDataSlice`, see parallelism.md). Don't use `--useworktables`
  with `msbulk` / `pgcopy`, and avoid `Ntile` / `DataDriven` on a date or timestamp key or on a key
  holding NULLs: both lose rows without an error there (troubleshooting.md). Recommend 1.0.
  Before 0.18.2, always pass `--license` with the full path of the file.
- **0.17** → as 0.18, and `DBPartition`, `Timepartition` and `FASTTRANSFER_DATEFORMAT` don't exist yet:
  use `DataDriven` (e.g. on `YEAR(col)` or a `$PARTITION` function) or `Ntile` instead.
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
3. **Target** – engine → `--targetconnectiontype` (table below); `--targetserver`,
   `--targetdatabase`, auth; **`--targetschema` + `--targettable` are mandatory**.
4. **Load behaviour** – `--loadmode Append` (default) or `Truncate`; `--mapmethod Position` (default)
   or `Name`; optional `--batchsize`.
5. **Parallelism** – only for large volumes (docs: more than about 1M cells). Pick the method below.
6. **Ops** – `--runid`, `--loglevel`, `--nobanner` for scripts, and the license: a license file is
   required for every run, **including the free trial**. Leave `--license` out only when the file is
   `FastTransfer.lic` in the working directory or next to the executable; otherwise add
   `--license "<path|url>"` (or the content from a variable). When you leave it out, say so in the summary
   ("assumes `FastTransfer.lic` in the working directory or next to the binary"). `adbc_*` runs also need
   the ADBC driver block in that file (see connections.md).
7. **Output** – one fenced code block for the right shell, then a one-line summary and any assumption made.

## Choosing the connection types

| Engine | Source `--sourceconnectiontype` | Target `--targetconnectiontype` | ADBC, both ends (1.0+) |
|---|---|---|---|
| SQL Server | `mssql` | `msbulk` | `adbc_mssql` |
| PostgreSQL | `pgcopy` or `pgsql` | `pgcopy` (COPY; binary when the source is also `pgcopy`, text otherwise) or `pgsql` | `adbc_pgsql` |
| Oracle | `oraodp` | `orabulk` (bulk) or `oradirect` | `adbc_oracle` |
| Netezza | `nzsql` | `nzbulk` | `adbc_netezza` |
| Teradata | `teradata` | `teradata` | `adbc_teradata` |
| IBM Db2 for LUW | `odbc` (IBM driver) | – | `adbc_db2` |
| MySQL / MariaDB | `mysql` | `mysqlbulk` | – |
| ClickHouse | `clickhouse` | `clickhousebulk` | – |
| SAP HANA | `hana` | `hanabulk` | – |
| DuckDB file | `duckdb` | `duckdb` | – |
| Files (CSV, Parquet, JSON) | `duckdbstream` | – | – |

There is no ODBC or OLE DB target. Other sources: connections.md.

**ADBC or native?** The `adbc_*` types use the Arpe ADBC drivers bundled in FastTransfer 1.0 and work
**only as a pair**: `adbc_*` on the source **and** on the target (any two of the six). Take the first
rule that applies:

1. The user names the connection types, or already runs a command: keep them (and mention a known
   limitation of that pair if there is one).
2. Db2 is one end and the other end is one of the six ADBC engines: `adbc_*` on both ends.
3. Both ends are ADBC engines and the native pair has an entry in troubleshooting.md, "Known
   limitations in 1.0": when the user's table is known to hit it (e.g. the column types named there),
   write the `adbc_*` pair and say why; when unknown, write the native command and mention the
   limitation and the `adbc_*` alternative in one sentence. `adbc_*` needs the ADBC driver block in
   the license.
4. Otherwise: the native types.

Consequences of the pairing: Db2 to a native-only target (MySQL…) goes through an `odbc` source, nothing
loads into Db2 from a native-only source, and file imports (`duckdbstream`) always use a native target
(so not Db2).

## Choosing a parallel method

| Situation | `--parallelmethod` | Also needs |
|---|---|---|
| Small table / file | `None` (default) | – |
| Natively partitioned table, partitions of similar size (SQL Server, PostgreSQL, Oracle, MySQL, Teradata; Db2 for LUW range partitioning with `adbc_db2`, 1.0+) (0.18+) | `DBPartition` | `--sourceschema`, `--sourcetable`; no `--query` |
| Date column, one stream per year / month / day / ISO week (0.18+) | `Timepartition` | `--distributekeycolumn "(col,year,month)"` |
| PostgreSQL, Oracle or Netezza source; SQL Server source on **read-only** data (1.0+) | `Physical` | `--sourceschema`, `--sourcetable` (not needed on Netezza) |
| Any source, sortable key (numeric, date, string) | `Ntile` | `--distributekeycolumn` |
| Any source, numeric key without gaps (identity) | `RangeId` | `--distributekeycolumn` |
| Any source, few distinct values, an expression, or one chunk per file | `DataDriven` (+ optional `--datadrivenquery`) | `--distributekeycolumn` |
| Any source, numeric key, nothing better | `Random` | `--distributekeycolumn` (numeric) |

`Physical` picks the engine strategy from the source connection type (Ctid, Rowid, Physloc,
NZDataSlice). Those four names are **deprecated** in 1.0 (warning in the log, removed in 2.0): write
`Physical`, except on 0.18 and older, and for an `odbc` source to SQL Server (`Physical` refuses `odbc`:
keep `Physloc`). Then set `--degree` (default `-4` = cores/4; `0` = all cores; `N>0` = N threads;
`-N` = cores/N).

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
- Methods `Random`, `DataDriven`, `RangeId`, `Ntile`, `Timepartition` require `--distributekeycolumn`; FastTransfer stops
  with an error otherwise. Ask for the column if unknown; never guess one.
- `Physical` (and the deprecated `Ctid`, `Rowid`, `Physloc`) requires `--sourceschema` and
  `--sourcetable`; a `--query` can be added, to filter that table only. Netezza needs neither.
- `Physical` works with PostgreSQL (`pgsql`, `pgcopy`, `adbc_pgsql`), Oracle (`oraodp`, `adbc_oracle`),
  SQL Server (`mssql`, `msoledbsql`, `adbc_mssql`; only on data that is not being written) and Netezza
  (`nzsql`, `nzbulk`, `nzoledb`, `adbc_netezza`) sources. Any other source is refused: use a key-based method.
- `DBPartition` requires `--sourceschema` + `--sourcetable`; `--query` / `--fileinput` are not allowed. On a
  non-partitioned table or an unsupported source it falls back to `None` with a warning.
- **`adbc_*`**: never mix with a native type (refused before anything runs); Windows x64 and Linux x64
  only (not Linux ARM64); write `--targetschema` / `--targettable` in the exact stored case (`ORDERS` on
  Oracle, Db2 and Teradata, `orders` on PostgreSQL, unless the table was created with quoted names); no
  `--useworktables`; `--batchsize` has no effect.
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

Order arguments as in the example below: source connection → source schema / table / query → target
connection → target schema / table → load behaviour → parallelism → advanced.

## Example

PostgreSQL table to SQL Server on Linux, parallel on the physical row location (resolved to Ctid),
target emptied first:

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
  --parallelmethod Physical \
  --degree -2 \
  --runid "pg_to_mssql_orders" \
  --nobanner
```

## Running and checking

- Check the version first (see above). `./FastTransfer '-?'` prints the help (quote `-?`: `?` is a shell
  wildcard).
- Exit code 0 = success, non-zero = failure: 99 = no or invalid license (or no ADBC driver block for an
  `adbc_*` run), 100 = license expired. Otherwise read the log.
- Use `--loglevel debug` to diagnose; reuse the scheduler's run id in `--runid`.
- On a large table, try a filtered `--query` first to validate mapping and types.
