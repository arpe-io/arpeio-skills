# FastTransfer – Parallel transfer

FastTransfer splits the source read into chunks. It runs one source query per chunk, each with its own
`WHERE` clause, and bulk loads the chunks into the target table in parallel. Parallelism is worth it only
for large volumes (the docs say more than about 1M cells).

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
| `Ctid` | `pgsql`, `pgcopy` | `--sourceschema`, `--sourcetable` | PostgreSQL CTID ranges. Recommended for PostgreSQL |
| `Physloc` | `mssql` (or `odbc` to SQL Server) | – | Physical page location. **Only on read-only tables, databases or snapshots**, because it relies on physical page locations that may change during writes |
| `Rowid` | `oraodp` | `--sourceschema`, `--sourcetable` | Oracle ROWID slices |
| `NZDataSlice` | `nzsql` | `--sourceschema`, `--sourcetable` | Netezza data slices. Alternative: `DataDriven` on `DATASLICEID` |
| `Ntile` | any | key | Even buckets with NTILE; the key can be numeric, date, datetime or string |
| `RangeId` | any | numeric key | Ranges between MIN and MAX; best on an identity / sequence without gaps |
| `DataDriven` | any | key | One chunk per distinct value of the key, or per row of `--datadrivenquery`. More values than threads are queued |
| `Random` | any | numeric key | Modulo on the key |
| `DBPartition` (0.18+) | `mssql`, `msoledbsql`, `pgsql`, `pgcopy`, `oraodp`, `mysql`, `teradata` | `--sourceschema`, `--sourcetable`; `--query` / `--fileinput` not allowed | One stream per native partition, biggest first (details below) |
| `Timepartition` (0.18+) | any | date tuple `"(col,year[,month[,day]])"` or `"(col,year,week)"` | One stream per year, month, day or ISO week of a date column, biggest first |

`--sourceschema`/`--sourcetable` are required by `Ctid`, `Rowid` and `NZDataSlice` even when a `--query` is
given. FastTransfer stops with an error otherwise.

## DBPartition (0.18+)

FastTransfer reads the partitions from the catalog and transfers each one in its own stream, biggest first.
No key column. It falls back to `None` (single stream) with a warning when the table is not partitioned,
has a single partition, or the source type is not supported.

| Source | Partitions used | How a partition is read |
|---|---|---|
| `mssql`, `msoledbsql` | Partitions of the partition scheme | `WHERE $PARTITION.<function>(<column>) = n` |
| `pgsql`, `pgcopy` | Leaf partitions (sub-partitions included) | The leaf table directly |
| `oraodp` | Partitions and sub-partitions | `PARTITION (…)` / `SUBPARTITION (…)` |
| `mysql` | Partitions and sub-partitions | `PARTITION (…)` |
| `teradata` | Populated row partitions (PPI / MLPPI); row partitioning only | `WHERE PARTITION = n` |

Works best when the partitions have similar sizes.

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
2. The engine-native method if there is one: `Ctid` (PostgreSQL), `Rowid` (Oracle), `NZDataSlice`
   (Netezza), `Physloc` (SQL Server, read-only data only).
3. Otherwise `Ntile` on a key (even chunks), or `RangeId` on a dense numeric key.
4. `Timepartition` (0.18+) when a date column gives a natural year / month / day split.
5. `DataDriven` when a natural grouping exists (year, month, region, partition function), or for
   **files**: one chunk per file with `filename=true` + `--datadrivenquery "select file from glob('…')"`.
6. `Random` only when nothing else fits.

On 0.17 and older, replace `DBPartition` with `DataDriven` on the partition function (SQL Server
`$PARTITION.PF(col)`) and `Timepartition` with `DataDriven` on an expression such as `YEAR(col)`.

## Examples

```
--parallelmethod Ctid --sourceschema "public" --sourcetable "orders" --degree -2
--parallelmethod Ntile --distributekeycolumn "o_orderkey" --degree 8
--parallelmethod DataDriven --distributekeycolumn "YEAR(o_orderdate)"
--parallelmethod DataDriven --distributekeycolumn "o_ordermonth" --datadrivenquery "SELECT DISTINCT o_ordermonth FROM dim_month"
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
