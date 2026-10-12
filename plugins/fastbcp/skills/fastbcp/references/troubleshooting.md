# FastBCP – Troubleshooting, pitfalls and doc inconsistencies

## First steps when a run fails

1. Re-run with `--loglevel Debug` and a recognisable `--runid`.
2. Check `--version` against the features used. `DBPartition`, `adbc_oracle`, `adbc_netezza`,
   `FASTBCP_DBPARTITION_TARGET_MODE` and `AZURE_STORAGE_SAS_URL` need 1.3.
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
| `adbc_mssql` fails on `SQL_VARIANT` | Not supported by the ADBC connector. Use `mssql`, or cast the column in a `--query` |
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
