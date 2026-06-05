# Database Indexing

---

## What is Indexing?

**Database indexes** are data structures that improve the speed of data retrieval at the cost of additional writes and storage. They act like a book's index — instead of scanning every page, you look up the term in the index and jump directly to the relevant pages. The default index type in most databases is the **B-Tree**, but other types like **Hash**, **GiST**, **GIN**, and **BRIN** serve specialized use cases.

### Key Concepts:

1. **How Indexes Work**:

   - An index is a copy of selected columns from a table, organized in a search-optimized structure (typically a B-Tree).
   - Without an index, the database performs a **sequential (full) table scan** — reading every row.
   - With an index, the database can locate rows using a **tree traversal (O(log n))** or **hash lookup (O(1))**, avoiding the full scan.

2. **B-Tree Index Structure**:

   B-Trees are balanced trees where:
   - **Root node** contains pivot values that guide search direction.
   - **Internal nodes** direct the search to the correct child node.
   - **Leaf nodes** contain the actual index entries plus a pointer to the heap row.
   - All leaf nodes are at the same depth (balanced property).
   - Height is typically 3-5 levels for billions of rows.

3. **Index Scan Types**:

   - **Index Scan (Index Seek)** — Traverse B-Tree to leaf, then fetch row from heap by pointer.
   - **Index-Only Scan** — All needed columns are in the index; no heap access required. Fastest option.
   - **Bitmap Index Scan** — Build a bitmap of matching page locations, then fetch heap pages in order. Reduces random I/O for multi-condition queries.
   - **Skip Scan** (PostgreSQL 15+) — Efficiently find distinct values when the leading column has few values and the trailing column is filtered.

4. **Composite Indexes**:

   Indexes on multiple columns follow the **leftmost prefix rule**. An index on `(A, B, C)` can be used for queries on:
   - `A` only (uses prefix)
   - `A AND B` (full match on prefix)
   - `A AND B AND C` (full match)
   - But NOT for `B` alone or `C` alone

   Column order matters: put **equality conditions first**, then **range conditions**, then **sort columns**.

5. **Other Index Types**:

   - **Hash Index** — O(1) lookup for equality only. No sorting, no range queries.
   - **GiST** — Extensible index for full-text search, geometric data, and range types.
   - **GIN** — Index for composite values like arrays, JSONB, and full-text search vectors.
   - **BRIN** — Tiny index for naturally ordered data (time-series, logs). Very space-efficient.
   - **Partial Index** — Indexes only a subset of rows (e.g., `WHERE status = 'ACTIVE'`). Saves space.
   - **Covering Index** — Includes extra columns via `INCLUDE` to enable index-only scans.
   - **Functional Index** — Index on an expression like `LOWER(email)`.

---

## Core Concepts

### 1. How Indexes Affect Write Operations

   Indexes speed up reads but slow down writes:
   - **INSERT** — New entry added to each index (O(log n) per index).
   - **UPDATE** — If indexed column changes, the index entry is moved (delete + insert).
   - **DELETE** — Index entry removed from each index.
   - **Page Splits** — When a B-Tree node is full, it splits into two — an expensive operation. Use `fillfactor` (e.g., `70`) to reserve free space for updates and reduce splits.

### 2. Index Selectivity

   - **High selectivity** (many unique values) — Index is very efficient, quickly narrows results.
   - **Low selectivity** (few unique values, like a boolean) — Index may not help for common values but helps for rare values. A **partial index** is ideal for low-selectivity columns.

   ```sql
   -- For a table where 99% of orders are 'COMPLETED' and 1% are 'PENDING':
   CREATE INDEX idx_orders_pending ON orders(id) WHERE status = 'PENDING';
   ```

### 3. JPA Index Declarations

   ```java
   @Entity
   @Table(name = "orders", indexes = {
       @Index(name = "idx_orders_status", columnList = "status"),
       @Index(name = "idx_orders_user_status", columnList = "user_id, status"),
       @Index(name = "idx_orders_created_at", columnList = "created_at DESC")
   })
   public class Order { ... }
   ```

### 4. Managing Indexes with Migrations

   Complex indexes (partial, functional, concurrent) belong in migration scripts:

   ```sql
   CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
   REINDEX INDEX CONCURRENTLY idx_orders_status;
   ```

   `CONCURRENTLY` avoids locking writes during index creation — essential for production.

---

## Common Mistakes

1. **No index on foreign key columns** — JOINs perform sequential scans instead of index seeks.

2. **Indexing every column** — Each index adds write overhead and storage cost. Only index columns used in WHERE, JOIN, and ORDER BY.

3. **Wrong column order in composite indexes** — The leading column does not match query patterns, making the index useless. Put equality columns first.

4. **Creating indexes during business hours without CONCURRENTLY** — Blocks writes on the table. Always use `CREATE INDEX CONCURRENTLY` in production.

5. **Over-indexing small tables** — Small tables (few hundred rows) are faster with sequential scans. Index overhead outweighs benefits.

