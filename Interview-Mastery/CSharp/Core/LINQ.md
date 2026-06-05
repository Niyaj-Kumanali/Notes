# LINQ

---

## Overview

- **Definition:** Language Integrated Query — a set of language and runtime features for writing declarative, composable queries over data sources (objects, XML, databases) using a fluent API built on extension methods.
- **Why It Exists:** Provides a consistent, declarative query model across diverse data sources with deferred execution, lazy evaluation, and composability. Reduces imperative loop boilerplate and enables query translation (e.g., LINQ to SQL via EF Core).
- **Key Concepts:** **Extension methods** (`Where`, `Select`, `Aggregate` in `System.Linq.Enumerable`), **expression trees** (`IQueryable<T>` translates C# to provider-specific queries), **deferred execution** (most operators don't materialize until iterated), **streaming vs buffering operators**, **`IAsyncEnumerable<T>`** (async streaming), and **LINQ query syntax** vs **fluent syntax**.

---

## Core Concepts

- **Standard Query Operators:**
  - **Filtering:** `Where`, `OfType` — streaming, O(n)
  - **Projection:** `Select`, `SelectMany` — streaming, O(n)
  - **Partitioning:** `Take`, `Skip`, `TakeWhile`, `SkipWhile` — streaming
  - **Ordering:** `OrderBy`, `ThenBy`, `OrderByDescending`, `Reverse` — buffering, O(n log n)
  - **Grouping:** `GroupBy`, `ToLookup` — buffering, O(n)
  - **Set:** `Distinct`, `Union`, `Intersect`, `Except` — buffering
  - **Element:** `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`, `Last`, `ElementAt` — short-circuiting
  - **Aggregation:** `Count`, `Sum`, `Min`, `Max`, `Average`, `Aggregate`
  - **Quantifiers:** `Any`, `All`, `Contains`, `SequenceEqual`
  - **Conversion:** `ToArray`, `ToList`, `ToDictionary`, `ToHashSet`, `AsEnumerable`, `Cast`

```csharp
// Streaming operators yield as they go; buffering operators consume all before yielding
source.Where(x => x > 5).OrderBy(x => x.Name).Select(x => x * 2)
// Where: streaming → OrderBy: BUFFERS all → Select: streaming
```

- **Iterator State Machine:** The compiler transforms LINQ operators into state machines using `yield return`. Each operator creates a new wrapper `IEnumerable<T>` with its own enumerator struct. The `MoveNext()` method advances through the pipeline, calling `MoveNext()` on the source and applying transformations.

```csharp
// Compiler generates a state machine for: source.Where(x => x > 5).Select(x => x * 2)
// Internally: foreach (var x in source) { if (x > 5) yield return x * 2; }
```

- **Expression Trees vs Delegates:** `Func<T,bool>` is compiled IL executed locally. `Expression<Func<T,bool>>` is an expression tree that can be analyzed and translated (e.g., by EF Core to SQL). `IQueryable<T>` stores expression trees; `IEnumerable<T>` works with delegates.
- **Streaming vs Buffering:**
  - **Streaming:** `Where`, `Select`, `SelectMany`, `Take`, `Skip`, `Any`, `All`, `First` — process one element at a time, O(1) memory
  - **Buffering:** `OrderBy`, `GroupBy`, `Distinct`, `Union`, `Intersect`, `Reverse`, `ToLookup`, `ToArray`, `ToList` — consume entire sequence before yielding, O(n) or more memory

```csharp
// Bad: OrderBy before Where sorts everything, then filters
query.OrderBy(x => x.Name).Where(x => x.Age > 5); // Sorts 100% of data, filters after
// Good: filter first, then sort
query.Where(x => x.Age > 5).OrderBy(x => x.Name); // Filters before sorting
```

- **IQueryable vs IEnumerable:** `IQueryable<T>` composes expression trees, translated by the provider (e.g., SQL). `IEnumerable<T>` executes in memory with compiled IL. Calling `AsEnumerable()` switches from server-side to client-side evaluation. Mixing them can cause accidental client evaluation (EF Core warns about this).

---

## Common Mistakes

- **Multiple enumeration** — A LINQ query is re-evaluated each time it's enumerated. `filtered.Count()` and `filtered.First()` enumerate twice. Fix: materialize with `.ToList()` once.
- **Using `SingleOrDefault` when `FirstOrDefault` is intended** — `SingleOrDefault` throws if more than one match exists. Use `FirstOrDefault` unless the query is guaranteed to return at most one result.
- **Forgetting deferred execution** — LINQ queries are not evaluated until enumerated. Mutations to the source after query definition are reflected in results. Materialize early if the source may change.
- **Not understanding IQueryable vs IEnumerable boundary** — Calling `.AsEnumerable()` before a filter causes the filter to run client-side (pulling all data from database). Ensure filters are applied to `IQueryable` before materialization.
- **Modifying source in `Select`** — `items.Select(x => { x.Count++; return x; })` has side effects. `Select` should be pure. Use `foreach` for mutations.
- **Null collection** — `nullSource.Where(x => x > 5)` throws `NullReferenceException`. Use `source ?? Array.Empty<T>()`.
- **Exception inside iterator** — `source.Select(x => Parse(x))` doesn't throw until the result is enumerated. Wrap iteration in try/catch.

```csharp
// Problem: multiple enumeration
var filtered = source.Where(x => x > 5);
int count = filtered.Count();   // Enumerates once
var first = filtered.First();   // Enumerates again!
// Fix: materialize once
var materialized = source.Where(x => x > 5).ToList();
```

---

## Key Design Considerations

- **Choose `IQueryable` vs `IEnumerable` consciously** — `IQueryable` composes server-side SQL (reduced data transfer). `IEnumerable` executes in memory (arbitrary .NET code). Watch for accidental client evaluation.
- **Understand streaming vs buffering** — Chain streaming operators first (`Where`, `Select`), buffering operators last (`OrderBy`, `GroupBy`). This minimizes memory usage.
- **Prefer `Any()` over `Count() > 0`** — `Any()` short-circuits on the first match. `Count()` enumerates the entire sequence (unless `ICollection<T>` optimization applies).
- **Use `IAsyncEnumerable<T>` for async streaming** — `await foreach` with `SelectAwait`, `WhereAwait` for async transformations in streaming pipelines.
- **Expression tree manipulation for dynamic queries** — Build predicates by combining expressions with `Expression.AndAlso`/`OrElse` for dynamic filtering without string concatenation.

```csharp
public static Expression<Func<T, bool>> AndAlso<T>(
    this Expression<Func<T, bool>> left,
    Expression<Func<T, bool>> right)
{
    var parameter = Expression.Parameter(typeof(T));
    var body = Expression.AndAlso(
        Expression.Invoke(left, parameter),
        Expression.Invoke(right, parameter));
    return Expression.Lambda<Func<T, bool>>(body, parameter);
}
```

- **Avoid `OrderBy` before `Where`** — Filtering first reduces the data that needs sorting. This can dramatically improve performance for large datasets.

---

## Real-World Scenarios

### Scenario 1: Real-Time Log Aggregation Pipeline
**Context:** A monitoring system ingests 50K log entries/sec. Entries must be filtered, grouped by severity, enriched with context, and aggregated into 1-minute sliding windows.

```csharp
public class LogAggregationService
{
    private readonly ConcurrentDictionary<string, List<LogEntry>> _buffer = new();

    public IAsyncEnumerable<AggregatedMetric> AggregateAsync(
        IAsyncEnumerable<LogEntry> logStream, CancellationToken ct)
    {
        return logStream
            .Where(entry => entry.Timestamp > DateTime.UtcNow.AddMinutes(-1))
            .GroupBy(entry => entry.ServiceName)
            .SelectAwait(async group =>
            {
                var entries = await group.ToListAsync();
                return new AggregatedMetric
                {
                    ServiceName = group.Key,
                    ErrorCount = entries.Count(e => e.Severity >= LogLevel.Error),
                    AvgLatency = entries.Average(e => e.LatencyMs),
                    P99Latency = ComputeP99(entries.Select(e => e.LatencyMs)),
                    WindowStart = entries.Min(e => e.Timestamp)
                };
            });
    }

    private static double ComputeP99(IEnumerable<double> latencies)
    {
        var sorted = latencies.OrderBy(l => l).ToList();
        return sorted[(int)(sorted.Count * 0.99)];
    }
}
```

### Scenario 2: ETL Pipeline with Change Tracking
**Context:** A nightly ETL job reads 10M records from a source database, transforms them (filter, project, enrich with lookup data), and writes to a data warehouse. Must minimize memory and track progress.

```csharp
public class EtlPipeline
{
    public async Task RunEtlAsync(IQueryable<SourceRecord> source, CancellationToken ct)
    {
        var query = source
            .Where(r => r.ModifiedAt > _lastRun)
            .Select(r => new
            {
                r.Id,
                r.Name,
                r.CategoryId,
                r.Amount
            });

        // Execute in batches to avoid memory pressure
        var batchSize = 1000;
        int processed = 0, skipped = 0;

        await foreach (var batch in query.AsAsyncEnumerable().Buffer(batchSize))
        {
            var enriched = batch
                .Join(_lookupTable, r => r.CategoryId, l => l.Id, (r, l) => new TargetRecord
                {
                    Id = r.Id,
                    Name = r.Name,
                    Category = l.CategoryName,
                    Amount = r.Amount,
                    Tier = r.Amount > 1000 ? "Premium" : "Standard"
                })
                .Where(r => !string.IsNullOrEmpty(r.Name))
                .ToList();

            await BulkInsertAsync(enriched, ct);
            processed += enriched.Count;
            skipped += batch.Count - enriched.Count;
            
            _logger.LogInformation("ETL progress: {Processed}/{Total}", processed, await query.CountAsync(ct));
        }
    }
}
```

### Scenario 3: Hierarchical Data Flattening with SelectMany
**Context:** An organization chart must be flattened to produce a report of all employees and their reporting chains. The organization is an arbitrary-depth tree.

```csharp
public class OrgChartService
{
    private List<Employee> _allEmployees;

    // Recursive flatten with depth tracking using LINQ
    public IEnumerable<OrgReport> FlattenReportingChain()
    {
        return _allEmployees
            .Where(e => e.ManagerId == null) // Top-level managers
            .SelectMany(manager => FlattenSubtree(manager, 0));
    }

    private IEnumerable<OrgReport> FlattenSubtree(Employee employee, int depth)
    {
        yield return new OrgReport(employee.Name, employee.Title, depth);
        
        var reports = _allEmployees
            .Where(e => e.ManagerId == employee.Id)
            .OrderBy(e => e.Name)
            .ToList();

        foreach (var report in reports)
        {
            foreach (var sub in FlattenSubtree(report, depth + 1))
                yield return sub;
        }
    }
}
```

---

## Scenario-Based Questions

1. **Q: You are building a product catalog API. The query supports 15 optional filters (price range, category, rating, brand, etc.) and returns paged results. How do you build the LINQ query without client-side evaluation?**
   A: Start with `IQueryable<Product>` and conditionally append `.Where()` clauses based on which filters are provided. Each `.Where()` adds to the expression tree, and the final query is translated to a single SQL statement. Never call `.ToList()` or `.AsEnumerable()` before applying filters. Example:
```csharp
IQueryable<Product> query = _context.Products.AsQueryable();
if (minPrice.HasValue) query = query.Where(p => p.Price >= minPrice);
if (category != null) query = query.Where(p => p.Category == category);
var results = await query.Skip(page * size).Take(size).ToListAsync();
```
This generates a single SQL query with all filters in the WHERE clause. The order of `.Where()` calls doesn't affect SQL generation.

2. **Q: You have a performance-critical report that joins 5 tables and aggregates millions of rows. The LINQ query generates a Cartesian product with multiple `Include` calls. How do you fix it?**
   A: Replace `Include` with `Select` projections to pick only needed columns. Multiple `Include` calls on collection navigations generate JOINs that multiply rows (Cartesian explosion). Use `.AsSplitQuery()` to issue separate queries per collection, or use `Select` to project to anonymous types:
```csharp
var report = await _context.Orders
    .Where(o => o.Date > cutoff)
    .Select(o => new {
        o.Id, o.Total,
        Items = o.Items.Select(i => new { i.ProductName, i.Quantity }),
        Customer = new { o.Customer.Name, o.Customer.Email }
    })
    .ToListAsync();
```
This generates efficient SQL with separate SELECT statements for related data.

3. **Q: You need to find duplicate records in a 10M-row dataset by a composite key. How do you write the LINQ query for maximum performance?**
   A: Use `GroupBy` with a composite key (anonymous type) and filter groups with `Count() > 1`:
```csharp
var duplicates = await _context.Records
    .GroupBy(r => new { r.SourceId, r.TargetId })
    .Where(g => g.Count() > 1)
    .Select(g => new { g.Key.SourceId, g.Key.TargetId, Count = g.Count() })
    .ToListAsync();
```
This translates to SQL `GROUP BY ... HAVING COUNT(*) > 1`, which is efficient (database does the grouping). For very large datasets, consider a raw SQL approach with window functions (`ROW_NUMBER() OVER (PARTITION BY ...)`) for potentially better performance.

4. **Q: You have an `IEnumerable<T>` in memory and need to partition it into batches of 100 without materializing the entire sequence. How?**
   A: Use a custom `Batch` extension method that lazily yields batches using `yield return`:
```csharp
public static IEnumerable<IEnumerable<T>> Batch<T>(this IEnumerable<T> source, int size)
{
    using var enumerator = source.GetEnumerator();
    while (enumerator.MoveNext())
    {
        yield return BatchInner(enumerator, size);
    }
}
private static IEnumerable<T> BatchInner<T>(IEnumerator<T> enumerator, int size)
{
    do { yield return enumerator.Current; }
    while (--size > 0 && enumerator.MoveNext());
}
```
For `IAsyncEnumerable<T>`, `System.Linq.Async` NuGet provides `.Buffer(size)`. This pattern avoids materializing the whole sequence, processing in streaming fashion with O(batchSize) memory.

5. **Q: You are debugging a LINQ query that works locally but times out in production. What profiling steps do you take?**
   A: (1) Log the generated SQL: set `_context.Database.Log = sql => _logger.LogDebug(sql)` or use `_context.Products.ToQueryString()`. (2) Check for N+1: enable lazy loading logging or use a profiler. (3) Look for client evaluation: EF Core logs a warning when a query can't be translated. (4) Check for missing indexes by running the SQL in SSMS with `SET STATISTICS IO ON`. (5) Examine parameter sniffing: identical queries with different parameters may use different plans. (6) Use `AsSplitQuery()` if Cartesian explosion is suspected.

6. **Q: You need to implement a full-text search over a list of products in memory. The naive `Where(p => p.Name.Contains(query))` is too slow for 1M products. How do you optimize?**
   A: Build an inverted index: tokenize product names into words, create a `Lookup<string, Product>`, and search by token:
```csharp
var index = products
    .SelectMany(p => p.Name.Split(' ').Distinct(), (p, word) => new { p, word })
    .ToLookup(x => x.word, x => x.p);
var results = query.Split(' ')
    .Select(word => index[word])
    .Aggregate((a, b) => a.Intersect(b));
```
This reduces search from O(n) to O(m) where m is the number of products matching the rarest term. For prefix matching, use a trie data structure. For production, consider dedicated search (Elasticsearch, Azure Cognitive Search).

7. **Q: You have a LINQ query that uses `Skip` and `Take` for paging, but performance degrades as page number increases. Why and how do you fix it?**
   A: `Skip(10000).Take(20)` still reads and discards 10K rows — the database must scan and sort all rows to determine the correct offset. Fix: use keyset pagination (also called "seek method"):
```csharp
var lastSeenId = 0; // Store from the last item of the previous page
var page = await _context.Products
    .Where(p => p.Id > lastSeenId)
    .OrderBy(p => p.Id)
    .Take(20)
    .ToListAsync();
```
This uses an index seek instead of a scan, O(log n) per page regardless of page number. Works best with a unique, sequential key. For non-sequential keys, use `ORDER BY` + `WHERE (col1, col2) > (@val1, @val2)`.

8. **Q: You are using `Count()` in a loop over subsets of data. The database is hit 1000 times. How do you reduce this to a single query?**
   A: Use `GroupBy` with conditional aggregation:
```csharp
var counts = await _context.Orders
    .GroupBy(o => 1)
    .Select(g => new
    {
        Total = g.Count(),
        Pending = g.Count(o => o.Status == "Pending"),
        Shipped = g.Count(o => o.Status == "Shipped"),
        Cancelled = g.Count(o => o.Status == "Cancelled")
    })
    .FirstAsync();
```
This generates a single SQL query with `COUNT(*)` and `COUNT(CASE WHEN ...)` expressions. For multiple different groupings, use multiple `GroupBy` key selectors in separate queries, or use raw SQL with `SELECT COUNT(*) FILTER (WHERE ...)`.

9. **Q: You are mixing `IQueryable` and `IEnumerable` in an EF Core query and it's pulling all data into memory before filtering. How do you detect and fix client evaluation?**
   A: EF Core logs a warning when a query can't be translated and falls back to client evaluation. In EF Core 6+, enable `throwOnClientEvaluation: true` in `DbContextOptionsBuilder`. Common causes: using a custom C# method in `Where`, calling `ToList`/`AsEnumerable` too early, or using `Sum` on a non-translatable expression. Fix: ensure all filter expressions are composed on `IQueryable` before materialization. Move custom logic to a `Select` that translates (or use `FromSqlRaw` for complex logic).

10. **Q: You need to stream 1M records from a database to a CSV file without loading all into memory. How do you use LINQ to achieve this?**
    A: Use `AsAsyncEnumerable()` with streaming projection and write each row:
```csharp
await foreach (var record in _context.LargeTable
    .AsNoTracking()
    .Where(r => r.Date >= start)
    .Select(r => new { r.Id, r.Name, r.Value })
    .AsAsyncEnumerable()
    .WithCancellation(ct))
{
    await writer.WriteLineAsync($"{record.Id},{EscapeCsv(record.Name)},{record.Value}");
    Interlocked.Increment(ref count);
}
```
This translates to a single SQL query with streaming (`CommandBehavior.SequentialAccess`). Never call `ToListAsync()` — it materializes all rows. Use `AsNoTracking()` to avoid change tracking overhead. For maximum throughput, use `SqlBulkCopy` or `CsvHelper` with streaming.

---

## Interview Questions

1. **What is LINQ?**
   A: Language Integrated Query — a set of language and runtime features for writing declarative, composable queries over data sources using a fluent API built on extension methods. Supports objects, XML, databases, and more.

2. **What is the difference between `IEnumerable<T>` and `IQueryable<T>`?**
   A: `IEnumerable<T>` works with delegates (compiled IL) — queries execute in memory. `IQueryable<T>` works with expression trees — queries are translated by a provider (e.g., EF Core to SQL) and executed on the server. `IQueryable<T>` extends `IEnumerable<T>` and adds a `Provider` and `Expression`.

3. **What is deferred execution?**
   A: Most LINQ operators don't execute until the query is enumerated. The query is built as a chain of iterators, and data flows through the pipeline only when `MoveNext()` is called (e.g., via `foreach`, `.ToList()`, `.Count()`). Multiple enumerations re-execute the query.

4. **What is the difference between `Select` and `SelectMany`?**
   A: `Select` projects each element to a single result: `IEnumerable<T> → IEnumerable<U>`. `SelectMany` projects each element to an `IEnumerable<U>` and flattens: `IEnumerable<T> → IEnumerable<U>`. Query syntax: `from c in customers from o in c.Orders` is `SelectMany`.

5. **What is the difference between `First` and `Single`?**
   A: `First` returns the first element (throws if empty). `Single` returns the only element (throws if empty OR more than one). Use `First` when the result may have multiple matches but you only need the first. Use `Single` when exactly one element is expected (acts as an assertion).

6. **How does `GroupBy` work?**
   A: It's a buffering operator that groups elements by a key selector. It creates a `Lookup<TKey, TElement>` internally, iterating the entire source and storing elements in hash buckets by key. It yields each `IGrouping<TKey, TElement>` (key + enumerable of elements). Memory is O(n).

7. **What's the difference between `Count()` and `Any()`?**
   A: `Count()` enumerates the entire sequence (unless `ICollection<T>` optimization applies). `Any()` short-circuits on the first match. Use `Any()` to check if any elements exist — it's O(1) when the first match is early, while `Count() > 0` is always O(n).

8. **Explain streaming vs buffering operators.**
   A: Streaming operators (Where, Select, Take, Any) process one element at a time with O(1) memory. Buffering operators (OrderBy, GroupBy, Distinct) consume the entire sequence before yielding results, using O(n) or more memory. Chain streaming operators first, buffering operators last.

9. **What is the `AsEnumerable()` method used for?**
   A: It casts an `IQueryable<T>` to `IEnumerable<T>`, switching from server-side (LINQ-to-SQL) to client-side (LINQ-to-Objects) evaluation. All subsequent operators execute in memory. Use it to force client evaluation when the server can't translate a query, but beware of pulling too much data.

10. **How does `Join` work internally in LINQ?**
    A: `Enumerable.Join` uses a hash join algorithm: it buffers the inner sequence into a `Lookup<TKey, TInner>`, then iterates the outer sequence, looking up matching elements. O(n+m) time, O(m) memory. `Queryable.Join` translates to SQL `JOIN` and lets the database choose the join algorithm.

---

## Developer Recommendations

- **Prefer `Any()` over `Count() > 0`** — `Any()` short-circuits on the first match, while `Count()` enumerates the entire sequence (unless `ICollection<T>` optimization applies). The difference is meaningful for large sequences or database queries where `COUNT(*)` is more expensive than checking for existence.

- **Avoid multiple enumeration of LINQ queries** — Each `foreach`, `.ToList()`, `.Count()`, or `.First()` on an `IEnumerable<T>` re-executes the query. Materialize with `.ToList()` once if you need multiple operations. For database queries, each enumeration hits the database. Use `.AsEnumerable()` only when you intend client-side execution.

- **Filter with `Where` before sorting with `OrderBy`** — `Where` is streaming (O(1) memory), `OrderBy` is buffering (O(n) memory). Applying `Where` first reduces the data that needs sorting, improving performance and memory usage. This also applies to database queries — filters before sorts produce more efficient SQL.

- **Use `ToListAsync()` instead of `.ToList()` in async contexts** — Blocking on a database query with `.ToList()` or `.Count()` ties up a thread pool thread. Always use the async variants (`ToListAsync()`, `CountAsync()`, `FirstAsync()`) in async methods to prevent thread pool starvation.

- **Understand the `IQueryable`/`IEnumerable` boundary** — Calling `.AsEnumerable()` or `.ToList()` before a `.Where()` causes the filter to execute client-side, pulling all data from the database. Always apply filters to `IQueryable` before materializing. Use `ToQueryString()` (EF Core 5+) to inspect the generated SQL and verify translation.

- **Prefer `Aggregate` over loops for cumulative operations** — `Aggregate` expresses accumulation declaratively and is often more readable:
```csharp
var result = numbers.Aggregate((a, b) => a + b); // Sum
var csv = items.Aggregate("", (acc, item) => acc + "," + item)[1..]; // CSV
```
However, `Aggregate` is less readable for complex logic — use `foreach` when clarity matters.

- **Use `ToLookup` for one-to-many dictionary patterns** — `ToLookup()` creates an `ILookup<TKey, TElement>` — an immutable dictionary of sequences. Unlike `GroupBy`, it's materialized immediately and provides O(1) key lookup. Unlike `ToDictionary`, it handles duplicate keys (returns all elements per key). Ideal for building index structures.

---

## Operator Performance

| Operator | Behavior | Memory |
|---|---|---|
| `Where` / `Select` | Streaming | O(1) |
| `SelectMany` | Streaming | O(1) |
| `Take(n)` / `Skip(n)` | Streaming | O(1) |
| `OrderBy` / `ThenBy` | Buffering | O(n) |
| `GroupBy` | Buffering | O(n) |
| `Distinct` | Buffering | O(n) |
| `Union` / `Intersect` | Buffering | O(n+m) |
| `Join` / `GroupJoin` | Buffering (inner) | O(m) |
| `Reverse` | Buffering | O(n) |
| `ToArray` / `ToList` | Buffering | O(n) |

## LINQ vs SQL Mapping

| LINQ | SQL |
|---|---|
| `.Where(p)` | `WHERE p` |
| `.Select(m)` | `SELECT m` |
| `.GroupBy(k)` | `GROUP BY k` |
| `.OrderBy(k)` | `ORDER BY k` |
| `.Join(...)` | `JOIN ... ON` |
| `.SelectMany` | `CROSS APPLY` |
