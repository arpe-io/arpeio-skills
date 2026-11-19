# FastTransfer – CLI parameter reference (1.0)

Generate **long flags only**, spelled exactly as below.

## Source

| Flag | Value |
|---|---|
| `--sourceconnectiontype` | `mssql`, `msoledbsql`, `pgsql`, `pgcopy`, `mysql`, `oraodp`, `hana`, `teradata`, `nzsql`, `nzbulk`, `nzoledb`, `clickhouse`, `duckdb`, `duckdbstream`, `odbc`, `oledb`; 1.0+: `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_db2`, `adbc_teradata` (only with an `adbc_*` target) |
| `--sourceserver` | server (see [connections.md](connections.md)) |
| `--sourcedatabase` | database |
| `--sourcetrusted` | switch: integrated auth |
| `--sourceuser` / `--sourcepassword` | credentials |
| `--sourceconnectstring` | full connection string |
| `--sourcedsn` | ODBC DSN |
| `--sourceprovider` | OLE DB provider |

## What to transfer

| Flag | Value |
|---|---|
| `--sourceschema` | schema (required by `DBPartition`, and by `Physical` on PostgreSQL, Oracle and SQL Server) |
| `--sourcetable` | table (required by `DBPartition`, and by `Physical` on PostgreSQL, Oracle and SQL Server) |
| `--query` | SQL query; used instead of the source table when given. With `Physical` it filters the source table |
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
| `--targettable` | **mandatory**; the table must already exist; exact stored case with an `adbc_*` target |

## Load behaviour

| Flag | Default | Values |
|---|---|---|
| `--loadmode` | `Append` | `Append`, `Truncate` (empties the target table first) |
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

## Parallelism

| Flag | Default | Values |
|---|---|---|
| `--parallelmethod` | `None` | `None`, `Physical` (1.0+), `Ntile`, `RangeId`, `DataDriven`, `Random`, `DBPartition` (0.18+), `Timepartition` (0.18+). `Ctid`, `Rowid`, `Physloc`, `NZDataSlice` are deprecated in 1.0: use `Physical` (`--method` is an accepted alias) |
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

There is no YAML config file and no `--traceparent` in FastTransfer 1.0 (those are FastBCP features).
