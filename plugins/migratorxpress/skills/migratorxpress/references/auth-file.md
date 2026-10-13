# MigratorXpress – Credentials file (`--auth`)

A JSON object. Each key is an identifier referenced on the command line by `--source_db_auth_id`,
`--target_db_auth_id` and `--migration_db_auth_id`. Each entry has `ds_type`, `auth_mode` and `info`, and
nothing else.

```json
{
  "oracle_prod": {
    "ds_type": "oracle",
    "auth_mode": "classic",
    "info": {
      "username": "migrator",
      "password": "<password>",
      "server": "oracle-host.example.com",
      "port": 1521,
      "database": "ORCLPDB1"
    }
  },
  "mssql_target": {
    "ds_type": "mssql",
    "auth_mode": "classic",
    "info": {
      "username": "migrator",
      "password": "<password>",
      "server": "sql-host.example.com",
      "port": 1433,
      "database": "target_db"
    }
  },
  "ms_mig_log": {
    "ds_type": "mssql",
    "auth_mode": "classic",
    "info": {
      "username": "migrator",
      "password": "<password>",
      "server": "sql-host.example.com",
      "port": 1433,
      "database": "migration_log"
    }
  }
}
```

## Auth modes by database type

| `ds_type` | `classic` | `connection_string` | `trusted` | `odbc` |
|---|---|---|---|---|
| `oracle` | yes | – | – | – |
| `netezza` | yes | – | – | – |
| `postgres` | yes | yes | – | – |
| `mssql` | yes | yes | yes | source and tracking DB only (not target) |

Every entry is validated at startup, before any connection: an unknown `ds_type`, a mode not allowed for the
type, a missing or unexpected `info` key, or an unparsable connection string exits with code 10 and a message
naming the entry and the key (never the value). An auth id absent from the file also exits with code 10.

## `classic`

| Field | Values / notes |
|---|---|
| `info.username` | required |
| `info.password` | required |
| `info.server` | host name or IP |
| `info.port` | **JSON integer** 1–65535 (usually 1521 Oracle, 5432 PostgreSQL, 1433 SQL Server, 5480 Netezza) |
| `info.database` | database name; for Oracle the **service name** (e.g. `ORCLPDB1`) |
| `info.lib_dir` | Oracle only, optional: Oracle Instant Client directory to use **thick mode** (Oracle 11.2 and earlier). Omit it for thin mode (12.1+) |

No other key is accepted.

## `connection_string` (PostgreSQL, SQL Server)

```json
"pg_target": {
  "ds_type": "postgres",
  "auth_mode": "connection_string",
  "info": {
    "connection_string": "Host=pg.example.com;Port=5432;Database=dwh;Username=migrator;Password=<password>;SSL Mode=Require"
  }
},
"mssql_cs": {
  "ds_type": "mssql",
  "auth_mode": "connection_string",
  "info": {
    "connection_string": "Server=tcp:sql.example.com,1433;Database=target_db;User ID=migrator;Password=<password>;Encrypt=True;TrustServerCertificate=True",
    "driver_options": { "login_timeout": 30 }
  }
}
```

- The ADO.NET string FastTransfer understands: Npgsql keywords for PostgreSQL, SqlClient keywords for SQL Server.
  It is passed unchanged to FastTransfer and parsed for MigratorXpress's own metadata connections.
- The database name comes from the string (`Database=`).
- Mapped to the Python drivers: `SSL Mode`, `Timeout`, `Application Name`, `Root Certificate` (PostgreSQL);
  `Encrypt`, `Connect Timeout`, `Application Name` (SQL Server). `TrustServerCertificate` and unknown keys are
  logged as a warning and used by FastTransfer only.
- Optional `info.driver_options`: extra arguments merged over the parsed ones (psycopg2 / pymssql).
- SQL Server: `Integrated Security=true` (or `SSPI`) / `Trusted_Connection=yes` behaves like `trusted` below;
  any `User ID` / `Password` is then ignored with a warning.

## `trusted` (SQL Server integrated authentication)

```json
"mssql_trusted": {
  "ds_type": "mssql",
  "auth_mode": "trusted",
  "info": { "server": "sqlhost.corp.example.com", "port": 1433, "database": "target_db" }
}
```

`server` and `database` required; `port` and `driver_options` optional; no username or password. Windows: SSPI
with the account running MigratorXpress. Linux (Kerberos), for MigratorXpress and FastTransfer alike:

- a Kerberos ticket for the account running the migration (`kinit`, or `kinit -kt <keytab>`), valid for the
  whole run;
- `/etc/krb5.conf` with the realm and its KDC;
- `server` = the **fully qualified domain name** (not an IP or a CNAME alias);
- the SPN `MSSQLSvc/<fqdn>:<port>` registered for the SQL Server service account;
- the system GSSAPI library (`libgssapi_krb5`) for FastTransfer.

## `odbc` (SQL Server source or tracking DB)

```json
"mssql_src_odbc": {
  "ds_type": "mssql",
  "auth_mode": "odbc",
  "info": { "dsn": "MSSQL_DSN", "username": "migrator", "password": "<password>", "database": "SourceDB" }
}
```

- `dsn` required; `username`, `password`, `database`, `driver_options` (extra `pyodbc.connect` arguments)
  optional. Set `database` (otherwise it is read from the server).
- **Not for the target** (FastTransfer has no ODBC target; exit 10): use `classic`, `connection_string` or
  `trusted` there.
- Not bundled, host prerequisites: on Linux the unixODBC driver manager (`libodbc.so.2`); on every platform the
  Microsoft ODBC Driver 18 for SQL Server and the DSN declared (`/etc/odbc.ini`, `~/.odbc.ini`, or the Windows
  ODBC Data Source Administrator). If missing, the run stops at startup with exit code 10 saying what to install.

## Secrets

- No environment-variable substitution: values are used as written. Generate the file from a secret store if
  needed, and restrict its permissions (`chmod 600`).
- Passwords and connection strings are masked in the logs (FastTransfer command and output, event log).

## Which roles each database can play

| `ds_type` | Source | Target | Tracking DB |
|---|---|---|---|
| `oracle` | yes | no | no |
| `postgres` | yes | yes | yes |
| `mssql` | yes | yes | yes |
| `netezza` | yes (to SQL Server only) | no | no |

The tracking database can be the same server as the target (separate database recommended, e.g.
`migration_log`). MigratorXpress creates its tracking tables on first use.

## Permissions (typical)

- Source: read the catalog and the tables (Oracle: `SELECT_CATALOG_ROLE` is used for the Rowid parallel method).
- Target: create schemas and tables, bulk insert, alter tables (constraints).
- Tracking DB: create tables and read/write them.

## Database names

The source database is always the one in the source auth entry; `--source_db_name` is deprecated and ignored
(a different value is logged as a warning). The target database is the target entry's; `--target_db_name` is
optional and, when given, must match it (case-insensitive for SQL Server, exact for PostgreSQL), otherwise the
run stops with exit code 20 (`TARGET_DB_NAME_MISMATCH`).
