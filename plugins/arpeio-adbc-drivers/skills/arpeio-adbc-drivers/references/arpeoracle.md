# ArpeOracle – Oracle ADBC driver (0.4.5)

Load name `arpeoracle`. Native TNS + TTC over TCP/TLS: **no OCI, no Instant Client**. Options
`adbc.arpeoracle.*`.

## Targets

- Oracle 19c (primary), 12.2, 18c, 21c, 23ai, 26ai validated. **12.1 is the minimum**: 11g and older are
  refused at connect.
- Oracle Cloud Autonomous Database: mutual TLS with the downloaded wallet (`ewallet.pem`), port **1522**,
  `_low` / `_tp`… service, `ssl_mode=verify-full`; password, OCI IAM token or Entra ID login.
- RAC / SCAN listeners: redirects followed. Oracle Spatial `SDO_GEOMETRY`: read / write as WKB.
- Not supported: partitioned reads, Substrait, `sqlnet.ora`, LDAP naming, NTS / Windows integrated auth
  (SSPI), Kerberos on Windows, `BFILE`, `REF CURSOR` OUT parameters, writing user object types (other
  than `SDO_GEOMETRY`).
- The session opens on first use (first query / metadata call).

## Connecting

Three forms, all database options. **Every option** is accepted in all three (0.4.3+), except `server`
and `port`, which a URI carries only in its authority (`?port=` is refused). Service name **or** SID,
not both.

Precedence (0.4.3+):
- Before `AdbcDatabaseInit` (the `dbapi.connect(db_kwargs=…)` case), order does not matter; per field:
  discrete key > `adbc.arpeoracle.server` (with its port / service) > `connection_string` > `uri`.
- Two discrete keys for one field that disagree (`sid` vs `service_name`, `username` vs
  `adbc.arpeoracle.username`) → `INVALID_ARGUMENT`; the same value twice is fine.
- After Init (`set_options` on an initialised database): the last value set wins.
- A string / URI sets only the fields it carries, is parsed whole before applying (a bad one changes
  nothing), and must not give one option twice under any keyword (`User ID=a;UID=b` is refused).
- Up to 0.4.2 a discrete option always won, and URI vs connection string went by call order.

**URI**: `oracle://[user[:password]@]host[:port][/service_name][?key=value&…]` (also
`arpeoracle://`). Path = **service name**; `?sid=ORCL` for a SID. Query parameters: every option's short
name (`?auth_method=kerberos&krb5_cred_mode=ccache`, `?sdu=65535`, `?token_file=…`), plus `user`,
`connect_timeout`, `application_name`, `buffer_size`, `sslmode`, `sslrootcert`. Percent-decoded (`+` =
space in the query); a `%00` escape is refused (0.4.4+). IPv6: `[::1]`. Up to 0.4.2 only a fixed subset was accepted.

```text
oracle://scott:<pw>@dbhost:2484/orclpdb1?ssl_mode=verify-full&wallet_location=/etc/oracle/wallet
oracle://scott:<pw>@dbhost:1521?sid=ORCL
```

**Connection string** (`adbc.arpeoracle.connection_string`, ADO.NET grammar): every option's short name
is a keyword, `_` matching a space (`ssl_mode` = `SSL Mode`; `Token Location`, `Access Token`, `Token
File`, `SDU`, `TNS Admin`, `Ingest Method`, `Direct Path Parallel`, `License File`…). Extra keywords:
`Server` / `Data Source` / `Host` / `Address` / `Addr` (host, **EZConnect**, TNS descriptor or tnsnames
alias), `Service Name` / `Database` / `Initial Catalog`, `User ID` / `UID` / `User`, `Password` / `PWD`,
`Application Name` / `App`, `Connection Timeout` / `Connect Timeout` / `Timeout` (→ `login_timeout`),
`Buffer Size` (→ `batch_size`), `sslmode`, `sslrootcert`, `Kerberos Cred Mode` / `Cache` / `Keytab` /
`Principal` / `Password` / `SPN` / `Realm` / `Forward` (→ `krb5_*`). Unknown keyword → refused.

```text
Server=dbhost:2484/orclpdb1;User ID=scott;Password=<pw>;SSL Mode=verify-full;Wallet Location=/etc/oracle/wallet
```