6. **Assuming indexes work for `LIKE '%pattern'`** — B-Tree indexes only help for prefix patterns (`pattern%`). Use GIN with trigrams for suffix/containment patterns.

7. **Not analyzing after bulk data loads** — Stale statistics cause the optimizer to ignore valid indexes. Run `ANALYZE` after large data changes.

8. **Indexing boolean columns without a partial index** — Most rows have the same value, making the index rarely useful. Use `WHERE col = TRUE`.

---

## Real-World Scenarios

### 1. E-Commerce Product Search with Composite Indexes

A product catalog with 10M products. Users filter by `category_id` and sort by `price`. Without a composite index on `(category_id, price)`, the database filters then sorts in memory. With the index, the B-Tree stores rows sorted by price within each category:

```sql
CREATE INDEX idx_category_price ON products(category_id, price);

SELECT id, name, price FROM products
WHERE category_id = 42
ORDER BY price
LIMIT 20;
```

### 2. Time-Series Monitoring with BRIN Indexes

An IoT platform ingests 1B events/day into `sensor_readings`. Queries filter by `timestamp`. A B-Tree index takes 50GB. A BRIN index takes 50MB — 1000x smaller:

```sql
CREATE INDEX idx_readings_time ON sensor_readings USING BRIN(timestamp);

SELECT * FROM sensor_readings
WHERE timestamp >= '2024-06-01' AND timestamp < '2024-06-02';
```

### 3. Social Media Notifications with Partial Indexes

A notifications table has 100M rows, 99.9% are `read = TRUE`. A full index on `(user_id, created_at)` is 2GB. A partial index with `WHERE read = FALSE` is 2MB with identical performance:

```sql
CREATE INDEX idx_unread_notifications ON notifications(user_id, created_at) WHERE read = FALSE;

SELECT * FROM notifications WHERE read = FALSE AND user_id = 123 ORDER BY created_at DESC LIMIT 50;
```

## Scenario-Based Questions

1. **Q: You are building an e-commerce product listing where users filter by any combination of 8 optional attributes. How do you index without creating 256 indexes?**
   A: Use a composite index on the 2-3 most selective columns queried most often. For the rest, let PostgreSQL use bitmap scans combining multiple single-column indexes. Alternatively, use a GIN index on a JSONB column storing all attribute values, or create partial indexes per common filter combination (e.g., `WHERE category_id IS NOT NULL AND price IS NOT NULL`).

2. **Q: Your application's write throughput dropped 80% after adding 5 B-Tree indexes to a table receiving 10K writes/second. The indexes are necessary for reads. How do you fix it?**
   A: First, replace full indexes with partial indexes where possible (e.g., only index active records). Second, reduce `fillfactor` to 70-80 to minimize page splits. Third, batch writes to reduce per-row index maintenance overhead. Fourth, consider table partitioning so each partition has smaller indexes.

3. **Q: A 500M row orders table dashboard query groups by `status` and `region` aggregating `total`. It runs fine at 2AM but times out at 2PM with a different execution plan. How do you stabilize it?**
   A: Plan instability caused by changing statistics or parameter values. Create a covering index on `(status, region) INCLUDE (total)` for index-only scans. Increase `work_mem` to prevent disk spills for hash aggregates. Use a materialized view refreshed periodically for the dashboard query.

4. **Q: Your team uses UUID primary keys. Insert performance degrades beyond 10M rows. What is the root cause and how do you fix it?**
   A: Random UUIDs cause frequent B-Tree page splits — new rows insert at random positions. The tree becomes unbalanced and cache hit ratio drops. Fix by switching to UUID v7 (time-ordered), using ULIDs, or using a sequential `BIGSERIAL` as the clustered PK with UUID as a secondary unique index.

5. **Q: A `WHERE status IN ('PENDING', 'PROCESSING')` query scans the full index but performs poorly. Both values are common (~15% each). The index is on `(status, created_at)`. How do you optimize?**
   A: For IN lists with common values, the optimizer may choose a bitmap or seq scan. Create a partial index for rare statuses and let common statuses use a different path. Reorder to `(created_at, status)` if the query always uses a date range. Add a BRIN index on `created_at` combined with a separate status index.

6. **Q: `SELECT COUNT(*) FROM orders WHERE status = 'SHIPPED'` on a 200M row table takes 30 seconds despite having an index. How do you make it instant?**
   A: The index still visits all 200M matching entries to count them. Use a partial index: `CREATE INDEX idx_shipped ON orders(id) WHERE status = 'SHIPPED'` so `COUNT(*)` scans only matching entries. Alternatively, maintain a materialized view with pre-aggregated counts.

7. **Q: After `VACUUM FULL` on a heavily updated table, queries are slower than before. What happened?**
   A: `VACUUM FULL` rebuilds the table and indexes in physical order. If the original order matched query patterns (time-ordered inserts), the new order may increase random I/O. It also invalidates cached plans and statistics. Run `ANALYZE` afterward.

