# FastTransfer – CLI parameter reference (1.1)

Generate **long flags only**, spelled exactly as below.

## Source

| Flag | Value |
|---|---|
| `--sourceconnectiontype` | `mssql`, `msoledbsql`, `pgsql`, `pgcopy`, `mysql`, `oraodp`, `hana`, `teradata`, `nzsql`, `nzbulk`, `nzoledb`, `clickhouse`, `duckdb`, `duckdbstream`, `odbc`, `oledb`; 1.0+: `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_db2`, `adbc_teradata` (only with an `adbc_*` target); 1.1+: `backup_mssql`, `backup_rman` (only with an `adbc_*` target) |
| `--sourceserver` | server (see [connections.md](connections.md)); `backup_*`: directory of the backup files |
| `--sourcedatabase` | database; `backup_*`: backup file names or patterns, `;` separated |
| `--sourcetrusted` | switch: integrated auth |
| `--sourceuser` / `--sourcepassword` | credentials; `backup_mssql`: certificate key file of an encrypted backup, and its password |
| `--sourceconnectstring` | full connection string |
| `--sourcedsn` | ODBC DSN |
| `--sourceprovider` | OLE DB provider |

## What to transfer

| Flag | Value |
|---|---|
| `--sourceschema` | schema (required by `DBPartition`, and by `Physical` on PostgreSQL, Oracle and SQL Server) |
| `--sourcetable` | table (required by `DBPartition`, and by `Physical` on PostgreSQL, Oracle and SQL Server) |
| `--query` | SQL query; used instead of the source table when given. With `Physical` it filters the source table. A backup source only takes `SELECT *` or a column list `FROM [schema.]table` |
| `--fileinput` | path to a file containing the SQL query (must exist) |

A table or a query is mandatory. `--query` and `--fileinput` are mutually exclusive, and neither is
allowed with `--parallelmethod DBPartition`.

## Target

| Flag | Value |
|---|---|
| `--targetconnectiontype` | `msbulk`, `pgcopy`, `pgsql`, `orabulk`, `oradirect`, `mysqlbulk`, `clickhousebulk`, `hanabulk`, `teradata`, `nzbulk`, `duckdb`; 1.0+: `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_db2`, `adbc_teradata` (only with an `adbc_*` source) |
| `--targetserver` | server, or DuckDB file path |
| `--targetdatabase` | database |
| `--targettrusted` | switch: integrated auth |
| `--targetuser` / `--targetpassword` | credentials |
| `--targetconnectstring` | full connection string |
| `--targetschema` | **mandatory**; exact stored case with an `adbc_*` target |
| `--targettable` | **mandatory**; the table must already exist, unless `--loadmode Create` (1.1+); exact stored case with an `adbc_*` target |

## Load behaviour

| Flag | Default | Values |
|---|---|---|
| `--loadmode` | `Append` | `Append`, `Truncate` (empties the target table first), `Create` (1.1+: creates the target table from the source when it does not exist, then loads it; when the table exists, truncates it and loads it). `Replace` is an alias of `Create` since 1.1 and behaved like `Append` before: don't generate it |
| `--inferdatatype` (1.1+) | `fast` | With `--loadmode Create`, how the column types of the created table are derived: `fast` (source metadata, no row read), `allvarchar` (every column as text), `best` (profiling scan of the source data: Oracle sources and files read through DuckDB; as `fast` for the others). Ignored, with a warning, with another load mode |
| `--mapmethod` | `Position` | `Position` (column order), `Name` (case-insensitive name match; target columns without a source match are ignored, and stay NULL) |
| `--batchsize` | `1048576` | rows per bulk batch. No effect with an `adbc_*` target (the drivers size their own batches) |
| `--useworktables` | off | switch: load through intermediate work tables. Rare cases, e.g. a clustered columnstore index on the target. **1.0+ only** with `msbulk` / `pgcopy` (older versions load nothing and exit 0, see troubleshooting.md). Refused with an `adbc_*` target |

Batch size guidance from the docs:

