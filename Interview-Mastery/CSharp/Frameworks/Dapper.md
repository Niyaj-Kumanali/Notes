# Dapper

## 1. Executive Summary

Dapper is a lightweight, high-performance micro-ORM created by the Stack Overflow team. It extends IDbConnection with extension methods for querying, executing, and mapping results to objects. Dapper is not a full ORM — it provides no change tracking, no LINQ provider, no migrations, and no unit of work. Instead, it focuses on fast execution of raw SQL with minimal overhead, making it ideal for performance-critical paths and scenarios where full SQL control is required.

## 2. Core Theory

### Key Extension Methods

```csharp
// Execute a command (returns rows affected)
int rows = connection.Execute("UPDATE Products SET Price = @Price WHERE Id = @Id",
    new { Price = 10.0m, Id = 1 });

// Query and map to typed objects
IEnumerable<Product> products = connection.Query<Product>(
    "SELECT * FROM Products WHERE Price > @MinPrice",
    new { MinPrice = 100.0m });

// Query a single row
Product product = connection.QueryFirstOrDefault<Product>(
    "SELECT * FROM Products WHERE Id = @Id",
    new { Id = 1 });

// Query multiple result sets
using var multi = connection.QueryMultiple(
    "SELECT * FROM Products; SELECT * FROM Categories");
var products = multi.Read<Product>();
var categories = multi.Read<Category>();

// Query with buffered vs unbuffered
var stream = connection.Query<Product>(sql, buffered: false); // Streaming
```

### How Dapper Maps Results

```csharp
// 1. Dapper reads the IDataReader from the command
// 2. For each row, it creates an instance of T (activator)
// 3. For each column in the result set:
//    a. Look up a property/field on T with matching name (case-insensitive)
//    b. If found, set the value (type coercion via Convert.ChangeType if needed)
// 4. Mapping results are cached in ConcurrentDictionary (by type + column set)

// Dynamic mapping:
dynamic result = connection.QueryFirst("SELECT Id, Name FROM Products WHERE Id = @Id", new { Id = 1 });
int id = result.Id;
string name = result.Name;
```

## 3. Under-the-Hood Deep Dive

### Method Cache Internals

```csharp
// Dapper maintains several caches:
// - SqlMapper.LazyCache: Dictionary<Type, Func<IDataReader, object>> for type deserialization
// - TypeDeserializerCache: caches per-type serializers
// - GetPropertyInfo: cached per TypeInfo

// Deserialization function compilation:
// 1. When Query<T> is first called, Dapper inspects T's properties and the result set's columns
// 2. For each column, find matching member (case-insensitive, exact then fallback)
// 3. Emit IL or use Expression trees to create a fast deserialization delegate
// 4. Cache the delegate for subsequent calls

// This means first query for a given <T, columns> combination is slower
// Subsequent queries are very fast (delegate invocation).
```

### Connection Management

```csharp
// Dapper does NOT manage connections. It uses the IDbConnection you pass.
// Best practice:
// - ASP.NET Core: open connection per request (scoped)
// - Connection pooling handled by ADO.NET (SqlConnectionPool)
// - Always use 'using' or dispose connections

// Dapper will:
// - Open the connection if it's closed (on first command)
// - Close it after the command if it opened it
// - NOT close it if you opened it yourself

// This is particularly important for transactions:
connection.Open();
using var tx = connection.BeginTransaction();
connection.Execute(sql, transaction: tx);
tx.Commit();
// connection.Close() is YOUR responsibility
```

### GridReader (QueryMultiple)

```csharp
// QueryMultiple returns a SqlMapper.GridReader
// Internally it holds the IDataReader and advances through result sets
// Read<T>() reads the current result set and advances to the next

// IMPORTANT: Dispose GridReader when done to close the underlying reader
// Unconsumed result sets are dropped on dispose
```

## 4. Production Code Examples

