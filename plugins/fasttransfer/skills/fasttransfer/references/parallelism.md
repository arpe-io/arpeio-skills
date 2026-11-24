# FastTransfer – Parallel transfer

FastTransfer splits the source read into chunks. It runs one source query per chunk, each with its own
`WHERE` clause, and bulk loads the chunks into the target table in parallel. Parallelism is worth it only
for large volumes (the docs say more than about 1M cells).

Since 1.1, a `--query` read with `Ntile`, `Random`, `RangeId`, `DataDriven` or a physical method keeps
every clause it holds (`QUALIFY`, a named `WINDOW`, `CONNECT BY`) when the method adds its predicate.
1.0 and older lost such a clause (see troubleshooting.md).

## Parameters

| Flag | Default | Notes |
|---|---|---|
| `--parallelmethod` | `None` | Method, see below |
| `--distributekeycolumn` | – | Column or expression, e.g. `"YEAR(o_orderdate)"`, `"(id/10000)::int"`, `"filename"`. Required by `Random`, `DataDriven`, `RangeId`, `Ntile`, `Timepartition` (a date tuple, see below) |
| `--degree` | `-4` | `N>0`: N threads (if N is more than twice the core count, the core count is used); `0`: all cores; `-N`: cores / N (minimum 1) |
| `--datadrivenquery` | – | `DataDriven` only: query returning the list of values to split on |

## Methods

| Method | Source types | Needs | How it splits |
|---|---|---|---|
| `None` | any | – | Single stream |
| `Physical` (1.0+) | PostgreSQL, Oracle, SQL Server, Netezza; 1.1+: SQL Server and Oracle RMAN backups (table below) | `--sourceschema`, `--sourcetable` (not on Netezza) | Physical row location of the source engine; the strategy is picked from the connection type |
| `Ntile` | any | key | Even buckets with NTILE; the key can be numeric, date, datetime or string. Rows whose key is NULL go to a chunk of their own. On an Oracle or MySQL source, use 1.1+ (see troubleshooting.md) |
| `RangeId` | any | numeric key | Ranges between MIN and MAX; best on an identity / sequence without gaps |
| `DataDriven` | any | key | One chunk per distinct value of the key (NULL is one of them), or per row of `--datadrivenquery`. More values than threads are queued |
| `Random` | any | numeric key | Modulo on the key |
| `DBPartition` (0.18+) | `mssql`, `msoledbsql`, `pgsql`, `pgcopy`, `oraodp`, `mysql`, `teradata`; 1.0+: `adbc_mssql`, `adbc_pgsql`, `adbc_oracle`, `adbc_teradata`, `adbc_db2` | `--sourceschema`, `--sourcetable`; `--query` / `--fileinput` not allowed | One stream per native partition, biggest first (details below) |
| `Timepartition` (0.18+) | any | date tuple `"(col,year[,month[,day]])"` or `"(col,year,week)"` | One stream per year, month, day or ISO week of a date column, biggest first |
| `Ctid`, `Rowid`, `Physloc`, `NZDataSlice` | see "Physical" below | see "Physical" below | **Deprecated in 1.0** (warning in the log, removed in 2.0). Use `Physical`; on 0.18 and older they are the only way to write it |

`RangeId` and `Random` never match a NULL key (a range or a modulo is not true for NULL): use them on a
key without NULLs, or use `Ntile` / `DataDriven`.

Every method works with the `adbc_*` types (1.0+), within the source lists above. A backup source
(`backup_mssql`, `backup_rman`, 1.1+) only takes `None` and `Physical`: a backup has no SQL engine to
run a split query, and any other method stops the run.

## Physical (1.0+)

One value for the physical-location split. FastTransfer resolves it from `--sourceconnectiontype`:

