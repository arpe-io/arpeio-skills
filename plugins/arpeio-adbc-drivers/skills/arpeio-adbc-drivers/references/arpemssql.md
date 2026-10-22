# ArpeMSSQL – SQL Server ADBC driver (0.7.6)

Load name `arpemssql`. Native MS-TDS over TCP/TLS (no ODBC, no FreeTDS): TDS 7.4 by default, TDS 8.0
strict encryption on request (since 0.7.5). Options `adbc.arpemssql.*`.

## Targets

| Target | State |
|---|---|
| SQL Server 2022 (primary), 2025 (adds `VECTOR`, `JSON`) | Validated |
| SQL Server 2016, 2017, 2019 | Supported |
| SQL Server 2012 / 2014 | Not validated (TLS needs TLS 1.2 on the server) |
| Azure SQL Database (SQL and Entra ID login, Proxy and Redirect policies) | Validated; Redirect needs outbound TCP 11000–11999 |
| Microsoft Fabric Warehouse / Lakehouse SQL endpoint (`*.datawarehouse.fabric.microsoft.com`, Entra ID only) | Validated since 0.7.5 (earlier versions could not follow the login redirect) |
| Azure SQL MI, Synapse dedicated pool | Not yet validated |
| Express named instances | Via SQL Server Browser (UDP 1434) or an explicit port |
| LocalDB `(localdb)\…` | **Not supported** (named pipes) |

Not supported: MARS (one open result per connection), MultiSubnetFailover, Always Encrypted, client certs.

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
sqlserver://<client id>%40<tenant>:<secret>@srv.database.windows.net/?database=sales&fedauth=ActiveDirectoryServicePrincipal
```

URI parameters (case-insensitive): `database` / `initial catalog`, `user id` / `user` / `uid`,
`password` / `pwd`, `encrypt` (`strict` = TDS 8.0), `tds_version`, `TrustServerCertificate`,
`connection timeout` / `connect timeout` / `dial timeout`, `packet size`, `app name` /
`application name`, `ApplicationIntent`, `hostNameInCertificate`, `certificate` / `ServerCertificate`
(CA file), `fedauth` (same values as `Authentication`, plus `ActiveDirectoryApplication`; since 0.7.5).

**Connection string** (`adbc.arpemssql.connection_string`, ADO.NET): keywords case-insensitive,
`;`-separated, values quoted with `'` or `"` when they contain `;` (double the quote to escape it).
`Server` takes `host`, `host,port`, `tcp:host,port`, `host\INSTANCE[,port]`, `.` / `(local)`.

| Keyword(s) | Field |
|---|---|
| `Server`, `Data Source`, `Address`, `Addr`, `Network Address` | server (+ port) |
| `Database`, `Initial Catalog` | database |
| `User ID`, `UID`, `User` / `Password`, `PWD` | SQL login (Entra ids under `Authentication=ActiveDirectory*`) |
| `Encrypt` (`Strict` = TDS 8.0; also `Mandatory` / `Optional`), `TDS Version` / `TDSVersion`, `TrustServerCertificate`, `HostNameInCertificate`, `ServerCertificate` (CA file), `AllowCleartextLogin` | TLS |
| `Connection Timeout`, `Connect Timeout`, `Timeout` | login timeout |
| `Packet Size`, `Application Name` / `App`, `ApplicationIntent` (`ReadOnly` / `ReadWrite`) | |
| `Integrated Security` / `Trusted_Connection` (`SSPI`, `yes`…), `Authenticator` (`sspi`, `krb5`, `sql`), `Authentication` (`SqlPassword`, `ActiveDirectoryDefault`, `ActiveDirectoryServicePrincipal`, `ActiveDirectoryManagedIdentity` / `ActiveDirectoryMSI`) | auth mode (one of them only) |
| `Service Principal Name`, `RequireKerberos`, `Krb5 Keytab File`, `Krb5 Credential Cache`, `Krb5 Principal`, `Krb5 Password` | integrated auth |

