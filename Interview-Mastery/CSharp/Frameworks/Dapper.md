# Dapper

---

## Overview

- **Definition:** A lightweight, high-performance micro-ORM created by the Stack Overflow team. Extends `IDbConnection` with extension methods for querying, executing, and mapping results to objects.
- **Why It Exists:** Provides near-ADO.NET performance (~2-10µs overhead per query after cache warmup) with automatic object mapping, parameterized queries, and multi-result-set support, without the overhead of a full ORM (no change tracking, no LINQ provider, no migrations).
- **Key Concepts:** **`Query<T>`** (returns `IEnumerable<T>`), **`Execute`** (returns rows affected), **`QueryMultiple`** (multiple result sets), **`DynamicParameters`** (flexible parameter bags), **multi-mapping** (one-to-one/one-to-many via `splitOn`), **`buffered: false`** (streaming), and **type handlers** (custom column mapping).

---

## Core Concepts

- **Key Extension Methods:**

```csharp
int rows = connection.Execute("UPDATE Products SET Price = @Price WHERE Id = @Id", new { Price = 10m, Id = 1 });
IEnumerable<Product> products = connection.Query<Product>("SELECT * FROM Products WHERE Price > @Min", new { Min = 100m });
Product product = connection.QueryFirstOrDefault<Product>("SELECT * FROM Products WHERE Id = @Id", new { Id = 1 });
```

- **Deserialization Cache:** When `Query<T>` is first called for a given `<T, column-set>` combination, Dapper inspects `T`'s properties and the result set columns, emits an IL delegate for fast deserialization, and caches it in a `ConcurrentDictionary`. Subsequent calls are fast delegate invocations. First call is slower (IL emit).
- **Connection Management:** Dapper does NOT manage connections. It opens the connection if closed (on first command) and closes it after if it opened it. If you opened the connection, you must close it — especially important for transactions.
- **Parameterization:** All query parameters via anonymous types or `DynamicParameters` are fully parameterized, preventing SQL injection. Never concatenate user input into SQL strings.

```csharp
// Multi-mapping: one-to-one
var order = connection.Query<Order, Customer, Order>(
    "SELECT o.*, c.* FROM Orders o JOIN Customers c ON c.Id = o.CustomerId",
    (order, customer) => { order.Customer = customer; return order; },
    splitOn: "CustomerId").FirstOrDefault();
```

- **Multi-Mapping (`splitOn`):** Dapper splits result set columns at the `splitOn` column name. The first `T` gets columns before `splitOn`; the second `T` gets columns after. Default `splitOn` is `"Id"`. For one-to-many, use a dictionary lookup to group children.
- **GridReader (`QueryMultiple`):** Returns a `SqlMapper.GridReader` holding the `IDataReader`. `Read<T>()` reads the current result set and advances to the next. Dispose the reader when done to close the underlying reader.
- **Type Handlers:** Custom mapping via `SqlMapper.AddTypeHandler<T>(new JsonTypeHandler<Metadata>())` for JSON columns, `DateOnly`, etc.

```csharp
// Type handler for JSON columns
public class JsonTypeHandler<T> : SqlMapper.TypeHandler<T>
{
    public override T Parse(object value) => JsonSerializer.Deserialize<T>((string)value)!;
    public override void SetValue(IDbDataParameter parameter, T value) =>
        parameter.Value = JsonSerializer.Serialize(value);
}
```

---

## Common Mistakes

