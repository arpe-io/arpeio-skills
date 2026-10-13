# arpe.io ADBC drivers – Using them from a client

Any ADBC driver manager loads the drivers by name once installed (install.md). Options are strings,
set on the **database** before init (Python `db_kwargs`) unless a driver file says otherwise.

## Loading by name in each language

Swap `arpemssql` and the URI for `arpepgsql`, `arpeoracle` or `arpenz` (driver files).

```python
# Python: pip install adbc-driver-manager pyarrow
import adbc_driver_manager.dbapi as dbapi
with dbapi.connect(driver="arpemssql",
                   db_kwargs={"uri": "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true"}) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM dbo.orders")
        table = cur.fetch_arrow_table()          # pyarrow.Table
```

```cpp
// C / C++ (error checks elided)
AdbcError e = {}; AdbcDatabase db = {}; AdbcConnection conn = {}; AdbcStatement st = {};
AdbcDatabaseNew(&db, &e);
AdbcDatabaseSetOption(&db, "driver", "arpemssql", &e);
AdbcDatabaseSetOption(&db, "uri", "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true", &e);
AdbcDatabaseInit(&db, &e);
AdbcConnectionNew(&conn, &e); AdbcConnectionInit(&conn, &db, &e);
AdbcStatementNew(&conn, &st, &e);
AdbcStatementSetSqlQuery(&st, "SELECT * FROM dbo.orders", &e);
ArrowArrayStream stream = {}; int64_t n = -1;
AdbcStatementExecuteQuery(&st, &stream, &n, &e);
```

```csharp
// C#: Apache.Arrow.Adbc + driver manager
using var driver = AdbcDriverManager.FindLoadDriver("arpemssql", loadOptions: AdbcLoadFlags.Default);
using var database = driver.Open(new Dictionary<string, string> {
    ["uri"] = "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true" });
using var connection = database.Connect(null);
using var statement = connection.CreateStatement();
statement.SqlQuery = "SELECT * FROM dbo.orders";
using var stream = statement.ExecuteQuery().Stream!;
```

```go
// Go: github.com/apache/arrow-adbc/go/adbc/drivermgr
var drv drivermgr.Driver
db, _ := drv.NewDatabase(map[string]string{
	"driver":          "arpemssql",
	adbc.OptionKeyURI: "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true",
})
conn, _ := db.Open(ctx)
stmt, _ := conn.NewStatement()
_ = stmt.SetSqlQuery("SELECT * FROM dbo.orders")
reader, _, _ := stmt.ExecuteQuery(ctx)
defer reader.Release()
```

```java
// Java: JNI driver manager
Map<String, Object> params = new HashMap<>();
JniDriver.PARAM_DRIVER.set(params, "arpemssql");
AdbcDriver.PARAM_URI.set(params, "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true");
try (BufferAllocator a = new RootAllocator();
     AdbcDatabase db = new JniDriver(a).open(params);
     AdbcConnection conn = db.connect();
     AdbcStatement st = conn.createStatement()) {
  st.setSqlQuery("SELECT * FROM dbo.orders");
  try (AdbcStatement.QueryResult r = st.executeQuery()) { ArrowReader reader = r.getReader(); }
}
```

```r
# R: adbcdrivermanager
library(adbcdrivermanager)
db  <- adbc_database_init(adbc_driver("arpemssql"),
                          uri = "sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true")
con <- adbc_connection_init(db)
table <- read_adbc(con, "SELECT * FROM dbo.orders") |> arrow::as_arrow_table()
```

```rust
// Rust: adbc_core + adbc_driver_manager
let mut driver = ManagedDriver::load_from_name("arpemssql", None, AdbcVersion::default(), LOAD_FLAG_DEFAULT, None)?;
let mut db = driver.new_database_with_opts([(OptionDatabase::Uri,
    OptionValue::String("sqlserver://sa:<pw>@host:1433/?database=db&encrypt=true".into()))])?;
let mut conn = db.new_connection()?;
let mut stmt = conn.new_statement()?;
stmt.set_sql_query("SELECT * FROM dbo.orders")?;
let reader = stmt.execute()?;
```

