# Database Indexing

---

## What is Indexing?

- **Definition**
  - **Database indexes** are data structures that improve the speed of data retrieval at the cost of additional writes and storage. They act like a book's index — instead of scanning every page, you look up the term in the index and jump directly to the relevant pages. The default index type in most databases is the **B-Tree**, but other types like **Hash**, **GiST**, **GIN**, and **BRIN** serve specialized use cases.

### How Indexes Work

- **Index Structure** — An index is a copy of selected columns from a table, organized in a search-optimized structure (typically a B-Tree). This copy is maintained separately from the main table data, which is why indexes consume additional storage space.
- **Full Table Scan** — Without an index, the database reads every row sequentially to find matches. This is O(n) complexity and becomes prohibitively slow as table size grows into millions of rows.
- **Indexed Lookup** — With an index, the database can locate rows using tree traversal in O(log n) time or hash lookup in O(1). For a table with 10 million rows, an index reduces lookups from scanning 10M rows to traversing roughly 3-4 B-Tree levels.

### B-Tree Index Structure

- **Root Node** — Contains pivot values that guide the search direction down the tree. The root is the single entry point for all index lookups.
- **Internal Nodes** — Direct the search to the correct child node based on comparison with boundary values. Each level narrows the search space significantly.
- **Leaf Nodes** — Contain the actual index entries plus a pointer (TID) to the heap row. Leaf nodes are linked horizontally, enabling efficient range scans.
- **Balanced Property** — All leaf nodes are at the same depth, ensuring consistent O(log n) search performance regardless of where the data lives in the tree.
- **Tree Height** — Typically 3-5 levels for billions of rows, making even the largest tables searchable in microseconds. This logarithmic scaling is why B-Trees dominate database indexing.

### Index Scan Types

- **Index Scan (Index Seek)** — Traverses the B-Tree from root to leaf, then fetches the row from the heap using the pointer. Requires two I/O operations per row — one for the index, one for the heap.
- **Index-Only Scan** — All needed columns are present in the index itself, so no heap access is required. This is the fastest scan type and is enabled by covering indexes that INCLUDE extra columns.
- **Bitmap Index Scan** — Builds a bitmap of matching page locations, then fetches heap pages in physical order. Reduces random I/O significantly when multiple conditions match rows scattered across many pages.
- **Skip Scan (PostgreSQL 15+)** — Efficiently finds distinct values when the leading column of a composite index has few values and the trailing column is filtered. Avoids scanning the entire index for sparse data patterns.

### Composite Indexes

- **Leftmost Prefix Rule** — An index on `(A, B, C)` can serve queries filtering on `A` alone, `A AND B`, or `A AND B AND C`. It cannot be used for queries on `B` alone or `C` alone because the index sorts by the leading column first.
- **Column Order Strategy** — Column order matters significantly for composite index effectiveness. Put equality conditions first, then range conditions, then sort columns to maximize the index's usefulness across different query patterns.

### Other Index Types

- **Hash Index** — Provides O(1) lookup for equality comparisons only. Does not support sorting or range queries, which limits its use cases to primary key or unique constraint lookups.
- **GiST** — An extensible index designed for full-text search, geometric data, and range types. Supports complex operators like proximity, containment, and overlap that B-Trees cannot handle.
- **GIN** — Indexes composite values such as arrays, JSONB, and full-text search vectors. Efficient for queries checking whether an element exists within a composite value or document.
- **BRIN** — A tiny, space-efficient index for naturally ordered data like time-series and logs. Can be 100-1000x smaller than a B-Tree but only works when physical page order correlates with the indexed column.
- **Partial Index** — Indexes only a subset of rows using a WHERE clause (e.g., `WHERE status = 'ACTIVE'`). Saves significant storage and write overhead when only a small fraction of rows need fast lookups.
- **Covering Index** — Includes extra columns via the INCLUDE clause at the leaf level, enabling index-only scans. The included columns don't affect the sort order but eliminate heap visits.
- **Functional Index** — Indexes the result of an expression like `LOWER(email)`. Enables efficient searches on transformed column values without wrapping the column in a function in the WHERE clause.

---

## Core Concepts

### How Indexes Affect Write Operations

- Indexes speed up reads but slow down writes:
  - **INSERT** — A new entry is added to each index on the table, costing O(log n) per index. The overhead grows linearly with the total number of indexes.
  - **UPDATE** — If an indexed column changes, the old entry is deleted and a new one is inserted in the index. This doubles the maintenance cost compared to inserts alone.
  - **DELETE** — The index entry is removed from every index on the table. Each index must be updated independently, making deletes proportionally more expensive on heavily indexed tables.
  - **Page Splits** — When a B-Tree node is full, it splits into two nodes, which is an expensive operation requiring reorganization. Use `fillfactor` (e.g., `70`) to reserve free space for updates and reduce the frequency of splits.

