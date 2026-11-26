# FastTransfer – Connections

## Source connection types (`--sourceconnectiontype`)

| Value | Source |
|---|---|
| `mssql` | SQL Server (.NET SqlClient) |
| `msoledbsql` | SQL Server through OLE DB (Windows) |
| `pgsql` | PostgreSQL (.NET Npgsql) |
| `pgcopy` | PostgreSQL using COPY (fastest; enables binary copy when the target is also `pgcopy`) |
| `mysql` | MySQL / MariaDB |
| `oraodp` | Oracle (ODP.NET) |
| `hana` | SAP HANA |
| `teradata` | Teradata |
| `nzsql` | Netezza (native .NET driver) |
| `nzbulk` | Netezza copy (marked work in progress: prefer `nzsql`) |
| `nzoledb` | Netezza through OLE DB (Windows) |
| `clickhouse` | ClickHouse |
| `duckdb` | DuckDB (database file, or `:memory:`) |
| `duckdbstream` | DuckDB in streaming mode (lower memory). **Use it for CSV / Parquet / JSON files** |
| `odbc` | Any ODBC source (DSN or connection string; driver must be installed) |
| `oledb` | Any OLE DB source (Windows; needs `--sourceprovider`) |
| `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_db2`, `adbc_teradata` (1.0+) | Bundled Arpe ADBC drivers, see "ADBC connection types" below. Only with an `adbc_*` target |
| `backup_mssql` (1.1+) | SQL Server full and differential backup files (`.bak`), read without restoring them and without SQL Server (bundled ArpeMSBackup driver). See "Backup sources" below. Only with an `adbc_*` target |
| `backup_rman` (1.1+) | Oracle RMAN backup sets (full, level 0, incremental level 1 to 4), read without restoring them and without Oracle (bundled ArpeOraBackup driver). See "Backup sources" below. Only with an `adbc_*` target |

Older docs list `nzcopy` ("Netezza copy, WIP"): the CLI does not accept it; the value is `nzbulk`.

Embedded drivers (nothing to install): ClickHouse, DuckDB, SQL Server, MySQL, Netezza, Oracle,
PostgreSQL, SAP HANA, Teradata, since 1.0 the ADBC drivers (Db2 included), and since 1.1 the two backup
drivers. Any other database
(e.g. Sybase) goes through `odbc` / `oledb`.

## Target connection types (`--targetconnectiontype`)

| Value | Target | Notes |
|---|---|---|
| `msbulk` | SQL Server | SQL Server BulkCopy |
| `pgcopy` | PostgreSQL | COPY: binary if the source is `pgcopy`, text (CSV) otherwise |
| `pgsql` | PostgreSQL | Native driver (INSERT … UNNEST) |
| `orabulk` | Oracle | Oracle BulkCopy |
| `oradirect` | Oracle | Oracle Direct |
| `mysqlbulk` | MySQL / MariaDB | MySQL BulkCopy |
| `clickhousebulk` | ClickHouse | ClickHouse BulkCopy |
| `hanabulk` | SAP HANA | |
| `teradata` | Teradata | |
| `nzbulk` | Netezza | Netezza BulkCopy |
| `duckdb` | DuckDB | `--targetserver` is the `.duckdb` file path |
| `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_db2`, `adbc_teradata` (1.0+) | SQL Server, PostgreSQL, Oracle, Netezza, Db2 for LUW, Teradata | Only with an `adbc_*` source |

The target table must already exist, unless `--loadmode Create` is used (1.1+, see
[parameters.md](parameters.md)). Before 1.1, FastTransfer does not create tables.

## Source connection flags

