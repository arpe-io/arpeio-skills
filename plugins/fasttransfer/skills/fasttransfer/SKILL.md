---
name: fasttransfer
description: Build, fix, explain or run FastTransfer command lines. FastTransfer (by arpe.io) is a high-performance bulk transfer tool that copies data in parallel between databases (SQL Server, PostgreSQL, Oracle, MySQL, ClickHouse, SAP HANA, Teradata, Netezza, IBM Db2, DuckDB, ODBC/OLE DB sources), natively or through the bundled arpe.io ADBC drivers, loads tables straight from SQL Server backup files (.bak) and Oracle RMAN backup sets without restoring them, imports CSV, Parquet and JSON files into a database through DuckDB, and can create the target table. Use when the user mentions FastTransfer, or wants to copy / migrate / load a table or query from one database to another, bulk load files into a database, choose a parallel transfer method, or troubleshoot a FastTransfer run.
---

# FastTransfer

FastTransfer reads a table or query from a **source** and bulk loads it into a **table** on a
**target**, optionally in parallel. The table exists already, or FastTransfer creates it
(`--loadmode Create`, 1.1+). One run = one source (table, query, files or backup) → one target table.

Goal: produce **one correct, copy-pasteable command** for the user's OS and shell, built only from
documented parameters. Never invent hosts, credentials, table or column names, or paths: ask.

Facts below reflect FastTransfer **1.1.0**. Detailed references (load only what you need):

| Need | File |
|---|---|
| Source and target connection types, ADBC (`adbc_*`), backup sources (`backup_*`), server formats, auth, connection strings | [references/connections.md](references/connections.md) |
| Every CLI flag with defaults and allowed values; creating the target table (`--loadmode Create`, `--inferdatatype`) | [references/parameters.md](references/parameters.md) |
| Parallel methods, distribute key, degree | [references/parallelism.md](references/parallelism.md) |
| Importing CSV, Parquet, JSON files (DuckDB `read_*`) | [references/files.md](references/files.md) |
| Install, license, Docker, env vars, logging | [references/environment.md](references/environment.md) |
| Complete commands from the docs | [references/examples.md](references/examples.md) |
| Pitfalls, known limitations, version notes | [references/troubleshooting.md](references/troubleshooting.md) |

