# Query Optimization

---

## What is Query Optimization?

**Query optimization** is the process of tuning SQL queries to minimize resource consumption (CPU, I/O, memory) and reduce response time. In production systems, poorly optimized queries are the most common cause of performance degradation — a single bad query can saturate database resources. Optimization requires understanding of **query plans**, **indexing**, **statistics**, and **database internals**. In Spring Boot applications, Hibernate-generated queries must also be inspected and tuned.

### Cost-Based Optimization (CBO)

- **Cost units** — The optimizer measures cost in I/O page reads, CPU time, and memory usage to compare different execution strategies.
- **Statistics** — The optimizer relies on table row counts, column distinct values, data distribution histograms, and NULL density to estimate costs.
- **Cardinality estimation** — The optimizer estimates how many rows each operation will return, which drives join order and algorithm selection.
- **Access paths** — The optimizer chooses between sequential scans, index scans, bitmap scans, and index-only scans based on estimated costs.

### Query Optimization Phases

```
Query → Parser → Logical Rewrite → Cost-Based Optimization → Plan Execution
```

- **Logical Rewrite** — The optimizer applies transformations like predicate pushdown, view expansion, subquery unnesting, and constant folding.
- **Physical Optimization** — The optimizer chooses specific algorithms: join algorithm, access method, and join order to minimize total cost.

### Key Principles

- **Reduce data accessed** — Filter rows as early as possible in the query and project only the columns actually needed.
- **Reduce data volume** — Aggregate early and use LIMIT to minimize the amount of data processed by downstream operations.
- **Efficient access paths** — Use indexes to avoid full table scans, especially for selective filters on large tables.
- **Efficient joins** — Choose the right join order and join algorithm based on data sizes and available indexes.
- **Minimize round trips** — Batch operations and use JOIN FETCH to avoid N+1 query problems in ORM applications.

### Response Time Components

```
Response Time = Network Time + Query Execution Time
Query Execution Time = Parse Time + Plan Time + Execute Time
Execute Time = I/O Wait + CPU Time + Lock Wait
```

---

## Core Concepts

### Identifying Slow Queries in Spring Boot

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 100
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.stat: DEBUG
```

### Optimizing JPA Queries

```java
// BAD: fetches all columns, no pagination
@Query("SELECT u FROM User u")
List<User> findAllUsers();

// GOOD: fetch only needed columns with projection
public interface UserSummary {
    Long getId();
    String getUsername();
    String getEmail();
}

@Query("SELECT u.id AS id, u.username AS username, u.email AS email FROM User u")
List<UserSummary> findAllUserSummaries();

// GOOD: pagination
@Query("SELECT u FROM User u")
Page<User> findAllUsersWithPagination(Pageable pageable);
```

### Large Dataset Processing

```java
// Streaming with JPA
@Query("SELECT u FROM User u")
Stream<User> streamAllUsers();

@Transactional(readOnly = true)
public void processLargeDataset() {
    try (Stream<User> users = userRepository.streamAllUsers()) {
        users.forEach(this::processUser);
    }
}

// Or use cursors with fetch size hint
@QueryHints(@QueryHint(name = HibernateHints.HINT_FETCH_SIZE, value = "100"))
@Query("SELECT u FROM User u")
Stream<User> streamAllUsersWithBatchSize();
```

### Bulk Updates

```java
// BAD: updates each row individually
List<User> users = userRepository.findByStatus("PENDING");
users.forEach(u -> { u.setStatus("PROCESSED"); userRepository.save(u); });

// GOOD: bulk update
@Modifying
@Query("UPDATE User u SET u.status = 'PROCESSED' WHERE u.status = 'PENDING'")
int bulkMarkProcessed();
```

### Pagination: Offset vs Keyset

```sql
-- OFFSET pagination (slow for large offsets)
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100000;

-- Keyset pagination (cursor-based, fast)
SELECT * FROM orders
WHERE (created_at, id) < ('2024-01-01', 1000)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