| Flag | Notes |
|---|---|
| `--sourceconnectiontype` | See table above |
| `--sourceserver` | `host`, `host:port`, `host,port`, `host\instance`, Oracle Easy Connect `host:1521/service`, Oracle TNS alias, DuckDB file path or `:memory:`. SQL Server: see "SQL Server server names" below. `backup_*`: the directory of the backup files |
| `--sourcedatabase` | Database name. Oracle: service/PDB name (e.g. `ORCLPDB`). `backup_*`: the backup file names or patterns, `;` separated |
| `--sourcetrusted` | Integrated auth (Windows / Kerberos). Not combinable with `--sourceuser`/`--sourcepassword` |
| `--sourceuser` | `backup_mssql`: the private key file of the certificate of an encrypted backup |
| `--sourcepassword` | Masked in logs. `backup_mssql`: the password of that key file |
| `--sourceconnectstring` | Full connection string, in double quotes. Overrides server/user/password/trusted/DSN/provider |
| `--sourcedsn` | ODBC DSN (`odbc`), unless `--sourceconnectstring` is used |
| `--sourceprovider` | OLE DB provider for `oledb`, e.g. `MSOLEDBSQL`, `NZOLEDB` |

## Target connection flags

| Flag | Notes |
|---|---|
| `--targetconnectiontype` | See table above |
| `--targetserver` | Same formats as `--sourceserver`; for `duckdb`, the database file path |
| `--targetdatabase` | Database name (Oracle: service/PDB name) |
| `--targettrusted` | Integrated auth. Not combinable with `--targetuser`/`--targetpassword` |
| `--targetuser` | |
| `--targetpassword` | Masked in logs |
| `--targetconnectstring` | Full connection string, overrides the other target connection flags |
| `--targetschema` | **Mandatory** |
| `--targettable` | **Mandatory** |

## SQL Server server names

For `mssql`, `msoledbsql`, `msbulk`, `adbc_mssql`, and `oledb` with a SQL Server provider,
`--sourceserver` / `--targetserver` take `host`, `host:port`, `host,port`, `host\INSTANCE` or
`host\INSTANCE,port`. `host` is a name, an IPv4 address or a bracketed IPv6 address (`[::1]:1433`).

- `host:port` is accepted by the native types since 1.1. On 1.0 and older, write `host,port` with them.
- `host\INSTANCE:port` and the `tcp:` prefix are refused before any connection (1.1+), with the form to
  use in the message: write `host\INSTANCE,port`, and drop `tcp:`.
- These rules apply to the server flags only: a `--sourceconnectstring` / `--targetconnectstring` is
  used as it is.

## ADBC connection types (1.0+)

The `adbc_*` types use the Arpe ADBC drivers **bundled** with FastTransfer (nothing to install). The data
travels as Arrow batches from one driver to the other, so **ADBC is needed on both ends**: an `adbc_*`
source with an `adbc_*` target, any two of the six. Windows x64 and Linux x64 only.

| Type | Database | Default port | Source | Target |
|---|---|---|---|---|
| `adbc_mssql` | SQL Server | 1433 | ✓ | ✓ |
| `adbc_pgsql` | PostgreSQL | 5432 | ✓ | ✓ |
| `adbc_oracle` | Oracle | 1521 | ✓ | ✓ |
| `adbc_netezza` | Netezza | 5480 | ✓ | ✓ |
| `adbc_db2` | Db2 for LUW | 50000 | ✓ | ✓ |
| `adbc_teradata` | Teradata | 1025 | ✓ | ✓ |

### With the usual flags (preferred)

Same flags as the native types: `--sourceserver`, `--sourcedatabase`, `--sourceuser` /
`--sourcepassword` or `--sourcetrusted`, and their target twins.

- `--sourceserver` / `--targetserver`: `host`, `host:port` or `host,port`. SQL Server: a named instance
  `host\INSTANCE`. Oracle: `host[:port]/service` or a TNS alias (`--sourcedatabase` is then the service
  name).
- `--sourcetrusted` / `--targettrusted`: integrated authentication. Windows SSPI or Kerberos for SQL
  Server and PostgreSQL; Kerberos for Oracle, Db2 and Teradata. **Not available for Netezza.**
- The session is named `FastTransfer` on the server (not for Teradata), and a SQL Server source connects
  with `ApplicationIntent=ReadOnly`, as with the native types.

### With a connection string

`--sourceconnectstring` / `--targetconnectstring` take the **driver's own vocabulary**, not an ADO.NET
string:

