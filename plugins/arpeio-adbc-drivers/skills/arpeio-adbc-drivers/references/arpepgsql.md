# ArpePGSQL – PostgreSQL ADBC driver (0.5.2)

Load name `arpepgsql`. Native Frontend/Backend protocol v3 over TCP/TLS (no libpq). Reads through
`COPY … TO STDOUT (FORMAT binary)`, writes through `COPY … FROM STDIN (FORMAT binary)`. Options
`adbc.arpepgsql.*`.

## Targets

- PostgreSQL 12, 15, 16, 17 validated; 13, 14 supported; 18 supported (type formats checked with 0.5.2,
  not in the continuous test matrix). Managed services (RDS, Azure, Cloud SQL, Supabase, Neon…): as
  their PostgreSQL version, with `sslmode=verify-full` and the provider's CA.
- PgBouncer: 1.15+ with default settings, session and transaction pools (0.5.2). 0.5.0 / 0.5.1 need
  PgBouncer 1.20+; 0.4.8 to 0.4.10 also need `extra_float_digits` in `ignore_startup_parameters`.
- Extensions validated: PostGIS (read and ingest), pgvector (`vector` / `halfvec` 0.4.8+, `sparsevec`
  0.5.1+).
- PG-wire engines without binary COPY (CockroachDB, Redshift, Materialize…): `copy_format=auto` falls
  back to the text path. Aurora, Greenplum, Yugabyte: not validated.
- Not supported: PG17 direct TLS (`sslnegotiation=direct`), client certificates, Entra ID.
- Session: `DateStyle=ISO` pinned at connect (0.4.8+); `client_encoding` must stay `UTF8` (a `SET`
  that changes it closes the connection, next statement reconnects). 0.5.2 no longer sends
  `IntervalStyle`; any interval style reads correctly.
- Login happens at the first statement: a refused login / unknown database raises on the first
  `execute()` (Python `ProgrammingError` with SQLSTATE `28P01` / `3D000` since 0.4.8, `OperationalError`
  before).

## Connecting

A discrete option always wins. Between a URI and a connection string, the later one wins for a shared
field. Then fallbacks fill empty fields: `PGUSER`, `PGPASSWORD`, password file, `PGSSLROOTCERT`,
`~/.postgresql/root.crt` (only for `verify-ca` / `verify-full`, 0.4.8+). `PGHOST`, `PGPORT`,
`PGDATABASE`, `PGSSLMODE` are **not** read.

**URI** (libpq form): `postgresql://[user[:password]@]host[:port][/dbname][?key=value&…]` (also
`postgres://`, `arpepgsql://`). Path = database. Parameters: `user`, `password`, `dbname`,
`application_name`, `connect_timeout`, `sslmode`, `sslrootcert`, `channel_binding`, `gssencmode`,
`krbsrvname`, `require_password_encryption`, `require_kerberos` (0.5.2). Unknown or repeated parameter
→ error. No URI form for `krb5.*`, type options, timeouts other than `connect_timeout`.

```text
postgresql://alice:p%40ss@db.example.com:5432/tpch?sslmode=verify-full&sslrootcert=/etc/ssl/pg-ca.pem
```

**Connection string** (`adbc.arpepgsql.connection_string`, ADO.NET / Npgsql): keywords
case-insensitive; quote values containing `;`; unknown keywords rejected.

| Keyword(s) | Field |
|---|---|
| `Server`, `Host`, `Data Source`, `Address` (`host`, `host:port`, `host,port`, `tcp:` prefix) / `Port` | target; a port in both must be equal (0.4.8+) |
| `Database`, `Initial Catalog` | database |
| `User ID`, `UID`, `User`, `Username` / `Password`, `PWD` | login |
| `SSL Mode` / `SslMode` (`Disable`, `Prefer`, `Require`, `VerifyCA`, `VerifyFull`), `Root Certificate` / `SslRootCert`, `Channel Binding`, `Encrypt`, `TrustServerCertificate`, `RequirePasswordEncryption` | TLS |
| `Connection Timeout`, `Timeout`, `Application Name` | |
| `Integrated Security`, `Authenticator`, `Authentication` (`SqlPassword` only), `KrbSrvName`, `Service Principal Name`, `Krb5 Keytab File`, `Krb5 Credential Cache`, `Krb5 Principal`, `Krb5 Password`, `GssEncMode`, `RequireKerberos` (0.5.2) | auth |