```java
// Keyset pagination in Spring Data
@Query("SELECT o FROM Order o WHERE o.createdAt < :cursor ORDER BY o.createdAt DESC")
List<Order> findOrdersBefore(@Param("cursor") LocalDateTime cursor, Pageable pageable);
```

### Eager vs Lazy Loading Tuning

```java
@Entity
public class Author {
    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private List<Book> books;
}

// Solution 1: Join fetch
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();

// Solution 2: Entity Graph
@EntityGraph(attributePaths = {"books"})
@Query("SELECT a FROM Author a")
List<Author> findAllWithBooksGraph();

// Solution 3: Batch fetching
@Entity
@BatchSize(size = 25)
public class Author { ... }
```

### EXPLAIN ANALYZE Metrics

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING) SELECT * FROM orders WHERE status = 'PENDING';
```

Key metrics to check:
- **actual rows vs estimated rows** — A cardinality mismatch greater than 10x indicates stale statistics.
- **rows removed by filter** — High numbers indicate inefficient filtering and a need for better index design.
- **buffers: shared hit/read** — The cache hit ratio should be above 99% for well-cached working sets.
- **Sort Method: external merge** — Indicates `work_mem` is too small and the sort spilled to disk.

### Optimization Target Table

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| High CPU | Missing index, large sort, heavy aggregations | Add index, reduce sort columns |
| High I/O | Full table scans, no index | Add covering index |
| High lock wait | Long transactions, contention | Reduce transaction scope |
| Slow first page | Missing LIMIT, unoptimized join | Add LIMIT, optimize join |
| Slow later pages | OFFSET pagination | Use keyset pagination |
| N+1 queries | Lazy loading in loops | Use JOIN FETCH or batch size |

### Read-Write Splitting

```java
@Configuration
public class RoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
            ? "replica" : "primary";
    }
}
```

---

## Common Mistakes

- **Premature optimization without measurement** — Always profile before optimizing; guessing the bottleneck wastes effort and often makes things worse.
- **Ignoring database statistics** — Missing or stale ANALYZE leads to bad execution plans based on incorrect cardinality estimates.
- **Using functions on indexed columns in WHERE** — Wrapping indexed columns in functions like `YEAR(date)` prevents index usage entirely.
- **Assuming join order in query matches execution order** — The optimizer freely reorders joins for performance; the query's FROM clause order is not the execution order.
- **Over-indexing** — Every additional index slows down INSERT, UPDATE, and DELETE operations on the table.
- **Not using EXPLAIN ANALYZE** — Guessing instead of measuring leads to wasted optimization effort on the wrong problems.
- **Fetching too much data** — Not using LIMIT and selecting unnecessary columns increases I/O, network transfer, and memory usage.
- **N+1 queries from ORM** — The most common Hibernate performance issue, caused by lazy loading in loops instead of using JOIN FETCH.
- **Not tuning connection pool** — Too small causes contention; too large causes resource exhaustion and database overload.
- **Relying on nested loop joins for large datasets** — A hash join or merge join is usually far more efficient for large result sets.

---

## Real-World Scenarios

### N+1 Query Detection in Spring Boot

A Spring Boot app loads 20 orders and their customers. With lazy loading, Hibernate executes 1 query for orders plus 20 queries for customers. `JOIN FETCH` reduces this to a single query:

```java
// BAD: N+1 queries
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    order.getCustomer().getName(); // Triggers query per order
}

// GOOD: Single query with JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();
```

### Keyset Pagination for Deep Pages

A social media feed uses `OFFSET 100000 LIMIT 20`. The database reads 100,020 rows and discards 100,000. Keyset pagination reads exactly 20 rows:

```sql
-- Slow: reads 100K + 20 rows
SELECT * FROM posts ORDER BY id LIMIT 20 OFFSET 100000;

