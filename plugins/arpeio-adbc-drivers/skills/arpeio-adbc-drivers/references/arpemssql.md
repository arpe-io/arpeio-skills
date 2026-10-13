# ArpeMSSQL – SQL Server ADBC driver (0.7.1)

Load name `arpemssql`. Native MS-TDS 7.4 over TCP/TLS (no ODBC, no FreeTDS). Options `adbc.arpemssql.*`.

## Targets

| Target | State |
|---|---|
| SQL Server 2022 (primary), 2025 (adds `VECTOR`) | Validated |
| SQL Server 2016, 2017, 2019 | Supported |
| SQL Server 2012 / 2014 | Not validated (TLS needs TLS 1.2 on the server) |
| Azure SQL Database (SQL and Entra ID login, Proxy and Redirect policies) | Validated; Redirect needs outbound TCP 11000–11999 |
| Microsoft Fabric Warehouse (`*.datawarehouse.fabric.microsoft.com`, Entra ID only), Azure SQL MI, Synapse dedicated pool | Not yet validated |
| Express named instances | Via SQL Server Browser (UDP 1434) or an explicit port |
| LocalDB `(localdb)\…` | **Not supported** (named pipes) |

Not supported: TDS 8 strict encryption, MARS (one open result per connection), MultiSubnetFailover,
Always Encrypted, client certificates, `sql_variant` columns (CAST them in the query).

## Connecting

A discrete option always wins over a URI / connection-string field. URI + connection string together:
merged, the later one wins for a shared field. A keyword or URI parameter given twice, or an unknown
one, is an error.

**URI**: `sqlserver://[user[:password]@]host[:port][/INSTANCE][?key=value&…]` (also `mssql://`,
`arpemssql://`). The **path is the instance name**; the database goes in `?database=`. Percent-encode
special characters in the password (`@` → `%40`).

```text
sqlserver://loader:p%40ss@sql01.corp.example:1433/?database=sales&encrypt=true
sqlserver://loader:secret@dbhost/SQLEXPRESS?database=sales&connection+timeout=10
```

URI parameters (case-insensitive): `database` / `initial catalog`, `user id` / `user` / `uid`,
`password` / `pwd`, `encrypt`, `TrustServerCertificate`, `connection timeout` / `connect timeout` /
`dial timeout`, `packet size`, `app name` / `application name`, `ApplicationIntent`,
`hostNameInCertificate`, `certificate` / `ServerCertificate` (CA file). `fedauth` → not implemented.

**Connection string** (`adbc.arpemssql.connection_string`, ADO.NET): keywords case-insensitive,
`;`-separated, values quoted with `'` or `"` when they contain `;` (double the quote to escape it).
`Server` takes `host`, `host,port`, `tcp:host,port`, `host\INSTANCE[,port]`, `.` / `(local)`.

| Keyword(s) | Field |
|---|---|
| `Server`, `Data Source`, `Address`, `Addr`, `Network Address` | server (+ port) |
| `Database`, `Initial Catalog` | database |
| `User ID`, `UID`, `User` / `Password`, `PWD` | SQL login |
| `Encrypt`, `TrustServerCertificate`, `HostNameInCertificate`, `ServerCertificate` (CA file), `AllowCleartextLogin` | TLS |
| `Connection Timeout`, `Connect Timeout`, `Timeout` | login timeout |
| `Packet Size`, `Application Name` / `App`, `ApplicationIntent` (`ReadOnly` / `ReadWrite`) | |
| `Integrated Security` / `Trusted_Connection` (`SSPI`, `yes`…), `Authenticator` (`sspi`, `sql`), `Authentication` (`SqlPassword` only), `Service Principal Name` | auth mode (one of them only) |

No connection-string or URI form for: `port` (use `host,port`), `query_timeout`, Entra ID options, type
rendering, ingest SRID, licence → discrete options.

```text
Server=sql01.corp.example,1433;Database=sales;User ID=loader;Password='p;w''d';Encrypt=true
```

