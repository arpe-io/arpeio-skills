# ArpeOracle – Oracle ADBC driver (0.3.12)

Load name `arpeoracle`. Native TNS + TTC over TCP/TLS: **no OCI, no Instant Client**. Options
`adbc.arpeoracle.*`.

## Targets

- Oracle 19c (primary), 12.2, 18c, 21c, 23ai, 26ai validated. **12.1 is the minimum**: 11g and older are
  refused at connect.
- Oracle Cloud Autonomous Database: mutual TLS with the downloaded wallet (`ewallet.pem`), port **1522**,
  `_low` / `_tp`… service, `ssl_mode=verify-full`; password, OCI IAM token or Entra ID login.
- RAC / SCAN listeners: redirects followed. Oracle Spatial `SDO_GEOMETRY`: read / write as WKB.
- Not supported: partitioned reads, Substrait, `sqlnet.ora`, LDAP naming, NTS / Windows integrated auth,
  Kerberos (code exists but is not in the released binaries), `BFILE`, user object types, collections.
- The session opens on first use (first query / metadata call).

## Connecting

All forms are database options. A discrete option always wins; between a URI and a connection string,
the later one wins. Service name **or** SID, not both.

**URI**: `oracle://[user[:password]@]host[:port][/service_name][?key=value&…]` (also
`arpeoracle://`). Path = **service name**; `?sid=ORCL` for a SID. Parameters: `user`, `password`,
`service_name`, `sid`, `ssl_mode`, `ssl_root_cert`, `wallet_location`, `wallet_password`,
`encryption`, `data_integrity`, `connect_timeout`, `proxy_user`, `number_mapping`,
`interval_mapping`, `application_name`. IPv6: `[::1]`.

```text
oracle://scott:<pw>@dbhost:2484/orclpdb1?ssl_mode=verify-full&wallet_location=/etc/oracle/wallet
oracle://scott:<pw>@dbhost:1521?sid=ORCL
```

**Connection string** (`adbc.arpeoracle.connection_string`, ADO.NET keywords): `Server` / `Data
Source` / `Host` / `Address` (host, **EZConnect**, TNS descriptor or tnsnames alias), `Port`, `Service
Name` / `Database`, `SID`, `User ID` / `UID` / `User`, `Password` / `PWD`, `Auth Method`, `Proxy User`,
`New Password`, `SSL Mode`, `SSL Root Cert`, `Wallet Location`, `Wallet Password`, `Encryption`, `Data
Integrity`, `Connection Timeout`, `Socket Timeout`, `Buffer Size`, `Number Mapping`, `Interval Mapping`,
`Application Name`.

```text
Server=dbhost:2484/orclpdb1;User ID=scott;Password=<pw>;SSL Mode=verify-full;Wallet Location=/etc/oracle/wallet
```

`server` (option or `Server` keyword) accepts:
- **EZConnect** `host:port/service` (optional leading `//`), or `host,port`. Bare IPv6 is refused.
- **TNS descriptor** `(DESCRIPTION=(ADDRESS=(PROTOCOL=TCPS)(HOST=…)(PORT=…))(CONNECT_DATA=(SERVICE_NAME=…)))`:
  only the first address and `SERVICE_NAME` / `SID` are used (no failover / load balancing); `TCPS` →
  `ssl_mode=require` unless set; the `SECURITY` section is ignored.
- **tnsnames.ora alias** (0.3.11+): a bare name with no service name / SID anywhere is looked up in
  `tnsnames.ora` from `adbc.arpeoracle.tns_admin`, else `$TNS_ADMIN`, else
  `$ORACLE_HOME/network/admin` (`IFILE` supported). Caution: a real host name without a service name is
  also looked up as an alias.

## Options (database, `db_kwargs`)

