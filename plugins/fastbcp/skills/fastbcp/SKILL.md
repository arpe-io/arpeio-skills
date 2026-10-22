---
name: fastbcp
description: Build, fix, explain or run FastBCP command lines. FastBCP (by arpe.io) is a high-performance bulk export tool that reads from SQL Server, PostgreSQL, Oracle, MySQL, SAP HANA, Teradata, Netezza, ClickHouse, ODBC/OLE DB sources (including arpe.io ADBC drivers) and writes CSV, TSV, Parquet, JSON, BSON, Excel or gzip files to local disk, UNC shares or cloud storage (S3, Azure Blob, ADLS Gen2, GCS, OneLake). Use when the user mentions FastBCP, or wants to bulk export / extract / unload a database table or query to files or to object storage, choose a parallel export method, or troubleshoot a FastBCP run.
---

# FastBCP

FastBCP exports a database table or query to files, in parallel. One run = one source (table or
query) → one output file, or one file per parallel chunk / partition.

Goal: produce **one correct, copy-pasteable command** for the user's OS and shell, built only from
documented parameters. Never invent hosts, credentials, table or column names, or paths: ask.

Facts below reflect FastBCP **1.4**. Detailed references (load only what you need):

| Need | File |
|---|---|
| Connection types, server formats, auth, connection strings, platform support | [references/connections.md](references/connections.md) |
| Every CLI flag, defaults, tokens, YAML `--config` | [references/parameters.md](references/parameters.md) |
| Parallel methods, distribute key, degree, merge | [references/parallelism.md](references/parallelism.md) |
| Formats, CSV/Parquet options, cloud targets and credentials | [references/outputs.md](references/outputs.md) |
| Install, license, Docker, env vars, logging | [references/environment.md](references/environment.md) |
| Complete commands taken from the docs | [references/examples.md](references/examples.md) |
| Pitfalls, doc inconsistencies, version changes | [references/troubleshooting.md](references/troubleshooting.md) |

If something is not covered there, read the official docs instead of guessing:
https://fastbcp-docs.arpe.io/latest/llms-full.txt (whole documentation in one file).

## Check the installed version

When you can run commands, run `./FastBCP --version` (Windows: `.\FastBCP.exe --version`) before
building a command; otherwise ask for the version when it matters (a rejected flag, an unexpected error).

- **1.4** → use this skill as is.
- **1.3** → `Physical` and `adbc_teradata` don't exist yet: use the engine method name instead
  (`Ctid`, `Rowid`, `Physloc`, `NZDataSlice`, see parallelism.md) and `teradata`. Avoid `Physloc`
  with an even `--paralleldegree` before 1.4 (one chunk did all the work); use an odd degree.
- **`Unknown argument found: version`** → a very old build: run `./FastBCP '-?'` to see what it accepts,
  and suggest upgrading.
- **Older** → options added since may be missing (see the version notes in troubleshooting.md):
  check the flag there, or suggest upgrading.
- **Newer** → this skill may miss new options or changed defaults: confirm anything you rely on in
  the official docs above, and say that you did.

## Workflow

1. **Source** – engine → `--connectiontype`; server, database, auth (`--trusted`, `--user`/`--password`,
   or `--sourceconnectstring`). See connections.md.
2. **What to export** – `--sourceschema` + `--sourcetable`, or `--query` / `--fileinput`.
   Both can be combined (needed for `{sourceschema}`/`{sourcetable}` tokens or a filtered parallel export).
3. **Target** – `--directory` (local path, UNC, or cloud URI) + `--fileoutput`.
   **The file extension sets the format**: `.parquet .csv .tsv .json .bson .xlsx .bin .csv.gz .tsv.gz .json.gz`.
4. **Format options** – only those relevant to the format (CSV: delimiter, quotes, date format…; Parquet:
   compression). See outputs.md.
5. **Parallelism** – only for large tables. Pick the method with the table below.
6. **Ops** – `--runid`, `--loglevel`, `--nobanner` for scripts, license location.
7. **Output** – one fenced code block for the right shell, then a one-line summary and any assumption made.

## Choosing a parallel method

Size the table in **cells** (rows × columns, from catalog statistics); count wide columns
(`NVARCHAR(MAX)`, `XML`, `VARBINARY`, LOBs) as one size class larger.

| Situation | `--parallelmethod` | `--distributekeycolumn` |
|---|---|---|
| Under ~20M cells | `None` (default). Splitting usually costs more than it saves | – |
| Integer key leading a selective clustered index (identity…), no big gaps | `RangeId` (min/max ranges) | key column |
| Indexed integer, date or string key, gaps or skew | `Ntile` (even row counts; key need not be unique) | key column |
| Natively partitioned table, partitions of similar size (SQL Server, PostgreSQL, Oracle, MySQL, Teradata and their `adbc_*`) | `DBPartition` | – |
| No usable key, SQL Server / PostgreSQL / Oracle / Netezza (1.4+) | `Physical` | – |
| Date column, want `year=…/month=…` folders | `Timepartition` | `"(col, year, month)"` |
| One file per business value, an expression like `YEAR(col)`, or incremental extracts | `DataDriven` (+ optional `--datadrivenquery`) | column or expression |
| Engine without `Physical` (Teradata, MySQL, HANA, ClickHouse, ODBC/OLE DB) and no good key | `DataDriven` on a business column | column |

