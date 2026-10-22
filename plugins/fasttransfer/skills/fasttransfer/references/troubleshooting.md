# FastTransfer – Troubleshooting and pitfalls

## First steps when a run fails

1. Re-run with `--loglevel debug` (or `verbose`) and a recognisable `--runid`.
2. Check `--version`. `DBPartition`, `Timepartition` and `FASTTRANSFER_DATEFORMAT` need 0.18; the env
   vars `FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES` and `FASTTRANSFER_SORTEDINPUT` need 0.17.
3. Test with a small `--query` (e.g. `TOP 1000` / `LIMIT 1000` / `FETCH FIRST 1000 ROWS ONLY`, in the
   source dialect) and `--parallelmethod None`.
4. Exit code 0 means success; any non-zero code is a failure. The reason is in the log.

## Common mistakes

| Symptom / mistake | Fix |
|---|---|
| "Failed parsing parameters" | A flag name or value is wrong. Check the exact spelling: `--sourceconnectstring` (not `--sourceconnectionstring`), `--degree` (not `--paralleldegree`), and allowed values (e.g. `--targetconnectiontype msbulk`, not `mssql`) |
| Target table does not exist | FastTransfer never creates tables. Create the table first with compatible column types |
| Columns land in the wrong target columns, or types fail | The default mapping is by **position**. Use `--mapmethod Name` when the column order differs |
| "Set a Distribute Key Column when using … method" | `Random`, `Ntile`, `RangeId`, `DataDriven`, `Timepartition` need `--distributekeycolumn` |
| `DBPartition` or `Timepartition` rejected | Needs 0.18. On 0.17 use `DataDriven` (see [parallelism.md](parallelism.md)) |
| `DBPartition` with `--query` / `--fileinput` | Not allowed. Give `--sourceschema` + `--sourcetable`; to filter rows, use another method |
| `DBPartition` logs a warning and runs single-stream | The table is not partitioned, has one partition, or the source type is not supported: it fell back to `None`. Use `Timepartition` or `Ntile` |
| `Timepartition`: the source can't convert the date bounds | The bounds are sent as `yyyy-MM-dd` text: set `FASTTRANSFER_DATEFORMAT=yyyyMMdd` if the source needs that |
| `Timepartition` stops with an error before the transfer starts | `FASTTRANSFER_DATEFORMAT` has a value other than `yyyy-MM-dd` or `yyyyMMdd` |
| *"Cannot bulk load. The bulk data stream was incorrectly specified as sorted…"* (`msbulk`, partitioned target with a rowstore clustered index, `DataDriven`) | Fixed in 0.18 (the hint is no longer sent there): upgrade. On 0.18, set `FASTTRANSFER_SORTEDINPUT=true` only when the source really is sorted on the partition column |
| `pgcopy` target, non-PostgreSQL source: NULL dates fail with *"invalid input syntax"*, NULL text loaded as empty strings | Fixed in 0.18: upgrade |
| Vulnerability scanner flags `System.Security.Cryptography.Pkcs` 9.0.13 (CVE-2026-84008, CVE-2026-70162) | Replace the binary with 0.18.1. Updating the server's .NET runtime doesn't help (self-contained binary) |
| "Set a Source Schema and Source Table when using … method" | `Ctid`, `Rowid`, `NZDataSlice` need `--sourceschema` + `--sourcetable`, even with `--query` |
| "Ctid method can be used only with Postgres source" (same for Rowid / Physloc / NZDataSlice) | Method and source type don't match. See [parallelism.md](parallelism.md) |
| Inconsistent rows with `Ctid` from a `pgcopy` source into a `pgcopy` target while the table is written (0.18.0 and older) | Fixed in 0.18.1 (one exported snapshot for all chunks). Upgrade, or avoid writes during the transfer |
| Query containing the word `COPY` (e.g. a table `COPYTABLE`) truncated (0.18.0 and older) | Fixed in 0.18.1 |
| Missing or duplicated rows with `Physloc` | The source was written during the transfer. Use `Physloc` only on read-only data or snapshots, otherwise use `Ntile` |
| `--sourcetrusted` together with `--sourceuser` | Not allowed. Use integrated auth or user/password, not both (same on the target side) |
| `--query` together with `--fileinput` | Not allowed. Use one of them |
| `nzcopy` source type rejected | Listed in the docs as WIP, but not accepted by the 0.17 CLI. Use `nzsql` |
| Bulk load times out after 60 minutes (`msbulk`, `orabulk`) | Set `FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES` higher |
| Slow `msbulk` load into a table with a clustered index | If the source is sorted on that index, set `FASTTRANSFER_SORTEDINPUT=true`. Tune `--batchsize` |
| Problems loading into a clustered columnstore table | Try `--useworktables` |
| Run fails although the source and target are fine | A database log sink in `FastTransfer_Settings.json` is unreachable. Fix it or remove that sink |
| Settings file ignored on Linux | The name is case-sensitive: `FastTransfer_Settings.json`, or pass `--settingsfile` |
| Database unreachable from Docker with `localhost` | Use `host.docker.internal` or the host IP |
| `$PARTITION…` broken in PowerShell | Escape `$` with a backtick inside double quotes |
| Double quotes inside `--query` | Double them: `"select * from ""tpch_10"".orders"` |
| Wants cloud files (`s3://`, `gs://`…) | Not documented for FastTransfer 0.18. Download the files first |
| Wants to export to files | That's FastBCP, not FastTransfer |

## Facts checked against the 0.17 source code

The docs were corrected to match these, but older copies (and other sites) may still differ:

- Negative `--degree` **divides** the cores: `cores / abs(N)`. The default is `-4`.
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