```
uri=sqlserver://user:password@host:1433/?database=tpch
uri=postgresql://user:password@host:5432/tpch
uri=oracle://user:password@host:1521/orclpdb1
uri=netezza://user:password@host:5480/system
uri=db2://user:password@host:50000/TPCH
uri=teradata://user:password@host/SALES
```

- Driver options go next to `uri=`, separated by `;`, e.g.
  `uri=sqlserver://host/?database=tpch;adbc.arpemssql.trusted=true` for integrated authentication.
- SqlClient keywords such as `Trusted_Connection=Yes` or `Data Source=` are not understood
  (`Unknown database option`).
- Special characters of a password must be percent-encoded in a `uri=` (`@` → `%40`, `:` → `%3A`,
  `/` → `%2F`). Prefer the usual flags when the password comes from a shell variable.
- `Driver=` is optional: without it, FastTransfer uses the driver it bundles for the connection type.
- `arpemssql.bulk_*` options (`arpemssql.bulk_tablock`, `arpemssql.bulk_parallel`…) in an `adbc_mssql`
  target string reach the load and win over FastTransfer's own choice (below).

### Licence

The drivers need an **ARPE ADBC DRIVER** licence block, in addition to the FastTransfer one.

- When the `--license` file holds that block next to the `FASTTRANSFER` one (one file, blocks in any
  order), FastTransfer hands it to the drivers: nothing else to set.
- A licence given to the driver directly takes precedence: `License=<file>` in the connection string, or
  the `ARPEIO_ADBC_LICENCE_FILE` environment variable.
- With no ADBC block anywhere, an `adbc_*` run stops with exit code 99.

### Target table and names

- The target table must exist, as for the native targets, unless `--loadmode Create` is used (1.1+).
- `--targetschema` and `--targettable` are used **exactly as typed** (quoted identifiers): they must
  match the stored names, case included: `ORDERS` on Oracle, Db2 and Teradata, `orders` on PostgreSQL,
  unless the table was created with quoted names in another case.
- A name that is not found stops the run before anything is truncated or loaded.

### What differs from the native types

| Topic | With `adbc_*` |
|---|---|
| Mixing with a native type | Refused before anything runs (so `--loadmode Truncate` empties nothing) |
| `--useworktables` | Refused with an ADBC target: the parallel streams load the target table directly |
| `--batchsize` | No effect (a warning says so): the drivers size their own batches |
| `--mapmethod Name` | Works; target columns with no source column stay NULL |
| Parallel methods | All of them. `Physical` for `adbc_pgsql`, `adbc_oracle`, `adbc_mssql`, `adbc_netezza`; `DBPartition` for `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_teradata`, `adbc_db2` (see parallelism.md) |
| Parallel load into `adbc_mssql` | A target with a clustered index (rowstore or columnstore) is loaded without TABLOCK so the streams run together; a heap keeps TABLOCK. A heap with a nonclustered index loads one stream at a time |
| Connection retry | A connection that cannot be made is tried again, up to 4 times over 10 seconds; a refused logon is not retried |
| Timeout | None by default. `FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES`, when set, becomes the drivers' timeout on both ends; a timeout already in the connection string is kept |
| TLS (`adbc_pgsql`, `adbc_netezza`) | Negotiated by default. `FASTTRANSFER_ADBC_SSLMODE` (1.1+) sets the drivers' `sslmode`, see below |
| Parallel load into `adbc_db2` (1.1+) | Each stream commits every 50,000 rows. An `adbc.arpedb2.ingest.commit_interval_rows` already in the connection string is kept |
| `adbc_db2` load request size (1.1.1+) | ArpeDb2 0.4.4 sends up to 32,767 rows per request (`adbc.arpedb2.bulk.chunk_rows`, 1,000 before), so the driver's 1 MiB request limit sets the size: about 25% faster ingest. `adbc.arpedb2.bulk.chunk_rows=1000` in the connection string restores the previous size |
| `adbc_db2` database code page (1.1.1+) | A Db2 LUW database created with a single-byte code set is read and written: CCSID 923 (ISO-8859-15), 850, 1252 and 5348, converted to and from UTF-8 (ArpeDb2 0.4.4). 1.1.0 refuses it: *column … VARCHAR has CCSID 923, which this driver cannot decode*. Any other non-Unicode CCSID is still refused, naming the column |
| Oracle server warnings (1.1.1+) | ArpeOracle 0.4.9 reports the warnings the server sends (password in its grace period ORA-28002, `CREATE` compiled with errors ORA-24344…); they appear in FastTransfer's log like the other driver messages |
| Driver messages | In FastTransfer's log (all sinks), labelled with the connection type, e.g. `WARN adbc_db2 driver: …`; a repeated message goes to Debug. `arpeio.adbc.warnings_to_stderr=true` in the connection string also prints the raw output on stderr |

