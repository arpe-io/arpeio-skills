# ArpePGSQL – PostgreSQL ADBC driver (0.4.7)

Load name `arpepgsql`. Native Frontend/Backend protocol v3 over TCP/TLS (no libpq). Reads through
`COPY … TO STDOUT (FORMAT binary)`, writes through `COPY … FROM STDIN (FORMAT binary)`. Options
`adbc.arpepgsql.*`.

## Targets

- PostgreSQL 12, 15, 16, 17 validated; 13, 14 supported. Managed services (RDS, Azure, Cloud SQL,
  Supabase, Neon…): as their PostgreSQL version, with `sslmode=verify-full` and the provider's CA.
- PG-wire engines without binary COPY (CockroachDB, Redshift, Materialize…): `copy_format=auto` falls
  back to the text path. Aurora, Greenplum, Yugabyte: not validated.
- Not supported: PG17 direct TLS (`sslnegotiation=direct`), client certificates, Entra ID, partitioned
  results.
- Every session sets `client_encoding=UTF8`, **`TimeZone=UTC`**, `IntervalStyle=postgres`.
- The session opens on first use: connect / auth errors appear on the first query as
  `native session open failed: …`.

## Connecting

A discrete option always wins. Between a URI and a connection string, the later one wins for a shared
field. Then env fallbacks fill empty fields: `PGUSER`, `PGPASSWORD`, password file, `PGSSLROOTCERT`
(only these: `PGHOST`, `PGPORT`, `PGDATABASE`, `PGSSLMODE` are **not** read).

**URI** (libpq form): `postgresql://[user[:password]@]host[:port][/dbname][?key=value&…]` (also
`postgres://`, `arpepgsql://`). Path = database. Parameters: `user`, `password`, `dbname`,
`application_name`, `connect_timeout`, `sslmode`, `sslrootcert`, `channel_binding`, `krbsrvname`,
`require_password_encryption`. Unknown or repeated parameter → error.

```text
postgresql://alice:p%40ss@db.example.com:5432/tpch?sslmode=verify-full&sslrootcert=/etc/ssl/pg-ca.pem
```

**Connection string** (`adbc.arpepgsql.connection_string`, ADO.NET / Npgsql): keywords
case-insensitive; quote values containing `;`.

| Keyword(s) | Field |
|---|---|
| `Server`, `Host`, `Data Source`, `Address` (`host`, `host:port`, `host,port`) / `Port` | target |
| `Database`, `Initial Catalog` | database |
| `User ID`, `UID`, `User`, `Username` / `Password`, `PWD` | login |
| `SSL Mode` / `SslMode` (`Disable`, `Prefer`, `Require`, `VerifyCA`, `VerifyFull`), `Root Certificate` / `SslRootCert`, `Channel Binding`, `Encrypt`, `TrustServerCertificate`, `RequirePasswordEncryption` | TLS |
| `Connection Timeout`, `Timeout`, `Application Name` | |
| `Integrated Security`, `Authenticator`, `Authentication` (`SqlPassword` only), `KrbSrvName`, `Service Principal Name` | auth |

```text
Host=db.example.com;Port=5432;Database=tpch;Username=alice;Password='p;w';SSL Mode=VerifyFull;Root Certificate=/etc/ssl/pg-ca.pem
```

## Options

In `db_kwargs` always use the full `adbc.arpepgsql.*` keys (bare `sslmode`, `encrypt`… are
connection-only and fail as database options with `Unknown database option`).

