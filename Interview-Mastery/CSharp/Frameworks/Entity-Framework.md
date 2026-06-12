# Entity Framework

---

## Overview

- **Definition:** Entity Framework Core (EF Core) is Microsoft's open-source, cross-platform object-relational mapper (ORM) for .NET. It enables working with databases using .NET objects, eliminating most data-access plumbing code.
- **Why It Exists:** Provides LINQ-to-SQL translation, change tracking, migrations, lazy loading, and multiple database provider support (SQL Server, PostgreSQL, SQLite, Cosmos DB). Raises the abstraction level from tables/rows to domain objects with relationships.
- **Key Concepts:** **`DbContext`** (unit of work / repository), **`DbSet<T>`** (table collection), **change tracker** (tracks entity states: Added, Modified, Deleted, Unchanged), **migrations** (code-based schema evolution), **LINQ translation** (IQueryable → expression trees → SQL), **global query filters** (soft delete, multi-tenancy), and **value conversions**.

---

## Core Concepts

- **Architecture Layers:** Application code → LINQ queries → EF Core runtime (expression → SQL translation) → Database provider (ADO.NET) → Database server. Each layer adds abstraction but also overhead (~50-500µs per query).
- **Entity States:** `Detached → Added → Unchanged → Modified → Deleted`. `Detached` (not tracked), `Added` (will INSERT), `Unchanged` (no changes), `Modified` (UPDATE), `Deleted` (DELETE). `DetectChanges` compares current vs original values on `SaveChanges`.
- **Query Compilation and Caching:** EF Core compiles LINQ queries into an internal representation, then caches the compiled plan (keyed on expression tree structure). First execution: expression → SQL translation + parameter extraction. Subsequent executions: cache lookup. Queries with different parameter values reuse the same cached plan.
- **Change Tracking Internals:** Each tracked entity has an `InternalEntityEntry` with `OriginalValues` (snapshot at load), `CurrentValues` (current), entity state, key info, and navigation references. `DetectChanges` compares snapshots to detect modifications.

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

- **LINQ Translation Pipeline:** Pre-processing (simplify expressions) → Convert to query model (SelectExpression) → Apply null semantics, parameterize constants → Translate to provider-specific SQL → Compose with client-evaluation markers → Execute via `DbCommand`. Providers translate into `SELECT`, `JOIN`, `WHERE`, etc.
- **Lazy Loading Proxies:** Requires `Microsoft.EntityFrameworkCore.Proxies` + `UseLazyLoadingProxies()`. Navigation properties must be `virtual`. Castle.Core DynamicProxy creates a proxy subclass that intercepts navigation property getters to issue DB queries. Causes N+1 if used in loops.

```csharp
// Entity configuration with Fluent API
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products", "inventory");
        builder.HasKey(p => p.Id);
        builder.Property(p => p.Name).HasMaxLength(200).IsRequired();
        builder.HasOne(p => p.Category).WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId).OnDelete(DeleteBehavior.Restrict);
        builder.HasIndex(p => p.Name).IsUnique();
    }
}
```

- **Compiled Queries:** `EF.CompileQuery((AppDbContext ctx, int id) => ctx.Products.AsNoTracking().FirstOrDefault(p => p.Id == id))` — avoids expression parsing and cache lookup overhead for hot paths. Best for queries executed very frequently.

---

## Common Mistakes

- **Context lifecycle too long (Captive context)** — DbContext should be scoped per request in ASP.NET Core. Long-lived contexts grow the change tracker, consuming memory and returning stale data.
  - **Why it looks correct:** the DbContext is injected via DI and appears to work indefinitely, but the change tracker accumulates every loaded entity as a tracked snapshot — after processing 10K records the context holds references to all of them, and `SaveChanges` becomes increasingly slow as `DetectChanges` iterates over every tracked entity.
- **Missing async** — `_context.Products.ToList()` blocks the thread. Use `ToListAsync()` in async contexts.
  - **Why it looks correct:** the code compiles and returns the expected results, but the synchronous call blocks the ASP.NET thread pool thread for the duration of the database query — under load this causes thread pool starvation and cascading latency spikes across all requests.
