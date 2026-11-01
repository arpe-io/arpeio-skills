# FastBCP – Connections

## Connection types (`--connectiontype`, short `-C`, default `mssql`)

| Value | Source | Notes |
|---|---|---|
| `mssql` | Microsoft SQL Server | Default |
| `adbc_mssql` | SQL Server via arpe.io ADBC FastDriver | Parquet only, x64 only; about 2x faster for Parquet |
| `pgsql` | PostgreSQL, standard connection | |
| `pgcopy` | PostgreSQL via COPY protocol | Fastest non-ADBC PostgreSQL path |
| `adbc_pgsql` | PostgreSQL via ADBC FastDriver | Parquet only, x64 only |
| `mysql` | MySQL / MariaDB | |
| `oraodp` | Oracle (ODP.NET) | |
| `adbc_oracle` | Oracle via ADBC FastDriver | Parquet only, x64 only |
| `hana` | SAP HANA | |
| `hanaexport` | SAP HANA using native `EXPORT` | Shown in the docs' examples only; recommended there for large datasets |
| `teradata` | Teradata | |
| `adbc_teradata` (1.4+) | Teradata via ADBC (embedded ArpeTeradata driver); the TLS logon needs no Teradata client software | Parquet only, x64 only; no `--trusted` |
| `nzsql` | IBM Netezza | |
| `adbc_netezza` | Netezza / IBM NPS via ADBC (embedded driver, no NZdotNET, works with IBM Cloud NPS) | Parquet only, x64 only |
| `clickhouse` | ClickHouse | |
| `odbc` | Any ODBC source (e.g. DB2) | Needs `--dsn` or `--sourceconnectstring`, and the ODBC driver installed |
| `oledb` | Any OLE DB source | Windows only. Needs `--provider` (e.g. `MSOLEDBSQL`, `NZOLEDB`, `OraOLEDB.Oracle`) |
| `backup_mssql` (1.5+) | SQL Server full backup files (`.bak`), read without restoring them and without a SQL Server instance (embedded ArpeMSBackup ADBC driver) | Parquet only, x64 only; no credentials. See below |
| `backup_rman` (1.5.2+) | Oracle RMAN backup sets, read without restoring them and without an Oracle instance (embedded ArpeOraBackup ADBC driver) | Parquet only, x64 only; no credentials. See below |

`msoledbsql`, `nzcopy` and `nzoledb` also appear in the docs (e.g. as `Physical` sources) but not in
this table; use them only if the user asks.

Embedded drivers (nothing to install): ClickHouse, DuckDB, SQL Server, MySQL, Netezza, Oracle,
PostgreSQL, SAP HANA, Teradata. Any other database goes through `odbc` / `oledb` with its own driver.

### Platform support

| | Windows x64 | Linux x64 | Linux ARM64 |
|---|---|---|---|
| `adbc_*`, `backup_mssql`, `backup_rman` | yes | yes | **no** |
| Teradata, Netezza, SAP HANA | yes | yes | **no** |
| OLE DB | yes | **no** | **no** |
| All others | yes | yes | yes |

## Connection parameters

| Long | Short | Notes |
|---|---|---|
| `--server` | `-S` | Required unless `--sourceconnectstring` is used. Formats: `host`, `host:port`, `host,port`, `host\instance`, Oracle Easy Connect `host:1521/service`, or an Oracle TNS alias. With `backup_mssql`: the directory of the backups; with `backup_rman`: the directory of the backup pieces |
| `--database` | `-I` | Optional (engine dependent). Not `-d`, which is the CSV delimiter. With `backup_mssql`: the backup file name; with `backup_rman`: the piece names or patterns, `;` separated |
| `--trusted` | `-A` | Windows / Kerberos integrated auth, if the RDBMS supports it (not with `adbc_teradata`) |
| `--user` | `-U` | |
| `--password` | `-X` | Masked in FastBCP logs |
| `--sourceconnectstring` | `-g` | Full connection string, in double quotes. Overrides every other connection parameter. Works with any connection type |
| `--dsn` | `-N` | ODBC DSN (for `odbc`, unless `--sourceconnectstring` is used) |
| `--provider` | `-P` | OLE DB provider (required for `oledb`) |
| `--applicationintent` | – | `ReadOnly` (default) or `ReadWrite`. `mssql`, `adbc_mssql`, `oledb` only. `ReadOnly` routes to an Always On readable secondary when there is one |