| Key | Default | Notes |
|---|---|---|
| `adbc.arpepgsql.server` / `.port` / `.database` | `localhost` / `5432` / role's default | |
| `adbc.arpepgsql.username` / `.password` | `PGUSER` / `PGPASSWORD`, password file | A role is required |
| `adbc.arpepgsql.application_name` | not sent | `pg_stat_activity` |
| `adbc.arpepgsql.auth_type` | `sql` | `integrated` / `sspi` (Windows only) |
| `adbc.arpepgsql.trusted` | `false` | `true` = integrated |
| `adbc.arpepgsql.krb5.spn` / `.krbsrvname` | `POSTGRES/<server>` / `POSTGRES` | Windows SSPI |
| `adbc.arpepgsql.sslmode` | `prefer` | `disable`, `prefer`, `require`, `verify-ca`, `verify-full` (**lower case exactly** here) |
| `adbc.arpepgsql.ssl_root_cert` | `PGSSLROOTCERT`, else OpenSSL defaults | PEM file. With `require` it turns on chain verification (0.4.7) |
| `adbc.arpepgsql.channel_binding` | `prefer` | `require` = only SCRAM-SHA-256-PLUS over TLS |
| `adbc.arpepgsql.require_password_encryption` | `false` | Never send a cleartext/MD5 password without TLS |
| `adbc.arpepgsql.encrypt` / `.trust_server_cert` | `false` | Used only if no `sslmode`: `encrypt=true` → `verify-full` (or `require` with `trust_server_cert=true`) |
| `adbc.arpepgsql.login_timeout` | `30` | Seconds (`0` = 30) |
| `adbc.arpepgsql.socket_timeout` | `0` (none) | Per read/write, also TLS handshake and login |
| `adbc.arpepgsql.query_timeout` | `0` | **Accepted but not applied**: use `socket_timeout`, `SET statement_timeout`, or cancel |
| `adbc.arpepgsql.buffer_size` | `32000` | Rows per Arrow batch (1..10,000,000) |
| `adbc.arpepgsql.copy_format` | `auto` | `binary`, `text` (env `ARPEIO_ADBC_COPY_FORMAT` when unset) |
| `adbc.arpepgsql.numeric_format` | `decimal` | `string` = every NUMERIC as text (lossless, keeps NaN / Infinity) |
| `adbc.arpepgsql.numeric_special` | `error` | `null` = NaN / ±Infinity in a decimal column read as NULL |
| `adbc.arpepgsql.uuid_casing` | `lower` | `upper` |
| `adbc.arpepgsql.geospatial` | `geoarrow.wkb` | `wkb`, `binary` (PostGIS geometry / geography) |
| `arpeio.adbc.license`, `arpeio.adbc.license_file` | – | Licence |

Connection options: `adbc.connection.autocommit` (`true` / `false` only), `adbc.connection.catalog`
(**reconnects**: temp tables and session settings are lost; not inside a transaction),
`adbc.connection.db_schema` (`SET search_path`). No isolation-level option: use SQL.

Statement options: `adbc.arpepgsql.batch_size`, `adbc.arpepgsql.max_batch_size` (1,000,000),
`arpepgsql.bulk_batch_size` (rows per COPY chunk, chunks also flush at ~4 MiB). No effect:
`memory_budget_mb`, `prefetch`, `arpepgsql.bulk_tablock` / `keep_identity` / `check_constraints` /
`fire_triggers` / `keep_nulls` (COPY always fires triggers, checks constraints, keeps NULLs).

## Authentication

| Mode | Options | Notes |
|---|---|---|
| SCRAM-SHA-256 (and -PLUS over TLS) | `username` + `password` | Chosen by the server; `channel_binding=require` to insist on -PLUS |
| MD5, cleartext password | `username` + `password` | Use TLS or `require_password_encryption=true` |
| Windows integrated (SSPI: Kerberos, NTLM fallback) | `auth_type=integrated`, `username` = PostgreSQL role, no password | **Windows only**; answers `gss` / `sspi` pg_hba lines; connect by FQDN |
| Integrated on Linux | – | Not available in the released binary ("Trusted/integrated authentication is unavailable …") |

`.pgpass`: `PGPASSFILE`, else `~/.pgpass` (Windows `%APPDATA%\postgresql\pgpass.conf`); format
`host:port:database:user:password`, `*` wildcards; ignored if group/world readable (`chmod 0600`).

TLS by `sslmode`:

| `sslmode` | Encrypted | Chain checked | Host checked |
|---|---|---|---|
| `disable` | no | – | – |
| `prefer` (default) | if the server accepts, else plaintext with a stderr warning | no | no |
| `require` | yes | only if `ssl_root_cert` / `PGSSLROOTCERT` is set (0.4.7) | no |
| `verify-ca` | yes | yes | no |
| `verify-full` | yes | yes | yes |

The driver never reads the OS certificate store: on RHEL-family and Windows give `ssl_root_cert`
(RHEL: `/etc/pki/tls/certs/ca-bundle.crt`) or `SSL_CERT_FILE`. TLS 1.2 minimum. `~/.postgresql/root.crt`
is not read.

## Types

