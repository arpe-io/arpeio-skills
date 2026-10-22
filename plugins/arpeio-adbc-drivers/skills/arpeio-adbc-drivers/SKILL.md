---
name: arpeio-adbc-drivers
description: Install, configure and use the arpe.io (Arpeio) ADBC drivers - ArpeMSSQL (`arpemssql`, SQL Server / Azure SQL / Fabric), ArpePGSQL (`arpepgsql`, PostgreSQL), ArpeOracle (`arpeoracle`, Oracle / Autonomous Database), ArpeNetezza (`arpenz`, IBM Netezza) and ArpeTeradata (`arpeteradata`, Teradata / VantageCloud). Native wire-protocol drivers returning Apache Arrow, loaded by name from Python (adbc_driver_manager), C/C++, C#, Go, Java, R or Rust. Use when the user mentions one of these drivers or arpe.io ADBC, or wants to read a SQL Server / PostgreSQL / Oracle / Netezza / Teradata table into Arrow, pandas or Parquet through ADBC, bulk-load Arrow data into one of them, write a connection string or URI for these drivers, set up their licence, or troubleshoot a connection, TLS, authentication or ingest error.
---

# arpe.io ADBC drivers

Five pure-native [ADBC](https://arrow.apache.org/adbc/) drivers. Each speaks its database's wire
protocol directly (no ODBC, no vendor client library) and moves Apache Arrow both ways: query results
come back as Arrow record batches, and Arrow data is bulk-loaded with ADBC ingest.

Goal: give **working code** (Python by default) or a correct install command, built only from
documented options. Never invent hosts, credentials or names: ask. Never write real passwords or licence
text into code you show; use placeholders or environment variables.

Facts below are checked against the docs and the driver code at the latest release tags: **ArpeMSSQL
0.7.6, ArpePGSQL 0.4.7, ArpeOracle 0.4.2, ArpeNetezza 0.3.7, ArpeTeradata 0.3.5** (releases are frequent;
the docs site can lag a patch behind). References (load only what you need):

| Need | File |
|---|---|
| Installer, versions, manifests, licence | [references/install.md](references/install.md) |
| Using a driver from Python / C / C# / Go / Java / R / Rust, query, parameters, ingest, transactions | [references/clients.md](references/clients.md) |
| SQL Server driver: options, URI, connection string, auth (SQL, Kerberos / SSPI, Entra ID), TLS, TDS 8.0, types, bulk insert | [references/arpemssql.md](references/arpemssql.md) |
| PostgreSQL driver: options, URI, connection string, SCRAM, sslmode, `.pgpass`, types, COPY ingest | [references/arpepgsql.md](references/arpepgsql.md) |
| Oracle driver: options, EZConnect, TNS, tnsnames, wallets, IAM / Entra tokens, NNE, types, direct path ingest | [references/arpeoracle.md](references/arpeoracle.md) |
| Netezza driver: options, URI, TLS (NPS 11 / NPS 7), types, external-table ingest | [references/arpenetezza.md](references/arpenetezza.md) |
| Teradata driver: options, URI, connection string, TLS vs legacy port (TTU), TD2 / LDAP / JWT / Kerberos, cloud, types, ingest | [references/arpeteradata.md](references/arpeteradata.md) |

Official docs: https://adbc-drivers-docs.arpe.io/

## The five drivers

| | ArpeMSSQL | ArpePGSQL | ArpeOracle | ArpeNetezza | ArpeTeradata |
|---|---|---|---|---|---|
| Database | SQL Server 2016–2025, Azure SQL, Fabric | PostgreSQL 12–17 (+ PG-wire engines) | Oracle 12.1+ (19c, 21c, 23ai/26ai), Autonomous DB | Netezza NPS 11 (IBM Cloud), NPS 7 | Teradata 20.00 (TLS logon: 16.20.53.30+), cloud |
| Load name (`driver=`) | `arpemssql` | `arpepgsql` | `arpeoracle` | **`arpenz`** | `arpeteradata` |
| Option prefix | `adbc.arpemssql.*` | `adbc.arpepgsql.*` | `adbc.arpeoracle.*` | `adbc.arpenz.*` | `adbc.arpeteradata.*` |
| URI schemes | `sqlserver://`, `mssql://` | `postgresql://`, `postgres://` | `oracle://` | `netezza://`, `postgresql://` | `teradata://` |
| URI path means | **instance name** (database in `?database=`) | database | service name | database | default database |
| Default port | 1433 | 5432 | 1521 | 5480 | 443 (TLS), 1025 (legacy) |
| Parameter marker | `?` (or `@p1…`, 0.7.6) | `$1, $2…` | `:1` / `:name` | `?` or `$1` | `?` |
| Default transport security | login-only TLS (`encrypt=false`) | `sslmode=prefer` | none (`ssl_mode=disable`, NNE `accepted`) | `sslmode=prefer` | `sslmode=prefer` (TLS unverified, else legacy port in clear) |
| Ingest mechanism | native `INSERT BULK` | `COPY … FROM STDIN (FORMAT binary)` | direct path load (autocommit on), else array-bound `INSERT` | external table, INSERT fallback | packed parameterised `INSERT` |

All five: Linux x64 (glibc ≥ 2.28) and Windows x64 only (no macOS), ADBC API 1.1, OpenSSL built in, a
licence required at runtime (no trial build). ArpeTeradata needs Teradata's TTU (CLIv2) only for the
legacy port; its TLS logon needs nothing from Teradata.

## Check the installed version

When you can run commands, check what is installed before writing code; otherwise ask when it matters
(an unknown option, an unexpected error):

```sh
curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh | sh -s -- --installed
```

```python
conn.adbc_get_info()["driver_version"]     # from an open connection
```

- **Same as above** (MSSQL 0.7.6, PGSQL 0.4.7, Oracle 0.4.2, Netezza 0.3.7, Teradata 0.3.5) → use this
  skill as is.
- **Older** → options and fixes noted "since …" in the driver file may be missing (e.g. MSSQL Linux
  Kerberos, Entra `Authentication=` and TDS 8.0 need 0.7.5, WKB into geometry 0.7.6; Oracle direct path
  ingest 0.4.0, token logins over TLS only 0.4.1): suggest upgrading with the installer.
- **Newer** → this skill may miss new options or changed defaults: confirm anything you rely on in
  https://adbc-drivers-docs.arpe.io/llms-full.txt (whole documentation in one file), and say that you
  did. The installer's `--list` gives the latest published versions.

## Workflow

1. **Install** with the one-line installer and the user's licence (install.md):
   `curl -fsSL https://raw.githubusercontent.com/arpe-io/adbc-drivers/main/install.sh | sh -s -- <load-name> --license /path/to/your.lic`.
   Python also needs `pip install adbc-driver-manager pyarrow`. ArpeTeradata on the legacy port also
   needs TTU CLIv2 17.10+ installed (not for TLS).
2. **Connect**: pick one form per need (driver file):
   - a **URI** (`db_kwargs={"uri": "…"}`) for simple cases;
   - an ADO.NET **connection string** (`adbc.arpe<db>.connection_string`) when the user already has one;
   - **discrete options** (`adbc.arpe<db>.<field>`, `adbc.arpenz.*` for Netezza) for everything else, and
     for the options that have no URI / connection-string form (TLS files, type mappings, timeouts,
     licence).
   A discrete option always wins over the same field from a URI or connection string.
3. **Secure it**: choose the TLS setting explicitly (verified TLS for anything outside a lab), and the
   auth mode the driver really supports on the user's OS (see hard rules).
4. **Query / ingest** (clients.md): `cursor.execute(…)` + `fetch_arrow_table()` / `fetch_record_batch()`;
   `cursor.adbc_ingest(table, data, mode=…)`.
5. **Output**: the install command, then one complete code block, then a short note on assumptions
   (TLS mode, autocommit, names' case).

## Hard rules

- **Python `dbapi.connect()` starts with autocommit OFF**: call `conn.commit()` after DML or
  `adbc_ingest`, or pass `autocommit=True`. Otherwise the data is rolled back at close.
- **Load names are exact**: `arpenz` for Netezza (not `arpenetezza`). Set every option in `db_kwargs`
  (database options). Use the full `adbc.arpe<db>.*` keys there: some bare aliases are connection-only
  (e.g. PostgreSQL `sslmode` → use `adbc.arpepgsql.sslmode`).
- **Integrated / Kerberos auth**:
  - ArpeMSSQL: Windows SSPI (NTLM fallback unless `require_kerberos=true`), and Linux Kerberos since
    0.7.5 (needs MIT Kerberos on the host: `krb5-libs` / `libgssapi-krb5-2`).
  - ArpePGSQL: Windows SSPI only.
  - ArpeOracle, ArpeNetezza: none in released binaries, even though the code accepts the options.
  - ArpeTeradata: `logmech=KRB5` / `TDNEGO` over TLS on Linux (MIT `libgssapi_krb5.so.2`; Windows SSPI
    built but not validated), or on the legacy port through TTU. LDAP works **only on the legacy port**.
  Otherwise use passwords (or Entra ID / OCI IAM / JWT tokens where supported).
- **TLS trust store**: ArpePGSQL and ArpeNetezza never read OS certificate stores: give `ssl_root_cert`
  (or `SSL_CERT_FILE=/etc/pki/tls/certs/ca-bundle.crt` on RHEL). ArpeMSSQL (Linux bundles 0.7.3+),
  ArpeOracle (0.3.12+) and ArpeTeradata use the Windows ROOT store and the Linux distro CA bundles.
- **Teradata `sslmode=prefer` (default) can fall back to the clear legacy port** (1025, needs TTU): use
  `require` or `verify-full` to stay on TLS (cloud / trial environments: `require` on 443).
- **One open result per connection**: read a result to the end (or close the cursor) before the next
  statement on the same connection, or open another connection. Parallelism = several connections.
- **Ingest table names are quoted as given** (case-sensitive): on Oracle use upper-case names
  (`"SALES"`, not `"sales"`). `adbc_ingest` modes: `create`, `append`, `replace`, `create_append`.
- **Oracle fast ingest needs autocommit on**: `ingest_method=auto` uses the direct path load only with
  autocommit on, so from dbapi pass `autocommit=True` (otherwise array-bound `INSERT`, much slower).
- **Teradata ingest truncates long strings silently** (Teradata mode; created columns are
  `VARCHAR(4000)`): raise `adbc.arpeteradata.ingest.varchar_length`, or pre-create the table and `append`.
- **`query_timeout` does nothing** on ArpePGSQL and ArpeNetezza (use `socket_timeout`, SQL
  `statement_timeout`, or cancel); ArpeOracle has none (`socket_timeout`, `login_timeout`). It works on
  ArpeMSSQL (per wait, cancels, connection kept) and ArpeTeradata (socket read timeout, connection
  closed).
- **Licence**: the first source found is the only one used (no fall-through): option
  `arpeio.adbc.license` → option `arpeio.adbc.license_file` → env `ARPEIO_ADBC_LICENCE` → env
  `ARPEIO_ADBC_LICENCE_FILE` (British spelling) → `arpeio_adbc.lic` next to the driver library.

## Example

Read a SQL Server table into Arrow, then load it into PostgreSQL:

```python
import os
import adbc_driver_manager.dbapi as dbapi

mssql_uri = (f"sqlserver://etl:{os.environ['MSSQL_PASSWORD']}@sql01.corp.example:1433/"
             "?database=sales&encrypt=true")
with dbapi.connect(driver="arpemssql", db_kwargs={"uri": mssql_uri}) as src, src.cursor() as cur:
    cur.execute("SELECT * FROM dbo.orders WHERE order_date >= ?", parameters=["2026-01-01"])
    orders = cur.fetch_arrow_table()

pg_uri = (f"postgresql://loader:{os.environ['PG_PASSWORD']}@pg01.corp.example:5432/dwh"
          "?sslmode=verify-full&sslrootcert=/etc/ssl/corp-ca.pem")
with dbapi.connect(driver="arpepgsql", db_kwargs={"uri": pg_uri}) as dst:
    with dst.cursor() as cur:
        cur.adbc_ingest("orders", orders, mode="create_append", db_schema_name="staging")
    dst.commit()   # autocommit is off by default in dbapi
```

## When something fails

- `licence check failed [ARROW_LIC_…]` → install.md (licence).
- `driver not found` → the manifest is not on the driver manager's path: `install.sh --installed`, then
  `ADBC_DRIVER_PATH`.
- Connection, TLS, auth, type or ingest errors → the driver's reference file, section Troubleshooting.
- Check the licence without connecting: `conn.adbc_database.get_option("arpeio.adbc.license.status")`.
