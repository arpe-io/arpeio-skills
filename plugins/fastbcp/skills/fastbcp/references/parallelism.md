# FastBCP – Parallel export

FastBCP splits the source into chunks and runs one query per chunk in parallel. Each query has a
`WHERE` clause that reads one part of the data.

## Parameters

| Long | Short | Default | Notes |
|---|---|---|---|
| `--parallelmethod` | `-m` | `None` | See methods below |
| `--distributekeycolumn` | `-c` | – | Column or SQL expression (e.g. `"YEAR(o_orderdate)"`). Required by `Random`, `DataDriven`, `RangeId`, `Ntile`, `Timepartition` |
| `--paralleldegree` | `-p` | `-2` | `N>0`: exactly N threads (capped at the core count); `0`: all cores; `-N`: cores / N |
| `--datadrivenquery` | – | – | `DataDriven` only: query returning the list of split values |
| `--merge` | `-M` | `false` | `true` merges the chunk files into one. CSV and Parquet only; not available for cloud targets |

Set `--sourceschema` and `--sourcetable` with any parallel method; the docs mark them as mandatory.
A `--query` can be added to filter rows.

## Methods

| Method | Key column | Sources | How it splits |
|---|---|---|---|
| `None` | – | any | No parallelism |
| `Random` | integer/bigint with at least as many distinct values as the degree | any | Distributes rows across threads using the key |
| `DataDriven` | column or expression | any | One chunk per distinct value, or per value returned by `--datadrivenquery`. Values beyond the degree are queued |
| `RangeId` | numeric | any | Ranges between MIN and MAX of the key |
| `Ntile` | sortable; not necessarily unique | any | Evenly sized buckets (NTILE) on the key |
| `Timepartition` | `"(datecol, year)"`, `"(datecol, year, month)"`, `"(datecol, year, month, day)"`, `"(datecol, year, week)"` | any | One chunk per time period; writes `year=/month=/day=` style output |
| `Ctid` | – | `pgsql`, `pgcopy` (and some PostgreSQL-compatible DBs) | Physical row id (CTID) ranges |
| `Physloc` | – | `mssql` | Internal hidden physical-location ranges |
| `Rowid` | – | `oraodp`, `adbc_oracle` | ROWID ranges. The docs spell it both `Rowid` and `RowId`; both appear in examples |
| `NZDataSlice` | – | `nzsql`, `adbc_netezza` | One chunk per Netezza data slice, taken from a shared work queue |
| `DBPartition` (1.3+) | – | `mssql`, `adbc_mssql`, `pgsql`, `pgcopy`, `adbc_pgsql`, `oraodp`, `adbc_oracle`, `mysql`, `teradata` | One extraction per native partition/subpartition, largest first. Falls back to `None` with a warning when the table is not partitioned, has one partition, or the source is unsupported |

`adbc_netezza` supports `None`, `NZDataSlice`, `Ntile`, `Random`, `RangeId` and `DataDriven`.

### How to choose

1. A table that is natively partitioned, when per-partition files are useful: `DBPartition`.
2. Otherwise, the engine-native method if there is one: `Ctid` (PostgreSQL), `Physloc` (SQL Server),
   `Rowid` (Oracle), `NZDataSlice` (Netezza). These need no key column.
3. Otherwise `Ntile` on a numeric or sortable key (even chunks) or `RangeId` (cheaper planning,
   but uneven if the key has gaps).
4. `DataDriven` when you want one file per business value (year, region…) or incremental extraction
   with `--datadrivenquery`.
5. `Timepartition` for time-series data written as year/month/day folders.
6. `Random` only when nothing else fits.

### DBPartition output layout (`FASTBCP_DBPARTITION_TARGET_MODE`)

- `CHUNK` (default): `orders_chunk_001.parquet`, `orders_chunk_002.parquet`, …
- `HIVE`: one directory per partition:
  - PostgreSQL, Oracle, MySQL: `partition=<p>/[subpartition=<sp>/]orders.parquet`
  - SQL Server: `bounds=<lower>_<upper>/orders.parquet`
  - Teradata: `L1=<n>/[L2=<n>/]orders.parquet`
- `--merge` works in both modes. HIVE output can be read as a partitioned dataset by Spark, DuckDB,
  Trino or Athena.

```bash
export FASTBCP_DBPARTITION_TARGET_MODE=HIVE
./FastBCP --connectiontype oraodp --server "dbserver:1521/ORCLPDB" \
  --user "$FASTBCP_USER" --password "$FASTBCP_PASSWORD" \
  --sourceschema "SALES" --sourcetable "ORDERS" \
  --directory "/data/export/orders" --fileoutput "orders.parquet" \
  --parallelmethod DBPartition --paralleldegree 8 --merge false
```

```powershell
$env:FASTBCP_DBPARTITION_TARGET_MODE = "HIVE"
```

## Distribute key expressions

You can use an expression as the key, in the source's SQL dialect:

```
--parallelmethod DataDriven --distributekeycolumn "YEAR(o_orderdate)"
--parallelmethod DataDriven --datadrivenquery "SELECT o_orderdate FROM dim_date WHERE ref_date > getdate() - 30"
```

In PowerShell, escape `$` inside double quotes, e.g. SQL Server partition function:
`` --distributekeycolumn "`$partition.PF_DATE(o_orderdate)" ``.

## Merge

- Default `false` since 0.32 (it was `true` before). The Parallel Parameters page states both
  defaults, so always set `--merge` explicitly when it matters.
- `--merge true`: chunk files are merged into the single `--fileoutput` file. CSV and Parquet only,
  local or UNC targets only.
- `--merge false`: keeps one file per chunk. This is required for cloud targets and for JSON, XLSX
  and other formats.

## Degree tips

- Start with the default `-2` (half the cores). Don't use more parallel queries than the source
  database can serve.
- `--paralleldegree -1` uses all cores. A positive value is capped at the core count.
- With `DataDriven`, if there are more distinct values than the degree, chunks are queued (throttled).