### TLS for PostgreSQL and Netezza (1.1+)

`adbc_pgsql` and `adbc_netezza` negotiate TLS by default, which fails against a Netezza server without
TLS or with an old TLS version only. The environment variable `FASTTRANSFER_ADBC_SSLMODE` sets the
drivers' `sslmode`: `disable`, `prefer`, `require`, `verify-ca` or `verify-full` (case-insensitive).

- It applies to both ends, with the usual flags and with connection strings alike, and to every
  connection of the run (parallel streams included).
- Any other value stops the run before it connects.
- An `sslmode` already in the connection string is kept (`adbc.arpenz.sslmode=`,
  `adbc.arpepgsql.sslmode=`, or `?sslmode=` in the `uri=`). On 1.0, the connection string is the only
  place to set it.
- It does not apply to `adbc_mssql` (always encrypted, server certificate trusted) nor to `adbc_oracle`
  (`adbc.arpeoracle.encryption` / `data_integrity`).

```powershell
$env:FASTTRANSFER_ADBC_SSLMODE = "disable"     # e.g. a Netezza server without TLS
```

### Data types

The values travel as Arrow data, which differs from the native types in a few places:

- SQL Server `datetimeoffset`, Oracle `TIMESTAMP WITH TIME ZONE` and Teradata time zone offsets arrive as
  the same instant in UTC (offset `+00:00`).
- SQL Server `time(7)` and `datetime2(7)` are cut to microseconds unless
  `adbc.arpemssql.timestamp_precision=nanosecond` is set (dates between 1677 and 2262 only).
- A PostgreSQL target converts a value whose type differs from its column (an Oracle `NUMBER` into
  `bigint`, text into `integer` or `date`). A value that does not fit (`3000000000` into `integer`) stops
  the load with an error naming the column and row. Decimals are rounded to the column's scale. Text
  into a non-text column makes that load a text-format `COPY`, slower than the binary one.
- PostgreSQL `infinity` / `-infinity` dates and timestamps are copied exactly between two `adbc_pgsql`
  ends. Towards another database the load stops with an error naming the column: map them in the source
  query, e.g. `NULLIF(valid_to, 'infinity')`. A distribution key holding `infinity` cannot be used with
  `Ntile`, `DataDriven` or `Timepartition`.

## Backup sources (`backup_mssql`, `backup_rman`, 1.1+)

FastTransfer reads a table straight from backup files, without restoring them and without a running
SQL Server or Oracle instance. The drivers (ArpeMSBackup, ArpeOraBackup) are bundled. They return Arrow
data, like the ADBC sources, so **the target must be an `adbc_*` type** (any of the six); another target
is refused before anything runs. Windows x64 and Linux x64 only. The `--license` file needs the ARPE
ADBC DRIVER block, as for any run with an `adbc_*` target.

| Type | Reads |
|---|---|
| `backup_mssql` | SQL Server full and differential backups (`.bak`): compressed, striped, encrypted or TDE ones included |
| `backup_rman` | Oracle RMAN backup sets: full, level 0 and incremental level 1 to 4 |

### Flags

The usual source flags, with these meanings:

| Flag | Meaning for a backup source |
|---|---|
| `--sourceserver` | Directory of the backup files |
| `--sourcedatabase` | File names or patterns in that directory, separated by `;` |
| `--sourceschema`, `--sourcetable` | Table to read |
| `--query` | `SELECT * FROM [schema.]table` or `SELECT col, col... FROM [schema.]table`; `backup_mssql` also takes `TOP n`. No other SQL: no `WHERE`, no join |
| `--sourceuser`, `--sourcepassword` | `backup_mssql`, encrypted backup only: the private key file of the certificate (`.pvk`, `.pfx`, PEM…) and its password |
| `--sourceconnectstring` | Driver options, given directly (`adbc.arpemsbackup.*` / `adbc.arpeorabackup.*`). An option given there wins over the environment variables |

Which files to give in `--sourcedatabase`:

- `backup_mssql`: the full backup and its differentials, and every stripe, e.g.
  `"mydb_full.bak;mydb_diff_*.bak"`.
- `backup_rman`: `*` and `?` in a name, `**` for any number of directories, and a directory stands for
  its files, e.g. `"l0;l1/**"`.
- An entry that matches no file stops the run, naming it.

### Parallel read

A backup has no SQL engine, so the methods that send a query do not apply. Two values of
`--parallelmethod` are accepted:

| Value | What it does |
|---|---|
| `None` | One stream |
| `Physical` (resolved to `Pages`) | Each of the `--degree` streams reads one slice of the table: pages of a rowstore, rowgroups of a columnstore, ranges of the Oracle blocks |

Any other method stops the run: `… method is not available with source type backup_mssql: a backup has
no SQL engine to split it ==> use --method None or Physical`.

### Which version of the data is read

The **latest version the files hold**: full backup plus differential, or level 0 plus incrementals.

- A backup that could be the newest and cannot be used stops the run, naming it, and FastTransfer logs
  what can be done.
- The backup is opened and the table looked up **before any pre-load step**: a refused backup, or a
  table the backup does not hold, stops the run with the target untouched, `--loadmode Truncate`
  included.
- The log gives the settings: `Backup version to read : restore point latest, restore mode strict`.

| Environment variable | Values | Effect |
|---|---|---|
| `FASTTRANSFER_BACKUP_RESTORE_MODE` | `strict` (default), `best_effort` | `best_effort` reads the newest usable version instead of stopping: an older version of the table can be read without failing the transfer. A warning says so; the backups skipped are reported on the standard error output only, not in the FastTransfer logs |
| `FASTTRANSFER_BACKUP_RESTORE_POINT` | `latest` (default), `full` | `full` reads the last full (or level 0) backup and ignores the differential and incremental ones |

Any other value stops the run before anything is read. The driver options `restore_mode` and
`restore_point` (`adbc.arpemsbackup.…` / `adbc.arpeorabackup.…`) in `--sourceconnectstring` do the same
and win over the variables.

### The backup index

On the first read, the driver builds an index of each backup and saves it (minutes on a large compressed
backup); the next reads reuse it and start at once.

- Default directory: under `%LOCALAPPDATA%` on Windows, `$XDG_CACHE_HOME` or `~/.cache` on Linux.
- `FASTTRANSFER_INDEX_DIR` chooses another directory. It must be a fully qualified path.
- FastTransfer warns when the directory cannot be written (`Backup index not saved: …`): every run then
  indexes the backup again. Set `FASTTRANSFER_INDEX_DIR` to a writable directory (typical for a service
  account with no usable home directory).
- `backup_mssql` with 1.1.1 (ArpeMSBackup 0.5.0): the backup is indexed with unbuffered reads, and a
  compressed backup in one pass, block tables and pages together (a 36 GB compressed backup in 6 stripes
  was indexed in 5.5 s). Two environment variables of the driver go back to the 1.1.0 behaviour:
  `ARPEMSBACKUP_DIRECT_IO=0` reads through the file cache, `ARPEMSBACKUP_SINGLE_PASS=0` indexes in two
  passes.
- `adbc.arpemsbackup.index_threads_per_backup_file` (in `--sourceconnectstring`, 2 by default, from 1
  to 16) sets how many threads read the block table of each compressed backup file. In 1.1.1 it only
  applies with `ARPEMSBACKUP_SINGLE_PASS=0`.

### Driver options in a connection string

