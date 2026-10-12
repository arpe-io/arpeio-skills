# FastTransfer – Troubleshooting and pitfalls

## First steps when a run fails

1. Re-run with `--loglevel debug` (or `verbose`) and a recognisable `--runid`.
2. Check `--version`. The env vars `FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES` and `FASTTRANSFER_SORTEDINPUT`
   need 0.17.
3. Test with a small `--query` (e.g. `TOP 1000` / `LIMIT 1000` / `FETCH FIRST 1000 ROWS ONLY`, in the
   source dialect) and `--parallelmethod None`.
4. Exit code 0 means success; any non-zero code is a failure. The reason is in the log.

## Common mistakes

| Symptom / mistake | Fix |
|---|---|
| "Failed parsing parameters" | A flag name or value is wrong. Check the exact spelling: `--sourceconnectstring` (not `--sourceconnectionstring`), `--degree` (not `--paralleldegree`), and allowed values (e.g. `--targetconnectiontype msbulk`, not `mssql`) |
| Target table does not exist | FastTransfer never creates tables. Create the table first with compatible column types |
| Columns land in the wrong target columns, or types fail | The default mapping is by **position**. Use `--mapmethod Name` when the column order differs |
| "Set a Distribute Key Column when using … method" | `Random`, `Ntile`, `RangeId`, `DataDriven` need `--distributekeycolumn` |
| "Set a Source Schema and Source Table when using … method" | `Ctid`, `Rowid`, `NZDataSlice` need `--sourceschema` + `--sourcetable`, even with `--query` |
| "Ctid method can be used only with Postgres source" (same for Rowid / Physloc / NZDataSlice) | Method and source type don't match. See [parallelism.md](parallelism.md) |
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
| Wants cloud files (`s3://`, `gs://`…) | Not documented for FastTransfer 0.17. Download the files first |
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
