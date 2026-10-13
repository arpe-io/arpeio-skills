# LakeXpress – Credentials file (`--auth_file`)

One JSON object. Each top-level key is an id you choose and pass on the command line
(`--lxdb_auth_id`, `--source_db_auth_id`, `--target_storage_id`, `--publish_target`). An entry has
`ds_type`, `auth_mode` and `info`. The file can hold any number of entries.

## Environment variables: `$env{VAR}`

- Any string value can contain `$env{VAR_NAME}`, alone or inside a longer string (e.g. a connection
  string). Values without `$env{` are used as written.
- The **whole file** is expanded when it is read: a `$env{}` that is not set, **in any entry**, stops
  every command (exit 10, `Environment variable 'X' is not set`), even if that entry is not used.
- `$VAR` / `${VAR}` are **not** expanded (only a deprecation warning).
- Restrict the file (`chmod 600 credentials.json`); use one file per environment.

## LakeXpress DB

LakeXpress creates its tables on `lxdb init` or on first use. The entry is checked strictly: it must have
`ds_type`, `auth_mode` and `info`; for server databases (not `connection_string`) `info.server` and
`info.database` are required.

| Backend | Entry | Database to create first |
|---|---|---|
| PostgreSQL (recommended) | `"ds_type": "postgres", "auth_mode": "classic"`, info `username`, `password`, `server`, `port`, `database` | `CREATE USER lakexpress_admin WITH PASSWORD '…'; CREATE DATABASE lakexpress_log OWNER lakexpress_admin;` (owner needed on PostgreSQL 15+; tables in `public`) |
| SQL Server | `"ds_type": "mssql"`, same info (+ `trust_server_certificate`) | `CREATE DATABASE lakexpress_log;` + a login/user in `db_owner` (tables and sequences in `dbo`) |
| MySQL | `"ds_type": "mysql"`, same info | `CREATE DATABASE lakexpress_log CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;` + `GRANT ALL PRIVILEGES ON lakexpress_log.*` |
| SQLite | `"ds_type": "sqlite", "auth_mode": "filesystem", "info": {"filepath": "/abs/path/lakexpress_log.sqlite"}` | nothing (file and folders created) |
| DuckDB | `"ds_type": "duckdb", "auth_mode": "filesystem", "info": {"filepath": "/abs/path/lakexpress_log.duckdb"}` | nothing; `:memory:` not supported |

- SQLite / DuckDB: use an **absolute** path (a relative one is resolved from the current directory, so
  another directory means a new, empty LakeXpress DB); one machine, one sync at a time.
- For incremental syncs use PostgreSQL, SQL Server or SQLite (troubleshooting.md: DuckDB).
- Oracle, MariaDB, SAP HANA and Teradata cannot be the LakeXpress DB.

```json
{
  "lxdb_postgres": {
    "ds_type": "postgres",
    "auth_mode": "classic",
    "info": {
      "username": "$env{LX_LXDB_USER}",
      "password": "$env{LX_LXDB_PASSWORD}",
      "server": "pg.company.com",
      "port": 5432,
      "database": "lakexpress_log"
    }
  }
}
```

## Sources

The source user only needs read access. `info.database` is the default source database
(`config create --source_db_name` overrides it).