Never read from a string / URI: `adbc.connection.*`, `adbc.ingest.*`, `arpeoracle.sdo.*`, read-only keys.

Over-long values are refused, never cut (bytes, same in every form): `password` / `new_password` 1024;
`username`, `proxy_user`, host, service / SID, `wallet_password`, `krb5_spn` / `_realm` / `_principal` /
`_password`, a whole EZConnect value 255; `app_name` 127; file / folder paths (`ssl_root_cert`,
`wallet_location`, `tns_admin`, `krb5_keytab`, `krb5_ccache`, `token_location`, `token_file`, licence
file) 1023; `ssl_server_cert_dn` 511; `access_token`, licence, a whole `(DESCRIPTION=…)` 8191 (0.4.3+).
A whole string / URI value ≥ 16384 bytes is refused. A host, service name or SID holding `(`, `)`,
`=`, a quote, a backslash, whitespace or a control character is refused before connecting (0.4.3+).

`server` (option or `Server` keyword) accepts:
- **EZConnect** `host:port/service` (optional leading `//`), or `host,port`. Bare IPv6 is refused.
- **TNS descriptor** `(DESCRIPTION=(ADDRESS=(PROTOCOL=TCPS)(HOST=…)(PORT=…))(CONNECT_DATA=(SERVICE_NAME=…)))`:
  only the first address and `SERVICE_NAME` / `SID` are used (no failover / load balancing). `TCPS` →
  `ssl_mode=verify-full` unless set (**0.4.1+**; `require` with no check before). From `SECURITY`:
  `SSL_SERVER_DN_MATCH=no` → `verify-ca`, `SSL_SERVER_CERT_DN` → DN match; wallet entries ignored. An
  explicit `ssl_mode` always wins.
- **tnsnames.ora alias** (0.3.11+): a bare name with no service name / SID anywhere is looked up in
  `tnsnames.ora` from `adbc.arpeoracle.tns_admin`, else `$TNS_ADMIN`, else
  `$ORACLE_HOME/network/admin` (`IFILE` supported). Caution: a real host name without a service name is
  also looked up as an alias. An explicit discrete `port` (or `Port` in the same string) wins over the
  entry's `PORT` (0.4.4+; the entry's always won before).

`AdbcDatabaseGetOption` reads any option back (0.4.3+), except secrets (`NOT_FOUND`).

## Options (database, `db_kwargs`)

