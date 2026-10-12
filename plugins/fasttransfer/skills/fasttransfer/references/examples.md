# FastTransfer – Examples

These are adapted from the official docs (FastTransfer 0.17) with canonical long flags. Credentials such as
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

## PostgreSQL → PostgreSQL, binary COPY with Ctid (PowerShell)

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
  --parallelmethod "Ctid" `
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

## PostgreSQL → Oracle, orabulk with Ctid (PowerShell)

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "pgsql" `
  --sourceserver "localhost:5432" `
  --sourcedatabase "tpch" `
  --sourceuser "FastUser" `
  --sourcepassword "FastPassword" `
  --sourceschema "tpch_10" `
  --sourcetable "orders" `
  --targetconnectiontype "orabulk" `
  --targetserver "localhost:1521/ORCLPDB" `
  --targetdatabase "ORCLPDB" `
  --targetuser "TPCH_IN" `
  --targetpassword "TPCH_IN" `
  --targetschema "TPCH_IN" `
  --targettable "ORDERS" `
  --parallelmethod "Ctid" `
  --degree 12 `
  --loadmode "Truncate" `
  --batchsize 100000
```

## Oracle → SQL Server, Rowid (PowerShell)

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
  --parallelmethod "Rowid" `
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

## Read-only SQL Server source → SQL Server, Physloc, map by name (PowerShell)

```powershell
.\FastTransfer.exe `
  --sourceconnectiontype "mssql" `
  --sourceserver "localhost" `
  --sourcetrusted `
  --sourcedatabase "tpch_test" `
  --sourceschema "dbo" `
  --query "select * from dbo.orders_15M" `
  --targetconnectiontype "msbulk" `
  --targetserver "localhost,11433" `
  --targettrusted `
  --targetdatabase "tpch_test" `
  --targetschema "dbo" `
  --targettable "orders_ccsi" `
  --loadmode "Truncate" `
  --batchsize 120000 `
  --parallelmethod "Physloc" `
  --degree 10 `
  --mapmethod "Name"
```

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

## Docker: CSV files → SQL Server

See [environment.md](environment.md#docker).