```csharp
// Full CRUD with Dapper
public class ProductRepository
{
    private readonly string _connectionString;

    public ProductRepository(string connectionString) =>
        _connectionString = connectionString;

    public async Task<Product?> GetByIdAsync(int id)
    {
        await using var conn = new SqlConnection(_connectionString);
        return await conn.QueryFirstOrDefaultAsync<Product>(
            "SELECT * FROM Products WHERE Id = @Id",
            new { Id = id });
    }

    public async Task<IReadOnlyList<Product>> GetByPriceRangeAsync(decimal min, decimal max)
    {
        await using var conn = new SqlConnection(_connectionString);
        var results = await conn.QueryAsync<Product>(
            "SELECT * FROM Products WHERE Price BETWEEN @Min AND @Max",
            new { Min = min, Max = max });
        return results.ToList();
    }

    public async Task<int> CreateAsync(Product product)
    {
        await using var conn = new SqlConnection(_connectionString);
        return await conn.ExecuteAsync(
            @"INSERT INTO Products (Name, Price, CategoryId, CreatedAt)
              VALUES (@Name, @Price, @CategoryId, @CreatedAt)",
            product);
    }

    public async Task<bool> UpdatePriceAsync(int id, decimal price)
    {
        await using var conn = new SqlConnection(_connectionString);
        int rows = await conn.ExecuteAsync(
            "UPDATE Products SET Price = @Price WHERE Id = @Id",
            new { Id = id, Price = price });
        return rows > 0;
    }

    public async Task<bool> DeleteAsync(int id)
    {
        await using var conn = new SqlConnection(_connectionString);
        int rows = await conn.ExecuteAsync(
            "DELETE FROM Products WHERE Id = @Id",
            new { Id = id });
        return rows > 0;
    }
}
```

```csharp
// Multi-mapping: one-to-one relationships
public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public Customer? Customer { get; set; }
    public decimal Total { get; set; }
}

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

// Query with JOIN and split on CustomerId
var sql = @"SELECT o.*, c.*
            FROM Orders o
            INNER JOIN Customers c ON c.Id = o.CustomerId
            WHERE o.Id = @OrderId";

var order = await connection.QueryAsync<Order, Customer, Order>(
    sql,
    (order, customer) =>
    {
        order.Customer = customer;
        return order;
    },
    new { OrderId = 1 },
    splitOn: "CustomerId")  // Dapper splits columns at this column name
    .FirstOrDefault();
```

```csharp
// Multi-mapping: one-to-many relationships
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<Product> Products { get; set; } = new();
}

var sql = @"SELECT c.*, p.*
            FROM Categories c
            LEFT JOIN Products p ON p.CategoryId = c.Id
            WHERE c.Id = @CategoryId";

var lookup = new Dictionary<int, Category>();
var categories = connection.Query<Category, Product, Category>(
    sql,
    (category, product) =>
    {
        if (!lookup.TryGetValue(category.Id, out var existing))
        {
            existing = category;
            lookup.Add(category.Id, existing);
        }
        if (product is not null)
            existing.Products.Add(product);
        return existing;
    },
    new { CategoryId = 1 },
    splitOn: "Id")
    .Distinct()
    .ToList();
```

```csharp
// Stored procedure execution
public async Task<List<Product>> SearchProductsAsync(
    string searchTerm, int page, int pageSize)
{
    await using var conn = new SqlConnection(_connectionString);
    var results = await conn.QueryAsync<Product>(
        "usp_SearchProducts",
        new
        {
            SearchTerm = searchTerm,
            PageNumber = page,
            PageSize = pageSize
        },
        commandType: CommandType.StoredProcedure);
    return results.ToList();
}
```

```csharp
// Bulk insert with table-valued parameters
public async Task BulkInsertAsync(IEnumerable<Product> products)
{
    await using var conn = new SqlConnection(_connectionString);
    var table = new DataTable();
    table.Columns.Add("Name", typeof(string));
    table.Columns.Add("Price", typeof(decimal));
    table.Columns.Add("CategoryId", typeof(int));

    foreach (var p in products)
        table.Rows.Add(p.Name, p.Price, p.CategoryId);

    await conn.ExecuteAsync(
        "INSERT INTO Products (Name, Price, CategoryId) SELECT Name, Price, CategoryId FROM @Products",
        new { Products = table.AsTableValuedParameter("dbo.ProductType") });
}
```

```csharp
// Unit of work with transactions
public class UnitOfWork : IDisposable
{
    private readonly IDbConnection _connection;
    private IDbTransaction? _transaction;

    public UnitOfWork(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
        _connection.Open();
        _transaction = _connection.BeginTransaction();
    }

    public IDbConnection Connection => _connection;
    public IDbTransaction? Transaction => _transaction;

    public async Task CommitAsync()
    {
        _transaction?.Commit();
        _transaction?.Dispose();
        _transaction = null;
    }

    public async Task RollbackAsync()
    {
        _transaction?.Rollback();
        _transaction?.Dispose();
        _transaction = null;
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _connection?.Dispose();
    }
}
```

