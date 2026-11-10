# FastBCP – Troubleshooting, pitfalls and doc inconsistencies

## First steps when a run fails

1. Re-run with `--loglevel Debug` and a recognisable `--runid`.
2. Check `--version` against the features used. `backup_mssql` needs 1.5, differential backups 1.6.7; `Physical`, `adbc_teradata`
   and `FASTBCP_TTU_PATH` need 1.4; `DBPartition`, `adbc_oracle`, `adbc_netezza`, `FASTBCP_DBPARTITION_TARGET_MODE` and
   `AZURE_STORAGE_SAS_URL` need 1.3.
3. Test the connection with a tiny query, e.g. `--query "SELECT 1 AS x"` with method `None`.
4. Treat any non-zero exit code as a failure. The docs only document the license codes (1.5+):
   **99** (no license, invalid signature, missing ADBC block) and **100** (expired past grace period).

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
| Exit code 99 after upgrading to 1.5, run worked before | 1.5 has no build-date trial: a license file is required. With `adbc_*` / `backup_mssql`, the file also needs an `ARPE ADBC DRIVER LICENCE` block (or set `ARPEIO_ADBC_LICENCE` / `ARPEIO_ADBC_LICENCE_FILE`) |
| `FastBCP.lic` found in an unexpected place (1.5+) | The current directory is searched before the executable's directory. Pass `--license` explicitly in scheduled jobs |
| `<method> method is not available with connection type backup_mssql` | Use `--parallelmethod None` or `Physical` |
| `backup_mssql` refuses the backup | QAT_DEFLATE compression, a file with no full backup, or an incomplete stripe set (the message names the missing stripes). Before 1.6.6: a TDE database or `WITH ENCRYPTION` too. Since 1.6.7: see the rows below, and "Upgrading from 1.6.6" in connections.md |
| An export that worked with 1.6.6 fails on a backup (1.6.7+) | A backup that could be the newest and cannot be used now fails the export. The warning after the error gives the ways out; the table is in connections.md. Fix the files first: `FASTBCP_BACKUP_RESTORE_MODE=best_effort` and `FASTBCP_BACKUP_RESTORE_POINT=full` can read older rows |
| `… holds no full database backup for set N: this differential (finished …) needs the full backup …, which is not among the files given` (1.6.7+) | Give the full backup too, in `--database`, in the same `--server` directory |
| `…, set N: incomplete (no end of set): the backup was interrupted or the file truncated` (1.6.7+) | Copy the file again, or wait for it. A full backup alone that only lost its end: `FASTBCP_BACKUP_RESTORE_POINT=full` reads it, `best_effort` cannot as it has no other version. Check the file first |
| `…, set N: a file, filegroup or partial backup: it does not hold the whole database` (1.6.7+) | Give a full backup. In a `WITH NOINIT` file after a full backup, `FASTBCP_BACKUP_RESTORE_POINT=full` reads as 1.6.6, `best_effort` every table from the full backup, so older rows |
| `…: not a backup` (1.6.7+) | The pattern matches another file: narrow `--database` |
| `… is locked by another process: a backup still being written?` (1.6.7+) | Wait for the end of the backup, or leave that file out of `--database` |
| `… are backups of a database and of a restored copy of it …: name the files of one` (1.6.7+) | Narrow `--database` to the files of one database |
| `no usable backup in the files given: …` (1.6.7+) | No version can be read, with any setting: give a full backup that can be used |
| `… is a complete backup on its own, but N files were given: name one backup` (1.6.6 and older, or `FASTBCP_BACKUP_RESTORE_POINT=full`) | One backup only with these. Since 1.6.7, without that variable, several backups are read at their latest version |
| `FASTBCP_BACKUP_RESTORE_MODE must be one of strict\|best_effort` / `FASTBCP_BACKUP_RESTORE_POINT must be one of latest\|full` (1.6.7+) | Fix the value of the variable: the run stops at startup |
| An export reads older rows than expected (1.6.7+) | `FASTBCP_BACKUP_RESTORE_MODE=best_effort` skipped a newer backup: see the standard error output, as the FastBCP logs do not list it. Or `FASTBCP_BACKUP_RESTORE_POINT=full` is set, or the differential is not in `--database`. The logs do not name the files read: name exactly the two files to be sure |
| A `WITH NOINIT` file with differentials gives other rows after upgrading (1.6.7+) | Expected: its newest differential is now read. `FASTBCP_BACKUP_RESTORE_POINT=full` gives the last full backup, as 1.6.6 |
| `[ArpeMSBackup] …: its data cannot be read without the private key of certificate 0x…` (1.6.6+) | Encrypted or TDE backup: give the key file with `--sourceuser`. The thumbprint is the one `sys.certificates` shows on the server |
| `[ArpeMSBackup] …: the private key given does not open it: it needs the private key of certificate 0x…` | Not the right certificate. A backup `WITH ENCRYPTION` of a TDE database can need two: separate the files with `;` |
| `[ArpeMSBackup] …: the private key is protected by a password` | Give it with `--sourcepassword`. If a non-ASCII PVK password is reported as wrong, export the key with an ASCII password |
| `… holds a certificate without its private key` | `--sourceuser` names a `.cer` / `.crt` with no `.pvk`, `.pfx`, `.p12`, `.key` or `.pem` of the same name next to it |
| `page … is under the previous database encryption key` | Uncompressed backup of a TDE database taken while its key was being changed. Take it again after the change, or `WITH COMPRESSION` |
| `truncated or damaged backup` (1.6.6+) | The data of the backup ends past the end of the file: copy it again. It failed before too, later in the export |
| An export hangs on a damaged or crafted `.bak` (1.6.5 and older) | Fixed in 1.6.6: it now stops with an error |
| `Backup index not saved: …` (1.6.6+) | A warning, the export still runs, but every run indexes the backup again. Set `FASTBCP_INDEX_DIR` to a writable directory, or change the `index_dir` option if `--sourceconnectstring` sets one |
| `FASTBCP_INDEX_DIR` stops the run at startup (Windows) | The path must be fully qualified: `C:\index` or `\\server\share\index`, not `C:index` or `\index` |
| `backup_mssql` refuses the table: `json` or `vector(n, float16)` column (1.6.5+) | List the other columns in `--query`, or use `adbc_mssql` |
| `backup_mssql` with CSV output, a `WHERE` in `--query`, or credentials | Parquet only; `--query` is limited to `SELECT * \| columns FROM [schema.]table`; no `--trusted`, and `--user` / `--password` are the certificate key of an encrypted backup, not a login |
| `[ArpeMSBackup] catalog of …: catalog page (x:y) not in the backup` (1.5.0) | Fixed in 1.5.1 (ArpeMSBackup 0.1.7): the driver followed a stale pointer to the system tables. Upgrade; the backup itself is fine |
| `[ArpeMSBackup] … data id … outside the dictionaries of column …` (1.5.0) | Fixed in 1.5.1: clustered columnstore segments with a secondary dictionary (large tables, e.g. TPC-H `lineitem`). Upgrade |
| `backup_mssql` export differs from the live table | The backup is read as its pages are: the log in the backup is not replayed, a differential taken during writes included, and log backups are not applied. Before 1.6.7, differentials are not applied either |
| `<method> method is not available with connection type backup_rman` | Use `--parallelmethod None` or `Physical` |
| `backup_rman` refuses the backup | Encrypted backup set (`SET ENCRYPTION`), an incomplete multi-piece set (the message names the missing piece), a non-default block size tablespace, or a big-endian platform's backup. Up to 1.6.7, an incremental level 1 set too (`is an incremental level 1 backup`): upgrade to 1.6.8 and give the whole chain, or give the full or level 0 pieces only |
| `… its parent is not among the files given: a backup of this file with … is needed` (`backup_rman`, 1.6.8+) | An incremental backup without its level 0, or without a level in between: add the missing backups to `--database` (`"l0/*;l1/**"`, or `"**"` for every subdirectory of `--server`). `FASTBCP_BACKUP_RESTORE_MODE=best_effort` reads the version before the hole, `FASTBCP_BACKUP_RESTORE_POINT=full` the level 0 only: both give older rows |
| `Backup files: no file matches '…' in …` / `'…' is not a file or a directory in …` (`backup_rman`, 1.6.8+) | An entry of `--database` matches nothing: check its spelling and the directory. Every entry must match at least one file. Up to 1.6.7, any entry with a `/` gave `no file matches`: upgrade, or name the pieces in `--server` only |
| `Backup files: … has a ';' in its path` (`backup_rman`, 1.6.8+) | `;` separates the entries: rename the file or directory |
| `Backup files : cannot list …: the backups below it, if any, are not read` (warning, 1.6.8+) | A subdirectory reached by `**` cannot be read by the account running FastBCP: grant it read access, or leave it out |
| An export that worked with 1.6.7 fails on an RMAN backup (1.6.8+) | A backup set without its last piece, or a table truncated then its tablespace backed up alone, is now refused instead of read wrong. Give the whole set; see "Upgrading from 1.6.7" in connections.md |
| `backup_rman`: a level 1 fails a table, naming a block that is not in the backup (1.6.8+) | The level 1 was taken under heavy writes: the next level 1, taken at a quieter time, reads it |
| `backup_rman` refuses the table | Partitioned / IOT / clustered / object / temporary / external table, or a LOB / `INTERVAL` / `ROWID` / `JSON` / `VECTOR` column (the message names it): export the other columns with `--query "SELECT col1, col2 FROM OWNER.TABLE"` |
| `backup_rman`: table not found | Give every piece of the backup (`--database "db_*;ctl_*"`, quoted); a table of the root or of another PDB is `CONTAINER.OWNER.TABLE` in `--query` |
| `backup_rman` export differs from the live table | An online backup is read as its blocks are (redo not applied). Since 1.6.8, the latest version the files give is read: check that the newest incremental backups are in `--database`, and that `FASTBCP_BACKUP_RESTORE_POINT=full` or `best_effort` is not set (what `best_effort` skipped goes to the standard error output). Up to 1.6.7, only the full or level 0 backup is read |
| A scheduled run on Windows is much slower than in a console (1.5.1 and older) | Windows 11 keeps programs without a window on the efficiency cores. Fixed in 1.6.0 (FastBCP asks for full speed); `FASTBCP_ECOQOS=1` restores the old behaviour |
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
| `Physical` + `--query` on `backup_mssql` | The Physical method says a `--query` may add a simple `WHERE`; the `backup_mssql` pages limit it to `SELECT … FROM [schema.]table` with no filter | No filter with `backup_mssql` |
| `backup_mssql` example on the Connection Parameters page | `./FastBCP` with a Windows path (`D:\Backups\MSSQL`) | Match the executable and path style to the OS |

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
- **1.5**: `backup_mssql` (SQL Server `.bak` files without restore, ArpeMSBackup 0.1.6) and the `Pages`
  strategy behind `Physical`; a license file always required (no build-date trial, no embedded ADBC trial
  token), `FastBCP.lic` looked up in the current directory first, multi-block license files, ADBC license
  forwarded inline, exit codes 99 / 100; ArpeMSSQL 0.7.8 (connections spaced 50 ms, faster 4-way reads),
  ArpeOracle 0.4.3, ArpeTeradata 0.3.5 (`uri` / `connection_string`).