| Key | Default | Notes |
|---|---|---|
| `adbc.arpeoracle.server` | – | Host, EZConnect, descriptor or alias |
| `adbc.arpeoracle.port` | `1521` | TCPS listeners often 2484 (on-prem) / 1522 (ADB) |
| `adbc.arpeoracle.service_name` / `.sid` | – | One of them |
| `adbc.arpeoracle.tns_admin` | – | Folder of `tnsnames.ora` |
| `adbc.arpeoracle.username` / `.password` | – | Password login (O5LOGON; the password is never sent in clear) |
| `adbc.arpeoracle.auth_method` | `password` | `password`, `kerberos` (Linux, 0.4.4+), `token` (OCI IAM), `oauth2` (Entra / OAuth2). Only `oauth2` is auto-selected |
| `adbc.arpeoracle.allowed_logon_version` | `11` | 0.4.1+. `12` (or `12a`) refuses the 11G (SHA-1) verifier |
| `adbc.arpeoracle.proxy_user` | – | `CONNECT THROUGH` target schema |
| `adbc.arpeoracle.new_password` | – | Change an expired password at login |
| `adbc.arpeoracle.token_location` | – | OCI IAM: folder with `token` + `oci_db_key.pem` (`oci iam db-token get`, default `~/.oci/db-token`) |
| `adbc.arpeoracle.access_token` / `.token_file` | – | OAuth2 / Entra JWT, inline or file (selects `oauth2`; inline wins) |
| `adbc.arpeoracle.krb5_cred_mode` | `ccache` | Kerberos: `ccache` (ambient `kinit` / `KRB5CCNAME`), `keytab`, `password` |
| `adbc.arpeoracle.krb5_ccache` / `.krb5_keytab` | – | Credential cache (`ccache` mode) / keytab file (`keytab` mode) |
| `adbc.arpeoracle.krb5_principal` / `.krb5_password` | – | Client principal (`keytab`, `password` modes) / its password (`password` mode) |
| `adbc.arpeoracle.krb5_spn` | `oracle/<host>` | Database service principal |
| `adbc.arpeoracle.krb5_realm` | – | Default realm when the principal has none |
| `adbc.arpeoracle.krb5_forward` | `false` | `true` delegates the user's TGT to the server (database links); needs a forwardable ticket |
| `adbc.arpeoracle.ssl_mode` | `disable` | `require`, `verify-ca`, `verify-full` (chain + host). TCPS descriptor / alias: `verify-full` (0.4.1+) |
| `adbc.arpeoracle.ssl_root_cert` | – | PEM CA bundle (replaces wallet CAs and system trust) |
| `adbc.arpeoracle.ssl_server_cert_dn` | – | 0.4.1+. `verify-full` matches the cert subject to this DN instead of the host |
| `adbc.arpeoracle.wallet_location` / `.wallet_password` | – | Folder with `ewallet.pem` (client cert + key + CAs) |
| `adbc.arpeoracle.encryption` / `.data_integrity` | `accepted` | Native Network Encryption stance: `rejected`, `accepted`, `requested`, `required` |
| `adbc.arpeoracle.data_integrity_types` | `sha256,sha1` | 0.4.3+. NNE integrity algorithms offered (`sha256`, `sha1`, comma list); the server picks |
| `adbc.arpeoracle.login_timeout` | `30` | Seconds for the whole login (TCP, TLS, NNE, auth; 0.4.1+, TCP only before), `0` = no limit |
| `adbc.arpeoracle.socket_timeout` | `0` (none) | Per read / write. TCP keepalive always on (0.4.1+): dead server seen in ~2 min |
| `adbc.arpeoracle.batch_size` | `32000` | Rows per Arrow batch (max 10,000,000); a memory knob |
| `adbc.arpeoracle.sdu` | `65535` | Session data unit, 512..2097152 (8192 before 0.4.1); server keeps the lower |
| `adbc.arpeoracle.statement_cache_size` | `20` | Server cursors reused per connection (`0` = off) |
| `adbc.arpeoracle.number_mapping` | `string` | Unconstrained `NUMBER` / `FLOAT`: `string` → utf8 (lossless; called `auto` before 0.4.4, still accepted), `double` → float64, `decimal:P,S` → decimal128(P,S) |
| `adbc.arpeoracle.interval_mapping` | `parquet` | `parquet`: YEAR TO MONTH → int64 months, DAY TO SECOND → ISO-8601 text; `native` → interval[month_day_nano] |
| `adbc.arpeoracle.unsupported_types` | `error` | 0.4.4+. `binary`: an undecodable column reads as raw bytes (`ARPEORACLE:raw=true`) instead of failing; `BFILE` / `REF CURSOR` still fail |
| `adbc.arpeoracle.app_name` | `ArpeOracle` | `V$SESSION.PROGRAM` / `MODULE` |
| `adbc.arpeoracle.ingest_method` | `auto` | 0.4.0+. `auto`, `direct_path`, `insert` (also a statement option) |
| `adbc.arpeoracle.ingest_batch_rows` | `50000` | 0.4.1+. Max rows per `INSERT` execute (also ~16 MiB cap); statement option too |
| `adbc.arpeoracle.direct_path_parallel` | `false` | 0.4.2+. `true`: direct path load in parallel mode (shared table lock); table must have no index (ORA-26002: `direct_path` → `NOT_IMPLEMENTED`, `auto` → `INSERT`). Statement option too |
| `adbc.arpeoracle.openssl_version` | read-only | 0.4.1+. Bundled OpenSSL (3.5.9 LTS since 0.4.5, 3.6.5 before); also GetInfo code `10000` |
| `arpeio.adbc.license`, `arpeio.adbc.license_file` | – | Licence (re-checked at every connection) |

Enumerated values (`ssl_mode`, `encryption`, `auth_method`, `number_mapping`…) are case-insensitive;
`autocommit`, `temporary` and ingest modes are exact.

