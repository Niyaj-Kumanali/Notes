# LINQ

## 1. Executive Summary

Language Integrated Query (LINQ) is a set of language and runtime features for writing declarative, composable queries over data sources (objects, XML, databases, etc.). LINQ to Objects operates on `IEnumerable<T>`, using deferred execution, lazy evaluation, and a fluent API built on extension methods. Mastery of LINQ is essential for writing concise, readable, and efficient C# code.

## 2. Core Theory

LINQ is built on three pillars:

- **Extension Methods**: `Where`, `Select`, `Aggregate`, etc. are static methods in `System.Linq.Enumerable` extending `IEnumerable<T>`.
- **Expression Trees**: `IQueryable<T>` providers (EF Core) translate C# expressions into SQL.
- **Deferred Execution**: Most operators do not materialize results until iterated.

Standard Query Operators categories:
- Filtering: `Where`, `OfType`
- Projection: `Select`, `SelectMany`
- Partitioning: `Take`, `Skip`, `TakeWhile`, `SkipWhile`
- Ordering: `OrderBy`, `ThenBy`, `OrderByDescending`, `Reverse`
- Grouping: `GroupBy`, `ToLookup`
- Set: `Distinct`, `Union`, `Intersect`, `Except`
- Element: `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`, `Last`, `ElementAt`
- Aggregation: `Count`, `Sum`, `Min`, `Max`, `Average`, `Aggregate`
- Quantifiers: `Any`, `All`, `Contains`, `SequenceEqual`
- Conversion: `ToArray`, `ToList`, `ToDictionary`, `ToHashSet`, `AsEnumerable`, `Cast`

## 3. Under-the-Hood Deep Dive

### Iterator Pattern and State Machine

```csharp
// The compiler transforms LINQ into state machine calls.
// Example: source.Where(x => x > 5).Select(x => x * 2)

// Compiler generates something like:
IEnumerable<int> WhereSelect(IEnumerable<int> source)
{
    foreach (var x in source)
    {
        if (x > 5)            // Where predicate
            yield return x * 2;  // Select projection
    }
}
```

```csharp
// Decompiled state machine (simplified)
internal class WhereSelectIterator : IEnumerable<int>, IEnumerator<int>
{
    private int _state;
    private int _current;
    private IEnumerator<int> _sourceEnumerator;

    public bool MoveNext()
    {
        switch (_state)
        {
            case 0: _state = 1;
                    _sourceEnumerator = _source.GetEnumerator(); break;
            case 1: goto loop;
        }
        return false;

        loop:
        while (_sourceEnumerator.MoveNext())
        {
            int x = _sourceEnumerator.Current;
            if (x > 5)
            {
                _current = x * 2;
                return true;
            }
        }
        _state = -1;
        _sourceEnumerator?.Dispose();
        return false;
    }
}
```

### Expression Trees vs Delegates

```csharp
// Delegate: compiled IL, executed locally
Func<int, bool> predicate = x => x > 5;
source.Where(predicate);  // Enumerable.Where

// Expression tree: represented as tree of Node objects, can be analyzed/translated
Expression<Func<int, bool>> expr = x => x > 5;
// expr.Body is BinaryExpression (GreaterThan)
// expr.Parameters[0] is ParameterExpression "x"
// expr.Body.Left is x, Body.Right is ConstantExpression 5
queryable.Where(expr);    // Queryable.Where -> translates to SQL
```

### Method Chaining and Intermediate Allocations

```csharp
// Each operator creates a new wrapper IEnumerable<T>.
// source.Where(...).OrderBy(...).Select(...)
// -> WhereEnumerableIterator wraps source
// -> OrderByEnumerable wraps Where iterator (buffers all! due to sorting)
// -> SelectEnumerableIterator wraps OrderBy

// OrderBy is a BUFFERING operator: consumes entire input before yielding.
// Where/Select are STREAMING operators: yield as they go.
```

## 4. Production Code Examples

