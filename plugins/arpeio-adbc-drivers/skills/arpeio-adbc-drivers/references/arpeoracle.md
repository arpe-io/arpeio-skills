# ArpeOracle – Oracle ADBC driver (0.4.2)

Load name `arpeoracle`. Native TNS + TTC over TCP/TLS: **no OCI, no Instant Client**. Options
`adbc.arpeoracle.*`.

## Targets

- Oracle 19c (primary), 12.2, 18c, 21c, 23ai, 26ai validated. **12.1 is the minimum**: 11g and older are
  refused at connect.
- Oracle Cloud Autonomous Database: mutual TLS with the downloaded wallet (`ewallet.pem`), port **1522**,
  `_low` / `_tp`… service, `ssl_mode=verify-full`; password, OCI IAM token or Entra ID login.
- RAC / SCAN listeners: redirects followed. Oracle Spatial `SDO_GEOMETRY`: read / write as WKB.
- Not supported: partitioned reads, Substrait, `sqlnet.ora`, LDAP naming, NTS / Windows integrated auth,
  Kerberos (code and `krb5_*` options exist, but it is compiled out of the released binaries), `BFILE`,
  user object types, collections.
- The session opens on first use (first query / metadata call).

## Connecting

All forms are database options. A discrete option always wins; between a URI and a connection string,
the later one wins. Service name **or** SID, not both.

**URI**: `oracle://[user[:password]@]host[:port][/service_name][?key=value&…]` (also
`arpeoracle://`). Path = **service name**; `?sid=ORCL` for a SID. Parameters: `user`, `password`,
`service_name`, `sid`, `ssl_mode`, `ssl_root_cert`, `wallet_location`, `wallet_password`,
`encryption`, `data_integrity`, `connect_timeout`, `proxy_user`, `number_mapping`,
`interval_mapping`, `application_name`, `allowed_logon_version` (0.4.1+). IPv6: `[::1]`.

```text
oracle://scott:<pw>@dbhost:2484/orclpdb1?ssl_mode=verify-full&wallet_location=/etc/oracle/wallet
oracle://scott:<pw>@dbhost:1521?sid=ORCL
```

**Connection string** (`adbc.arpeoracle.connection_string`, ADO.NET keywords): `Server` / `Data
Source` / `Host` / `Address` (host, **EZConnect**, TNS descriptor or tnsnames alias), `Port`, `Service
Name` / `Database`, `SID`, `User ID` / `UID` / `User`, `Password` / `PWD`, `Auth Method`, `Proxy User`,
`New Password`, `Allowed Logon Version` (0.4.1+), `SSL Mode`, `SSL Root Cert`, `Wallet Location`,
`Wallet Password`, `Encryption`, `Data Integrity`, `Connection Timeout`, `Socket Timeout`, `Buffer
Size`, `Number Mapping`, `Interval Mapping`, `Application Name`.

```text
Server=dbhost:2484/orclpdb1;User ID=scott;Password=<pw>;SSL Mode=verify-full;Wallet Location=/etc/oracle/wallet
```

Over-long values are refused, never cut (0.4.1+): password 1024 bytes, user / host / service 255.