-- Fast: reads exactly 20 rows
SELECT * FROM posts WHERE id > :last_seen_id ORDER BY id LIMIT 20;
```

### Bulk Update vs Row-by-Row

A batch job marks 10,000 invoices as processed. The naive approach loads all entities and updates individually. The optimized approach uses a single JPQL UPDATE:

```java
// BAD: 10K SELECT + 10K UPDATE
List<Invoice> invoices = invoiceRepo.findByStatus("PENDING");
invoices.forEach(inv -> { inv.setStatus("PROCESSED"); });
invoiceRepo.saveAll(invoices);

// GOOD: single UPDATE
@Modifying
@Query("UPDATE Invoice i SET i.status = 'PROCESSED' WHERE i.status = 'PENDING'")
int bulkMarkProcessed();
```

## Scenario-Based Questions

- **Q: You are debugging a Spring Boot endpoint that loads 50 articles. Each article has an author and comments. The endpoint makes 102 SQL queries. How do you identify and fix the N+1 queries?** A: Enable `hibernate.generate_statistics=true` and check logs. Fix with `JOIN FETCH` for immediate relationships and `@EntityGraph` for nested graphs: `@EntityGraph(attributePaths = {"author", "comments"})`. For deeply nested graphs, use `@BatchSize`.
- **Q: A query filtering by `WHERE YEAR(created_at) = 2024` is slow despite an index on `created_at`. The table has 50M rows. How do you fix it without changing application code?** A: Create a functional index: `CREATE INDEX idx_orders_year ON orders((EXTRACT(YEAR FROM created_at)))`. However, rewriting to `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'` is better — sargable predicates allow range scans on the existing index.
- **Q: A paginated query with `ORDER BY created_at DESC LIMIT 20 OFFSET 100000` on a 10M row table takes 5 seconds. Each page gets slower. You cannot change the UI. How do you fix this?** A: Switch to keyset pagination internally while keeping the same API. Map `page=100` to `WHERE created_at < :cursor`. Return `created_at` as the cursor in the response. If arbitrary page jumps are required, use a covering index on `(created_at) INCLUDE (needed_columns)`.
- **Q: EXPLAIN ANALYZE shows `actual rows: 50000, plan rows: 1000` for an index scan. The query jumped from 50ms to 5s after a deployment. What caused this?** A: Stale statistics — a bulk data load changed distribution without ANALYZE. The optimizer underestimated rows and chose a plan optimized for 1000 rows. Run `ANALYZE` and increase autoanalyze frequency for tables with rapid changes.
- **Q: A JOIN between orders (10M rows) and customers (5M rows) uses a nested loop with 10M iterations. CPU is at 100% and the query takes 30 seconds. How do you make it use a hash join?** A: Run ANALYZE on both tables first. If the optimizer still chooses nested loop, `work_mem` may be too low for the hash table. Increase it, or temporarily force with `enable_nestloop = off` to confirm hash join is faster.
- **Q: `SELECT * FROM products WHERE description LIKE '%organic%'` on a 5M row table takes 20 seconds. The `description` column has a B-Tree index. Why doesn't the index help?** A: B-Tree indexes only support prefix search (`LIKE 'organic%'`). Wildcards at the start prevent index usage. Install `pg_trgm` and create a GIN trigram index: `CREATE INDEX idx_description_trgm ON products USING GIN (description gin_trgm_ops)`.
- **Q: A Spring Boot app logs 500ms for a method calling `findById` 50 times in a loop. Each `findById` is 10ms. How do you reduce total to under 50ms?** A: The 50 calls are 50 SQL queries with network round trips. Use `findAllById` generating a single `WHERE id IN (...)` query. The single query is efficient with an indexed primary key.
- **Q: A query joining 4 tables with WHERE, GROUP BY, and ORDER BY takes 20 seconds. The plan shows a sort spilling to disk (50MB). How do you fix it?** A: Increase `work_mem` to 64MB for this session. Better: create a composite index on the GROUP BY and ORDER BY columns to avoid the sort entirely. Check if the GROUP BY columns are covered by an existing index.
- **Q: Your team uses `@Query("SELECT u FROM User u")` and filters in Java streams. The `users` table has 2M rows. The app runs out of memory. What's wrong?** A: The query loads all 2M users into memory. Use database-side filtering with WHERE. If full processing is needed, use streaming: `Stream<User>` with `@QueryHint(name = "org.hibernate.fetchSize", value = "100")` and process in a try-with-resources block.
- **Q: A query that was fast yesterday (50ms) is slow today (5s). No code or schema changes. The plan is different. What do you check?** A: Plan regression — the optimizer chose a different plan due to changed statistics. Check `pg_stat_statements` for plan changes. Run EXPLAIN ANALYZE and compare to baseline. Fix: run ANALYZE, increase `default_statistics_target`, use `pg_hint_plan` to pin the good plan, or create extended statistics.