| Target | Suggested `--batchsize` |
|---|---|
| `msbulk` | 10,000 – 100,000 |
| `pgcopy` | 10,000 – 50,000 |
| `orabulk` | 100,000 – 500,000 |
| `oradirect` | 1,000 – 10,000 |
| `clickhousebulk` | 10,000 – 1,000,000 |
| `hanabulk` | 5,000 – 10,000 |

## Creating the target table (`--loadmode Create`, 1.1+)

`--loadmode Create` loads a table, a query or files (CSV, Parquet, JSON) into a target table that does
not exist yet.

| Target table | What happens |
|---|---|
| Does not exist | `CREATE TABLE` from the source, then load |
| Exists, with the name exactly as typed | Truncated, then loaded. Its structure is kept: nothing is inferred, nothing is altered |
| Exists only in another case (`--targettable Orders`, stored `orders`) | Error, nothing is created |

- The created table has the column names and the column order of the source. Every column is nullable,
  and the table has no key, index or constraint. The target schema must exist.
- If the load fails after the table was created, the table stays, empty or partly loaded. The next run
  finds it, truncates it and loads it.
- **Same engine on both sides** (Oracle to Oracle, SQL Server to SQL Server…): the source types are
  copied unchanged, whatever `--inferdatatype` says (`allvarchar` excepted).

### `--inferdatatype`

| Mode | Column types | When to use it |
|---|---|---|
| `fast` (default) | The declared types of the source: from the catalog for a table, from the engine for a query. No row is read | Sources with precise metadata: SQL Server, PostgreSQL, MySQL, Db2, Teradata |
| `best` | The strictest type that holds every value, found by one scan of the source. Oracle sources and files read through DuckDB; the same types as `fast` for the other sources | Oracle sources, and CSV, Parquet and JSON files |
| `allvarchar` | Text columns only: Unicode text for the text columns, plain text for the others. The values are converted on the source, in one format whatever the machine: `.` as the decimal point, ISO 8601 dates and timestamps, `true` / `false`, upper case hexadecimal for binary values. A binary column with no declared length (`BLOB`, `varbinary(max)`) stays binary | Raw landing zones, where typing happens later |

Oracle declares little about its numbers and dates, so `fast` plays safe:

| Oracle source | Created by `fast` as |
|---|---|
| `NUMBER` with no precision | A float (`FLOAT(53)`, `DOUBLE PRECISION`…) |
| `NUMBER(p,0)` | `SMALLINT`, `INT` or `BIGINT` by precision, `DECIMAL(p,0)` above 18 digits |
| `NUMBER(p,s)` | `DECIMAL(p,s)` |
| `DATE` | `DATETIME2(0)` / `TIMESTAMP(0)`: an Oracle `DATE` holds a time |

`best` looks at the data:

| Source column | Created by `best` as |
|---|---|
| Oracle `NUMBER` with no precision, `NUMBER(*,0)`, `NUMBER(p,0)` above 18 digits, `FLOAT` | `SMALLINT`, `INT`, `BIGINT` or `DECIMAL(p,s)`, from the real range and number of decimals; a float above 18 decimals |
| Oracle `DATE` | `DATE` when no row has a time part |
| Oracle `CLOB`, a file `VARCHAR` | A bounded text column of the length of the longest value |
| File integers, `DECIMAL`, `DOUBLE` | The smallest integer or decimal type |
| File `TIMESTAMP` | `DATE`, or a timestamp with the fractional digits really used |

- **The types are guarded.** Every narrowed column is read through a check: a row that does not fit the
  inferred type (written after the scan, or missed by a sample) fails the load instead of being rounded
  or truncated. The table created by that run is then dropped, so the next run infers the types again.
- The types fit the data read at creation time: a later `--loadmode Append` with larger values fails on
  the target.

### Text lengths, types with no equivalent

- A text column with no declared length (`CLOB`, `text`, `varchar(max)`, a Parquet string) gets
  **256 characters** in `fast` and `allvarchar`. A value longer than its column fails the load, it is
  never truncated: raise `FASTTRANSFER_INFER_DEFAULT_STRING_MAXSIZE`.
