# LakeXpress – Publishing to data platforms

Publishing is optional and runs after the export, from the Parquet files in storage. It is set in
`config create` (`--publish_target` + options) and run by `sync` or `'sync[publish]'`.

## Overview

| Target | Storage | `external` (default) | `internal` | `--pk_constraints` | Notes |
|---|---|---|---|---|---|
| Snowflake | S3, Azure or GCS, through a Snowflake **stage** | external tables + `V_<table>` views | tables loaded with `COPY INTO` | internal | `--no_views` |
| Databricks (Unity Catalog) | S3 | external tables | managed Delta tables (`COPY INTO`) | internal | catalog = `catalog` of the entry |
| Microsoft Fabric Lakehouse | OneLake | SQL analytics views over the Parquet files (Files section) | Delta tables (Tables section), recommended | internal (listed by `--help`) | flat namespace: no schema pattern |
| BigQuery | GCS | external tables | native tables (load jobs) | internal | dataset = schema pattern or `--publish_database_name` |
| Amazon Redshift | S3, same region | Spectrum external tables (Glue catalog) | tables loaded with `COPY … FORMAT AS PARQUET` | internal, informational | |
| MotherDuck | S3, GCS or Azure | views over `read_parquet()` | `CREATE TABLE AS SELECT` | internal | storage secret created automatically |
| AWS Glue Data Catalog | S3 | Glue tables (Athena, Spectrum, EMR) | – | – | database = schema pattern or `--publish_database_name` |
| DuckLake | local or S3, GCS, Azure | files registered in the catalog, no copy | – | – | `--publish_method` has no effect |

## Naming

| Option | Default | Rules |
|---|---|---|
| `--publish_schema_pattern` | `EXT_{schema}` with `external`, `{schema}` with `internal` (DuckLake too: `EXT_{schema}` unless set) | Tokens `{schema}`, `{database}`, `{date}`, `{timestamp}`, `{uuid}`, `{subpath}`; **no `{table}`** |
| `--publish_table_pattern` | `{table}` | Must contain `{table}`; may add `{schema}` etc. |
| `--publish_database_name` | – | Glue database or BigQuery dataset, literal (no tokens), replaces the schema pattern. Unused elsewhere |

- `{date}` = YYYYMMDD, `{timestamp}` = YYYYMMDD_HHMMSS (local time), `{uuid}` = one UUID4 per run,
  `{subpath}` = the `--sub_path` value.
- Several source schemas into one target schema: `--publish_schema_pattern "consolidated"` (or
  `"{subpath}"`) with `--publish_table_pattern "{schema}_{table}"`.
- A wrong pattern stops `sync` before the export (exit 10).
- Existing tables are replaced on each publish (Glue always recreates; DuckLake drops and recreates).

## Snowflake

- Prerequisites: a database, a warehouse and a **stage pointing to the export location**. The storage
  entry's `directory` must be exactly the stage URL (`DESC STAGE …` shows it), otherwise files are not
  found.
- Grants (docs): `CREATE SCHEMA ON DATABASE`, `USAGE ON WAREHOUSE`, `USAGE ON STAGE` for the role used.
- External: tables expose a `VALUE` column; LakeXpress also creates views `V_<table>` with typed columns
  (prefix fixed), unless `--no_views`. Refresh external tables after new files:
  `ALTER EXTERNAL TABLE EXT_TPCH_1.CUSTOMER REFRESH;`.
- Internal: `--pk_constraints` adds the source primary keys.
- Auth modes: `password`, `pat` (recommended for automation), `keypair`, `oauth` (credentials.md).
- Renamed flags: `--publish` → `--publish_target`; `--snowflake_table_type` → `--publish_method`;
  `--snowflake_pk_constraints` → `--pk_constraints`; `--snowflake_no_views` → `--no_views`;
  `--snowflake_view_prefix` removed.

## Databricks

- Unity Catalog workspace, a SQL warehouse (`http_path`), `USE CATALOG`, `CREATE SCHEMA`, `CREATE
  TABLE`. External tables also need a storage credential and an external location covering the S3 path:

```sql
CREATE STORAGE CREDENTIAL my_s3_credential
WITH (AWS_IAM_ROLE = 'arn:aws:iam::123456789012:role/DatabricksS3AccessRole');
CREATE EXTERNAL LOCATION my_s3_location
URL 's3://my-datalake-bucket/lakexpress/' WITH (STORAGE CREDENTIAL my_s3_credential);
```