8. **Q: Your Spring Boot app generates `WHERE id IN (1, 2, ..., 50)`. Performance is fine for 50 IDs but degrades at 500 IDs. Why?**
   A: The planner may choose a seq scan over repeated index lookups when the IN list is large — random I/O for 500 index probes exceeds seq scan cost. Use a temporary table with JOIN for large lists, or batch in chunks of 50-100.

9. **Q: A migration adds a new index and application queries start blocking during creation. What went wrong?**
   A: The index was created without `CONCURRENTLY`. Plain `CREATE INDEX` acquires a lock blocking writes. Always use `CREATE INDEX CONCURRENTLY` in production. Monitor progress via `pg_stat_progress_create_index`.

10. **Q: A query joining `orders` (10M rows) and `customers` (5M rows) on `customer_id` uses a nested loop with 10M loops despite an index on `orders.customer_id`. Why is this slow and how do you fix it?**
    A: Even with an index, 10M random I/O operations are expensive. The optimizer chose nested loop expecting few rows. Ensure statistics are current. Force a hash join temporarily with `enable_nestloop = off` or increase `work_mem` to accommodate the hash table.

## Interview Questions

1. **What is a B-Tree index and how does it work?**
   A: A B-Tree is a balanced tree data structure where leaf nodes contain index entries pointing to heap rows. Search traverses from root to leaf in O(log n) time, supporting equality and range lookups.

2. **What is the difference between clustered and non-clustered indexes?**
   A: A clustered index determines physical row order on disk — a table can have only one. A non-clustered index is a separate structure with pointers to heap rows. InnoDB uses the PK as clustered; PostgreSQL uses heap with separate indexes.

3. **When would you use a GIN index over a B-Tree?**
   A: GIN indexes are for composite values like JSONB, arrays, and full-text search vectors. B-Trees cannot index the individual elements of an array or JSONB field efficiently.

4. **What is index selectivity and why does it matter?**
   A: Selectivity measures how many rows a query condition matches. High selectivity (few rows) makes indexes efficient. Low selectivity (many rows) may make a sequential scan cheaper. Partial indexes address low selectivity.

5. **What is the leftmost prefix rule?**
   A: For a composite index on (A, B, C), queries must reference column A (the leftmost) for the index to be used. The index supports A, A+B, and A+B+C but not B alone or C alone.

6. **When does PostgreSQL choose a bitmap scan over an index scan?**
   A: When multiple conditions match rows scattered across disk pages. A bitmap scan builds a page-level bitmap of matching locations, sorts them, then fetches pages in order — reducing random I/O.

7. **How do you detect and remove unused indexes?**
   A: Query `pg_stat_user_indexes` for indexes with `idx_scan = 0`. Drop unused indexes to reduce write overhead and storage. Use `DROP INDEX CONCURRENTLY` in production.

8. **How does CREATE INDEX CONCURRENTLY differ from regular CREATE INDEX?**
   A: `CREATE INDEX CONCURRENTLY` builds the index without locking writes, using three passes. It takes longer but avoids downtime. If it fails, the invalid index must be dropped before retrying.

9. **What is index bloat, what causes it, and how do you fix it?**
   A: Index bloat is wasted space from dead index entries left by UPDATE/DELETE operations. Fix with `REINDEX INDEX CONCURRENTLY`. Ensure autovacuum is tuned. Monitor with `pgstattuple`.

10. **How would you index a 1B row table with frequent GROUP BY queries on `department_id` aggregating `salary`?**
    A: Create a covering index on `(department_id) INCLUDE (salary)` for index-only scans. If `department_id` has low cardinality, consider partitioning by department.

## Developer Recommendations

- **Use composite indexes with equality-first column ordering** — An index on `(status, created_at)` supports `WHERE status = 'PENDING' ORDER BY created_at`. Placing range columns last prevents the index from being used for equality filters after the first range condition.

- **Prefer partial indexes for low-selectivity columns** — A full index on a boolean column indexes both true and false values. A partial index with `WHERE is_active = TRUE` is dramatically smaller and equally effective.

- **Monitor index usage via pg_stat_user_indexes** — Indexes with `idx_scan = 0` waste write throughput and storage with no read benefit. Drop them after testing.

- **Use covering indexes with INCLUDE for index-only scans** — Adding non-filtered columns via `INCLUDE` avoids heap fetches for frequent queries. The trade-off is larger index size but faster reads.

- **Set fillfactor on tables with frequent UPDATEs** — Default 100 fills pages completely, causing expensive page splits on updates. Set fillfactor to 70-80 to reserve space for in-place updates (HOT updates in PostgreSQL).

- **Create indexes CONCURRENTLY in production** — Plain `CREATE INDEX` blocks writes. `CREATE INDEX CONCURRENTLY` adds the index without downtime at the cost of slower creation and higher resource usage.

- **Consider BRIN over B-Tree for append-only data** — BRIN indexes are 100-1000x smaller than B-Tree for time-ordered data. They work best when physical page order correlates with the indexed column (e.g., timestamps in log tables).