- **Not disposing connections** — `var conn = new SqlConnection(connStr); conn.Query(sql);` leaks the connection. Always use `await using`. This *looks correct* because the query returns results and the variable goes out of scope, but the underlying TCP connection is not returned to the ADO.NET connection pool until `Dispose` is called — without `using`, connections accumulate in a "leaked" state and the pool eventually starves, causing `SqlException: The connection pool has been exhausted` under load.
- **Forgetting to open connection for transactions** — Dapper opens/closes automatically for simple queries, but transactions require manual `connection.Open()` before `BeginTransaction`. This *looks correct* because Dapper normally handles connection lifetime automatically, but `BeginTransaction` requires an open connection — forgetting the manual `Open()` throws `InvalidOperationException` at runtime with a message that can be confusing when you expect automatic management.
- **Wrong `splitOn` in multi-mapping** — If a JOIN has multiple `Id` columns and `splitOn` defaults to `"Id"`, all split columns after the first get the second table's `Id` value — wrong matching. Specify `splitOn` explicitly at each join boundary. This *looks correct* because `splitOn: "Id"` is the default and the query returns data without error, but the second mapped object receives `Id` from the wrong table — the `Customer.Id` column value becomes the `Customer` object's `Name` property, producing silently incorrect domain objects.
- **Using `buffered: true` for huge result sets** — `connection.Query<HugeRow>("SELECT * FROM BillionRowTable")` loads all into memory. Use `buffered: false` and iterate. This *looks correct* because the query executes and returns results without error, but Dapper's default buffered mode materializes the entire result set into a `List<T>` in memory — a 500MB result set causes a large LOH allocation, triggers Gen 2 GC, and can crash with `OutOfMemoryException` on memory-constrained servers.
- **Mixing sync and async** — `connection.QueryAsync(sql).Result` can deadlock. Always await all the way. This *looks correct* because `.Result` accesses the task's result and appears to work in console applications, but in contexts with a `SynchronizationContext` (ASP.NET Classic, UI apps) the blocked thread is exactly the thread the async continuation needs to resume on, causing a deadlock that hangs the request indefinitely.
- **Multi-mapping with no matching rows** — `Query<Order, Customer, Order>(...).FirstOrDefault()` returns `null` if no rows. Check for null before accessing. This *looks correct* because `FirstOrDefault` is the standard way to handle empty results, but the returned `null` cascades — accessing `order.Customer` throws `NullReferenceException` at a point far from the query, making the root cause hard to identify.
- **Not checking return value** — `ExecuteAsync` returns `Task<int>` (rows affected). Ignoring it misses error signals or unexpected results. This *looks correct* because the query executed without throwing an exception, but a zero return value indicates no rows were affected — silently ignoring this means the application believes data was modified when the WHERE clause matched nothing, leading to silent data loss in production.

```csharp
// Always parameterize — never concatenate
connection.Execute($"DELETE FROM Products WHERE Id = {userInput}"); // Injection!
connection.Execute("DELETE FROM Products WHERE Id = @Id", new { Id = userInput }); // Safe
```

---

## Key Design Considerations

- **Dapper is not a replacement for EF Core** — Use Dapper for queries requiring full SQL control and maximum performance. Use EF Core for complex domain logic with change tracking.
- **CQRS pattern with Dapper** — EF Core on the command side (write with domain logic); Dapper on the query side (read models, reporting, complex joins).
- **Type handlers for custom column types** — Register type handlers for JSON columns, spatial types, `DateOnly`, etc. at application startup.
- **Use `AsList()` on `QueryAsync` results** — `(await conn.QueryAsync<Product>(sql)).AsList()` returns `List<T>` to avoid multiple enumeration of the cached result.

```csharp
var results = (await conn.QueryAsync<Product>(sql)).AsList();
```

- **Connection pooling is automatic** — `new SqlConnection(connStr)` uses ADO.NET's connection pool keyed by connection string. Creating many connections is cheap if pooling is enabled (default).
- **Dapper.Contrib for simple CRUD** — Provides `Get<T>`, `Insert<T>`, `Update<T>`, `Delete<T>` for simple cases without writing SQL.
- **Use `CommandDefinition` for fine-grained control** — Set `commandTimeout`, `commandType`, `cancellationToken`, and transaction on a `CommandDefinition` object.

---

## Real-World Scenarios

### Scenario 1: CQRS-Based Order Management System
**Context:** An e-commerce system uses CQRS — EF Core for commands (write), Dapper for queries (read). The read side must efficiently fetch complex order summaries with multiple joins.

```csharp
public class OrderQueries
{
    private readonly string _connectionString;

    public async Task<OrderSummary?> GetOrderSummaryAsync(int orderId)
    {
        const string sql = @"
            SELECT 
                o.Id, o.OrderNumber, o.OrderDate, o.Status, o.TotalAmount,
                c.Id, c.Name, c.Email,
                oi.Id, oi.ProductName, oi.Quantity, oi.UnitPrice
            FROM Orders o
            JOIN Customers c ON c.Id = o.CustomerId
            JOIN OrderItems oi ON oi.OrderId = o.Id
            WHERE o.Id = @OrderId
            ORDER BY oi.Id";

        await using var conn = new SqlConnection(_connectionString);

        var orderLookup = new Dictionary<int, OrderSummary>();

        await conn.QueryAsync<OrderSummary, CustomerInfo, OrderItem, OrderSummary>(
            sql,
            (order, customer, item) =>
            {
                if (!orderLookup.TryGetValue(order.Id, out var existing))
                {
                    order.Customer = customer;
                    order.Items = new List<OrderItem>();
                    orderLookup[order.Id] = existing = order;
                }
                existing.Items.Add(item);
                return existing;
            },
            new { OrderId = orderId },
            splitOn: "Id,Id");

        return orderLookup.Values.FirstOrDefault();
    }
}
```

