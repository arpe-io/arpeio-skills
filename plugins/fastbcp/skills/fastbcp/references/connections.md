# FastBCP – Connections

## Connection types (`--connectiontype`, short `-C`, default `mssql`)

| Value | Source | Notes |
|---|---|---|
| `mssql` | Microsoft SQL Server | Default |
| `adbc_mssql` | SQL Server via arpe.io ADBC FastDriver | Parquet only, x64 only; about 2x faster for Parquet |
| `pgsql` | PostgreSQL, standard connection | |
| `pgcopy` | PostgreSQL via COPY protocol | Fastest non-ADBC PostgreSQL path |
| `adbc_pgsql` | PostgreSQL via ADBC FastDriver | Parquet only, x64 only |
| `mysql` | MySQL / MariaDB | |
| `oraodp` | Oracle (ODP.NET) | |
| `adbc_oracle` | Oracle via ADBC FastDriver | Parquet only, x64 only |
| `hana` | SAP HANA | |
| `hanaexport` | SAP HANA using native `EXPORT` | Shown in the docs' examples only; recommended there for large datasets |
| `teradata` | Teradata | |
| `nzsql` | IBM Netezza | |
| `adbc_netezza` | Netezza / IBM NPS via ADBC (embedded driver, no NZdotNET, works with IBM Cloud NPS) | Parquet only, x64 only |
| `clickhouse` | ClickHouse | |
| `odbc` | Any ODBC source (e.g. DB2) | Needs `--dsn` or `--sourceconnectstring`, and the ODBC driver installed |
| `oledb` | Any OLE DB source | Windows only. Needs `--provider` (e.g. `MSOLEDBSQL`, `NZOLEDB`, `OraOLEDB.Oracle`) |

Embedded drivers (nothing to install): ClickHouse, DuckDB, SQL Server, MySQL, Netezza, Oracle,
PostgreSQL, SAP HANA, Teradata. Any other database goes through `odbc` / `oledb` with its own driver.

### Platform support

| | Windows x64 | Linux x64 | Linux ARM64 |
|---|---|---|---|
| `adbc_*` | yes | yes | **no** |
| Teradata, Netezza, SAP HANA | yes | yes | **no** |
| OLE DB | yes | **no** | **no** |
| All others | yes | yes | yes |

## Connection parameters

| Long | Short | Notes |
|---|---|---|
| `--server` | `-S` | Required unless `--sourceconnectstring` is used. Formats: `host`, `host:port`, `host,port`, `host\instance`, Oracle Easy Connect `host:1521/service`, or an Oracle TNS alias |
| `--database` | `-I` | Optional (engine dependent). Not `-d`, which is the CSV delimiter |
| `--trusted` | `-A` | Windows / Kerberos integrated auth, if the RDBMS supports it |
| `--user` | `-U` | |
| `--password` | `-X` | Masked in FastBCP logs |
| `--sourceconnectstring` | `-g` | Full connection string, in double quotes. Overrides every other connection parameter. Works with any connection type |
| `--dsn` | `-N` | ODBC DSN (for `odbc`, unless `--sourceconnectstring` is used) |
| `--provider` | `-P` | OLE DB provider (required for `oledb`) |
| `--applicationintent` | – | `ReadOnly` (default) or `ReadWrite`. `mssql`, `adbc_mssql`, `oledb` only. `ReadOnly` routes to an Always On readable secondary when there is one |

Many doc examples (PostgreSQL, MySQL, ClickHouse) also pass `--port "<port>"`, even though it is not
listed on the Connection Parameters page. PostgreSQL also accepts `--server "host:port"`. For MySQL and
ClickHouse, follow the docs' examples and use `--server "host" --port "3306"`.

## Credentials without leaking them

FastBCP has no built-in env var for the DB user or password. The docs suggest storing them in shell
variables (for example `FASTBCP_USER` / `FASTBCP_PASSWORD`) and referencing them:

```bash
export FASTBCP_PASSWORD='...'   # or read from a secret store
./FastBCP ... --user "$FASTBCP_USER" --password "$FASTBCP_PASSWORD" ...
```

```powershell
.\FastBCP.exe ... --user $env:FASTBCP_USER --password $env:FASTBCP_PASSWORD ...
```

`pgsql` and `adbc_pgsql` also honour the libpq variables `PGUSER`, `PGPASSWORD`, `PGPASSFILE` /
`~/.pgpass` and `PGSSLROOTCERT` when `--user` / `--password` / `--sourceconnectstring` don't set them.

## Connection string examples

SQL Server:

```
--sourceconnectstring "Server=myserver;Database=mydb;User ID=FastUser;Password=FastPassword;Encrypt=True;TrustServerCertificate=True"
```

DB2 through ODBC:

```
--connectiontype "odbc" --sourceconnectstring "Driver={IBM DB2 ODBC DRIVER};Database=SAMPLE;Hostname=db2server.company.corp;Port=25010;Protocol=TCPIP;Uid=db2user;Pwd=password;CurrentSchema=DB2ADMIN"
```

Netezza with the OLE DB provider (Windows):

```
--connectiontype "oledb" --provider "NZOLEDB"
```

### ADBC connection strings

With `adbc_mssql`, `adbc_pgsql`, `adbc_oracle` and `adbc_netezza`, `--sourceconnectstring` accepts:

- native ADO.NET-style `Key=Value;` pairs, translated to driver keys. Keys with no equivalent
  (e.g. `Pooling=false`) are dropped with a warning;
- explicit `adbc.*` / `arpeio.*` keys, which take precedence over translated keys;
- a whole-connection `uri` / `adbc.<driver>.connection_string`. With this form, native keys are
  dropped, so put everything inside the URI.

Since 1.3, metadata/planning and data both go through the ADBC driver and the same connection string.

Per-driver notes:

- **adbc_netezza**: keys such as `Server`, `Host`, `Port`, `Database`, `User ID`, `Password`,
  `SSL Mode`, `sslmode`, `tls_min_version`, `char_encoding`, `login_timeout`. **NPS 7.x needs
  `tls_min_version=1.0`.**
  ```
  --connectiontype adbc_netezza --sourceconnectstring "Server=nps-host;Port=5480;Database=SALES;User ID=admin;Password=FastPassword;SSL Mode=require"
  ```
- **adbc_pgsql**: a `NaN` in a `NUMERIC` column makes the read fail. Add
  `adbc.arpepgsql.numeric_special=null` to the connection string to read it as `NULL`.
- **adbc_oracle**: resolves TNS aliases through `tnsnames.ora` (from `TNS_ADMIN`, then
  `$ORACLE_HOME/network/admin`). Encryption and integrity are negotiated (`accepted`). Override with
  the `FASTBCP_ADBC_ENCRYPTION` / `FASTBCP_ADBC_DATA_INTEGRITY` env vars.
- **adbc_mssql**: an `ApplicationIntent` key in the connection string is passed to the driver.
  Supports up to 1024 columns (since 1.2).
