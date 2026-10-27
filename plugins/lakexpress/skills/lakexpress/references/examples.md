# LakeXpress – Examples (0.6.0)

Entry names (`lxdb_postgres`, `source_postgres`, `source_mssql`, `source_oracle`, `s3_01`, `gcs_01`,
`azure_01`, `onelake_01`, `snowflake_prod`, …) are keys of `credentials.json` (credentials.md). Linux
syntax; PowerShell at the end. FastBCP is bundled in `engine/` and the license is saved in
its default place (environment.md). Every `sync` takes `--auth_file`, `--lxdb_auth_id` and
`--sync_id` (`<SYNC_ID>` = the id printed by `config create`, or the one you chose).

## First export: PostgreSQL schema → local Parquet

```bash
./LakeXpress lxdb init --auth_file credentials.json --lxdb_auth_id lxdb_postgres

./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres \
  --source_schema_name public \
  --output_dir ./exports \
  --n_jobs 4 \
  --fastbcp_p 2

./LakeXpress sync --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID>
./LakeXpress status --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID>
```

Files: `exports/public/<table>/*.parquet`. To try without a database server, use a SQLite LakeXpress DB
entry (`"ds_type": "sqlite", "auth_mode": "filesystem", "info": {"filepath": "/abs/path/lx.sqlite"}`).

## SQL Server → Azure (ADLS Gen2), with sub-path and CDM metadata

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_mssql \
  --source_schema_name sales \
  --target_storage_id azure_01 \
  --sub_path adventureworks/sales \
  --n_jobs 4 \
  --fastbcp_p 2 \
  --generate_metadata
```

## Filtered schemas and tables → S3

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres \
  --source_schema_name "sales%, marketing%" \
  --include "fact_%, dim_%" \
  --exclude "temp%, test%" \
  --target_storage_id s3_01 \
  --sub_path data-lake/prod \
  --n_jobs 8 \
  --fastbcp_p 4
```

## Snowflake external tables (default) with views

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres \
  --source_schema_name tpch_1 \
  --target_storage_id s3_01 \
  --publish_target snowflake_prod \
  --n_jobs 4
```

Schema `EXT_tpch_1`, external tables + `V_<table>` views. Add `--no_views` to skip the views.

## Snowflake internal tables, custom names, primary keys

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres \
  --source_schema_name tpch_1 \
  --target_storage_id s3_01 \
  --sub_path production \
  --publish_target snowflake_prod \
  --publish_method internal \
  --publish_schema_pattern "INT_{subpath}_{date}" \
  --publish_table_pattern "{schema}_{table}" \
  --pk_constraints \
  --n_jobs 4 \
  --fastbcp_p 2
```

Schema `INT_production_<YYYYMMDD of the run>`, tables `tpch_1_customer`, …

## Databricks managed Delta tables

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_mssql \
  --source_schema_name dbo \
  --target_storage_id s3_01 \
  --publish_target databricks_prod \
  --publish_method internal \
  --publish_schema_pattern "bronze_{schema}" \
  --n_jobs 4
```

Tables `<catalog of the entry>.bronze_dbo.<table>`.

## Microsoft Fabric Lakehouse (Delta tables)

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres \
  --source_schema_name tpch_1 \
  --target_storage_id onelake_01 \
  --publish_target fabric_lakehouse \
  --publish_method internal \
  --publish_table_pattern "{schema}_{table}" \
  --n_jobs 4 \
  --fastbcp_p 2
```

The `fabric_lakehouse` entry has `"fabric_target": "lakehouse"`. External (SQL views):
`--publish_method external --publish_table_pattern "VW_{schema}_{table}"`.

## BigQuery native tables

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres \
  --source_schema_name tpch_1 \
  --target_storage_id gcs_01 \
  --publish_target bigquery_prod \
  --publish_method internal \
  --publish_schema_pattern "lx_{schema}" \
  --n_jobs 4
