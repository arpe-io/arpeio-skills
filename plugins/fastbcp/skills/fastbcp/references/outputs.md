# FastBCP – Outputs, formats and cloud targets

## Format = extension of `--fileoutput`

| Extension | Format | Notes |
|---|---|---|
| `.parquet` | Parquet | Recommended. Only format allowed with `adbc_*` connectors |
| `.csv` | CSV | Delimiter configurable |
| `.tsv` | TSV | Tab separated |
| `.json` | JSON | |
| `.bson` | BSON | Binary JSON |
| `.xlsx` | Excel | Large datasets may be split across several sheets |
| `.bin` | PostgreSQL COPY binary | |
| `.csv.gz`, `.tsv.gz`, `.json.gz` | Gzip-compressed text | Streamed gzip |

`--encoding` (default `UTF-8`) applies to CSV, TSV and JSON only, not Parquet.

## CSV / TSV options

| Long | Short | Default | Notes |
|---|---|---|---|
| `--delimiter` | `-d` | `,` | e.g. `"\t"`, `"|"`, `";"` |
| `--quotes` | `-t` | off | Wraps every field in `"`. Embedded `"` becomes `\"` |
| `--dateformat` | `-f` | `yyyy-MM-dd HH:mm:sszzz` | .NET codes: `yyyy MM dd HH mm ss zzz` |
| `--decimalseparator` | `-n` | `.` | |
| `--noheader` | `-h` | header written | |
| `--boolformat` | `-b` | `True/False` | `true/false`, `t/f`, `1/0`, `yes/no`, `y/n` |

The date format adapts to the column type:

- date-only columns always come out as `yyyy-MM-dd`;
- datetime/timestamp columns come out as `yyyy-MM-dd HH:mm:ss` even if the format contains `zzz`;
- DateTimeOffset columns keep the offset.

The default changed in 1.1 (it was `yyyy-MM-dd`). It can also be set globally with `FASTBCP_DATEFORMAT`.

## Parquet options

- `--parquetcompression`: `Zstd` (default; usually best), `Snappy`, `Gzip`, `Lz4`, `Lzo`, `None`.
- Row group size: env var `FASTBCP_RGSIZE` (default `max(2^18/sqrt(degree), 64000)` rows). Lower it
  to reduce memory use.

## Output naming

- Tokens in `--directory` and `--fileoutput`: `{sourcedatabase}`, `{sourceschema}`, `{sourcetable}`,
  `{starttimestamp}` (yyyyMMddHHmmss), `{starttimestamp:FORMAT}`, `{startdate}` (yyyyMMdd),
  `{starttime}` (HHmmss), `{starthour}` (HH), `{user}`, `{filetype}`, `{datadrivencolumn}`.
- If you use `{sourceschema}` / `{sourcetable}`, you must pass `--sourceschema` / `--sourcetable`,
  even when exporting a `--query`.
- `--timestamped` appends `_yyyyMMddHHmmss` to the file name. Tokens give more control.
- Without merge, a parallel export writes one file per chunk, e.g. `orders_part_Chunk000.parquet`.
  The exact names depend on the method.

## Destinations (`--directory`)

| Target | Value |
|---|---|
| Local | `/exports/sales` or `C:\exports\sales` |
| Windows share | `\\server\share\exports` |
| AWS S3 | `s3://bucket/path/` |
| S3-compatible (MinIO, OVH, Wasabi, DigitalOcean Spaces) | `s3://bucket/path/` + env `S3_ENDPOINT=https://…` |
| Azure Blob | `abs://account.blob.core.windows.net/container/path/` |
| Azure Data Lake Gen2 | `abfss://account.dfs.core.windows.net/container/path/` |
| Google Cloud Storage | `gs://bucket/path/` |
| Microsoft OneLake (Fabric) | `onelake://workspace/lakehouse/path/` (the docs also show `onelake://workspace.dfs.core.windows.net/lakehouse/path`) |

### Cloud credentials

Never put keys on the command line. FastBCP looks for credentials in this order: `--cloudprofile <name>`,
then the provider's standard env vars, then managed identity / IAM role.

| Provider | Credentials |
|---|---|
| AWS / S3-compatible | `--cloudprofile` (AWS profile), or `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`, optional `AWS_SESSION_TOKEN`; IAM role on EC2/ECS |
| Azure Blob / ADLS | `az login` (Azure CLI), managed identity, `AZURE_STORAGE_CONNECTION_STRING`, `AZURE_STORAGE_SAS_URL` (1.3+; quote it because it contains `&`), or service principal `AZURE_TENANT_ID` / `AZURE_CLIENT_ID` / `AZURE_CLIENT_SECRET` |
| GCS | `gcloud auth application-default login`, or `GOOGLE_APPLICATION_CREDENTIALS=/path/key.json` |
| OneLake | Entra service principal through `AZURE_TENANT_ID` / `AZURE_CLIENT_ID` / `AZURE_CLIENT_SECRET`. The principal needs a role on the Fabric workspace |

### Cloud behaviour and tuning

- Uploads are multipart. Advise a bucket lifecycle rule that deletes incomplete multipart uploads,
  because a killed run leaves them behind and they are billed.
- `--merge true` is **not available** for cloud destinations. A parallel export produces several files.
- Azure: a missing container is created. HTTP 403 usually means a firewall, RBAC or SAS permission issue.
- `FASTBCP_CLOUD_ISBUFFERED=True` (buffered upload through a temp file in `FASTBCP_TMPPATH`) is
  recommended for ADLS Gen2 / OneLake.
- `FASTBCP_CLOUD_MAXUPLOADTHREAD_PER_STREAM`: upload threads per stream (default 8 with method `None`,
  2 otherwise).
- Prefer Parquet for lake destinations. Hive-style paths (`year=2024/`) can be written directly in
  `--directory`, or produced by `Timepartition` / `DBPartition` with `HIVE` mode.
