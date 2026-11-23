# ArpeDb2 – IBM Db2 ADBC driver (0.4.0)

Load name `arpedb2`. Native DRDA over TCP or TLS (no Db2 client, no CLI/ODBC, no JCC). Options
`adbc.arpedb2.*`. ADBC 1.1.0. One driver for Db2 for LUW and Db2 for i: the family is detected at connect
and the catalog queries, text encoding and package handling follow it.

## Targets

| Server | State |
|---|---|
| Db2 for LUW 12.1 | Validated (`VECTOR` columns from 12.1.2) |
| Db2 for LUW 11.5 | Supported, not run continuously |
| Db2 for LUW 11.1 and older | Not yet validated |
| Db2 for i (IBM i 7.5) | Validated: queries, DML, parameters, ingest, `CALL`, metadata, statistics |
| Db2 for z/OS | **Not supported** |

- Clients: Linux x64 (glibc 2.28+), Windows x64 (self-contained DLL). No macOS. OpenSSL linked in.
- Default ports: Db2 for LUW 50000 (its SSL port is separate, e.g. 50001); Db2 for i 446, TLS 448.
- Not supported: `ExecutePartitions` / `ReadPartition`, Substrait, client certificates, z/OS.

## Connecting

A discrete option always wins over the same field from a URI or connection string, whatever the order.
`server`, `database`, `username` and `password` are required for a password logon (missing one →
`Cannot connect: adbc.arpedb2.<field> is required`, before any socket).

**URI** (`uri`): `db2://[user[:password]@]host[:port][/database][?key=value&…]` (also `arpedb2://`).
The path is the **database name**. Percent-encode `@ : / ? # & % +` in the password (or leave it out and
pass `adbc.arpedb2.password`). IPv6 in brackets. Query parameters: `user`, `password`, `database` /
`dbname`, `sslmode`, `sslrootcert`, `sslhostname`; anything else, or a value given twice, is refused.

```text
db2://db2inst1:<pw>@db2.example.com:50001/sample?sslmode=verify-full
```

**Connection string** (`adbc.arpedb2.connection_string`, ADO.NET): case-insensitive keys, unknown or
repeated keys refused.

| Keys | Field |
|---|---|
| `Server`, `Hostname`, `Host`, `Data Source`, `Address`, `Addr`, `Network Address` | server (also `host,port`, `tcp:host,port`) |
| `Port` / `Database`, `Initial Catalog` | port / database |
| `User ID`, `UID`, `User` / `Password`, `PWD` | login |
| `sslmode`, `sslrootcert`, `sslhostname` / `HostNameInCertificate` | TLS |

Every other option is a discrete option.

## Options (database, `db_kwargs`)

Booleans: `true`/`false`, `1`/`0`, `yes`/`no`, `on`/`off`. A bad value → `INVALID_ARGUMENT`
`Invalid <key> '<value>' (expected ...)`; too long a value (255 bytes for server / user / password, 18
for database, 1023 for paths) is refused, never cut.