```text
Host=db.example.com;Port=5432;Database=tpch;Username=alice;Password='p;w';SSL Mode=VerifyFull;Root Certificate=/etc/ssl/pg-ca.pem
```

## Options

Use the full `adbc.arpepgsql.*` keys in `db_kwargs`. Since 0.4.8 the bare aliases (`sslmode`,
`krb5_spn`…) work at both database and connection level; up to 0.4.7 several (`sslmode`,
`ssl_root_cert`, `encrypt`, `krb5_spn`…) failed as database options with `Unknown database option`.
Booleans take `true`/`yes`/`1`/`false`/`no`/`0` (any case); anything else is `INVALID_ARGUMENT` (0.4.8+).

| Key | Default | Notes |
|---|---|---|
| `adbc.arpepgsql.server` / `.port` / `.database` | `localhost` / `5432` / role's default | |
| `adbc.arpepgsql.username` / `.password` | `PGUSER` / `PGPASSWORD`, password file | A role is required |
| `adbc.arpepgsql.application_name` | not sent | `pg_stat_activity` |
| `adbc.arpepgsql.auth_type` | `sql` | `integrated` / `sspi` / `trusted`: Kerberos on Linux (0.4.8+), SSPI on Windows |
| `adbc.arpepgsql.trusted` | `false` | `true` = integrated (database level only) |
| `adbc.arpepgsql.krb5.ccache` / `.keytab` | ambient cache (`kinit`, `KRB5CCNAME`) | Linux Kerberos (0.4.8+) |
| `adbc.arpepgsql.krb5.principal` / `.password` | – | Linux: driver gets the ticket itself; `principal` also picks a keytab entry (0.4.8+) |
| `adbc.arpepgsql.krb5.spn` / `.krbsrvname` | `postgres/<server>` (Linux), `POSTGRES/<server>` (Windows) | Full SPN override / service part only |
| `adbc.arpepgsql.require_kerberos` | `false` | Windows: refuse a login that fell back to NTLM. No effect on Linux (0.5.2) |
| `adbc.arpepgsql.gssencmode` | `disable` | `prefer`, `require`: GSS transport encryption, Linux only (0.4.8+). Default differs from libpq |
| `adbc.arpepgsql.sslmode` | `prefer` | `disable`, `prefer`, `require`, `verify-ca`, `verify-full` (**lower case exactly** here) |
| `adbc.arpepgsql.ssl_root_cert` | `PGSSLROOTCERT`, `~/.postgresql/root.crt`, else OS store (`verify-full` only) | PEM file. With `require` it turns on chain verification (0.4.7+) |
| `adbc.arpepgsql.channel_binding` | `prefer` | `require` = only SCRAM-SHA-256-PLUS over TLS; every other method refused |
| `adbc.arpepgsql.require_password_encryption` | `false` | Never send a cleartext/MD5 password without encryption |
| `adbc.arpepgsql.encrypt` / `.trust_server_cert` | `false` | With no `sslmode`: `encrypt=true` → `verify-full` (`require` with `trust_server_cert=true`). With an `sslmode` they must agree (0.4.8+) |
| `adbc.arpepgsql.login_timeout` | `30` | Seconds, whole login (TCP + TLS/GSS + auth) since 0.4.8; `0` = no limit (was 30 before 0.4.8) |
| `adbc.arpepgsql.socket_timeout` | `0` (none) | Per read/write |
| `adbc.arpepgsql.query_timeout` | `0` | Seconds; sent as `statement_timeout`, server cancels (0.4.8+; ignored before). See below |
| `adbc.arpepgsql.buffer_size` | `32000` | Rows per Arrow batch (1..10,000,000); `0`, negative or `10abc` rejected |
| `adbc.arpepgsql.copy_format` | `auto` | `binary`, `text` (env `ARPEIO_ADBC_COPY_FORMAT` when unset) |
| `adbc.arpepgsql.numeric_format` | `decimal` | `string` = every NUMERIC as text (lossless, keeps NaN / Infinity) |
| `adbc.arpepgsql.numeric_special` | `error` | `null` = NaN / ±Infinity in a decimal column read as NULL |
| `adbc.arpepgsql.array_lower_bound` | `error` | `ignore` = read arrays with lower bound ≠ 1 as plain lists (0.5.1+) |
| `adbc.arpepgsql.json_extension` | `off` | `arrow.json` tags `json` / `jsonb` (still utf8) (0.5.1+) |
| `adbc.arpepgsql.uuid_casing` | `lower` | `upper` |
| `adbc.arpepgsql.geospatial` | `geoarrow.wkb` | ISO WKB + GeoArrow metadata (0.4.8+; EWKB before). `wkb`, `binary` = raw EWKB |
| `arpeio.adbc.license`, `arpeio.adbc.license_file` | – | Licence (≤ 8191 / ≤ 1023 bytes) |