```csharp
// Complex ETL pipeline
public IEnumerable<ReportRow> GenerateReport(IEnumerable<RawData> source)
{
    return source
        .Where(r => r.Status != Status.Deleted)
        .SelectMany(r => r.LineItems)
        .GroupBy(li => new { li.Category, li.Region })
        .Select(g => new ReportRow
        {
            Category = g.Key.Category,
            Region = g.Key.Region,
            TotalSales = g.Sum(li => li.Amount),
            AveragePrice = g.Average(li => li.UnitPrice),
            TransactionCount = g.Count()
        })
        .OrderByDescending(r => r.TotalSales)
        .ThenBy(r => r.Category);
}
```

```csharp
// Paginated API with total count
public async Task<PagedResult<T>> GetPagedAsync<T>(
    IQueryable<T> query,
    int page,
    int pageSize,
    Expression<Func<T, bool>> filter = null)
{
    if (filter != null) query = query.Where(filter);

    var totalCount = await query.CountAsync();
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();

    return new PagedResult<T>
    {
        Items = items,
        TotalCount = totalCount,
        Page = page,
        PageSize = pageSize
    };
}
```

```csharp
// Lookup for 1:N relationship (avoids GroupBy materialization)
public ILookup<int, Order> BuildOrderLookup(IEnumerable<Order> orders) =>
    orders.ToLookup(o => o.CustomerId);

// Usage: var customerOrders = lookup[customerId]; // fast
```

```csharp
// Custom LINQ operator
public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source) where T : class
{
    foreach (var item in source)
    {
        if (item is not null)
            yield return item;
    }
}
```

```csharp
// Batch processing with channels
public async IAsyncEnumerable<T> ProcessBatchedAsync<T>(
    IEnumerable<T> source,
    int batchSize,
    Func<List<T>, Task> processor)
{
    var batch = new List<T>(batchSize);
    foreach (var item in source)
    {
        batch.Add(item);
        if (batch.Count >= batchSize)
        {
            await processor(batch);
            foreach (var processed in batch) yield return processed;
            batch.Clear();
        }
    }
    if (batch.Count > 0)
    {
        await processor(batch);
        foreach (var processed in batch) yield return processed;
    }
}
```

## 5. Real-World Scenarios

**Scenario 1: Reporting Dashboard**
- Aggregate 1M+ records with `GroupBy` + aggregations on the database side (IQueryable).
- Avoid pulling raw data into memory; let EF translate to SQL GROUP BY.

**Scenario 2: Real-Time Data Pipeline**
- Use `Channel<T>` with `ReadAllAsync()` and `Select` transforms.
- Use `ToLookup` for 1:N joins in memory.

**Scenario 3: Message Routing**
- Use `OfType<T>()` to filter messages from a heterogeneous stream.

**Scenario 4: Validation Rule Engine**
- Chain `Where`, `SelectMany`, `Any`, `All` to express complex rules.

## 6. Performance

| Operator          | Behavior     | Notes                                  |
|-------------------|--------------|----------------------------------------|
| Where             | Streaming    | O(n), one pass                         |
| Select            | Streaming    | O(n), no internal buffering            |
| SelectMany        | Streaming    | O(n*m), flattens                       |
| OrderBy           | Buffering    | O(n log n), consumes entire sequence   |
| GroupBy           | Buffering    | O(n), builds lookup table in memory    |
| Distinct          | Buffering    | O(n), uses Set<T> internally           |
| Union/Intersect   | Buffering    | O(n+m), uses Set<T>                    |
| Take(n)           | Streaming    | O(n) worst, short-circuits             |
| Skip(n)           | Streaming    | O(n), must traverse skipped elements   |
| First/Any         | Streaming    | Short-circuits on match                |
| Count()           | Streaming   | O(n) unless ICollection<T> optimized   |
| ElementAt(i)      | Streaming    | O(n) unless IList<T> optimized         |
| Reverse           | Buffering    | O(n), buffers all                      |
| ToList/ToArray    | Buffering    | O(n), materializes                     |
| Contains          | Streaming    | O(n) unless ICollection<T>             |

