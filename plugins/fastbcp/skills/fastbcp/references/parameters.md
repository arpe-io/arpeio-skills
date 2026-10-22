# FastBCP – CLI parameter reference (1.4)

Always write **long flags** in generated commands. Short flags are listed only so you can read
existing scripts.

## Connection

| Long | Short | Value |
|---|---|---|
| `--connectiontype` | `-C` | `mssql` (default), `adbc_mssql`, `pgsql`, `pgcopy`, `adbc_pgsql`, `mysql`, `oraodp`, `adbc_oracle`, `hana`, `teradata`, `adbc_teradata`, `nzsql`, `adbc_netezza`, `clickhouse`, `odbc`, `oledb` |
| `--server` | `-S` | `host`, `host:port`, `host,port`, `host\instance`, `host:1521/service`, TNS alias |
| `--database` | `-I` | database name |
| `--trusted` | `-A` | switch: integrated auth |
| `--user` | `-U` | user name |
| `--password` | `-X` | password |
| `--sourceconnectstring` | `-g` | full connection string (overrides the above) |
| `--dsn` | `-N` | ODBC DSN |
| `--provider` | `-P` | OLE DB provider |
| `--applicationintent` | – | `ReadOnly` (default) / `ReadWrite` (SQL Server, OLE DB) |

Details: [connections.md](connections.md).

## Source

| Long | Short | Value |
|---|---|---|
| `--query` | `-q` | `"SELECT …"` |
| `--fileinput` | `-F` | path to a `.sql` file with one query, no trailing `;` |
| `--sourceschema` | `-s` | schema name |
| `--sourcetable` | `-T` | table name |

Use a query (`--query` / `--fileinput`) or a table (`--sourceschema` + `--sourcetable`). Combine both
when you need tokens, a parallel method, or a filtered export of a known table.

## Output

| Long | Short | Value |
|---|---|---|
| `--directory` | `-D` | local, UNC or cloud URI (`s3://`, `abs://`, `abfss://`, `gs://`, `onelake://`) |
| `--fileoutput` | `-o` | file name; **extension = format** |
| `--timestamped` | `-x` | switch: append `_yyyyMMddHHmmss` |
| `--encoding` | `-e` | default `UTF-8` (text formats) |
| `--cloudprofile` | – | cloud credentials profile name |
| `--delimiter` | `-d` | CSV/TSV, default `,` |
| `--quotes` | `-t` | CSV/TSV switch: quote fields |
| `--dateformat` | `-f` | default `yyyy-MM-dd HH:mm:sszzz` |
| `--decimalseparator` | `-n` | default `.` |
| `--noheader` | `-h` | switch: no header row |
| `--boolformat` | `-b` | default `True/False` |
| `--parquetcompression` | – | `Zstd` (default), `Snappy`, `Gzip`, `Lz4`, `Lzo`, `None` |

Details and tokens: [outputs.md](outputs.md).

## Parallelism

| Long | Short | Value |
|---|---|---|
| `--parallelmethod` | `-m` | `None` (default), `Physical`, `RangeId`, `Ntile`, `DBPartition`, `Timepartition`, `DataDriven`, `Random`; deprecated: `Ctid`, `Physloc`, `Rowid`, `NZDataSlice` |
| `--distributekeycolumn` | `-c` | column or expression; Timepartition: `"(col, year, month)"` |
| `--paralleldegree` | `-p` | default `-2` |
| `--datadrivenquery` | – | query listing the DataDriven values |
| `--merge` | `-M` | `true` / `false` (default `false`) |

Details: [parallelism.md](parallelism.md).

## Advanced (long form only)

| Flag | Purpose |
|---|---|
| `--runid "<id>"` | Run identifier for log correlation; generated if omitted. Pass the scheduler's run id |
| `--loglevel "Information\|Debug"` | Log verbosity (default `Information`) |
| `--settingsfile "<path.json>"` | Logging settings file (Serilog JSON) |
| `--config "<path.yaml>"` | YAML file holding the run parameters |
| `--traceparent "<w3c>"` | W3C trace context for OpenTelemetry correlation |
| `--license "<path\|url\|content>"` | License location or content (default: `FastBCP.lic` next to the binary) |
| `--nobanner` | Hide the banner (scripts / CI) |
| `--version` | Print version |
| `--help` (also `-?`) | Print the usage and exit with code 0 (1.4+; before, `--help` exited with code 1 after parameter errors) |

Some doc examples use the aliases `--method` (for `--parallelmethod`), `--degree` (for
`--paralleldegree`) and `--licence`. Generate the canonical names.

## YAML config file (`--config`)

Lets the whole run live in one version-controlled file:

```yaml
# Usage: ./FastBCP --config orders.yaml
connection:
  type: mssql
  server: localhost
  database: tpch10
  trusted: true

source:
  schema: dbo
  table: orders

output:
  file: "mssql_orders_{startdate}.csv"
  directory: 'D:\temp\{database}\{schema}\{table}\full\'
  delimiter: "|"
  decimal_separator: "."
  date_format: "yyyy-MM-dd HH:mm:ss"
  encoding: UTF-8

performance:
  method: RangeId
  degree: -2
  distribute_key_column: o_orderkey
  merge: false

logging:
  run_id: mssql_to_csv_parallel-2_rangeid
```

This is the only YAML example in the docs. It shows the keys above, and in YAML paths the tokens
appear as `{database}`, `{schema}`, `{table}`. For keys not shown here (user/password, query,
parquet compression…), check the docs before using them.
