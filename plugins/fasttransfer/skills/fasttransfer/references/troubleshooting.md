# FastTransfer – Troubleshooting and pitfalls

## First steps when a run fails

1. Re-run with `--loglevel debug` (or `verbose`) and a recognisable `--runid`.
2. Check `--version`. `--loadmode Create`, `--inferdatatype`, the backup sources (`backup_mssql`,
   `backup_rman`) and `FASTTRANSFER_ADBC_SSLMODE` need 1.1; the `adbc_*` types and `Physical` need 1.0;
   `DBPartition`, `Timepartition` and
   `FASTTRANSFER_DATEFORMAT` need 0.18; the env vars `FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES` and
   `FASTTRANSFER_SORTEDINPUT` need 0.17.
3. Test with a small `--query` (e.g. `TOP 1000` / `LIMIT 1000` / `FETCH FIRST 1000 ROWS ONLY`, in the
   source dialect) and `--parallelmethod None`.
4. Exit code 0 means success; any non-zero code is a failure, with the reason in the log. 99 = no or
   invalid license (or no ADBC driver block for an `adbc_*` run), 100 = license expired.

## Common mistakes

| Symptom / mistake | Fix |
|---|---|
| "Failed parsing parameters" | A flag name or value is wrong. Check the exact spelling: `--sourceconnectstring` (not `--sourceconnectionstring`), `--degree` (not `--paralleldegree`), and allowed values (e.g. `--targetconnectiontype msbulk`, not `mssql`). `adbc_*` and `Physical` are rejected this way before 1.0; `backup_mssql`, `backup_rman`, `--loadmode Create` and `--inferdatatype` before 1.1. Since 1.1 an empty or stray argument also ends here (it crashed before) |
| Target table does not exist | Create the table first with compatible column types, or let FastTransfer create it with `--loadmode Create` (1.1+, see [parameters.md](parameters.md)). Before 1.1 FastTransfer never creates tables |
| After an upgrade to 1.1, a script with `--loadmode Replace` empties the target table | `Replace` is an alias of `Create` since 1.1: it creates the table, or **truncates** it when it exists. Up to 1.0 it behaved like `Append`. Write `Append` to keep the rows |
| Columns land in the wrong target columns, or types fail | The default mapping is by **position**. Use `--mapmethod Name` when the column order differs |
| "Set a Distribute Key Column when using … method" | `Random`, `Ntile`, `RangeId`, `DataDriven`, `Timepartition` need `--distributekeycolumn` |
| `DBPartition` or `Timepartition` rejected | Needs 0.18. On 0.17 use `DataDriven` (see [parallelism.md](parallelism.md)) |
| `DBPartition` with `--query` / `--fileinput` | Not allowed. Give `--sourceschema` + `--sourcetable`; to filter rows, use another method |
| `DBPartition` logs a warning and runs single-stream | The table is not partitioned, has one partition, or the source type is not supported: it fell back to `None`. Use `Timepartition` or `Ntile` |
| `Timepartition`: the source can't convert the date bounds | The bounds are sent as `yyyy-MM-dd` text: set `FASTTRANSFER_DATEFORMAT=yyyyMMdd` if the source needs that |
| `Timepartition` stops with an error before the transfer starts | `FASTTRANSFER_DATEFORMAT` has a value other than `yyyy-MM-dd` or `yyyyMMdd` |
| *"Cannot bulk load. The bulk data stream was incorrectly specified as sorted…"* (`msbulk`, partitioned target with a rowstore clustered index, `DataDriven`) | Fixed in 0.18 (the hint is no longer sent there): upgrade. On 0.18+, set `FASTTRANSFER_SORTEDINPUT=true` only when the source really is sorted on the partition column |
| `pgcopy` target, non-PostgreSQL source: NULL dates fail with *"invalid input syntax"*, NULL text loaded as empty strings | Fixed in 0.18: upgrade |
| Vulnerability scanner flags `System.Security.Cryptography.Pkcs` 9.0.13 (CVE-2026-84008, CVE-2026-70162) | Replace the binary with 0.18.1 or later. Updating the server's .NET runtime doesn't help (self-contained binary) |
| "Set a Source Schema and Source Table when using Physical (Ctid) method even if you define a query" (also `Physical (Rowid)`, `Physical (Physloc)`, or the bare engine name) | `Physical` needs `--sourceschema` + `--sourcetable` on PostgreSQL, Oracle and SQL Server, even with `--query`. New in 1.0 for SQL Server (`Physloc` took a query alone before) |
| "Physical method is not supported for source connection type …" | No physical split for that engine (MySQL, Teradata, HANA, ClickHouse, Db2, DuckDB). Use `DBPartition`, `Ntile`, `RangeId` or `DataDriven` |
| "Physical method cannot identify the source engine behind connection type 'odbc'" | Use a native or `adbc_*` type; for SQL Server through `odbc`, the deprecated `Physloc` is still accepted |
| "Parallel method Ctid is deprecated and will be removed in 2.0 ==> use --parallelmethod Physical" (warning) | Replace `Ctid` / `Rowid` / `Physloc` / `NZDataSlice` with `Physical`. The run is not affected |
| "Ctid method can be used only with Postgres source" (same for Rowid / Physloc / NZDataSlice) | A deprecated engine name on the wrong source type. Write `Physical`. See [parallelism.md](parallelism.md) |
| Inconsistent rows with `Ctid` from a `pgcopy` source into a `pgcopy` target while the table is written (0.18.0 and older) | Fixed in 0.18.1 (one exported snapshot for all chunks). Upgrade, or avoid writes during the transfer |
| Query containing the word `COPY` (e.g. a table `COPYTABLE`) truncated (0.18.0 and older) | Fixed in 0.18.1 |
| Missing or duplicated rows with `Physical` on SQL Server (`Physloc`) | The source was written during the transfer. Use it only on read-only data or snapshots, otherwise use `Ntile` |
| SQL Server `Physloc` with an even `--degree` no faster than one stream (0.18 and older) | The former split key left a single active stream. Fixed in 1.0; before, use `Ntile` or `RangeId` |
| `--useworktables` with `msbulk` or `pgcopy`: the log shows *"Fail Merge Data"*, the target stays empty, exit code 0 (0.18 and older) | **Data is lost silently.** Fixed in 1.0, where a failed merge fails the run. Before 1.0, don't use `--useworktables` with these targets |
| `Ntile` on a date / timestamp key, or `Ntile` / `DataDriven` on a key holding NULLs: fewer rows in the target than in the source, exit code 0 (0.18 and older) | **Rows are left out silently** (rows on a chunk boundary, and all NULL-key rows). Fixed in 1.0. Before, use a numeric NOT NULL key (`RangeId`, `Ntile`), and compare row counts |
| Oracle source (`oraodp`), `Ntile`: *ORA-00932 inconsistent datatypes: expected NUMBER got DATE*; `Ntile` / `DataDriven` on a `DATE` key: *ORA-01861* / *ORA-01843*; PostgreSQL `pgsql` source, `Ntile` on a `date`: *Unable to cast object of type 'System.DateOnly'* (0.18 and older) | Fixed in 1.0. Before, use `RangeId` on a numeric key, or `Rowid` / `Ctid` |
| Rows with a NULL key missing with `RangeId` or `Random` | By design: a range or a modulo never matches NULL. Use a NOT NULL key, or `Ntile` / `DataDriven` (1.0+ transfer the NULL-key rows) |
| Oracle `--query` holding `#` (e.g. `SERIAL#`) cut at the `#` (0.18 and older) | Oracle queries are parsed with Oracle rules since 1.0. Upgrade |
| Oracle `--query` with `FROM t PARTITION (P1)` fails at parse time (1.0+) | Filter with a `WHERE` on the partition key, or transfer the whole table with `DBPartition` |
| Db2 `--query` with `CURRENT DATE`, `CURRENT TIMESTAMP(6)` or `CURRENT DATE - 30 DAYS` fails to parse (0.18 and older, `odbc`) | Db2 rules apply since 1.0 (`adbc_db2`, and `odbc` on a Db2 server). Upgrade |
| `--query` with `QUALIFY`, a named `WINDOW` or `CONNECT BY`, read with `Ntile`, `Random`, `RangeId`, `DataDriven` or a physical method: the clause is lost and the rows differ (1.0 and older) | Fixed in 1.1, which keeps every clause of the query. Such a query returns other rows after the upgrade: the ones the query asks for |
| `Ntile` on an Oracle (`oraodp`) or MySQL source: fails with a `--query`, splits the rows wrongly with a table (1.0 and older) | Fixed in 1.1. Before, use `RangeId` on a numeric key, or `Physical` on Oracle |
| SQL Server: *"the tcp: prefix is not accepted (the connection is TCP anyway)…"* or *"a port after a named instance is written with a comma…"* (1.1+) | Drop `tcp:`, and write `host\INSTANCE,port`. Accepted forms: `host`, `host:port`, `host,port`, `host\INSTANCE`, `host\INSTANCE,port` |
| SQL Server native types (`mssql`, `msbulk`, `msoledbsql`) with `host:port`: the connection hangs or fails after the connect timeout (1.0 and older) | `host:port` is accepted since 1.1. Before, write `host,port` |
| `pgcopy` target, non-`pgcopy` source: fractional seconds lost, binary values loaded as the text `System.Byte[]`, a column with an upper case or spaced name not found (1.0 and older) | Fixed in 1.1. The time zone shift of `timestamptz` is not (see "Known limitations in 1.1") |
| `--sourcetrusted` together with `--sourceuser` | Not allowed. Use integrated auth or user/password, not both (same on the target side) |
| `--query` together with `--fileinput` | Not allowed. Use one of them |
| `nzcopy` source type rejected | Not a CLI value. Use `nzsql` (or `nzbulk`, work in progress) |
| Bulk load times out after 60 minutes (`msbulk`, `orabulk`) | Set `FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES` higher |
| Slow `msbulk` load into a table with a clustered index | If the source is sorted on that index, set `FASTTRANSFER_SORTEDINPUT=true`. Tune `--batchsize` |
| Problems loading into a clustered columnstore table (`msbulk`) | Try `--useworktables`, **on 1.0+ only** (see above) |
| Run fails although the source and target are fine | A database log sink in `FastTransfer_Settings.json` is unreachable. Fix it or remove that sink |
| Settings file ignored on Linux | The name is case-sensitive: `FastTransfer_Settings.json`, or pass `--settingsfile` |
| Database unreachable from Docker with `localhost` | Use `host.docker.internal` or the host IP |
| `$PARTITION…` broken in PowerShell | Escape `$` with a backtick inside double quotes |
| Double quotes inside `--query` | Double them: `"select * from ""tpch_10"".orders"` |
| Wants cloud files (`s3://`, `gs://`…) | Not documented for FastTransfer 1.1. Download the files first |
| Wants to export to files | That's FastBCP, not FastTransfer |

