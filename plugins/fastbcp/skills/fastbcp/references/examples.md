# FastBCP – Examples

These are taken from the official docs (FastBCP 1.5). Credentials such as `FastUser` / `FastPassword`
are doc placeholders. In real commands use `--trusted` or shell variables (see SKILL.md, "Secrets").

## SQL Server → CSV, integrated auth, filtered query (Linux)

```bash
./FastBCP \
  --connectiontype mssql \
  --server "localhost" \
  --database "AdventureWorks" \
  --trusted \
  --query "SELECT OrderID, OrderDate, CustomerName, TotalAmount, IsShipped FROM Sales.Orders WHERE OrderDate >= '2024-01-01'" \
  --directory "/exports/sales" \
  --fileoutput "orders_{startdate}.csv" \
  --delimiter "," \
  --quotes \
  --dateformat "yyyy-MM-dd HH:mm:ss" \
  --decimalseparator "." \
  --boolformat "1/0" \
  --encoding "UTF-8"
```

## SQL Server → CSV, Ntile, custom logging settings (PowerShell)

```powershell
.\FastBCP.exe `
  --connectiontype mssql `
  --server localhost `
  --database "tpch10_collation_bin2" `
  --trusted `
  --query "SELECT * FROM dbo.orders" `
  --directory "D:\temp" `
  --fileoutput "mssql_orders.csv" `
  --parallelmethod Ntile `
  --distributekeycolumn "o_orderkey" `
  --paralleldegree 7 `
  --settingsfile "D:\logs\config\FastBCP_settings.json"
```

## SQL Server → CSV, Timepartition by year/month (Linux)

```bash
./FastBCP \
  --connectiontype "mssql" \
  --server "localhost" \
  --database "tpch_copy" \
  --user "FastUser" \
  --password "FastPassword" \
  --sourceschema "tpch_10" \
  --sourcetable "orders_date_sorted" \
  --directory "/data/export/TestPartition" \
  --fileoutput "orders.csv" \
  --decimalseparator "," \
  --delimiter "|" \
  --dateformat "yyyy-MM-dd HH:mm:ss" \
  --encoding "UTF-8" \
  --parallelmethod "Timepartition" \
  --distributekeycolumn "(o_orderdate,year,month)" \
  --paralleldegree 16 \
  --merge false \
  --runid "mssql_to_csv_parallel8_timepartition"
```

## PostgreSQL → Parquet, DataDriven on an expression, license file (Linux)

```bash
./FastBCP \
  --connectiontype pgsql \
  --server "localhost:15432" \
  --database "tpch10" \
  --user "FastUser" \
  --password "FastPassword" \
  --query "SELECT * FROM dbo.orders" \
  --directory "/temp" \
  --fileoutput "pgsql_orders.parquet" \
  --parallelmethod DataDriven \
  --distributekeycolumn "YEAR(o_orderdate)" \
  --paralleldegree 7 \
  --merge false \
  --runid "pgsql_to_parquet_parallel_datadriven" \
  --license "//SharedUnc/licenses/fastbcp/fastbcp.lic"
```

## PostgreSQL partitioned table → Parquet, one folder per partition (Linux)

```bash
export FASTBCP_DBPARTITION_TARGET_MODE=HIVE

./FastBCP \
  --connectiontype "pgsql" \
  --server "localhost:5432" \
  --database "tpch" \
  --user "FastUser" \
  --password "FastPassword" \
  --sourceschema "tpch_10" \
  --sourcetable "orders_partitioned" \
  --parallelmethod "DBPartition" \
  --paralleldegree 8 \
  --directory "/data/export/orders" \
  --fileoutput "orders.parquet" \
  --merge false \
  --runid "pgsql_to_parquet_dbpartition"
```

## PostgreSQL → OneLake, Physical (Linux)

```bash
export AZURE_TENANT_ID="YOUR_TENANT_ID"
export AZURE_CLIENT_ID="APPLICATION_ENTRA_ID"
export AZURE_CLIENT_SECRET="Your_AZClient_Secret"

./FastBCP \
  --connectiontype "pgsql" \
  --server "localhost:5432" \
  --database "ecommerce" \
  --user "postgres" \
  --password "postgres" \
  --sourceschema "sales" \
  --sourcetable "orders" \
  --query "SELECT * FROM orders WHERE order_date between '2024-01-01' and '2024-12-31'" \
  --directory "onelake://myworkspace/mylakehouse/{sourcedatabase}/{sourceschema}/{sourcetable}/year=2024/" \
  --fileoutput "orders_2024.parquet" \
  --parallelmethod "Physical" \
  --paralleldegree 8
```

## MySQL → CSV, DataDriven on a status column (PowerShell)

```powershell
.\FastBCP.exe `
  --connectiontype "mysql" `
  --server "localhost" `
  --port "3306" `
  --database "test" `
  --user "root" `
  --password "password" `
  --sourceschema "test" `
  --sourcetable "orders" `
  --directory "C:\temp" `
  --fileoutput "mysql_orders_dd.csv" `
  --decimalseparator "." `
  --delimiter "|" `
  --dateformat "yyyy-MM-dd HH:mm:ss" `
  --encoding "UTF-8" `
  --parallelmethod "DataDriven" `
  --distributekeycolumn "status" `
  --paralleldegree 10 `
  --merge false `
  --runid "mysql_to_csv_parallel10_datadriven"
```