| Key | Default | Notes |
|---|---|---|
| `adbc.arpedb2.server` / `.port` / `.database` | – / `50000` / – | Target |
| `adbc.arpedb2.username` / `.password` | – | Also the standard `username` / `password` |
| `adbc.arpedb2.auth.mechanism` | unset (negotiate) | `password`, `encrypted_password`, `kerberos`, `token`; an explicit value is never switched |
| `adbc.arpedb2.auth.encryption` | `auto` | `aes`, `des` for the encrypted logon; `aes` never uses DES |
| `adbc.arpedb2.auth.require_encryption` (alias `.require_password_encryption`) | `false` | Never send the password in clear |
| `adbc.arpedb2.kerberos.spn` / `.cred` / `.keytab` | – / `default` / – | SPN `<instance>/<fqdn>@<REALM>` required; `cred`: `default` (`ccache`), `keytab`, `password` |
| `adbc.arpedb2.sslmode` | `disable` | `require`, `verify-ca`, `verify-full` |
| `adbc.arpedb2.sslrootcert` / `.sslhostname` | – | CA PEM / name `verify-full` checks |
| `adbc.arpedb2.login_timeout` | `30` | Whole logon (TCP + TLS + DRDA), `0` = none |
| `adbc.arpedb2.query_timeout` | `0` | Per wait; on expiry `TIMEOUT` and **the connection is closed** |
| `adbc.arpedb2.keepalive` / `_idle` / `_interval` / `_count` | `true` / 60 / 10 / 6 | TCP keepalive |
| `adbc.arpedb2.alternate_servers` | – | `host[:port],…` (max 16), tried in order at connect only |
| `adbc.arpedb2.connect_retry_count` / `_delay` | 0 / 3 | Extra rounds / seconds between rounds |
| `adbc.arpedb2.client.application_name` / `.workstation` / `.user` / `.accounting` | – | `CURRENT CLIENT_*` registers; also connection options |
| `adbc.arpedb2.package_collection` | Db2 for i: current library; LUW: `NULLID` | DRDA package collection (database option only) |
| `adbc.arpedb2.package_auto_bind` | `true` | Bind the package on `SQLCODE -805` and retry |
| `adbc.arpedb2.decfloat_as` | `double` | `string` = exact `DECFLOAT` text |
| `adbc.arpedb2.timestamp_precision` | `micro` | `exact`: `TIMESTAMP(7..9)` → `timestamp[ns]`, `(10..12)` → text |
| `adbc.arpedb2.bit_data_ccsid` | `0` | A CCSID reads `FOR BIT DATA` as text |
| `adbc.arpedb2.buffer_size` | `32000` | Rows per Arrow batch (also statement) |
| `adbc.arpedb2.batch_max_bytes` | `0` | Byte cap per batch (also statement) |
| `adbc.arpedb2.qryblksz` | `0` (default) | DRDA query block bytes, `512..10485760` |
| `adbc.arpedb2.max_lob_bytes` | 64 MiB | Larger LOB / XML value → error naming the column |
| `adbc.arpedb2.defer_prepare` | `true` | Prepare travels with the first execute (SQL errors reported there) |
| `adbc.arpedb2.concurrent_access_resolution` | `default` | `currently_committed`, `wait_for_outcome`, `skip_locked_data` (Db2 for i) |
| `adbc.arpedb2.ingest.commit_interval_rows` | `0` | Commit every N rows (autocommit on) |
| `adbc.arpedb2.ingest.max_rejected_rows` | `0` | `-1` / N: set bad rows aside instead of failing |
| `adbc.arpedb2.trace` / `_file` / `_sql` | `off` / stderr / `true` | DRDA trace (`summary`, `hex`); env `ARPEDB2_TRACE*` too |

Read-only: connection `adbc.arpedb2.auth.negotiated_mechanism`, `.auth.negotiated_encryption`,
`.tls.version`, `.tls.group`, `.tls.cipher`, `.connected_server`; database `adbc.arpedb2.openssl_version`.
Standard: `adbc.connection.autocommit`, `adbc.connection.transaction.isolation_level`,
`adbc.connection.db_schema` (`SET CURRENT SCHEMA`, not a metadata filter).

Statement options: `buffer_size`, `batch_max_bytes`, `defer_prepare`, `bulk.chunk_rows` (1000),
`ingest.varchar_length` (4000), `ingest.create_table_suffix` (e.g. `IN TS1 COMPRESS YES`),
`ingest.rejected_rows` (read-only JSON), `call.out_params`, `call.out_values` (read-only JSON),
`call.next_result_set`; standard `adbc.ingest.temporary`.

## Authentication

| Mode | Set | Notes |
|---|---|---|
| Password (default) | `username` + `password` | Without TLS: encrypted logon first, plain password only if the server does not offer it, with a one-line stderr warning. Inside TLS: plain first |
| Encrypted password | `auth.mechanism=encrypted_password` | AES or DES (`auth.encryption`); protects against passive listening only |
| Kerberos | `auth.mechanism=kerberos` + `kerberos.spn` | Linux: MIT Kerberos loaded from the host (`krb5-libs` / `libgssapi-krb5-2`). Windows: SSPI, not yet validated, no `keytab` |
| Token / API key | `auth.mechanism=token`, token as `password` | Requires `verify-full`, or `verify-ca` / `require` with `sslrootcert` |