Many doc examples (PostgreSQL, MySQL, ClickHouse) also pass `--port "<port>"`, even though it is not
listed on the Connection Parameters page. PostgreSQL also accepts `--server "host:port"`. For MySQL and
ClickHouse, follow the docs' examples and use `--server "host" --port "3306"`.

## Credentials without leaking them

FastBCP has no built-in env var for the DB user or password. The docs suggest storing them in shell
variables (for example `FASTBCP_USER` / `FASTBCP_PASSWORD`) and referencing them:

```bash
export FASTBCP_PASSWORD='...'   # or read from a secret store
./FastBCP ... --user "$FASTBCP_USER" --password "$FASTBCP_PASSWORD" ...
```

```powershell
.\FastBCP.exe ... --user $env:FASTBCP_USER --password $env:FASTBCP_PASSWORD ...
```

`pgsql` and `adbc_pgsql` also honour the libpq variables `PGUSER`, `PGPASSWORD`, `PGPASSFILE` /
`~/.pgpass` and `PGSSLROOTCERT` when `--user` / `--password` / `--sourceconnectstring` don't set them.

## Connection string examples

SQL Server:

```
--sourceconnectstring "Server=myserver;Database=mydb;User ID=FastUser;Password=FastPassword;Encrypt=True;TrustServerCertificate=True"
```

DB2 through ODBC:

```
--connectiontype "odbc" --sourceconnectstring "Driver={IBM DB2 ODBC DRIVER};Database=SAMPLE;Hostname=db2server.company.corp;Port=25010;Protocol=TCPIP;Uid=db2user;Pwd=password;CurrentSchema=DB2ADMIN"
```

Netezza with the OLE DB provider (Windows):

```
--connectiontype "oledb" --provider "NZOLEDB"
```

### ADBC connection strings

With `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza` and `adbc_teradata`, `--sourceconnectstring` accepts:

- native ADO.NET-style `Key=Value;` pairs, translated to driver keys. Keys with no equivalent
  (e.g. `Pooling=false`) are dropped with a warning;
- explicit `adbc.*` / `arpeio.*` keys, which take precedence over translated keys;
- a whole-connection `uri` / `adbc.<driver>.connection_string`. With this form, native keys are
  dropped, so put everything inside the URI.

Since 1.3, metadata/planning and data both go through the ADBC driver and the same connection string.

Per-driver notes:

- **adbc_netezza**: keys such as `Server`, `Host`, `Port`, `Database`, `User ID`, `Password`,
  `SSL Mode`, `sslmode`, `tls_min_version`, `char_encoding`, `login_timeout`. **NPS 7.x needs
  `tls_min_version=1.0`.**
  ```
  --connectiontype adbc_netezza --sourceconnectstring "Server=nps-host;Port=5480;Database=SALES;User ID=admin;Password=FastPassword;SSL Mode=require"
  ```
- **adbc_pgsql**: a `NaN` in a `NUMERIC` column makes the read fail. Add
  `adbc.arpepgsql.numeric_special=null` to the connection string to read it as `NULL`.
- **adbc_oracle**: resolves TNS aliases through `tnsnames.ora` (from `TNS_ADMIN`, then
  `$ORACLE_HOME/network/admin`). Encryption and integrity are negotiated (`accepted`). Override with
  the `FASTBCP_ADBC_ENCRYPTION` / `FASTBCP_ADBC_DATA_INTEGRITY` env vars. Since 1.4 (ArpeOracle 0.4.1):
  TCPS verifies the server certificate by default; Kerberos forwards the TGT only with
  `krb5_forward=true` and requires mutual authentication; token and OAuth2 logons require TCPS.
  Since 1.5 (ArpeOracle 0.4.3), `adbc.arpeoracle.data_integrity_types` chooses the Native Network
  Encryption integrity algorithm.
- **adbc_mssql**: an `ApplicationIntent` key in the connection string is passed to the driver.
  Supports up to 1024 columns (since 1.2). Reads `sql_variant` and the SQL Server 2025 `JSON` type
  since 1.4 (ArpeMSSQL 0.7.6). Since 1.5 (ArpeMSSQL 0.7.8), parallel connections are opened 50 ms apart
  so SQL Server spreads them over its schedulers (4-way parallel reads up to twice as fast);
  `ARPEMSSQL_CONNECT_SPACING_MS` changes the gap.