```

Dataset `lx_tpch_1`. Without `--publish_method internal` and a pattern: external tables in dataset
`EXT_tpch_1`.

## Oracle → S3 → AWS Glue (Athena)

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_oracle \
  --source_schema_name SALES \
  --target_storage_id s3_01 \
  --publish_target glue_catalog \
  --publish_database_name analytics \
  --publish_table_pattern "{schema}_{table}" \
  --fastbcp_table_config "SALES.ORDERS:Rowid::8" \
  --n_jobs 4 \
  --fastbcp_p 4
```

## Amazon Redshift internal tables

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres \
  --source_schema_name public \
  --target_storage_id s3_01 \
  --sub_path mydata \
  --publish_target redshift_classic \
  --publish_method internal \
  --publish_schema_pattern "INT_{subpath}" \
  --publish_table_pattern "{schema}_{table}" \
  --pk_constraints
```

## MotherDuck and DuckLake

```bash
# MotherDuck internal tables in schema lx_tpch_1 of the entry's database
./LakeXpress config create \
  --auth_file credentials.json --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres --source_schema_name tpch_1 \
  --target_storage_id s3_01 \
  --publish_target motherduck_prod --publish_method internal \
  --publish_schema_pattern "lx_{schema}" --n_jobs 4

# DuckLake catalog over the exported files (schema = source schema)
./LakeXpress config create \
  --auth_file credentials.json --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres --source_schema_name tpch_1 \
  --target_storage_id s3_01 \
  --publish_target ducklake_catalog --publish_schema_pattern "{schema}"
```

## Teradata → local, by month

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_teradata \
  --source_schema_name claims \
  --output_dir /data/claims \
  --fastbcp_table_config "CARRIER_CLAIMS:Timepartition:(CLM_FROM_DT,year,month):4" \
  --fastbcp_table_config "PRESCRIPTION_DRUG_EVENTS:Timepartition:(SRVC_DT,year,month):4" \
  --n_jobs 4
```

## Incremental sync with upsert into Snowflake

```bash
./LakeXpress config create \
  --auth_file credentials.json \
  --lxdb_auth_id lxdb_postgres \
  --source_db_auth_id source_postgres \
  --source_schema_name sales \
  --target_storage_id s3_01 \
  --incremental_table "sales.orders:updated_at:datetime!upsert" \
  --incremental_table "sales.order_items:created_at:datetime" \
  --incremental_safety_lag 600 \
  --publish_target snowflake_prod \
  --publish_method internal \
  --sync_id sales-incremental \
  --n_jobs 4

# every run (cron): first run = full baseline, then only new rows
./LakeXpress sync --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id sales-incremental
```

`sales.orders` must have a primary key. Other tables of `sales` are fully exported each run.

## Export now, publish later; resume; clean up

```bash
./LakeXpress 'sync[export]' --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID>
./LakeXpress 'sync[publish]' --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID>

# after exit 40: retry the failed tables of the latest incomplete run (then publishes)
./LakeXpress sync --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID> --resume
./LakeXpress sync --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID> --resume --run_id <RUN_ID>

# a run left "running" after a crash
./LakeXpress cleanup --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID> --status running --dry-run
./LakeXpress cleanup --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID> --status running

# change a configuration = delete + create again
./LakeXpress config delete --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID>
./LakeXpress config delete --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID> --confirm
```

## Windows PowerShell

```powershell
$env:LX_PG_USER = "lakexpress_reader"
$env:LX_PG_PASSWORD = "<password>"

.\LakeXpress.exe config create `
  --auth_file credentials.json `
  --lxdb_auth_id lxdb_postgres `
  --source_db_auth_id source_postgres `
  --source_schema_name public `
  --output_dir C:\data\exports `
  --n_jobs 4 `
  --fastbcp_p 2

.\LakeXpress.exe sync --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID>
.\LakeXpress.exe 'sync[publish]' --auth_file credentials.json --lxdb_auth_id lxdb_postgres --sync_id <SYNC_ID>
```
