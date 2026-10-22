# ArpeTeradata – Teradata ADBC driver (0.3.5)

Load name `arpeteradata`. Native Teradata parcel protocol (no ODBC, no `teradatasql`): over TLS
(WebSocket on the gateway's HTTPS port) with the driver's own logon, or on the legacy DBS port after a
logon through **your installed TTU CLIv2**, then the driver runs its own protocol on that socket. Options
`adbc.arpeteradata.*`. ADBC 1.1.0.

## Targets

| Server | State |
|---|---|
| Teradata 20.00 (Vantage Express) | Validated: TLS and legacy-port logon, ingest, cancel |
| Teradata 16.20.53.30+ with a gateway certificate | Needed for the TLS logon; older: legacy port only. Not yet validated |
| Teradata cloud (VantageCloud) | Checked on the free trial (`<name>-<id>.env.trial.teradata.com`, 20.00); VantageCloud itself listed as not yet validated |

- Clients: Linux x64 (glibc 2.28+), Windows x64 (self-contained DLL), NuGet `ArpeTeradata.AdbcDriver`
  (native only). No macOS, no Linux arm64. OpenSSL linked in; no Teradata file shipped.
- TTU (CLIv2 **17.10+** and TeraGSS; 20.00 tested) is needed **only** for the legacy port.
- The logon runs at connect (`dbapi.connect()`): host, TLS, TTU and credential errors come from it.
- Session: charset `UTF8`, `DECIMAL` up to 38 digits, time zone forced to **UTC**. Not supported: LDAP
  over TLS, client certificates, `ExecutePartitions`, Substrait, `GRAPHIC` columns.

## Connecting

A discrete option always wins over the same field from a URI or connection string, whatever the order.
Setting both `uri` and `connection_string`, an unknown key, a field given twice or a malformed value
fails `AdbcDatabaseInit` (`INVALID_ARGUMENT`). `server` is required, `username` too for TD2.

**URI** (`uri`, alias `adbc.arpeteradata.uri`): `teradata://[user[:password]@]host[:port][/database][?key=value&…]`
(also `arpeteradata://`, `teradatasql://`; case-insensitive). Path = default database. `port` = legacy
port (default **1025**); the TLS port is the `https_port` key (default **443**). IPv6 in brackets.
Percent-encode `@ : / ? # & % +` in the password. Query keys = the option names below without prefix
(`username`, `sslmode`, `logmech`…), plus `user`, `dbs_port`, `logon_timeout`, `buffer_size`.
teradatasql's `encryptdata`, `connect_timeout`, `request_timeout` are refused.

```text
teradata://alice:<pw>@td.example.com/SALES?sslmode=verify-full&sslca=/etc/ssl/td-ca.pem
```

**Connection string** (`adbc.arpeteradata.connection_string`, alias `connection_string`, ADO.NET): keys
case-insensitive, spaces / underscores ignored; quote values with `'…'` or `"…"`.

| Keys | Field |
|---|---|
| `Server`, `Host`, `Hostname`, `Data Source`, `Address` | server (host only, no `host,port` form) |
| `Port`, `DBS Port`, `Port Number` / `HTTPS Port` | legacy / TLS port |
| `Database`, `Initial Catalog` | database |
| `User ID`, `User`, `Username`, `UID` / `Password`, `PWD` | login |
| `Charset`, `TMode`, `Session Mode`, `Batch Size`, `Buffer Size` | session, rows per batch |
| `Login Timeout`, `Logon Timeout`, `Connect Timeout`, `Connection Timeout`, `Query Timeout` | timeouts (whole seconds) |
| `sslmode`, `sslca`, `sslcapath`, `TTU Path` | TLS / TTU |
| `LogMech`, `Authentication Mechanism` / `LogData`, `Authentication String` / `Kerberos SPN` | logon mechanism |

```text
Data Source=td.example.com;Database=SALES;User ID=alice;Password='<pw>';sslmode=verify-full
```

## Options (database, `db_kwargs`)

| Key | Default | Notes |
|---|---|---|
| `adbc.arpeteradata.server` | – | Required |
| `adbc.arpeteradata.port` / `.https_port` | **1025** / **443** | Legacy DBS port / TLS (HTTPS) port |
| `adbc.arpeteradata.database` (alias `adbc.connection.catalog`) | user's own database | `DATABASE <name>` after logon; unknown name fails the connect |
| `adbc.arpeteradata.username` / `.password` (aliases `username` / `password`) | – | Any character allowed in the password |
| `adbc.arpeteradata.logmech` | `TD2` | `TD2`, `LDAP`, `KRB5`, `TDNEGO`, `JWT` (case-insensitive) |
| `adbc.arpeteradata.logdata` | – | `token=<JWT>`, or `authcid=<user> password=<pw>` for LDAP; ≤ 65535 bytes; never returned |
| `adbc.arpeteradata.kerberos_spn` | `TERADATA/<server>` | Set when connecting by IP or DNS alias |
| `adbc.arpeteradata.sslmode` | `prefer` | `disable`, `allow`, `prefer`, `require`, `verify-ca`, `verify-full` (case-insensitive) |
| `adbc.arpeteradata.sslca` / `.sslcapath` | system CAs | PEM file / directory; ≤ 1023 bytes |
| `adbc.arpeteradata.ttu_path` | search order | Directory of `libcliv2.so` / `wincli32.dll`; loads code: no group/world-writable dirs |
| `adbc.arpeteradata.charset` | `UTF8` | Any other value refused |
| `adbc.arpeteradata.tmode` | server default (Teradata) | `TERA` / `TERADATA` or `ANSI`; other values = server default |
| `adbc.arpeteradata.login_timeout` | **60** (also `0`) | Per logon attempt; with `prefer` / `allow` the TLS try is capped at 5 s; DNS not covered |
| `adbc.arpeteradata.query_timeout` | `0` (none) | **Works**: socket read timeout after logon; on expiry the call fails and the connection is closed |
| `adbc.arpeteradata.batch_size` (alias `.buffer_size`) | `32000` | Rows per Arrow batch; a batch also closes at 256 MiB |
| `arpeio.adbc.license`, `arpeio.adbc.license_file` | – | Licence (also `ARPEIO_ADBC_LICENCE[_FILE]`, `arpeio_adbc.lic` next to the library) |

- `adbc.connection.autocommit`: `true` (default) / `false` only, anything else `INVALID_ARGUMENT`.
  Python dbapi turns it **off** by default: pass `autocommit=True` or `conn.commit()`.
- `adbc.connection.db_schema`: get / set the session's default database (`DATABASE <name>`, unknown
  → `NOT_FOUND`). No catalog level.