```csharp
// Dynamic parameters for flexible queries
public async Task<IEnumerable<Product>> FlexibleSearchAsync(
    string? name = null,
    decimal? minPrice = null,
    decimal? maxPrice = null,
    int? categoryId = null)
{
    var sql = new StringBuilder("SELECT * FROM Products WHERE 1=1");
    var parameters = new DynamicParameters();

    if (name is not null)
    {
        sql.Append(" AND Name LIKE @Name");
        parameters.Add("Name", $"%{name}%");
    }
    if (minPrice.HasValue)
    {
        sql.Append(" AND Price >= @MinPrice");
        parameters.Add("MinPrice", minPrice.Value);
    }
    if (maxPrice.HasValue)
    {
        sql.Append(" AND Price <= @MaxPrice");
        parameters.Add("MaxPrice", maxPrice.Value);
    }
    if (categoryId.HasValue)
    {
        sql.Append(" AND CategoryId = @CategoryId");
        parameters.Add("CategoryId", categoryId.Value);
    }

    return await connection.QueryAsync<Product>(sql.ToString(), parameters);
}
```

## 5. Real-World Scenarios

**Scenario 1: Stack Overflow Itself**
- Dapper was created for Stack Overflow's high-traffic environment.
- Raw SQL for full control over query plans.
- Microseconds-level overhead per query.

**Scenario 2: Reporting System**
- Complex SQL with window functions, CTEs, and aggregations.
- Dapper maps to flat DTOs or dynamic types.
- `QueryMultiple` for paginated results + total count.

**Scenario 3: High-Throughput API**
- `buffered: false` for streaming large result sets.
- `Async` methods for non-blocking I/O.
- Raw SQL with query hints for index selection.

**Scenario 4: Migration from EF Core**
- Performance-critical queries moved to Dapper.
- `Dapper.Contrib` for simple CRUD boilerplate.
- Manual SQL migration scripts.

## 6. Performance

```csharp
// Dapper overhead per query: ~2-10 microseconds (after cache warmup)
// EF Core overhead: ~50-500 microseconds (query compilation + change tracking)

// Raw ADO.NET: ~1-5 microseconds (but no object mapping)
// Dapper vs EF Core vs ADO.NET benchmark (1000 queries):

// Dapper:          ~15ms, 1.2MB allocated
// EF Core (NT):    ~80ms, 8.5MB allocated
// EF Core (Track): ~120ms, 15MB allocated
// ADO.NET (manual):~10ms, 0.8MB allocated
```

### Optimization Tips

```csharp
// 1. Use buffered: false for large result sets (streams, no memory spike)
var stream = connection.Query<Product>("SELECT * FROM HugeTable", buffered: false);

// 2. Use Async methods for non-blocking
await connection.QueryAsync<Product>(sql);

// 3. Use Dapper's type handler for custom types
SqlMapper.AddTypeHandler(new JsonTypeHandler<Metadata>());

// 4. Pre-compile queries? Dapper doesn't have compiled queries
//    But the deserialization is cached per <T, columns>, so subsequent calls are fast

// 5. Use CommandBehavior.SequentialAccess for large binary data
var reader = connection.ExecuteReader(sql, commandBehavior: CommandBehavior.SequentialAccess);
```

## 7. Security

```csharp
// Dapper parameterizes ALL query parameters automatically
// This prevents SQL injection when using anonymous types or DynamicParameters

// SAFE: parameterized
connection.Execute("DELETE FROM Products WHERE Id = @Id", new { Id = userInput });

// DANGEROUS: string concatenation
connection.Execute($"DELETE FROM Products WHERE Id = {userInput}"); // Injection!

// For dynamic table/column names:
// - DO validate against a whitelist
// - DO NOT concatenate user input directly
var allowedTables = new[] { "Products", "Categories" };
if (!allowedTables.Contains(tableName))
    throw new ArgumentException("Invalid table name");
connection.Execute($"SELECT * FROM {tableName}"); // Safe because whitelisted

// Connection strings: use managed identity or environment variables
```

## 8. Common Mistakes