`server` (option or `Server` keyword) accepts:
- **EZConnect** `host:port/service` (optional leading `//`), or `host,port`. Bare IPv6 is refused.
- **TNS descriptor** `(DESCRIPTION=(ADDRESS=(PROTOCOL=TCPS)(HOST=…)(PORT=…))(CONNECT_DATA=(SERVICE_NAME=…)))`:
  only the first address and `SERVICE_NAME` / `SID` are used (no failover / load balancing). `TCPS` →
  `ssl_mode=verify-full` unless set (**0.4.1+**; `require` with no check before). From `SECURITY`:
  `SSL_SERVER_DN_MATCH=no` → `verify-ca`, `SSL_SERVER_CERT_DN` → DN match; wallet entries ignored.
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
| `adbc.arpeoracle.allowed_logon_version` | `11` | 0.4.1+. `12` (or `12a`) refuses the 11G (SHA-1) verifier |
| `adbc.arpeoracle.proxy_user` | – | `CONNECT THROUGH` target schema |
| `adbc.arpeoracle.new_password` | – | Change an expired password at login |
| `adbc.arpeoracle.token_location` | – | OCI IAM: folder with `token` + `oci_db_key.pem` (`oci iam db-token get`, default `~/.oci/db-token`) |
| `adbc.arpeoracle.access_token` / `.token_file` | – | OAuth2 / Entra JWT, inline or file (selects `oauth2`) |
| `adbc.arpeoracle.ssl_mode` | `disable` | `require`, `verify-ca`, `verify-full` (chain + host). TCPS descriptor / alias: `verify-full` (0.4.1+) |
| `adbc.arpeoracle.ssl_root_cert` | – | PEM CA bundle (replaces wallet CAs and system trust) |
| `adbc.arpeoracle.ssl_server_cert_dn` | – | 0.4.1+. `verify-full` matches the cert subject to this DN instead of the host |
| `adbc.arpeoracle.wallet_location` / `.wallet_password` | – | Folder with `ewallet.pem` (client cert + key + CAs) |
| `adbc.arpeoracle.encryption` / `.data_integrity` | `accepted` | Native Network Encryption stance: `rejected`, `accepted`, `requested`, `required` |
| `adbc.arpeoracle.login_timeout` | `30` | Seconds for the whole login (TCP, TLS, NNE, auth; 0.4.1+, TCP only before), `0` = no limit |
| `adbc.arpeoracle.socket_timeout` | `0` (none) | Per read / write. TCP keepalive always on (0.4.1+): dead server seen in ~2 min |
| `adbc.arpeoracle.batch_size` | `32000` | Rows per Arrow batch (max 10,000,000); a memory knob |
| `adbc.arpeoracle.sdu` | `65535` | Session data unit, 512..2097152 (8192 before 0.4.1); server keeps the lower |
| `adbc.arpeoracle.statement_cache_size` | `20` | Server cursors reused per connection (`0` = off) |
| `adbc.arpeoracle.number_mapping` | `auto` | Unconstrained `NUMBER` / `FLOAT`: `auto` → utf8 (lossless), `double` → float64, `decimal:P,S` → decimal128(P,S) |
| `adbc.arpeoracle.interval_mapping` | `parquet` | `parquet`: YEAR TO MONTH → int64 months, DAY TO SECOND → ISO-8601 text; `native` → interval[month_day_nano] |
| `adbc.arpeoracle.app_name` | `ArpeOracle` | `V$SESSION.PROGRAM` / `MODULE` |
| `adbc.arpeoracle.ingest_method` | `auto` | 0.4.0+. `auto`, `direct_path`, `insert` (also a statement option) |
| `adbc.arpeoracle.ingest_batch_rows` | `50000` | 0.4.1+. Max rows per `INSERT` execute (also ~16 MiB cap); statement option too |
| `adbc.arpeoracle.direct_path_parallel` | `false` | 0.4.2+. Parallel direct path loads into one table (statement option too) |
| `adbc.arpeoracle.openssl_version` | read-only | 0.4.1+. Bundled OpenSSL (3.6.5); also GetInfo code `10000` |
| `arpeio.adbc.license`, `arpeio.adbc.license_file` | – | Licence (re-checked at every connection) |

Enumerated values (`ssl_mode`, `encryption`, `auth_method`, `number_mapping`…) are case-insensitive;
`autocommit`, `temporary` and ingest modes are exact. Discrete-only (no URI / connection-string form):
`sdu`, `statement_cache_size`, `ssl_server_cert_dn`, `ingest_*`, `direct_path_parallel`, tokens, licence.

Connection options: `adbc.connection.autocommit` (`true` / `false`), `adbc.connection.db_schema`
(settable, 0.3.12+: `CURRENT_SCHEMA`, case-sensitive → upper case), `adbc.connection.catalog` (read-only).

## Authentication