Read-only (0.4.10+): connection `adbc.arpepgsql.tls.version` / `.tls.group` / `.tls.cipher` (empty
without TLS or before the first statement), database `adbc.arpepgsql.openssl_version` (OpenSSL 3.5.9 in
0.5.2). Env `ARPEIO_ADBC_LOG_TLS=1` prints one stderr line per handshake.

**`query_timeout`** (0.4.8+): covers every statement incl. ingest and metadata, and time spent reading
rows. Timeout → `ADBC_STATUS_TIMEOUT`, SQLSTATE `57014`, connection kept (inside a transaction:
roll back). Settable on an open connection; `0` sends nothing (server / role `statement_timeout`
applies). Capped at 2147483 s.

Server, database, username, password > 255 bytes or `ssl_root_cert` path > 1023 bytes → rejected
("… is too long"), also from `PGUSER` / `PGPASSWORD` / `PGSSLROOTCERT`.

Connection options: `adbc.connection.autocommit` (`true` / `false` / `1` / `0` only),
`adbc.connection.catalog` (**reconnects**: temp tables and session settings are lost; not inside a
transaction), `adbc.connection.db_schema` (`SET search_path`). No isolation-level option: use SQL.
`numeric_format`, `numeric_special`, `array_lower_bound`, `query_timeout` can change on an open
connection.

Statement options: `adbc.arpepgsql.batch_size`, `adbc.arpepgsql.max_batch_size` (1,000,000),
`arpepgsql.bulk_batch_size` (rows per COPY chunk, chunks also flush at ~4 MiB). No effect:
`memory_budget_mb`, `prefetch`, `arpepgsql.bulk_tablock` / `keep_identity` / `check_constraints` /
`fire_triggers` / `keep_nulls` (booleans still validated; COPY always fires triggers, checks
constraints, keeps NULLs). Integer options reject trailing text.

## Authentication

| Mode | Options | Notes |
|---|---|---|
| SCRAM-SHA-256 (and -PLUS over TLS) | `username` + `password` | Chosen by the server; `channel_binding=require` to insist on -PLUS |
| MD5, cleartext password | `username` + `password` | Use TLS or `require_password_encryption=true` |
| Kerberos, Linux (0.4.8+) | `auth_type=integrated`, `username` = role, no password; optional `krb5.ccache` / `krb5.keytab` / `krb5.principal` + `krb5.password` | Answers `gss` pg_hba lines. Needs MIT Kerberos on the host (`krb5-libs` / `libgssapi-krb5-2`), loaded at first Kerberos login only |
| Windows integrated (SSPI) | `auth_type=integrated`, `username` = role, no password | `gss` line → Kerberos package, `sspi` line → Negotiate (NTLM fallback, 0.4.8+); `require_kerberos=true` refuses NTLM (0.5.2). Connect by FQDN |

Mutual authentication always enforced; `PGPASSWORD` / password file not read with integrated auth.
Up to 0.4.7 Linux had no integrated auth ("Trusted/integrated authentication is unavailable").

`gssencmode` (Linux): `require` replaces TLS, so it cannot combine with `sslmode=require` /
`verify-ca` / `verify-full`, and channel binding is unavailable. `prefer` falls back to TLS / plain.
On Windows `require` fails and `prefer` does nothing.

`.pgpass`: `PGPASSFILE`, else `~/.pgpass` (Windows `%APPDATA%\postgresql\pgpass.conf`); format
`host:port:database:user:password`, `*` wildcards; ignored if group/world readable (`chmod 0600`).

TLS by `sslmode`:

| `sslmode` | Encrypted | Chain checked | Host checked |
|---|---|---|---|
| `disable` | no | – | – |
| `prefer` (default) | if the server accepts, else plaintext with a stderr warning | no | no |
| `require` | yes | only if `ssl_root_cert` / `PGSSLROOTCERT` is set | no |
| `verify-ca` | yes | yes; a root cert is **required** (0.4.8+) | no |
| `verify-full` | yes | yes; root cert, else OS CA store (0.4.8+) | yes |

CA source order for `verify-ca` / `verify-full` (0.4.8+): `ssl_root_cert` → `PGSSLROOTCERT` →
`~/.postgresql/root.crt` (Windows `%APPDATA%\postgresql\root.crt`) → OS store (`verify-full` only:
Linux distro CA bundle, or `SSL_CERT_FILE` / `SSL_CERT_DIR`; Windows `ROOT` store). Up to 0.4.7 no OS
store was read (RHEL / Windows needed `ssl_root_cert`). Private CA: always `ssl_root_cert`.
`sslrootcert=system` is a file name. `encrypt` / `trust_server_cert` contradicting `sslmode` (e.g.
`Encrypt=yes;SslMode=disable`) → `Conflicting TLS settings` (0.4.8+). TLS 1.2 minimum;
`X25519MLKEM768` offered first (0.4.10+). No SNI for an IP address. With no server set, `localhost`
is checked and sent as SNI (0.4.8+).

## Types

Read (PostgreSQL → Arrow): `bool`; `int2/4/8` → int16/32/64; `oid` → uint32; `float4/8` →
float32/64; `numeric(p,s)` → decimal128 (p ≤ 38) / decimal256 (p ≤ 76), unconstrained or larger →
utf8; `money` → decimal128(19, `lc_monetary` scale) (0.5.1+); `bytea` → binary; `date` → date32;
`time(p)` / `timestamp(p)` → s (p=0), ms (p 1–3), else us; `timestamptz` → timestamp tz UTC;
`interval` → interval[month_day_nano]; `timetz` → utf8; PostGIS → `geoarrow.wkb` (ISO WKB);
`vector` / `halfvec` → list<float32 / float16> (0.4.8+); `sparsevec` → struct<dim, indices (0-based),
values> (0.5.1+).

- utf8: text types, `uuid`, `json`, `jsonb`, `xml`, `inet`, `cidr`, `macaddr`, `bit`; since 0.5.1
  also `jsonpath`, `macaddr8`, `pg_lsn`, `"char"`, enums, `citext`, geometric types (`point`…),
  ranges, multiranges, composites, `tsvector`, extension types (`hstore`, `ltree`), `reg*`, `xid`,
  `tid`. Before 0.5.1 these (and arrays, `money`) read as binary wire bytes.
- Arrays → `list<T>` nested per dimension (0.5.1+); arrays of ranges / composites / domains → utf8.
  Depth is taken from the first non-empty value (read-ahead up to 1M rows / 64 MiB, 0.5.2); a later
  value of another depth, or a lower bound ≠ 1, fails: cast to `text` or `array_lower_bound=ignore`.
- `'infinity'` dates / timestamps fail the read (`column "valid_to": 'infinity' / '-infinity' has no
  Arrow date32 value …`): map them in SQL (`NULLIF(col, 'infinity')`).
- PostGIS default mode returns **ISO WKB** (0.4.8+): no per-value SRID; `crs` metadata only for a
  typmod column with SRID ≠ 0/4326. For SRIDs: cast to `geometry(Geometry, 2154)`, select
  `ST_SRID(geom)`, or `geospatial=wkb` (EWKB).
- The text path (`copy_format=text` / `auto` fallback) returns the same Arrow types; one statement per
  query text. Several statements in one string cannot carry a `money` column.

