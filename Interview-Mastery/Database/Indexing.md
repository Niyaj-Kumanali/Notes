# Indexing

## 1. Executive Summary

Database indexes are data structures that improve the speed of data retrieval at the cost of additional writes and storage. Indexes are critical for query performance — a missing index is the most common cause of slow queries. However, over-indexing degrades write performance and increases storage costs. Understanding index internals (B-trees, hash indexes, GiST, GIN) enables engineers to design effective indexing strategies. In Spring Boot applications, JPA provides declarative index definitions that complement database-native index creation.

## 2. Core Theory

### 2.1 What an Index Does

An index is a copy of selected columns from a table, organized in a search-optimized structure. Without an index, the database performs a sequential (full) table scan. With an index, the database can locate rows using a tree traversal (O(log n)) or hash lookup (O(1)).

### 2.2 B-Tree Index

The default and most common index type in SQL databases.

- **Structure**: Balanced tree where leaf nodes contain pointers to heap rows (or the actual data in index-organized tables)
- **Height**: Typically 3-5 levels for billions of rows
- **Fanout**: Number of entries per node (depends on page size, typically ~200-500 entries per node)
- **Ordering**: Entries are sorted, enabling range scans and ORDER BY without additional sorting

### 2.3 Other Index Types

| Type | Use Case | Supported By |
|------|----------|-------------|
| B-Tree | General purpose, equality, range, ordering | All databases |
| Hash | Equality lookups only | PostgreSQL, MySQL (MEMORY) |
| GiST | Full-text, geometric, range types | PostgreSQL |
| GIN | Array contains, JSONB queries, full-text search | PostgreSQL |
| BRIN | Large tables with naturally ordered data | PostgreSQL |
| Bitmap | Low-cardinality columns, data warehousing | Oracle, PostgreSQL |
| Clustered | Physical row ordering (table as index) | MySQL/InnoDB, SQL Server |
| Covering | All columns needed by query in index | All databases |
| Partial | Only subset of rows indexed | PostgreSQL, SQL Server |
| Functional | Index on expression | All databases |
| Spatial | Geographic coordinates (R-tree) | MySQL, PostgreSQL, SQL Server |

## 3. Under-the-Hood Deep Dive

### 3.1 B-Tree Structure

```
         [10, 20]
        /    |    \
   [1,5,8] [12,15] [22,25,30]
   / | | \  / | \   / |  | \
  p1 p2 p3 p4 p5 p6 p7 p8 p9 p10
```

- **Root node**: Top level, contains pivot values
- **Internal nodes**: Guide search direction
- **Leaf nodes**: Contain index entries + pointer (row ID / TID / primary key) to heap row
- All leaf nodes are at the same depth (balanced property)

### 3.2 Index Scan Types

1. **Index Scan (Index Seek)**: Traverse B-tree to leaf, then fetch row from heap by pointer
2. **Index-Only Scan**: All needed columns are in the index; no heap access required
3. **Bitmap Index Scan**: Build bitmap of matching page locations, then fetch heap pages in order (reduces random I/O)
4. **Skip Scan (PostgreSQL 15+)**: Efficiently find distinct values when leading column has few values and trailing column is filtered

### 3.3 How Indexes Affect INSERT/UPDATE/DELETE

- **INSERT**: New entry added to each index on the table (O(log n) per index)
- **UPDATE**: If indexed column changes, index entry is moved (delete old + insert new)
- **DELETE**: Index entry must be removed from each index
- **Page splits**: When a B-tree node is full, it splits into two nodes — an expensive operation

### 3.4 Fillfactor

```sql
CREATE INDEX idx_orders_status ON orders(status) WITH (fillfactor = 70);
```

Fillfactor reserves free space in index pages for future updates. For tables with frequent updates to indexed columns, a fillfactor of 70-80 reduces page splits.

### 3.5 Composite Indexes

```sql
CREATE INDEX idx_orders_status_date ON orders(status, created_at DESC);
```

- **Leftmost prefix rule**: The index can be used for queries on `status`, or `status AND created_at`, but NOT `created_at` alone
- **Order of columns matters**: Put high-selectivity columns first, or columns used in equality conditions before range conditions

## 4. Production Code Examples

### 4.1 JPA Index Declarations