| Mode | Options |
|---|---|
| Password | `username` + `password` |
| Proxy | proxy account `username` + `password`, `proxy_user` = target (`ALTER USER target GRANT CONNECT THROUGH proxy`) |
| Expired password | add `new_password` |
| OCI IAM token | `auth_method=token` (explicit) + `token_location`; no username. **TLS required** (0.4.1+). ADB: + wallet + `ssl_mode=verify-full` |
| OAuth2 / Entra ID | `access_token` or `token_file` (no username). **TLS required** (0.4.1+). ADB with Entra: the App ID URI must be `https://<tenant>.onmicrosoft.com/<appId>` |
| Mutual TLS | `wallet_location` (+ `wallet_password`) with `ssl_mode` ≠ `disable` |

Without TLS a MITM can downgrade O5LOGON to the 11G verifier: `verify-full` or `allowed_logon_version=12`.

## Encryption

- Defaults: `ssl_mode=disable` + NNE `accepted` → **plaintext** against a server left at `ACCEPTED`. For
  encryption use `ssl_mode=verify-full` (TCPS), or `encryption=required` + `data_integrity=required`
  (NNE: AES-256 with SHA-256 / SHA-1; passive eavesdropping only, TLS also stops an active attacker).
- TLS 1.2 minimum. Trust: `ssl_root_cert` → wallet → system store (0.3.12+: Linux distro bundles, RHEL
  included; Windows ROOT store). `require` with a CA or wallet verifies the chain (0.4.1+).
- `verify-full` by IP needs an iPAddress SAN (0.4.1+); after a SCAN redirect either host name passes.
- Wallets: only `ewallet.pem` is read; for `cwallet.sso` run `orapki wallet pkcs12_to_pem -wallet <dir>`.
- NNE stance vs server: `accepted` turns encryption on only if the server `REQUESTED` / `REQUIRED` it;
  `required` fails if the server `REJECTED` it. Since 0.4.1, `encryption=requested`/`required` always
  brings integrity (`data_integrity=rejected` alongside is refused), and DH groups under 2048 bits fail.

## Types

Read (Oracle → Arrow): `NUMBER(p,0)` p ≤ 18 → int64; other constrained `NUMBER(p,s)` → decimal128(p,s);
`NUMBER(p,-s)` → decimal128(p+s,0) (0.4.1+); unconstrained `NUMBER` / `FLOAT` → per `number_mapping`
(default **utf8**); `BINARY_FLOAT/DOUBLE` → float32/64; `VARCHAR2`, `CHAR`, `NVARCHAR2`, `CLOB`,
`NCLOB`, `LONG`, `ROWID` → utf8; `BLOB`, `RAW` → binary; **`DATE` → timestamp[s]** (not date32);
`TIMESTAMP(n)` → s / ms / us / ns by n; `WITH (LOCAL) TIME ZONE` → tz UTC; intervals per
`interval_mapping`; `BOOLEAN` (23ai) → bool; `JSON` (21c+ columns and expressions, 23ai) → utf8;
`VECTOR` → list<float32 | float64 | int8>; `SYS.XMLTYPE` (0.3.12+) → utf8; `SDO_GEOMETRY` → WKB
(`geoarrow.wkb`; point, line, single-ring polygon; 3D / M / ZM, rectangles since 0.4.1).

- Each field carries `ARPEORACLE:type` metadata (`VARCHAR2(100)`, `NUMBER(12,2)`…), 0.3.12+.
- Values that do not fit fail, never round (0.4.1+): `decimal:P,S` overflow → SQLSTATE 22003;
  `TIMESTAMP(7..9)` outside 1677..2262 → 22008 (`CAST` to `TIMESTAMP(6)`).
- Dates before 1582-10-15 convert from Oracle's Julian calendar (0.4.1+). LOB batches cut at 2 GiB.

