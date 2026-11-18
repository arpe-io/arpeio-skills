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
  Tables of incremental runs are kept across runs (incremental.md).

## Snowflake

- Prerequisites: a database, a warehouse and a **stage pointing to the export location**. The storage
  entry's `directory` must be exactly the stage URL (`DESC STAGE …` shows it). LakeXpress checks it
  before creating any table and stops with an error naming both when they differ (0.6.2+; a
  `directory` that is a parent of the stage URL is not accepted). Snowflake's `gcs://` and `azure://`
  forms count as LakeXpress's `gs://` and `abfss://`.
- Grants (docs): `CREATE SCHEMA ON DATABASE`, `USAGE ON WAREHOUSE`, `USAGE ON STAGE` for the role used.
- External: tables expose a `VALUE` column; LakeXpress also creates views `V_<table>` with typed columns
  (prefix fixed), unless `--no_views`. For a table exported in full, refresh it after new files:
  `ALTER EXTERNAL TABLE EXT_TPCH_1.CUSTOMER REFRESH;` (never an incremental `append` table: see below).
- Internal: `--pk_constraints` adds the source primary keys.
- Sessions are opened with `TIMEZONE = 'UTC'` (0.6.6+), whatever the user or account setting, so
  `ALTER USER … SET TIMEZONE = 'UTC'` is not needed: timestamps without a time zone keep the source
  wall clock, `TIMESTAMP_TZ` values are stored with a `+00:00` offset, and `_lx_updated_at` is UTC.
  Before 0.6.6, values loaded with `COPY INTO` were shifted by the session time zone (an Oracle `DATE`
  `1992-01-01 00:00:00` loaded as `1991-12-31 16:00:00` with `America/Los_Angeles`). Tables loaded
  before keep their shifted values until published again: a full publish recreates them, incremental
  tables must be dropped first.
- Incremental tables (0.6.8+; before 0.6.8 they were not published correctly, troubleshooting.md):
  - `upsert` / `delete_insert` tables are internal tables in schema `<schema>` (not `EXT_<schema>`),
    whatever `--publish_method`, and get no `V_` view.
  - `append` with `external`: the external table is located above the run folders, and each publish
    registers the files of its run. Do not `REFRESH` such a table yourself: a full refresh would also
    register runs that were exported but not published.
  - An external `append` table created before 0.6.8 points at a single run folder and cannot take
    another run: the publish of that table stops with an error that says so. Drop the table; the next
    baseline rebuilds it in full.
- Auth modes: `password`, `pat` (recommended for automation), `keypair`, `oauth` (credentials.md).
- Renamed flags: `--publish` → `--publish_target`; `--snowflake_table_type` → `--publish_method`;
  `--snowflake_pk_constraints` → `--pk_constraints`; `--snowflake_no_views` → `--no_views`;
  `--snowflake_view_prefix` removed.

### Oracle → Snowflake (0.6.5+)

Set `"use_adbc": true` on the Oracle source. FastBCP's native driver truncates `TIMESTAMP(n)` to
milliseconds, writes `TIMESTAMP WITH TIME ZONE` as text, shifts `TIMESTAMP WITH LOCAL TIME ZONE` by
the UTC offset of the machine running LakeXpress, and truncates a `NUMBER` without precision beyond 18
decimal places. Through ADBC one difference remains: dates before 1582-10-15 shift (`0001-01-01`
becomes `0000-12-30`).

| Oracle type | Snowflake type |
|---|---|
| `NUMBER` (no precision, no scale) | `NUMBER(38,18)`, or `VARCHAR` with `--unbounded_number_type varchar` |
| `INTEGER`, `NUMBER(*,0)` / `NUMBER(*,s)` / `NUMBER(p,s)` | `NUMBER(38,0)` / `NUMBER(38,s)` / `NUMBER(p,s)` |
| `FLOAT`, `BINARY_FLOAT`, `BINARY_DOUBLE` | `FLOAT` |
| `VARCHAR2(n)`, `NVARCHAR2(n)`, `CHAR(n)`, `NCHAR(n)` | `VARCHAR(n)` |
| `CLOB`, `NCLOB`, `LONG`, `ROWID`, `UROWID`, `XMLTYPE`, `INTERVAL DAY TO SECOND` | `VARCHAR` |
| `JSON` | `VARCHAR` (a JSON string: `PARSE_JSON`) |
| `DATE` | `TIMESTAMP_NTZ(0)` (an Oracle `DATE` has a time of day) |
| `TIMESTAMP(n)` | `TIMESTAMP_NTZ(n)` |
| `TIMESTAMP(n) WITH TIME ZONE` | `TIMESTAMP_TZ(n)` with `"use_adbc": true`; `VARCHAR` with the native driver |
| `TIMESTAMP(n) WITH LOCAL TIME ZONE` | `TIMESTAMP_LTZ(n)` |
| `INTERVAL YEAR TO MONTH` | `NUMBER(38,0)` (total months) |
| `RAW`, `BLOB` | `BINARY` |

External tables read the two time-zone types as UTC, whatever the session time zone.

`--unbounded_number_type` (on `config create`) says what happens to a `NUMBER` declared without
precision, which can hold more digits than a Snowflake `NUMBER`:

| Value | Snowflake type | Values that do not fit |
|---|---|---|
| `decimal` (default) | `NUMBER(38,18)` | Snowflake rounds beyond the 18th decimal place (`1E-20` loads as `0`) and rejects values of 10^20 or more |
| `varchar` | `VARCHAR` | every digit kept, as text; exact only with `"use_adbc": true` (the native driver has already truncated to 18 decimal places, and fails above about 7.9E28) |
| `decimal_strict` | `NUMBER(38,18)` | the export fails, naming the column and the value (`1/3`, `1E-20`); needs `"use_adbc": true` |

- `varchar` and `decimal_strict` need a Snowflake publish target, and `decimal_strict` needs
  `"use_adbc": true`: otherwise `sync` stops before the export (exit 10). On a source other than
  Oracle the option is ignored with a warning.
- These types changed in 0.6.5. A full publish drops and recreates its tables. A table kept across
  runs (incremental or row-level strategies) keeps its old column types: drop it, or publish to a new
  schema, before the first run with 0.6.5.

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
- `auth_mode` is `token` only: `oauth` is rejected with an error (0.6.2+; it never ran an OAuth flow).

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
- `--pk_constraints` adds `PRIMARY KEY (...) NOT ENFORCED` to the native tables LakeXpress creates
  (0.6.2+; no effect before). An existing table kept for `append` is not changed.

## Amazon Redshift

- Provisioned cluster or Serverless; user with `CREATE SCHEMA` / `CREATE TABLE`; port 5439 reachable.
- `iam_role` that Redshift can assume (`redshift.amazonaws.com`), with S3 read access (both methods) and
  Glue access (external). Cluster and bucket in the **same region**. Without `iam_role` in the entry,
  publishing stops with an error (0.6.2+; before, it created empty tables and reported success).
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