- **adbc_teradata** (1.4+): same SQL dialect as `teradata`. Since 1.5 (ArpeTeradata 0.3.5), a `uri` or
  ADO.NET `connection_string` key is parsed, like the other drivers.
  - Connect with `--server "host[:port]"` (legacy port, default `1025`), `--user`, `--password`, optional
    `--database`, or a connection string in the Teradata provider spelling: `Server`/`Host`/`Data Source`,
    `Port`/`DBS Port`/`Port Number`, `Database`/`Initial Catalog`, `User Id`/`User Name`/`UID`,
    `Password`/`PWD`, `SSL Mode`, `HTTPS Port`, `SSL CA`, `SSL CA Path`, `TMODE`, `Login Timeout`,
    `Query Timeout`, `TTU Path`. Explicit `adbc.arpeteradata.*` keys win.
  - `--trusted` is refused (`-C adbc_teradata does not support trusted (-A) authentication`).
  - Logon: by default (`sslmode=prefer`) TLS on the HTTPS port 443, with no Teradata client software.
    If the server has no TLS, or with `SSL Mode=disable`, it uses the legacy port 1025, which needs the
    customer's Teradata Tools and Utilities (TTU, CLIv2). Linux: TTU is found through `FASTBCP_TTU_PATH`
    (see environment.md), then a `ttu/` or `TeradataToolsAndUtilitiesBase/` folder next to the binary.
    Windows: the driver finds `wincli32.dll` itself.
  - Logon mechanisms (`logmech`): `LDAP`, `JWT`, `KRB5`, `TDNEGO`. JWT and Kerberos work over TLS without TTU.
  ```
  --connectiontype adbc_teradata --sourceconnectstring "Server=td-host;Port=1025;User Id=FastUser;Password=FastPassword;SSL Mode=disable"
  ```

## SQL Server backup files (`backup_mssql`, 1.5+)

Reads a table straight from a SQL Server **full backup** (`.bak`), without restoring it and without any
SQL Server instance. The backup is read in place (random access by page), never loaded in memory, so
backups much larger than the RAM work.

- `--server`: the **directory** holding the backups (`D:\Backups\MSSQL`, `/var/opt/mssql/backup`).
- `--database`: the **backup file** in that directory, or a pattern (`*`, `?`) matching one backup.
- **No credentials**: `--user`, `--password`, `--trusted` and `--sourceconnectstring` are not used. The
  driver is licensed like the other ADBC FastDrivers (see environment.md, "License").
- Use **1.5.1+** (ArpeMSBackup 0.1.7): 1.5.0 fails on some backups and columnstore tables (see
  troubleshooting.md).
- Backups read: full and copy-only full, uncompressed or compressed with **MS_XPRESS** or **ZSTD**
  (not QAT_DEFLATE), any `BLOCKSIZE` / `MAXTRANSFERSIZE`, SQL Server **2017, 2019, 2022, 2025**
  (Windows or Linux), several data files and filegroups.
- **Striped backups** (`BACKUP ... TO DISK = a, DISK = b, …`): name every stripe with a pattern
  (`--database "tpch_*.bak"`) or a `;` list (`"tpch_1.bak;tpch_2.bak;tpch_3.bak"`), in any order. An
  incomplete set, stripes of different backups or a lone stripe are refused, naming the missing stripes.
- A file holding several backups (`WITH NOINIT`): the **last full backup** is read. Differential and log
  backups are not applied; a file without a full backup is refused.
- Tables: heaps, clustered indexes, partitions, ROW / PAGE compression, LOBs (`(n)varchar(max)`,
  `varbinary(max)`, `text`, `ntext`, `image`, row-overflow), `xml`, sparse and persisted computed
  columns, and **clustered columnstore** (archive compression and open delta store included).
  `sql_variant` is exported as text.
- **Refused**: backups of **TDE** databases and **encrypted backups** (`WITH ENCRYPTION`).
- Source: `--sourceschema` + `--sourcetable`, or a `--query` limited to
  `SELECT * | col1, col2, … FROM [schema.]table` (no filter).
