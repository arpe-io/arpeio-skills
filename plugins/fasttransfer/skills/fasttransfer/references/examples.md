# FastTransfer – Examples

These are adapted from the official docs and the README shipped with FastTransfer 1.0, with canonical
long flags. On 0.18 and older, replace `Physical` with the engine name given in each title. Credentials such as
`FastUser` / `FastPassword` are doc placeholders. In real commands use trusted auth or shell variables
(see SKILL.md, "Secrets").

## SQL Server → SQL Server, query, settings file (Linux)

```bash
./FastTransfer \
  --sourceconnectiontype "mssql" \
  --sourceserver "localhost" \
  --sourcedatabase "production" \
  --sourcetrusted \
  --targetconnectiontype "msbulk" \
  --targetserver "localhost" \
  --targetdatabase "warehouse" \
  --targettrusted \
  --targetschema "dbo" \
  --targettable "sales" \
  --query "select * from sales where order_date >= '2024-01-01'" \
  --runid "daily-sales-sync-2024-01-15" \
  --settingsfile "config/production-settings.json" \
  --loglevel information \
  --nobanner
```

## SQL Server → PostgreSQL, Random (PowerShell)

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "mssql" `
  --sourceserver "localhost" `
  --sourcedatabase "tpch_test" `
  --sourcetrusted `
  --targetconnectiontype "pgcopy" `
  --targetserver "localhost:5432" `
  --targetdatabase "tpch_import" `
  --targetuser "FastUser" `
  --targetpassword "FastPassword" `
  --targetschema "tpch_10" `
  --targettable "orders" `
  --query "select * from dbo.orders" `
  --parallelmethod "Random" `
  --distributekeycolumn "o_orderkey" `
  --degree 10 `
  --loadmode "Truncate"
```

## PostgreSQL → PostgreSQL, binary COPY with Physical (Ctid) (PowerShell)

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "pgcopy" `
  --sourceserver "localhost:5432" `
  --sourcedatabase "tpch" `
  --sourceuser "FastUser" `
  --sourcepassword "FastPassword" `
  --sourceschema "tpch_10" `
  --sourcetable "orders" `
  --targetconnectiontype "pgcopy" `
  --targetserver "localhost:15432" `
  --targetdatabase "tpch_import" `
  --targetuser "FastUser" `
  --targetpassword "FastPassword" `
  --targetschema "tpch_10" `
  --targettable "orders" `
  --parallelmethod "Physical" `
  --degree 10 `
  --loadmode "Truncate"
```

## PostgreSQL → SQL Server, DataDriven, license on a share (Linux)

```bash
./FastTransfer \
  --sourceconnectiontype "pgsql" \
  --sourceserver "localhost" \
  --sourcedatabase "production" \
  --sourceuser "FastUser" \
  --sourcepassword "FastPassword" \
  --targetconnectiontype "msbulk" \
  --targetserver "localhost" \
  --targetdatabase "warehouse" \
  --targettrusted \
  --targetschema "dbo" \
  --targettable "orders" \
  --query "SELECT * FROM orders WHERE order_date >= '2024-01-01'" \
  --parallelmethod DataDriven \
  --distributekeycolumn "EXTRACT(MONTH FROM order_date)" \
  --degree 4 \
  --license "//SharedUnc/licenses/FastTransfer/FastTransfer.lic"
```

## PostgreSQL → Oracle through ADBC, Physical (Ctid) (PowerShell, 1.0+)

The native pair (`pgsql` / `pgcopy` → `orabulk`) fails when the source table has a `date` column, as
`orders` does (troubleshooting.md, "Known limitations in 1.0"): this transfer uses `adbc_*` on both ends.
The Oracle names are in their stored case.

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "adbc_pgsql" `
  --sourceserver "localhost:5432" `
  --sourcedatabase "tpch" `
  --sourceuser "FastUser" `
  --sourcepassword "FastPassword" `
  --sourceschema "tpch_10" `
  --sourcetable "orders" `
  --targetconnectiontype "adbc_oracle" `
  --targetserver "localhost:1521/ORCLPDB" `
  --targetdatabase "ORCLPDB" `
  --targetuser "TPCH_IN" `
  --targetpassword "TPCH_IN" `
  --targetschema "TPCH_IN" `
  --targettable "ORDERS" `
  --parallelmethod "Physical" `
  --degree 4 `
  --loadmode "Truncate"
```