Connection options (`conn_kwargs`):
- `adbc.connection.autocommit` (`true` / `false`).
- `adbc.connection.db_schema`: `CURRENT_SCHEMA`, used as given (quoted: pass upper case); unknown →
  `NOT_FOUND`, unchanged. 0.4.4+: reading it asks the session (sees SQL `ALTER SESSION`, proxy, Kerberos
  / token schema); can be set before connection init (`conn_kwargs={"adbc.connection.db_schema": "HR"}`,
  checked at init); readable / settable while a result stream is open.
- `adbc.connection.catalog` (read-only, the service name).
- Read-only (0.4.5+): `adbc.arpeoracle.tls.version`, `.tls.group`, `.tls.cipher` (negotiated TLS; `""`
  on plain TCP or before the session opens). `ARPEIO_ADBC_LOG_TLS=1` prints them on stderr.

## Authentication

| Mode | Options |
|---|---|
| Password | `username` + `password` |
| Proxy | proxy account `username` + `password`, `proxy_user` = target (`ALTER USER target GRANT CONNECT THROUGH proxy`) |
| Expired password | add `new_password` |
| Kerberos (0.4.4+, **Linux only**) | `auth_method=kerberos` (explicit), `krb5_*`; no username. User `IDENTIFIED EXTERNALLY AS '<principal>'`. Needs MIT Kerberos on the host (`krb5-libs` / `libkrb5-3`), loaded at first Kerberos login (`ARPEORACLE_KRB5_LIBRARY` to point elsewhere). Combine with `encryption=required` for NNE |
| OCI IAM token | `auth_method=token` (explicit) + `token_location`; no username. **TLS required** (0.4.1+): use `verify-full` / `verify-ca` (`require` without a CA lets a MITM relay the logon). ADB: + wallet |
| OAuth2 / Entra ID | `access_token` or `token_file` (no username). **TLS required** (0.4.1+), verified as above. ADB with Entra: the App ID URI must be `https://<tenant>.onmicrosoft.com/<appId>` |
| Mutual TLS | `wallet_location` (+ `wallet_password`) with `ssl_mode` ≠ `disable` |

Token options are also keywords / URI parameters since 0.4.3 (`Token Location`, `Access Token`, `Token
File`); discrete options only before. Without TLS a MITM can downgrade O5LOGON to the 11G verifier:
`verify-full` or `allowed_logon_version=12`.

## Encryption

- Defaults: `ssl_mode=disable` + NNE `accepted` → **plaintext** against a server left at `ACCEPTED`. For
  encryption use `ssl_mode=verify-full` (TCPS), or `encryption=required` + `data_integrity=required`
  (NNE: AES-256 with SHA-256 / SHA-1; passive eavesdropping only, TLS also stops an active attacker).
- TLS 1.2 minimum, 1.3 when offered; 0.4.5+ offers hybrid `X25519MLKEM768` first. Trust: `ssl_root_cert`
  → wallet → system store (0.3.12+: Linux distro bundles, RHEL included; Windows ROOT store). `require`
  with a CA or wallet verifies the chain (0.4.1+).
- `verify-full` by IP needs an iPAddress SAN (0.4.1+); after a SCAN redirect either host name passes.
- Wallets: only `ewallet.pem` is read; for `cwallet.sso` run `orapki wallet pkcs12_to_pem -wallet <dir>`.
- NNE stance vs server: `accepted` turns encryption on only if the server `REQUESTED` / `REQUIRED` it;
  `required` fails if the server `REJECTED` it. Since 0.4.1, `encryption=requested`/`required` always
  brings integrity (`data_integrity=rejected` alongside is refused), and DH groups under 2048 bits fail.
- Integrity algorithm: the server chooses from the offer (21c defaults pick SHA-1). To exclude SHA-1:
  `data_integrity_types=sha256` **with** `data_integrity=required` (otherwise no common algorithm just
  turns integrity off).

## Types