- **Tracking overhead for read-only queries** — Default tracking creates snapshots for every loaded entity. Use `.AsNoTracking()` for reads.
  - **Why it looks correct:** the query returns correct data either way, but tracking stores an original-value snapshot for each row in a dictionary — for a 10K-row result set this adds megabytes of memory and CPU overhead from snapshot comparison on `SaveChanges` even though no changes were made.
- **Including too much (Cartesian explosion)** — Multiple `Include` + `ThenInclude` on a single query generates JOINs that multiply rows. Use `.AsSplitQuery()` to issue multiple queries instead.
  - **Why it looks correct:** `Include` is the standard way to load related data and the code reads naturally, but each collection `Include` adds a JOIN that multiplies the row count — three collection includes can turn 100 orders into 100 × 5 items × 3 shipments × 2 payments = 3,000 rows, and the database sends all that duplicated data over the wire.
- **Client-side evaluation of WHERE clause** — Calling `.ToList()` before `Where()` pulls all data into memory before filtering. Apply filters to `IQueryable` before materialization.
  - **Why it looks correct:** the code compiles and returns the right results, but the entire table is transferred from the database before filtering happens in memory — for a table with 1M rows where only 100 match the filter, 999,900 unnecessary rows cross the network and are allocated as objects before being discarded.
- **N+1 via lazy loading** — Accessing navigation properties in a loop triggers one query per iteration. Disable lazy loading or use `Include` for eager loading.
  - **Why it looks correct:** `order.Customer.Name` is a simple property access that returns the expected value, but behind the scenes the lazy loading proxy intercepts the getter and issues a new SQL query — in a loop of 1,000 orders this produces 1,001 queries instead of 1, turning a 10ms operation into a 5-second one.
- **Disposing context before lazy load completes** — Accessing a navigation property after disposing the context throws `ObjectDisposedException`.
  - **Why it looks correct:** the entity object is still in scope and appears usable, but its navigation properties are proxied — accessing them requires an active `DbContext` to issue the lazy load query, and after disposal the proxy throws instead of returning data.

```csharp
// N+1 queries — each order.Customer triggers a DB query
foreach (var order in orders) Console.WriteLine(order.Customer.Name);
// Fix: eager load with Include
var orders = await _context.Orders.Include(o => o.Customer).ToListAsync();
```

---

## Key Design Considerations

- **Use compiled queries for hot paths** — `EF.CompileQuery` avoids expression tree parsing and cache lookup. Use for queries executed hundreds of times per second.
- **Batch operations with `ExecuteUpdate`/`ExecuteDelete` (EF Core 7+)** — Instead of loading entities, modifying them, and calling `SaveChanges`, issue a single SQL UPDATE/DELETE without loading data.

```csharp
await _context.Products.Where(p => p.CategoryId == categoryId)
    .ExecuteUpdateAsync(setter => setter.SetProperty(p => p.Price, p => p.Price * 1.1m));
```

- **Use raw SQL for complex queries** — LINQ translation has limits: window functions, recursive CTEs, full-text search, query hints. Use `FromSqlRaw` or `FromSqlInterpolated` (safe) for these scenarios.
- **Understand query split behavior** — `AsSplitQuery()` issues one query per `Include` collection, avoiding Cartesian explosion but adding round trips. Measure to find the sweet spot.
- **Design DbContext for testability** — Use `IDbContextFactory<T>` for Blazor and worker services. Avoid injecting DbContext directly into domain objects.
- **Consider compiled models** — `dotnet ef dbcontext optimize` generates code for known queries, reducing startup time significantly in large models.
- **Use `HasConversion` for value objects** — Map domain value objects (Email, Money) to database columns cleanly.

```csharp
modelBuilder.Entity<User>()
    .Property(u => u.Email)
    .HasConversion(v => v.Value, v => Email.Create(v));
```

---

## Real-World Scenarios

### Scenario 1: E-Commerce Order Processing with Concurrency Handling
**Context:** An e-commerce system processes 1000 orders/minute. Inventory levels must be decremented atomically, and concurrent orders for the last item must be handled correctly.

