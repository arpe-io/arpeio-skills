# FastTransfer – CLI parameter reference (0.17)

Generate **long flags only**, spelled exactly as below.

## Source

| Flag | Value |
|---|---|
| `--sourceconnectiontype` | `mssql`, `msoledbsql`, `pgsql`, `pgcopy`, `mysql`, `oraodp`, `hana`, `teradata`, `nzsql`, `nzoledb`, `clickhouse`, `duckdb`, `duckdbstream`, `odbc`, `oledb` |
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
| `--sourceschema` | schema (required by `Ctid`, `Rowid`, `NZDataSlice`) |
| `--sourcetable` | table (required by `Ctid`, `Rowid`, `NZDataSlice`) |
| `--query` | SQL query; used instead of the source table when given |
| `--fileinput` | path to a file containing the SQL query (must exist) |

A table or a query is mandatory. `--query` and `--fileinput` are mutually exclusive.

## Target

| Flag | Value |
|---|---|
| `--targetconnectiontype` | `msbulk`, `pgcopy`, `pgsql`, `orabulk`, `oradirect`, `mysqlbulk`, `clickhousebulk`, `hanabulk`, `teradata`, `nzbulk`, `duckdb` |
| `--targetserver` | server, or DuckDB file path |
| `--targetdatabase` | database |
| `--targettrusted` | switch: integrated auth |
| `--targetuser` / `--targetpassword` | credentials |
| `--targetconnectstring` | full connection string |
| `--targetschema` | **mandatory** |
| `--targettable` | **mandatory**; the table must already exist |

## Load behaviour

| Flag | Default | Values |
|---|---|---|
| `--loadmode` | `Append` | `Append`, `Truncate` (empties the target table first) |
| `--mapmethod` | `Position` | `Position` (column order), `Name` (case-insensitive name match; target columns without a source match are ignored) |
| `--batchsize` | `1048576` | rows per bulk batch |
| `--useworktables` | off | switch: load through intermediate work tables. Rare cases, e.g. a clustered columnstore index on the target |

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
| `--parallelmethod` | `None` | `None`, `Ctid`, `Physloc`, `Rowid`, `NZDataSlice`, `Ntile`, `RangeId`, `DataDriven`, `Random` (`--method` is an accepted alias) |
| `--distributekeycolumn` | – | column or SQL expression in the source dialect |
| `--degree` | `-4` | `N>0` threads, `0` = all cores, `-N` = cores / N |
| `--datadrivenquery` | – | query returning the values to split on (`DataDriven`) |

Details: [parallelism.md](parallelism.md).

## Advanced

| Flag | Purpose |
|---|---|
| `--runid "<id>"` | Run identifier for logs; a GUID is generated if omitted |
| `--loglevel` | `verbose`, `debug`, `information` (default), `warning`, `error` |
| `--settingsfile "<path>"` | Logging settings file (default `FastTransfer_Settings.json`, see environment.md) |
| `--license "<path\|url\|content>"` | License (default `FastTransfer.lic` next to the binary). `--licence` is an alias |
| `--nobanner` | Hide the startup banner |
| `--version` | Print the version |
| `-?` / `--help` | Print the usage |

There is no YAML config file and no `--traceparent` in FastTransfer 0.17 (those are FastBCP features).
