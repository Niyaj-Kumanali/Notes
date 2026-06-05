# Query Optimization

## 1. Executive Summary

Query optimization is the process of tuning SQL queries to minimize resource consumption (CPU, I/O, memory) and reduce response time. In production systems, poorly optimized queries are the most common cause of performance degradation. A single bad query can saturate database resources and bring down an entire application. Optimization requires understanding of query plans, indexing, statistics, and database internals. In Java/Spring Boot applications, Hibernate-generated queries must also be inspected and tuned.

## 2. Core Theory

### 2.1 Cost-Based Optimization (CBO)

Modern databases use cost-based optimization:
- **Cost units**: I/O page reads, CPU time, memory usage
- **Statistics**: Table row count, column distinct values, data distribution, NULL density
- **Cardinality estimation**: Estimating how many rows each operation returns
- **Access paths**: Sequential scan, index scan, bitmap scan, index-only scan

### 2.2 Query Optimization Phases

```
Query -> Parser -> Logical Rewrite -> Cost-Based Optimization -> Plan Execution
```

- **Logical Rewrite**: Predicate pushdown, view expansion, subquery unnesting, constant folding
- **Physical Optimization**: Choose join algorithm, access method, join order

### 2.3 Key Optimization Principles

1. **Reduce data accessed** — filter early, project only needed columns
2. **Reduce data volume** — aggregate early, use LIMIT
3. **Efficient access paths** — use indexes to avoid full scans
4. **Efficient joins** — proper join order, suitable join algorithm
5. **Minimize round trips** — batch operations, avoid N+1

## 3. Under-the-Hood Deep Dive

### 3.1 Statistics Collection

```sql
-- PostgreSQL: manually analyze
ANALYZE users;

-- Check statistics
SELECT tablename, attname, n_distinct, most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename = 'users';
```

Stale statistics lead to bad cardinality estimates and poor plan choices. Auto-vacuum and auto-analyze help but may not keep up with high write volumes.

### 3.2 Join Order Optimization

```sql
SELECT *
FROM A JOIN B ON A.id = B.a_id
JOIN C ON B.id = C.b_id
```

The optimizer considers join orders: (A join B) join C, (B join C) join A, etc. For 3 tables there are 12 possible join orders; for 10 tables there are 17 million. The optimizer uses dynamic programming to prune the search space.

### 3.3 Predicate Pushdown

```sql
-- Query
SELECT * FROM (
    SELECT * FROM orders WHERE total > 100
) AS o JOIN customers c ON o.customer_id = c.id;

-- Optimizer rewrites to (predicate pushdown)
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id
WHERE o.total > 100;
```

Filters are pushed as close to the data source as possible to reduce intermediate rows.

### 3.4 Subquery vs Join Execution

```sql
-- Subquery
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders);

-- Optimizer may rewrite as SEMI JOIN
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

Most optimizers rewrite IN subqueries to EXISTS or JOIN with DISTINCT, choosing the lowest-cost plan.

### 3.5 Materialized CTEs and Optimization Fences

```sql
-- PostgreSQL: CTE may be materialized (optimization fence pre-12)
WITH filtered AS MATERIALIZED (
    SELECT * FROM users WHERE status = 'ACTIVE'
)
SELECT * FROM filtered WHERE created_at > '2024-01-01';
```

In PostgreSQL 12+, use `MATERIALIZED` or `NOT MATERIALIZED` to control behavior.

## 4. Production Code Examples

### 4.1 Identifying Slow Queries in Spring Boot

```yaml
# application.yml
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
    org.hibernate.stat: DEBUG
    org.hibernate.SQL_SLOW: INFO
```

```java
// Programmatic query logging
@Configuration
public class HibernateConfig {
    @Bean
    public HibernatePropertiesCustomizer slowQueryLogging() {
        return props -> {
            props.put("hibernate.generate_statistics", "true");
            props.put("hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS", "100");
        };
    }
}
```

### 4.2 Optimizing JPA Queries

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

### 4.3 Large Dataset Processing

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

// Or use cursors
@Transactional(readOnly = true)
@QueryHints(@QueryHint(name = org.hibernate.jpa.HibernateHints.HINT_FETCH_SIZE, value = "100"))
@Query("SELECT u FROM User u")
Stream<User> streamAllUsersWithBatchSize();
```

### 4.4 Bulk Updates