If something is not covered there, read the official docs instead of guessing:
https://fasttransfer-docs.arpe.io/latest/llms-full.txt. When `latest` still describes 1.0, the 1.1
additions (`--loadmode Create`, `--inferdatatype`, `backup_*`) are in the 1.1 pages of that site
(https://fasttransfer-docs.arpe.io/1.1/) and in the `README.md` shipped in the FastTransfer zip.

## Check the installed version

When you can run commands, run `./FastTransfer --version` (Windows: `.\FastTransfer.exe --version`)
before building a command; otherwise ask for the version when it matters (a rejected flag, an unexpected
error).

- **1.1** → use this skill as is.
- **1.0** → no `--loadmode Create` and no `--inferdatatype`: the target table must exist, and
  `--loadmode Replace` behaves like `Append` there. No `backup_mssql` / `backup_rman` sources. No
  `FASTTRANSFER_ADBC_SSLMODE`: put the driver's `sslmode` in the connection string. With the native SQL
  Server types write a port as `host,port` (`host:port` is accepted from 1.1). A `--query` holding
  `QUALIFY`, a named `WINDOW` or `CONNECT BY` loses that clause when a parallel method adds its
  predicate, and `Ntile` on an Oracle or MySQL source fails with a query and splits the rows wrongly
  with a table (troubleshooting.md). Recommend 1.1.
- **0.18** → as 1.0, and no `adbc_*` types (Db2 goes through `odbc`) and no `Physical`: write the engine
  name instead (`Ctid`, `Rowid`, `Physloc`, `NZDataSlice`, see parallelism.md). Don't use `--useworktables`
  with `msbulk` / `pgcopy`, and avoid `Ntile` / `DataDriven` on a date or timestamp key or on a key
  holding NULLs: both lose rows without an error there (troubleshooting.md). Recommend 1.1.
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
   For a SQL Server `.bak` or an Oracle RMAN backup (1.1+): `backup_mssql` / `backup_rman`,
   `--sourceserver` = the directory of the backup files, `--sourcedatabase` = their names or patterns
   (`;` separated), no database credentials, and an `adbc_*` target (see connections.md).
2. **What to read** – `--sourceschema` + `--sourcetable`, or `--query` / `--fileinput`. If both a table
   and a query are given, the query is used (keep schema/table when the parallel method needs them).
3. **Target** – engine → `--targetconnectiontype` (table below); `--targetserver`,
   `--targetdatabase`, auth; **`--targetschema` + `--targettable` are mandatory**.
4. **Load behaviour** – `--loadmode Append` (default), `Truncate`, or `Create` (1.1+: creates the target
   table from the source when it is missing, truncates it when it exists; `--inferdatatype` chooses how
   the column types are derived, see parameters.md); `--mapmethod Position` (default) or `Name`;
   optional `--batchsize`.
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
| SQL Server backup files (`.bak`) (1.1+) | `backup_mssql` | – | target must be `adbc_*` |
| Oracle RMAN backup sets (1.1+) | `backup_rman` | – | target must be `adbc_*` |

There is no ODBC or OLE DB target. Other sources: connections.md.

**ADBC or native?** The `adbc_*` types use the Arpe ADBC drivers bundled since FastTransfer 1.0 and work
**only as a pair**: `adbc_*` on the source **and** on the target (any two of the six). Take the first
rule that applies:

1. The user names the connection types, or already runs a command: keep them (and mention a known
   limitation of that pair if there is one).
2. Db2 is one end and the other end is one of the six ADBC engines: `adbc_*` on both ends.
3. Both ends are ADBC engines and the native pair has an entry in troubleshooting.md, "Known
   limitations in 1.1": when the user's table is known to hit it (e.g. the column types named there),
   write the `adbc_*` pair and say why; when unknown, write the native command and mention the
   limitation and the `adbc_*` alternative in one sentence. `adbc_*` needs the ADBC driver block in
   the license.
4. Otherwise: the native types.

Consequences of the pairing: Db2 to a native-only target (MySQL…) goes through an `odbc` source, nothing
loads into Db2 from a native-only source, and file imports (`duckdbstream`) always use a native target
(so not Db2). A backup source (`backup_mssql`, `backup_rman`, 1.1+) is the one non-`adbc_*` source an
`adbc_*` target accepts, and it loads into nothing else.

## Choosing a parallel method

| Situation | `--parallelmethod` | Also needs |
|---|---|---|
| Small table / file | `None` (default) | – |
| Natively partitioned table, partitions of similar size (SQL Server, PostgreSQL, Oracle, MySQL, Teradata; Db2 for LUW range partitioning with `adbc_db2`, 1.0+) (0.18+) | `DBPartition` | `--sourceschema`, `--sourcetable`; no `--query` |
| Date column, one stream per year / month / day / ISO week (0.18+) | `Timepartition` | `--distributekeycolumn "(col,year,month)"` |
| PostgreSQL, Oracle or Netezza source; SQL Server source on **read-only** data (1.0+) | `Physical` | `--sourceschema`, `--sourcetable` (not needed on Netezza) |
| SQL Server or Oracle RMAN backup (`backup_*`, 1.1+), any size worth splitting | `Physical` (resolved to `Pages`); only `None` and `Physical` are accepted | – |
| Any source, sortable key (numeric, date, string) | `Ntile` | `--distributekeycolumn` |
| Any source, numeric key without gaps (identity) | `RangeId` | `--distributekeycolumn` |
| Any source, few distinct values, an expression, or one chunk per file | `DataDriven` (+ optional `--datadrivenquery`) | `--distributekeycolumn` |
| Any source, numeric key, nothing better | `Random` | `--distributekeycolumn` (numeric) |

`Physical` picks the engine strategy from the source connection type (Ctid, Rowid, Physloc,
NZDataSlice; Pages for a backup source). The first four names are **deprecated** since 1.0 (warning in
the log, removed in 2.0): write
`Physical`, except on 0.18 and older, and for an `odbc` source to SQL Server (`Physical` refuses `odbc`:
keep `Physloc`). Then set `--degree` (default `-4` = cores/4; `0` = all cores; `N>0` = N threads;
`-N` = cores/N).

## Hard rules

- **Use long flags only.** Never generate short flags (`-c`, `-i`, `-s`…).
- **Exact flag names** (the parser rejects near-misses):
  `--sourceconnectstring` / `--targetconnectstring` (not `…connectionstring`), `--degree` (not
  `--paralleldegree`), `--distributekeycolumn` (lowercase; camelCase is a legacy alias), `--fileinput`,
  `--sourcedsn`. Use `--parallelmethod` (`--method` is an accepted alias).
- **The target table must already exist** with a compatible structure, unless `--loadmode Create` is
  used (1.1+). `Create` needs the target schema to exist, creates a table with the source column names,
  every column nullable, no key, index or constraint, and **truncates the table when it already
  exists**. Before 1.1 FastTransfer never creates a table: tell the user to create it first.
- **Never generate `--loadmode Replace`.** Since 1.1 it is an alias of `Create` (it creates the target,
  or **truncates** it); up to 1.0 it behaved like `Append`. Write `Append`, `Truncate` or `Create`, and
  warn a user whose scripts pass `Replace` before they upgrade.
- `--inferdatatype` (`fast` by default, `allvarchar`, `best`) only applies with `--loadmode Create`;
  with another load mode it is ignored with a warning. `best` is for an Oracle source and for files; it
  gives the same types as `fast` elsewhere (parameters.md).
- Column mapping is by **position** unless `--mapmethod Name` is set. Use `Name` when the source column
  order differs from the target, and for file imports.
- Methods `Random`, `DataDriven`, `RangeId`, `Ntile`, `Timepartition` require `--distributekeycolumn`; FastTransfer stops
  with an error otherwise. Ask for the column if unknown; never guess one.
- `Physical` (and the deprecated `Ctid`, `Rowid`, `Physloc`) requires `--sourceschema` and
  `--sourcetable`; a `--query` can be added, to filter that table only. Netezza needs neither.
- `Physical` works with PostgreSQL (`pgsql`, `pgcopy`, `adbc_pgsql`), Oracle (`oraodp`, `adbc_oracle`),
  SQL Server (`mssql`, `msoledbsql`, `adbc_mssql`; only on data that is not being written) and Netezza
  (`nzsql`, `nzbulk`, `nzoledb`, `adbc_netezza`) sources, and with the backup sources (1.1+). Any other
  source is refused: use a key-based method.
- `DBPartition` requires `--sourceschema` + `--sourcetable`; `--query` / `--fileinput` are not allowed. On a
  non-partitioned table or an unsupported source it falls back to `None` with a warning.
- **`adbc_*`**: never mix with a native type (refused before anything runs); Windows x64 and Linux x64
  only (not Linux ARM64); write `--targetschema` / `--targettable` in the exact stored case (`ORDERS` on
  Oracle, Db2 and Teradata, `orders` on PostgreSQL, unless the table was created with quoted names); no
  `--useworktables`; `--batchsize` has no effect.
- **Backup sources** (`backup_mssql`, `backup_rman`, 1.1+): `adbc_*` target only (another target is
  refused before anything runs); Windows x64 and Linux x64 only; `--parallelmethod None` or `Physical`
  only. The source is `--sourceschema` + `--sourcetable`, or a `--query` of the form
  `SELECT * | col [, col...] FROM [schema.]table` (`TOP n` too with `backup_mssql`): no `WHERE`, no
  join. `--sourceuser` / `--sourcepassword` are not database credentials: they name the certificate
  key file of an encrypted SQL Server backup and its password. Give the full backup with its
  differentials and every stripe (`backup_mssql`), or the level 0 with its incrementals
  (`backup_rman`): the latest version the files hold is read. Before proposing
  `FASTTRANSFER_BACKUP_RESTORE_MODE=best_effort` or `FASTTRANSFER_BACKUP_RESTORE_POINT=full`, say that
  they can read an older version of the table (connections.md).
- SQL Server `--sourceserver` / `--targetserver`: `host`, `host:port` (1.1+), `host,port`,
  `host\INSTANCE` or `host\INSTANCE,port`. Never `tcp:host` nor `host\INSTANCE:port`: 1.1 refuses them
  before any connection.
- `--loadmode Truncate` empties the target table first, and so does `Create` when the table exists.
  Confirm with the user before suggesting either on a table that may hold data they want to keep.
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