- XML, JSON and arrays are created with the unbounded text type of the target (`NVARCHAR(MAX)`, `TEXT`,
  `CLOB`…).
- A column whose type has no equivalent on the target (range, network address, interval, enum…) is
  created as text, with a warning.

| Environment variable | Default | Effect |
|---|---|---|
| `FASTTRANSFER_INFER_DEFAULT_STRING_MAXSIZE` | `256` | Length of a text column with no declared length, in `fast` and `allvarchar`: from 1 to the largest bounded text length of the target, or `max` for the unbounded type |
| `FASTTRANSFER_INFER_STRING_MARGIN` | `0` | Margin added to a length measured by `best`: a number of characters (`50`) or a factor from `2x` to `10x` |
| `FASTTRANSFER_INFER_SAMPLE_PERCENT` | `100` | `best` reads a sample instead of every row: a block sample of an Oracle table (a query is read in full), `USING SAMPLE` on files |

An invalid value stops the run, naming the variable. Without `--loadmode Create`, a variable that is set
is ignored with a warning.

### Limits

- `allvarchar` needs a source whose engine is known: it is not available with a generic `odbc` or
  `oledb` source.
- With a `--query` and `--parallelmethod Physical`, `allvarchar` and `best` need a query that is a
  single `SELECT` listing its columns, or `SELECT *` on one table. Otherwise give `--sourceschema` and
  `--sourcetable` without a query.
- **MySQL target**: the server must be in strict mode. The text columns of a row hold at most 65,535
  bytes (4 bytes per character in utf8mb4): a table above that limit is not created. Create it yourself,
  with `TEXT` columns where needed, and load it with `--loadmode Truncate`.
- **SQL Server target**: NaN and infinity are refused in `REAL` and `FLOAT` columns.
- **MySQL source**: a `TIME` is a duration; a value outside 00:00:00 to 23:59:59 fails on a target `TIME`
  column.
- Tested on SQL Server, PostgreSQL, Oracle, MySQL, Db2, Teradata (as a source) and DuckDB files. Not
  tested in 1.1.0: ClickHouse, Netezza and SAP HANA (source or target), Teradata as a target, and the
  backup sources. More limits: troubleshooting.md, "Known limitations in 1.1".

## Parallelism

| Flag | Default | Values |
|---|---|---|
| `--parallelmethod` | `None` | `None`, `Physical` (1.0+), `Ntile`, `RangeId`, `DataDriven`, `Random`, `DBPartition` (0.18+), `Timepartition` (0.18+). `Ctid`, `Rowid`, `Physloc`, `NZDataSlice` are deprecated in 1.0: use `Physical` (`--method` is an accepted alias). A backup source (1.1+) only takes `None` or `Physical` |
| `--distributekeycolumn` | – | column or SQL expression in the source dialect; `Timepartition`: `"(col,year)"`, `"(col,year,month)"`, `"(col,year,month,day)"` or `"(col,year,week)"` |
| `--degree` | `-4` | `N>0` threads, `0` = all cores, `-N` = cores / N |
| `--datadrivenquery` | – | query returning the values to split on (`DataDriven`) |

Details: [parallelism.md](parallelism.md).

## Advanced

| Flag | Purpose |
|---|---|
| `--runid "<id>"` | Run identifier for logs; a GUID is generated if omitted |
| `--loglevel` | `verbose`, `debug`, `information` (default), `warning`, `error` |
| `--settingsfile "<path>"` | Logging settings file (default `FastTransfer_Settings.json`, see environment.md) |
| `--license "<path\|url\|content>"` | License (default `FastTransfer.lic` in the working directory, then next to the binary). `--licence` is an alias |
| `--nobanner` | Hide the startup banner |
| `--version` | Print the version (works without a license) |
| `-?` / `--help` | Print the usage (works without a license) |

There is no YAML config file and no `--traceparent` in FastTransfer 1.1 (those are FastBCP features).