```java
// BAD: updates each row individually
List<User> users = userRepository.findByStatus("PENDING");
users.forEach(u -> { u.setStatus("PROCESSED"); userRepository.save(u); });

// GOOD: bulk update
@Modifying
@Query("UPDATE User u SET u.status = 'PROCESSED' WHERE u.status = 'PENDING'")
int bulkMarkProcessed();
```

## 5. Real-World Scenarios

### 5.1 Pagination Deep Dive

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

### 5.2 Eager vs Lazy Loading Tuning

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

### 5.3 Read-Write Splitting

```yaml
spring:
  datasource:
    primary:
      url: jdbc:postgresql://primary:5432/db
    replica:
      url: jdbc:postgresql://replica:5432/db
```

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

## 6. Performance

### 6.1 Query Response Time Components

```
Response Time = Network Time + Query Execution Time
Query Execution Time = Parse Time + Plan Time + Execute Time
Execute Time = I/O Wait + CPU Time + Lock Wait
```

### 6.2 Optimization Targets

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| High CPU | Missing index, large sort, heavy aggregations | Add index, reduce sort columns |
| High I/O | Full table scans, no index | Add covering index |
| High lock wait | Long transactions, contention | Reduce transaction scope |
| Slow first page | Missing LIMIT, unoptimized join | Add LIMIT, optimize join |
| Slow later pages | OFFSET pagination | Use keyset pagination |
| N+1 queries | Lazy loading in loops | Use JOIN FETCH or batch size |

### 6.3 Explain Plan Metrics

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING) SELECT * FROM orders WHERE status = 'PENDING';
```

Key metrics:
- **actual rows / estimated rows** — cardinality mismatch indicates stale statistics
- **rows removed by filter** — inefficient filtering
- **buffers: shared hit/read** — hit ratio (hit/(hit+read)) should be >99%
- **execution time** — actual runtime

### 6.4 Index-Only Scans

```sql
-- Covering index avoids heap lookups
CREATE INDEX idx_orders_status_created ON orders(status, created_at);