```java
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_status", columnList = "status"),
    @Index(name = "idx_orders_user_status", columnList = "user_id, status"),
    @Index(name = "idx_orders_created_at", columnList = "created_at DESC")
})
public class Order {
    @Id
    private Long id;

    @Column(name = "status", length = 20)
    private String status;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "user_id")
    private Long userId;
}
```

### 4.2 Unique Indexes for Constraint Enforcement

```java
@Entity
@Table(name = "users", uniqueConstraints = {
    @UniqueConstraint(name = "uq_users_email", columnNames = {"email"}),
    @UniqueConstraint(name = "uq_users_username", columnNames = {"username"})
})
public class User {
    // ...
}
```

### 4.3 Functional Index in SQL

```sql
-- PostgreSQL: index on expression
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- Query that uses this index:
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
```

```java
// Spring Data JPA with function index
@Query("SELECT u FROM User u WHERE LOWER(u.email) = LOWER(:email)")
Optional<User> findByEmailCaseInsensitive(@Param("email") String email);
```

### 4.4 Partial Index

```sql
-- Only index active users (reduces index size)
CREATE INDEX idx_users_active ON users(email) WHERE status = 'ACTIVE';

-- Query that uses this partial index:
SELECT * FROM users WHERE status = 'ACTIVE' AND email = 'test@test.com';
```

```java
// Partial indexes are database-side; JPA cannot declare them
// Use Flyway/Liquibase for schema management
```

### 4.5 Managing Indexes with Flyway

```sql
-- V2__add_indexes.sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
-- CONCURRENTLY avoids locking writes during index creation
```

```java
// @Index annotations for JPA, but use migration for complex indexes
@Configuration
public class FlywayConfig {
    @Bean
    public FlywayMigrationStrategy flywayStrategy() {
        return flyway -> {
            flyway.migrate();
            // Verify indexes exist
        };
    }
}
```

## 5. Real-World Scenarios

### 5.1 Indexing for Common Query Patterns

```sql
-- Common query:
SELECT id, title, status FROM posts
WHERE status = 'PUBLISHED'
  AND category_id = 5
  AND created_at > '2024-01-01'
ORDER BY created_at DESC
LIMIT 20;

-- Best index:
CREATE INDEX idx_posts_lookup ON posts(category_id, status, created_at DESC) INCLUDE (id, title);
```

### 5.2 Eliminating Sorting with Index

```sql
-- Query with ORDER BY
SELECT * FROM orders WHERE user_id = 100 ORDER BY created_at DESC LIMIT 10;

-- Without index: sort of all user's orders
-- With (user_id, created_at) index: B-tree already orders by created_at within user_id
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);
```

### 5.3 Hot Spot Detection and Index Maintenance

```sql
-- Check index usage statistics
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Find unused indexes
SELECT indexrelid::regclass AS index_name, relid::regclass AS table_name,
       idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

### 5.4 Reindexing

```sql
-- Rebuild index to remove bloat
REINDEX INDEX idx_orders_status;
REINDEX TABLE orders;
-- In production, use CONCURRENTLY to avoid locking
REINDEX INDEX CONCURRENTLY idx_orders_status;
```

### 5.5 Indexing for Text Search

```sql
-- PostgreSQL: GIN index for full-text search
CREATE INDEX idx_posts_fts ON posts USING GIN(to_tsvector('english', title || ' ' || content));