### Scenario 2: High-Throughput Telemetry Ingestion
**Context:** A telemetry service ingests 10K device readings/second. Each reading must be inserted with minimal overhead. Uses Dapper with table-valued parameters for batch inserts.

```csharp
public class TelemetryIngestionService
{
    private readonly string _connectionString;

    public async Task IngestBatchAsync(IEnumerable<TelemetryReading> readings, CancellationToken ct)
    {
        await using var conn = new SqlConnection(_connectionString);
        await conn.OpenAsync(ct);
        using var tran = conn.BeginTransaction();

        try
        {
            // Create DataTable for TVP
            var dt = new DataTable();
            dt.Columns.Add("DeviceId", typeof(string));
            dt.Columns.Add("Timestamp", typeof(DateTime));
            dt.Columns.Add("MetricType", typeof(string));
            dt.Columns.Add("Value", typeof(double));
            dt.Columns.Add("Metadata", typeof(string));

            foreach (var r in readings)
                dt.Rows.Add(r.DeviceId, r.Timestamp, r.MetricType, r.Value, 
                    JsonSerializer.Serialize(r.Metadata));

            // Bulk insert via TVP — single round trip for all rows
            await conn.ExecuteAsync(
                "INSERT INTO Telemetry (DeviceId, Timestamp, MetricType, Value, Metadata) " +
                "SELECT DeviceId, Timestamp, MetricType, Value, Metadata FROM @Readings",
                new { Readings = dt.AsTableValuedParameter("dbo.TelemetryType") },
                transaction: tran,
                commandTimeout: 30);

            tran.Commit();
        }
        catch
        {
            tran.Rollback();
            throw;
        }
    }
}
```

### Scenario 3: Dynamic Reporting with Custom Type Handlers
**Context:** A reporting dashboard allows users to write custom SQL queries. Results include JSON columns and spatial data that need custom deserialization.

```csharp
// Register custom type handlers at startup
SqlMapper.AddTypeHandler(new JsonListHandler<AuditEntry>());
SqlMapper.AddTypeHandler(new SqlGeographyHandler());

public class ReportingService
{
    public async Task<IEnumerable<dynamic>> ExecuteReportAsync(string sql, object parameters)
    {
        await using var conn = new SqlConnection(_connectionString);
        
        // Use buffered: false for large result sets
        return await conn.QueryAsync(sql, parameters, commandTimeout: 120, buffered: false);
    }
}

// Custom type handler for JSON columns
public class JsonListHandler<T> : SqlMapper.TypeHandler<List<T>>
{
    public override List<T> Parse(object value) =>
        JsonSerializer.Deserialize<List<T>>((string)value) ?? new List<T>();

    public override void SetValue(IDbDataParameter parameter, List<T> value) =>
        parameter.Value = JsonSerializer.Serialize(value);
}

// Custom type handler for SQL Geography
public class SqlGeographyHandler : SqlMapper.TypeHandler<SqlGeography>
{
    public override SqlGeography Parse(object value) =>
        SqlGeography.Parse(new SqlString((string)value));

    public override void SetValue(IDbDataParameter parameter, SqlGeography value) =>
        parameter.Value = value.ToString();
}
```

---

## Scenario-Based Questions

1. **Q: You are building a high-traffic API endpoint that returns a list of products with their categories. The query joins 3 tables. You get the same performance with Dapper and EF Core. What should you consider?**
   A: For simple queries, EF Core's overhead (~50-200µs) may be negligible at low concurrency. As traffic scales, Dapper's lower per-query overhead (~2-10µs) and 4-10x lower memory allocation become significant. Consider CQRS: use EF Core for writes (change tracking is valuable) and Dapper for reads. Also consider: EF Core's query plan cache can cause memory pressure with many unique queries; Dapper's cache is purely column-based and bounded in practice.