Ingest `create` DDL: bool → NUMBER(1); int8 → NUMBER(3); int16 → NUMBER(5); int32 → NUMBER(10); int64
→ NUMBER(19); uint64 → NUMBER(20) (0.4.1+); float32/64 → BINARY_FLOAT / BINARY_DOUBLE; decimal128 →
NUMBER(p,s); **utf8 / string_view → VARCHAR2(4000)**; **binary / binary_view → RAW(2000)**;
fixed_size_binary(n ≤ 2000) → RAW(n); date → DATE; timestamp → TIMESTAMP(0/3/6/9) (tz → WITH TIME
ZONE); null → VARCHAR2(1); `geoarrow.wkb` → SDO_GEOMETRY. Refused: float16, decimal256, time,
intervals, lists, structs. Dictionary columns are written as their values.

- Strings are always bound as VARCHAR2 (max 4000 bytes) and binary as RAW (max 2000 bytes), even when
  appending into CLOB / BLOB columns: longer values cannot be ingested with this driver.
- Names are quoted as given: `my_table` becomes the case-sensitive `"my_table"`. Use upper case.

## Ingest

`cur.adbc_ingest("SALES_STAGE", data, mode=…, db_schema_name="SCOTT", temporary=False)`. Optional DDL,
then either an Oracle **direct path load** (0.4.0+; 6M rows in 2.9 s vs 112 s) or an array-bound `INSERT` per
batch (split at `ingest_batch_rows`, 0.4.1+).

| Mode | Behaviour |
|---|---|
| `create` | `CREATE TABLE` (fails if it exists, ORA-00955) |
| `append` | Insert only |
| `replace` | `DROP TABLE … PURGE`, then create |
| `create_append` | Create if missing |

| `ingest_method` | Behaviour |
|---|---|
| `auto` | Direct path only with **autocommit on** and a table with no enabled INSERT trigger, foreign key, primary / unique key, needed column `DEFAULT` (identity, `DEFAULT ON NULL`), unusable index, or non-loadable type; else `INSERT` |
| `direct_path` | Always, or fail; **commits even with autocommit off**; duplicate keys commit, then `INTEGRITY` naming the `UNUSABLE` indexes |
| `insert` | Per-row semantics, triggers, constraints, caller's transaction |

- Direct path locks the table exclusively, skips triggers / foreign keys, needs no pending work on the
  connection, and does not load LOB, `LONG`, `NCHAR` / `NVARCHAR2`, intervals, LTZ, objects, 23ai types,
  or text in a non-UTF-8 database. Spatial columns and temporary tables always use `INSERT`.
- Python dbapi connects with autocommit off, so `auto` → `INSERT`: pass `autocommit=True` for speed.
- `direct_path_parallel=true` (0.4.2+, on each connection): loads from several connections into one
  table run concurrently (shared lock). The table must have **no index** (ORA-26002: `direct_path` →
  `NOT_IMPLEMENTED`, `auto` → `INSERT`). Each connection commits separately.
- `temporary=True` → `GLOBAL TEMPORARY TABLE … ON COMMIT PRESERVE ROWS`. `catalog_name` = service name.
- Autocommit on: each batch commits as it goes (earlier batches stay if a later one fails; a table the
  call created is dropped again, 0.4.1+). dbapi default (off): `conn.commit()`.
- Spatial: after creating a table with `geoarrow.wkb` columns, the driver registers
  `USER_SDO_GEOM_METADATA` and a spatial index (`arpeoracle.sdo.spatial_index=false` to skip, required
  for a non-upper-case name, 0.4.1+). `arpeoracle.sdo.srid` forces the read SRID. 3D WKB is refused.

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
}, autocommit=True)  # autocommit on: adbc_ingest can use direct path
with conn.cursor() as cur:
    cur.execute("SELECT * FROM hr.employees WHERE department_id = :1", parameters=[50])
    print(cur.fetch_arrow_table())