### Index Selectivity

- **High Selectivity** — When a column has many unique values relative to the row count, the index narrows results to very few rows per probe. Highly selective indexes are extremely efficient for point lookups.
- **Low Selectivity** — When a column has few unique values (like a boolean), the full index may not help for common values but is effective for rare values. A partial index targeting the rare value is the ideal solution:

```sql
-- For a table where 99% of orders are 'COMPLETED' and 1% are 'PENDING':
CREATE INDEX idx_orders_pending ON orders(id) WHERE status = 'PENDING';
```

### JPA Index Declarations

```java
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_status", columnList = "status"),
    @Index(name = "idx_orders_user_status", columnList = "user_id, status"),
    @Index(name = "idx_orders_created_at", columnList = "created_at DESC")
})
public class Order { ... }
```

### Managing Indexes with Migrations

- Complex indexes (partial, functional, concurrent) belong in migration scripts:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
REINDEX INDEX CONCURRENTLY idx_orders_status;
```

- `CONCURRENTLY` avoids locking writes during index creation, which is essential for production environments.

---

## Common Mistakes

- **No index on foreign key columns**
  - JOINs on foreign key columns without indexes perform sequential scans instead of index seeks. This is one of the most common causes of slow query performance in relational databases.
  - **Why it looks correct:** The join works and returns correct results; the sequential scan is invisible until the referenced table grows past a few thousand rows.

- **Indexing every column**
  - Each additional index adds write overhead and consumes storage. Only index columns that actually appear in WHERE, JOIN, and ORDER BY clauses.
  - **Why it looks correct:** "More indexes = faster queries" seems intuitively right, and the write slowdown only becomes apparent under insert-heavy workloads.

- **Wrong column order in composite indexes**
  - If the leading column does not match query patterns, the index is effectively useless for those queries. Always put equality-filtered columns first in composite indexes.
  - **Why it looks correct:** Creating an index on (A, B) seems sufficient; the leftmost prefix rule is non-intuitive to developers who haven't studied B-Tree structure.

- **Creating indexes during business hours without CONCURRENTLY**
  - A plain `CREATE INDEX` acquires a lock that blocks writes on the table. Always use `CREATE INDEX CONCURRENTLY` in production environments.
  - **Why it looks correct:** The index creation command succeeds without errors; the silent lock escalation only becomes visible when application writes start timing out.

- **Over-indexing small tables**
  - Tables with only a few hundred rows are faster with sequential scans because index overhead outweighs any benefit. Index maintenance costs are not justified for very small tables.
  - **Why it looks correct:** Indexing is always taught as "the solution to slow queries," and developers don't expect that an index can make a small-table query slower than a sequential scan.

- **Assuming indexes work for `LIKE '%pattern'`**
  - B-Tree indexes only support prefix search patterns (`pattern%`). For suffix or containment searches, use a GIN index with the `pg_trgm` extension.
  - **Why it looks correct:** A B-Tree index exists on the column and the query planner uses it for prefix patterns, so developers assume it works for all LIKE patterns equally.

- **Not analyzing after bulk data loads**
  - Stale statistics cause the query optimizer to ignore valid indexes or choose suboptimal plans. Always run `ANALYZE` after large data changes.
  - **Why it looks correct:** The query still runs and returns results; the optimizer's wrong choice is invisible without comparing estimated versus actual rows in the execution plan.

- **Indexing boolean columns without a partial index**
  - When most rows have the same boolean value, a full index on the column is rarely useful. Create a partial index with `WHERE col = TRUE` for focused performance.
  - **Why it looks correct:** A full index on the boolean column appears in EXPLAIN plans and seems helpful; the bloat from indexing every row including the 99% majority is invisible to casual inspection.

---

## Real-World Scenarios

### E-Commerce Product Search with Composite Indexes

- A product catalog with 10M products. Users filter by `category_id` and sort by `price`. Without a composite index on `(category_id, price)`, the database filters then sorts in memory. With the index, the B-Tree stores rows sorted by price within each category:

```sql
CREATE INDEX idx_category_price ON products(category_id, price);

SELECT id, name, price FROM products
WHERE category_id = 42
ORDER BY price
LIMIT 20;
```

### Time-Series Monitoring with BRIN Indexes

- An IoT platform ingests 1B events/day into `sensor_readings`. Queries filter by `timestamp`. A B-Tree index takes 50GB. A BRIN index takes 50MB — 1000x smaller:

```sql
CREATE INDEX idx_readings_time ON sensor_readings USING BRIN(timestamp);