SELECT status, created_at FROM orders WHERE status = 'SHIPPED';
-- Can be fully served from index, no heap access needed
```

## 7. Security

### 7.1 Query Plan Information Leakage

Explain plans can leak schema information. In production:
```sql
-- Restrict EXPLAIN to privileged users
REVOKE EXECUTE ON FUNCTION pg_stat_statements_reset() FROM PUBLIC;
```

### 7.2 Time-Based Information Leakage

Avoid queries whose response time depends on data values if you might leak information through timing side channels. Use constant-time comparisons for sensitive operations.

## 8. Common Mistakes

1. **Premature optimization without measurement** — always profile first
2. **Ignoring database statistics** — missing or stale ANALYZE leads to bad plans
3. **Using functions on indexed columns in WHERE** — prevents index usage
4. **Assuming join order in query matches execution order** — optimizer reorders
5. **Over-indexing** — every index slows down writes
6. **Not using EXPLAIN ANALYZE** — guessing instead of measuring
7. **Fetching too much data** — not using LIMIT, selecting unnecessary columns
8. **N+1 queries from ORM** — most common Hibernate performance issue
9. **Not tuning connection pool** — too small causes contention, too large causes resource exhaustion
10. **Relying on nested loop joins for large datasets** — hash join or merge join is usually better

## 9. Senior Engineer Perspective

Production query optimization workflow:

1. **Measure**: Enable slow query logging, set threshold at 100ms
2. **Identify**: Use pg_stat_statements, slow query log, APM tools
3. **Profile**: Run EXPLAIN (ANALYZE, BUFFERS)
4. **Analyze**: Check for full scans, bad cardinality estimates, missing indexes
5. **Fix**: Add indexes, rewrite queries, update statistics
6. **Verify**: Re-profile to confirm improvement
7. **Monitor**: Track query performance over time

Advanced techniques:
- **Plan stability**: Use pg_hint_plan or SQL plan management to force good plans
- **Query rewriting**: Sometimes the optimizer needs help with complex queries
- **Materialized views**: Pre-compute expensive joins/aggregations refreshed periodically
- **Partition pruning**: Ensure WHERE clause matches partition key

## 10. Interview Questions (20)

### Easy (10)

1. What is the main goal of query optimization?
2. What is the difference between a sequential scan and an index scan?
3. What does EXPLAIN do in SQL?
4. Why is SELECT * considered bad practice?
5. What is the N+1 query problem?
6. What does the LIMIT clause do for performance?
7. Why should you avoid functions on indexed columns in WHERE?
8. What is the purpose of database statistics?
9. How does adding an index affect INSERT performance?
10. What is the difference between logical and physical I/O?

### Medium (10)

11. Explain the difference between `EXPLAIN` and `EXPLAIN ANALYZE`.
12. How do you detect and fix the N+1 problem in Hibernate?
13. What is a covering index and when would you use one?
14. Explain predicate pushdown and why it helps.
15. How does keyset (cursor-based) pagination improve performance over offset pagination?
16. What is the difference between a hash join and a nested loop join?
17. How do you identify a missing index from an EXPLAIN plan?
18. What is parameter sniffing and how does it affect query plans?
19. How would you optimize a query that uses LIKE '%pattern%'?
20. Explain the role of the query optimizer's cost model.

## 11. Advanced Interview Questions (20)

### Hard (10)

1. How does the optimizer choose between a hash join, merge join, and nested loop join?
2. Explain the concept of join skew and how to handle it.
3. How would you optimize a query that joins 15 tables?
4. What is a bitmap scan and when is it preferable to an index scan?
5. Explain the difference between adaptive joins and static join selection.
6. How does the optimizer estimate cardinality for complex predicates?
7. What is the Halloween Problem in database optimization?
8. Explain the concept of late materialization in columnar databases.
9. How would you handle query optimization across sharded databases?
10. What is plan regression and how do you prevent it?

### System Design (11-20)

11. Design a slow query monitoring and alerting system for a production database.
12. How would you design a query cache layer between your application and database?
13. Design a query routing system that sends read queries to replicas and writes to the primary.
14. How would you architect a system to handle seasonal traffic spikes?
15. Design a system for automatic query performance regression detection in CI/CD.
16. How would you design a data archiving strategy to keep the main query table performant?
17. Design a near-real-time reporting system that doesn't impact the OLTP database.
18. How would you design a query optimization advisor that suggests index changes?
19. Design a multi-tenant system where query performance isolation is critical.
20. How would you design a database migration strategy that minimizes query downtime?

## 12. Expert-Level Interview Questions (10)

1. How would you approach optimizing a query where the optimizer chooses a bad plan due to correlation between columns? The statistics don't capture multi-column correlation.
2. Design a query rewrite engine that automatically detects and fixes N+1 queries at runtime in a Hibernate application.
3. You have a table with 1 billion rows and a skewed distribution — 90% of queries filter on the 10% most common values. The optimizer chooses a full scan because it thinks the common value will match most rows, but the actual queries filter for rare values. How do you fix this?
4. How would you implement a cost model for a new storage engine that uses a mix of SSD and HDD?
5. Design a system that can autotune PostgreSQL configuration parameters (work_mem, shared_buffers, etc.) based on query workload patterns.
6. How would you implement a distributed query optimizer for a system that spans PostgreSQL, Kafka, and S3?
7. Describe the algorithm for incremental view maintenance in a data warehouse.
8. How would you implement an approximate query processing system that returns fast, accurate-enough results for aggregate queries?
9. Design a query fingerprinting and normalization system for a query performance database.
10. How would you design a self-healing database that detects and automatically fixes regressed query plans?

## 13. Debugging & Troubleshooting

### PostgreSQL Query Diagnostics

```sql
-- Top 10 queries by total execution time
SELECT queryid, query, calls, total_exec_time,
       mean_exec_time, rows, shared_blks_hit, shared_blks_read
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries with most I/O
SELECT queryid, query, calls, shared_blks_read, shared_blks_hit,
       ROUND(shared_blks_hit::numeric /
           NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 2) AS hit_ratio
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY shared_blks_read DESC
LIMIT 10;
```

```sql
-- Find queries waiting for locks
SELECT pid, wait_event_type, wait_event, query, state
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

### MySQL Query Diagnostics

```sql
-- Slow query log configuration
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- Check query execution
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE id = 1;
```

### Java-Side Diagnostics

```java
// Hibernate statistics listener
@Configuration
public class HibernateStatsConfig {
    @EventListener
    public void onSessionFactoryCreated(SessionFactoryCreatedEvent event) {
        SessionFactory factory = event.getSessionFactory();
        factory.getStatistics().setStatisticsEnabled(true);
    }
}
```

## 14. Comparison Section

| Technique | When to Use | Trade-off |
|-----------|-------------|-----------|
| **Index scan** | Small result set (<5% of rows) | Slower for larger fractions |
| **Bitmap scan** | Multiple conditions, moderate selectivity | Extra CPU for bitmap merge |
| **Sequential scan** | Large portion of table | Reads entire table |
| **Hash join** | Large, unsorted datasets | Memory for hash table |
| **Merge join** | Sorted/pre-indexed data | Sort overhead if not pre-sorted |
| **Nested loop** | One small set, indexed inner | O(n*m) without index |
| **Covering index** | Frequent query pattern | Duplicates data, increases write cost |
| **Materialized view** | Complex but stable queries | Staleness, refresh cost |

