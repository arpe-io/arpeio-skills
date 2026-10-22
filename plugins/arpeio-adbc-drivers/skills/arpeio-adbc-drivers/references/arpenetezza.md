# ArpeNetezza – IBM Netezza ADBC driver (0.3.7)

Load name **`arpenz`** (not `arpenetezza`). Native Netezza protocol (HSV2 handshake + DBOS binary
results) over TCP/TLS: no ODBC, no nzpy. Options `adbc.arpenz.*`.

## Targets

| Server | State |
|---|---|
| NPS 11.x (IBM Cloud) | Validated, TLS 1.2 / 1.3 |
| NPS 7.x | Validated **unencrypted** only; it offers TLS 1.0 / 1.1 only (`tls_min_version=1.0`, not validated) |
| Other NPS versions | Not validated |

- Windows DLL: not yet validated against a server.
- The session opens on first use: connection / TLS / login errors appear on the first query as
  `NPS session open failed: …`.
- Not supported: Kerberos / SSPI, client certificates, partitioned execution, Substrait, fixed `BINARY`
  columns (cast to VARBINARY).

## Connecting

A discrete option always wins. Between a URI and a connection string, the **first** one applied wins for
a shared field.

**URI**: `netezza://[user[:password]@]host[:port][/database][?key=value&…]` (also `postgresql://`,
`postgres://`, `arpenz://`; `GetOption("uri")` always returns `netezza://`). Path = database. Parameters: `user`, `password`, `dbname`,
`application_name`, `connect_timeout`, `sslmode`, `sslrootcert`, `require_password_encryption`.

```text
netezza://admin:<pw>@nzhost:5480/SYSTEM?sslmode=verify-full&sslrootcert=/etc/nz/ca.pem
```

**Connection string** (`adbc.arpenz.connection_string`): `Server` / `Data Source` / `Address`
(`host`, `host,port`, `tcp:host,port`; **no `Port` keyword**), `Database` / `Initial Catalog`,
`User ID` / `UID` / `User`, `Password` / `PWD` (max 255 bytes), `Application Name`, `Connection
Timeout`, `Encrypt`, `TrustServerCertificate`, `RequirePasswordEncryption`. No `sslmode`,
`ssl_root_cert` or `tls_min_version` keyword: use discrete options or the URI.

```text
Server=nzhost,5480;Database=SYSTEM;User ID=admin;Password=<pw>;Encrypt=true;TrustServerCertificate=false
```

## Options (database, `db_kwargs`)

| Key | Default | Notes |
|---|---|---|
| `adbc.arpenz.server` / `.port` | `localhost` / **5480** | |
| `adbc.arpenz.database` | – | Set it (e.g. `SYSTEM`) |
| `adbc.arpenz.username` / `.password` | – | Only supported login |
| `adbc.arpenz.require_password_encryption` | `false` | Refuse to send the password without TLS |
| `adbc.arpenz.application_name` | `arpenetezza` | |
| `adbc.arpenz.sslmode` | `prefer` | `disable`, `prefer`, `require`, `verify-ca`, `verify-full` (lower case) |
| `adbc.arpenz.ssl_root_cert` | OpenSSL defaults | PEM with the **intermediate and root** CA |
| `adbc.arpenz.tls_min_version` | `1.0` with `prefer`, `1.2` otherwise | `1.0`, `1.1`, `1.2`, `1.3` |
| `adbc.arpenz.encrypt` / `.trust_server_cert` | `false` / `true` | Used only without `sslmode`: `encrypt=true` → `require`, or `verify-full` with `trust_server_cert=false` |
| `adbc.arpenz.login_timeout` | `30` | TCP connect, seconds (`0` = 30) |
| `adbc.arpenz.socket_timeout` | `0` (none) | Per read / write |
| `adbc.arpenz.query_timeout` | `0` | **No effect**: cancel instead |
| `adbc.arpenz.buffer_size` | `32000` | Rows per Arrow batch |
| `adbc.arpenz.char_encoding` | `latin9` | `utf8` = pass CHAR / VARCHAR bytes through untranscoded |
| `adbc.arpenz.interval_type` | `text` | `native` → interval[month_day_nano] (not Parquet-writable) |
| `adbc.arpenz.json_extension` | `off` | `on` tags JSON / JSONB columns `arrow.json` |
| `adbc.arpenz.ingest_method` | `auto` | `external`, `insert` |
| `arpeio.adbc.license`, `arpeio.adbc.license_file` | – | Licence |