| Source connection type | Resolved to | Notes |
|---|---|---|
| `pgsql`, `pgcopy`, `adbc_pgsql` | `Ctid` | CTID ranges. Recommended for PostgreSQL |
| `oraodp`, `adbc_oracle` | `Rowid` | ROWID slices |
| `mssql`, `msoledbsql`, `adbc_mssql` | `Physloc` | `%%physloc%%`, rowstore and clustered columnstore tables. **Only on read-only tables, databases or snapshots**: physical locations may change during writes |
| `nzsql`, `nzbulk`, `nzoledb`, `adbc_netezza` | `NZDataSlice` | Netezza data slices. With a `--query`: not tested against a real database in 1.1.0 |
| `backup_mssql`, `backup_rman` (1.1+) | `Pages` | The backup driver splits the table into one slice per stream: pages of a rowstore, rowgroups of a columnstore, ranges of the Oracle blocks. A `--query` can only select columns of the table (connections.md) |
| anything else | refused | Error: `Physical method is not supported for source connection type …`; for `odbc` / `oledb`: `Physical method cannot identify the source engine behind connection type …` |

- No `--distributekeycolumn`.
- `--sourceschema` and `--sourcetable` are required for PostgreSQL, Oracle and SQL Server, even when a
  `--query` is given (the query then filters that table). FastTransfer stops otherwise with
  `Set a Source Schema and Source Table when using Physical (Ctid) method even if you define a query`.
  Netezza needs neither.
- The log shows the resolution: `Distribute Method : Physical resolved to Ctid for source type pgsql`.
- No `Physical` for MySQL, Teradata, SAP HANA, ClickHouse, Db2, DuckDB, `odbc` and `oledb`: use
  `DBPartition`, `Ntile`, `RangeId` or `DataDriven`.
- The engine names still work in 1.0 and log `Parallel method Ctid is deprecated and will be removed in
  2.0 ==> use --parallelmethod Physical`. Keep them in two cases only:
  - **0.18 and older**, where `Physical` does not exist. There `Ctid`, `Rowid` and `NZDataSlice` require
    `--sourceschema` + `--sourcetable`, `Physloc` requires nothing and is limited to `mssql` (or `odbc` to
    SQL Server), and `NZDataSlice` to `nzsql`.
  - **An `odbc` source to SQL Server** in 1.0: `Physical` refuses `odbc`, `Physloc` is still accepted
    (with `--sourceschema` + `--sourcetable`).
- Since 1.0 the SQL Server split uses a new key, which spreads the rows over all the streams (the former
  key could leave a single active stream with an even degree) and handles clustered columnstore tables.

## DBPartition (0.18+)

FastTransfer reads the partitions from the catalog and transfers each one in its own stream, biggest first.
No key column. It falls back to `None` (single stream) with a warning when the table is not partitioned,
has a single partition, or the source type is not supported.

| Source | Partitions used | How a partition is read |
|---|---|---|
| `mssql`, `msoledbsql`, `adbc_mssql` | Partitions of the partition scheme | `WHERE $PARTITION.<function>(<column>) = n` |
| `pgsql`, `pgcopy`, `adbc_pgsql` | Leaf partitions (sub-partitions included) | The leaf table directly |
| `oraodp`, `adbc_oracle` | Partitions and sub-partitions | `PARTITION (…)` / `SUBPARTITION (…)` |
| `mysql` | Partitions and sub-partitions | `PARTITION (…)` |
| `teradata`, `adbc_teradata` | Populated row partitions (PPI / MLPPI); row partitioning only | `WHERE PARTITION = n` |
| `adbc_db2` (1.0+) | Visible data partitions of a Db2 for LUW range-partitioned table | One range predicate per partition, so Db2 reads only that partition |

Works best when the partitions have similar sizes. On Db2 for i or z/OS, `DBPartition` falls back to `None`.

## Timepartition (0.18+)

FastTransfer first groups the source by the time parts and counts the rows of each group, then transfers
each group in its own stream, biggest first.

| `--distributekeycolumn` | One stream per |
|---|---|
| `"(date_column,year)"` | Year |
| `"(date_column,year,month)"` | Month |
| `"(date_column,year,month,day)"` | Day |
| `"(date_column,year,week)"` | ISO week |