-- Query:
SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || content) @@ to_tsquery('english', 'database & optimization');
```

## 6. Performance

### 6.1 Index Selectivity

- **High selectivity** (unique values): Index is very efficient
- **Low selectivity** (boolean column): Index may not help for 50% values, but helps for the rare value

```sql
-- For a table where 99% of orders are 'COMPLETED' and 1% are 'PENDING':
-- Index on status is only useful for WHERE status = 'PENDING' queries
-- A partial index is ideal:
CREATE INDEX idx_orders_pending ON orders(id) WHERE status = 'PENDING';
```

### 6.2 Index Size Estimation

```sql
-- Check index size
SELECT pg_size_pretty(pg_indexes_size('orders')) AS index_size;
SELECT pg_size_pretty(pg_relation_size('idx_orders_status')) AS single_index_size;
```

### 6.3 Bitmap vs B-Tree for Low Cardinality

For low-cardinality columns (e.g., status with 3 values), a bitmap scan can be more efficient than a B-tree scan because it uses bitwise operations to combine conditions.

### 6.4 Multi-Column Index Ordering Rules

| Query Pattern | Best Index Order |
|--------------|-----------------|
| `WHERE a = 1 AND b = 2` | Either (a,b) or (b,a) — both work well |
| `WHERE a = 1 AND b > 5` | (a, b) — equality first, then range |
| `WHERE b > 5 ORDER BY a` | (b, a) — or consider (a, b) if b range is small |
| `WHERE a = 1 ORDER BY b` | (a, b) — equality then sort column |
| `WHERE a > 1 AND b = 2` | (b, a) — equality on b, then range on a |

## 7. Security

### 7.1 Index-Based Information Leakage

Indexes can leak information through timing side channels. An attacker can determine if a value exists by measuring response time (index hit vs miss). For applications handling highly sensitive data, consider:
- Using constant-time comparison queries
- Not indexing fields that could leak existence information

### 7.2 Index on Encrypted Columns

```sql
-- Indexing encrypted data is tricky
CREATE INDEX idx_encrypted_email ON users(encrypted_email);
-- This indexes the encrypted value, which is useless for lookup by plaintext
```

For indexed lookups on encrypted data:
- Use deterministic encryption (e.g., AES with same IV per value) — but reduces security
- Consider a hash index on a salted hash of the value
- Use application-level search indexes (Elasticsearch) with field-level encryption

## 8. Common Mistakes

1. **No index on foreign key columns** — JOINs perform sequential scans
2. **Indexing every column** — each index adds write overhead
3. **Wrong column order in composite index** — leading column doesn't match query patterns
4. **Ignoring NULL filtering** — PostgreSQL B-tree indexes do include NULLs (unless WHERE clause excludes them)
5. **Creating indexes during business hours without CONCURRENTLY** — causes table locks
6. **Over-indexing small tables** — small tables benefit from sequential scans; index overhead outweighs benefits
7. **Forgetting to drop unused indexes** — waste of space and write performance
8. **Assuming index will be used for `LIKE '%pattern'`** — only prefix patterns (`pattern%`) use B-tree indexes
9. **Not analyzing after bulk data loads** — stale statistics lead to optimizer ignoring valid indexes
10. **Indexing boolean columns without partial index** — most values are equal, index rarely useful

## 9. Senior Engineer Perspective

### Indexing Strategy Design

1. **Gather query patterns**: Analyze slow query log, identify most common WHERE, JOIN, ORDER BY patterns
2. **Design indexes for queries, not tables**: An index must match the query pattern, including column order
3. **Consider index maintenance**: Indexes on high-write tables need periodic reindexing
4. **Monitor index bloat**: In MVCC databases (PostgreSQL), updated/deleted index entries leave dead tuples
5. **Plan for zero-downtime**: Use CREATE INDEX CONCURRENTLY in migrations
6. **Test in staging**: Verify index usage with EXPLAIN before rolling to production
7. **Automation**: Use tools like pg_qualstats, pg_stat_statements, or pgbadger to identify missing index opportunities

### Index Design Decision Tree

```
Is the table read-heavy or write-heavy?
  read-heavy: Add more indexes for query coverage
  write-heavy: Minimize indexes, prioritize critical queries

What is the query pattern?
  Equality on multiple columns -> composite index (equality first)
  Range condition -> index on range column after equality columns
  ORDER BY -> include sort column in index
  JOIN column -> always index foreign keys
  
What is the column cardinality?
  High cardinality -> B-tree index
  Low cardinality with rare value -> partial index
  Low cardinality with many AND/OR -> bitmap or GIN index
  JSON/array -> GIN index