| `ds_type` | `auth_mode` and `info` | Read permissions (docs) |
|---|---|---|
| `postgres` | `classic`: `username`, `password`, `server`, `port` (5432), `database`, `sslmode` (default `prefer`). `connection_string`: `connection_string` only (libpq form, e.g. `host=… port=5432 dbname=… user=… password=… sslmode=require`) | `GRANT USAGE ON SCHEMA s`, `GRANT SELECT ON ALL TABLES IN SCHEMA s` |
| `mssql` | `classic` (`username`, `password`, `server`, `port` 1433, `database`, `encrypt` default `true`, `trust_server_certificate` default `false`); `integrated` (Windows auth: `server`, `port`, `database`, no user/password); `connection_string` (ODBC string, e.g. `DRIVER={ODBC Driver 18 for SQL Server};SERVER=host,1433;DATABASE=…;UID=…;PWD=…;Encrypt=yes;TrustServerCertificate=yes;`) | `GRANT SELECT ON SCHEMA::dbo`, `GRANT VIEW DEFINITION ON SCHEMA::dbo` |
| `oracle` | `classic` only: `username`, `password`, `server`, `port` (1521), `database` (**service name or SID**), `lib_dir` (Instant Client folder = thick mode, needed for Oracle 11.2 and earlier; omit for thin mode, 12.1+) | `GRANT SELECT_CATALOG_ROLE` (Rowid parallel export, metadata) |
| `mysql` | `classic` only: `username`, `password`, `server` (default `localhost`), `port` (3306), `database` (optional) | `GRANT SELECT ON db.*` |
| `mariadb` | as `mysql` (use `mariadb`, not `mysql`, to label the source correctly) | `GRANT SELECT ON db.*` |
| `saphana` | `classic`: `username`, `password`, `server`, `port` (39017), `database` (tenant, MDC only). `connection_string`: ODBC string `DRIVER={HDBODBC};SERVERNODE=host:39017;DATABASE=…;UID=…;PWD=…` | `SELECT` on the schema, `SYS.TABLES`, `SYS.COLUMNS` |
| `teradata` | `classic`: `username`, `password`, `server`, `port` (1025), `database` (optional). `connection_string`: a JSON string of `teradatasql` parameters, e.g. `"{\"host\":\"…\",\"dbs_port\":\"1025\",\"user\":\"…\",\"password\":\"…\",\"database\":\"DW\"}"` | `SELECT` on the database, `DBC.TablesV`, `DBC.ColumnsV`, `DBC.IndicesV`, `DBC.All_RI_ChildrenV` |

- SQL Server: LakeXpress uses the ODBC driver and falls back to `pymssql` if none is installed;
  `pymssql` supports `classic` only. Self-signed certificate (test servers, `localhost`): add
  `"trust_server_certificate": true` to stay on the ODBC path.
- SAP HANA: FastBCP needs the HDBDOTNETCORE runtime; set the `HDBDOTNETCORE` environment variable to its
  folder before running LakeXpress.
- `port` is a JSON number.

```json
"source_oracle": {
  "ds_type": "oracle",
  "auth_mode": "classic",
  "info": {
    "username": "$env{LX_ORACLE_USER}",
    "password": "$env{LX_ORACLE_PASSWORD}",
    "server": "oracle-prod.company.com",
    "port": 1521,
    "database": "PRODPDB"
  }
}
```

## Storage (`--target_storage_id`)

Local disk needs no entry: `config create --output_dir PATH`. Cloud storage entries:

| `ds_type` | `auth_mode` | `info` |
|---|---|---|
| `s3` (AWS and S3-compatible: MinIO, OVH, Wasabi…) | `profile` | `directory` (`s3://bucket/path`), `profile` (AWS profile in `~/.aws/credentials`; for S3-compatible storage put `endpoint_url` and `region` in `~/.aws/config`) |
| `gcs` | `profile` | `directory` (`gs://bucket/path`), `profile` (**path** of a service-account JSON key) |
| `azure` | `service_principal` | `directory` (`abfss://container@account.dfs.core.windows.net/path` for ADLS Gen2, `abs://container@account.blob.core.windows.net/path` for Blob), `azure_client_id`, `azure_tenant_id`, `azure_client_secret` |
| `onelake` | `service_principal` | `directory` (`onelake://workspace/lakehouse/path/`, trailing `/`), `azure_client_id`, `azure_tenant_id`, `azure_client_secret` |

Permissions: S3 `s3:PutObject`, `s3:PutObjectAcl`, `s3:GetObject`, `s3:ListBucket`; GCS `Storage Object
Creator` (+ `Storage Object Viewer`); Azure `Storage Blob Data Contributor` on the account; OneLake: the
service principal is Contributor (or Admin) of the Fabric workspace. Older `ds_type` values ending in
`_s3` (e.g. `minio_s3`) still work; prefer `s3`.

```json
"s3_01": {
  "ds_type": "s3",
  "auth_mode": "profile",
  "info": { "directory": "s3://my-bucket/lakexpress/exports", "profile": "my-aws-profile" }
}
```

```ini
# ~/.aws/config for MinIO
[profile minio]
endpoint_url = http://localhost:9000
region = us-east-1
```