- Source: a table (`--sourceschema` / `--sourcetable`) or a `--query`; with a query, the date column must
  be in the query output.
- Rows with a `NULL` date go to a dedicated stream.
- Each stream reads a date range such as `l_shipdate >= '1996-07-01' AND l_shipdate < '1996-08-01'`. The
  bounds are text literals in `yyyy-MM-dd`; set `FASTTRANSFER_DATEFORMAT=yyyyMMdd` if the source needs
  the compact form (any other value stops FastTransfer before the transfer). Oracle (`oraodp`) bounds are
  wrapped in `TO_DATE(…)`.
- Granularity (docs): several times more groups than the degree, but not tens of thousands of tiny
  groups. 7 years by `(col,year,month)` ≈ 84 streams suits 8 to 16 threads.

## SQL Server target and the sorted hint

With a **partitioned** `msbulk` target, `DataDriven`, `DBPartition` and `Timepartition` send a sorted hint
on the target partition column. Since 0.18 the hint is not sent when the target has a **rowstore clustered
index** (SQL Server would reject the load with *"Cannot bulk load. The bulk data stream was incorrectly
specified as sorted…"*); heap and clustered columnstore targets still get it. Set
`FASTTRANSFER_SORTEDINPUT=true` to force it only when the source really is sorted on the partition column
(e.g. `ORDER BY` in the `--query`).

## How to choose

1. A natively partitioned table with partitions of similar size: `DBPartition` (0.18+).
2. `Physical` (1.0+) when the source is PostgreSQL, Oracle or Netezza, or SQL Server on read-only data:
   no key to pick. It is also the only parallel method of a backup source (1.1+).
3. Otherwise `Ntile` on a key (even chunks), or `RangeId` on a dense numeric key.
4. `Timepartition` (0.18+) when a date column gives a natural year / month / day split.
5. `DataDriven` when a natural grouping exists (year, month, region, partition function), or for
   **files**: one chunk per file with `filename=true` + `--datadrivenquery "select file from glob('…')"`.
6. `Random` only when nothing else fits.

On 0.18 and older, replace `Physical` with the engine name (`Ctid`, `Rowid`, `Physloc`, `NZDataSlice`).
On 0.17 and older, also replace `DBPartition` with `DataDriven` on the partition function (SQL Server
`$PARTITION.PF(col)`) and `Timepartition` with `DataDriven` on an expression such as `YEAR(col)`.

## Examples

```
--parallelmethod Physical --sourceschema "public" --sourcetable "orders" --degree -2
--parallelmethod Ntile --distributekeycolumn "o_orderkey" --degree 8
--parallelmethod DataDriven --distributekeycolumn "YEAR(o_orderdate)"
--parallelmethod DataDriven --distributekeycolumn "o_ordermonth" --datadrivenquery "SELECT DISTINCT o_ordermonth FROM dim_month"
--parallelmethod Physical                                                # Netezza: no schema / table needed
--parallelmethod Physical --sourceschema "dbo" --sourcetable "orders" --degree 8   # backup_mssql / backup_rman (1.1+)
--parallelmethod DataDriven --distributekeycolumn "DATASLICEID"          # Netezza alternative
--parallelmethod DBPartition --sourceschema "dbo" --sourcetable "lineitem_part" --degree 8
--parallelmethod Timepartition --distributekeycolumn "(l_shipdate,year,month)" --degree 8
```

SQL Server partition function as key, in PowerShell (escape `$`):

```
--parallelmethod DataDriven --distributekeycolumn "`$PARTITION.PF_DATE(o_orderdate)"
```

## Degree tips

- The default `-4` uses a quarter of the cores. `-2` (half) is a common choice for a dedicated server.
- Don't open more parallel sessions than the source and the target can serve. Bulk loads with many
  threads compete for the same target table.
- With `DataDriven`, values beyond the degree wait for a free thread.