```csharp
// MISTAKE 1: Not disposing connections
var conn = new SqlConnection(connStr);
var result = conn.Query(sql); // Connection not disposed!
// FIX: await using var conn = new SqlConnection(connStr);

// MISTAKE 2: Forgetting to open connection
// (Dapper opens/closes automatically, but for transactions you must open manually)

// MISTAKE 3: Not handling Dapper's "splitOn" correctly in multi-mapping
connection.Query<Order, Customer, Order>(sql, map, splitOn: "Id");
// splitOn defaults to "Id". If your JOIN has multiple Id columns, results will be wrong
// FIX: specify splitOn explicitly for each JOIN boundary

// MISTAKE 4: Using buffered: true for huge result sets
var all = connection.Query<HugeRow>("SELECT * FROM BillionRowTable"); // OutOfMemory!
// FIX: use buffered: false and iterate

// MISTAKE 5: Ignoring Dapper's return type
// ExecuteAsync returns Task<int> (rows affected), not null
// Always check the return value

// MISTAKE 6: Multi-mapping in query with no matching rows
var order = connection.Query<Order, Customer, Order>(sql, map, splitOn: "Id").FirstOrDefault();
// If no rows, FirstOrDefault returns default(Order) = null

// MISTAKE 7: Mixing sync and async
connection.QueryAsync(sql).Wait(); // Deadlock possible!
// FIX: await all the way

// MISTAKE 8: Not caching connection strings
// FIX: store in configuration, not hardcoded

// MISTAKE 9: Using Dapper with Entity Framework in same transaction
// EF and Dapper use different connections; wrap in TransactionScope for distributed tx

// MISTAKE 10: Overlooking Dapper.Contrib for simple CRUD
// Dapper.Contrib provides Get<T>, Insert<T>, Update<T>, Delete<T> for simple cases
```

## 9. Senior Engineer Perspective

**1. Dapper is not a replacement for EF Core.** Use Dapper for queries where you need full SQL control and maximum performance; use EF Core for complex domain logic with change tracking.

**2. Use Dapper with CQRS:** EF Core on the command side (domain logic); Dapper on the query side (read models, reporting).

**3. Type handlers for custom column mappings:**

```csharp
public class JsonTypeHandler<T> : SqlMapper.TypeHandler<T>
{
    public override T Parse(object value) =>
        JsonSerializer.Deserialize<T>((string)value)!;

    public override void SetValue(IDbDataParameter parameter, T value) =>
        parameter.Value = JsonSerializer.Serialize(value);
}

SqlMapper.AddTypeHandler(new JsonTypeHandler<Metadata>());
```

**4. Use `SqlMapper.IParameterCallback` for output parameters.**

**5. Consider `Dapper.FastCrud` or `DapperExtensions` if you need more LINQ-like syntax but prefer Dapper's performance.**

**6. Always use `AsList()` on `QueryAsync` results** to avoid multiple enumeration:

```csharp
var results = (await conn.QueryAsync<Product>(sql)).AsList();
```

**7. Connection pooling:** Each `new SqlConnection` uses the pool (by connection string). Creating many connections is cheap if pooling is enabled (default).

## 10. Interview Questions (Easy)

1. What is Dapper and why was it created?
2. How do you execute a query with Dapper?
3. How does Dapper map query results to objects?
4. What is the difference between `Query` and `Execute`?
5. How do you pass parameters to a Dapper query?
6. What is `QueryFirstOrDefault`?
7. What is `DynamicParameters` used for?
8. How does Dapper handle SQL injection?
9. What is the difference between `buffered: true` and `buffered: false`?
10. How do you execute a stored procedure with Dapper?

## 11. Interview Questions (Medium)

1. Explain how Dapper's multi-mapping (`Query<T1, T2, TReturn>`) works internally.
2. What is the `splitOn` parameter and how does it work?
3. How does Dapper cache type deserialization and what are the implications?
4. Compare Dapper + raw SQL vs EF Core LINQ for complex reporting queries.
5. How do you use `QueryMultiple` and what's the internal mechanism?
6. Explain how to handle one-to-many relationships with Dapper.
7. What is `Dapper.Contrib` and when would you use it?
8. How does Dapper handle custom type mapping (type handlers)?
9. Explain the performance characteristics of Dapper vs ADO.NET vs EF Core.
10. How do you use Dapper with transactions?

## 12. Advanced Interview Questions (Hard)