- `adbc.connection.autocommit`: only `true` or `1` turn it on; **any other value (`True`, `yes`)
  turns it off**. `adbc.connection.catalog` switches database (reconnects; uncommitted work lost; an
  unknown database → `NOT_FOUND`, 0.3.7).
  `adbc.connection.db_schema` sets `search_path`.
- Statement options: `adbc.arpenz.batch_size`, `adbc.arpenz.max_batch_size` (1,000,000).
  `memory_budget_mb` and `prefetch` are accepted no-ops (any value since 0.3.7).
- Removed (rejected as unknown since 0.3.0): `copy_format`, `geospatial`, `uuid_casing`,
  `arpenz.bulk_*`. Accepted but useless: `trusted`, `auth_type=integrated`, `krb5.*`, `channel_binding`,
  `gssencmode` (`require` values are refused at connect).

## TLS

| `sslmode` | If the server declines TLS | Verifies |
|---|---|---|
| `disable` | – (fails on a TLS-only server) | – |
| `prefer` (default) | continues in clear | nothing |
| `require` | fails | nothing |
| `verify-ca` | fails | chain |
| `verify-full` | fails | chain + host name |

- The OS certificate store is never read: on RHEL-family and Windows give `ssl_root_cert` (RHEL:
  `SSL_CERT_FILE=/etc/pki/tls/certs/ca-bundle.crt` also works).
- IBM Cloud doesn't send its intermediate certificate: put intermediate **and** root in
  `ssl_root_cert` (DigiCert Global G2 TLS RSA SHA256 2020 CA1 + DigiCert Global Root G2).
- `verify-full`: connect by DNS name.

## Types

Read (Netezza → Arrow): `BOOLEAN` → bool; `BYTEINT` → int8; `SMALLINT` / `INTEGER` / `BIGINT` →
int16/32/64; `REAL` / `DOUBLE` → float32/64; `NUMERIC(p,s)` → decimal128(p,s); `MONEY` →
decimal128(19,2); `CHAR` / `VARCHAR` (LATIN9 → UTF-8), `NCHAR` / `NVARCHAR`, `JSON`, `JSONB`,
`JSONPATH`, `TIME WITH TIME ZONE` → utf8; `VARBINARY`, `ST_GEOMETRY` → binary; `DATE` → date32;
`TIME` → time64[us]; `TIMESTAMP` → timestamp[us] (no tz); `INTERVAL` → utf8 (or native).

Since 0.3.7 every result field carries its Netezza type name in the Arrow field metadata key
`ARPENZ:type` (`NUMERIC(18,2)`, `CHARACTER VARYING(100)`…), for queries, `ExecuteSchema` and
`GetTableSchema`: `table.schema.field(0).metadata[b"ARPENZ:type"]`. DDL reports 0 rows affected
(was -1).

Ingest `create` DDL: bool → BOOLEAN; int8 → SMALLINT; int16/32/64 → SMALLINT / INTEGER / BIGINT;
float32/64 → REAL / DOUBLE PRECISION; utf8 → NVARCHAR(n) (width shared from the 64 KB row budget, up to
16000); binary → VARBINARY(n); decimal128 → NUMERIC(p,s); date32 → DATE; time → TIME; timestamp →
TIMESTAMP (tz ignored, UTC value stored, ns → µs). Refused: unsigned ints, date64, decimal256,
string_view, fixed_size_binary. For specific widths, create the table yourself and `append`.

Parameters (`?` or `$1`) are substituted client-side as literals; N parameter rows run the statement N
times.