Ingest `create` DDL: bool → boolean; int8/uint8/int16 → smallint; uint16/int32 → integer;
uint32/int64 → bigint; uint64 → numeric(20,0); float32/64 → real / double precision; utf8 → text
(`arrow.json` field → jsonb); binary → bytea; decimal → numeric(p,s); date32 / date64 → date; time →
time(0/3/6); timestamp → timestamp(0/3/6) (tz → timestamptz); intervals → interval; list → `T[]`
(0.5.1+), but list<float32 / float16> → pgvector `vector` / `halfvec` (`(n)` for fixed_size_list);
sparsevec-shaped struct → `sparsevec`; dictionary → its value type (0.4.8+); null → text (0.4.8+);
`geoarrow.wkb` → geometry (`edges: spherical` → geography), SRID from `crs` (absent → 4326, `null` →
none), typmod with dimension (0.5.1+, SRID rules 0.5.2). Refused: duration, map, other structs.
Nanoseconds are truncated to microseconds.

Append into an existing table (permanent or temporary, 0.4.8+) introspects column types; field names
must match exactly (case-sensitive since 0.4.8). Conversions (0.4.8+):

- integer → any integer column (range-checked), `numeric`, float (only if exact); float32 → double; date → timestamp(tz);
  timestamp (no tz) → `date` only if midnight (0.4.9+); utf8 → text types, `json`, `jsonb`, `inet`,
  `cidr`, `macaddr`, `bit`, `uuid` (+ `macaddr8`, `pg_lsn`, enum, `citext` 0.5.1+); binary → bytea /
  uuid; decimal → `money`; ISO WKB / EWKB → geometry (takes the column's SRID).
- Anything that would change values (float64 → bigint, int64/utf8 → timestamp, tz timestamp → date…)
  → `ALREADY_EXISTS`, SQLSTATE `42804`, before COPY: cast the Arrow column first. Out-of-range value →
  `INVALID_ARGUMENT` naming column and row; that ingest stores nothing.

## Ingest (COPY binary)

`cur.adbc_ingest("orders", data, mode=…, db_schema_name="staging", temporary=False)`; `temporary=True`
→ `pg_temp."orders"` (cannot combine with a schema, 0.4.8+). Without a schema: `current_schema()`
(never a same-named temp table). `adbc.ingest.target_catalog` must name the connected database, else
`NOT_IMPLEMENTED` (0.4.8+).

Ingest runs **on the connection's own session**:
- autocommit off (dbapi default): DDL + COPY join the transaction → `conn.commit()`;
- autocommit on: DDL + COPY in one transaction (a failed `replace` keeps the old table).

C / Rust: always set `adbc.ingest.mode`; unset means `create` since 0.4.8 (it appended before).
Python's `adbc_ingest()` always sends a mode. Bind a batch or a stream. `executemany` pipelines up to
1024 rows per round trip (0.4.8+), same outcome as row by row.

## Errors and results

- One command at a time: read a result to the end or close it before the next statement. An early
  release cancels the rest (after ~250 ms for a plain read, ~2 s for anything that may write);
  inside a transaction a cancelled bound / text-path query aborts it → `rollback()`.
- `commit()` after a failed statement rolls back and fails `INVALID_STATE`, SQLSTATE `25P02` (0.4.8+).
- Release statements before the connection (`N statement(s) still open`, 0.4.8+).
- Server errors carry ADBC 1.1 details under libpq names (`PG_DIAG_MESSAGE_DETAIL`,
  `PG_DIAG_CONSTRAINT_NAME`, `PG_DIAG_TABLE_NAME`…), 0.4.8+. Python: `e.sqlstate`,
  `{k.decode(): v.decode() for k, v in e.details}`; fetch errors raise the matching DB-API class.

## Example

```python
import os
import adbc_driver_manager.dbapi as dbapi

conn = dbapi.connect(driver="arpepgsql", db_kwargs={
    "adbc.arpepgsql.server": "db.example.com",
    "adbc.arpepgsql.database": "tpch",
    "adbc.arpepgsql.username": "alice",
    "adbc.arpepgsql.password": os.environ["PG_PASSWORD"],
    "adbc.arpepgsql.sslmode": "verify-full",
    "adbc.arpepgsql.ssl_root_cert": "/etc/ssl/pg-ca.pem",
    "adbc.arpepgsql.numeric_format": "string",
})
with conn.cursor() as cur:
    cur.execute("SELECT * FROM orders WHERE o_orderkey = $1", parameters=[42])
    print(cur.fetch_arrow_table())
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Unknown database option: sslmode` | 0.4.7 or older: use `adbc.arpepgsql.sslmode` in `db_kwargs` |
| `Connection timed out` | Host / port / firewall |
| `login timed out after N s` | Server or pooler stalls in TLS / auth (LDAP, RADIUS); raise `login_timeout` |
| `FATAL: password authentication failed` | Credentials, stale `.pgpass` line, pg_hba |
| `WARNING: password file … has group or world access … ignored` | `chmod 0600 ~/.pgpass` |
| `no pg_hba.conf entry for host …, SSL off` | `sslmode=require` or stronger |
| `TLS handshake failed: … unable to get local issuer certificate` | `ssl_root_cert` (private CA; or 0.4.7 and older on RHEL / Windows) |
| `sslmode=verify-ca needs a root certificate …` | Set `ssl_root_cert`, or use `verify-full` |
| `Conflicting TLS settings: …` | Keep only `sslmode`, or make `encrypt` / `trust_server_cert` agree |
| stderr `server declined TLS; continuing on an UNENCRYPTED connection` | `sslmode=verify-full` (or `require`) |
| `channel_binding=require, but the connection cannot use SCRAM-SHA-256-PLUS` | Enable TLS + `scram-sha-256` in pg_hba (not with `gssencmode`), or `channel_binding=prefer` |
| `channel_binding=require, but the server requested authentication method …` | pg_hba uses `password` / `md5` / `gss` / `sspi` / `trust`: switch to `scram-sha-256`, or `channel_binding=prefer` |
| `unsupported startup parameter: extra_float_digits` (or `IntervalStyle`) via PgBouncer | Upgrade the driver to 0.5.2, or list it in `ignore_startup_parameters` |
| An option that used to load now fails `INVALID_ARGUMENT` | Strict values: booleans `true`/`false`/`yes`/`no`/`1`/`0`, integers without trailing text, `buffer_size` > 0 |
| `server requires GSS/SSPI authentication; set auth_mode=integrated` | `auth_type=integrated` (Linux: 0.4.8+ and a Kerberos ticket), or a password pg_hba entry |
| `Trusted/integrated authentication is unavailable` (Linux) | 0.4.7 or older: upgrade, or use a password |
| `… needs MIT Kerberos, which was not found` | Install `krb5-libs` (RHEL) / `libgssapi-krb5-2` (Debian, Ubuntu) |
| GSSAPI `Cannot find KDC` / `Server not found in Kerberos database` | `klist`; `krb5.ccache` / `krb5.keytab`; SPN `postgres/<server>` registered, or `krbsrvname` / `krb5.spn` |
| `Negotiate fell back to NTLM … require_kerberos is set` | Register `POSTGRES/<fqdn>` (`setspn -S`), connect by FQDN, or set `krbsrvname` / `krb5.spn` |
| `… a result set is still open on this connection …` | Read / close the previous result first |
| `current transaction is aborted` (25P02) / `commit()` "rolled back, not committed" | `conn.rollback()`, rerun the transaction |
| `query timed out: it ran longer than query_timeout` (57014) | Raise `query_timeout`; roll back if in a transaction |
| `NUMERIC NaN/Infinity cannot be represented as decimal128` | `numeric_format=string` or `numeric_special=null` |
| `column "…": 'infinity' / '-infinity' has no Arrow … value` | `NULLIF(col, 'infinity')` in SQL |
| `array with a lower bound other than 1` / `different number of dimensions` | Cast to `text`; `array_lower_bound=ignore` for the bound |
| Text path fails naming `DateStyle` / `client_encoding was changed` | `RESET DateStyle`; leave `client_encoding` as UTF8 |
| `query returned more than one result set` | Text path: one statement per query |
| `server does not implement binary COPY` | `copy_format=auto` or `text` |
| Ingest `ALREADY_EXISTS` | Table exists: `append` / `replace` / `create_append` (C/Rust: mode unset = `create` since 0.4.8) |
| `Arrow float64 values cannot be written to the existing bigint column` (42804) | Cast the Arrow column (`pyarrow.compute.cast`) or alter the table |
| `No PostgreSQL type mapping for Arrow format …` | Cast the column before ingest |
| Ingested rows disappear | `conn.commit()` |
| SIGSEGV in another thread after the loading thread exited | 0.4.7 or older: upgrade, or `ARPEIO_ADBC_ALLOCATOR=system` |

Docs vs driver: the docs say Arrow nanosecond values are rejected, the driver truncates them to
microseconds.
