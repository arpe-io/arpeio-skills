# FastTransfer – Install, license, Docker, environment variables, logging

## Install

FastTransfer is a self-contained .NET 10 binary: `FastTransfer.exe` on Windows, `FastTransfer` on Linux.
Download it after a trial request at https://arpe.io/get-trial?product=FastTransfer.

```powershell
Expand-Archive -Path .\FastTransfer.zip -DestinationPath .\FastTransfer
.\FastTransfer.exe --version
```

```bash
unzip FastTransfer.zip -d FastTransfer && cd FastTransfer
chmod +x FastTransfer
./FastTransfer --version
```

Supported OS:

- **Windows x64**: 10/11, Server 2012–2025.
- **Linux x64 and ARM64**: RHEL 8–10, Debian 12, Ubuntu 20.04–24.10, Alpine, Amazon/Azure Linux, SLES,
  and others.
- **macOS**: use Docker.

Embedded drivers: ClickHouse, DuckDB, SQL Server, MySQL, Netezza, Oracle, PostgreSQL, SAP HANA,
Teradata. Others need an ODBC or OLE DB driver installed.

## License

- Default lookup: `FastTransfer.lic` next to the executable.
- `--license` accepts a file path, a UNC path, an HTTP(S) URL, or the license content:
  ```bash
  ./FastTransfer ... --license "/opt/fasttransfer/FastTransfer.lic"
  ./FastTransfer ... --license "$FT_LICENSE"          # content held in a shell variable
  ```
- The trial mode is reported in the log ("trial mode will end on …"). A license is needed after the trial.

## Docker

Image `arpeio/fasttransfer`:

- Tags: `latest`, or `vX.Y.Z` (pin a version in production). Images are rebuilt weekly for security patches.
- Base image `debian:trixie-slim`; entrypoint `/usr/local/bin/FastTransfer`.
- Volumes: `/config` (settings files), `/data` (input files), `/work` (working directory), `/logs`.
- Pass the **license content** with `--license "$licenseContent"`.

```bash
export licenseContent=$(cat ./FastTransfer.lic)
docker run --rm \
  -v /local/data:/data \
  arpeio/fasttransfer:latest \
  --sourceconnectiontype "duckdbstream" \
  --sourceserver ":memory:" \
  --query "SELECT * FROM read_csv('/data/lineitem*.CSV')" \
  --targetconnectiontype "msbulk" \
  --targetserver "host.docker.internal,1433" \
  --targetuser "$FT_TARGET_USER" --targetpassword "$FT_TARGET_PASSWORD" \
  --targetdatabase "tpch_test" --targetschema "dbo" --targettable "lineitem" \
  --parallelmethod "Random" --distributekeycolumn "l_orderkey" \
  --mapmethod "Name" --loadmode "Truncate" --degree 16 \
  --license "$licenseContent"
```

- Inside a container, `localhost` is the container. Use `host.docker.internal` or the host IP to reach a
  database on the host. Integrated (trusted) auth generally doesn't work from a Linux container; use SQL
  logins.
- File paths in `read_*` are container paths (`/data/...`): mount the folder with `-v`.
- `docker run --rm arpeio/fasttransfer:latest '-?'` prints the help.

Integrations: Airflow (DockerOperator), Kestra (Docker Run), the `pg_fasttransfer` PostgreSQL extension
(PG 15–18), and FastWrappers-TSQL for SQL Server (`dbo.xp_RunFastTransfer_secure`).

## Environment variables read by FastTransfer

| Variable | Applies to | Default | Purpose |
|---|---|---|---|
| `FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES` | `msbulk`, `orabulk` | `60` | Timeout of a bulk load step, in minutes. Raise it for very large loads (0.17+) |
| `FASTTRANSFER_SORTEDINPUT` | `msbulk` | `false` | `true` tells SQL Server bulk copy the input is sorted on the clustered index (0.17+) |

```bash
export FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES=120
```

```powershell
$env:FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES = "120"
```

(cmd.exe: `set FASTTRANSFER_LOADSTEPTIMEOUT_MINUTES=120`)

## Logging

- Console logging is always on. `--loglevel debug` gives detailed diagnostics; `verbose` is even more
  detailed.
- `FastTransfer_Settings.json` (Serilog) adds sinks. FastTransfer looks for it in the working directory,
  then next to the executable, or at the path given by `--settingsfile`. If none is found, it logs to the
  console only. On Linux the name is case-sensitive.
  - **Console**, **File**, **Map**: file paths can use `{logdate}`, `{logtimestamp}`, `{sourcedatabase}`,
    `{sourceschema}`, `{sourcetable}`, `{filename}`, `{runid}`, `{traceid}`.
  - **Database**: SQL Server (connection string key `MS_FastTransferLogs`) or PostgreSQL
    (`PG_FastTransferLogs`). If the log database is unreachable, FastTransfer fails and refuses to continue.
- `MinimumLevel` in the settings file sets the level; `--loglevel` overrides it. `Fatal` is only available
  through the settings file.
- Passwords are masked in all sinks.
- Set `--runid` from the scheduler to correlate runs.

Full settings samples: https://fasttransfer-docs.arpe.io/latest/documentation/logging/examples