| Key | Default | Notes |
|---|---|---|
| `adbc.arpeoracle.server` | – | Host, EZConnect, descriptor or alias |
| `adbc.arpeoracle.port` | `1521` | TCPS listeners often 2484 (on-prem) / 1522 (ADB) |
| `adbc.arpeoracle.service_name` / `.sid` | – | One of them |
| `adbc.arpeoracle.tns_admin` | – | Folder of `tnsnames.ora` |
| `adbc.arpeoracle.username` / `.password` | – | Password login (O5LOGON; the password is never sent in clear) |
| `adbc.arpeoracle.auth_method` | `password` | `password`, `token` (OCI IAM), `oauth2` (Entra / OAuth2) |
| `adbc.arpeoracle.proxy_user` | – | `CONNECT THROUGH` target schema |
| `adbc.arpeoracle.new_password` | – | Change an expired password at login |
| `adbc.arpeoracle.token_location` | – | OCI IAM: folder with `token` + `oci_db_key.pem` (`oci iam db-token get`, default `~/.oci/db-token`) |
| `adbc.arpeoracle.access_token` / `.token_file` | – | OAuth2 / Entra JWT, inline or file (selects `oauth2`) |
| `adbc.arpeoracle.ssl_mode` | `disable` | `require` (no check), `verify-ca`, `verify-full` (chain + host) |
| `adbc.arpeoracle.ssl_root_cert` | – | PEM CA bundle (replaces wallet CAs and system trust) |
| `adbc.arpeoracle.wallet_location` / `.wallet_password` | – | Folder with `ewallet.pem` (client cert + key + CAs) |
| `adbc.arpeoracle.encryption` / `.data_integrity` | `accepted` | Native Network Encryption stance: `rejected`, `accepted`, `requested`, `required` |
| `adbc.arpeoracle.login_timeout` | `30` | Seconds, `0` = no limit |
| `adbc.arpeoracle.socket_timeout` | `0` (none) | Per read |
| `adbc.arpeoracle.batch_size` | `32000` | Rows per Arrow batch (max 10,000,000) |
| `adbc.arpeoracle.sdu` | 8192 | Session data unit, 512..2097152 |
| `adbc.arpeoracle.statement_cache_size` | `20` | Server cursors reused per connection (`0` = off) |
| `adbc.arpeoracle.number_mapping` | `auto` | Unconstrained `NUMBER` / `FLOAT`: `auto` → utf8 (lossless), `double` → float64, `decimal:P,S` → decimal128(P,S) |
| `adbc.arpeoracle.interval_mapping` | `parquet` | `parquet`: YEAR TO MONTH → int64 months, DAY TO SECOND → ISO-8601 text; `native` → interval[month_day_nano] |
| `adbc.arpeoracle.app_name` | `ArpeOracle` | `V$SESSION.PROGRAM` / `MODULE` |
| `arpeio.adbc.license`, `arpeio.adbc.license_file` | – | Licence |

Enumerated values (`ssl_mode`, `encryption`, `auth_method`, `number_mapping`…) are case-insensitive;
`autocommit`, `temporary` and ingest modes are exact.

Connection options: `adbc.connection.autocommit` (`true` / `false`), `adbc.connection.db_schema`
(settable since 0.3.12: `ALTER SESSION SET CURRENT_SCHEMA`, case-sensitive → upper case),
`adbc.connection.catalog` (read-only: the service name).

## Authentication

| Mode | Options |
|---|---|
| Password | `username` + `password` |
| Proxy | proxy account `username` + `password`, `proxy_user` = target (`ALTER USER target GRANT CONNECT THROUGH proxy`) |
| Expired password | add `new_password` |
| OCI IAM token | `auth_method=token` (explicit) + `token_location`; no username. ADB: + wallet + `ssl_mode=verify-full` |
| OAuth2 / Entra ID | `access_token` or `token_file` (no username). ADB with Entra: the App ID URI must be `https://<tenant>.onmicrosoft.com/<appId>` |
| Mutual TLS | `wallet_location` (+ `wallet_password`) with `ssl_mode` ≠ `disable` |

## Encryption

- Defaults: `ssl_mode=disable` + NNE `accepted` → **plaintext** against a server left at `ACCEPTED`. For
  encryption use `ssl_mode=verify-full` (TCPS), or `encryption=required` + `data_integrity=required`
  (NNE: AES-256 with SHA-256 / SHA-1).
- TLS 1.2 minimum. Trust: `ssl_root_cert` → wallet → (0.3.12+) system store (Linux distro bundles,
  Windows ROOT store).
- Wallets: only `ewallet.pem` is read; for `cwallet.sso` run `orapki wallet pkcs12_to_pem -wallet <dir>`.
- NNE stance vs server: `accepted` turns encryption on only if the server `REQUESTED` / `REQUIRED` it;
  `required` fails if the server `REJECTED` it.

## Types

Read (Oracle → Arrow): `NUMBER(p,0)` p ≤ 18 → int64; other constrained `NUMBER(p,s)` → decimal128(p,s);
unconstrained `NUMBER` / `FLOAT` → per `number_mapping` (default **utf8**); `BINARY_FLOAT/DOUBLE` →
float32/64; `VARCHAR2`, `CHAR`, `NVARCHAR2`, `CLOB`, `NCLOB`, `LONG`, `ROWID` → utf8; `BLOB`, `RAW` →
binary; **`DATE` → timestamp[s]** (not date32); `TIMESTAMP(n)` → s / ms / us / ns by n;
`WITH (LOCAL) TIME ZONE` → tz UTC; intervals per `interval_mapping`; `BOOLEAN` (23ai) → bool; `JSON`
(21c+, 23ai) → utf8; `VECTOR` → list<float32 | float64 | int8>; `SYS.XMLTYPE` (0.3.12+) → utf8;
`SDO_GEOMETRY` → WKB (`geoarrow.wkb`; 2D point, line, single-ring polygon).