```csharp
// BAD: Multiple enumeration
var filtered = source.Where(x => x > 5);
int count = filtered.Count();     // Enumerates once
var first = filtered.First();     // Enumerates again!
// FIX: Materialize once, use the list
var materialized = source.Where(x => x > 5).ToList();

// BAD: OrderBy before Where
query.OrderBy(x => x.Name).Where(x => x.Age > 5); // Sorts everything, then filters
// FIX: Filter first, then sort
query.Where(x => x.Age > 5).OrderBy(x => x.Name);
```

## 7. Security

```csharp
// SQL Injection via dynamic LINQ
public IQueryable<User> SearchUsers(string name)
{
    // DANGEROUS: string interpolation in IQueryable
    return _context.Users.Where($"Name == \"{name}\""); // Injection!
}

// SAFE: parameterized predicates
return _context.Users.Where(u => u.Name == name);

// SAFE: pass Expression<Func<T,bool>> instead of raw strings
```

```csharp
// Denial of service via large sequences
public IEnumerable<T> UnsafePagination<T>(IEnumerable<T> source, int page, int pageSize)
{
    return source.Skip((page - 1) * pageSize).Take(pageSize);
    // If page is huge, Skip iterates through millions of elements
    // FIX: Cap page to, e.g., maxPage = 1000
}
```

## 8. Common Mistakes

```csharp
// MISTAKE 1: Multiple enumeration
var result = source.Where(pred);
if (result.Any())                     // Enumerates
    ProcessItems(result.ToList());    // Enumerates again!
// FIX: ToList once

// MISTAKE 2: Using SingleOrDefault when FirstOrDefault is intended
var user = users.SingleOrDefault(u => u.Id == id);
// Throws if more than one match; use FirstOrDefault unless uniqueness is enforced

// MISTAKE 3: Forgetting deferred execution
var items = source.Where(x => x > 5);
source.Add(10);  // Mutation after query definition
// items now includes 10!

// MISTAKE 4: Not understanding IQueryable vs IEnumerable
var query = _context.Products.Where(p => p.Price > 100);
var filtered = query.Where(p => p.Category == "Electronics"); // Still IQueryable, goes to DB
// vs
var local = query.AsEnumerable().Where(p => SomeLocalFunc(p)); // Client-side eval

// MISTAKE 5: Nested SelectMany with anonymous types causing Cartesian explosion
var q = from c in customers
        from o in c.Orders
        from i in o.Items
        select new { c.Name, o.Date, i.Product };  // Large result set
// FIX: Be specific, use joins or pagination

// MISTAKE 6: Modifying source in Select
items.Select(x => { x.Count++; return x; }); // Side-effects in projection
// FIX: Use foreach for mutations

// MISTAKE 7: Exception inside iterator
var source = GetData();
var result = source.Select(x => Parse(x));  // No exception yet
foreach (var r in result) { } // Exception thrown here!
// FIX: Use ToList or wrap in try/catch around enumeration

// MISTAKE 8: Null collection
IEnumerable<int> nullSource = null;
nullSource.Where(x => x > 5); // NullReferenceException
// FIX: Use ?? Array.Empty<int>()
```

## 9. Senior Engineer Perspective

**1. Choose IQueryable vs IEnumerable consciously.**
- IQueryable = compose SQL, execute on server. Benefits: reduced data transfer.
- IEnumerable = execute in memory. Benefits: arbitrary .NET code in predicates.
- Watch for accidental client-side evaluation: EF Core logs a warning.

**2. Understand streaming vs buffering operators.**
- `OrderBy`, `GroupBy`, `Distinct`, `Union`, `Join` all buffer.
- Chain streaming operators first, buffering operators last.

**3. Use `ToHashSet()` for unique lookups instead of `Distinct().ToDictionary()`.**

