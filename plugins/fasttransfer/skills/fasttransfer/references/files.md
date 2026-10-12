# FastTransfer – Importing CSV, Parquet and JSON files

FastTransfer reads files through its embedded DuckDB engine. The source is an in-memory DuckDB, and the
`--query` calls a DuckDB table function (`read_csv`, `read_parquet`, `read_json`).

```
--sourceconnectiontype "duckdbstream"
--sourceserver ":memory:"
--query "SELECT * FROM read_csv('D:\data\file.csv')"
```

- `duckdbstream` streams the rows, so memory stays low. `:memory:` doesn't mean the file is loaded into
  memory.
- Use `--mapmethod "Name"` for file imports, so columns are matched to the target by name.
- The target table must already exist.
- The query uses **DuckDB SQL**, whatever the target engine. That includes `--distributekeycolumn`
  expressions and `--datadrivenquery`.
- Paths are local or UNC paths, as seen by the machine (or container) running FastTransfer. Globs
  `*` and `**` are supported.

## CSV / TSV: `read_csv`

```sql
SELECT * FROM read_csv('D:\data\orders.csv')                         -- auto-detects delimiter, header, types
SELECT * FROM read_csv('D:\data\orders*.csv')                        -- several files
SELECT * FROM read_csv('file.tsv', delim='\t')
SELECT * FROM read_csv('file.csv', header=false, columns={'col1': 'VARCHAR', 'col2': 'INTEGER'})
SELECT * FROM read_csv('file.csv', skip=3)
SELECT * FROM read_csv('file.csv', encoding='UTF-8')
SELECT * FROM read_csv('file.csv', quote='''', escape='''')
```

Other options shown in the docs: `auto_detect=false`, `new_line`, `comment`, `filename=true`.

## Parquet: `read_parquet`

```sql
SELECT * FROM read_parquet('D:\data\events\*.parquet')
SELECT * FROM read_parquet('D:\data\sales\**\*.parquet', hive_partitioning=true)   -- adds year=/month= columns
SELECT * FROM read_parquet(['D:\data\2023\*.parquet', 'D:\data\2024\*.parquet'], union_by_name=true)
SELECT * FROM read_parquet('D:\data\daily\*.parquet', filename=true)
```

## JSON: `read_json`

```sql
SELECT * FROM read_json('D:\data\customers.json')
SELECT * FROM read_json('D:\data\events.jsonl', format='newline_delimited')
SELECT * FROM read_json('D:\data\data.json', format='array', maximum_depth=10)
SELECT * FROM read_json('D:\data\logs.jsonl.gz', compression='gzip')
```

Nested data is handled in the query:

```sql
SELECT id, user.name AS user_name, user.email AS user_email, order_date FROM read_json('D:\data\orders.json')
SELECT order_id, order_date, unnest(items, recursive := true) FROM read_json('D:\data\orders_with_items.json')
```

## Parallel file import

**One chunk per file** (best for many files):

```
--query "SELECT * exclude filename FROM read_csv('D:\data\lineitem*.CSV', filename=true)"
--parallelmethod "DataDriven"
--distributekeycolumn "filename"
--datadrivenquery "select file from glob('D:\data\*.CSV')"
```

`filename=true` adds a `filename` column used to split the files, and `exclude filename` keeps it out of
the loaded rows.

**One big file**, split on a computed key:

```
--query "SELECT * FROM read_csv('D:\data\articles.csv') rcsv"
--parallelmethod "DataDriven"
--distributekeycolumn "(id/10000)::int"
```

**Hive-partitioned Parquet**, one chunk per partition:

```
--query "SELECT * FROM read_parquet('D:\data\sales\year=*/month=*\*.parquet', hive_partitioning=true)"
--parallelmethod "DataDriven"
--distributekeycolumn "year || '-' || month"
--datadrivenquery "select distinct year || '-' || month as partition_key from read_parquet('D:\data\sales\year=*/month=*\*.parquet', hive_partitioning=true)"
```

`Random` and `Ntile` on a numeric column also work (e.g. `--parallelmethod "Random" --distributekeycolumn "l_orderkey"`).

## Quoting

The DuckDB string literals use single quotes inside the double-quoted `--query`. A literal single quote
inside a DuckDB string is doubled (`''`).

## Cloud storage

FastTransfer 0.17 documents no cloud source or target (no `s3://`, `az://`, `gs://` usage). Don't generate
cloud URIs for FastTransfer. Download the files first, or use FastBCP to export to cloud storage.