Other options: add them next to `uri` (C: `AdbcDatabaseSetOption`; Python: more `db_kwargs` keys).

## Python patterns

```python
import os
import adbc_driver_manager.dbapi as dbapi

db_kwargs = {
    "adbc.arpepgsql.server": "pg01.corp.example",
    "adbc.arpepgsql.database": "dwh",
    "adbc.arpepgsql.username": "reader",
    "adbc.arpepgsql.password": os.environ["PG_PASSWORD"],
    "adbc.arpepgsql.sslmode": "verify-full",
    "arpeio.adbc.license_file": "/etc/arpeio/arpeio_adbc.lic",   # optional
}
conn = dbapi.connect(driver="arpepgsql", db_kwargs=db_kwargs)
```

| Task | Code |
|---|---|
| Query → Arrow table | `cur.execute(sql); t = cur.fetch_arrow_table()` |
| Query → pandas | `cur.fetch_df()` (or `fetch_arrow_table().to_pandas()`) |
| Stream big results | `reader = cur.fetch_record_batch()` then `for batch in reader: …` (constant memory) |
| Parameters | `cur.execute(sql, parameters=[…])` with the driver's marker: `?` (SQL Server, Netezza), `$1` (PostgreSQL, also Netezza), `:1` / `:name` (Oracle) |
| Many parameter rows | `cur.executemany(sql, [(…), (…)])` |
| Bulk load Arrow / pandas | `cur.adbc_ingest("table", data, mode="create" \| "append" \| "replace" \| "create_append", db_schema_name="schema", temporary=False)` – `data` = pyarrow Table / RecordBatch / RecordBatchReader |
| Rows per result batch | `cur.adbc_statement.set_options(**{"adbc.<prefix>.batch_size": "100000"})` |
| Transactions | `dbapi.connect(...)` starts with **autocommit off**: `conn.commit()` / `conn.rollback()`; or `dbapi.connect(..., autocommit=True)` |
| Cancel a running query | `cur.adbc_cancel()` (from another thread) |
| Driver / server info | `conn.adbc_get_info()` |
| Tables and columns | `conn.adbc_get_objects(...)`, `conn.adbc_get_table_schema("t", db_schema_filter="s")` |
| Licence status | `conn.adbc_database.get_option("arpeio.adbc.license.status")` |

Write a large result to Parquet without loading it all:

```python
import pyarrow.parquet as pq
with conn.cursor() as cur:
    cur.execute("SELECT * FROM big_table")
    reader = cur.fetch_record_batch()
    with pq.ParquetWriter("big_table.parquet", reader.schema) as w:
        for batch in reader:
            w.write_batch(batch)
```

## Rules common to the four drivers

- One ADBC connection = one database session; not for concurrent use from several threads. Run work in
  parallel with one connection per worker.
- One open result per connection: read it to the end or close the cursor before the next statement
  (otherwise `INVALID_STATE` "… still open …").
- Ingest modes: `create` fails if the table exists; `append` needs it; `replace` drops and recreates it;
  `create_append` creates it if missing. Table and column names are quoted as given (case-sensitive).
  The Arrow column names must match the target columns for `append`.
- After `adbc_ingest` with autocommit off, `conn.commit()` or the rows are lost at close.
- Every result field carries the server type name in its metadata (`ARPEMSSQL:type`, `ARPEPGSQL:type`,
  `ARPEORACLE:type`), e.g. `decimal(10,2)`, `VARCHAR2(100)`.
- ADBC errors carry the server error (SQLSTATE / error number) in their details (Python: exception
  `.details`, `.sqlstate`, `.vendor_code`).
- Partitioned execution and Substrait are not supported.
