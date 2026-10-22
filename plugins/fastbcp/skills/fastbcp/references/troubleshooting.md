# FastBCP – Troubleshooting, pitfalls and doc inconsistencies

## First steps when a run fails

1. Re-run with `--loglevel Debug` and a recognisable `--runid`.
2. Check `--version` against the features used. `Physical`, `adbc_teradata` and `FASTBCP_TTU_PATH`
   need 1.4; `DBPartition`, `adbc_oracle`, `adbc_netezza`, `FASTBCP_DBPARTITION_TARGET_MODE` and
   `AZURE_STORAGE_SAS_URL` need 1.3.
3. Test the connection with a tiny query, e.g. `--query "SELECT 1 AS x"` with method `None`.
4. Treat any non-zero exit code as a failure (the docs list no specific return codes).

## Common mistakes

| Symptom / mistake | Fix |
|---|---|
| CSV/JSON requested with an `adbc_*` connector | ADBC writes Parquet only. Use the non-ADBC type (`mssql`, `pgsql`/`pgcopy`, `oraodp`, `nzsql`) |
| `adbc_*`, Teradata, Netezza or HANA fail on Linux ARM64 | Not supported on ARM64. Use an x64 host |
| `oledb` on Linux | OLE DB is Windows only. Use `odbc` |
| `--merge true` with an `s3://`, `gs://`, `abs://`, `abfss://` or `onelake://` target | Merge is not available for cloud targets. Use `--merge false` |
| Merge with JSON / XLSX / BSON | Merge only works for CSV and Parquet |
| Query file fails | `--fileinput` must hold exactly one query, **without a trailing `;`** |
| Tokens `{sourceschema}`/`{sourcetable}` empty or rejected | Pass `--sourceschema` and `--sourcetable`, even with `--query` |
| `-d` used for the database | `-d` is the CSV delimiter. Database is `--database` (`-I`) |
| `Timepartition` key rejected | Use the format `"(datecol, year, month)"`, with parentheses and the column first |
| `Random` gives few or unbalanced chunks | The key must be integer/bigint with at least as many distinct values as the degree. Prefer `Ntile` |
| `DBPartition` runs single-threaded | The table is not partitioned, has one partition, or the source is unsupported (it falls back to `None` with a warning) |
| `$partition…` expression broken in PowerShell | Escape `$` with a backtick inside double quotes |
| DB unreachable from Docker with `localhost` | Use `host.docker.internal` or the host IP |
| Azure 403 | Firewall, RBAC role (e.g. Storage Blob Data Contributor) or SAS permissions |
| Unexpected cloud storage cost | Interrupted multipart uploads. Add a lifecycle rule that deletes incomplete multipart uploads |
| High memory with Parquet | Lower `FASTBCP_RGSIZE` and/or `--paralleldegree` |
| ADLS / OneLake upload errors or slowness | Try `FASTBCP_CLOUD_ISBUFFERED=True` (+ `FASTBCP_TMPPATH` on a large disk) |
| Run fails although the export is fine | A database log sink in `FastBCP_settings.json` is unreachable. Fix it or remove that sink |
| `adbc_pgsql` fails on NUMERIC `NaN` (1.3) | Add `adbc.arpepgsql.numeric_special=null` to `--sourceconnectstring` |
| NPS 7.x refuses the TLS connection with `adbc_netezza` | Add `tls_min_version=1.0` to the connection string |
| `adbc_mssql` fails on `SQL_VARIANT` (before 1.4) | Upgrade to 1.4 (ArpeMSSQL 0.7.6 reads it), use `mssql`, or cast the column in a `--query` |
| `Physical method is not supported for connection type …` | `Physical` works only on SQL Server, PostgreSQL, Oracle and Netezza connection types. Use `Ntile`/`RangeId`/`DataDriven` (or `DBPartition`) |
| `Physical method cannot identify the source engine behind connection type 'odbc'` | Use the native connection type (`mssql`, `pgsql`, `oraodp`, `nzsql`… or `adbc_*`) |
| `With Physical (…) Method you must define source schema … and source table` | Add `--sourceschema` and `--sourcetable` (1.4 also requires them for `Physloc` and `Rowid`) |
| `Parallel method Ctid is deprecated …` warning | Replace `Ctid`/`Rowid`/`Physloc`/`NZDataSlice` with `Physical`. If a log parser treats warnings as failures, update the command, not the parser |
| `Physloc` run: one chunk does all the work (before 1.4) | Known bug with an even `--paralleldegree`. Upgrade to 1.4, or use an odd degree |
| `-C adbc_teradata does not support trusted (-A) authentication` | Give `--user` and `--password` |
| `adbc_teradata` logon fails with `DBCHINI failed`, or on the legacy port | The TTU folder is incomplete or not versioned (`ttu/20.00`, not `ttu/`), or not found: set `FASTBCP_TTU_PATH` (Linux). Or let the server use TLS (port 443), which needs no TTU |
| `adbc_oracle` fails on TCPS / Kerberos / token logon after upgrading to 1.4 | ArpeOracle 0.4.1 verifies the server certificate, needs `krb5_forward=true` to forward the TGT, and requires TCPS for token / OAuth2 |
| A script checking `--help`'s exit code breaks after 1.4 | `--help` now exits with 0 (it was 1) |
| Dates lost their time part, or gained one, after an upgrade | The default date format changed in 1.1. Set `--dateformat` explicitly |

