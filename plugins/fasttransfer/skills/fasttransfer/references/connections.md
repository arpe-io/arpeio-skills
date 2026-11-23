# FastTransfer – Connections

## Source connection types (`--sourceconnectiontype`)

| Value | Source |
|---|---|
| `mssql` | SQL Server (.NET SqlClient) |
| `msoledbsql` | SQL Server through OLE DB (Windows) |
| `pgsql` | PostgreSQL (.NET Npgsql) |
| `pgcopy` | PostgreSQL using COPY (fastest; enables binary copy when the target is also `pgcopy`) |
| `mysql` | MySQL / MariaDB |
| `oraodp` | Oracle (ODP.NET) |
| `hana` | SAP HANA |
| `teradata` | Teradata |
| `nzsql` | Netezza (native .NET driver) |
| `nzbulk` | Netezza copy (marked work in progress: prefer `nzsql`) |
| `nzoledb` | Netezza through OLE DB (Windows) |
| `clickhouse` | ClickHouse |
| `duckdb` | DuckDB (database file, or `:memory:`) |
| `duckdbstream` | DuckDB in streaming mode (lower memory). **Use it for CSV / Parquet / JSON files** |
| `odbc` | Any ODBC source (DSN or connection string; driver must be installed) |
| `oledb` | Any OLE DB source (Windows; needs `--sourceprovider`) |
| `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_db2`, `adbc_teradata` (1.0+) | Bundled Arpe ADBC drivers, see "ADBC connection types" below. Only with an `adbc_*` target |

Older docs list `nzcopy` ("Netezza copy, WIP"): the CLI does not accept it; the value is `nzbulk`.

Embedded drivers (nothing to install): ClickHouse, DuckDB, SQL Server, MySQL, Netezza, Oracle,
PostgreSQL, SAP HANA, Teradata, and since 1.0 the ADBC drivers (Db2 included). Any other database
(e.g. Sybase) goes through `odbc` / `oledb`.

## Target connection types (`--targetconnectiontype`)

| Value | Target | Notes |
|---|---|---|
| `msbulk` | SQL Server | SQL Server BulkCopy |
| `pgcopy` | PostgreSQL | COPY: binary if the source is `pgcopy`, text (CSV) otherwise |
| `pgsql` | PostgreSQL | Native driver (INSERT … UNNEST) |
| `orabulk` | Oracle | Oracle BulkCopy |
| `oradirect` | Oracle | Oracle Direct |
| `mysqlbulk` | MySQL / MariaDB | MySQL BulkCopy |
| `clickhousebulk` | ClickHouse | ClickHouse BulkCopy |
| `hanabulk` | SAP HANA | |
| `teradata` | Teradata | |
| `nzbulk` | Netezza | Netezza BulkCopy |
| `duckdb` | DuckDB | `--targetserver` is the `.duckdb` file path |
| `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_db2`, `adbc_teradata` (1.0+) | SQL Server, PostgreSQL, Oracle, Netezza, Db2 for LUW, Teradata | Only with an `adbc_*` source |

The target table must already exist; FastTransfer does not create tables.

## Source connection flags

| Flag | Notes |
|---|---|
| `--sourceconnectiontype` | See table above |
| `--sourceserver` | `host`, `host:port`, `host,port`, `host\instance`, Oracle Easy Connect `host:1521/service`, Oracle TNS alias, DuckDB file path or `:memory:` |
| `--sourcedatabase` | Database name. Oracle: service/PDB name (e.g. `ORCLPDB`) |
| `--sourcetrusted` | Integrated auth (Windows / Kerberos). Not combinable with `--sourceuser`/`--sourcepassword` |
| `--sourceuser` | |
| `--sourcepassword` | Masked in logs |
| `--sourceconnectstring` | Full connection string, in double quotes. Overrides server/user/password/trusted/DSN/provider |
| `--sourcedsn` | ODBC DSN (`odbc`), unless `--sourceconnectstring` is used |
| `--sourceprovider` | OLE DB provider for `oledb`, e.g. `MSOLEDBSQL`, `NZOLEDB` |

## Target connection flags

| Flag | Notes |
|---|---|
| `--targetconnectiontype` | See table above |
| `--targetserver` | Same formats as `--sourceserver`; for `duckdb`, the database file path |
| `--targetdatabase` | Database name (Oracle: service/PDB name) |
| `--targettrusted` | Integrated auth. Not combinable with `--targetuser`/`--targetpassword` |
| `--targetuser` | |
| `--targetpassword` | Masked in logs |
| `--targetconnectstring` | Full connection string, overrides the other target connection flags |
| `--targetschema` | **Mandatory** |
| `--targettable` | **Mandatory** |