2. **Q: You need to import 1M rows from a CSV into SQL Server daily. Using Dapper's `ExecuteAsync` with individual INSERTs takes 30 minutes. How do you optimize?**
   A: Use table-valued parameters (TVP): define a user-defined table type in SQL, create a `DataTable` in C#, populate it, and execute a single `conn.ExecuteAsync("INSERT ... SELECT * FROM @tvp", new { tvp = dt.AsTableValuedParameter("..."))`. This reduces 1M round trips to 1. For even faster bulk operations, use `SqlBulkCopy` directly (which is what Dapper does not wrap). TVP via Dapper typically achieves 100K+ rows/second vs ~1K rows/second with individual inserts.

3. **Q: You have a Dapper query that returns 500K rows for a reporting export. The server's memory spikes to 2GB. How do you fix it?**
   A: Use `buffered: false` in `QueryAsync` — this returns a streaming `IEnumerable<T>` that reads one row at a time from the `IDataReader`. Materialize only what you need. For CSV export, write each row to the response stream as it's read. Example: `await conn.QueryAsync<Product>(sql, buffered: false).SelectAwait(async p => await writer.WriteLineAsync(p.ToCsv()))`. This keeps memory at O(1) regardless of result set size.
   > **Interview follow-up:** With `buffered: false`, the `IDataReader` stays open while you stream — what happens to the database connection if the consumer is slow, and how does this affect connection pooling?

4. **Q: Your Dapper multi-mapping query isn't populating child objects correctly. The `splitOn` column appears in multiple tables. What's happening?**
   A: With `splitOn: "Id"`, Dapper splits on the FIRST `Id` column in the result set. If both `Orders` and `Customers` have an `Id` column, the customer's `Id` gets mapped as part of the `Order` object (wrong). Fix: use column aliases in SQL (e.g., `SELECT o.Id AS OrderId, o.*, c.Id AS CustomerId, c.* ...`) and set `splitOn: "CustomerId"`. This ensures each split point is unambiguous.

5. **Q: You are using Dapper in a multitenant SaaS application. Each tenant has a separate database. How do you manage connections efficiently?**
   A: Use a connection string provider that maps tenant IDs to connection strings (from a secure config or key vault). Create a new `SqlConnection` per request — ADO.NET connection pooling makes this cheap (pool hit is ~1µs). Never reuse connections across requests. Use `IHttpContextAccessor` to resolve the tenant ID and pass it to a factory method. For multi-tenant single database with row-level security, pass `TenantId` as a parameter to every query.

6. **Q: You are debugging a thread-pool starvation issue caused by Dapper queries. The pattern is `Task.Run(() => conn.Query(sql)).Result`. What's wrong and how do you fix it?**
   A: This blocks the thread pool thread with `.Result` AND uses `Task.Run` to push sync work to another thread pool thread. The blocked thread can't process other work items, causing starvation. Fix: use async Dapper methods (`QueryAsync`) and `await` them all the way up. If you must call sync Dapper from an async context (rare), offload to a dedicated thread, not the thread pool. For console apps, use `.GetAwaiter().GetResult()`. In ASP.NET Core, never block on async.
   > **Interview follow-up:** If you offload to a dedicated thread using `Task.Factory.StartNew` with `LongRunning`, how do you bound the number of dedicated threads to prevent resource exhaustion from concurrent offloaded calls?

7. **Q: You need to implement a paginated search with dynamic filters and sorting. Dapper queries are raw SQL. How do you build the SQL without SQL injection?**
   A: Build the WHERE clause dynamically using a `List<string>` conditions and `DynamicParameters`. Never concatenate user input into SQL. Example:
```csharp
var sql = "SELECT * FROM Products WHERE 1=1";
var parameters = new DynamicParameters();
if (!string.IsNullOrEmpty(name)) { sql += " AND Name LIKE @Name"; parameters.Add("Name", $"%{name}%"); }
if (minPrice.HasValue) { sql += " AND Price >= @MinPrice"; parameters.Add("MinPrice", minPrice.Value); }
sql += " ORDER BY Id OFFSET @Offset ROWS FETCH NEXT @PageSize ROWS ONLY";
parameters.Add("Offset", page * pageSize);
parameters.Add("PageSize", pageSize);
```
For sorting, use a whitelist of allowed column names — never concatenate user-provided column names directly.