## Publish targets (`--publish_target`)

| Target | `ds_type` | `auth_mode` and `info` |
|---|---|---|
| Snowflake | `snowflake` | `password`: `account`, `user`, `password`, `warehouse`, `database`, `stage`. `pat`: `token` instead of `password` (needs a network policy for the user). Also `keypair` (`private_key_path`, optional `private_key_passphrase`) and `oauth` (`oauth_token` or `authenticator`) |
| Databricks | `databricks` | `token`: `host` (workspace host name), `http_path` (SQL warehouse), `access_token` (PAT), `catalog` (Unity Catalog, required), optional `schema`. `oauth` also exists |
| Microsoft Fabric Lakehouse | `fabric` + **`"fabric_target": "lakehouse"`** at the entry's top level | `service_principal`: `workspace_id`, `lakehouse_id`, `lakehouse_name`, `sql_endpoint` (host name only), `azure_client_id`, `azure_tenant_id`, `azure_client_secret` |
| BigQuery | `bigquery` | `profile`: `project_id`, `location` (default `US`; match the bucket region), `profile` (service-account JSON key path) |
| Amazon Redshift | `redshift` | `classic`: `host`, `port` (5439), `database` (`dev`), `username`, `password`, `iam_role` (ARN, required), `region` (default `us-east-1`). `iam`: `host`, `port`, `database`, `cluster_identifier`, `iam_role`, `region` |
| MotherDuck | `motherduck` | `token`: `database`, `token`. `env`: `database` only, token read from the `motherduck_token` environment variable |
| AWS Glue | `aws_glue` (or `glue`) | `profile`: `profile`, `region`. `keys`: `aws_access_key_id`, `aws_secret_access_key`, `region`. `role`: `role_arn`, optional `external_id`, `region`. All accept optional `catalog_id` (another account's catalog) |
| DuckLake | `sqlite` / `duckdb` (`info.filepath`, default `ducklake_catalog.sqlite` / `.duckdb` in the working directory) or `postgres` / `mysql` (`server`, `port`, `database`, `username`, `password`; `postgres` also `schema` for the catalog's own tables) | The entry **is** the DuckLake catalog |

- Fabric: the Workspace and Lakehouse ids are in the Lakehouse URL
  `https://app.fabric.microsoft.com/groups/{workspace_id}/lakehouses/{lakehouse_id}`; the SQL endpoint
  host comes from "SQL analytics endpoint → Copy SQL connection string". Without `fabric_target` (or with
  `"warehouse"`), the Fabric **Warehouse** publisher is used, which the docs do not describe.
- BigQuery, Databricks, Glue, Redshift and Snowflake entries always come with a storage entry (GCS for
  BigQuery; S3 for Glue and Redshift; the stage location for Snowflake).
- Full setup per platform: publishing.md.

## Complete file (PostgreSQL → S3 → Snowflake)

```json
{
  "lxdb_postgres": {
    "ds_type": "postgres",
    "auth_mode": "classic",
    "info": {
      "username": "$env{LX_LXDB_USER}",
      "password": "$env{LX_LXDB_PASSWORD}",
      "server": "localhost",
      "port": 5432,
      "database": "lakexpress_log"
    }
  },
  "source_postgres": {
    "ds_type": "postgres",
    "auth_mode": "classic",
    "info": {
      "username": "$env{LX_PG_USER}",
      "password": "$env{LX_PG_PASSWORD}",
      "server": "pg-prod.company.com",
      "port": 5432,
      "database": "production_db"
    }
  },
  "s3_01": {
    "ds_type": "s3",
    "auth_mode": "profile",
    "info": { "directory": "s3://my-datalake-bucket/lakexpress/", "profile": "lakexpress" }
  },
  "snowflake_prod": {
    "ds_type": "snowflake",
    "auth_mode": "password",
    "info": {
      "account": "myorg-myaccount",
      "user": "LAKEXPRESS_USER",
      "password": "$env{LX_SNOWFLAKE_PASSWORD}",
      "warehouse": "LAKEXPRESS_WH",
      "database": "LAKEXPRESS_DB",
      "stage": "LAKEXPRESS_STAGE"
    }
  }
}
```
