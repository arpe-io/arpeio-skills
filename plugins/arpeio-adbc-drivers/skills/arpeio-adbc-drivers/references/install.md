# arpe.io ADBC drivers – Install, versions, licence

## One-line installer

The installer downloads the driver library for the OS, checks it against the release `SHA256SUMS`,
installs it, writes an ADBC driver manifest (`<load-name>.toml`) so clients load it **by name**, and
optionally installs the licence.

```sh
# Linux (x64)
curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh \
  | sh -s -- arpemssql --license /path/to/your.lic
```

```powershell
# Windows (x64), PowerShell
& ([scriptblock]::Create((irm https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.ps1))) `
  arpemssql -License C:\path\to\your.lic
```

Load names: `arpemssql`, `arpepgsql`, `arpeoracle`, `arpenz` (Netezza), `arpeteradata`. macOS is refused
(`macOS is not supported yet`).

| `install.sh` | `install.ps1` | Meaning |
|---|---|---|
| `<driver>` | `<driver>` (first argument) | Load name to install |
| `--version X.Y.Z` | `-Version X.Y.Z` | Specific version (default `latest`) |
| `--user` (default) | `-Scope user` | Per-user install, no admin |
| `--system` | `-Scope system` | Machine-wide (sudo / elevated shell) |
| `--license <path>` | `-License <path>` | Copy a `.lic` file next to the driver as `arpeio_adbc.lic` |
| `--license-content <text>` | `-LicenseContent <text>` | Write the licence text (visible in the process list) |
| `--prefix <dir>` | `-Prefix <dir>` | Library install directory |
| `--download-only` | `-DownloadOnly` | Download binary + manifest + `SHA256SUMS` into `--dir`; nothing installed, no licence copied |
| `--offline` | `-Offline` | Install a bundle made with `--download-only` (from `--dir`) without network: normal managed install |
| `--skip-checksum` | `-SkipChecksum` | With `--offline` only: accept a bundle without `SHA256SUMS` |
| `--dir <dir>` | `-Dir <dir>` | Bundle directory (default: current directory) |
| `--list` | `-List` | Available drivers and latest versions (authoritative) |
| `--versions [<driver>]` | `-Versions [<driver>]` | Every published version, newest first |
| `--installed` | `-Installed` | Drivers installed here: version, scope, path, licence present |
| `--uninstall <driver>` | `-Uninstall <driver>` | Remove library, copied licence and manifest (`--system` for a machine install) |

```sh
curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh | sh -s -- --list
curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh | sh -s -- --installed
# air-gapped: on a connected machine, then copy ./bundle and install.sh, and finish offline
curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh | sh -s -- arpeoracle --download-only --dir ./bundle
sh install.sh arpeoracle --offline --dir ./bundle --license /path/to/your.lic
```

## Where things go

| | Linux user | Linux system | Windows user | Windows system |
|---|---|---|---|---|
| Library + `arpeio_adbc.lic` | `~/.local/lib/arpeio-adbc/<driver>/` | `/opt/arpeio-adbc/<driver>/` | `%LOCALAPPDATA%\arpeio-adbc\<driver>\` | `%ProgramFiles%\arpeio-adbc\<driver>\` |
| Manifest `<driver>.toml` | `${XDG_CONFIG_HOME:-~/.config}/adbc/drivers` | `/etc/adbc/drivers` | `%LOCALAPPDATA%\ADBC\Drivers` | `%ProgramData%\arpeio-adbc\adbc\drivers` (added to the machine `ADBC_DRIVER_PATH`) |

- Releases: https://github.com/arpe-io/adbc-drivers/releases, tag `<driver>-v<X.Y.Z>`
  (e.g. `arpeoracle-v0.4.2`, `arpeteradata-v0.3.5`), assets `lib<driver>_adbc_driver-linux-x64.so` /
  `<driver>_adbc_driver-win-x64.dll` + `SHA256SUMS`.
- Client can't find the driver: point `ADBC_DRIVER_PATH` at the manifest directory, or load the library
  by full path (`driver="/path/libarpepgsql_adbc_driver-linux-x64.so"`).
- Installed version: ADBC `GetInfo` (driver version).
- Windows: the DLLs are self-contained (OpenSSL and the MSVC runtime inside); the host process must be
  64-bit. Linux: glibc ≥ 2.28 (RHEL/Rocky/Alma 8+, Debian 10+, Ubuntu 20.04+; not RHEL 7).
- NuGet packages exist for .NET (e.g. `ArpePGSQL.AdbcDriver`).

## Licence

Binaries are free to download but need a valid Arpeio licence at runtime (no trial build). Licences:
sales@arpe.io. A licence file can hold blocks for several drivers, or a family grant covering all of them.

At install time the installer takes the first of: `--license`, `--license-content`,
`$ARPEIO_ADBC_LICENCE_FILE` (path), `$ARPEIO_ADBC_LICENCE` (content), and writes `arpeio_adbc.lic` next
to the library.

At runtime each driver takes the **first source present**, and uses only that one (a configured but
missing file is an error, it does not fall through):

| # | Source |
|---|---|
| 1 | database option `arpeio.adbc.license` (licence text) |
| 2 | database option `arpeio.adbc.license_file` (path) |
| 3 | env `ARPEIO_ADBC_LICENCE` (text) – British spelling |
| 4 | env `ARPEIO_ADBC_LICENCE_FILE` (path) |
| 5 | `arpeio_adbc.lic` in the folder of the loaded driver library |

- Checked at `AdbcDatabaseInit` and again at every `AdbcConnectionInit`.
- Status without connecting: `conn.adbc_database.get_option("arpeio.adbc.license.status")` →
  `<valid|grace|expired|none|invalid>;code=ARROW_LIC_…;tier=…;expires=<epoch>`.
- Grace period after expiry: works, prints `WARNING: <Driver> licence expired on YYYY-MM-DD. The driver
  will stop working in N day(s) …` on stderr.
- Error text: `<Driver> licence check failed [ARROW_LIC_<CODE>]: <message> (source: <source>)`.

| Code | Meaning / fix |
|---|---|
| `ARROW_LIC_NO_SOURCE` | No licence anywhere: install one (`--license`) or set an option / env var |
| `ARROW_LIC_FILE_NOT_FOUND` | A configured file (option or env var) is missing or unreadable: fix the path or unset it |
| `ARROW_LIC_MALFORMED` | Truncated or altered text (check line breaks when passing it inline) |
| `ARROW_LIC_EXPIRED` | Expired past grace: renew |
| `ARROW_LIC_WRONG_DRIVER` / `ARROW_LIC_NO_DRIVER_BLOCK` | Licence for another driver (the message lists what it covers) |
| `ARROW_LIC_BAD_SIGNATURE` | Altered or corrupt licence |
| `ARROW_LIC_UNKNOWN_KEY` | Driver too old for this licence: upgrade the driver |
| `ARROW_LIC_NOT_YET_VALID`, `ARROW_LIC_VERSION`, `ARROW_LIC_CAPABILITY` | Licence not valid yet / not for this version / missing capability |

`AdbcDatabaseInit has not succeeded on this database` on connect = the database init failed first
(usually the licence): fix that error.

Tools that embed these drivers (FastBCP, FastTransfer) pass their own licence to them.

## Installer troubleshooting

| Problem | Fix |
|---|---|
| Checksum verification failed | Truncated download or rewriting proxy: re-run, or download the asset and compare with `sha256sum` |
| Permission denied with `--system` | `sudo` / elevated shell, or per-user install |
| `driver not found` in the client | `--installed`, then `ADBC_DRIVER_PATH=<manifest dir>`; exact load name (`arpenz` for Netezza) |
| `ARROW_LIC_*` at connect | Licence (above) |