Read (Oracle → Arrow): `NUMBER(p,0)` p ≤ 18 → int64; other constrained `NUMBER(p,s)` → decimal128(p,s);
`NUMBER(p,-s)` → decimal128(p+s,0) (0.4.1+); unconstrained `NUMBER` / `FLOAT` → per `number_mapping`
(default **utf8**); `BINARY_FLOAT/DOUBLE` → float32/64; `VARCHAR2`, `CHAR`, `NVARCHAR2`, `CLOB`,
`NCLOB`, `LONG`, `ROWID` → utf8; `BLOB`, `RAW` → binary; **`DATE` → timestamp[s]** (not date32);
`TIMESTAMP(n)` → s / ms / us / ns by n; `WITH (LOCAL) TIME ZONE` → tz UTC; intervals per
`interval_mapping`; `BOOLEAN` (23ai) → bool; `JSON` (21c+ columns and expressions, 23ai) → utf8;
`VECTOR` → list<float32 | float64 | int8> (`BINARY` → list<uint8>); `VECTOR(n, fmt, SPARSE)` (0.4.4+) →
struct<num_dimensions int64, indices list<uint32>, values list<…>>; `SYS.XMLTYPE` (0.3.12+) → utf8;
`SDO_GEOMETRY` → WKB (`geoarrow.wkb`; point, line, polygon, 3D / M / ZM, rectangles since 0.4.1;
polygons with holes and multi-point / line / polygon since 0.4.4; collections, arcs, circles refused).

- User `OBJECT` → struct (attributes in order), `VARRAY` / nested table → list (`item`), nested to any
  depth (0.4.4+; failed before). Refused: `NOT FINAL` types, LOB / `REF` / `XMLTYPE` / opaque attributes,
  types not visible in `ALL_TYPES`.
- Undecodable columns (those objects, flexible `VECTOR(n, *)`) fail naming the column, or read as raw
  binary with `unsupported_types=binary` (0.4.4+). `BFILE` and `REF CURSOR` always fail.
- Each field carries `ARPEORACLE:type` metadata (`VARCHAR2(100)`, `NUMBER(12,2)`…), 0.3.12+.
- Values that do not fit fail, never round (0.4.1+): `decimal:P,S` overflow → SQLSTATE 22003;
  `TIMESTAMP(7..9)` outside 1677..2262 → 22008 (`CAST` to `TIMESTAMP(6)`).
- Dates before 1582-10-15 convert from Oracle's Julian calendar (0.4.1+). LOB batches cut at 2 GiB.

Ingest `create` DDL: bool → NUMBER(1); int8 → NUMBER(3); int16 → NUMBER(5); int32 → NUMBER(10); int64
→ NUMBER(19); uint64 → NUMBER(20) (0.4.1+); float32/64 → BINARY_FLOAT / BINARY_DOUBLE; decimal32 / 64 /
128 / 256 → NUMBER(p,s) (p ≤ 38; 32 / 64 / 256 since 0.4.4); **utf8 / string_view → VARCHAR2(4000)**,
**binary / binary_view → RAW(2000)**, each → CLOB / BLOB if a value of the **first batch** is longer
(0.4.4+); fixed_size_binary(n) → RAW(n), BLOB over 2000 (0.4.4+); date → DATE; timestamp →
TIMESTAMP(0/3/6/9) (tz → WITH TIME ZONE); null → VARCHAR2(1); `geoarrow.wkb` → SDO_GEOMETRY (2D);
list<float32 | float64 | int8> → `VECTOR(*, FMT)`, fixed_size_list[n] → `VECTOR(n, FMT)` (23ai+, 0.4.4+).
Refused: float16, time, intervals, other lists, structs. Dictionary columns are written as their values.

- Long values (0.4.4+): strings over 4000 bytes / binary over 2000 bind and ingest through `INSERT` into
  `CLOB` / `NCLOB` / `BLOB` columns; not into `VARCHAR2(4000)` / `RAW(2000)` (ORA-01461 / ORA-12899) and
  not in SQL expressions (ORA-01460). A later batch longer than the first one's columns fails: pre-create
  with `CLOB` / `BLOB` and `append`. Up to 0.4.3 such values were refused.
- `VECTOR` writes (0.4.4+, 23ai+): Arrow lists by bind, `INSERT` and direct path; NULL elements and over
  65535 elements refused; `SPARSE` / `BINARY` not written. Created `VECTOR` columns need an ASSM default
  tablespace (ORA-43853).