**4. Custom LINQ operators via `yield return` are zero-allocation wrappers (no extra list).**

**5. Prefer `Any()` over `Count() > 0`** — Any short-circuits on first match.

**6. Use `IAsyncEnumerable<T>` for async streaming** with `await foreach`.

```csharp
await foreach (var item in GetResultsAsync().SelectAwait(ProcessAsync))
{
    Console.WriteLine(item);
}
```

**7. Expression tree manipulation for dynamic queries:**

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

## 10. Interview Questions (Easy)

1. What is LINQ and what are its main components?
2. Explain deferred execution vs immediate execution with examples.
3. What is the difference between `Select` and `SelectMany`?
4. What does `Where` do? Can you chain multiple `Where` calls?
5. What is the difference between `First`, `FirstOrDefault`, `Single`, and `SingleOrDefault`?
6. How does `IEnumerable<T>` differ from `IQueryable<T>`?
7. What is `OfType<T>` used for?
8. Explain the purpose of `ToList()` and `ToArray()`.
9. What is the difference between `Any` and `All`?
10. How do you use `OrderBy` and `ThenBy` for multi-level sorting?

## 11. Interview Questions (Medium)

1. Explain how the compiler transforms a LINQ query with `yield return`.
2. What is the difference in behavior between `Enumerable.OrderBy` and `Queryable.OrderBy`?
3. How does `SelectMany` flatten nested collections? Show the equivalent query syntax.
4. What is `ILookup<TKey, TElement>` and how does it differ from `IGrouping<TKey, TElement>`?
5. Explain how `GroupBy` works internally (what does it buffer?).
6. What is expression tree and how does LINQ to SQL (EF Core) use it?
7. Write a `DistinctBy` equivalent (pre-.NET 6) using `GroupBy`.
8. How does `Join` work in LINQ (what algorithm does it use internally)?
9. Explain `Enumerable.Aggregate` with a practical example.
10. What happens when you call `Count()` on an `ICollection<T>` vs a pure `IEnumerable<T>`?

## 12. Advanced Interview Questions (Hard)

1. Implement a custom LINQ operator `Batch(n)` that splits sequence into chunks without materializing the entire sequence.
2. Design a streaming `MergeJoin` operator that merges two sorted sequences in O(n+m).
3. Explain how `Expression<TDelegate>` enables query translation. Write a minimal SQL translator for a subset of LINQ.
4. How would you implement `ToFrozenDictionary` using perfect hashing? What are the constraints?
5. Implement a `DistinctBy` operator that preserves stable order and is O(n).
6. Design a `Rank()` window function operator (like SQL RANK() OVER (ORDER BY ...)).
7. Explain the trade-offs between `IAsyncEnumerable<T>`, `IObservable<T>`, and `Task<IEnumerable<T>>`.
8. Implement a lazy `CartesianProduct` operator using LINQ and deferred execution.
9. Design an operator `WhereWithCancellation` that respects `CancellationToken` during streaming.
10. Write a `Memoize` operator that caches enumeration results for replay.

## 13. Interview Questions (System Design)

1. Design a real-time analytics pipeline using `IAsyncEnumerable<T>` and windowed aggregations.
2. Design a graph traversal engine using LINQ-style operators (BFS, DFS).
3. Design a distributed query engine that pushes predicates down to shards.
4. Design an ETL framework using composable LINQ operators with checkpointing.
5. Design a rule engine with 1000s of rules using expression trees and LINQ.
6. Design a log aggregation system using LINQ over structured log streams.
7. Design a streaming change-data-capture (CDC) processor with LINQ operators.
8. Design a multi-tenant reporting system where each tenant has custom LINQ queries.
9. Design an in-memory data warehouse with star-schema joins using LINQ.
10. Design a real-time fraud detection system using windowed LINQ queries over event streams.

## 14. Expert-Level Interview Questions (Architect)

