# LakeXpress – Export: parallelism, per-table settings, output layout

All data is read by FastBCP and written as Parquet (compression `--compression_type`, default `Zstd`).

## Parallelism

| Level | Flag | Default |
|---|---|---|
| Tables at the same time | `--n_jobs` | 1 |
| FastBCP processes inside one table | `--fastbcp_p` | 1 |
| Minimum rows for a parallel table | `--large_table_threshold` | 100000 |

`--n_jobs 4 --fastbcp_p 2` = 4 tables at once, each split in 2. Tables under the threshold always use one
stream. If the source has no parallel read method, `--fastbcp_p` > 1 is ignored with a warning.

Suggested `--fastbcp_p` (docs): < 100K rows 1; 100K–10M 2–4; 10M–100M 4–8; > 100M 8–16. Mind the load on
the source: total processes ≈ `n_jobs × fastbcp_p`.

## Methods (chosen automatically per table)

| Method | Source | Key column |
|---|---|---|
| `None` | all | – (sequential) |
| `Ctid` | PostgreSQL | – |
| `Rowid` | Oracle (needs `SELECT_CATALOG_ROLE`) | – |
| `Physloc` | SQL Server | – |
| `Random` | any | integer / bigint column |
| `DataDriven` | any | column or expression with few distinct values (`YEAR(l_shipdate)`, `TO_CHAR(l_shipdate, 'YYYY')`) |
| `RangeId` | any | numeric or date column (min/max ranges) |
| `Ntile` | any | numeric column (even chunks) |
| `Timepartition` | any | `(date_col,year)`, `(date_col,year,month)`, `(date_col,year,month,day)`, `(date_col,year,week)` |

`Timepartition` writes Hive-style folders: `year=2008/month=01/<table>.parquet`.

## Per-table override: `--fastbcp_table_config`

```text
--fastbcp_table_config "[schema.]table:method:key_column:degree;[schema.]table:method:key_column:degree"
```

- The flag can be repeated or entries joined with `;`.
- **The degree is required** (integer): an entry without a valid degree is silently ignored.
- Method and key can be empty to keep the automatic method: `orders:::4`; no key for `Ctid`, `Rowid`,
  `Physloc`: `orders:Ctid::4`.
- Without a schema, the entry applies to that table in every schema (warning logged).
- A database-specific method used on another database (e.g. `Ctid` on SQL Server) is replaced by the
  right one, with a warning.

```bash
--fastbcp_table_config "dbo.lineitem:DataDriven:YEAR(l_shipdate):8;dbo.orders:Physloc::4"
--fastbcp_table_config "lineitem:DataDriven:TO_CHAR(l_shipdate, 'YYYY'):8;orders:Ctid::4"
--fastbcp_table_config "CARRIER_CLAIMS:Timepartition:(CLM_FROM_DT,year,month):4" \
--fastbcp_table_config "PRESCRIPTION_DRUG_EVENTS:Timepartition:(SRVC_DT,year,month):4"
```

Persistent alternative, which takes priority over the CLI: rows in the `partition_columns` table of the
LakeXpress DB.

```sql
INSERT INTO partition_columns (
    source_db_type, source_database, source_schema, source_table,
    fastbcp_method, fastbcp_parallel_degree, fastbcp_distribution_key
) VALUES ('postgres', 'analytics', 'public', 'events', 'Ctid', 4, NULL);
```

Priority: `partition_columns` > `--fastbcp_table_config` > automatic.

## Table selection

- `--source_schema_name`: comma list of schemas, `LIKE` patterns (`"tpch_1%"`); omitted = all schemas.
- `--include` / `--exclude`: comma lists of table `LIKE` patterns (`%` any characters, `_` one); they can
  be combined (`--include "report_%" --exclude "%_test"`).
- `--min_rows` / `--max_rows`: by row count.
- Tables and views are exported. A selection that matches nothing ends with exit 0 and the warning `No
  tables found to export`.

## Output layout

```text
<base>/[<sub_path>/]<schema>/<table>/part-00000.parquet
```

`<base>` is `--output_dir` (local, saved as an absolute path since 0.4.9) or the storage entry's
`directory`. `--sub_path staging/daily/2025-01-15` adds folders (date partitioning, environments,
projects). `--sub_path "daily/$(date +%Y%m%d)"` is evaluated once, by the shell running `config create`:
the value is then fixed in the configuration.

## CDM metadata

`--generate_metadata` writes Common Data Model files (`manifest.json`, `<table>.cdm.json`) next to the
Parquet files; `--manifest_name` names the manifest (default: schema or database name). With OneLake
storage the generated paths may be wrong in 0.4.10.

## Monitoring a run (LakeXpress DB)

```sql
SELECT source_schema, source_table, job_status, started_at, completed_at, num_rows_exported
FROM export_jobs
WHERE run_id = '<RUN_ID>'
ORDER BY started_at DESC;
```