- `internal` = managed Delta tables (time travel, `OPTIMIZE`…). No `--databricks_catalog` flag: the catalog
  is the entry's `catalog`. `--databricks_table_type` was renamed `--publish_method`.

## Microsoft Fabric (Lakehouse)

- The entry needs `"fabric_target": "lakehouse"`; storage is a `onelake` entry
  (`onelake://workspace/lakehouse/` or `onelake://workspace/lakehouse.Lakehouse/Files/`).
- Service principal: Member or Contributor of the workspace; OneLake Storage Blob Data Contributor.
- The Lakehouse namespace is flat: `--publish_schema_pattern` is not used. With several source schemas
  use `--publish_table_pattern "{schema}_{table}"`; views next to tables of the same name:
  `"VW_{schema}_{table}"`.
- Recommended: `--publish_method internal` (Delta tables, SQL endpoint, Power BI).
- `--generate_metadata` with OneLake storage may write wrong paths in the CDM files: check them or omit it.

## BigQuery

- Roles: BigQuery Data Editor, BigQuery Job User (internal), Storage Object Viewer (external). The same
  service-account key can serve the GCS and BigQuery entries.
- `location` of the entry should match the bucket region ("Dataset not found" otherwise).
- Dataset name = schema pattern (default `EXT_{schema}` for external) or `--publish_database_name`.
- External tables: no clustering or partitioning. Native (`internal`) tables are a snapshot.

## Amazon Redshift

- Provisioned cluster or Serverless; user with `CREATE SCHEMA` / `CREATE TABLE`; port 5439 reachable.
- `iam_role` that Redshift can assume (`redshift.amazonaws.com`), with S3 read access (both methods) and
  Glue access (external). Cluster and bucket in the **same region**.
- External (Spectrum): the external schema creates its Glue database if needed. Internal: `COPY` reads
  every Parquet file of the table folder. Decimals keep the source precision (`DECIMAL(38,18)` only when
  the source has none).
- `--pk_constraints`: informational, internal tables only.

## MotherDuck

- `database` must already exist. Token in the entry (`token`) or in `motherduck_token` (`env` mode).
- LakeXpress creates `CREATE OR REPLACE SECRET lakexpress_storage` from the `--target_storage_id` entry on
  every publish. To query external views outside LakeXpress, create a DuckDB secret yourself
  (`CREATE SECRET … (TYPE S3, KEY_ID …, SECRET …, REGION …)`).
- External: `CREATE OR REPLACE VIEW … AS SELECT * FROM read_parquet('s3://…/schema/table/*.parquet')`.

## AWS Glue

- IAM: `glue:CreateDatabase`, `GetDatabase`, `DeleteDatabase`, `CreateTable`, `GetTable`, `DeleteTable`,
  `UpdateTable`, `GetTables`, `BatchCreatePartition`, `GetPartitions`.
- Glue "database" = namespace in the catalog: `--publish_schema_pattern` (default `EXT_{schema}`) or the
  literal `--publish_database_name`. No `--glue_skip_existing` flag: tables are always recreated.
- External only; incremental `upsert` / `delete_insert` not supported.

## DuckLake

- `--publish_target` = the catalog entry (`sqlite`, `duckdb`, `postgres` or `mysql`). LakeXpress installs
  and loads the `ducklake` DuckDB extension itself (the machine must be able to download it once).
- Files are registered with `ducklake_add_data_files` (no data copy). For cloud storage a DuckDB secret is
  created. Column types come from the Parquet files when reachable.
- Tables are published one by one, whatever `--n_jobs`.
- Query: `ATTACH 'sqlite:/path/ducklake_catalog.sqlite' AS lake (TYPE ducklake);` (after
  `INSTALL ducklake; LOAD ducklake;`).

## Incremental loading strategies by target

`!append` works everywhere. `!upsert` and `!delete_insert` (need a source primary key):

| Target | `upsert` / `delete_insert` |
|---|---|
| Snowflake, Redshift, MotherDuck, DuckLake | yes; those tables are always persistent internal tables, whatever `--publish_method` |
| Databricks, BigQuery | yes, with `--publish_method internal` |
| Fabric Warehouse | yes |
| Fabric Lakehouse, AWS Glue | **no** |

Unsupported combinations stop the publish step with an error naming the table (incremental.md).