```

## 10. Interview Questions (20)

### Easy (10)

1. What is a database index?
2. What is the difference between a clustered and non-clustered index?
3. What data structure does the default index use?
4. How does an index speed up queries?
5. What is a primary key index?
6. What is a unique index?
7. Can you index a NULL value?
8. How many indexes can a table have?
9. What is the trade-off of adding an index?
10. What is a full table scan?

### Medium (10)

11. Explain the leftmost prefix rule for composite indexes.
12. What is the difference between an index scan and an index-only scan?
13. How would you choose column order in a composite index?
14. What is a covering index? Give an example.
15. When would a bitmap index be more efficient than a B-tree index?
16. Explain index selectivity and why it matters.
17. What is index bloat and how do you address it?
18. How does a partial index work? When would you use one?
19. What is a functional index? Give an example.
20. How does the database use indexes for ORDER BY?

## 11. Advanced Interview Questions (20)

### Hard (10)

1. Explain the internal algorithm for a B-tree page split and how it affects concurrency.
2. How does PostgreSQL's heap-only-tuple (HOT) optimization reduce index bloat?
3. What is the difference between a GiST and GIN index? When would you use each?
4. How would you design an index for geospatial queries (points within a radius)?
5. Explain the concept of index skip scan and how PostgreSQL 15+ implements it.
6. How do partial indexes interact with prepared statements and parameterized queries?
7. What is the effect of fillfactor on B-tree performance and maintenance?
8. How would you index a table with 500 columns that are all queryable?
9. Explain the difference between synchronous and asynchronous index creation (CONCURRENTLY).
10. How does index deduplication work in PostgreSQL 13+ B-tree indexes?

### System Design (11-20)

11. Design an index maintenance strategy for a 24/7 production system.
12. How would you automate detection of missing indexes across thousands of databases?
13. Design a system that recommends indexes based on query workload analysis.
14. How would you index a time-series database where queries filter on time ranges and device IDs?
15. Design a search system using GIN indexes for a document store with 10M+ documents.
16. How would you implement index-based full-text search with ranking in PostgreSQL?
17. Design a hybrid indexing strategy combining in-memory and disk-based indexes.
18. How would you handle index rebuilds for a table with 1B rows without downtime?
19. Design an index sharding strategy for a globally distributed database.
20. How would you implement a covering index strategy for a table with frequently accessed BLOB columns?

## 12. Expert-Level Interview Questions (10)

1. You have a table with 100M rows and 20 indexes. Write-heavy workload is causing index bloat and poor performance. How do you redesign the indexing strategy?
2. Design a concurrent B-tree index structure that supports lock-free searches during page splits.
3. How would you implement a multi-dimensional index (e.g., for 10D vector similarity) on top of PostgreSQL's GiST framework?
4. Describe how a database can use LSM-trees (LevelDB/RocksDB style) instead of B-trees. When would LSM be preferable?
5. How would you implement an index advisor that uses machine learning to predict the cost/benefit of candidate indexes?
6. Design a system where indexes are stored on NVMe SSD while the heap is on HDD, optimizing for cost and performance.
7. How would you implement partial updates to a JSONB column such that only changed keys are reindexed in the GIN index?
8. Describe the algorithm for merging multiple B-tree indexes into a single composite index without downtime.
9. How would you design an index structure that supports both point lookups and similarity search (hybrid B-tree + HNSW)?
10. Design a distributed index for a global database where write latency to different regions varies from 1ms to 500ms.

## 13. Debugging & Troubleshooting

### Identifying Missing Indexes

```sql
-- PostgreSQL: Queries that would benefit from indexes
SELECT relname, seq_scan, seq_tup_read, idx_scan,
       seq_tup_read / NULLIF(seq_scan, 0) AS avg_rows_per_seq_scan
FROM pg_stat_user_tables
WHERE seq_scan > 1000
ORDER BY seq_tup_read DESC
LIMIT 20;

-- MySQL: Full scan queries
SELECT * FROM sys.schema_unused_indexes;
SELECT * FROM sys.schema_index_statistics;
```

### Detecting Index Bloat

```sql
-- PostgreSQL: estimate index bloat
SELECT
    current_database(), schemaname, tablename, indexname,
    ROUND(100 * (avg_leaf_density - 4) / (69 - 4)) AS bloat_pct
FROM pg_stat_user_indexes;

-- More precise bloat estimation with pgstattuple
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstatindex('idx_orders_status');
```

### Checking JPA Index Creation

```yaml
logging:
  level:
    org.hibernate.tool.hbm2ddl: DEBUG