## Options (database, `db_kwargs`)

Booleans as discrete options: lowercase `true` / `false` / `1` / `0` / `yes` / `no`.

| Key | Default | Notes |
|---|---|---|
| `adbc.arpemssql.server` | – | `host`, `host,port`, `host:port`, `[v6]:port`, `host\INSTANCE` (Browser lookup unless a port is given) |
| `adbc.arpemssql.port` | `1433` | An explicit port skips the Browser lookup |
| `adbc.arpemssql.database` | login default | |
| `adbc.arpemssql.username` / `.password` | – | SQL login |
| `adbc.arpemssql.app_name` | `ArpeMSSQL-…` | `program_name` in `sys.dm_exec_sessions` |
| `adbc.arpemssql.application_intent` | `ReadWrite` | `ReadOnly` → readable secondary through an AG listener |
| `adbc.arpemssql.trusted` | `false` | `true` / `sspi` = Windows integrated auth |
| `adbc.arpemssql.auth_type` | SQL | `sql`, `integrated` / `sspi`, `ActiveDirectoryDefault` |
| `adbc.arpemssql.krb5.spn` | `MSSQLSvc/<host>:<port>` | SPN override (Windows SSPI) |
| `adbc.arpemssql.encrypt` | `false` | `false` = TLS for the login only (if the server allows); `true` = whole session in TLS, certificate and host name verified |
| `adbc.arpemssql.trust_server_cert` | `false` | `true` = no certificate check (labs only) |
| `adbc.arpemssql.tls_ca_file` | – | PEM CA (or the server's self-signed cert), trusted in addition to the system store |
| `adbc.arpemssql.host_name_in_certificate` | the host | Name expected in the certificate |
| `adbc.arpemssql.allow_cleartext_login` | `false` | Allow a SQL password when the server has no encryption at all (insecure) |
| `adbc.arpemssql.access_token` | – | Entra bearer token |
| `adbc.arpemssql.tenant_id`, `.client_id`, `.client_secret` | – | Entra service principal |
| `adbc.arpemssql.managed_identity` | – | `true` / `system` = system-assigned, or a user-assigned client id |
| `adbc.arpemssql.login_timeout` | `30` | TCP connect budget, seconds (`0` = 30) |
| `adbc.arpemssql.query_timeout` | `0` (none) | Seconds; timeout → `ADBC_STATUS_TIMEOUT` |
| `adbc.arpemssql.buffer_size` | `32000` | Rows per Arrow batch (max 10,000,000) |
| `adbc.arpemssql.packet_size` | `8192` | `512..32767` |
| `adbc.arpemssql.uuid_casing` | `lower` | `upper` |
| `adbc.arpemssql.geospatial` | `geoarrow` | `geoarrow` (WKB + `geoarrow.wkb` extension), `wkb`, `varbinary` (raw bytes) |
| `adbc.arpemssql.timestamp_precision` | `microsecond` | `nanosecond` for DATETIME2(7) / TIME(7) |
| `adbc.arpemssql.ingest.srid` | unset | SRID for WKB ingest when the column has no CRS (else geometry 0, geography 4326) |
| `arpeio.adbc.license`, `arpeio.adbc.license_file` | – | Licence (install.md) |

Connection options: `adbc.connection.autocommit` (set it on the connection, e.g.
`dbapi.connect(..., autocommit=True)`; the database-level value is ignored), `adbc.connection.catalog`
(switch database), `adbc.connection.db_schema`, `adbc.connection.transaction.isolation_level`
(`…isolation.read_committed`, `snapshot`, `serializable`…). `adbc.connection.readonly=true` → not
implemented (use `application_intent`).

Statement options: `adbc.arpemssql.batch_size` (rows per batch), `adbc.arpemssql.max_batch_size`
(1,000,000), ingest options below. Ignored: `mars`, `compression`, `memory_budget_mb`, `prefetch`.

## Authentication

| Mode | Options | Notes |
|---|---|---|
| SQL login | `username` + `password` | |
| Windows integrated (SSPI / Kerberos) | `trusted=true` (or `Integrated Security=SSPI`), no user / password | **Windows clients only**; connect by the FQDN the SPN is registered for |
| Entra access token | `access_token` | Always full TLS |
| Entra service principal | `tenant_id` + `client_id` + `client_secret` | Principal must be a DB user: `CREATE USER [<app>] FROM EXTERNAL PROVIDER` |
| Entra managed identity | `managed_identity=true` or `=<client-id>` | On an Azure host |
| Entra default chain | `auth_type=ActiveDirectoryDefault` | Env `AZURE_TENANT_ID` + `AZURE_CLIENT_ID` + `AZURE_CLIENT_SECRET` → managed identity → `az account get-access-token` |

- Integrated or Entra auth plus a username/password → error (remove the password).
- Linux binaries have no Kerberos: integrated auth fails ("…unavailable: this build has neither Windows
  SSPI nor POSIX Kerberos…"); use a SQL login or Entra ID.
- Connection-string `Authentication=ActiveDirectory*` → not implemented: use the Entra options above.

TLS: certificates are checked against the system store (Linux OpenSSL paths, Windows ROOT store) plus
`tls_ca_file`. RHEL: `SSL_CERT_FILE=/etc/pki/tls/certs/ca-bundle.crt` or `tls_ca_file`. TLS 1.2 minimum.

## Types

Read (SQL Server → Arrow): `TINYINT` → uint8; `SMALLINT`/`INT`/`BIGINT` → int16/32/64; `BIT` → bool;
`REAL`/`FLOAT` → float32/64; `DECIMAL(p,s)` → decimal128(p,s); `MONEY` → decimal128(19,4); all char,
`XML` and `UNIQUEIDENTIFIER` (36 chars, lower case) → utf8; binary types and `hierarchyid` → binary;
`DATE` → date32; `SMALLDATETIME` → timestamp[s]; `DATETIME` → timestamp[ms]; `DATETIME2(n)` /
`TIME(n)` → s / ms / us by scale (scale 7 → us, or ns with `timestamp_precision=nanosecond`);
`DATETIMEOFFSET` → timestamp with tz UTC; `VECTOR(n)` → fixed_size_list<float32>[n]; geometry /
geography → WKB per `geospatial`. `sql_variant` → error.

Ingest `create` DDL (Arrow → SQL Server): bool → BIT; int8 → SMALLINT; uint8 → TINYINT; int16 →
SMALLINT; int32 → INT; int64 → BIGINT; uint64 → DECIMAL(20,0); float32/64 → REAL/FLOAT; decimal →
DECIMAL(p,s); **utf8 → NVARCHAR(4000)**; **binary → VARBINARY(8000)**; date → DATE; time →
TIME(0/3/6/7); timestamp → DATETIME2(0/3/6/7) (with tz → DATETIMEOFFSET); fixed_size_list<float32> →
VECTOR(n). Refused: null, float16, duration, interval, nested types.

- Longer strings / binaries: create the table yourself with `NVARCHAR(MAX)` / `VARBINARY(MAX)`, then
  `append`.
- WKB / `geoarrow.wkb` columns load into existing `geometry` / `geography` columns (`create` makes
  VARBINARY).
- Nanoseconds are floored to 100 ns.

## Ingest (native INSERT BULK)

`cur.adbc_ingest("orders", data, mode=…, db_schema_name="dbo", catalog_name=…, temporary=…)`;
`temporary=True` → session temp table `#orders`.

| Statement option | Default | Meaning |
|---|---|---|
| `arpemssql.bulk_tablock` | `true` | TABLOCK (minimal logging) |
| `arpemssql.bulk_batch_size` | `10000000` | Rows per sub-batch |
| `arpemssql.bulk_atomic` | `true` | With autocommit on: whole ingest (DDL + rows) in one transaction; `false` = each sub-batch commits |
| `arpemssql.bulk_keep_identity` | `false` | Load source values into IDENTITY |
| `arpemssql.bulk_check_constraints`, `arpemssql.bulk_fire_triggers` | `false` | |
| `arpemssql.bulk_keep_nulls` | `true` | NULLs instead of column defaults |
| `arpemssql.bulk_order` | – | `col ASC, …` matching the clustered index (data must be in that order) |

Set them with `cur.adbc_statement.set_options(**{"arpemssql.bulk_batch_size": "500000"})` before
`adbc_ingest`. With autocommit off (dbapi default) the ingest runs in the caller's transaction:
`commit()`. Encrypted bulk ingest needs **0.7.1+** (0.7.0 failed with "Connection reset by server (TLS)"
/ "SSL write error 5"). `ARPEIO_ADBC_QUIET=1` silences the `[TDS TIMING]` stderr line.

## Example

```python
import os
import adbc_driver_manager.dbapi as dbapi

conn = dbapi.connect(driver="arpemssql", db_kwargs={
    "adbc.arpemssql.server": "myserver.database.windows.net",
    "adbc.arpemssql.database": "sales",
    "adbc.arpemssql.encrypt": "true",
    "adbc.arpemssql.tenant_id": os.environ["AZURE_TENANT_ID"],
    "adbc.arpemssql.client_id": os.environ["AZURE_CLIENT_ID"],
    "adbc.arpemssql.client_secret": os.environ["AZURE_CLIENT_SECRET"],
}, autocommit=True)
with conn.cursor() as cur:
    cur.execute("SELECT TOP 10 * FROM dbo.orders WHERE amount > ?", parameters=[100])
    print(cur.fetch_arrow_table())
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| Connect fails after ~30 s | Firewall / port; lower `login_timeout` while testing |
| `Cannot resolve SQL Server instance …` | Start SQL Server Browser and open UDP 1434, or give the port (`host\INST,1433` / `adbc.arpemssql.port`) |
| `unable to get local issuer certificate` | `tls_ca_file`, or `SSL_CERT_FILE` (RHEL); labs only: `trust_server_cert=true` |
| Host name mismatch | `host_name_in_certificate`, or connect by the certificate's name |
| `server refused encryption: encrypt=true requested …` / `server does not support encryption …` | Enable TLS on the server (or `allow_cleartext_login=true`, insecure) |
| `Login failed for user` | Credentials / DB access |
| `Trusted/integrated authentication is unavailable …` | Linux: use SQL login or Entra ID |
| `InitializeSecurityContext: SSPI 0x…` | SPN: connect by FQDN, register `MSSQLSvc/host:port`, or set `krb5.spn` |
| Entra token acquired but login fails | `CREATE USER [<name>] FROM EXTERNAL PROVIDER` |
| `ActiveDirectoryDefault: no credential available …` | Set the `AZURE_*` env vars, run on Azure, or `az login` |
| `The connection is busy with an open result set` | Read / close the previous result first (no MARS) |
| `column N is sql_variant …` | CAST in the query |
| Ingest `ALREADY_EXISTS` | Use `append`, `replace` or `create_append` |
| Long strings fail on ingest | Pre-create `NVARCHAR(MAX)` columns and `append` |
| Ingested rows disappear | `conn.commit()` (dbapi autocommit is off) |
| `Recv timed out after N seconds` / `TIMEOUT` | Raise `query_timeout` (`0` = none) |
| Server error details | Python exception `.details`: `arpemssql.sqlserver.error_number`, `.state`, `.severity`, `.message` |
| Stop a query | `cur.adbc_cancel()` → `CANCELLED`, the connection stays usable |

Docs (0.7.0) vs driver 0.7.1: `tls_ca_file` and `host_name_in_certificate` exist although the docs say
there is no CA-file or `HostNameInCertificate` option; cancel works (docs: not implemented); Windows TLS
reads the Windows certificate store; `adbc.ingest.mode` has no default in the driver (Python
`adbc_ingest` sends `create`); `memory_budget_mb` has no effect.