| Approach | Offset Pagination | Cursor Pagination |
|----------|------------------|-------------------|
| Latency | Increases with page number | Constant |
| Skips rows on insert | No | No |
| Random page access | Yes | No |
| Implementation | Simple | Requires cursor column |
| Index utilization | Poor for large offsets | Excellent |

## 15. Revision Notes

- Always EXPLAIN ANALYZE before optimizing
- Look for Seq Scan on large tables — this usually needs an index
- Compare estimated row count vs actual row count — large discrepancy = stale stats
- Common causes of bad performance: missing indexes, N+1 queries, no LIMIT, SELECT *
- Cursor pagination over OFFSET for large datasets
- Batch operations in Hibernate: flush periodically, use @Modifying for bulk updates
- Connection pool size: (num_cores * 2) + effective_spindle_count is a starting point
- Indexing strategy should match query patterns, not table structure

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                    QUERY OPTIMIZATION CHEAT SHEET                  |
+-------------------------------------------------------------------+
|                                                                   |
|  DIAGNOSTIC COMMANDS:                                             |
|                                                                   |
|  EXPLAIN (ANALYZE, BUFFERS, TIMING) <query>                      |
|  EXPLAIN (FORMAT JSON) <query>                                    |
|  SHOW pg_stat_statements (top slow queries)                       |
|  SHOW pg_stats (column statistics)                                |
|                                                                   |
+-------------------------------------------------------------------+
|  COMMON PLAN NODES:                                               |
|                                                                   |
|  +------------------+------------------------------------------+ |
|  | Seq Scan         | Full table scan (worst for large tables)  | |
|  | Index Scan       | Index lookup -> heap access               | |
|  | Index Only Scan  | All data in index (fastest)               | |
|  | Bitmap Scan      | Bitmap index + heap access (multi-cond)   | |
|  | Nested Loop      | For each outer->inner (small tables)      | |
|  | Hash Join        | Build hash, probe (large unsorted)        | |
|  | Merge Join       | Merge sorted inputs (pre-sorted data)     | |
|  | Sort             | ORDER BY / DISTINCT / merge join prep     | |
|  | Aggregate        | GROUP BY / scalar aggregation             | |
|  +------------------+------------------------------------------+ |
|                                                                   |
+-------------------------------------------------------------------+
|  OPTIMIZATION CHECKLIST:                                          |
|                                                                   |
|  [ ] Is there an index for the WHERE clause?                     |
|  [ ] Are all needed columns in the index (covering)?             |
|  [ ] Are statistics up to date? (ANALYZE)                        |
|  [ ] Is the join order optimal?                                  |
|  [ ] Are there unnecessary columns in SELECT?                    |
|  [ ] Is pagination using keyset cursor?                          |
|  [ ] Is there an N+1 query problem?                              |
|  [ ] Are functions used on indexed columns?                      |
|  [ ] Is LIMIT present?                                           |
|  [ ] Is the connection pool properly sized?                      |
|                                                                   |
+-------------------------------------------------------------------+
|  HIBERNATE TUNING:                                                |
|                                                                   |
|  spring.jpa.properties.hibernate:                                 |
|    generate_statistics: true                                      |
|    session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 100             |
|    jdbc.batch_size: 50                                            |
|    order_inserts: true                                            |
|    order_updates: true                                            |
|    fetch_size: 100                                                |
|                                                                   |
+-------------------------------------------------------------------+
|  RESPONSE TIME BUDGET:                                            |
|                                                                   |
|  < 10ms    -> Excellent (critical API paths)                     |
|  10-50ms   -> Good (most queries)                                 |
|  50-200ms  -> Acceptable (admin/reporting)                        |
|  200-1000ms-> Needs optimization                                  |
|  > 1s      -> Problematic (alert threshold)                       |
+-------------------------------------------------------------------+
|  THROUGHPUT ESTIMATION (single core):                             |
|                                                                   |
|  1ms query  -> ~1000 QPS                                         |
|  10ms query -> ~100 QPS                                          |
|  100ms query-> ~10 QPS                                           |
|  (scale linearly with cores, accounting for pool contention)      |
+-------------------------------------------------------------------+
