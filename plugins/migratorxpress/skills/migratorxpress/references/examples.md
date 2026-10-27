# MigratorXpress – Examples (0.8.0)

All examples assume a `credentials.json` with entries `oracle_prod` (Oracle, database `ORCLPDB1`),
`mssql_source` (SQL Server, `SourceDB`), `pg_source` (PostgreSQL, `sales`), `mssql_target` (SQL Server,
`target_db`), `pg_target` (PostgreSQL, `dwh`) and `ms_mig_log` (SQL Server tracking DB). See
[auth-file.md](auth-file.md). Database names come from the auth entries: no `--source_db_name` /
`--target_db_name`.

## Oracle schema → SQL Server: tables, data, validation (Linux)

```bash
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id oracle_prod \
  --source_schema_name SALES \
  --target_db_auth_id mssql_target \
  --target_schema_name sales \
  --migration_db_auth_id ms_mig_log \
  --n_jobs 4 \
  --fasttransfer_p 2 \
  --task_list translate create transfer diff
```

## Then the constraints (SQL Server target)

```bash
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id oracle_prod \
  --source_schema_name SALES \
  --target_db_auth_id mssql_target \
  --target_schema_name sales \
  --migration_db_auth_id ms_mig_log \
  --fk_mode untrusted \
  --task_list copy_pk copy_ak copy_fk
```

## Everything in one run, tagged, logs in a folder (PowerShell)

```powershell
.\MigratorXpress.exe --auth .\credentials.json `
  --source_db_auth_id oracle_prod `
  --source_schema_name SALES `
  --target_db_auth_id mssql_target `
  --target_schema_name sales `
  --migration_db_auth_id ms_mig_log `
  --n_jobs 4 `
  --fasttransfer_p 2 `
  --task_list all `
  --project sales_migration `
  --log_dir .\logs
```

## Several schemas, each keeping its name (pattern; no `--target_schema_name`)

```bash
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id oracle_prod \
  --source_schema_name "APP_%" \
  --target_db_auth_id mssql_target \
  --migration_db_auth_id ms_mig_log \
  --n_jobs 4 \
  --task_list translate create transfer diff
```

## PostgreSQL → SQL Server, selected tables, large-migration tuning

```bash
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id pg_source \
  --source_schema_name public \
  --target_db_auth_id mssql_target \
  --target_schema_name sales \
  --migration_db_auth_id ms_mig_log \
  --include_tables "fact_%,dim_%" \
  --exclude_tables "tmp_%" \
  --n_jobs 8 \
  --fasttransfer_p 4 \
  --ft_large_table_th 500000 \
  --task_list translate create transfer diff
```

## Recreate existing target tables, then reload (confirm with the user first)

```bash
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id mssql_source \
  --source_schema_name dbo \
  --target_db_auth_id mssql_target \
  --target_schema_name dbo_copy \
  --migration_db_auth_id ms_mig_log \
  --drop_tables_if_exists true \
  --task_list translate create transfer diff
```

## SQL Server → PostgreSQL, with constraints (no `copy_fk`, no `all`)

```bash
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id mssql_source \
  --source_schema_name dbo \
  --target_db_auth_id pg_target \
  --target_schema_name sales \
  --migration_db_auth_id ms_mig_log \
  --n_jobs 4 \
  --task_list translate create transfer diff copy_pk copy_ak
```

Target names are lower case in PostgreSQL (`dbo.Orders` → `sales.orders`).

## SQL Server source with integrated (Kerberos) authentication, Linux

Auth entry `mssql_trusted` with `"auth_mode": "trusted"` and `server` given as the FQDN; get a ticket first.

```bash
kinit migrator@CORP.EXAMPLE.COM
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id mssql_trusted \
  --source_schema_name dbo \
  --target_db_auth_id mssql_target \
  --target_schema_name dbo \
  --migration_db_auth_id ms_mig_log \
  --task_list translate create transfer diff
```

## Reload data only into existing tables, appending

```bash
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id oracle_prod \
  --source_schema_name SALES \
  --target_db_auth_id mssql_target \
  --target_schema_name sales \
  --migration_db_auth_id ms_mig_log \
  --load_mode append \
  --task_list transfer diff
```

## Resume a run after failures

```bash
./MigratorXpress --auth ./credentials.json \
  --source_db_auth_id oracle_prod \
  --target_db_auth_id mssql_target \
  --migration_db_auth_id ms_mig_log \
  --resume 20260203-cf5be0ac-159f-4dad-861e-8b2b6f3868d7
```

## Upgrade a 0.6.x tracking database

```bash
./MigratorXpress --auth ./credentials.json \
  --migration_db_auth_id ms_mig_log \
  --upgrade_migdb
```

## Check the result in the tracking DB

```sql
-- per-table job status of a run
SELECT * FROM sourcetargetloads WHERE run_id = '20260203-cf5be0ac-159f-4dad-861e-8b2b6f3868d7';

-- errors of a run
SELECT eventtimestamp, eventstatus, eventmessage
  FROM eventlog
 WHERE eventrunid = '20260203-cf5be0ac-159f-4dad-861e-8b2b6f3868d7' AND eventstatus = 'FAILED';
```
