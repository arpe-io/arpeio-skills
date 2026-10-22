# FastBCP – Install, license, Docker, environment variables, logging

## Install

FastBCP is a self-contained .NET binary: `FastBCP.exe` on Windows, `FastBCP` on Linux. Download it after a
trial request at https://arpe.io/get-trial?product=FastBCP.

```powershell
Expand-Archive -Path .\FastBCP.zip -DestinationPath .\FastBCP
.\FastBCP.exe --version
```

```bash
unzip FastBCP.zip -d FastBCP && cd FastBCP
chmod +x FastBCP
./FastBCP --version
```

Supported OS:

- **Windows x64**: 10/11, Server 2012–2025 (including Core/Nano).
- **Linux x64 and ARM64**: RHEL 8–10, Debian 12, Ubuntu 20.04–24.10, Alpine, Amazon Linux 2023,
  SLES, and others.
- **macOS**: use Docker.

## License

- Default lookup: `FastBCP.lic` in the same directory as the executable.
- Otherwise, `--license` accepts a file path, an HTTP(S) URL, or the license content:
  ```bash
  ./FastBCP ... --license "/opt/fastbcp/FastBCP.lic"
  ./FastBCP ... --license "$FASTBCP_LICENSE"        # content held in a shell variable
  ```
  ```powershell
  .\FastBCP.exe ... --license $env:FASTBCP_LICENSE
  ```
- After the trial period, a valid license is required. License errors: check the file location and the
  expiry date, then contact arpe.io.

## Docker

Image `arpeio/fastbcp`:

- Tags: `latest`, or `vX.Y.Z` (pin a version in production).
- Base image `debian:trixie-slim`; entrypoint `/usr/local/bin/FastBCP`.
- Volumes: `/config` (settings), `/data`, `/work`, `/logs`.
- Pass the **license content** with `--license`, not a host path.

```bash
export licenseContent=$(cat ./FastBCP.lic)
docker run --rm \
  -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY -e AWS_REGION \
  arpeio/fastbcp:latest \
  --connectiontype "mssql" \
  --server "host.docker.internal,1433" \
  --user "$FASTBCP_USER" --password "$FASTBCP_PASSWORD" \
  --database "tpch_test" \
  --query "SELECT * FROM dbo.orders WHERE year(o_orderdate)=1998" \
  --directory "s3://my-bucket/exports/" \
  --fileoutput "orders.parquet" \
  --parallelmethod "Ntile" --distributekeycolumn "o_orderkey" --paralleldegree 12 \
  --merge false \
  --license "$licenseContent"
```

- Inside a container, `localhost` is the container itself. Use `host.docker.internal` (Docker Desktop)
  or the host's IP to reach a database on the host.
- For local output, mount a volume and write to it: `-v /srv/exports:/data … --directory "/data"`.
- Prefer `--env-file`, Docker/Compose/Kubernetes secrets or managed identities over inline values,
  and keep the license out of shell history.
- `docker run --rm arpeio/fastbcp:latest` with no arguments prints the help.

The same image is used by the Airflow (DockerOperator) and Kestra (Docker Run) integrations. A
PostgreSQL extension, `pg_fastbcp`, can also run FastBCP from SQL (PG 15–18).

## Environment variables read by FastBCP

| Variable | Default | Purpose |
|---|---|---|
| `FASTBCP_DBPARTITION_TARGET_MODE` | `CHUNK` | `CHUNK` / `HIVE` file layout for `DBPartition` (1.3+) |
| `FASTBCP_TTU_PATH` | not set | Linux, `adbc_teradata` legacy-port logon only (1.4+): Teradata Tools and Utilities client directory, e.g. `/opt/teradata/client/20.00`. Give the versioned client folder, not just `libcliv2.so` |
| `FASTBCP_RGSIZE` | `max(2^18/sqrt(degree), 64000)` | Parquet row group size (rows). Lower it to reduce memory |
| `FASTBCP_DATEFORMAT` | `yyyy-MM-dd HH:mm:sszzz` | Default date-to-string format |
| `FASTBCP_CLOUD_ISBUFFERED` | `False` | Buffer cloud uploads through a temp file (recommended for ADLS/OneLake) |
| `FASTBCP_TMPPATH` | system temp | Temp dir for buffered uploads |
| `FASTBCP_CLOUD_MAXUPLOADTHREAD_PER_STREAM` | 8 (method None) / 2 | Upload threads per output stream |
| `FASTBCP_UID_STRING_CASING` | `lower` | Casing of GUID/UUID values written as strings (`lower` / `upper`) |
| `FASTBCP_ADBC_BATCHCELLS` | `500000` | ADBC fetch batch size, in cells |
| `FASTBCP_ADBC_ENCRYPTION`, `FASTBCP_ADBC_DATA_INTEGRITY` | negotiated | Oracle ADBC network encryption / integrity override |
| `FASTBCP_SYSTEMINFO_COLLECT` | `False` | Collect system info during the run |
| `FASTBCP_PERFORMANCE_COLLECT` | `False` | Collect performance info during the run |

Also read: the cloud provider variables (see [outputs.md](outputs.md)) and, for PostgreSQL,
`PGUSER`/`PGPASSWORD`/`PGPASSFILE`/`PGSSLROOTCERT`. For Oracle, `TNS_ADMIN`.

Set them with `export NAME=value` (bash) or `$env:NAME = "value"` (PowerShell) before running.

## Logging

- Console logging is always on. `--loglevel Debug` gives detailed diagnostics.
- `FastBCP_settings.json` (Serilog) adds sinks. FastBCP looks for it in the working directory, then
  next to the executable, or at the path given by `--settingsfile`.
  - **Console**, **File** (rolling, JSON); file paths can use `{logdate}`, `{logtimestamp}`,
    `{sourcedatabase}`, `{sourceschema}`, `{sourcetable}`, `{filename}`, `{runid}`, `{traceid}`.
  - **Database**: SQL Server (`MSSqlServer`) or PostgreSQL. The log table is created automatically,
    with no indexes and no purge. If the log database is unreachable, the run fails.
  - **OpenTelemetry**: configured in the settings file or with `OTEL_EXPORTER_OTLP_ENDPOINT`.
    Use `--traceparent` to join an upstream trace.
- Level: `MinimumLevel` in the settings file; `--loglevel` on the command line overrides it.
- Passwords (from `--password` or `--sourceconnectstring`) are masked in every sink.
- Always set `--runid` from the scheduler (Airflow run id, SQL Agent job id…) to correlate logs.

Minimal settings file (console + daily rolling JSON files):

```json
{
  "Serilog": {
    "Using": ["Serilog.Sinks.Console", "Serilog.Sinks.File"],
    "MinimumLevel": "Information",
    "WriteTo": [
      { "Name": "Console" },
      {
        "Name": "File",
        "Args": {
          "path": "Logs/FastBCP_.json",
          "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 10
        }
      }
    ]
  }
}
```

The docs have full samples (Console + File + SQL Server/PostgreSQL sinks):
https://fastbcp-docs.arpe.io/latest/documentation/logging/examples