Don't recommend `Random` unless asked (kept for compatibility and benchmarks; often the slowest). `Ctid`, `Rowid`,
`Physloc` and `NZDataSlice` are **deprecated** in 1.4: write `Physical`, which picks the same strategy
from the connection type (only on 1.3 and older use the engine name). On SQL Server, `Physical` is
rarely the fastest: prefer `RangeId`/`Ntile` on a good clustered key. Details: parallelism.md.

Then set `--paralleldegree` (default `-2` = cores/2; `0` = all cores; `N>0` = exactly N threads,
capped at core count; `-N` = cores/N) and decide on `--merge` (see parallelism.md).

## Hard rules

- **Use long flags only.** Short flags are inconsistent across doc pages (e.g. `-d` is the CSV
  delimiter, not the database; database is `-I`). Long flags are unambiguous.
- **ADBC connectors** (`adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_teradata`) write **Parquet only**
  and run on Windows/Linux **x64 only**. If the user wants CSV with ADBC, explain and propose the
  non-ADBC connector.
- **Merge** (`--merge true`) works for CSV and Parquet only and **not for cloud destinations**.
  Its default is `false` since 0.32. For cloud targets, omit it or set `--merge false`.
- `--fileinput` SQL files must contain one query **without a trailing `;`**.
- `DBPartition` needs `--sourceschema` + `--sourcetable`, no key column; it falls back to `None`
  with a warning when the table is not partitioned.
- `Physical` (and the deprecated `Physloc`, `Rowid`, `Ctid`) needs `--sourceschema` + `--sourcetable`
  and no key column; a `--query` may only add a simple `WHERE`. It fails on `odbc`, `oledb`, `mysql`,
  `clickhouse`, `hana`, `teradata` and `adbc_teradata`.
- `adbc_teradata` refuses `--trusted`: use `--user`/`--password` (or a connection string).
- Methods `Random`, `DataDriven`, `RangeId`, `Ntile`, `Timepartition` require `--distributekeycolumn`.
  Ask for the column if unknown, never guess one.
- Parallel methods: the docs say `--sourceschema`/`--sourcetable` are mandatory. Set them whenever the
  table is known, even if a `--query` is also given.
- Quote every value containing spaces, backslashes, `$`, `&`, `|` or other shell characters.
  In PowerShell, escape a literal `$` with a backtick (e.g. `` "`$partition.PF(col)" ``).

## Secrets

- Never write a real password or license content into a command you show. Prefer `--trusted`
  (integrated auth); otherwise reference a shell variable the user sets beforehand, e.g.
  `--password "$FASTBCP_PASSWORD"` (bash) or `--password $env:FASTBCP_PASSWORD` (PowerShell).
  FastBCP does not read such variables by itself; the shell expands them.
- Cloud credentials come from `--cloudprofile` or the provider's standard env vars / managed identity,
  never from the command line. See outputs.md.
- FastBCP masks passwords in its own logs, but not in shell history.

## Shell conventions

| OS / shell | Executable | Line continuation |
|---|---|---|
| Linux / macOS (bash, zsh) | `./FastBCP` | `\` |
| Windows PowerShell | `.\FastBCP.exe` | `` ` `` (backtick) |
| Windows cmd.exe | `FastBCP.exe` | `^` |
| Docker | `docker run --rm arpeio/fastbcp:latest` | `\` |

Order arguments: connection → source → output → format → parallelism → advanced.

## Example

SQL Server table to Parquet on Linux, integrated auth, parallel without a key column:

```bash
./FastBCP \
  --connectiontype mssql \
  --server "sqlprod01" \
  --database "Sales" \
  --trusted \
  --sourceschema "dbo" \
  --sourcetable "Orders" \
  --directory "/exports/{sourcedatabase}/{sourcetable}" \
  --fileoutput "orders_{startdate}.parquet" \
  --parallelmethod Physical \
  --paralleldegree -2 \
  --runid "sales_orders_daily" \
  --nobanner
```

## Running and checking

- Check the version first (see above). Help is `./FastBCP --help` (1.4+, exit code 0) or
  `./FastBCP '-?'` (quoted: `?` is a shell wildcard). Before 1.4, `--help` printed the usage followed by
  parameter errors and exited with code 1; use `'-?'` there.
- Add `--loglevel Debug` to diagnose a failing run; reuse the scheduler's run id in `--runid`.
- For a quick test on a large table, export a filtered `--query` first (e.g. a `WHERE` on a date).
- Exit code: the docs don't list return codes. Treat any non-zero exit code as a failure and read the log.