Parquet type mappings differ between connectors (e.g. `adbc_mssql` vs `mssql`). Check the docs'
"Datatypes Conversion" pages before switching connector on an existing pipeline:
https://fastbcp-docs.arpe.io/latest/datatypes-conversion/overview

## Inconsistencies in the docs (how this skill resolves them)

| Topic | What the docs say | Rule used here |
|---|---|---|
| `--merge` default | Parallel Parameters page says both `false` and `true`; 0.32 release notes: switched to `false` | Default `false`. Always set it explicitly |
| Merge on cloud | "Not available for cloud destinations", yet one GCS example uses `--merge "True"` | Never merge on cloud targets |
| Schema/table with parallel methods | "Mandatory" on the Source Info page, but many examples use only `--query` | Set both whenever the table is known |
| Short flags | Some pages use `-d` for database, `-Q` for query, `-f` for settings file | Generate long flags only |
| `Rowid` vs `RowId` | Both spellings in examples | Use `Rowid` |
| Flag aliases | `--method`, `--degree`, `--licence` in some examples | Use `--parallelmethod`, `--paralleldegree`, `--license` |
| `--port` | Used in many examples, absent from the parameter page | Accepted; for PostgreSQL `host:port` in `--server` also works |
| Connection types `hanaexport`, `nzoledb`, `nzcopy` | In examples / release notes, not in the connection type table | Use only if the user asks for them; mention they are not in the main table |
| Line continuation | A few Linux snippets mix `` ` `` and `\` | bash: `\`; PowerShell: `` ` `` |

## Version notes

- **0.32**: `--merge` default switched from `true` to `false`; `.csv.gz` output added.
- **1.1**: default `--dateformat` became `yyyy-MM-dd HH:mm:sszzz` (was `yyyy-MM-dd`); more
  `--boolformat` values.
- **1.2**: `odbc` + `--dsn` now honours `--user`/`--password` (before, it fell back to integrated
  auth); `adbc_mssql` supports up to 1024 columns (was < 256).
- **1.3**: `DBPartition` + `FASTBCP_DBPARTITION_TARGET_MODE`; `adbc_oracle`, `adbc_netezza`; ADBC used
  end to end (metadata + data); Oracle ADBC encryption negotiation; **Ntile could lose rows before 1.3**
  (last bucket upper bound), so re-check older Ntile exports; `adbc_mssql` named instances now
  resolved through SQL Server Browser; NZDataSlice uses a shared work queue.
- **1.4**: `Physical` parallel method (deprecates `Ctid`, `Rowid`, `Physloc`, `NZDataSlice`, removed in
  2.0); `adbc_teradata` + `FASTBCP_TTU_PATH`; `Physloc`/`Rowid` require `--sourceschema`/`--sourcetable`;
  `Physloc` fixed for even degrees (1.6–2.6× faster) and clustered columnstore; **`Ntile` could drop rows
  with sub-millisecond timestamp keys before 1.4**, and a source `WHERE` with `OR` could export extra rows
  with any parallel method, so compare row counts of older exports; `Timepartition` exports `NULL` dates
  to `<col>=NULL`, no more week duplicates, biggest periods first; `--help` exits with 0; ArpeOracle 0.4.1
  stricter security defaults; `adbc_mssql` reads `sql_variant` and SQL Server 2025 `JSON`.
