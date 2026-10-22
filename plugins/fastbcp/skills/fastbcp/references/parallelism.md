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
| `Random` | integer/bigint with at least as many distinct values as the degree | any | Modulo-style split on the key. Kept for compatibility and benchmarks; often the slowest, don't recommend it |
| `DataDriven` | column or expression | any | One chunk per distinct value, or per value returned by `--datadrivenquery`. Values beyond the degree are queued |
| `RangeId` | numeric | any | Ranges between MIN and MAX of the key |
| `Ntile` | sortable; not necessarily unique | any | Evenly sized buckets (NTILE) on the key |
| `Timepartition` | `"(datecol, year)"`, `"(datecol, year, month)"`, `"(datecol, year, month, day)"`, `"(datecol, year, week)"` | any | One chunk per time period, biggest periods first (1.4+); writes `year=/month=/day=` style output. Rows with a `NULL` date go to a `<col>=NULL` directory (1.4+) |
| `Physical` (1.4+) | – | see below | Physical row location of the source engine; strategy picked from the connection type |
| `DBPartition` (1.3+) | – | `mssql`, `adbc_mssql`, `pgsql`, `pgcopy`, `adbc_pgsql`, `oraodp`, `adbc_oracle`, `mysql`, `teradata`, `adbc_teradata` | One extraction per native partition/subpartition (Teradata: PPI), largest first. Falls back to `None` with a warning when the table is not partitioned, has one partition, or the source is unsupported |
| `Ctid`, `Physloc`, `Rowid`, `NZDataSlice` | – | – | **Deprecated in 1.4** (warning in the log, removed in 2.0). Use `Physical` |

`adbc_netezza` supports `None`, `Physical` (NZDataSlice), `Ntile`, `Random`, `RangeId` and `DataDriven`.
`adbc_teradata` supports the same methods as `teradata`: `Random`, `Ntile`, `DataDriven`, `RangeId`,
`Timepartition`, `DBPartition`.

### Physical (1.4+)

| Connection types | Resolved strategy |
|---|---|
| `pgsql`, `pgcopy`, `adbc_pgsql` | `Ctid` (CTID ranges) |
| `oraodp`, `adbc_oracle` | `Rowid` (ROWID ranges) |
| `mssql`, `msoledbsql`, `adbc_mssql` | `Physloc` (`%%physloc%%`) |
| `nzsql`, `nzcopy`, `nzoledb`, `adbc_netezza` | `NZDataSlice` (one chunk per data slice, shared work queue) |

- Needs `--sourceschema` and `--sourcetable`; a `--query` may add a simple `WHERE` filter.
  `--distributekeycolumn` is not used.
- Any other connection type stops with an error: `mysql`, `clickhouse`, `hana`, `teradata`,
  `adbc_teradata`, and `odbc`/`oledb` (FastBCP can't tell the engine behind them: use a native type).
- The log shows the resolved strategy, e.g. `Distribute Method : Physical (Ctid)`.
- SQL Server: on a rowstore table, chunks are made from the two low bytes of the page id; on a
  clustered columnstore table FastBCP detects it and splits on the row id. Before 1.4, `Physloc` only
  parallelised with an **odd** degree (with 2, 4, 8… one chunk got every row).
- The deprecated names still work in 1.4 and log
  `Parallel method Ctid is deprecated and will be removed in 2.0 ==> use --parallelmethod Physical`.
  Since 1.4, `Physloc` and `Rowid` also require `--sourceschema` and `--sourcetable`.

### How to choose

From the arpe.io guide "Choosing the right FastBCP parallel method" (1.4). Size the table in
**cells** = rows × columns, read from catalog statistics (refresh stale statistics first). Treat
tables with wide values (`NVARCHAR(MAX)`, `XML`, `VARBINARY`, LOBs) as one size class larger.

1. **Connection type first**: for Parquet, prefer an `adbc_*` type where one exists (`adbc_mssql`,
   `adbc_pgsql`, `adbc_oracle`, `adbc_netezza`, `adbc_teradata`).
2. **Hive layout required?** Then the layout decides: `DBPartition` with
   `FASTBCP_DBPARTITION_TARGET_MODE=HIVE` mirrors the native partitions; `Timepartition` writes
   `year=…/month=…` folders.
3. **By size:**

| Cells | Method |
|---|---|
| < 100K | `None` |
| 100K – 20M | `None` is often fastest; parallelise only for wide cells or a network bottleneck |
| 20M – 500M | `RangeId` if a selective integer leads the clustered index; else `Ntile` (gaps, date or string key), `DBPartition` (even partitions) or `Physical` (simplest, no key) |
| ≥ 500M | As above, plus `Timepartition` when there is a date column; on engines without `Physical`, `DataDriven` on a business column |

`DataDriven` also fits business-driven, partial or incremental extracts (`--datadrivenquery`).
`Random` is not part of the decision.

### Pitfalls

- `Physical` is rarely the fastest on SQL Server, and without `--merge true` it writes one file per thread.
- `RangeId` on a multi-column clustered index whose leading column has few values (e.g.
  `(region_id, customer_id)` keyed on `customer_id`): every chunk scans the whole index. Big gaps in
  the key values give unbalanced chunks; use `Ntile` instead.
- `Ntile` needs an index (or columnstore) on the key to stay cheap. Before 1.4 it could drop rows with
  sub-millisecond timestamp keys (`datetime2(4..7)`, PostgreSQL `timestamp`, Oracle `TIMESTAMP(4..9)`).
- `Timepartition` periods can be very uneven, and the date column needs an index or partitioning to
  avoid repeated scans. Before 1.4 it skipped `NULL` dates and duplicated rows with week granularity.
- `DataDriven` is unbalanced when a few values hold most rows; many values mean many queries.
- Before 1.4, a source `WHERE` containing `OR` could export extra rows with any parallel method (the
  added predicate was not parenthesized). Add parentheses yourself on older versions.

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