```

## Troubleshooting

ORA errors carry their ADBC status, SQLSTATE and ORA number as `vendor_code` (0.4.1+; before: `INVALID_ARGUMENT`).

| Symptom | Fix |
|---|---|
| `no server or service name configured …` | Set `service_name` (or `/service` in EZConnect / URI) |
| `TNS alias 'X' not found in <file> …` | Fix the alias, or give `service_name` (a host without a service is looked up as an alias) |
| `… (Oracle 11g or older) … 12.1 or later is required` | Server too old |
| `login timed out after N s during the <step>` | Step names where it stalled (TCP connect, TLS handshake, TNS connect…) |
| `'<Keyword>' value too long (max N bytes)` | Value over its limit (0.4.1+) |
| `no username configured` | Password login needs `username` (or use token / oauth2) |
| ORA-01017 | Credentials; proxy grant; Entra App ID URI (`https://…`); identity not mapped (`IDENTIFIED GLOBALLY`); expired token |
| `auth_method=token requires token_location …` / `… requires a TLS (TCPS) connection` | Give `token_location` / set `ssl_mode=verify-full` (0.4.1+) |
| `server offered the 11G password verifier, refused by allowed_logon_version=12` | Reset the password to get a 12C verifier, or MITM |
| `Kerberos (KERBEROS5) authentication is unavailable …` | Not in released binaries: use another method |
| `unable to get local issuer certificate` (also TCPS descriptors since 0.4.1) | `ssl_root_cert` / wallet, 0.3.12+ (system store), or `ssl_mode=require` |
| `certificate is for neither … nor …` / DN mismatch | Check host, IP SAN, `ssl_server_cert_dn` |
| `wallet_location/ssl_root_cert were given but ssl_mode is 'disable'` | Set `ssl_mode` |
| `wallet '<dir>' holds cwallet.sso …` | `orapki wallet pkcs12_to_pem -wallet <dir>` |
| `… negotiated without data integrity` | Server `CRYPTO_CHECKSUM_SERVER=REJECTED`: enable it, or use TCPS |
| ORA-12660 / ORA-12650 | NNE stance or cipher mismatch: align stances; server must allow AES256 and SHA256/SHA1 |
| Slower connects | `encryption=rejected` + `data_integrity=rejected` if NNE is not wanted |
| NUMBER columns come back as strings | `number_mapping=double` or `decimal:P,S` |
| `… out of range for decimal128(P,S)` (22003) / `timestamp[ns]` (22008) | Widen `P,S` / `CAST(col AS TIMESTAMP(6))` |
| `column … (Oracle type N) is not supported by this driver yet` | Convert in SQL (e.g. `TO_CHAR`) or exclude the column |
| Ingested table "does not exist" | Names are case-sensitive: use upper case |
| `string of N bytes exceeds Oracle's 4000-byte VARCHAR2 limit …` | Values over 4000 bytes can't be ingested |
| Ingest slow (Python, or one table only) | Autocommit off, or keys / triggers / FKs / defaults → `INSERT`: `autocommit=True`, key-less staging table |
| `direct path load cannot be used on "T": it has enabled foreign keys` (or triggers, defaults) | Use `auto` / `insert`, or disable them |
| `direct path load committed N rows … UNUSABLE` | Duplicate keys: dedupe, `ALTER INDEX … REBUILD` |
| Ingest committed with autocommit off / waits on a lock | `direct_path` always commits and locks the table exclusively |
| Spatial ingest `INVALID_ARGUMENT` naming `arpeoracle.sdo.spatial_index` | Upper-case table name, or `spatial_index=false` |
| `a result stream from this connection is still open …` | Read / close the previous result |
| Commit / Rollback `INVALID_STATE` | Turn autocommit off first |
| `Commit` fails `INVALID_STATE` / 08007 (session lost; "committed is unknown") | `Rollback`, check the data, redo (0.4.1+) |
| `cannot allocate memory in static TLS block` (two Arpe drivers in one process, Linux) | Upgrade to 0.3.12+ |

Docs vs driver 0.4.2: the docs stop at 0.4.1 and do not cover `direct_path_parallel`; the data-types
page still says a bind runs one execution per Arrow row (the driver array-binds each batch); the docs
do not mention the Kerberos code paths (compiled out of released builds).