## Oracle → SQL Server, Physical (Rowid) (PowerShell)

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "oraodp" `
  --sourceserver "LOCALHOST:1521/ORCLPDB" `
  --sourcedatabase "ORCLPDB" `
  --sourceuser "TPCH_IN" `
  --sourcepassword "TPCH_IN" `
  --sourceschema "TPCH_IN" `
  --sourcetable "ORDERS_FLAT" `
  --targetconnectiontype "msbulk" `
  --targetserver "localhost" `
  --targetdatabase "tpch_test" `
  --targettrusted `
  --targetschema "dbo" `
  --targettable "orders" `
  --parallelmethod "Physical" `
  --loadmode "Truncate" `
  --degree 12
```

## SQL Server → MySQL (PowerShell)

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "mssql" `
  --sourceserver "localhost" `
  --sourcedatabase "tpch_test" `
  --sourcetrusted `
  --targetconnectiontype "mysqlbulk" `
  --targetserver "localhost" `
  --targetdatabase "tpch" `
  --targetuser "FastUser" `
  --targetpassword "FastPassword" `
  --targetschema "tpch" `
  --targettable "orders" `
  --query "select * from dbo.orders" `
  --parallelmethod "None" `
  --loadmode "Truncate"
```

## Read-only SQL Server source → SQL Server, Physical (Physloc), map by name (PowerShell)

`--sourceschema` and `--sourcetable` are required since 1.0; the `--query` filters that table.

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "mssql" `
  --sourceserver "localhost" `
  --sourcetrusted `
  --sourcedatabase "tpch_test" `
  --sourceschema "dbo" `
  --sourcetable "orders_15M" `
  --query "select * from dbo.orders_15M where o_orderdate >= '1995-01-01'" `
  --targetconnectiontype "msbulk" `
  --targetserver "localhost,11433" `
  --targettrusted `
  --targetdatabase "tpch_test" `
  --targetschema "dbo" `
  --targettable "orders_ccsi" `
  --loadmode "Truncate" `
  --batchsize 120000 `
  --parallelmethod "Physical" `
  --degree 10 `
  --mapmethod "Name"
```

## Partitioned SQL Server table → PostgreSQL, DBPartition (Linux, 0.18+)

One stream per partition of `lineitem_part` (partitioned by month), biggest first. No `--query` allowed.

```bash
./FastTransfer \
  --sourceconnectiontype "mssql" \
  --sourceserver "localhost" \
  --sourcedatabase "tpch" \
  --sourcetrusted \
  --sourceschema "dbo" \
  --sourcetable "lineitem_part" \
  --targetconnectiontype "pgcopy" \
  --targetserver "localhost:5432" \
  --targetdatabase "tpch" \
  --targetuser "postgres" \
  --targetpassword "mypassword" \
  --targetschema "public" \
  --targettable "lineitem" \
  --loadmode "Truncate" \
  --parallelmethod "DBPartition" \
  --degree 8
```

## SQL Server query → PostgreSQL, Timepartition by month (PowerShell, 0.18+)

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "mssql" `
  --sourceserver "localhost" `
  --sourcedatabase "tpch" `
  --sourcetrusted `
  --query "SELECT * FROM dbo.lineitem WHERE l_shipmode = 'AIR'" `
  --targetconnectiontype "pgcopy" `
  --targetserver "localhost:5432" `
  --targetdatabase "tpch" `
  --targetuser "postgres" `
  --targetpassword "mypassword" `
  --targetschema "public" `
  --targettable "lineitem_air" `
  --loadmode "Truncate" `
  --parallelmethod "Timepartition" `
  --distributekeycolumn "(l_shipdate,year,month)" `
  --degree 8
```

If the source needs compact date literals, set `$env:FASTTRANSFER_DATEFORMAT = "yyyyMMdd"` first
(bash: `export FASTTRANSFER_DATEFORMAT=yyyyMMdd`).

## Many CSV files → SQL Server, one file per thread (PowerShell)

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "duckdbstream" `
  --sourceserver ":memory:" `
  --query "SELECT * exclude filename FROM read_csv('D:\temp\FastBCP_lineitem_Extract\lineitem*.CSV', filename=true)" `
  --targetconnectiontype "msbulk" `
  --targetserver "localhost" `
  --targettrusted `
  --targetdatabase "tpch_test" `
  --targetschema "dbo" `
  --targettable "lineitem" `
  --parallelmethod "DataDriven" `
  --distributekeycolumn "filename" `
  --datadrivenquery "select file from glob('D:\temp\FastBCP_lineitem_Extract\*.CSV')" `
  --mapmethod "Name" `
  --loadmode "Truncate" `
  --degree 16