## ADBC connection types (1.0+)

The `adbc_*` types use the Arpe ADBC drivers **bundled** with FastTransfer (nothing to install). The data
travels as Arrow batches from one driver to the other, so **ADBC is needed on both ends**: an `adbc_*`
source with an `adbc_*` target, any two of the six. Windows x64 and Linux x64 only.

| Type | Database | Default port | Source | Target |
|---|---|---|---|---|
| `adbc_mssql` | SQL Server | 1433 | ✓ | ✓ |
| `adbc_pgsql` | PostgreSQL | 5432 | ✓ | ✓ |
| `adbc_oracle` | Oracle | 1521 | ✓ | ✓ |
| `adbc_netezza` | Netezza | 5480 | ✓ | ✓ |
| `adbc_db2` | Db2 for LUW | 50000 | ✓ | ✓ |
| `adbc_teradata` | Teradata | 1025 | ✓ | ✓ |

### With the usual flags (preferred)

Same flags as the native types: `--sourceserver`, `--sourcedatabase`, `--sourceuser` /
`--sourcepassword` or `--sourcetrusted`, and their target twins.

- `--sourceserver` / `--targetserver`: `host`, `host:port` or `host,port`. SQL Server: a named instance
  `host\INSTANCE`. Oracle: `host[:port]/service` or a TNS alias (`--sourcedatabase` is then the service
  name).
- `--sourcetrusted` / `--targettrusted`: integrated authentication. Windows SSPI or Kerberos for SQL
  Server and PostgreSQL; Kerberos for Oracle, Db2 and Teradata. **Not available for Netezza.**
- The session is named `FastTransfer` on the server (not for Teradata), and a SQL Server source connects
  with `ApplicationIntent=ReadOnly`, as with the native types.

### With a connection string

`--sourceconnectstring` / `--targetconnectstring` take the **driver's own vocabulary**, not an ADO.NET
string:

```
uri=sqlserver://user:password@host:1433/?database=tpch
uri=postgresql://user:password@host:5432/tpch
uri=oracle://user:password@host:1521/orclpdb1
uri=netezza://user:password@host:5480/system
uri=db2://user:password@host:50000/TPCH
uri=teradata://user:password@host/SALES
```

- Driver options go next to `uri=`, separated by `;`, e.g.
  `uri=sqlserver://host/?database=tpch;adbc.arpemssql.trusted=true` for integrated authentication.
- SqlClient keywords such as `Trusted_Connection=Yes` or `Data Source=` are not understood
  (`Unknown database option`).
- Special characters of a password must be percent-encoded in a `uri=` (`@` → `%40`, `:` → `%3A`,
  `/` → `%2F`). Prefer the usual flags when the password comes from a shell variable.
- `Driver=` is optional: without it, FastTransfer uses the driver it bundles for the connection type.
- `arpemssql.bulk_*` options (`arpemssql.bulk_tablock`, `arpemssql.bulk_parallel`…) in an `adbc_mssql`
  target string reach the load and win over FastTransfer's own choice (below).

### Licence

The drivers need an **ARPE ADBC DRIVER** licence block, in addition to the FastTransfer one.

- When the `--license` file holds that block next to the `FASTTRANSFER` one (one file, blocks in any
  order), FastTransfer hands it to the drivers: nothing else to set.
- A licence given to the driver directly takes precedence: `License=<file>` in the connection string, or
  the `ARPEIO_ADBC_LICENCE_FILE` environment variable.
- With no ADBC block anywhere, an `adbc_*` run stops with exit code 99.

### Target table and names

- The target table must exist, as for the native targets.
- `--targetschema` and `--targettable` are used **exactly as typed** (quoted identifiers): they must
  match the stored names, case included: `ORDERS` on Oracle, Db2 and Teradata, `orders` on PostgreSQL,
  unless the table was created with quoted names in another case.
- A name that is not found stops the run before anything is truncated or loaded.

### What differs from the native types