1. Explain Dapper's IL-based deserialization cache in detail. How does it achieve near-ADO.NET performance?
2. Design a Dapper-based repository that supports unit of work, transactions, and multi-tenancy.
3. Implement a custom Dapper type handler for a `DateOnly` / `TimeOnly` column.
4. Explain how `SqlMapper.LazyCache` works and how to clear it if needed.
5. Design a Dapper extension that automatically adds soft-delete filters to queries.
6. Implement a paging helper using Dapper that returns total count + page data in one round trip.
7. How would you handle database provider abstraction with Dapper (SQL Server vs PostgreSQL)?
8. Design a bulk insert strategy using Dapper that outperforms individual inserts.
9. Explain how to use Dapper's `CommandDefinition` for fine-grained command control.
10. Implement a Dapper-based multi-result-set sproc executor that maps to a unit of work.

## 13. Interview Questions (System Design)

1. Design a CQRS system using Dapper for reads and EF Core for writes.
2. Design a high-throughput read layer for a social media feed using Dapper.
3. Design a reporting dashboard with complex SQL queries mapped via Dapper.
4. Design a pagination strategy for 10M+ rows using keyset pagination with Dapper.
5. Design a data warehouse ingestion pipeline using Dapper bulk operations.
6. Design a multi-database query executor that queries SQL Server + PostgreSQL via Dapper.
7. Design an audit logging system using Dapper with temporal tables.
8. Design a distributed query engine using Dapper across sharded databases.
9. Design a materialized view refresh system using Dapper.
10. Design a real-time analytics pipeline combining Dapper with SignalR streaming.

## 14. Expert-Level Interview Questions (Architect)

1. Architect a high-performance ORM that combines Dapper's mapping speed with EF Core's change tracking and LINQ support.
2. Design a distributed query federator using Dapper that transparently routes queries to appropriate shards and aggregates results.
3. Architect a code-first model generator that reverse-engineers SQL schema into C# classes and Dapper mapping code.
4. Design a query profiler interceptor for Dapper that captures query execution time, row counts, and parameter values across all queries.
5. Architect a schema migration tool that works alongside Dapper, generating versioned SQL scripts with rollback support.
6. Design a polyglot persistence layer where Dapper abstracts SQL Server, PostgreSQL, and MySQL behind a common repository interface.
7. Architect a real-time change data capture (CDC) system that uses Dapper to poll for changes and streams them via Kafka.
8. Design a connection multiplexing layer for Dapper that pools and reuses connections across multiple repository instances.
9. Architect a dynamic query builder that generates optimized SQL based on runtime parameters, using Dapper for execution.
10. Design a cross-cutting instrumentation system using Dapper's profiling hooks that emits OpenTelemetry spans for every query.

## 15. Debugging & Troubleshooting

```csharp
// Enable Dapper logging
SqlMapper.LogHandler = (message, parameters) =>
{
    Debug.WriteLine(message);
    // Log parameters too
};

// Or use MiniProfiler with Dapper
// Install-Package MiniProfiler.AspNetCore

// Common issues:
// - "Column does not match property": column vs property name mismatch
//   -> Use column aliases: SELECT Name AS ProductName FROM ...
// - "No parameterless constructor": Dapper needs parameterless constructor or constructor with matching params
// - "Invalid object name": wrong table name or schema
// - "Timeout expired": long-running query; add CommandTimeout
// - "Cannot open database": connection string or server issue

// Debug SQL:
// - SQL Server Profiler / Extended Events
// - `SET STATISTICS IO ON` before queries
// - Use `Print` or `RAISERROR` in stored procedures
```

## 16. Comparison Section

```
+-----------------------+----------------+----------------+----------------+
| Feature               | Dapper         | EF Core        | ADO.NET        |
+-----------------------+----------------+----------------+----------------+
| ORM type              | Micro-ORM      | Full ORM       | None           |
| SQL abstraction       | Raw SQL        | LINQ + SQL     | Raw SQL        |
| Object mapping        | Automatic      | Automatic      | Manual         |
| Change tracking       | No             | Yes            | No             |
| Query compilation     | Cached IL      | LINQ -> SQL    | N/A            |
| Migration support     | No             | Yes            | No             |
| Performance overhead  | ~2-10 us       | ~50-500 us     | ~1-5 us        |
| Learning curve        | Low            | High           | Low            |
| SQL injection safety  | Parameterized  | Parameterized  | Manual         |
| Best for              | Performance    | Complex domain | Lowest level   |
+-----------------------+----------------+----------------+----------------+

+-----------------------+----------------+----------------+----------------+
| Aspect                | Dapper sync    | Dapper async   | Raw ADO async  |
+-----------------------+----------------+----------------+----------------+
| Overhead per query    | ~2 us          | ~5 us          | ~1 us          |
| Thread usage          | Blocks         | Non-blocking   | Non-blocking   |
| Memory allocation     | Low            | Low            | Minimal        |
| Code complexity       | Low            | Low            | High           |
| Cancellation support  | No             | Yes            | Yes            |
| Scalability           | Good           | Excellent      | Excellent      |
+-----------------------+----------------+----------------+----------------+
```