```csharp
public class OrderService
{
    private readonly AppDbContext _context;

    public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
    {
        // Use serializable isolation for inventory operations
        await using var transaction = await _context.Database
            .BeginTransactionAsync(IsolationLevel.Serializable);

        try
        {
            var product = await _context.Products
                .FirstOrDefaultAsync(p => p.Id == request.ProductId);

            if (product == null || product.Stock < request.Quantity)
                return OrderResult.Failed("Insufficient stock");

            product.Stock -= request.Quantity;

            var order = new Order
            {
                ProductId = request.ProductId,
                Quantity = request.Quantity,
                TotalAmount = product.Price * request.Quantity,
                OrderDate = DateTime.UtcNow
            };
            _context.Orders.Add(order);

            // Optimistic concurrency: Product has [Timestamp] byte[] Version
            try
            {
                await _context.SaveChangesAsync();
                await transaction.CommitAsync();
                return OrderResult.Success(order.Id);
            }
            catch (DbUpdateConcurrencyException ex)
            {
                await transaction.RollbackAsync();
                // Retry logic: reload product and re-check stock
                var entry = ex.Entries.Single();
                await entry.ReloadAsync();
                return await PlaceOrderAsync(request); // Retry with fresh data
            }
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### Scenario 2: Multi-Tenant SaaS with Soft Delete and Query Filters
**Context:** A SaaS application stores data for 500 tenants in a shared database. Every query must filter by tenant and exclude soft-deleted records automatically.

```csharp
public class TenantDbContext : DbContext
{
    private readonly ITenantProvider _tenantProvider;

    public TenantDbContext(DbContextOptions<TenantDbContext> options, ITenantProvider tenantProvider)
        : base(options) => _tenantProvider = tenantProvider;

    public DbSet<Document> Documents => Set<Document>();
    public DbSet<User> Users => Set<User>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Global query filters: applied to EVERY query
        modelBuilder.Entity<Document>().HasQueryFilter(d =>
            d.TenantId == _tenantProvider.TenantId && !d.IsDeleted);
        modelBuilder.Entity<User>().HasQueryFilter(u =>
            u.TenantId == _tenantProvider.TenantId && !u.IsDeleted);

        // Soft delete configuration
        modelBuilder.Entity<Document>().Property(d => d.IsDeleted).HasDefaultValue(false);
        modelBuilder.Entity<Document>().HasIndex(d => new { d.TenantId, d.IsDeleted });
    }

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Automatically set soft delete on any entity being deleted
        foreach (var entry in ChangeTracker.Entries<ISoftDeletable>())
        {
            if (entry.State == EntityState.Deleted)
            {
                entry.State = EntityState.Modified;
                entry.Entity.IsDeleted = true;
                entry.Entity.DeletedAt = DateTime.UtcNow;
            }
        }
        return base.SaveChangesAsync(ct);
    }
}

// Admin override: ignore filters
public async Task<List<Document>> GetDeletedDocumentsAsync(int tenantId)
{
    return await _context.Documents
        .IgnoreQueryFilters()
        .Where(d => d.TenantId == tenantId && d.IsDeleted)
        .ToListAsync();
}
```

### Scenario 3: Reporting with Split Queries and Compiled Queries
**Context:** A dashboard loads orders with items and payments. The query is executed 50 times/second. Must avoid Cartesian explosion and maximize query performance.

```csharp
// Compiled query for hot path — avoids expression tree parsing
private static readonly Func<AppDbContext, int, Task<Order?>> GetOrderById =
    EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
        ctx.Orders
            .AsNoTrackingWithIdentityResolution()
            .Include(o => o.Items)
            .Include(o => o.Payments)
            .AsSplitQuery()
            .FirstOrDefault(o => o.Id == id));