- **1.5.1**: ArpeMSBackup 0.1.7, two `backup_mssql` fixes: backups whose system tables have a stale
  first-page pointer (`catalog page (x:y) not in the backup`), and clustered columnstore tables with
  secondary dictionaries (`data id … outside the dictionaries`). No option or behaviour change.
- **1.6.0**: `backup_rman` (Oracle RMAN backup sets without restore or Oracle, ArpeOraBackup 0.1.0,
  AdbcFileExport 0.5.6): uncompressed and BASIC / LOW / MEDIUM / HIGH compressed, Oracle 19c, 21c, 23ai,
  26ai; `Physical` resolves to `Pages`. On Windows, FastBCP opts out of the efficiency cores at start
  (`FASTBCP_ECOQOS=1` keeps the Windows policy). Large striped SQL Server backups ready in seconds.
- **1.6.1**: `backup_rman` refuses incremental level 1 sets. **1.6.0 read them as full backups, with no error
  and only the changed rows**: export such tables again from the full or level 0 backup.
- **1.6.2**: `datetime2(7)` and `datetimeoffset(7)` as microseconds by default, so the whole SQL Server date
  range is exported (`backup_mssql`, `adbc_mssql`); `backup_mssql` columnstore and accented path fixes.
- **1.6.3**: `backup_mssql` exports up to 2.8× faster, backups opened again in milliseconds, columnstore fixes.
- **1.6.4**: `backup_rman` backups opened again in milliseconds; **`LONG` / `LONG RAW` values longer than a block
  and `TIMESTAMP WITH LOCAL TIME ZONE` values came out wrong from 1.6.0 to 1.6.3**: export them again. Exact
  unsigned 64-bit integers from HFSQL (OLE DB) and MySQL.