- Statement options: `adbc.arpeteradata.batch_size` (wins over the database value), the
  `adbc.ingest.*` keys, `adbc.arpeteradata.ingest.varchar_length`.
- Env: `ARPETERADATA_TTU_PATH`, `ARPETERADATA_GSSAPI_LIBRARY`, `ARPETERADATA_NO_TLS_WARNING=1`.

Transactions: Teradata mode uses `BT` / `ET` / `ABORT`; a failed request rolls back the whole open
transaction (autocommit stays off). ANSI mode uses `COMMIT` / `ROLLBACK` (with autocommit on the
driver commits after each statement). Don't send `BT`, `ET`, `COMMIT`… as SQL with autocommit off. With
autocommit on, a bound execute split into several requests (> 16000 rows or ~1 MB) is one
transaction. Closing rolls back work that may have written. Cancel works on Linux and Windows
(`HY008`, connection stays usable).

## Authentication

| `logmech` | Credentials | Over TLS, no TTU | Legacy port (TTU) |
|---|---|---|---|
| `TD2` (default) | `username` + `password` | Yes (Linux, Windows) | Yes |
| `LDAP` | directory user / password, or `logdata` | **No** (`NOT_IMPLEMENTED` with `require` / `verify-*`; `prefer` / `allow` go straight to the legacy port) | Yes |
| `JWT` | `logdata=token=<JWT>`, no username (`sub` names the user) | Yes | Yes |
| `KRB5`, `TDNEGO` | none: process's Kerberos ticket | Linux: MIT `libgssapi_krb5.so.2` (validated); Windows: SSPI, domain user (built, not yet validated against a DC) | Yes, through TTU |

- Kerberos: `kinit` first; SPN `TERADATA/<server>` as given (no DNS lookup); errors never fall back.
  Legacy port on Debian / Ubuntu: symlink `libgssapi_krb5.so.2` into `/usr/lib64` (else CLIv2 error 507).
- LDAP, JWT and Kerberos users need `GRANT LOGON ON ALL TO <user> WITH NULL PASSWORD;` (else 8017),
  and the mechanism configured in the server's TeraGSS.
- TTU lookup: `ttu_path`, `ARPETERADATA_TTU_PATH`, the driver's dir, then Linux `/opt/teradata/client/…/lib64`
  and the library path (a folder with `libcliv2.so` is enough), Windows the registry (install TTU with CLIv2
  + TeraGSS) and `%ProgramFiles%\Teradata\Client\<ver>\bin`. Loaded once per process.

## TLS

| `sslmode` | Tries | Falls back | Verifies |
|---|---|---|---|
| `disable` | legacy port (TTU) | – | – (session in clear) |
| `allow` | legacy port | to TLS | nothing |
| `prefer` (default) | TLS | to the legacy port (TTU) | nothing |
| `require` | TLS only | no | nothing |
| `verify-ca` | TLS only | no | chain |
| `verify-full` | TLS only | no | chain + host name (or IP) |