8. **Q: You have a query that uses `QueryMultiple` to fetch an order and its items in one round trip. The `Read<Order>()` succeeds but `Read<OrderItem>()` returns empty. Why?**
   A: Most likely the SQL has `SELECT ... FROM Orders; SELECT ... FROM OrderItems;` without a semicolon or the second result set isn't producing rows. Debug by capturing the `GridReader` and checking `reader.IsConsumed`. Also ensure the SQL is valid when executed as a batch. Common mistake: forgetting the semicolon between SELECT statements in some providers (SQL Server requires it; PostgreSQL doesn't but it's good practice). Also verify that `Read<OrderItem>()` is called BEFORE disposing the `GridReader`.

9. **Q: You are migrating from ADO.NET to Dapper. A stored procedure returns multiple result sets with complex mappings. How do you structure the Dapper code?**
   A: Use `QueryMultiple` with `CommandType.StoredProcedure`. Read each result set sequentially:
```csharp
using var multi = await conn.QueryMultipleAsync("usp_GetFullOrder", new { OrderId = id },
    commandType: CommandType.StoredProcedure);
var order = await multi.ReadSingleAsync<Order>();
var items = await multi.ReadAsync<OrderItem>();
var payments = await multi.ReadAsync<Payment>();
var notes = await multi.ReadAsync<Note>();
```
Each `Read` call advances the reader to the next result set. Maintain the same order as the stored procedure's SELECT statements. Use `ReadSingleAsync` for exactly-one-row result sets.

10. **Q: Your Dapper queries are 3x slower than raw ADO.NET. You suspect the deserialization delegate generation is the bottleneck. What do you do?**
     A: Dapper generates IL delegates on first use per `<T, column-set>`. This is expensive (~5-20ms). Warm up the cache at startup by executing a representative query for each entity type. For dynamic queries with varying column sets (SELECT *), the cache grows and each unique column set triggers a new IL generation. Fix: use explicit column lists (`SELECT Id, Name, ...`) to keep column sets stable. Also consider: Dapper's overhead vs ADO.NET is only ~2-5µs after cache warmup — if you're seeing 3x slower, profile to check if the issue is elsewhere (connection management, parameter sniffing, indexing).
    > **Interview follow-up:** If you have a query with 50 columns and only 5 are used after mapping, is Dapper still paying the cost of generating a delegate for all 50 columns, and how would you measure that overhead?

---

## Interview Questions

1. **What is Dapper?**
   A: A lightweight micro-ORM by Stack Overflow that extends `IDbConnection` with extension methods for querying, executing, and mapping results to objects. Provides near-ADO.NET performance (~2-10µs overhead per query after cache warmup).

2. **What is the difference between Dapper and EF Core?**
   A: Dapper is a micro-ORM — raw SQL, no change tracking, no LINQ provider, no migrations. EF Core is a full ORM — LINQ-to-SQL, change tracking, migrations, and multiple database providers. Dapper is faster (~2-10µs vs ~50-500µs) but requires writing SQL manually.

3. **What does `buffered: false` do in Dapper?**
   A: By default (`buffered: true`), Dapper materializes all results into a `List<T>` before returning. With `buffered: false`, it returns a streaming `IEnumerable<T>` that reads rows lazily from the `IDataReader`, keeping memory at O(1). Use for large result sets to avoid memory pressure.

4. **How does Dapper's type deserialization cache work?**
   A: On first `Query<T>` for a given `<T, column-set>` combination, Dapper inspects `T`'s properties and the result set columns, emits an IL delegate for fast deserialization, and caches it in a `ConcurrentDictionary`. Subsequent calls are fast delegate invocations. First call is slower (IL emit).

5. **What is `splitOn` in multi-mapping?**
   A: The parameter that specifies the column name where Dapper splits the result set between mapped types. Default is `"Id"`. All columns before (but not including) the `splitOn` column map to the first type; the `splitOn` column and everything after map to the second type.

6. **What is `QueryMultiple` used for?**
   A: Executes a SQL batch with multiple SELECT statements and returns a `GridReader`. Each `Read<T>()` reads the current result set into objects and advances to the next result set via `NextResult()` on the `IDataReader`. Enables fetching related data in one round trip.

7. **What are Dapper type handlers?**
   A: Custom mappings for types that Dapper can't handle natively (JSON columns, spatial types, enums). Implement `SqlMapper.TypeHandler<T>` with `Parse` and `SetValue` methods, then register via `SqlMapper.AddTypeHandler<T>()` at startup.