```

## Hive-partitioned Parquet → PostgreSQL (PowerShell)

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "duckdbstream" `
  --sourceserver ":memory:" `
  --query "SELECT * FROM read_parquet('D:\DATA\sales\year=*/month=*\*.parquet', hive_partitioning=true)" `
  --targetconnectiontype "pgcopy" `
  --targetserver "localhost:5432" `
  --targetuser "postgres" `
  --targetpassword "YourPassword" `
  --targetdatabase "sales" `
  --targetschema "public" `
  --targettable "sales_data" `
  --parallelmethod "DataDriven" `
  --distributekeycolumn "year || '-' || month" `
  --datadrivenquery "select distinct year || '-' || month as partition_key from read_parquet('D:\DATA\sales\year=*/month=*\*.parquet', hive_partitioning=true)" `
  --mapmethod "Name" `
  --loadmode "Truncate" `
  --degree 12
```

## ODBC DSN source → SQL Server (PowerShell)

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "odbc" `
  --sourcedsn "DSNMySQLTPCH" `
  --sourcedatabase "tpch" `
  --sourceuser "FastUser" `
  --sourcepassword "FastPassword" `
  --query "select * from tpch.orders_part LIMIT 10" `
  --targetconnectiontype "msbulk" `
  --targetserver "localhost" `
  --targetdatabase "tpch_test" `
  --targettrusted `
  --targetschema "dbo" `
  --targettable "orders_part" `
  --parallelmethod "None" `
  --loadmode "Append"
```

## Db2 → SQL Server through ADBC, usual flags (PowerShell, 1.0+)

`adbc_*` on both ends. The target names are written in their stored case; the `--license` file holds the
FastTransfer block and the ADBC driver block.

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "adbc_db2" `
  --sourceserver "dbhost:50000" `
  --sourcedatabase "TPCH" `
  --sourceuser "db2inst1" `
  --sourcepassword $env:DB2_PASSWORD `
  --sourceschema "TPCH_SF10" `
  --sourcetable "ORDERS" `
  --targetconnectiontype "adbc_mssql" `
  --targetserver "localhost\SS2025" `
  --targettrusted `
  --targetdatabase "tpch_test" `
  --targetschema "dbo" `
  --targettable "orders" `
  --parallelmethod "None" `
  --license "FastTransfer.lic"
```

## PostgreSQL → SQL Server through ADBC, RangeId (Linux, 1.0+)

```bash
./FastTransfer \
  --sourceconnectiontype "adbc_pgsql" \
  --sourceserver "pghost:5432" \
  --sourcedatabase "tpch" \
  --sourceuser "$FT_SOURCE_USER" \
  --sourcepassword "$FT_SOURCE_PASSWORD" \
  --sourceschema "public" \
  --sourcetable "lineitem" \
  --targetconnectiontype "adbc_mssql" \
  --targetserver "sqlhost:1433" \
  --targetdatabase "tpch" \
  --targetuser "$FT_TARGET_USER" \
  --targetpassword "$FT_TARGET_PASSWORD" \
  --targetschema "dbo" \
  --targettable "lineitem" \
  --loadmode "Truncate" \
  --parallelmethod "RangeId" \
  --distributekeycolumn "l_orderkey" \
  --degree 4 \
  --license "/opt/fasttransfer/FastTransfer.lic"
```

## Oracle → PostgreSQL through ADBC, connection strings (Linux, 1.0+)

The connection strings use the drivers' `uri=` form (not ADO.NET keywords); a password holding `@`, `:`
or `/` must be percent-encoded. `ORA_URI` and `PG_URI` are shell variables set beforehand, e.g.
`uri=oracle://user:password@orahost:1521/orclpdb1` and `uri=postgresql://user:password@pghost:5432/tpch`.

```bash
./FastTransfer \
  --sourceconnectiontype "adbc_oracle" \
  --sourceconnectstring "$ORA_URI" \
  --sourceschema "TPCH" \
  --sourcetable "ORDERS" \
  --targetconnectiontype "adbc_pgsql" \
  --targetconnectstring "$PG_URI" \
  --targetschema "public" \
  --targettable "orders" \
  --loadmode "Truncate" \
  --parallelmethod "Physical" \
  --degree 4
```

## Docker: CSV files → SQL Server

See [environment.md](environment.md#docker).