```

## 14. Comparison Section

| Index Type | Pros | Cons | Best For |
|-----------|------|------|----------|
| B-Tree | Balance, general purpose | Not for complex types | Equality, range, sort |
| Hash | O(1) lookup | Only equality, no sorting | Primary key lookups |
| GiST | Extensible, multi-purpose | Slower build, larger | FTS, geometry, ranges |
| GIN | Fast for composite values | Slow inserts | Arrays, JSONB, FTS |
| BRIN | Tiny index for ordered data | Only works for natural order | Time-series, logs |
| Bitmap | Efficient for multiple conds | Not for OLTP | Data warehouse |
| Partial | Small index size | Only covers subset | Frequent filter on rare value |
| Covering | No heap access | Duplicates data | Frequent query columns |
| Clustered | Fast range scans | One per table | Primary key range scans |

## 15. Revision Notes

- Index is a copy of data optimized for search — costs storage and write performance
- B-tree is the default; height stays low even for billions of rows
- Composite index: leftmost prefix rule — order columns by query pattern
- Index types matter: use GIN for JSONB/arrays, GiST for geometry, BRIN for time-series
- Partial indexes save space when queries only filter on a subset
- Covering indexes (INCLUDE) enable index-only scans
- Monitor unused indexes and index bloat
- Use CREATE INDEX CONCURRENTLY to avoid production downtime
- Hibernate @Index is convenient; complex indexes go in migration scripts

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                      INDEXING CHEAT SHEET                         |
+-------------------------------------------------------------------+
|                                                                   |
|  B-TREE INDEX STRUCTURE:                                          |
|                                                                   |
|             [Root: 50, 100]                                       |
|            /        |        \                                    |
|   [10,20,30,40]  [60,70,80]  [110,120,130]                       |
|   /  |  |  |  \   /  |  |  \   /   |   |   \                    |
|  p1 p2 p3 p4 p5  p6 p7 p8 p9  p10  p11 p12 p13                  |
|  Leaf nodes: (key, row_pointer)                                  |
|  Height ~ log_fanout(rows) -> typically 3-5 levels               |
|                                                                   |
+-------------------------------------------------------------------+
|  INDEX TYPE SELECTION:                                            |
|                                                                   |
|  Query Type                  Recommended Index                   |
|  ---------------------------+----------------------------------- |
|  =, <>, IN, <, >, BETWEEN   | B-Tree (default)                   |
|  ORDER BY                    | B-Tree (match sort order)          |
|  LIKE 'prefix%'              | B-Tree                            |
|  LIKE '%any%'                | GIN (trigram) or Full-Text Search |
|  JSONB @> key-value          | GIN                               |
|  array_column @> {value}     | GIN                               |
|  tsvector @@ tsquery         | GIN                               |
|  geo_column <@ polygon       | GiST (SP-GiST)                    |
|  timestamp range             | BRIN (if naturally ordered)       |
|  WHERE status = 'RARE'       | Partial B-Tree                    |
|  SELECT col1, col2 WHERE...  | Covering B-Tree (with INCLUDE)    |
|                                                                   |
+-------------------------------------------------------------------+
|  COMPOSITE INDEX DESIGN RULES:                                    |
|                                                                   |
|  1. Equality conditions first, range conditions last              |
|  2. High selectivity columns first                                |
|  3. Leftmost prefix: index (a,b,c) supports:                     |
|     - WHERE a = ?                    (uses prefix)                |
|     - WHERE a = ? AND b = ?          (full match)                 |
|     - WHERE a = ? AND b = ? AND c = ?(full match)                |
|     - WHERE b = ?                    (no index)                   |
|     - WHERE a = ? AND c = ?          (only a used)                |
|                                                                   |
+-------------------------------------------------------------------+
|  SPRING BOOT JPA INDEX ANNOTATIONS:                               |
|                                                                   |
|  @Table(indexes = {                                               |
|    @Index(name = "idx_name", columnList = "col1, col2")           |
|  })                                                               |
|                                                                   |
|  @Table(uniqueConstraints = {                                     |
|    @UniqueConstraint(name = "uq_name", columnNames = {"col1"})    |
|  })                                                               |
|                                                                   |
+-------------------------------------------------------------------+
|  PRODUCTION INDEX COMMANDS:                                       |
|                                                                   |
|  CREATE INDEX CONCURRENTLY idx ON t(col);     -- no lock          |
|  CREATE UNIQUE INDEX idx ON t(col);           -- unique           |
|  CREATE INDEX idx ON t(col1, col2 DESC);      -- composite/sort   |
|  CREATE INDEX idx ON t USING GIN(col);        -- GIN              |
|  CREATE INDEX idx ON t(col) WHERE cond;       -- partial          |
|  CREATE INDEX idx ON t(LOWER(col));           -- functional       |
|  DROP INDEX CONCURRENTLY IF EXISTS idx;       -- drop without lock|
|  REINDEX INDEX CONCURRENTLY idx;              -- rebuild          |
|                                                                   |
+-------------------------------------------------------------------+