- TLS 1.2 / 1.3. A rejected logon or failed certificate check never falls back. With `allow`, `prefer`,
  `require` stderr gets once `WARNING: ArpeTeradata: TLS logon without server certificate verification …`.
- Trust store without `sslca` / `sslcapath`: Linux uses OpenSSL defaults, then the first distribution
  bundle found (`/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem`, `/etc/pki/tls/certs/ca-bundle.crt`,
  `/etc/ssl/certs/ca-certificates.crt`, `/etc/ssl/ca-bundle.pem`, `/etc/ssl/cert.pem`);
  `SSL_CERT_FILE` / `SSL_CERT_DIR` replace that search. **Windows reads the ROOT certificate store.**
  An internal CA goes in `sslca`.
- Legacy port: credentials protected by TeraGSS, but query text and results travel in clear.
- Cloud / trial: `sslmode=require` on 443, no TTU (`prefer` could fall back to the clear port 1025). The
  trial certificate's CA is not shipped anywhere, so `verify-*` fail (`unable to get local issuer
  certificate`). Resume a suspended environment first; trial user `demo_user` (database `DEMO_USER`).

## Types

Read (Teradata → Arrow): `BYTEINT` / `SMALLINT` / `INTEGER` / `BIGINT` → int8/16/32/64; `FLOAT` /
`REAL` / `DOUBLE PRECISION` → float64; `DECIMAL` / `NUMERIC` / `NUMBER(p,s)` p ≤ 38 → decimal128(p,s);
unbounded `NUMBER` → utf8 (lossless text); `CHAR` (blanks trimmed), `VARCHAR`, `LONG VARCHAR` → utf8;
`BYTE`, `VARBYTE` → binary; `DATE` → date32; `TIME` → time64[us]; `TIMESTAMP` → timestamp[us];
`TIMESTAMP WITH TIME ZONE` → timestamp[us, UTC]; `TIME WITH TIME ZONE`, `INTERVAL` → utf8; `PERIOD` →
struct<begin, end>; `ARRAY` → (nested) list; `VECTOR` / `VECTOR32` → list<float64 / float32>;
distinct UDT → base type; structured UDT → utf8; `CLOB`, `XML`, `DATASET CSV` → large_string; `JSON`
→ large_string `arrow.json`; `BLOB`, `DATASET AVRO` → large_binary; `ST_Geometry` → large_string
`geoarrow.wkt`. Each field carries `ARPETERADATA:type` metadata. `GRAPHIC` family: refused by the server
(6705 / 6701): `TRANSLATE(col USING GRAPHIC_TO_UNICODE)`. A `VARCHAR` value exports at most 64000 bytes.

Bound parameters (`?` only): int8–64, float64, decimal32–256 (p ≤ 38), utf8, binary, date32, time,
timestamp (tz → `WITH TIME ZONE` at `+00:00`), null; also bool, uint8–64, float32, large_string /
large_binary, fixed_size_binary, date64, time32, ns units (truncated to µs). Refused: string or binary
values **over 64000 bytes**, p > 38, dictionary, list, struct; dates outside years 1–9999 (`value out of
range`). A bound `SELECT` must fit in one request (~1 MB).

Ingest `create` DDL: bool → BYTEINT; int8/16/32/64 → BYTEINT / SMALLINT / INTEGER / BIGINT; uint8/16/32
→ SMALLINT / INTEGER / BIGINT; uint64 → DECIMAL(20,0); float32/64 → FLOAT; decimal (p ≤ 38) →
DECIMAL(p,s); utf8 / large_utf8 → `VARCHAR(n) CHARACTER SET UNICODE` (n = `varchar_length`, default
4000, max 32000); binary → VARBYTE(2n) capped at 64000; fixed_size_binary(n) → BYTE(n); date32/64 →
DATE; time / timestamp s, ms, us, ns → (0), (3), (6), (6) (tz → `WITH TIME ZONE`); null →
VARCHAR(1). Non-nullable → `NOT NULL`. Refused (`NOT_IMPLEMENTED`): string_view, binary_view,
dictionary, list, struct, map, interval, duration.

## Ingest

Packed, parameterised `INSERT` requests (no FastLoad / TPT). Modes `create`, `append`, `replace`
(`DROP` then `CREATE`, not atomic), `create_append`; `temporary=True` → volatile table `ON COMMIT
PRESERVE ROWS` in your own database (not with `db_schema_name`). Created tables are `MULTISET`, `NO
PRIMARY INDEX`: for a PI, create the table yourself and `append`.

- Autocommit on: DDL on its own, then all rows in one transaction (failed load leaves the table empty).
- Autocommit off: `append` joins your transaction; `create` / `replace` / `create_append` need a
  transaction with no writes yet (else `INVALID_STATE`) and commit the DDL; then `conn.commit()`.
- **Strings longer than the column are truncated silently in Teradata mode** (refused, 3996, in ANSI):
  raise `adbc.arpeteradata.ingest.varchar_length` (statement option). Too wide a row: 3933 at `CREATE`.
- Names over 128 characters or containing `?` are refused.

## Example

```python
import os
import adbc_driver_manager.dbapi as dbapi
import pyarrow as pa