- Names are quoted as given: `my_table` becomes the case-sensitive `"my_table"`. Use upper case.

## PL/SQL

- OUT / IN OUT parameters (0.4.4+): mark the bound field with metadata `ORACLE:parameter_type` = `out` /
  `inout`; statement must be `BEGIN … END;` / `DECLARE …`; `cur.execute(sql, batch)` then fetch returns
  one row per bound row, one column per OUT / IN OUT parameter. Strings / RAW up to 32767 bytes.
  `ExecuteUpdate`, `BindStream`, ingest refuse them; `VECTOR` and `REF CURSOR` cannot be OUT.
- Implicit results (0.4.4+): a block calling `DBMS_SQL.RETURN_RESULT` returns its **first** implicit
  result as the query result (others closed). Before 0.4.4 such a block failed and lost the session.

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
| `auto` | Direct path only with **autocommit on** and a table with no enabled INSERT trigger, foreign key, primary / unique key, needed column `DEFAULT` (identity, `DEFAULT ON NULL`), unusable index, LOB column, or non-loadable type; else `INSERT` |
| `direct_path` | Always, or fail; **commits even with autocommit off**; duplicate keys commit, then `INTEGRITY` naming the `UNUSABLE` indexes |
| `insert` | Per-row semantics, triggers, constraints, caller's transaction |

- Direct path locks the table exclusively, skips triggers / foreign keys, needs no pending work on the
  connection, and does not load LOB, `LONG`, `NCHAR` / `NVARCHAR2`, intervals, LTZ, objects, 23ai `JSON`
  / `BOOLEAN`, `SPARSE` / `BINARY` vectors, or text in a non-UTF-8 database. Dense `VECTOR` loads (0.4.4+).
  Spatial columns, temporary tables and tables it creates with `CLOB` / `BLOB` always use `INSERT`.
- Python dbapi connects with autocommit off, so `auto` → `INSERT`: pass `autocommit=True` for speed.
- `temporary=True` → `GLOBAL TEMPORARY TABLE … ON COMMIT PRESERVE ROWS`. `catalog_name` = service name.
- Autocommit on: each batch commits as it goes (earlier batches stay if a later one fails; a table the
  call created is dropped again, 0.4.1+). dbapi default (off): `conn.commit()`.
- A cancel during a direct path load keeps the session (0.4.3+); `CANCELLED` + 08007 = may have committed.
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

ORA errors carry their ADBC status and SQLSTATE (0.4.1+; before: `INVALID_ARGUMENT`). The ORA number:
- **0.4.4+ with ADBC 1.1.0 error details (always in Python)**: `Error.vendor_code` is `None`; read
  `Error.details` keys `arpeoracle.oracle.error_number` (e.g. `942`), `.error_position`, `.message`
  (values are bytes).
- ADBC 1.0.0 callers, and 0.4.1-0.4.3: in `vendor_code`.

