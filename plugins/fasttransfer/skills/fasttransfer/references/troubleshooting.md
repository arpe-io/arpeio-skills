# FastTransfer – Troubleshooting and pitfalls

## First steps when a run fails

1. Re-run with `--loglevel debug` (or `verbose`) and a recognisable `--runid`.
2. Check `--version`. The `adbc_*` types and `Physical` need 1.0; `DBPartition`, `Timepartition` and
   `FASTTRANSFER_DATEFORMAT` need 0.18; the env vars `FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES` and
   `FASTTRANSFER_SORTEDINPUT` need 0.17.
3. Test with a small `--query` (e.g. `TOP 1000` / `LIMIT 1000` / `FETCH FIRST 1000 ROWS ONLY`, in the
   source dialect) and `--parallelmethod None`.
4. Exit code 0 means success; any non-zero code is a failure, with the reason in the log. 99 = no or
   invalid license (or no ADBC driver block for an `adbc_*` run), 100 = license expired.

## Common mistakes

| Symptom / mistake | Fix |
|---|---|
| "Failed parsing parameters" | A flag name or value is wrong. Check the exact spelling: `--sourceconnectstring` (not `--sourceconnectionstring`), `--degree` (not `--paralleldegree`), and allowed values (e.g. `--targetconnectiontype msbulk`, not `mssql`). `adbc_*` and `Physical` are rejected this way before 1.0 |
| Target table does not exist | FastTransfer never creates tables. Create the table first with compatible column types |
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
| Wants cloud files (`s3://`, `gs://`…) | Not documented for FastTransfer 1.0. Download the files first |
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
| Parallel load into an `adbc_db2` target fails at volume with *SQLCODE -911 SQLSTATE 40001* | Lock escalation between the streams. Use `--parallelmethod None`, or a `--targetconnectstring` with `adbc.arpedb2.ingest.commit_interval_rows=50000` |
| Time zone offsets differ from the source (SQL Server `datetimeoffset`, Oracle `TIMESTAMP WITH TIME ZONE`, Teradata) | They arrive as the same instant in UTC (`+00:00`) |
| Driver warnings no longer on stderr | They go to FastTransfer's log. Add `arpeio.adbc.warnings_to_stderr=true` to the connection string to also get the raw output |

## Known limitations in 1.0

- **Native `pgcopy` target from a source that is not `pgcopy`**: some values are written wrong without an
  error. Fractional seconds are lost on `time` and `timestamp`, `timestamptz` can be shifted by the
  server time zone, `bytea` arrives as the text `System.Byte[]`, `inet` loses its prefix, `interval` and
  `infinity` values change. `pgcopy` → `pgcopy` and `adbc_pgsql` → `adbc_pgsql` copy these exactly. For
  tables holding such columns, use those pairs (or `adbc_*` on both ends from another engine) and compare
  the values after the load.
- **Native PostgreSQL source (`pgsql`, `pgcopy`) into `orabulk`** fails on a `date` column (*Unable to
  cast object of type 'System.DateOnly' to type 'System.IConvertible'*). `adbc_pgsql` → `adbc_oracle` works.
- **Db2 and Teradata as `adbc_*` targets**, known limitations: an Oracle `NUMBER` into a Db2 `BIGINT`,
  Db2 `DECFLOAT` columns, Teradata `PERIOD` columns, Teradata load speed on large tables, and parallel
  loads into Db2 at volume (table above). Test such tables on a sample and compare the values.
- **Parallel `orabulk` loads** can fail with *ORA-00054* (direct path lock on the target table). Lower
  `--degree`, or use `adbc_oracle` on both ends.
- **No ADBC driver for Linux ARM64**, and no native Db2 type: on Linux ARM64, Db2 goes through `odbc`
  as a source only.

## Facts checked against the source code

The docs were corrected to match these, but older copies (and other sites) may still differ:

- Negative `--degree` **divides** the cores: `cores / abs(N)`. The default is `-4` (the `-?` usage text
  and the README shipped in the 1.0 zip still say `1`).
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