8. **How does Dapper handle SQL injection?**
   A: Dapper fully parameterizes all queries. Values passed via anonymous types or `DynamicParameters` are sent as `SqlParameter` objects, preventing SQL injection. Never concatenate user input into SQL strings — always use Dapper's parameter syntax (`@ParamName`).

9. **What is `Dapper.Contrib`?**
   A: A NuGet package adding `Get<T>`, `Insert<T>`, `Update<T>`, `Delete<T>` methods for simple CRUD without writing SQL. Uses conventions (`[Key]`, `[Table]` attributes). Best for simple entity operations where writing SQL is repetitive.

10. **How does Dapper manage connections?**
    A: Dapper does NOT manage connections. It opens the connection if closed (on first command) and closes it after if it opened it. If you open the connection manually, you must close it. For transactions, you must open the connection before calling `BeginTransaction`.

---

## Developer Recommendations

- **Use `buffered: false` for large result sets** — Without it, Dapper materializes all rows into a `List<T>` before returning. For 500K rows, that's ~500MB of allocations. With `buffered: false`, rows are streamed from the `IDataReader`, keeping memory at O(1). Always use for reporting exports and batch processing. A reporting API once loaded a 2M-row export with `buffered: true` — the server ran out of memory, the process was killed by the OOM killer, and the entire service was down for 5 minutes until the auto-recovery restarted the instance.

- **Always use `await using` for Dapper connections** — Forgetting to dispose a `SqlConnection` leaks the connection back to the pool. The finalizer eventually returns it, but this delays resource reclamation and can cause pool exhaustion under load. Use `await using var conn = new SqlConnection(connStr)` for deterministic disposal.

- **Prefer Dapper for read queries and EF Core for writes (CQRS)** — Dapper gives you full SQL control with lower overhead — ideal for complex queries and reporting. EF Core's change tracking is invaluable on the write side. Mixing both in the same project is common and recommended: use EF Core for domain operations, Dapper for optimized read models.

- **Explicitly list columns in SELECT queries instead of `SELECT *`** — `SELECT *` causes different column sets when tables are altered, invalidating Dapper's deserialization cache and forcing re-compilation of IL delegates. Explicit column lists keep the cache stable and document exactly what data is needed. `SELECT *` also transfers unnecessary network data. A production incident occurred when a DBA added a `VARBINARY(MAX)` audit column to a table — `SELECT *` queries suddenly transferred megabytes per row and the deserialization cache for every query was invalidated, causing a cold-start lag spike that lasted until all unique query shapes had been re-cached.

- **Use `DynamicParameters` over anonymous types for complex queries** — Anonymous types work well for simple parameters. `DynamicParameters` supports output parameters, table-valued parameters, and DbType specification. For stored procedures with output parameters or TVP, always use `DynamicParameters`.

- **Use `CommandDefinition` for fine-grained control** — The `CommandDefinition` object lets you set `commandTimeout`, `commandType`, `cancellationToken`, and transaction in a single parameter. Pass it as the last parameter to any Dapper method. This avoids repetitive boilerplate and ensures consistent timeout/cancellation across all queries.

- **Warm up Dapper's type cache at application startup** — The first query for each `<T, column-set>` combination is slow due to IL emit. Execute representative queries during startup to warm the cache and avoid latency spikes on production traffic. This is especially important in serverless environments where cold starts matter.

---

## Dapper vs EF Core vs ADO.NET

| Feature | Dapper | EF Core | ADO.NET |
|---|---|---|---|
| ORM type | Micro-ORM | Full ORM | None |
| SQL abstraction | Raw SQL | LINQ + SQL | Raw SQL |
| Object mapping | Automatic | Automatic | Manual |
| Change tracking | No | Yes | No |
| Migration support | No | Yes | No |
| Overhead per query | ~2-10µs | ~50-500µs | ~1-5µs |
| Best for | Performance queries | Complex domain | Lowest level |

## Dapper Sync vs Async

| Aspect | Dapper sync | Dapper async |
|---|---|---|
| Overhead | ~2µs | ~5µs |
| Thread usage | Blocks | Non-blocking |
| Memory | Low | Low |
| Cancellation | No | Yes |
| Scalability | Good | Excellent |