| Symptom | Fix |
|---|---|
| `no server or service name configured …` | Set `service_name` (or `/service` in EZConnect / URI) |
| `TNS alias 'X' not found in <file> …` | Fix the alias, or give `service_name` (a host without a service is looked up as an alias) |
| `option 'X' is given different values by 'A' and 'B' …` | Two discrete keys for one field disagree: give it once (0.4.3+) |
| Other server / port / user after upgrading to 0.4.3+ | New precedence: discrete > `server` > connection string > URI before Init; last set wins after |
| Alias connects to another port (0.4.4+) | An explicit `port` now wins over the entry's: remove it |
| `… contains '(', which an Oracle Net connect descriptor cannot carry` | Stray character in host / service / SID |
| `… (Oracle 11g or older) … 12.1 or later is required` | Server too old |
| `login timed out after N s during the <step>` | Step names where it stalled (TCP connect, TLS handshake, TNS connect…) |
| `'<Keyword>' value too long (max N bytes)` | Value over its limit (0.4.1+) |
| `no username configured` | Password login needs `username` (or use kerberos / token / oauth2) |
| ORA-01017 | Credentials; proxy grant; Entra App ID URI (`https://…`); identity not mapped (`IDENTIFIED GLOBALLY`); expired token |
| `auth_method=token requires token_location …` / `… requires a TLS (TCPS) connection` | Give `token_location` / set `ssl_mode=verify-full` (0.4.1+) |
| `server offered the 11G password verifier, refused by allowed_logon_version=12` | Reset the password to get a 12C verifier, or MITM |
| `Kerberos (KERBEROS5) authentication is unavailable: this build has no POSIX Kerberos (GSSAPI) support` | Windows, or Linux before 0.4.4: upgrade on Linux, else another method |
| `… needs the MIT Kerberos client library libkrb5.so.3, which could not be loaded` | Install `krb5-libs` / `libkrb5-3`, or set `ARPEORACLE_KRB5_LIBRARY` |
| Kerberos login fails / SPN mismatch | `klist`; forwardable ticket (`kinit -f`) if `krb5_forward=true`; `krb5_spn` = the database's principal; right `krb5_cred_mode` |
| `unable to get local issuer certificate` (also TCPS descriptors since 0.4.1) | `ssl_root_cert` / wallet, 0.3.12+ (system store), or `ssl_mode=require` |
| `certificate is for neither … nor …` / DN mismatch | Check host, IP SAN, `ssl_server_cert_dn` |
| `wallet_location/ssl_root_cert were given but ssl_mode is 'disable'` | Set `ssl_mode` |
| `wallet '<dir>' holds cwallet.sso …` | `orapki wallet pkcs12_to_pem -wallet <dir>` |
| `… negotiated without data integrity` | Server `CRYPTO_CHECKSUM_SERVER=REJECTED`: enable it, or use TCPS |
| ORA-12660 / ORA-12650 | NNE stance or cipher mismatch: align stances; server must allow AES256 and an algorithm in `data_integrity_types` |
| Session uses SHA-1 integrity | Server's choice: `data_integrity_types=sha256` + `data_integrity=required` |
| Slower connects | `encryption=rejected` + `data_integrity=rejected` if NNE is not wanted |
| NUMBER columns come back as strings | `number_mapping=double` or `decimal:P,S` |
| `… out of range for decimal128(P,S)` (22003) / `timestamp[ns]` (22008) | Widen `P,S` / `CAST(col AS TIMESTAMP(6))` |
| `column X has Oracle type T, which this driver cannot decode` (before 0.4.4: `… is not supported by this driver yet`) | Exclude / convert in SQL, or `unsupported_types=binary` (0.4.4+); objects read as struct / list since 0.4.4 |
| Ingested table "does not exist" | Names are case-sensitive: use upper case |
| ORA-01461 / ORA-12899 on a long value | Target must be `CLOB` / `BLOB` (pre-create, `append`); before 0.4.4 such values can't be written |
| `VECTOR` ingest: ORA-43853 | User's default tablespace must be ASSM (not `SYSTEM`) |
| Ingest slow (Python, or one table only) | Autocommit off, or keys / triggers / FKs / defaults / LOBs → `INSERT`: `autocommit=True`, key-less staging table |
| `direct path load cannot be used on "T": it has enabled foreign keys` (or triggers, defaults) | Use `auto` / `insert`, or disable them |
| `direct path load committed N rows … UNUSABLE` | Duplicate keys: dedupe, `ALTER INDEX … REBUILD` |
| Ingest committed with autocommit off / waits on a lock | `direct_path` always commits and locks the table exclusively |
| ORA-26002 / `NOT_IMPLEMENTED` with `direct_path_parallel` | Table must have no index |
| Spatial ingest `INVALID_ARGUMENT` naming `arpeoracle.sdo.spatial_index` | Upper-case table name, or `spatial_index=false` |
| `a result stream from this connection is still open …` | Only another execute is refused (metadata calls work, 0.4.4+): read / close the previous result |
| Commit / Rollback `INVALID_STATE` | Turn autocommit off first |
| `Commit` fails `INVALID_STATE` / 08007 (session lost; "committed is unknown") | `Rollback`, check the data, redo (0.4.1+) |
| `cannot allocate memory in static TLS block` (two Arpe drivers in one process, Linux) | Upgrade to 0.3.12+ |