- **1.6.5**: `backup_mssql` writes the same Parquet as `adbc_mssql` (`geography`, `geometry`, `vector`,
  `time(7)`, `sql_variant` change: check the pipelines reading them); far less memory on wide clustered
  columnstore tables; `json` and `vector(n, float16)` refused; `FASTBCP_UID_STRING_CASING` applies.
- **1.6.8**: `backup_rman` reads incremental backups, levels 0 to 4, differential and cumulative: the latest version
  of each datafile from a level 0 and the incremental backups on it (ArpeOraBackup 0.3.1). `--database` takes paths,
  relative to `--server` or absolute, and `**` for any number of subdirectories; each entry must match a file.
  `FASTBCP_BACKUP_RESTORE_POINT` / `FASTBCP_BACKUP_RESTORE_MODE` apply to `backup_rman`. **A level 0 and its
  incremental backups now read fresher rows, and a set without its last piece, read short before, now fails**: see
  "Upgrading from 1.6.7" in connections.md. Also fixed: a column added with a `DEFAULT` read `NULL` on older rows;
  archived log pieces in the directory, more than 64 pieces and `COPIES 2` are read.
- **1.6.7**: `backup_mssql` reads the latest version from full and differential backups (ArpeMSBackup 0.2.0);
  `FASTBCP_BACKUP_RESTORE_POINT`, `FASTBCP_BACKUP_RESTORE_MODE`. **A backup that could be the newest and cannot be
  used now fails the export, and a `WITH NOINIT` file with differentials reads fresher rows**: see "Upgrading from
  1.6.6" in connections.md. Fixed: the end of some compressed backups was lost without an error, on about 2 % of
  the backups tried (no page lost on 168 test backups); a file or partial backup taken for a full backup. Each
  backup reads its structure once more on the first export.
- **1.6.6**: `backup_mssql` reads encrypted and TDE backups with the certificate's private key (`--sourceuser`,
  `--sourcepassword`, ArpeMSBackup 0.1.17); a damaged `.bak` no longer hangs; `FASTBCP_INDEX_DIR` and a warning
  when the index of a backup cannot be saved (ArpeOraBackup 0.1.3).
