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
| `nzoledb` | Netezza through OLE DB (Windows) |
| `clickhouse` | ClickHouse |
| `duckdb` | DuckDB (database file, or `:memory:`) |
| `duckdbstream` | DuckDB in streaming mode (lower memory). **Use it for CSV / Parquet / JSON files** |
| `odbc` | Any ODBC source (DSN or connection string; driver must be installed) |
| `oledb` | Any OLE DB source (Windows; needs `--sourceprovider`) |

The docs also list `nzcopy` ("Netezza copy, WIP"), but the 0.17 CLI does not accept it. Don't use it.

Embedded drivers (nothing to install): ClickHouse, DuckDB, SQL Server, MySQL, Netezza, Oracle,
PostgreSQL, SAP HANA, Teradata. Any other database (e.g. DB2, Sybase) goes through `odbc` / `oledb`.

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

## Special sources from the docs

- **ODBC DSN** (e.g. DB2, MySQL ODBC): `--sourceconnectiontype "odbc" --sourcedsn "MyDsn"`, plus
  `--sourceuser`/`--sourcepassword` when the DSN does not hold the credentials.
- **DuckDB database file**: `--sourceconnectiontype "duckdb" --sourceserver "D:\data\db.duckdb"`.
- **Tableau Hyper file through the PostgreSQL protocol** (Hyper API server running):
  `--sourceconnectiontype "pgsql" --sourceserver "localhost:8095" --sourcedatabase "D:\path\file.hyper" --sourceuser "tableau_internal_user"`.
- **Files** (CSV, Parquet, JSON): see [files.md](files.md).