public class DashboardService
{
    public async Task<OrderDetail> GetOrderDetailAsync(int orderId)
    {
        // Use compiled query for max performance
        var order = await GetOrderById(_context, orderId);
        if (order == null) return null;

        // After loading, project with Select for minimal data transfer
        return new OrderDetail
        {
            OrderNumber = order.OrderNumber,
            Total = order.Items.Sum(i => i.Quantity * i.UnitPrice),
            ItemCount = order.Items.Count,
            PaymentStatus = order.Payments.Any(p => p.Status == "Completed") ? "Paid" : "Pending"
        };
    }
}
```

---

## Scenario-Based Questions

1. **Q: You have a query that loads orders with their items and shipments. Response time is 30s for 1000 orders. The SQL generated has a massive Cartesian product. How do you fix it?**
   - **A:** This is the Cartesian explosion problem — multiple `Include` calls on collection navigations generate JOINs that multiply rows (1000 orders × 5 items × 2 shipments = 10K rows, but the JOIN multiplies to 10K × ...). Fix: use `AsSplitQuery()` to issue one query per collection navigation:
```csharp
var orders = await _context.Orders
    .Include(o => o.Items)
    .Include(o => o.Shipments)
    .AsSplitQuery()
    .ToListAsync();
```
This generates 3 queries: one for orders, one for items (JOIN orders), one for shipments (JOIN orders). Trade-off: more round trips but no row multiplication. For even better performance, use `Select` projections to pick only needed columns.

2. **Q: You are troubleshooting a production issue where `SaveChangesAsync` takes 5 seconds. The change tracker has 10K tracked entities. What's happening?**
   - **A:** `DetectChanges` is called automatically before `SaveChanges`. With 10K tracked entities, it iterates each entity and compares all property values against their snapshots — O(n * properties). For bulk operations, disable auto-detection: `_context.ChangeTracker.AutoDetectChangesEnabled = false` and call `DetectChanges()` manually at strategic points. Also consider: if you're loading entities only to update a single property, use `ExecuteUpdate` (EF Core 7+) instead — it issues a single SQL UPDATE without loading data.
   - **Interview follow-up:** If you disable `AutoDetectChangesEnabled`, which EF Core operations still implicitly call `DetectChanges` and might surprise you with inconsistent tracked state?

3. **Q: You are designing a multi-tenant system where each tenant's data must be isolated. You choose a shared database approach. How do you prevent accidentally querying another tenant's data?**
   - **A:** Use global query filters: `modelBuilder.Entity<T>().HasQueryFilter(e => e.TenantId == _tenantProvider.TenantId)`. This adds `WHERE TenantId = @__tenantProvider_TenantId_0` to EVERY query automatically. The tenant ID is resolved via `IHttpContextAccessor` or scoped service injected into `DbContext`. For truly airtight isolation, also use schema-per-tenant (each tenant gets its own schema). Global query filters can be bypassed with `IgnoreQueryFilters()` — only expose this on dedicated admin endpoints with authorization checks.

4. **Q: You are migrating from EF6 to EF Core. Your old system relied on lazy loading extensively, and performance is terrible. How do you refactor?**
   - **A:** Disable lazy loading by default (`optionsBuilder.UseLazyLoadingProxies(false)`). Use eager loading (`Include`/`ThenInclude`) for all navigation properties you know you'll access. For optional relationships, use explicit loading: `await context.Entry(order).Reference(o => o.Customer).LoadAsync()`. Apply the N+1 detection pattern: log or throw when lazy loading is triggered (EF Core 6+ can warn on unfiltered `Include`). Then systematically replace lazy loads with eager loads in hot paths. For truly optional navigation access, consider a `LazyLoader` that batches loads.

5. **Q: You need to update 50K products' prices by 10%. The naive approach loads all entities, modifies them, and calls `SaveChanges`. This takes 2 minutes. How do you make it instantaneous?**
   - **A:** Use `ExecuteUpdate` (EF Core 7+):
```csharp
await _context.Products
    .Where(p => p.CategoryId == categoryId)
    .ExecuteUpdateAsync(setter => setter.SetProperty(p => p.Price, p => p.Price * 1.1m));