| Topic | With `adbc_*` |
|---|---|
| Mixing with a native type | Refused before anything runs (so `--loadmode Truncate` empties nothing) |
| `--useworktables` | Refused with an ADBC target: the parallel streams load the target table directly |
| `--batchsize` | No effect (a warning says so): the drivers size their own batches |
| `--mapmethod Name` | Works; target columns with no source column stay NULL |
| Parallel methods | All of them. `Physical` for `adbc_pgsql`, `adbc_oracle`, `adbc_mssql`, `adbc_netezza`; `DBPartition` for `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_teradata`, `adbc_db2` (see parallelism.md) |
| Parallel load into `adbc_mssql` | A target with a clustered index (rowstore or columnstore) is loaded without TABLOCK so the streams run together; a heap keeps TABLOCK. A heap with a nonclustered index loads one stream at a time |
| Connection retry | A connection that cannot be made is tried again, up to 4 times over 10 seconds; a refused logon is not retried |
| Timeout | None by default. `FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES`, when set, becomes the drivers' timeout on both ends; a timeout already in the connection string is kept |
| Driver messages | In FastTransfer's log (all sinks), labelled with the connection type, e.g. `WARN adbc_db2 driver: …`; a repeated message goes to Debug. `arpeio.adbc.warnings_to_stderr=true` in the connection string also prints the raw output on stderr |

### Data types

The values travel as Arrow data, which differs from the native types in a few places:

- SQL Server `datetimeoffset`, Oracle `TIMESTAMP WITH TIME ZONE` and Teradata time zone offsets arrive as
  the same instant in UTC (offset `+00:00`).
- SQL Server `time(7)` and `datetime2(7)` are cut to microseconds unless
  `adbc.arpemssql.timestamp_precision=nanosecond` is set (dates between 1677 and 2262 only).
- A PostgreSQL target converts a value whose type differs from its column (an Oracle `NUMBER` into
  `bigint`, text into `integer` or `date`). A value that does not fit (`3000000000` into `integer`) stops
  the load with an error naming the column and row. Decimals are rounded to the column's scale. Text
  into a non-text column makes that load a text-format `COPY`, slower than the binary one.
- PostgreSQL `infinity` / `-infinity` dates and timestamps are copied exactly between two `adbc_pgsql`
  ends. Towards another database the load stops with an error naming the column: map them in the source
  query, e.g. `NULLIF(valid_to, 'infinity')`. A distribution key holding `infinity` cannot be used with
  `Ntile`, `DataDriven` or `Timepartition`.

## Credentials without leaking them

FastTransfer reads credentials from its flags only. Keep secrets in shell variables (or a secret store)
and reference them:

```bash
export FT_SOURCE_PASSWORD='...'
./FastTransfer ... --sourceuser "etl" --sourcepassword "$FT_SOURCE_PASSWORD" ...
```

```powershell
.\FastTransfer.exe ... --sourceuser "etl" --sourcepassword $env:FT_SOURCE_PASSWORD ...
```

Passwords given with `--sourcepassword` / `--targetpassword` or inside a connection string are masked in
console, file and database logs.

## Source query dialects (1.0+)

FastTransfer parses the source `--query` (and `--datadrivenquery`) with the SQL rules of the source
engine. Since 1.0 this includes Db2 and Oracle:

| Source | What parses |
|---|---|
| Db2 (`adbc_db2`, and `odbc` when the driver reports a Db2 server) | Special registers in several words (`CURRENT DATE`, `CURRENT TIMESTAMP(6)`, `CURRENT SCHEMA`…), labeled durations (`CURRENT DATE - 30 DAYS`, `ORDER_DATE + 1 MONTH`), `FETCH FIRST n ROWS ONLY` |
| Oracle (`oraodp`, `adbc_oracle`, and `odbc` when the driver reports an Oracle server) | `#` in a name (`SELECT SID, SERIAL# FROM V$SESSION`), a backslash as an ordinary character (`'C:\temp\'`), the `(+)` outer join, `IGNORE NULLS`, `T@DBLINK`, `CONNECT BY`, `ROWNUM` |

On an Oracle source, a `PARTITION (P1)` clause in the `FROM` of `--query` fails at parse time: filter
with a `WHERE` on the partition key instead, or transfer the whole table with
`--parallelmethod DBPartition`.

## Special sources from the docs

- **ODBC DSN** (e.g. MySQL ODBC, or Db2 with IBM's driver when the target is not `adbc_*`):
  `--sourceconnectiontype "odbc" --sourcedsn "MyDsn"`, plus `--sourceuser`/`--sourcepassword` when the DSN
  does not hold the credentials.
- **DuckDB database file**: `--sourceconnectiontype "duckdb" --sourceserver "D:\data\db.duckdb"`.
- **Tableau Hyper file through the PostgreSQL protocol** (Hyper API server running):
  `--sourceconnectiontype "pgsql" --sourceserver "localhost:8095" --sourcedatabase "D:\path\file.hyper" --sourceuser "tableau_internal_user"`.
- **Files** (CSV, Parquet, JSON): see [files.md](files.md).
