# MigratorXpress – Credentials file (`--auth`)

A JSON object. Each key is an identifier referenced on the command line by `--source_db_auth_id`,
`--target_db_auth_id` and `--migration_db_auth_id`.

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

## Fields (0.7.x)

| Field | Values / notes |
|---|---|
| `ds_type` | `oracle`, `postgres`, `mssql`, `netezza` |
| `auth_mode` | `classic` — the only mode that works in 0.7.x |
| `info.username` | required, non-empty |
| `info.password` | required, non-empty |
| `info.server` | host name or IP |
| `info.port` | **JSON integer** 1–65535 (usually 1521 Oracle, 5432 PostgreSQL, 1433 SQL Server, 5480 Netezza) |
| `info.database` | database name; for Oracle the **service name** (e.g. `ORCLPDB1`) |
| `info.lib_dir` | Oracle only, optional: Oracle Instant Client directory to use **thick mode** (Oracle 11.2 and earlier). Omit it for thin mode (12.1+) |

- All `info` fields except `lib_dir` are required and must be non-empty.
- There is **no** environment-variable substitution in 0.7.x: values are used as written. Generate the file
  from a secret store if needed, and restrict its permissions (`chmod 600`).
- ODBC DSNs, raw connection strings and Windows/Kerberos trusted authentication are **not** supported in 0.7.x
  (older docs mentioned them).

## Which roles each database can play

| `ds_type` | Source | Target | Tracking DB |
|---|---|---|---|
| `oracle` | yes | no | no |
| `postgres` | yes | experimental in 0.7.x | yes |
| `mssql` | yes | yes | yes |
| `netezza` | yes | no | no |

The tracking database can be the same server as the target (separate database recommended, e.g.
`migration_log`). MigratorXpress creates its tracking tables on first use.

## Permissions (typical)

- Source: read the catalog and the tables (Oracle: `SELECT_CATALOG_ROLE` is used for the Rowid parallel method).
- Target: create schemas and tables, bulk insert, alter tables (constraints).
- Tracking DB: create tables and read/write them.

## `--source_db_name` / `--target_db_name`

Both flags are required in 0.7.x. Pass the same value as the `database` of the corresponding auth entry: the
source connection always uses `info.database`, and the target name is used in qualified table names and passed
to FastTransfer.