- Parallel methods: `None` or `Physical` only (`Physical` resolves to `Pages`, see parallelism.md).
- Consistency: a backup taken while the database was being written is read as its pages are; the
  transaction log in the backup is not replayed.
- Parquet types: the `adbc_mssql` mapping, except `time(7)`, `datetime2(7)`, `datetimeoffset(7)` keep
  100 ns resolution (nanosecond unit, `adbc_mssql` writes microseconds), and `geography` / `geometry`
  are written as the SQL Server binary serialization (`adbc_mssql` writes WKB + GeoParquet metadata).
- Speed (docs, degree 8, TPC-H `orders` 15M rows): about 5M rows/s from an uncompressed backup, vs
  3.3M rows/s with `adbc_mssql` and 1.4M rows/s with `mssql` on the live database; 2.1–2.6M rows/s from
  MS_XPRESS / ZSTD backups.

## Oracle RMAN backup sets (`backup_rman`, 1.5.2+)

Reads a table straight from **RMAN backup sets**, without restoring them and without any Oracle
instance: on a backup server, from an archive, or for a database that no longer exists.

- `--server`: the **directory** holding the backup pieces (`/backup/rman/orcl`, `D:\Backups\RMAN`).
- `--database`: the **piece names or patterns** (`*`, `?`) in that directory, several separated by `;`,
  quoted: `--database "full_*"`, `--database "db_*;ctl_*"`. Give every piece of the backup (the pieces of
  a multi-piece set, the sets of several channels or of `FILESPERSET`), in any order. An incomplete set
  is refused, naming the missing piece. With several backups of the same files, the most recent copy
  of each datafile is read.
- **No credentials**: `--user`, `--password`, `--trusted` and `--sourceconnectstring` are not used. The
  driver is licensed like the other ADBC FastDrivers (see environment.md, "License").
- Backups read: full backup sets (database, tablespaces, datafiles, PDBs), uncompressed or
  `AS COMPRESSED BACKUPSET` with **BASIC**, **LOW**, **MEDIUM** or **HIGH**; Oracle **19c, 21c, 23ai,
  26ai** on Linux x86-64; CDB (root and PDBs) and non-CDB; bigfile and smallfile tablespaces.
- Incremental backups: a **level 0** backup is read like a full one; **level 1** sets (differential or
  cumulative, the changed blocks only) are refused, naming the set: give the full or level 0 pieces.
- Tables: heap tables (chained and migrated rows, more than 255 columns, empty tables).
- **Refused**, with a message: encrypted backup sets (`SET ENCRYPTION`); incremental level 1 sets; partitioned, index-organized,
  clustered, object, nested, temporary and external tables; tablespaces with a non-default block size;
  backups of big-endian platforms. LOB, `INTERVAL`, `ROWID` / `UROWID`, `JSON`, `VECTOR`, `XMLTYPE` and
  object columns: the table is refused, but its other columns can be exported by listing them in `--query`.
- Source: `--sourceschema` (the owner) + `--sourcetable`, or a `--query` limited to
  `SELECT * | col1, col2, … FROM [[container.]owner.]table` (no filter). A table of the root or of a PDB
  other than the only one: `CONTAINER.OWNER.TABLE` in `--query`.
- Parallel methods: `None` or `Physical` only (`Physical` resolves to `Pages`: ranges of the table's
  blocks, see parallelism.md).
- Consistency: a backup taken with the database open is read as its blocks are (redo not applied); a
  backup taken with the database mounted is consistent.
- Parquet types: the `adbc_oracle` mapping (NUMBER(p≤18,0) → int64, NUMBER(p,s) → decimal, NUMBER without
  precision → exact decimal text, DATE / TIMESTAMP(n) → timestamp with the unit of n, text → UTF-8).
- Speed (docs, a 1M-row table, degree 8, open included): under 1 s from an uncompressed, LOW, MEDIUM or
  HIGH backup; 6–7 s from a BASIC backup, or a HIGH backup of Oracle 19c.

  ```bash
  ./FastBCP \
    --connectiontype backup_rman \
    --server "/backup/rman/orcl" \
    --database "full_*" \
    --sourceschema "SALES" \
    --sourcetable "ORDERS" \
    --directory "/export/orders" \
    --fileoutput "orders.parquet" \
    --parallelmethod Physical \
    --paralleldegree 8
  ```