## Oracle → Excel, Physical, debug logging (PowerShell)

```powershell
.\FastBCP.exe `
  --connectiontype "oraodp" `
  --server "localhost:1521/OraService" `
  --database "tpch" `
  --user "FastUser" `
  --password "FastPassword" `
  --sourceschema "TPCH" `
  --sourcetable "ORDERS" `
  --directory "D:\temp" `
  --fileoutput "orders.xlsx" `
  --encoding "UTF-8" `
  --parallelmethod Physical `
  --paralleldegree 7 `
  --merge false `
  --loglevel "Debug" `
  --runid "oracle_to_excel_parallel_physical" `
  --settingsfile "C:\config\oracle-export.json" `
  --nobanner
```

## Oracle → Google Cloud Storage, Physical (Linux)

```bash
./FastBCP \
  --connectiontype "oraodp" \
  --server "localhost:1521/ORCLPDB1" \
  --user "FastUser" \
  --password "FastPassword" \
  --sourceschema "SALES" \
  --sourcetable "ORDERS" \
  --directory "gs://warehouse-bucket/oracle-export/orders" \
  --fileoutput "orders.parquet" \
  --parallelmethod "Physical" \
  --paralleldegree 8 \
  --merge false
```

(The docs' version of this example sets `--merge "True"`, but merge is documented as not available
for cloud targets, so `false` is used here.)

## Netezza via ADBC → Parquet, Physical (PowerShell)

```powershell
.\FastBCP.exe `
  --connectiontype adbc_netezza `
  --sourceconnectstring "Server=nps-host;Port=5480;Database=SALES;User ID=admin;Password=FastPassword" `
  --sourceschema "ADMIN" `
  --sourcetable "ORDERS" `
  --directory "D:\export\orders" `
  --fileoutput "orders.parquet" `
  --parallelmethod Physical `
  --paralleldegree 8
```

`Physical` resolves to `Ctid` on PostgreSQL, `Rowid` on Oracle and `NZDataSlice` on Netezza. On
FastBCP 1.3 or older, write those names instead.

## SQL Server backup file → Parquet, no restore, Physical (PowerShell, 1.5+)

```powershell
.\FastBCP.exe `
  --connectiontype "backup_mssql" `
  --server "D:\Backups\MSSQL" `
  --database "tpch_full.bak" `
  --sourceschema "dbo" `
  --sourcetable "orders" `
  --directory "D:\export\{sourcetable}" `
  --fileoutput "{sourcetable}.parquet" `
  --parallelmethod "Physical" `
  --paralleldegree 8
```

No credentials: `--server` is the backup directory and `--database` the backup file. `Physical`
resolves to `Pages`.

## Striped SQL Server backup → Parquet (Linux, 1.5+)

```bash
./FastBCP \
  --connectiontype "backup_mssql" \
  --server "/var/opt/mssql/backup" \
  --database "tpch_*.bak" \
  --sourceschema "dbo" \
  --sourcetable "lineitem" \
  --directory "/export/lineitem" \
  --fileoutput "lineitem.parquet" \
  --parallelmethod "Physical" \
  --paralleldegree 8
```

The pattern must match every stripe of the backup. A list works too:
`--database "tpch_1.bak;tpch_2.bak;tpch_3.bak"`. A column subset uses a `--query` with no filter, e.g.
`--query "SELECT OrderID, CustomerID, OrderDate FROM Sales.Orders"`.

## Teradata via ADBC → Parquet, Ntile (Linux)

```bash
export FASTBCP_TTU_PATH=/opt/teradata/client/20.00   # only needed for the legacy-port logon

./FastBCP \
  --connectiontype adbc_teradata \
  --server "td-host" \
  --database "TPCH" \
  --user "FastUser" \
  --password "FastPassword" \
  --sourceschema "TPCH" \
  --sourcetable "ORDERS" \
  --directory "/data/export/orders" \
  --fileoutput "orders.parquet" \
  --parallelmethod Ntile \
  --distributekeycolumn "o_orderkey" \
  --paralleldegree 8
```

## DB2 via ODBC → CSV, Ntile (PowerShell)

```powershell
.\FastBCP.exe `
  --connectiontype "odbc" `
  --sourceconnectstring "Driver={IBM DB2 ODBC DRIVER};Database=SAMPLE;Hostname=db2server.company.corp;Port=25010;Protocol=TCPIP;Uid=db2user;Pwd=password;CurrentSchema=DB2ADMIN" `
  --query "SELECT * FROM DB2ADMIN.ORDERS" `
  --directory "C:\temp" `
  --fileoutput "db2_orders.csv" `
  --decimalseparator "." `
  --delimiter "|" `
  --dateformat "yyyy-MM-dd HH:mm:ss" `
  --encoding "UTF-8" `
  --parallelmethod "Ntile" `
  --distributekeycolumn "ORDER_ID" `
  --paralleldegree -2 `
  --merge false
```

## Docker: SQL Server → S3

See [environment.md](environment.md#docker).