## Ingest

Data must come as a stream (Python `adbc_ingest` does this). Modes `create`, `append`, `replace`,
`create_append`.

| `ingest_method` | Path |
|---|---|
| `auto` (default) | External-table load (`INSERT … SELECT … FROM EXTERNAL … REMOTESOURCE`), parsed in parallel by the server; falls back to batched `INSERT` when it can't be used |
| `external` | External table only; an error instead of the fallback |
| `insert` | Batched `INSERT … UNION ALL` (≤ 1000 rows per statement; much slower) |

- The external load needs every Arrow column to match a target column **by exact name**, a target
  `NUMERIC` with precision, a record under 64 KB; any rejected row fails the whole load (`MAXERRORS 1`).
- Slow ingest = fallback to INSERT: set `ARPEIO_ADBC_WRITE_TIMING=1` and look for `path=insert-union`
  in the `[NZ INGEST SUMMARY]` line; usually a column-name mismatch.
- Autocommit on: the whole ingest is one transaction. Off (dbapi default): `conn.commit()`.
- `temporary=True` → `CREATE TEMP TABLE` (needs server, username and password on the connection).

## Example

```python
import os
import adbc_driver_manager.dbapi as dbapi
import pyarrow as pa

db_kwargs = {
    "uri": f"netezza://admin:{os.environ['NZ_PASSWORD']}@nzhost:5480/SYSTEM?sslmode=verify-full",
    "adbc.arpenz.ssl_root_cert": "/etc/nz/ibm_nps_ca.pem",
    "adbc.arpenz.ingest_method": "external",
}
with dbapi.connect(driver="arpenz", db_kwargs=db_kwargs) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM lineitem LIMIT 10")
        print(cur.fetch_arrow_table())
        cur.adbc_ingest("MY_TABLE", pa.table({"ID": [1, 2, 3], "NAME": ["a", "b", "c"]}),
                        mode="create_append")
    conn.commit()
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `NPS session open failed: …` on the first query | Connection error surfaced lazily: host, port 5480, firewall |
| `getaddrinfo(…)` | Host doesn't resolve (IBM Cloud instance stopped?) |
| `FATAL 1:  Database "…" does not exist` / `Cannot switch current_catalog to '…'` | Wrong database: `SELECT DATABASE FROM _v_database` |
| `Connection string: unknown keyword 'Port'` / `'sslmode'` | Port in `Server=host,5480`; TLS via discrete options or URI |
| `TLS handshake failed: the server only offers a TLS version older than the allowed minimum …` | NPS 7: `tls_min_version=1.0`, or `sslmode=disable` on a trusted network |
| `certificate verification failed: unable to get local issuer certificate …` | `ssl_root_cert` with intermediate + root |
| `IP address mismatch` | `verify-full`: connect by DNS name |
| `the server only accepts TLS connections …` | Drop `sslmode=disable` |
| `integrated/trusted authentication (Kerberos, SSPI) is not supported for Netezza yet` | Use username + password |
| `refusing to send the password: the connection is not encrypted …` | `sslmode=require` or stronger |
| `Access denied due to too many unsuccessful login attempts` | Account locked on the server |
| `DBOS decode: NzType N … not yet supported` | Fixed `BINARY` column: cast to VARBINARY |
| `Bulk ingest requested but no data was bound` | Use `adbc_ingest` (stream), set table and mode |
| `ingest: unsupported Arrow type …` | Cast the column (unsigned, date64…) |
| `ingest_method=external: the external-table load cannot express this ingest (…)` | Fix the reason (names, widths) or `auto` |
| `The transaction was rolled back by an earlier error …` | `conn.rollback()` and redo the work |
| Autocommit seems off | Use exactly `true` or `1` |
| `Unknown database option: …` | Typo or removed option |

Docs vs driver 0.3.7: when both a URI and a connection string set a field, the first one wins (docs:
the last); the INSERT fallback is also capped at 8000 ÷ columns rows per statement.