db_kwargs = {
    "adbc.arpeteradata.server": "td.example.com",
    "adbc.arpeteradata.database": "SALES",
    "adbc.arpeteradata.username": "alice",
    "adbc.arpeteradata.password": os.environ["TD_PASSWORD"],
    "adbc.arpeteradata.sslmode": "verify-full",
    "adbc.arpeteradata.sslca": "/etc/ssl/teradata-ca.pem",
}
with dbapi.connect(driver="arpeteradata", db_kwargs=db_kwargs, autocommit=True) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM SALES.ORDERS WHERE order_id = ?", parameters=[42])
        print(cur.fetch_arrow_table())
        cur.adbc_statement.set_options(**{"adbc.arpeteradata.ingest.varchar_length": "200"})
        cur.adbc_ingest("MY_TABLE", pa.table({"ID": [1, 2, 3], "NAME": ["a", "b", "c"]}),
                        mode="create_append")
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `logon requires adbc.arpeteradata.server and username` | Set both (no username needed for JWT / KRB5 / TDNEGO) |
| `uri: …` / `connection_string: …` at init | Named part is wrong; percent-encode the password; not both forms |
| `Teradata connection failed: TLS to <host>:443 unavailable (…); legacy port logon failed (…)` | `prefer` failed both ways: open 443, or install TTU and open 1025 |
| `Teradata TLS connection to <host>:443 failed: …` | `require` / `verify-*` never fall back: check 443, or `prefer` / `disable` (TTU) |
| `TLS certificate verification failed: unable to get local issuer certificate` | CA in `sslca`; Teradata trial: `sslmode=require` |
| `TTU CLIv2 not found (tried: …)` | Use TLS, or install TTU CLIv2 17.10+ / set `ttu_path`; `(skipped: …)` = writable dir |
| `logon mechanism LDAP is not supported over TLS (sslmode=…)` | `sslmode=disable` (TTU), or JWT / Kerberos |
| `Teradata logon failed: 8017 The UserId, Password or Account is invalid.` | Credentials; JWT / Kerberos users need `WITH NULL PASSWORD` grant |
| `Teradata logon failed: CLIv2 error 244: … EM_SSOLOGONFAIL …` | Legacy-port LDAP / JWT / Kerberos refused: password, token, user |
| `Kerberos: no service ticket for TERADATA/<server>: …` | `kinit`, `KRB5CCNAME`, or `kerberos_spn` when using an IP / alias |
| `Kerberos logon needs the GSS-API library of MIT Kerberos (libgssapi_krb5.so.2 …)` | Install `krb5-libs` / `libgssapi-krb5-2`, or `ARPETERADATA_GSSAPI_LIBRARY` |
| `Teradata logon timed out after N s` | Firewall dropping packets, wrong port, stopped (or suspended cloud) server |
| `CLIv2 has no free session slots (error 301)` | Close legacy-port connections or restart the process |
| `the connection was lost after an earlier error (…); open a new connection` | After an I/O error (e.g. `query_timeout`): reconnect |
| `…; Teradata rolled back the open transaction` | Teradata-mode failure: redo the work since the last commit |
| Writes gone after close | dbapi autocommit is off: `conn.commit()` or `autocommit=True` |
| 6705 / 6701 on `GRAPHIC` | `TRANSLATE(col USING GRAPHIC_TO_UNICODE)` |
| `unsupported column type code <n> for column <name>` | Cast in SQL |
| `value out of range for Teradata …` / 5404 | Date outside 1–9999 / fraction into `TIME(0)`, `TIMESTAMP(0)` |

Docs vs driver 0.3.5: the current-database connection option is `adbc.connection.db_schema` (ADBC
standard; docs: `adbc.connection.current_db_schema`, which the driver rejects as unknown); on close a
transaction that only read is committed (`ET` / `COMMIT`) and only one that may have written is rolled
back (docs: always rolls back); discrete `port` / `login_timeout` / `query_timeout` are not range-checked
(only URI / connection-string values are); the cloud page shows a validated trial connection while the
compatibility table lists VantageCloud as not yet validated.