`--sourceconnectstring` replaces the source connection flags, so it also names the files, with the
driver's own options separated by `;`:

```
--sourceconnectstring "adbc.arpemsbackup.path=D:\MSSQLBACKUP;adbc.arpemsbackup.file=tpch_full.bak;adbc.arpemsbackup.index_threads_per_backup_file=4"
```

`path` is the directory (`--sourceserver`), `file` the file names or patterns (`--sourcedatabase`);
`certificate` and `certificate_password` stand for `--sourceuser` / `--sourcepassword`, and `index_dir`
for `FASTTRANSFER_INDEX_DIR`. Prefer the usual flags when no driver option is needed. With 1.1.1 the
thread count of this example only takes effect with `ARPEMSBACKUP_SINGLE_PASS=0` (see above).

### Limits

- The target must be an `adbc_*` type.
- Only the `None` and `Physical` parallel methods.
- A query cannot filter or join: it selects columns of one table.
- Not available on Linux ARM64.
- `--loadmode Create` with a backup source is not tested in 1.1.0 (release notes).

## Credentials without leaking them

FastTransfer reads credentials from its flags only. Keep secrets in shell variables (or a secret store)
and reference them:

```bash
export FT_SOURCE_PASSWORD='...'
./FastTransfer ... --sourceuser "etl" --sourcepassword "$FT_SOURCE_PASSWORD" ...
```

```powershell
.\FastTransfer.exe ... --sourceuser "etl" --sourcepassword $env:FT_SOURCE_PASSWORD ...
```

Passwords given with `--sourcepassword` / `--targetpassword` or inside a connection string are masked in
console, file and database logs.

## Source query dialects (1.0+)

FastTransfer parses the source `--query` (and `--datadrivenquery`) with the SQL rules of the source
engine. Since 1.0 this includes Db2 and Oracle:

| Source | What parses |
|---|---|
| Db2 (`adbc_db2`, and `odbc` when the driver reports a Db2 server) | Special registers in several words (`CURRENT DATE`, `CURRENT TIMESTAMP(6)`, `CURRENT SCHEMA`…), labeled durations (`CURRENT DATE - 30 DAYS`, `ORDER_DATE + 1 MONTH`), `FETCH FIRST n ROWS ONLY` |
| Oracle (`oraodp`, `adbc_oracle`, and `odbc` when the driver reports an Oracle server) | `#` in a name (`SELECT SID, SERIAL# FROM V$SESSION`), a backslash as an ordinary character (`'C:\temp\'`), the `(+)` outer join, `IGNORE NULLS`, `T@DBLINK`, `CONNECT BY`, `ROWNUM` |

Since 1.1, a query read with `Ntile`, `Random`, `RangeId`, `DataDriven` or a physical method keeps every
clause it holds (`QUALIFY`, a named `WINDOW`, `CONNECT BY`) when the method adds its predicate. 1.0 and
older lost such a clause, so the same query can return different rows after an upgrade.

On an Oracle source, a `PARTITION (P1)` clause in the `FROM` of `--query` fails at parse time: filter
with a `WHERE` on the partition key instead, or transfer the whole table with
`--parallelmethod DBPartition`.

## Special sources from the docs

- **ODBC DSN** (e.g. MySQL ODBC, or Db2 with IBM's driver when the target is not `adbc_*`):
  `--sourceconnectiontype "odbc" --sourcedsn "MyDsn"`, plus `--sourceuser`/`--sourcepassword` when the DSN
  does not hold the credentials.
- **DuckDB database file**: `--sourceconnectiontype "duckdb" --sourceserver "D:\data\db.duckdb"`.
- **Tableau Hyper file through the PostgreSQL protocol** (Hyper API server running):
  `--sourceconnectiontype "pgsql" --sourceserver "localhost:8095" --sourcedatabase "D:\path\file.hyper" --sourceuser "tableau_internal_user"`.
- **Files** (CSV, Parquet, JSON): see [files.md](files.md).
- **Backup files** (SQL Server `.bak`, Oracle RMAN, 1.1+): see "Backup sources" above.