```
This issues a single SQL `UPDATE` statement — no data is loaded into memory. For deletes, use `ExecuteDelete`. Both bypass the change tracker entirely. For complex transformations, use raw SQL with `ExecuteSqlRaw` or `ExecuteSqlInterpolated`.

6. **Q: You are building a search endpoint that returns products with 12 optional filters. The LINQ query becomes a complex expression tree. How do you structure the query for maintainability and performance?**
   - **A:** Use the specification pattern or conditional `IQueryable` composition:
```csharp
IQueryable<Product> query = _context.Products.AsNoTracking();
if (request.MinPrice.HasValue) query = query.Where(p => p.Price >= request.MinPrice);
if (request.MaxPrice.HasValue) query = query.Where(p => p.Price <= request.MaxPrice);
if (!string.IsNullOrEmpty(request.Category)) query = query.Where(p => p.Category.Name == request.Category);
if (request.InStockOnly) query = query.Where(p => p.Stock > 0);
// ... more filters
query = query.OrderBy(p => p.Price).Skip(request.Page * request.Size).Take(request.Size);
var products = await query.Select(p => new ProductDto { ... }).ToListAsync();
```
This builds a single SQL query with only the relevant WHERE clauses. Each `.Where()` composes into the expression tree without any branching in SQL. Use AutoMapper's `ProjectTo` to generate efficient `SELECT` projections automatically.

7. **Q: You are experiencing deadlocks under high concurrency. Two transactions both read and update the same rows. How do you resolve this with EF Core?**
   - **A:** Deadlocks often occur when transactions acquire locks in different orders. Fix: (1) Ensure consistent access order — always update entities in the same sequence (e.g., by primary key). (2) Use `IsolationLevel.ReadCommitted` (default) — higher isolation levels like `Serializable` increase deadlock probability. (3) Use short-lived transactions — minimize work between `BeginTransaction` and `Commit`. (4) Use `SqlRetryExecutionStrategy` for automatic retry on deadlock (EF Core's default strategy retries on SQL Server deadlock error 1205):
```csharp
optionsBuilder.UseSqlServer(connStr, o => o.EnableRetryOnFailure(3));
```
(5) For high-contention counters (e.g., inventory), consider optimistic concurrency with retry instead of pessimistic locks.
   - **Interview follow-up:** If `EnableRetryOnFailure` retries the entire transaction on deadlock, how does it handle side effects from statements that already executed before the deadlock — for example, an `INSERT` that succeeded before the conflicting `UPDATE`?

8. **Q: You need to log every query EF Core executes for debugging. Some queries are generated inefficiently. How do you capture and analyze them?**
   - **A:** In development, enable logging: `optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information)`. For detailed analysis, use `ToQueryString()` on any `IQueryable` to see the generated SQL:
```csharp
var query = _context.Products.Where(p => p.Price > 100).OrderBy(p => p.Name);
var sql = query.ToQueryString(); // "SELECT ... FROM Products WHERE Price > @__p_0 ORDER BY Name"
```
For production, use an interceptor: `optionsBuilder.AddInterceptors(new TaggedQueryCommandInterceptor())`. EF Core 8+ has built-in query tags: `.TagWith("MyQuery")` adds comments to SQL for identification in database logs/profilers.

9. **Q: You are designing a Blazor Server app that holds DbContext open for the lifetime of the user's session. The change tracker grows to 100K entities. How do you fix this?**
   - **A:** Blazor Server should use `IDbContextFactory<T>` to create short-lived DbContext instances per operation, not a single long-lived one:
```csharp
services.AddDbContextFactory<AppDbContext>(options => ...);
// In component:
await using var context = await _factory.CreateDbContextAsync();
var products = await context.Products.AsNoTracking().ToListAsync();
```
If you must track entities across renders, use detached entities (no tracking) with manual state management. Alternatively, use `AsNoTrackingWithIdentityResolution()` to avoid tracking overhead while still resolving entity identity.

10. **Q: You are implementing the transactional outbox pattern. The background worker publishes outbox messages but sometimes duplicates occur after a crash. How do you handle idempotency?**
     - **A:** Each outbox message should have a unique, deterministic `MessageId` (e.g., `$"{aggregateType}-{aggregateId}-{eventSequence}"`). The message broker (or consumer) deduplicates by this ID. Use `IOutboxStore` with `ProcessedMessages` tracking:
```csharp
public class OutboxProcessor
{
    public async Task ProcessOutboxAsync(CancellationToken ct)
    {
        var messages = await _context.OutboxMessages
            .Where(m => !m.ProcessedAt.HasValue)
            .OrderBy(m => m.CreatedAt)
            .Take(100)
            .ToListAsync(ct);

        foreach (var msg in messages)
        {
            await _messageBus.PublishAsync(msg.Type, msg.Payload, msg.MessageId);
            msg.ProcessedAt = DateTime.UtcNow;
            msg.Error = null;
        }
        await _context.SaveChangesAsync(ct);
    }
}
```
Use `MessageId` for idempotent publishing — the broker checks if it already processed this ID. For exactly-once delivery, combine with consumer-side idempotency (store processed message IDs in the consumer database).
    - **Interview follow-up:** If the service crashes between publishing the message and saving `ProcessedAt`, the next run re-publishes the same message — is the window between publish and `SaveChanges` acceptable for your system, and how would you narrow or eliminate it?

---

## Interview Questions

1. **What is EF Core?**
   - **A:** Entity Framework Core is Microsoft's open-source, cross-platform ORM for .NET. It enables working with databases using .NET objects, providing LINQ-to-SQL translation, change tracking, migrations, and multiple database provider support.

2. **What is the difference between `AsNoTracking` and the default tracking behavior?**
   - **A:** Default tracking creates entity snapshots for change detection on `SaveChanges`. `AsNoTracking` skips snapshot creation, making queries ~30-50% faster with less memory. Use `AsNoTracking` for read-only queries; use tracking when entities will be modified and saved.

3. **What is the N+1 query problem?**
   - **A:** Lazy loading triggers a separate query each time a navigation property is accessed. In a loop over N parent entities, this produces 1 (parent query) + N (child lazy loads) = N+1 queries. Prevent with eager loading (`Include`/`ThenInclude`), explicit loading, or disabling lazy loading.

4. **What is the difference between `Include` and `ThenInclude`?**
   - **A:** `Include` specifies the first-level navigation property to eager load: `.Include(o => o.Customer)`. `ThenInclude` chains to load nested navigations: `.Include(o => o.Items).ThenInclude(i => i.Product)`. `ThenInclude` always follows an `Include` or another `ThenInclude`.

5. **What is `ExecuteUpdate` and when should you use it?**
   - **A:** Added in EF Core 7, it issues a single SQL `UPDATE` statement without loading entities into memory. Use for bulk updates: `_context.Products.Where(p => p.Price < 10).ExecuteUpdateAsync(setter => setter.SetProperty(p => p.Price, 10))`. Do not use when you need change tracking or validation.

6. **What are global query filters?**
   - **A:** LINQ WHERE predicates applied automatically to all queries for an entity. Configured in `OnModelCreating`: `modelBuilder.Entity<T>().HasQueryFilter(e => !e.IsDeleted)`. Used for soft deletes, multi-tenancy, and access control. Can be bypassed with `IgnoreQueryFilters()`.

7. **What is the difference between `FromSqlRaw` and `FromSqlInterpolated`?**
   - **A:** `FromSqlInterpolated` uses string interpolation syntax but converts parameters to `SqlParameter` — safe from injection. `FromSqlRaw` takes a raw SQL string — vulnerable to injection if concatenated with user input. Always prefer `FromSqlInterpolated` for dynamic values.

8. **What is `AsSplitQuery`?**
   - **A:** Instructs EF Core to issue separate queries for each collection `Include` instead of one large query with multiple JOINs. Avoids Cartesian explosion (row multiplication from multiple collections) at the cost of additional round trips. Use when including multiple collection navigations.

9. **What is the difference between TPH, TPT, and TPC inheritance mapping?**
   - **A:** **TPH** (default): single table with discriminator column — best performance, nullable subtype columns. **TPT**: one table per type with JOINs — normalized but slower. **TPC** (EF Core 8+): one table per concrete type — no discriminator, no JOINs for leaf types. TPH is usually the best choice.

10. **How does EF Core handle concurrency conflicts?**
    - **A:** Use `[Timestamp]` (row version) or `[ConcurrencyCheck]` attributes. EF Core includes the original value in the UPDATE/DELETE WHERE clause. If the value changed since load, no rows are affected and `DbUpdateConcurrencyException` is thrown. Handle by refreshing the entity and retrying, or notifying the user.

---

## Developer Recommendations

- **Use `AsNoTracking()` for all read-only queries** — Default tracking creates snapshots for every loaded entity, consuming memory and CPU. `AsNoTracking()` eliminates this overhead, making queries ~30-50% faster. Reserve tracking for entities that will be modified and saved. For read-only endpoints, `AsNoTrackingWithIdentityResolution()` offers a middle ground — no snapshots but fix-up for reference navigation identity. A reporting endpoint loading 50K products with default tracking once consumed 200MB of memory per request and caused frequent Gen 2 GC collections — switching to `AsNoTracking()` reduced memory to 5MB and eliminated the latency spikes.

- **Prefer `ExecuteUpdate`/`ExecuteDelete` over load-modify-save for bulk operations** — Loading 50K entities to update a single property wastes memory and bandwidth. `ExecuteUpdate` issues a single SQL UPDATE without loading data. Use for bulk operations, batch jobs, and administrative tasks. Reserve the load-modify-save pattern for operations requiring business logic, validation, or change tracking. A nightly batch job once loaded 100K orders to set a `ProcessedAt` timestamp, consuming 1.5GB of memory and taking 4 minutes — converting to `ExecuteUpdate` reduced it to a single SQL statement completing in under a second.

- **Avoid multiple `Include` on collection navigations without `AsSplitQuery()`** — Each collection `Include` adds a JOIN that multiplies rows, causing exponential data growth. A query with 3 collection `Include`s can return millions of rows from thousands of entities. Use `AsSplitQuery()` to issue one query per collection, or use `Select` projections to flatten only the needed data.

- **Use compiled queries for hot paths** — `EF.CompileAsyncQuery` avoids expression tree parsing and query plan cache lookup on every execution. For queries executed 50+ times/second, this can reduce per-query overhead by ~20-50µs. Compile once at startup, invoke repeatedly with parameters. Less beneficial for rare or unique queries.

- **Scope DbContext to the operation, not the application** — Long-lived DbContext instances accumulate tracked entities, consume memory, and return stale data. In ASP.NET Core, DbContext is scoped per request by default. In Blazor Server, use `IDbContextFactory<T>` to create short-lived instances per operation. Never store a DbContext in a singleton.

- **Use `Select` projections to minimize data transfer** — Instead of loading full entities with `Include`, project to DTOs with `Select`. This generates efficient SQL that retrieves only the needed columns and avoids loading unused navigations:
```csharp
var products = await _context.Products
    .Where(p => p.Price > 100)
    .Select(p => new ProductDto { Id = p.Id, Name = p.Name, Price = p.Price })
    .ToListAsync();