- Real protection is TLS `verify-full`; `auth.require_encryption=true` refuses the clear-text fallback.
- Server `AUTHENTICATION=SERVER_ENCRYPT` refuses plain passwords; `ALTERNATE_AUTH_ENC=AES_ONLY` refuses DES.

## TLS

**Implicit TLS on a separate port**: `port` must be the server's SSL port.

| `sslmode` | Encrypts | Verifies |
|---|---|---|
| `disable` (default) | no | – |
| `require` | yes | chain only if `sslrootcert` is set |
| `verify-ca` | yes | chain against `sslrootcert` (required) |
| `verify-full` | yes | chain + host name |

- Trust: `sslrootcert`, else the system store (Linux distro bundle, `SSL_CERT_FILE` / `SSL_CERT_DIR`;
  Windows ROOT store). Private CA → `sslrootcert`. Connecting by IP / alias / tunnel → `sslhostname`.
- TLS 1.2+, 1.3 when offered, `X25519MLKEM768` offered first. `ARPEIO_ADBC_LOG_TLS=1` logs each handshake.

## Transactions and isolation

- Autocommit on (default): each statement commits when it ends; a query commits at end of data or when
  its stream is released, so readers hold locks only while reading. `Commit` / `Rollback` →
  `INVALID_STATE`.
- Autocommit off: DDL and ingest `CREATE` / `DROP` roll back too. A failed load rolls back to a savepoint
  and keeps earlier work pending. Closing rolls back.
- After `SQLCODE -911` (deadlock, lock timeout) Db2 has rolled back the whole unit of work.
- Isolation: the four ADBC levels and `default` on both families; `adbc.arpedb2.transaction.isolation.no_commit`
  (Db2 for i `*NONE`, no rollback at all) for tables that are not journaled.
- Up to 64 result streams can be open at once on one connection (one package section each).
- Cancel (`adbc_cancel`) shuts the socket: the connection must be reopened.

## Types

Read: `SMALLINT`/`INTEGER`/`BIGINT` → int16/32/64; `REAL`/`DOUBLE` → float32/64; `DECIMAL`/`NUMERIC` →
decimal128(p,s); `DECFLOAT` → float64 (`decfloat_as=string` → exact text); char types, `GRAPHIC`, `CLOB`,
`DBCLOB`, `XML` → utf8; `FOR BIT DATA`, `BINARY`, `VARBINARY`, `BLOB` → binary; `DATE` → date32;
`TIME` → time32[s]; `TIMESTAMP(p)` → timestamp[us]; `BOOLEAN` → **int16 on Db2 for LUW** (the server
describes it as `SMALLINT`), bool on Db2 for i; `VECTOR` → utf8 text (`[1.5,-2.25,3]`, parse client-side).
Field metadata `ARPEDB2:type` = Db2 DDL type (`VARCHAR(40)`, `DECIMAL(9,2)`). Db2 for i: EBCDIC CCSIDs
(incl. Japanese, Chinese, Korean) converted to UTF-8; `ROWID` → binary; `DATALINK` → utf8 URL;
`DECIMAL` over 31 digits → query fails with SQLCODE `+802` (cast to `VARCHAR`).

Ingest `create` DDL: int8/uint8/int16 → SMALLINT; int32/uint16 → INTEGER; int64/uint32 → BIGINT;
uint64 → DECIMAL(20,0); float32/64 → REAL/DOUBLE; bool → BOOLEAN; decimal → DECIMAL(p,s) (p capped at 31);
utf8 → `VARCHAR(ingest.varchar_length)`; binary → VARBINARY; date → DATE; time → TIME; timestamp[s/ms/us/ns]
→ TIMESTAMP(0/3/6/9) (a time zone is not kept: UTC is stored); `fixed_size_list<float32|int8>[n]` →
VECTOR(n, …). A field carrying `ARPEDB2:type` keeps that Db2 type (Db2-to-Db2 copies keep `CHAR(3)` etc.).

## Ingest