1. Design a full LINQ provider for a custom data source (e.g., Redis, Elasticsearch) with full expression tree translation, caching of compiled queries, and client-side fallback.
2. Architect a distributed LINQ engine that transparently partitions queries across shards, aggregates results, and handles partial failures.
3. Design a query optimization framework that rewrites LINQ expression trees (e.g., push down filters, reorder joins, eliminate redundant subqueries).
4. Architect a reactive event sourcing system where projections are defined as LINQ queries over event streams with automatic checkpointing.
5. Design a compiler transformation that converts LINQ queries into SIMD-optimized vectorized loops for hot paths.
6. Architect a cross-language LINQ-like query system (C#, F#, Python) using a common intermediate query representation (QIR).
7. Design a stream processor that supports exactly-once semantics using LINQ operators with checkpointed state.
8. Architect a query federation layer that splits a single LINQ query across SQL, NoSQL, and file-based data sources.
9. Design a lazy-materialization framework where LINQ operators work over memory-mapped files with zero-copy.
10. Architect a self-tuning query engine that collects statistics and rewrites LINQ plans based on cardinality estimation.

## 15. Debugging & Troubleshooting

```csharp
// Debug LINQ pipelines
// Use .Select(x => { Debug.WriteLine(x); return x; }) as a peek operator
var result = source
    .Where(x => x > 5)
    .Select(x =>
    {
        Console.WriteLine($"Processing {x}");
        return x * 2;
    })
    .ToList();

// Use breakpoints inside lambda by adding a dummy variable:
var result = source
    .Where(x =>
    {
        bool flag = x > 5; // Breakpoint here
        return flag;
    })
    .ToList();

// Common issues:
// - "Operation not supported" with IQueryable -> client-side eval
// - NullReferenceException from null elements
// - StackOverflow with recursive SelectMany
// - OutOfMemory with buffering operators on large datasets

// Use LINQPad or dotnet-counters for memory diagnostics
```

## 16. Comparison Section

```
+------------------------+---------------------+--------------------------+
| Operator               | Streaming/Buffering | Memory                   |
+------------------------+---------------------+--------------------------+
| Where / Select         | Streaming           | O(1)                     |
| SelectMany             | Streaming           | O(1) + inner enumerator  |
| Take(n) / Skip(n)      | Streaming           | O(1)                     |
| OrderBy / ThenBy       | Buffering           | O(n)                     |
| GroupBy                | Buffering           | O(n) (builds lookup)     |
| Distinct               | Buffering           | O(n) (hash set)          |
| Union / Intersect      | Buffering           | O(n+m)                   |
| Join / GroupJoin       | Buffering (inner)   | O(n+m) (lookup)          |
| Concat                 | Streaming           | O(1)                     |
| Reverse                | Buffering           | O(n)                     |
| ToList / ToArray       | Buffering           | O(n)                     |
+------------------------+---------------------+--------------------------+

+--------------------+-----------------------+-----------------------+
| Aspect             | IEnumerable<T>        | IQueryable<T>         |
+--------------------+-----------------------+-----------------------+
| Execution          | In-memory (CLR)       | Provider-dependent    |
| Predicate          | Func<T,bool> (IL)     | Expression<Func>      |
| Deferred           | Yes                   | Yes                   |
| Translation        | None (native IL)      | Provider translates   |
| Overloads          | Enumerable class      | Queryable class       |
| Best for           | In-memory collections | Remote data sources   |
+--------------------+-----------------------+-----------------------+

+-----------------+--------------+---------------+---------------+
| Feature         | LINQ (Fluent)| LINQ (Query)  | SQL           |
+-----------------+--------------+---------------+---------------+
| Filter          | .Where(p)    | where p       | WHERE p       |
| Project         | .Select(m)   | select m      | SELECT m      |
| Group           | .GroupBy(k)  | group by k    | GROUP BY k    |
| Order           | .OrderBy(k)  | orderby k     | ORDER BY k    |
| Join            | .Join(...)   | join ... in   | JOIN ... ON   |
| Flatten         | .SelectMany  | from ... in   | CROSS APPLY   |
+-----------------+--------------+---------------+---------------+
```

## 17. Revision Notes

- Deferred execution: query is not evaluated until enumerated.
- Streaming operators process one element at a time; buffering operators (OrderBy, GroupBy) consume all.
- IQueryable translates to provider-specific query (SQL, etc.).
- Expression trees enable dynamic query composition.
- LINQ evaluation model: foreach loop with GetEnumerator/MoveNext/Dispose.
- Yield return generates a state machine struct.
- Always call ToList/ToArray if the source will be mutated later.
- Use Any() not Count() > 0 for collections.
- Use IAsyncEnumerable<T> for async streaming with await foreach.

## 18. Cheat Sheet

```
+------------------------------------------------------------------+
|                       LINQ CHEAT SHEET                            |
+------------------------------------------------------------------+
| FILTERING                                                         |
|  Where(pred)           - filter by predicate                     |
|  OfType<T>()           - filter by type                          |
|  Distinct()            - remove duplicates (uses default comparer)|
+------------------------------------------------------------------+
| PROJECTION                                                        |
|  Select(mapper)        - transform each element                  |
|  SelectMany(mapper)    - flatten nested collections               |
|  Zip(second, func)     - pairwise combine                        |
+------------------------------------------------------------------+
| ORDERING                                                          |
|  OrderBy(key)          - ascending sort                          |
|  OrderByDescending(key)- descending sort                         |
|  ThenBy(key)           - secondary sort                          |
|  Reverse()             - reverse order                           |
+------------------------------------------------------------------+
| AGGREGATION                                                       |
|  Count() / LongCount() - count elements                          |
|  Sum(s)                - sum of numeric values                   |
|  Min(s) / Max(s)       - min/max value                           |
|  Average(s)            - arithmetic mean                         |
|  Aggregate(seed, func) - custom accumulation                     |
+------------------------------------------------------------------+
| ELEMENT                                                           |
|  First(pred)           - first (throws if none)                  |
|  FirstOrDefault(pred)  - first or default                        |
|  Last(pred)            - last element                            |
|  Single(pred)          - exactly one (throws)                    |
|  ElementAt(i)          - element at index                        |
+------------------------------------------------------------------+
| QUANTIFIERS                                                       |
|  Any(pred)             - true if any match                       |
|  All(pred)             - true if all match                       |
|  Contains(item)        - true if contains item                   |
+------------------------------------------------------------------+
| SET OPERATIONS                                                    |
|  Distinct()            - unique elements                         |
|  Union(second)         - set union                               |
|  Intersect(second)     - set intersection                        |
|  Except(second)        - set difference                          |
+------------------------------------------------------------------+
| GROUPING & JOINING                                                |
|  GroupBy(keySelector)  - group by key                            |
|  ToLookup(keySelector) - 1:N lookup (immutable groups)           |
|  Join(inner, outerKey, innerKey, result) - inner join            |
|  GroupJoin(...)        - left outer join                         |
+------------------------------------------------------------------+
| CONVERSION                                                        |
|  ToList()              - materialize to List<T>                  |
|  ToArray()             - materialize to T[]                      |
|  ToDictionary(k, v)    - materialize to Dictionary<K,V>          |
|  ToHashSet()           - materialize to HashSet<T>               |
|  AsEnumerable()        - switch from IQueryable to IEnumerable   |
|  AsQueryable()         - wrap in IQueryable                      |
+------------------------------------------------------------------+
| EXECUTION TIPS                                                    |
|  Deferred: Where, Select, SelectMany, Take, Skip, Distinct       |
|  Immediate: ToList, ToArray, Count, First, Any, Sum             |
|  Buffering: OrderBy, GroupBy, Distinct, Reverse, ToLookup        |
+------------------------------------------------------------------+