Read (PostgreSQL → Arrow): `bool`; `int2/4/8` → int16/32/64; `oid` → uint32; `float4/8` →
float32/64; `numeric(p,s)` → decimal128 (p ≤ 38) / decimal256 (p ≤ 76), unconstrained or larger →
utf8; text types, `json`, `jsonb`, `xml`, `inet`, `cidr`, `macaddr`, `bit`, `timetz` → utf8; `uuid` →
utf8; `bytea` → binary; `date` → date32; `time(p)` / `timestamp(p)` → s (p=0), ms (p 1–3), else us;
`timestamptz` → timestamp tz UTC; `interval` → interval[month_day_nano]; PostGIS → EWKB with
`geoarrow.wkb`; arrays, ranges, enums, money, composites → binary (raw wire bytes): cast them in SQL
(`col::text`) for readable values.

Ingest `create` DDL: bool → boolean; int8/uint8/int16 → smallint; uint16/int32 → integer;
uint32/int64 → bigint; uint64 → numeric(20,0); float32/64 → real / double precision; utf8 → text;
binary → bytea; decimal → numeric(p,s); date → date; time → time(0/3/6); timestamp → timestamp(0/3/6)
(tz → timestamptz); interval → interval. Refused: duration, list, struct, map, dictionary, null.
Nanoseconds are truncated to microseconds.

- `append` into an existing permanent table maps utf8 into `json`, `jsonb`, `inet`, `uuid`… columns;
  not for temporary tables or modes that run DDL.
- COPY uses the Arrow column names as a quoted column list: names must match exactly.

## Ingest (COPY binary)

`cur.adbc_ingest("orders", data, mode=…, db_schema_name="staging", temporary=False)`; `temporary=True`
→ `pg_temp."orders"`. Without a schema: `current_schema()`. `adbc.ingest.target_catalog` is ignored
(switch database with `adbc.connection.catalog`).

Since 0.4.7 ingest runs **on the connection's own session**:
- autocommit off (dbapi default): DDL + COPY join the transaction → `conn.commit()`;
- autocommit on: DDL + COPY in one transaction (a failed `replace` keeps the old table).

Parallel loads: one connection per loader.

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
| `Unknown database option: sslmode` | Use `adbc.arpepgsql.sslmode` in `db_kwargs` |
| `native session open failed: connect(…) failed` / timeout | Host / port / firewall |
| `FATAL: password authentication failed` | Credentials, stale `.pgpass` line, pg_hba |
| `WARNING: password file … has group or world access … ignored` | `chmod 0600 ~/.pgpass` |
| `no pg_hba.conf entry for host …, SSL off` | `sslmode=require` or stronger |
| `TLS handshake failed: … unable to get local issuer certificate` | `ssl_root_cert` (RHEL / Windows) |
| `sslmode=require` fails after upgrading to 0.4.7 | A root cert is set (option or `PGSSLROOTCERT`) and doesn't match: fix it or unset `PGSSLROOTCERT` |
| stderr `server declined TLS; continuing on an UNENCRYPTED connection` | `sslmode=verify-full` (or `require`) |
| `channel_binding=require, but the connection cannot use SCRAM-SHA-256-PLUS` | Enable TLS + `scram-sha-256` in pg_hba, or `channel_binding=prefer` |
| `server requires GSS/SSPI authentication; set auth_mode=integrated` | Windows: `auth_type=integrated`; Linux: password pg_hba entry |
| `… a result set is still open on this connection …` | Read / close the previous result first |
| `current transaction is aborted` (25P02) | `conn.rollback()` |
| `NUMERIC NaN/Infinity cannot be represented as decimal128` | `numeric_format=string` or `numeric_special=null` |
| `'infinity' … has no Arrow timestamp value` | `NULLIF(col, 'infinity')` in SQL |
| `server does not implement binary COPY` | `copy_format=auto` or `text` |
| Ingest `ALREADY_EXISTS` | `append` / `replace` / `create_append`; on append, Arrow names / types don't match the table |
| `No PostgreSQL type mapping for Arrow format …` | Cast the column before ingest |
| Ingested rows disappear | `conn.commit()` |
| `query_timeout` ignored / query hangs | `socket_timeout`, `SET statement_timeout`, or `cur.adbc_cancel()` |

Docs (0.4.6) vs driver 0.4.7: `require` + root cert now verifies the chain; no SNI for IP addresses;
ingest joins the connection's transaction (docs: commits on its own); `query_timeout`,
`memory_budget_mb` and `adbc.ingest.target_catalog` have no effect; `adbc.ingest.mode` must be set
(Python does it); nanosecond values are truncated, not rejected; `oid` → uint32; `login_timeout=0`
means 30 s; invalid boolean / integer values are now rejected.