`cur.adbc_ingest(table, data, mode=…, db_schema_name=…, temporary=…)`: parameterised `INSERT`, up to 1000
rows (and 1 MiB) per request. Modes `create` (default), `append`, `replace`, `create_append`.
`temporary=True` → `DECLARE GLOBAL TEMPORARY TABLE` in `SESSION` (needs a user temporary table space;
not with a schema). Use a **new cursor** when the target changes. Large loads: set
`ingest.commit_interval_rows` (e.g. `50000`) with autocommit on, so the load does not hold every row lock
and log record until the end.

## Stored procedures

`cur.execute("CALL S.P(?)", [10])` returns the procedure's first result set. OUT / INOUT: set statement
option `adbc.arpedb2.call.out_params` (e.g. `"2,3"`), bind NULL for OUT positions; without a result set
they come back as a one-row result, otherwise read JSON from `adbc.arpedb2.call.out_values`. Further
result sets: set `adbc.arpedb2.call.next_result_set` and call `cur.adbc_statement.execute_query()` again
(the procedure runs once; `NOT_FOUND` when none is left).

## Example

```python
import os
import adbc_driver_manager.dbapi as dbapi

db_kwargs = {
    "adbc.arpedb2.server": "db2.example.com", "adbc.arpedb2.port": "50001",
    "adbc.arpedb2.database": "sample", "adbc.arpedb2.username": "db2inst1",
    "adbc.arpedb2.password": os.environ["DB2_PASSWORD"],
    "adbc.arpedb2.sslmode": "verify-full", "adbc.arpedb2.sslrootcert": "/etc/ssl/db2-ca.pem",
}
with dbapi.connect(driver="arpedb2", db_kwargs=db_kwargs, autocommit=True) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM SALES.ORDERS WHERE ORDER_DATE >= ?", ["2026-01-01"])
        orders = cur.fetch_arrow_table()
    with conn.cursor() as cur:
        cur.adbc_ingest("ORDERS_COPY", orders, mode="create", db_schema_name="STAGING")
```

## Troubleshooting

| Error | Fix |
|---|---|
| `login timed out after N s` with `sslmode` set | `port` is the plain port: use the SSL port |
| `DSS magic mismatch: expected 0xd0 …` | `port` is the SSL port without `sslmode`: set it |
| `database not found (RDBNFNRM)` | `database` is the Db2 database name, not an instance or schema |
| `security check failed: password invalid` | Wrong user / password, or no `CONNECT` |
| `the server does not offer encrypted password logon … require_encryption …` | Use TLS, or `SERVER_ENCRYPT` on the server |
| `sslmode=verify-ca needs adbc.arpedb2.sslrootcert` | Set the CA, or use `verify-full` |
| `auth.mechanism=token requires TLS that verifies …` | `verify-full`, or `sslrootcert` |
| `… needs MIT Kerberos, which was not found` / `SPN required` | Install `krb5-libs` / `libgssapi-krb5-2`; set `kerberos.spn` |
| `SQLCODE -805` (Db2 for i, package not found) | Bind failed: set `package_collection` to a library you own |
| `SQLCODE -7008` (Db2 for i) | Table not journaled: `no_commit` isolation, `WITH NC`, or journal it |
| `has CCSID N, which this driver cannot decode` | `CAST(col AS VARCHAR(n) CCSID 1208)` |
| `ExecuteQuery … takes exactly one parameter row` | `executemany` is for DML; use `IN (?, ?)` or a temp table join |
| `SQLCODE -418` on `SELECT ?` | `CAST(? AS INTEGER)` |
| `unmappable SQLTYPE` | Cast the column in SQL |
| `SQLCODE -911 SQLSTATE 40001` | Lock timeout / deadlock: the unit of work was rolled back, retry |
| `… timed out … (query_timeout); the connection is closed` | Raise `query_timeout`, reconnect |
| `SQL0964C` (log full) on a large load | `ingest.commit_interval_rows` |
| Schema missing from `GetObjects` | System schemas are hidden unless named exactly; filters are `LIKE` (escape `_` with `\`) |

Errors carry the Db2 SQLCODE as vendor code and the SQLSTATE. For anything else: `ARPEDB2_TRACE=hex`
and `ARPEDB2_TRACE_FILE=<path>` (SQL text and data are written unless `ARPEDB2_TRACE_SQL=off`).