```
This generates `SELECT Id, Name, Price FROM Products WHERE Price > 100` — no JOINs, no unused data.

- **Use raw SQL for queries that LINQ can't translate efficiently** — LINQ has limits: window functions, recursive CTEs, full-text search, query hints, and complex aggregations. Use `FromSqlInterpolated` for these scenarios. Dapper pairs well here — use EF Core for domain operations, Dapper for complex queries (CQRS pattern).

---

## Performance Patterns

| Pattern | Queries | Data Transferred | Memory |
|---|---|---|---|
| NoTracking + Select | 1 | Minimal | Low |
| NoTracking | 1 | All columns | Medium |
| Tracking (default) | 1 | All columns | High (snapshot) |
| Lazy Loading (N+1) | 1+N | Per entity | Very High |
| Split Query (2 Includes) | 3 | Per entity set | Medium |
| Eager Loading (1 Include) | 1 (JOIN) | Duplicated rows | High (cart) |

## EF Core vs Dapper

| Feature | EF Core | Dapper |
|---|---|---|
| Type | Full ORM | Micro ORM |
| Query method | LINQ (IQueryable) | Raw SQL |
| Change tracking | Built-in | Manual |
| Migrations | Yes | No (manual) |
| Performance | Medium (~50-500µs) | Fast (~2-10µs) |
| SQL control | Limited (LINQ) | Full |
| Compiled queries | `EF.CompileQuery` | N/A (raw SQL) |
