# FastTransfer – Parallel transfer

FastTransfer splits the source read into chunks. It runs one source query per chunk, each with its own
`WHERE` clause, and bulk loads the chunks into the target table in parallel. Parallelism is worth it only
for large volumes (the docs say more than about 1M cells).

## Parameters

| Flag | Default | Notes |
|---|---|---|
| `--parallelmethod` | `None` | Method, see below |
| `--distributekeycolumn` | – | Column or expression, e.g. `"YEAR(o_orderdate)"`, `"(id/10000)::int"`, `"filename"`. Required by `Random`, `DataDriven`, `RangeId`, `Ntile` |
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

`--sourceschema`/`--sourcetable` are required by `Ctid`, `Rowid` and `NZDataSlice` even when a `--query` is
given. FastTransfer stops with an error otherwise.

## How to choose

1. The engine-native method if there is one: `Ctid` (PostgreSQL), `Rowid` (Oracle), `NZDataSlice`
   (Netezza), `Physloc` (SQL Server, read-only data only).
2. Otherwise `Ntile` on a key (even chunks), or `RangeId` on a dense numeric key.
3. `DataDriven` when a natural grouping exists (year, month, region, partition function), or for
   **files**: one chunk per file with `filename=true` + `--datadrivenquery "select file from glob('…')"`.
4. `Random` only when nothing else fits.

## Examples

```
--parallelmethod Ctid --sourceschema "public" --sourcetable "orders" --degree -2
--parallelmethod Ntile --distributekeycolumn "o_orderkey" --degree 8
--parallelmethod DataDriven --distributekeycolumn "YEAR(o_orderdate)"
--parallelmethod DataDriven --distributekeycolumn "o_ordermonth" --datadrivenquery "SELECT DISTINCT o_ordermonth FROM dim_month"
--parallelmethod DataDriven --distributekeycolumn "DATASLICEID"          # Netezza alternative
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