## 17. Revision Notes

- Extends `IDbConnection` with `Query`, `QueryAsync`, `Execute`, `ExecuteAsync`.
- Parameters via anonymous types or `DynamicParameters` (always parameterized).
- Multi-mapping: `Query<T1,T2,TReturn>(sql, map, splitOn)`.
- `QueryMultiple` for multiple result sets from one command.
- `buffered: false` for streaming large results (avoid memory spikes).
- `Dapper.Contrib` adds `Get<T>`, `Insert<T>`, `Update<T>`, `Delete<T>`.
- Custom type handlers via `SqlMapper.AddTypeHandler`.
- Caches deserialization IL per `<T, columns>` combination.
- No change tracking, no LINQ, no migrations.
- Use for performance-critical queries with full SQL control.

## 18. Cheat Sheet

```
+------------------------------------------------------------------+
|                    DAPPER CHEAT SHEET                             |
+------------------------------------------------------------------+
| QUERYING                                                          |
|  Query<T>(sql)                    - returns IEnumerable<T>       |
|  QueryAsync<T>(sql)               - async version                |
|  QueryFirst<T>(sql)               - first row, throws if none    |
|  QueryFirstOrDefault<T>(sql)      - first row or default(T)      |
|  QuerySingle<T>(sql)              - exactly one, throws          |
|  QuerySingleOrDefault<T>(sql)     - exactly one or default       |
|  QueryMultiple(sql)               - multiple result sets         |
+------------------------------------------------------------------+
| EXECUTION                                                         |
|  Execute(sql)                     - returns rows affected        |
|  ExecuteAsync(sql)                - async version                |
|  ExecuteScalar(sql)               - single value                 |
|  ExecuteReader(sql)               - raw IDataReader              |
+------------------------------------------------------------------+
| PARAMETERS                                                        |
|  new { Id = 1, Name = "foo" }    - anonymous type parameters    |
|  DynamicParameters               - dynamic parameter bag         |
|    .Add("@name", value, dbType)   - typed parameter              |
|    .Add("@out", dbType, dir: Output) - output parameter          |
+------------------------------------------------------------------+
| MULTI-MAPPING                                                     |
|  Query<T1,T2,TR>(sql, map, "splitOn") - 1:1 or 1:N              |
|  Query<T1,T2,T3,TR>(sql, map, "splitOn") - up to 7 types        |
+------------------------------------------------------------------+
| MISC                                                              |
|  buffered: false                  - streaming (non-materialized) |
|  commandType: CommandType.SP      - stored procedure             |
|  commandTimeout: 30               - timeout in seconds           |
|  transaction: tx                  - enlist in transaction        |
+------------------------------------------------------------------+
| DAPPER.CONTRIB                                                    |
|  conn.Get<T>(id)                  - get by primary key           |
|  conn.Insert<T>(entity)           - insert, returns identity    |
|  conn.Update<T>(entity)           - update by primary key        |
|  conn.Delete<T>(entity)           - delete by primary key        |
|  conn.GetAll<T>()                 - select all                   |
+------------------------------------------------------------------+
| BEST PRACTICES                                                    |
|  await using var conn = new SqlConnection(connStr);              |
|  Always parameterize queries (no string concat)                  |
|  Use Async variants in async contexts                            |
|  Set commandTimeout for long-running queries                     |
|  Use buffered: false for large result sets                       |
|  Add type handlers for custom SQL column types                   |
|  Use MiniProfiler to trace Dapper queries in dev                 |
|  Combine with EF Core: EF for writes, Dapper for reads (CQRS)   |
+------------------------------------------------------------------+