SELECT * FROM sensor_readings
WHERE timestamp >= '2024-06-01' AND timestamp < '2024-06-02';
```

### Social Media Notifications with Partial Indexes

- A notifications table has 100M rows, 99.9% are `read = TRUE`. A full index on `(user_id, created_at)` is 2GB. A partial index with `WHERE read = FALSE` is 2MB with identical performance:

```sql
CREATE INDEX idx_unread_notifications ON notifications(user_id, created_at) WHERE read = FALSE;

SELECT * FROM notifications WHERE read = FALSE AND user_id = 123 ORDER BY created_at DESC LIMIT 50;
```

## Scenario-Based Questions

**Q: You are building an e-commerce product listing where users filter by any combination of 8 optional attributes. How do you index without creating 256 indexes?**

- Use a composite index on the 2-3 most selective columns queried most often. For the rest, let PostgreSQL use bitmap scans combining multiple single-column indexes. Alternatively, use a GIN index on a JSONB column storing all attribute values, or create partial indexes per common filter combination (e.g., `WHERE category_id IS NOT NULL AND price IS NOT NULL`).
- **Interview follow-up:** With the JSONB GIN approach, how do you handle range filters like `price BETWEEN 10 AND 100` on individual numeric attributes stored in a single JSONB column?

---

**Q: Your application's write throughput dropped 80% after adding 5 B-Tree indexes to a table receiving 10K writes/second. The indexes are necessary for reads. How do you fix it?**

- First, replace full indexes with partial indexes where possible (e.g., only index active records). Second, reduce `fillfactor` to 70-80 to minimize page splits. Third, batch writes to reduce per-row index maintenance overhead. Fourth, consider table partitioning so each partition has smaller indexes.
- **Interview follow-up:** After adding the indexes, you notice autovacuum runs constantly and still can't keep up with dead tuple accumulation. How do you determine whether index bloat or table bloat is the root cause?

---

**Q: A 500M row orders table dashboard query groups by `status` and `region` aggregating `total`. It runs fine at 2AM but times out at 2PM with a different execution plan. How do you stabilize it?**

- Plan instability caused by changing statistics or parameter values. Create a covering index on `(status, region) INCLUDE (total)` for index-only scans. Increase `work_mem` to prevent disk spills for hash aggregates. Use a materialized view refreshed periodically for the dashboard query.
- **Interview follow-up:** The covering index works, but the index size is 12GB and maintenance windows are tight. How do you trade off index size against refresh performance?

---

**Q: Your team uses UUID primary keys. Insert performance degrades beyond 10M rows. What is the root cause and how do you fix it?**

- Random UUIDs cause frequent B-Tree page splits — new rows insert at random positions, making the tree unbalanced and reducing cache hit ratio. Fix by switching to UUID v7 (time-ordered), using ULIDs, or using a sequential `BIGSERIAL` as the clustered PK with UUID as a secondary unique index.

---

**Q: A `WHERE status IN ('PENDING', 'PROCESSING')` query scans the full index but performs poorly. Both values are common (~15% each). The index is on `(status, created_at)`. How do you optimize?**

- For IN lists with common values, the optimizer may choose a bitmap or seq scan. Create a partial index for rare statuses and let common statuses use a different path. Reorder to `(created_at, status)` if the query always uses a date range. Add a BRIN index on `created_at` combined with a separate status index.

---

**Q: `SELECT COUNT(*) FROM orders WHERE status = 'SHIPPED'` on a 200M row table takes 30 seconds despite having an index. How do you make it instant?**

- The index still visits all 200M matching entries to count them. Use a partial index: `CREATE INDEX idx_shipped ON orders(id) WHERE status = 'SHIPPED'` so `COUNT(*)` scans only matching entries. Alternatively, maintain a materialized view with pre-aggregated counts.

---

**Q: After `VACUUM FULL` on a heavily updated table, queries are slower than before. What happened?**

- `VACUUM FULL` rebuilds the table and indexes in physical order. If the original order matched query patterns (time-ordered inserts), the new order may increase random I/O. It also invalidates cached plans and statistics — run `ANALYZE` afterward.

---

**Q: Your Spring Boot app generates `WHERE id IN (1, 2, ..., 50)`. Performance is fine for 50 IDs but degrades at 500 IDs. Why?**

- The planner may choose a seq scan over repeated index lookups when the IN list is large, since random I/O for 500 index probes exceeds seq scan cost. Use a temporary table with JOIN for large lists, or batch in chunks of 50-100.

---

**Q: A migration adds a new index and application queries start blocking during creation. What went wrong?**

- The index was created without `CONCURRENTLY`. Plain `CREATE INDEX` acquires a lock blocking writes on the table. Always use `CREATE INDEX CONCURRENTLY` in production and monitor progress via `pg_stat_progress_create_index`.

---

**Q: A query joining `orders` (10M rows) and `customers` (5M rows) on `customer_id` uses a nested loop with 10M loops despite an index on `orders.customer_id`. Why is this slow and how do you fix it?**

- Even with an index, 10M random I/O operations are expensive. The optimizer chose nested loop expecting few rows. Ensure statistics are current. Force a hash join temporarily with `enable_nestloop = off` or increase `work_mem` to accommodate the hash table.

## Interview Questions

- **What is a B-Tree index and how does it work?**
  - A B-Tree is a balanced tree data structure where leaf nodes contain index entries pointing to heap rows. Search traverses from root to leaf in O(log n) time, supporting both equality and range lookups.

- **What is the difference between clustered and non-clustered indexes?**
  - A clustered index determines physical row order on disk — a table can have only one. A non-clustered index is a separate structure with pointers to heap rows. InnoDB uses the PK as clustered; PostgreSQL uses heap storage with separate indexes.

- **When would you use a GIN index over a B-Tree?**
  - GIN indexes are designed for composite values like JSONB, arrays, and full-text search vectors. B-Trees cannot efficiently index the individual elements of an array or JSONB field.

- **What is index selectivity and why does it matter?**
  - Selectivity measures how many rows a query condition matches. High selectivity (few rows matched) makes indexes efficient, while low selectivity (many rows matched) may make a sequential scan cheaper. Partial indexes specifically address low-selectivity columns.

- **What is the leftmost prefix rule?**
  - For a composite index on (A, B, C), queries must reference column A (the leftmost) for the index to be used. The index supports A, A+B, and A+B+C queries but not B alone or C alone.

- **When does PostgreSQL choose a bitmap scan over an index scan?**
  - When multiple conditions match rows scattered across disk pages. A bitmap scan builds a page-level bitmap of matching locations, sorts them, then fetches heap pages in order — reducing random I/O significantly.

- **How do you detect and remove unused indexes?**
  - Query `pg_stat_user_indexes` for indexes with `idx_scan = 0`. Drop unused indexes to reduce write overhead and reclaim storage. Use `DROP INDEX CONCURRENTLY` in production to avoid blocking writes.

- **How does CREATE INDEX CONCURRENTLY differ from regular CREATE INDEX?**
  - `CREATE INDEX CONCURRENTLY` builds the index without locking writes, using a three-pass algorithm. It takes longer but avoids downtime. If it fails, the invalid index must be dropped before retrying.

- **What is index bloat, what causes it, and how do you fix it?**
  - Index bloat is wasted space from dead index entries left by UPDATE and DELETE operations. Fix with `REINDEX INDEX CONCURRENTLY`. Ensure autovacuum is properly tuned and monitor bloat with `pgstattuple`.

- **How would you index a 1B row table with frequent GROUP BY queries on `department_id` aggregating `salary`?**
  - Create a covering index on `(department_id) INCLUDE (salary)` for index-only scans. If `department_id` has low cardinality, consider partitioning by department for additional performance gains.

## Developer Recommendations

- **Use composite indexes with equality-first column ordering**
  - An index on `(status, created_at)` supports `WHERE status = 'PENDING' ORDER BY created_at`. Placing range columns last prevents the index from being used for equality filters after the first range condition.
  - **Production story:** A team indexed `(created_at, status)` for a query filtering by `status = 'ACTIVE'`. The index was useless — the optimizer fell back to a sequential scan on 50M rows, causing a 5-second query. Reordering to `(status, created_at)` dropped it to 5ms.

- **Prefer partial indexes for low-selectivity columns**
  - A full index on a boolean column indexes both true and false values. A partial index with `WHERE is_active = TRUE` is dramatically smaller and equally effective for queries filtering on the active subset.

- **Monitor index usage via pg_stat_user_indexes**
  - Indexes with `idx_scan = 0` waste write throughput and storage with no read benefit. Drop them after testing to reduce maintenance overhead.
  - **Production story:** A team inherited a schema with 14 unused indexes on a 200M-row table. Each index added ~3ms of write latency per INSERT. Dropping them cut a nightly batch ETL from 4 hours to 45 minutes.

- **Use covering indexes with INCLUDE for index-only scans**
  - Adding non-filtered columns via `INCLUDE` avoids heap fetches for frequent queries. The trade-off is larger index size but significantly faster reads.

- **Set fillfactor on tables with frequent UPDATEs**
  - Default 100 fills pages completely, causing expensive page splits on updates. Set fillfactor to 70-80 to reserve space for in-place updates (HOT updates in PostgreSQL).

- **Create indexes CONCURRENTLY in production**
  - Plain `CREATE INDEX` blocks writes. `CREATE INDEX CONCURRENTLY` adds the index without downtime at the cost of slower creation and higher resource usage.

- **Consider BRIN over B-Tree for append-only data**
  - BRIN indexes are 100-1000x smaller than B-Tree for time-ordered data. They work best when physical page order correlates with the indexed column (e.g., timestamps in log tables).