## Interview Questions

- **What is query optimization?** A: Tuning SQL queries to minimize resource consumption (CPU, I/O, memory) and reduce response time. Requires understanding query plans, indexing, statistics, and database internals.
- **What is the N+1 query problem in ORMs?** A: Loading N entities with one query, then N additional queries for related entities. Fix with JOIN FETCH, EntityGraph, or batch fetching.
- **What is the difference between OFFSET and keyset pagination?** A: OFFSET reads all skipped rows internally — O(n) per page. Keyset uses `WHERE id > :cursor` — O(1) per page. Keyset cannot jump to arbitrary pages.
- **What does EXPLAIN ANALYZE show and why is it useful?** A: Executes the query and shows actual times, row counts, and buffer usage. Reveals cardinality errors, disk spills, and inefficient operations.
- **What is a covering index?** A: Contains all columns needed by a query, enabling index-only scans without heap access. Uses `INCLUDE` for non-key columns. Reduces I/O significantly.
- **When should you use a bulk UPDATE?** A: When updating many rows with the same condition. A single UPDATE is orders of magnitude faster than row-by-row entity updates. Use `@Modifying @Query` in JPA.
- **What is work_mem in PostgreSQL?** A: Controls memory for sort operations, hash tables, and bitmap operations. Too low causes disk spills (external merge sort), which is 100x slower than in-memory.
- **How do you detect slow queries in Spring Boot?** A: Enable `hibernate.generate_statistics=true`, set `logging.level.org.hibernate.SQL=DEBUG`, and `LOG_QUERIES_SLOWER_THAN_MS=100`. On the DB side, query `pg_stat_statements`.
- **What causes cardinality estimation errors?** A: Stale statistics, correlated columns (optimizer assumes independence), low `default_statistics_target`, and uneven data distribution. Fix with ANALYZE or extended statistics.
- **Why is SELECT * bad for performance?** A: Reads all columns, increasing I/O, network, and memory. Prevents index-only scans. If schema changes, may return unexpected columns. Always select only needed columns.

## Developer Recommendations

- **Always profile before optimizing** — Guessing the bottleneck wastes effort. Use `EXPLAIN ANALYZE`, `pg_stat_statements`, and Hibernate statistics to identify actual slow queries.
- **Use JOIN FETCH to solve N+1 queries in JPA** — The most common Hibernate performance issue. `JOIN FETCH` loads related entities in the same query. For complex graphs, use `@EntityGraph`.
- **Prefer keyset pagination over OFFSET for deep pages** — OFFSET is O(n) and degrades with page number. Keyset is O(1). Trade-off: loses ability to jump to arbitrary pages.
- **Replace functions on indexed columns with sargable predicates** — `WHERE YEAR(date) = 2024` prevents index usage. Use `WHERE date >= '2024-01-01' AND date < '2025-01-01'`.
- **Use streaming for large result sets** — Loading 1M rows into memory causes OOM. Use `Stream<T>` with fetch size hint and process in a try-with-resources block.
- **Maintain a query performance baseline** — Compare current EXPLAIN ANALYZE against saved baselines. Plan regression is a common cause of sudden degradation.
- **Increase work_mem for queries with large sorts or hash joins** — Disk spills are 100x slower. Monitor for `Sort Method: external merge` or `Hash Join: spill` in execution plans.