## ADBC (`adbc_*`, 1.0+)

| Symptom / mistake | Fix |
|---|---|
| "ADBC connection types (adbc_*) cannot be mixed with other types: source is …, target is …" | Use `adbc_*` on both ends, or on neither. Nothing was truncated or loaded |
| "ADBC connection types (…) are available on Linux x64 and Windows x64 only" | Linux ARM64 build: use the native types there, or run the x64 build |
| "-W (--useworktables) is not available with an ADBC target" | Remove `--useworktables`: the parallel streams load the target table directly |
| Warning "-B (--batchsize) … has no effect with an ADBC target" | Remove `--batchsize`; the drivers size their own batches |
| Exit code 99 with "Licence : …" on an `adbc_*` run | The `--license` file has no ARPE ADBC DRIVER block. Use a file holding both blocks, or `License=<file>` in the connection string, or `ARPEIO_ADBC_LICENCE_FILE` |
| Target table not found although it exists | `--targetschema` / `--targettable` are case-sensitive with `adbc_*`: `ORDERS` on Oracle, Db2 and Teradata, `orders` on PostgreSQL |
| "Unknown database option" (e.g. `data source`, `Trusted_Connection`) | The connection string is in ADO.NET vocabulary. Use the usual flags (`--sourceserver`…), or `uri=…` with driver options (connections.md) |
| Logon fails with a password holding `@`, `:` or `/` in a `uri=` | Percent-encode it (`%40`, `%3A`, `%2F`), or use `--sourceuser` / `--sourcepassword` |
| `--sourcetrusted` / `--targettrusted` refused on Netezza | No integrated logon for `adbc_netezza`: use user and password |
| Load stops on a PostgreSQL `infinity` date or timestamp, error naming the column | Only two `adbc_pgsql` ends copy them. Map them in the query, e.g. `NULLIF(valid_to, 'infinity')` |
| "The distribution key … holds a PostgreSQL infinity date or timestamp" | Use another key, `RangeId` on a numeric key, or `None` |
| Load into a PostgreSQL target stops with an error naming a column and a row | The value does not fit the column (e.g. `3000000000` into `integer`). Widen the column or cast in the query |
| Parallel load into an `adbc_mssql` heap with a nonclustered index no faster than one stream | Expected: the streams wait on each other there. A heap without index, or a clustered index, loads in parallel |
| Parallel load into an `adbc_db2` target fails at volume with *SQLCODE -911 SQLSTATE 40001* (1.0) | Lock escalation between the streams. Fixed in 1.1, where each stream commits every 50,000 rows. On 1.0, use `--parallelmethod None`, or a `--targetconnectstring` with `adbc.arpedb2.ingest.commit_interval_rows=50000` |
| `adbc_netezza` or `adbc_pgsql` cannot connect to a server without TLS, or with an old TLS version only | The drivers negotiate TLS by default. 1.1+: set `FASTTRANSFER_ADBC_SSLMODE` (`disable`, `prefer`, `require`, `verify-ca`, `verify-full`). 1.0: put the driver's `sslmode` in the connection string (`adbc.arpenz.sslmode=`, `adbc.arpepgsql.sslmode=`) |
| "FASTTRANSFER_ADBC_SSLMODE='…' is not a valid sslmode" | Use one of `disable`, `prefer`, `require`, `verify-ca`, `verify-full`. The run stopped before connecting |
| Time zone offsets differ from the source (SQL Server `datetimeoffset`, Oracle `TIMESTAMP WITH TIME ZONE`, Teradata) | They arrive as the same instant in UTC (`+00:00`) |
| Driver warnings no longer on stderr | They go to FastTransfer's log. Add `arpeio.adbc.warnings_to_stderr=true` to the connection string to also get the raw output |
| `adbc_db2` source or target: *column … has CCSID 923, which this driver cannot decode* (also 850, 1252, 5348) | The Db2 database uses a single-byte code page. Upgrade to 1.1.1 (ArpeDb2 0.4.4), which converts these four. Another CCSID is still refused |
| `adbc_db2`: *DRDA connect failed: login timed out after 30 s (adbc.arpedb2.login_timeout)*, now and then, on a server that answers | The database is not activated: Db2 shuts it down when its last connection closes, and the next connection waits for it to start again (seen between two runs, and between FastTransfer's planning connection and the next one). Have the DBA run `db2 activate db <name>` (it stays active until `db2stop` or `db2 deactivate db`) |
| New `WARN adbc_oracle driver: warning: ORA-28002 …` (or ORA-24344) lines in the log after an upgrade to 1.1.1 | ArpeOracle 0.4.9 reports the server's warnings, which 1.1.0 dropped. The run is not affected; ORA-28002 means the password expires soon |

## Creating the target table (`--loadmode Create`, 1.1+)

| Symptom / mistake | Fix |
|---|---|
| Warning "--inferdatatype is only used with -L Create: ignored" (same for a `FASTTRANSFER_INFER_*` variable) | Add `--loadmode Create`, or drop the option: it does nothing with `Append` or `Truncate` |
| "Target table … was not found, but … exists: names are used exactly as typed…" | The table exists in another case. Write `--targetschema` / `--targettable` as stored. Nothing was created |
| The existing target table was emptied | `Create` truncates a table that exists and keeps its structure. Use `Append` to add rows |
| The existing table keeps its old column types | By design: when the table exists nothing is inferred and nothing is altered. Drop it to have it created again |
| The load fails on a value longer than its column | A value is never truncated. With `fast` / `allvarchar`, a text column with no declared length gets 256 characters: raise `FASTTRANSFER_INFER_DEFAULT_STRING_MAXSIZE` (or `max`). With `best`, add `FASTTRANSFER_INFER_STRING_MARGIN` |
| "A row of the source does not fit a type inferred by --inferdatatype best…" | A row written after the scan, or missed by a sample. The table created by that run was dropped: run again with a full scan (unset `FASTTRANSFER_INFER_SAMPLE_PERCENT`) or with `--inferdatatype fast` |
| "Infer Data Type best : the profiling of the source failed…" | Nothing was created. Run again with `--inferdatatype fast` to skip the profiling |
| A failed run leaves the new table empty or partly loaded | Expected. The next run finds it, truncates it and loads it |
| SQL Server to SQL Server (`mssql` → `msbulk`): the table is created, then *The locale id '1033' of the source column '…' and the locale id '1036' of the destination column '…' do not match* | The created text columns take the target database's collation, and its locale differs from the source column's. Use `adbc_mssql` on both ends (the load works, the columns keep the target's collation), or create the table with the source collation on its text columns (`COLLATE SQL_Latin1_General_CP1_CI_AS`…) and load with `--loadmode Truncate`. Drop the empty table the failed run left first |
| A later `--loadmode Append` fails on larger values (table created with `best`) | The types fit the data read at creation time. Widen the columns, or create the table with `FASTTRANSFER_INFER_STRING_MARGIN` |
| "--inferdatatype allvarchar needs a source whose engine is known…" | Not available with a generic `odbc` or `oledb` source: use a native or `adbc_*` source type |
| "The select list of the query (-q / -f) cannot be rewritten for the … method…" | With `Physical` and `allvarchar` / `best`, the query must be a single `SELECT` listing its columns, or `SELECT *` on one table. Or give `--sourceschema` + `--sourcetable` without a query |
| "The MySQL target is not in strict mode…" | Set a strict `sql_mode` (`STRICT_TRANS_TABLES` or `STRICT_ALL_TABLES`) on the MySQL target. Nothing was created |
| MySQL target: the table is not created (text columns above 65,535 bytes per row) | Create it yourself, with `TEXT` columns where needed, and load it with `--loadmode Truncate` |
| "FASTTRANSFER_INFER_…='…' is not valid…" | The run stopped before any connection. `…_DEFAULT_STRING_MAXSIZE`: an integer from 1, or `max`; `…_STRING_MARGIN`: a number of characters or a factor `2x` to `10x`; `…_SAMPLE_PERCENT`: above 0 and up to 100 |

## Backup sources (`backup_mssql`, `backup_rman`, 1.1+)

| Symptom / mistake | Fix |
|---|---|
| "Source type backup_mssql reads a SQL Server backup file (ArpeMSBackup), which returns Arrow batches: the target must be an ADBC type…" | Use an `adbc_*` target (`adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_db2`, `adbc_teradata`). Nothing ran |
| "… method is not available with source type backup_…: a backup has no SQL engine to split it" | Only `--parallelmethod None` or `Physical` |
| `--query` with a `WHERE` or a join refused | A backup source reads `SELECT * FROM [schema.]table` or a column list only (`TOP n` too with `backup_mssql`) |
| An entry of `--sourcedatabase` matches no file | The run stops, naming it. Check the directory in `--sourceserver` and the names or patterns |
| The run stops naming a backup that cannot be used (a differential without its full backup, a missing stripe or piece, a backup still being written…) | By design, rather than reading older data without saying so; the target is untouched, `Truncate` included. Give the missing files, wait for the backup, or leave it out of `--sourcedatabase`. `FASTTRANSFER_BACKUP_RESTORE_MODE=best_effort` reads the newest usable version and `FASTTRANSFER_BACKUP_RESTORE_POINT=full` the last full (or level 0) backup: both can read older data |
| Table not found in the backup | The run stops before any pre-load step. Check `--sourceschema` / `--sourcetable` and the files given |
| Encrypted SQL Server backup | `--sourceuser` = the private key file of the certificate (`.pvk`, `.pfx`, PEM…), `--sourcepassword` = its password |
| The first read of a backup takes minutes before any row | The driver builds the index of the backup; the next reads reuse it |
| Warning "Backup index not saved: …" and every run indexes the backup again | The index directory cannot be written. Set `FASTTRANSFER_INDEX_DIR` to a writable directory (or fix `adbc.arpemsbackup.index_dir` / `adbc.arpeorabackup.index_dir` when the connection string sets it) |
| "FASTTRANSFER_INDEX_DIR must be a fully qualified path…" | Give the full path, with its drive on Windows |
| "FASTTRANSFER_BACKUP_RESTORE_POINT must be one of …" / "FASTTRANSFER_BACKUP_RESTORE_MODE must be one of …" | `latest` or `full` for the point, `strict` or `best_effort` for the mode |
| Backup source on Linux ARM64 | Not available: Windows x64 and Linux x64 only |

## Known limitations in 1.1

- **Native `pgcopy` target from a source that is not `pgcopy`**: `timestamptz` can be shifted by the
  server time zone, without an error. 1.1 fixed the fractional seconds and the binary values (`bytea`
  arrived as the text `System.Byte[]`); 1.0 and older also write those wrong. `inet` prefixes, `interval`
  and `infinity` values, which 1.0 changes, are not among the fixes listed for 1.1. `pgcopy` → `pgcopy`
  and `adbc_pgsql` → `adbc_pgsql` copy these exactly. For tables holding such columns, use those pairs
  (or `adbc_*` on both ends from another engine) and compare the values after the load.
- **Native PostgreSQL source (`pgsql`, `pgcopy`) into `orabulk`** fails on a `date` column (*Unable to
  cast object of type 'System.DateOnly' to type 'System.IConvertible'*). `adbc_pgsql` → `adbc_oracle` works.
- **Db2 and Teradata as `adbc_*` targets**, known limitations, unchanged from 1.0: an Oracle `NUMBER`
  into a Db2 `BIGINT`, Db2 `DECFLOAT` columns, Teradata `PERIOD` columns, Teradata load speed on large
  tables. Test such tables on a sample and compare the values. (Parallel loads into Db2 at volume failed
  in 1.0, see the ADBC table above; 1.1 commits every 50,000 rows on each stream.)
- **Parallel `orabulk` loads** can fail with *ORA-00054* (direct path lock on the target table). Lower
  `--degree`, or use `adbc_oracle` on both ends.
- **No ADBC driver for Linux ARM64**, and no native Db2 type: on Linux ARM64, Db2 goes through `odbc`
  as a source only. The backup sources load into `adbc_*` targets only, so they are not available there
  either.
- **Time zone offsets** from SQL Server, Oracle and Teradata arrive as the same instant in UTC with
  `adbc_*` (unchanged from 1.0).
- **`--loadmode Create`**:
  - a text column with no declared length gets 256 characters, and a longer value fails the load (raise
    `FASTTRANSFER_INFER_DEFAULT_STRING_MAXSIZE`). XML, JSON and arrays are unbounded text;
  - SQL Server refuses NaN and infinity in `REAL` / `FLOAT`, and keeps 7 fractional digits of a
    nanosecond timestamp;
  - a MySQL `TIME` outside 00:00:00 to 23:59:59 fails on a target `TIME` column; a MySQL target table
    above the 65,535-byte row limit is refused;
  - a DuckDB target refuses a `time` value; `mysqlbulk` refuses a `YEAR` target column;
  - `adbc_oracle` reads the date 0001-01-01 as 0000-12-30.
- **`--loadmode Create` from SQL Server to SQL Server with `msbulk`** between databases whose collations
  have different locales: the load fails after the table is created (see "Creating the target table"
  above). `adbc_mssql` on both ends works.
- **`Physical` on an Oracle source** needs `SELECT_CATALOG_ROLE`; without it the run fails after the
  target was truncated, and the target is left empty. A backup source checks before truncating; an
  Oracle source does not.
- **Not tested against a real database in 1.1.0**: `--loadmode Create` with ClickHouse, Netezza or SAP
  HANA (source or target), with Teradata as a target, and with the backup sources; `NZDataSlice`
  (`Physical` on Netezza) with a `--query`. Test on a sample first, and say so to the user.

## Facts checked against the source code

The docs were corrected to match these, but older copies (and other sites) may still differ:

- Negative `--degree` **divides** the cores: `cores / abs(N)`. The default is `-4` (the `-?` usage text
  and the README shipped in the 1.1 zip still say `1`).
- `--mapmethod` defaults to `Position`.
- `pgcopy` target: binary COPY only when the source is also `pgcopy`; text (CSV) COPY otherwise.
- `--parallelmethod` and `--method` are both accepted, and so are `--distributekeycolumn` and the legacy
  `--distributeKeyColumn`. Generate the first form of each.
- `--loglevel` accepts `verbose`, `debug`, `information`, `warning`, `error` (no `fatal`).

## Version notes

- **0.16**: `--version` added; SAP HANA driver included on Linux. Last release line supporting .NET 7 (0.16.2).
- **0.17**: runs on .NET 10; new env vars `FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES` and `FASTTRANSFER_SORTEDINPUT`.
- **0.18.0**: `DBPartition` and `Timepartition` parallel methods; `FASTTRANSFER_DATEFORMAT`; `msbulk`
  sorted hint no longer sent to partitioned rowstore clustered targets (fixes the "incorrectly specified
  as sorted" failure with `DataDriven`); `pgcopy` target loads NULLs from non-PostgreSQL sources correctly.
- **0.18.1** (security update, recommended for all 0.18.0 users): bundles `System.Security.Cryptography.Pkcs`
  10.0.13 (fixes CVE-2026-84008 and CVE-2026-70162, denial of service only; FastTransfer doesn't call the
  vulnerable APIs, but scanners flag them) and .NET runtime 10.0.13; driver updates (SqlClient 7.1.1,
  Oracle 23.26.301, Npgsql 10.0.3, MySqlConnector 2.6.2, DuckDB.NET 1.5.6.1, Teradata 20.0.10); `Ctid`
  with `pgcopy` source and target reads all chunks from one snapshot; queries containing `COPY` no
  longer truncated.
- **0.18.2** (licensing): the license is always validated; only `--help` and `--version` run without
  a license file (exit code 99 otherwise), and the free trial is a 30-day license file. Multi-block license files
  (several products in one file, any order). The default `FastTransfer.lic` is looked up in the working
  directory, then next to the executable. `--help`, `-?` no longer crash after parsing.
- **1.0.0**: ADBC connection types `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_db2`,
  `adbc_teradata` as sources and targets, with the Arpe ADBC drivers bundled (Windows x64, Linux x64).
  `Physical` parallel method; `Ctid`, `Rowid`, `Physloc`, `NZDataSlice` deprecated (removed in 2.0).
  New SQL Server physical split key (all streams active, clustered columnstore handled); `Physloc` now
  requires `--sourceschema` + `--sourcetable`; `NZDataSlice` no longer does and accepts `nzbulk` /
  `nzoledb`; `msoledbsql` accepted for the physical split. `DBPartition` for Db2 for LUW (`adbc_db2`).
  Db2 and Oracle SQL rules for the source query. `Random` works on Db2 and Teradata. Fixes:
  `--useworktables` on `msbulk` / `pgcopy` lost the data with exit code 0; `Ntile` and `DataDriven` on
  date and timestamp keys lost rows between chunks and skipped NULL keys; Oracle `Ntile`.
- **1.1.0**: `--loadmode Create` creates the target table from the source when it does not exist, and
  truncates it when it does; `--inferdatatype allvarchar | fast | best` and the env vars
  `FASTTRANSFER_INFER_STRING_MARGIN`, `FASTTRANSFER_INFER_DEFAULT_STRING_MAXSIZE`,
  `FASTTRANSFER_INFER_SAMPLE_PERCENT`. Backup sources `backup_mssql` (SQL Server `.bak`, full and
  differential) and `backup_rman` (Oracle RMAN backup sets), read without restoring them, to `adbc_*`
  targets; env vars `FASTTRANSFER_INDEX_DIR`, `FASTTRANSFER_BACKUP_RESTORE_MODE`,
  `FASTTRANSFER_BACKUP_RESTORE_POINT`. `FASTTRANSFER_ADBC_SSLMODE` for `adbc_netezza` and `adbc_pgsql`.
  **Behaviour changes**: `--loadmode Replace` is an alias of `Create` (it behaved like `Append`); a
  source query read with `Ntile`, `Random`, `RangeId`, `DataDriven` or a physical method keeps every
  clause it holds (`QUALIFY`, named `WINDOW`, `CONNECT BY`), so a query that relied on a dropped clause
  returns different rows. Fixes: `Ntile` on an Oracle or MySQL source (failed with a query, split the
  rows wrongly with a table); `pgcopy` target (fractional seconds lost, binary values written as
  `System.Byte[]`, column names not quoted; the time zone shift is not addressed); `adbc_db2` target
  commits every 50,000 rows on each stream of a parallel load; an empty or stray command line argument
  is a parsing failure, not a crash; SQL Server `host:port` accepted, `tcp:` and `host\INSTANCE:port`
  refused with a clear message; every missing backup index directory is created private on Linux.
  Bundled drivers: ArpeMSSQL 0.8.4, ArpePGSQL 0.5.6, ArpeOracle 0.4.8, ArpeNetezza 0.3.12, ArpeDb2
  0.4.3, ArpeTeradata 0.3.8, ArpeMSBackup 0.4.1 (`adbc.arpemsbackup.index_threads_per_backup_file`),
  ArpeOraBackup 0.3.1.
- **1.1.1**: no CLI change; three bundled drivers updated. ArpeMSBackup 0.5.0: unbuffered reads to index
  a backup (`ARPEMSBACKUP_DIRECT_IO=0` goes back to the file cache) and one indexing pass for a
  compressed backup (`index_threads_per_backup_file` only with `ARPEMSBACKUP_SINGLE_PASS=0`). ArpeDb2
  0.4.4: databases in CCSID 923, 850, 1252 and 5348 read and written; bulk ingest about 25% faster
  (`adbc.arpedb2.bulk.chunk_rows` 32,767 by default instead of 1,000); a chunk above 2,114 rows no
  longer loses its other rows; reads use less memory and client CPU. ArpeOracle 0.4.9: server warnings
  reported in the log; AES-128 / AES-192 and SHA-384 / SHA-512 for native network encryption; a
  `NUMBER` holding infinity reads as `Infinity` / `+inf` instead of failing the query (a decimal or
  integer target refuses it). Bundled drivers: ArpeMSSQL 0.8.4, ArpePGSQL 0.5.6, ArpeOracle 0.4.9,
  ArpeNetezza 0.3.12, ArpeDb2 0.4.4, ArpeTeradata 0.3.8, ArpeMSBackup 0.5.0, ArpeOraBackup 0.3.1.