No connection-string or URI form for: `port` (use `host,port`), `query_timeout`, `buffer_size`,
`access_token` and the other discrete Entra options, type rendering, ingest SRID, licence → discrete
options.

```text
Server=sql01.corp.example,1433;Database=sales;User ID=loader;Password='p;w''d';Encrypt=true
```

## Options (database, `db_kwargs`)

Booleans (database, connection, statement): `true` / `false` / `1` / `0` / `yes` / `no`, any case;
any other value → `INVALID_ARGUMENT` (since 0.7.2; before, some read a typo as `false`).

| Key | Default | Notes |
|---|---|---|
| `adbc.arpemssql.server` | – | `host`, `host,port`, `host:port`, `[v6]:port`, `host\INSTANCE` (Browser lookup unless a port is given) |
| `adbc.arpemssql.port` | `1433` | An explicit port skips the Browser lookup |
| `adbc.arpemssql.database` | login default | |
| `adbc.arpemssql.username` / `.password` | – | SQL login |
| `adbc.arpemssql.app_name` | `ArpeMSSQL-…` | `program_name` in `sys.dm_exec_sessions` |
| `adbc.arpemssql.application_intent` | `ReadWrite` | `ReadOnly` → readable secondary through an AG listener |
| `adbc.arpemssql.trusted` | `false` | `true` / `sspi` = integrated auth (Windows SSPI, Linux Kerberos) |
| `adbc.arpemssql.auth_type` | SQL | `sql`, `integrated` / `sspi`, `ActiveDirectoryDefault` |
| `adbc.arpemssql.krb5.spn` | `MSSQLSvc/<host>:<port>` | SPN override (Windows and Linux) |
| `adbc.arpemssql.krb5.keytab`, `.krb5.ccache`, `.krb5.principal`, `.krb5.password` | – | Linux Kerberos credential (since 0.7.5) |
| `adbc.arpemssql.require_kerberos` | `false` | Windows: refuse a login that fell back to NTLM (since 0.7.6) |
| `adbc.arpemssql.encrypt` | `false` | `false` = TLS for the login only (if the server allows); `true` = whole session in TLS, certificate and host name verified; `strict` = `tds_version=8.0` |
| `adbc.arpemssql.tds_version` | `7.4` | `8.0` / `8` = TDS 8.0 strict encryption (since 0.7.5) |
| `adbc.arpemssql.trust_server_cert` | `false` | `true` = no certificate check (labs only); refused with TDS 8.0 |
| `adbc.arpemssql.tls_ca_file` | – | PEM CA (or the server's self-signed cert), trusted in addition to the OS store |
| `adbc.arpemssql.host_name_in_certificate` | the host | Name (or IP) expected in the certificate |
| `adbc.arpemssql.allow_cleartext_login` | `false` | Allow a SQL password when the server has no encryption at all (insecure) |
| `adbc.arpemssql.access_token` | – | Entra bearer token |
| `adbc.arpemssql.tenant_id`, `.client_id`, `.client_secret` | – | Entra service principal |
| `adbc.arpemssql.managed_identity` | – | `true` / `system` = system-assigned, or a user-assigned client id |
| `adbc.arpemssql.login_timeout` | `30` | TCP connect budget, seconds (`0` = 30) |
| `adbc.arpemssql.query_timeout` | `0` (none) | Seconds of server silence per wait (not whole query); cancels with ATTENTION → `ADBC_STATUS_TIMEOUT`, connection kept (since 0.7.2). Settable on an open connection |
| `adbc.arpemssql.buffer_size` | `32000` | Rows per Arrow batch (max 10,000,000) |
| `adbc.arpemssql.packet_size` | `8192` | `512..32767`; ingest on its own session requests 32767 when unset (since 0.7.2) |
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
(1,000,000), ingest options below. Ignored: `mars`, `compression`, `prefetch`, `memory_budget_mb`
(no-op since 0.7.5). Parameters: `?`, or since 0.7.6 `@p1` .. `@pN` in SQL with no `?` (DuckDB
`adbc_scanner` pushdown); vector parameters bind since 0.7.6 (float32 native, float16 as JSON).

## Authentication

| Mode | Options | Notes |
|---|---|---|
| SQL login | `username` + `password` | |
| Windows integrated (SSPI Negotiate) | `trusted=true` (or `Integrated Security=SSPI`), no user / password | Kerberos, NTLM fallback accepted unless `require_kerberos=true` |
| Linux Kerberos (GSSAPI, since 0.7.5) | `trusted=true` + keytab (`krb5.keytab`), or `krb5.principal` + `krb5.password`, else ambient ccache (`kinit`, `KRB5CCNAME`, `krb5.ccache`) | Needs MIT Kerberos (`krb5-libs` / `libgssapi-krb5-2`), loaded at first Kerberos login |
| Entra access token | `access_token` | Always full TLS |
| Entra service principal | `tenant_id` + `client_id` + `client_secret`, or `Authentication=ActiveDirectoryServicePrincipal;User ID=<client id>@<tenant>;Password=<secret>` | Principal must be a DB user: `CREATE USER [<app>] FROM EXTERNAL PROVIDER`. Tenant not discovered from the server |
| Entra managed identity | `managed_identity=true` or `=<client-id>`, or `Authentication=ActiveDirectoryManagedIdentity` (`User ID` = user-assigned client id) | On an Azure host |
| Entra default chain | `auth_type=ActiveDirectoryDefault` (or `Authentication=`) | Env `AZURE_TENANT_ID` + `AZURE_CLIENT_ID` + `AZURE_CLIENT_SECRET` → managed identity → `az account get-access-token` |

- Kerberos: connect by the FQDN the SPN `MSSQLSvc/host:port` is registered for; on Windows an IP, alias
  or loopback login falls back to NTLM. Integrated or Entra auth plus a SQL username/password →
  `INVALID_ARGUMENT` (`krb5.password` is not the SQL password).
- `Authentication=ActiveDirectory{Password,Interactive,DeviceCodeFlow,Integrated,WorkloadIdentity}` →
  `NOT_IMPLEMENTED` (use `access_token`). `Authentication=` / `fedauth` Entra values since 0.7.5.

TLS: certificates are checked against the OS store plus `tls_ca_file`: Windows trusted ROOT store; Linux
the distro CA bundle (RHEL, Debian / Ubuntu, SUSE paths; since 0.7.3), replaced by `SSL_CERT_FILE` /
`SSL_CERT_DIR` if set. Same store for the Entra sign-in. TLS 1.2 minimum. **TDS 8.0 strict** (`tds_version=8.0`, `Encrypt=Strict`, URI `?encrypt=strict`; since 0.7.5): TLS
handshake first (ALPN `tds/8.0`, SNI, TLS 1.2 / 1.3), certificate always verified, `trust_server_cert`
refused (use `tls_ca_file` + `host_name_in_certificate`). Needed when the server has Force Strict
Encryption (SQL Server 2022+, Azure SQL); also accepted by non-strict servers whose cert verifies.

## Types

Read (SQL Server → Arrow): `TINYINT` → uint8; `SMALLINT`/`INT`/`BIGINT` → int16/32/64; `BIT` → bool;
`REAL`/`FLOAT` → float32/64; `DECIMAL(p,s)` → decimal128(p,s); `MONEY` → decimal128(19,4); all char,
`XML` and `UNIQUEIDENTIFIER` (36 chars, lower case) → utf8; binary types and `hierarchyid` → binary;
`DATE` → date32; `SMALLDATETIME` → timestamp[s]; `DATETIME` → timestamp[ms]; `DATETIME2(n)` /
`TIME(n)` → s / ms / us by scale (scale 7 → us, or ns with `timestamp_precision=nanosecond`);
`DATETIMEOFFSET` → timestamp with tz UTC; `VECTOR(n)` → fixed_size_list<float32>[n]; `VECTOR(n, float16)`
→ utf8 (JSON text); geometry / geography → WKB per `geospatial`. Since 0.7.6: `sql_variant` → utf8
(each value in its base type's text form; base type lost), `JSON` → utf8 + `arrow.json` extension
(`ARPEMSSQL_JSON=off` → plain utf8). SQL type name in field metadata `ARPEMSSQL:type`.

Ingest `create` DDL (Arrow → SQL Server): bool → BIT; int8 → SMALLINT; uint8 → TINYINT; int16 →
SMALLINT; int32 → INT; int64 → BIGINT; uint64 → DECIMAL(20,0); float32/64 → REAL/FLOAT; decimal →
DECIMAL(p,s); **utf8 → NVARCHAR(4000)**; **binary → VARBINARY(8000)**; date → DATE; time →
TIME(0/3/6/7); timestamp → DATETIME2(0/3/6/7) (with tz → DATETIMEOFFSET); fixed_size_list<float32> →
VECTOR(n); since 0.7.6 fixed_size_list<float16> → VECTOR(n, float16), `arrow.json` → JSON (2025).
Refused: null, float16 scalars, duration, interval, nested types.

- Longer strings / binaries: create the table yourself with `NVARCHAR(MAX)` / `VARBINARY(MAX)`, then
  `append`. Values are checked against the target length before sending (since 0.7.4).
- `CHAR` / `VARCHAR` targets (since 0.7.2): the driver converts utf8 to the column's code page with SQL
  Server's own best-fit tables (same bytes as a server conversion; up to 8000 bytes); UTF-8 collations
  pass through; DBCS code pages (932, 936, 949, 950) and `string_view` go as NVARCHAR for the server.
  `ARPEMSSQL_BULK_VARCHAR_WIRE=0` restores the NVARCHAR wire.
- WKB / `geoarrow.wkb` columns load into existing `geometry` / `geography` columns (`create` makes
  VARBINARY). **Needs 0.7.6+** (earlier: error 4816); curves and geography `FULLGLOBE` since 0.7.6.
- `XML` / `JSON` targets take any utf8 column. NaN / infinity refused. Nanoseconds floored to 100 ns.

## Ingest (native INSERT BULK)

`cur.adbc_ingest("orders", data, mode=…, db_schema_name="dbo", catalog_name=…, temporary=…)`;
`temporary=True` → session temp table `#orders`. Mode defaults to `create` when only the target table
is set. The target is resolved as the server does (default schema, synonyms, collation; since 0.7.4).

| Statement option | Default | Meaning |
|---|---|---|
| `arpemssql.bulk_tablock` | `true` | TABLOCK: BU lock + minimal logging on a heap, exclusive lock on an indexed table |
| `arpemssql.bulk_parallel` | `false` | Set on every connection of a parallel load: TABLOCK only on a heap without index; explicit `bulk_tablock` wins |
| `arpemssql.bulk_batch_size` | `1000000` | Rows per sub-batch (was 10,000,000 before 0.7.2) |
| `arpemssql.bulk_atomic` | `false` | With autocommit on: each sub-batch commits (failed `replace` keeps the old table via a staging table, failed `create` drops it); `true` = whole ingest (DDL + rows) in one transaction. Was `true` before 0.7.2 |
| `arpemssql.bulk_keep_identity` | `true` | Load source values into IDENTITY (was `false` before 0.7.2) |
| `arpemssql.bulk_check_constraints`, `arpemssql.bulk_fire_triggers` | `false` | |
| `arpemssql.bulk_keep_nulls` | `true` | NULLs instead of column defaults |
| `arpemssql.bulk_order` | – | `col ASC, …` matching the clustered index (data must be in that order) |

Set them with `cur.adbc_statement.set_options(**{"arpemssql.bulk_batch_size": "500000"})` before
`adbc_ingest`. With autocommit off (dbapi default) the ingest runs in the caller's transaction (behind a
savepoint), holding the table lock until `commit()`. Columns match the target by name. A record batch
passed with `Bind` ingests since 0.7.3. `ARPEIO_ADBC_QUIET=1` silences the `[TDS TIMING]` stderr line.

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
| `Failed to receive Pre-Login response` | Server forces strict encryption: `tds_version=8.0` / `Encrypt=Strict` (0.7.5+) |
| `tds_version=8.0 (strict encryption) always validates …` | Drop `trust_server_cert`; use `tls_ca_file` |
| `unable to get local issuer certificate` | `tls_ca_file`; check a stale `SSL_CERT_FILE` / `SSL_CERT_DIR`; before 0.7.3, upgrade; labs only: `trust_server_cert=true` |
| Host name mismatch | `host_name_in_certificate`, or connect by the certificate's name |
| `server refused encryption: encrypt=true requested …` / `server does not support encryption …` | Enable TLS on the server (or `allow_cleartext_login=true`, insecure) |
| `Login failed for user` | Credentials / DB access |
| `… needs MIT Kerberos, which was not found …` / `Trusted/integrated authentication is unavailable …` | Linux: install `krb5-libs` (RHEL) or `libgssapi-krb5-2` (Debian / Ubuntu); the second means a driver older than 0.7.5: upgrade |
| `InitializeSecurityContext: SSPI 0x…` / GSSAPI error | SPN: connect by FQDN, register `MSSQLSvc/host:port`, or set `krb5.spn`; Linux: `klist`, `krb5.ccache` / `krb5.keytab` |
| `… negotiated NTLM, not Kerberos, and adbc.arpemssql.require_kerberos is set …` | Connect by FQDN and register the SPN (`setspn -S`), or unset `require_kerberos` |
| `Federated (access-token) authentication cannot be combined with a username/password` | Remove `username` / `password` |
| Entra token acquired but login fails | `CREATE USER [<name>] FROM EXTERNAL PROVIDER` |
| `ActiveDirectoryDefault: no credential available …` | Set the `AZURE_*` env vars, run on Azure, or `az login` |
| `… routing redirect (ENVCHANGE type N) outside the login response …` | Server / proxy breaks TDS; connect without it |
| `Connection reset by server (TLS)` after idle | Gateway dropped the connection; if `INVALID_STATE`, `rollback()` and retry |
| `The connection is busy with an open result set` | Read / close the previous result first (no MARS) |
| `column N is sql_variant …` | Before 0.7.6: upgrade, or CAST in the query |
| `SQL placeholder count mismatch …` | Use `?` or `@p1..@pN`, not both; before 0.7.6 only `?` (DuckDB `ATTACH` filters failed) |
| Ingest `ALREADY_EXISTS` | Use `append`, `replace` or `create_append` |
| `Column 'x', row N: value of … is longer than NVARCHAR(n)` / `… value NaN …` | Pre-create wider / `MAX` columns and `append`; replace NaN / infinity with NULL |
| `Invalid column type from bcp client` (4816) | Match Arrow nullability to the target (`nullable=False`); WKB into geometry: 0.7.6+ |
| Ingested rows disappear | `conn.commit()` (dbapi autocommit is off) |
| Parallel loads into an indexed table run one at a time | `arpemssql.bulk_parallel=true` on each connection |
| `Query timed out: no reply from the server within query_timeout …` / `TIMEOUT` | Raise `query_timeout` (`0` = none) |
| Bound DML fails | Error names the row: `failed at parameter row N (N rows before it were executed)`; value errors → `INVALID_ARGUMENT` |
| Server error details | Python exception `.details`: `arpemssql.sqlserver.error_number`, `.state`, `.severity`, `.message` |
| Stop a query | `cur.adbc_cancel()` → `CANCELLED`, the connection stays usable |

Docs (0.7.6) vs driver 0.7.6: `arpemssql.bulk_parallel` exists but the public docs do not list it
(they only say the `bcp_` aliases are frozen); the named-instance error is
`Cannot resolve SQL Server instance '…' on host '…'` (docs quote "could not determine port for
instance"); the federated-plus-password error reads as in the table above (docs: "Federated auth cannot
be combined with a password"); `ARPEMSSQL_BULK_VARCHAR_WIRE` and `ARPEIO_ADBC_QUIET` are not in the
public docs.