Ingest `create` DDL: bool → NUMBER(1); int8 → NUMBER(3); int16 → NUMBER(5); int32 → NUMBER(10); int64
→ NUMBER(19); float32/64 → BINARY_FLOAT / BINARY_DOUBLE; decimal128 → NUMBER(p,s); **utf8 →
VARCHAR2(4000)**; **binary → RAW(2000)**; date → DATE; timestamp → TIMESTAMP(0/3/6/9) (tz → WITH TIME
ZONE); `geoarrow.wkb` → SDO_GEOMETRY. Refused: time, intervals, lists, structs, string_view.

- Strings are always bound as VARCHAR2 (max 4000 bytes) and binary as RAW (max 2000 bytes), even when
  appending into CLOB / BLOB columns: longer values cannot be ingested with this driver.
- Names are quoted as given: `my_table` becomes the case-sensitive `"my_table"`. Use upper case.

## Ingest

Optional DDL, then one array-bound `INSERT` per Arrow batch (no direct-path load).
`cur.adbc_ingest("SALES_STAGE", data, mode=…, db_schema_name="SCOTT", temporary=False)`.

| Mode | Behaviour |
|---|---|
| `create` | `CREATE TABLE` (fails if it exists, ORA-00955) |
| `append` | Insert only |
| `replace` | `DROP TABLE … PURGE`, then create |
| `create_append` | Create if missing |

- `temporary=True` → `GLOBAL TEMPORARY TABLE … ON COMMIT PRESERVE ROWS`. `catalog_name` must be the
  connected service name.
- Autocommit on: each batch commits as it goes (earlier batches stay if a later one fails). dbapi
  default (off): `conn.commit()`.
- Spatial: after creating a table with `geoarrow.wkb` columns, the driver registers
  `USER_SDO_GEOM_METADATA` and a spatial index (`arpeoracle.sdo.spatial_index=false` to skip).
  `arpeoracle.sdo.srid` forces the SRID read from `SDO_GEOMETRY` columns.

## Example

```python
import os
import adbc_driver_manager.dbapi as dbapi

conn = dbapi.connect(driver="arpeoracle", db_kwargs={
    "adbc.arpeoracle.server": "dbhost.corp.example",
    "adbc.arpeoracle.port": "2484",
    "adbc.arpeoracle.service_name": "orclpdb1",
    "adbc.arpeoracle.username": "scott",
    "adbc.arpeoracle.password": os.environ["ORA_PASSWORD"],
    "adbc.arpeoracle.ssl_mode": "verify-full",
    "adbc.arpeoracle.number_mapping": "decimal:38,10",
})
with conn.cursor() as cur:
    cur.execute("SELECT * FROM hr.employees WHERE department_id = :1", parameters=[50])
    print(cur.fetch_arrow_table())
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `no server or service name configured …` | Set `service_name` (or `/service` in EZConnect / URI) |
| `TNS alias 'X' not found in <file> …` | Fix the alias, or give `service_name` (a host without a service is looked up as an alias) |
| `… (Oracle 11g or older) … 12.1 or later is required` | Server too old |
| `no username configured` | Password login needs `username` (or use token / oauth2) |
| ORA-01017 | Credentials; proxy grant; Entra App ID URI (`https://…`); identity not mapped (`IDENTIFIED GLOBALLY`); expired token |
| `auth_method=token requires token_location …` | Give `token_location` |
| `Kerberos (KERBEROS5) authentication is unavailable …` | Not in released binaries: use another method |
| `unable to get local issuer certificate` | `ssl_root_cert`, or upgrade to 0.3.12 (system store) |
| `wallet_location/ssl_root_cert were given but ssl_mode is 'disable'` | Set `ssl_mode` |
| `wallet '<dir>' holds cwallet.sso …` | `orapki wallet pkcs12_to_pem -wallet <dir>` |
| Session not encrypted | Set `ssl_mode` or `encryption=requested` / `required` |
| ORA-12660 / ORA-12650 | NNE stance or cipher mismatch: align stances; server must allow AES256 and SHA256/SHA1 |
| Slower connects | `encryption=rejected` + `data_integrity=rejected` if NNE is not wanted |
| NUMBER columns come back as strings | `number_mapping=double` or `decimal:P,S` |
| `column … (Oracle type N) is not supported by this driver yet` | Convert in SQL (e.g. `TO_CHAR`) or exclude the column |
| Ingested table "does not exist" | Names are case-sensitive: use upper case |
| `string of N bytes exceeds Oracle's 4000-byte VARCHAR2 limit …` | Values over 4000 bytes can't be ingested |
| `a result stream from this connection is still open …` | Read / close the previous result |
| Commit / Rollback `INVALID_STATE` | Turn autocommit off first |
| `cannot allocate memory in static TLS block` (two Arpe drivers in one process, Linux) | Upgrade to 0.3.12 |

Docs (0.3.11) vs driver 0.3.12: `db_schema` is settable; `GetStatistics` and `GetParameterSchema` are
implemented; TLS uses the OS certificate store; `XMLTYPE` and 21c `JSON` are read as text; binds and
ingest run per Arrow batch, not per row; `BindStream` works for DML; the licence is re-checked at every
connection.
