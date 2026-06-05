# Execution Plans

## 1. Executive Summary

An execution plan (query plan) is the sequence of operations a database engine uses to execute a SQL query. The query optimizer generates multiple candidate plans, estimates their cost, and selects the cheapest. Understanding how to read and analyze execution plans is the most important skill for query performance tuning. Every production engineer must master execution plan analysis to diagnose slow queries and verify that optimizations work as expected.

## 2. Core Theory

### 2.1 Plan Nodes

Each operation in a plan is a node. Common node types:

| Node Type | Description |
|-----------|-------------|
| Seq Scan | Full table scan |
| Index Scan | Index lookup + heap access |
| Index Only Scan | All data in index |
| Bitmap Heap/Index Scan | Multi-index combination |
| Nested Loop | For each outer row, probe inner |
| Hash Join | Build hash table, probe |
| Merge Join | Merge sorted inputs |
| Sort | Explicit sorting |
| Aggregate | GROUP BY or scalar aggregation |
| Limit | Stop after N rows |
| Materialize | Cache intermediate result |

### 2.2 Plan Tree Structure

Plans are tree-shaped:
- Inner nodes are join/aggregation/sort operations
- Leaf nodes are table/index access operations
- Data flows upward (from leaves to root)
- Each node's output becomes input to its parent

### 2.3 Cost Metrics

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING, FORMAT JSON) <query>;
```

- **startup cost**: Cost before first row is produced
- **total cost**: Cost to produce all rows
- **plan rows**: Estimated number of rows
- **plan width**: Estimated average row width in bytes
- **actual time**: Actual timing (with ANALYZE)
- **actual rows**: Actual row count (with ANALYZE)
- **loops**: Number of times this node executed (relevant for nested loops)
- **buffers**: Buffer hit/read/dirtied/written

## 3. Under-the-Hood Deep Dive

### 3.1 Cost Calculation

PostgreSQL cost parameters (simplified):

```sql
seq_page_cost = 1.0    -- cost of sequential page read
random_page_cost = 4.0 -- cost of random page read (4x sequential)
cpu_tuple_cost = 0.01  -- cost of processing a row
cpu_index_tuple_cost = 0.005 -- cost of processing an index entry
cpu_operator_cost = 0.0025 -- cost of a WHERE clause evaluation
```

On modern SSDs, `random_page_cost` should be reduced to 1.0-2.0 to reflect much lower random I/O cost.

### 3.2 Plan Optimization Techniques

The optimizer uses:
- **Dynamic programming** for join ordering (Selinger-style algorithm)
- **Genetic query optimizer** (GEQO) for queries with many joins (>12)
- **Cardinality estimation** based on table statistics
- **Predicate selectivity** calculation
- **Unique key detection** for early termination

### 3.3 Statistics and Histograms

```sql
-- View column statistics
SELECT tablename, attname, n_distinct, correlation,
       most_common_vals, most_common_freqs, histogram_bounds
FROM pg_stats
WHERE tablename = 'orders';
```

- **n_distinct**: Estimated distinct values (-1 means all unique)
- **correlation**: Physical ordering correlation (1 = perfectly sorted)
- **most_common_vals**: Top values (for non-uniform distributions)
- **histogram_bounds**: Distribution boundaries for non-MCV values

### 3.4 Parameterized Plan Caching

- PostgreSQL caches plans for simple queries (generic plan after 5 executions)
- Prepared statements may use generic or custom plans
- Generic plan is optimal when parameters don't affect plan choice
- Custom per-execution plan is better when different values need different strategies

## 4. Production Code Examples

### 4.1 Reading an EXPLAIN ANALYZE Output

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING)
SELECT o.id, o.total, u.username
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status = 'PENDING'
ORDER BY o.created_at DESC
LIMIT 20;
```

Output analysis:
```
Limit  (cost=0.56..12.34 rows=20 width=80)
  actual rows=20 loops=1
  ->  Nested Loop  (cost=0.56..1234.56 rows=2345 width=80)
        actual rows=20 loops=1
        ->  Index Scan Backward using idx_orders_created on orders o
              (cost=0.29..567.89 rows=2345 width=40)
              actual rows=20 loops=1
              Index Cond: (status = 'PENDING'::text)
              Buffers: shared hit=15
        ->  Index Scan using users_pkey on users u
              (cost=0.27..0.28 rows=1 width=44)
              actual rows=1 loops=20
              Index Cond: (id = o.user_id)
              Buffers: shared hit=20
```

Key observations:
- Nested Loop with 20 loops on the inner scan (one per order)
- Index Scan Backward avoids explicit sort
- All buffers are "hit" (cached), no disk reads
- Estimate vs actual: 2345 vs 20 rows (due to LIMIT propagating)

### 4.2 Using EXPLAIN in Spring Boot Tests

```java
@SpringBootTest
@TestConstructor(autowireMode = TestConstructor.AutowireMode.ALL)
class QueryPlanTest {

    @Autowired
    EntityManager entityManager;

    @Test
    void verifyQueryPlan() {
        String sql = "EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) " +
                     "SELECT * FROM orders WHERE status = 'PENDING'";

        Query query = entityManager.createNativeQuery(sql);
        String plan = (String) query.getSingleResult();
        System.out.println(plan);

        // Parse JSON to verify nodes don't include Seq Scan
        assertFalse(plan.contains("Seq Scan"));
    }
}
```

### 4.3 Programmatic Plan Analysis

```java
@Component
public class QueryPlanAnalyzer {

    private final JdbcTemplate jdbcTemplate;

    public QueryPlanAnalyzer(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public boolean hasSequentialScan(String sql) {
        String explainSql = "EXPLAIN (FORMAT JSON) " + sql;
        String plan = jdbcTemplate.queryForObject(explainSql, String.class);
        return plan.contains("\"Seq Scan\"") || plan.contains("\"Sequential Scan\"");
    }

    public double getEstimatedCost(String sql) {
        String explainSql = "EXPLAIN (FORMAT JSON) " + sql;
        String plan = jdbcTemplate.queryForObject(explainSql, String.class);
        // Parse JSON and extract total cost
        return extractTotalCost(plan);
    }

    private double extractTotalCost(String plan) {
        // Simplified extraction - in production use JSON parser
        return Double.parseDouble(plan.replaceAll(".*\"Total Cost\": ([0-9.]+).*", "$1"));
    }
}
```

## 5. Real-World Scenarios

### 5.1 Detecting Cardinality Mismatch

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'PENDING';
```

If `actual rows` differs from `plan rows` by >10x, statistics are stale or the distribution is skewed:

```
Index Scan using idx_orders_status on orders
  (cost=0.29..456.34 rows=1000 width=200)
  actual rows=50000 loops=1
  -> This 50x mismatch suggests stale statistics!
```

Fix:
```sql
ANALYZE orders;
```

### 5.2 Identifying Implicit Type Conversions

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = '123';
-- Note: user_id is BIGINT, '123' is TEXT
-- Plan may show a cast operation, preventing index usage
```

Fix: Use correct types in queries and JPA parameter bindings.

### 5.3 Nested Loop vs Hash Join

```sql
-- Bad plan: Nested Loop on large tables
-> Nested Loop (cost=0.00..50000.00 rows=10000)
    -> Seq Scan on orders (cost=0.00..10000.00 rows=10000)
    -> Index Scan on order_items (cost=0.00..4.00 rows=1)

-- Fix: Ensure statistics are updated, or increase work_mem for hash join
SET work_mem = '64MB';
```

### 5.4 Plan Regression Detection

```java
@Service
public class PlanRegressionDetector {

    private final Map<String, Double> baselineCosts = new ConcurrentHashMap<>();

    public boolean checkForRegression(String queryHash, String sql) {
        // Execute EXPLAIN and compare cost to baseline
        double currentCost = analyzeCost(sql);
        double baseline = baselineCosts.getOrDefault(queryHash, Double.MAX_VALUE);

        if (currentCost > baseline * 2) {
            // Plan has regressed significantly
            log.warn("Query plan regression detected for {}: {} -> {}",
                    queryHash, baseline, currentCost);
            return true;
        }
        return false;
    }

    public void saveBaseline(String queryHash, double cost) {
        baselineCosts.put(queryHash, Math.min(
                baselineCosts.getOrDefault(queryHash, Double.MAX_VALUE), cost));
    }
}
```

## 6. Performance

### 6.1 Key Metrics in Execution Plans

| Metric | What It Tells You | Action |
|--------|-------------------|--------|
| Rows Removed by Filter | Index returns too many rows, filtered post-fetch | Add composite index with all WHERE columns |
| Shared Hit vs Read | Cache hit ratio < 99% | Increase shared_buffers, check working set |
| Sort Method: external merge | Sort spilled to disk | Increase work_mem |
| One-Time Filter | Constant false condition | Check query logic |
| SubPlan vs InitPlan | SubPlan runs per-row, InitPlan runs once | Rewrite correlated subquery as join |
| Materialize | CTE/hash created full intermediate | Consider if materialization is needed |

### 6.2 work_mem Tuning

Operations that use memory:
- Sort (ORDER BY, DISTINCT, merge joins)
- Hash tables (hash joins, hash aggregates)
- Bitmap operations

Signs of insufficient work_mem:
```
Sort Method: external merge  Disk: 26152kB
```

### 6.3 Parallel Query Plans

```sql
EXPLAIN (ANALYZE)
SELECT /*+ PARALLEL(o 4) */ COUNT(*) FROM orders WHERE status = 'SHIPPED';
```

Parallel plan indicators:
- `Gather` or `Gather Merge` node
- Multiple workers
- Partial aggregates combined at Gather node

### 6.4 JPA Query Plan Analysis

Enable Hibernate SQL logging:
```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

Use Hibernate QueryPlanCache metrics:
```yaml
spring.jpa.properties.hibernate.query.plan_cache_max_size: 2048
spring.jpa.properties.hibernate.query.plan_parameter_metadata_max_size: 64
```

## 7. Security

### 7.1 EXPLAIN Information Leakage

EXPLAIN output can reveal table structure, row counts, and data distribution. Restrict EXPLAIN to trusted users:

```sql
REVOKE ALL ON FUNCTION pg_stat_statements_reset() FROM PUBLIC;
```

### 7.2 Plan Stability and SQL Injection

Plan caching for prepared statements is generally safe from injection, but ensure:
- Always use parameterized queries (never string concatenation)
- Verify that Hibernate's query plan cache doesn't grow unbounded (monitor its size)

## 8. Common Mistakes

1. **Not using ANALYZE** — EXPLAIN without ANALYZE shows estimates, not reality
2. **Ignoring the loops column** — a nested loop with 100K loops is very expensive
3. **Focusing only on total cost** — actual time matters more
4. **Not checking buffer stats** — high shared_read indicates insufficient caching
5. **Reading plans top-to-bottom** — plans execute bottom-to-top (leaf to root)
6. **Comparing costs across different databases** — costs are optimizer-specific and not comparable between databases
7. **Forgetting that ANALYZE actually executes the query** — be careful with production data
8. **Not using FORMAT JSON** — JSON format preserves more detail and is easier to parse
9. **Analyzing plans without realistic data** — small datasets produce unrealistic plans
10. **Trusting all-in-one cost number** — dig into individual node costs

## 9. Senior Engineer Perspective

### Production Plan Analysis Workflow

1. **Capture**: Log all slow queries (>100ms)
2. **Identify**: Find most expensive queries by total time / frequency
3. **Explain**: Run `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)`
4. **Identify bottleneck node**: Look for nodes with highest actual time
5. **Check buffers**: Verify cache hit ratio per node
6. **Check estimates**: Compare actual rows to plan rows
7. **Implement fix**: Add index, rewrite query, update statistics
8. **Verify**: Re-run EXPLAIN ANALYZE, confirm improvement
9. **Monitor**: Track query performance in production dashboard

### Plan Stability Program

```sql
-- Create plan baseline
SELECT * FROM dblink('dbname=prod', 'EXPLAIN (FORMAT JSON) SELECT ...') AS plan(explain_json text);

-- Compare with current plan
CREATE EXTENSION pg_plan_agg;
SELECT * FROM plan_diff('baseline_plan', 'current_plan');
```

## 10. Interview Questions (20)

### Easy (10)

1. What is an execution plan?
2. What does EXPLAIN do in SQL?
3. What is the difference between EXPLAIN and EXPLAIN ANALYZE?
4. What does a "Seq Scan" node mean?
5. What does an "Index Scan" node mean?
6. What does the "rows" value in EXPLAIN output represent?
7. What is the "cost" in an execution plan?
8. Why might estimated rows differ from actual rows?
9. What is an Index-Only Scan?
10. What does the "loops" column indicate?

### Medium (10)

11. How do you identify a missing index from an EXPLAIN plan?
12. What does "Rows Removed by Filter" mean?
13. Explain the difference between a Nested Loop, Hash Join, and Merge Join in execution plans.
14. How does LIMIT affect the execution plan?
15. What does "Sort Method: external merge Disk" mean?
16. How can you tell if a query is I/O-bound vs CPU-bound from the plan?
17. What does the "Buffers" section of EXPLAIN (ANALYZE, BUFFERS) tell you?
18. How do you detect a correlated subquery in a plan?
19. What is a "Materialize" node and when does it appear?
20. How does the shape of a plan change when parallel query is used?

## 11. Advanced Interview Questions (20)

### Hard (10)

1. How does the optimizer estimate the number of distinct values in a column?
2. Explain the algorithm PostgreSQL uses to calculate join order (dynamic programming vs genetic).
3. What is an "initPlan" vs "subPlan" and how do they differ in execution?
4. How does PostgreSQL's partial index detection work in the optimizer?
5. What is "parameterized path" in the context of nested loop joins?
6. Explain how the optimizer uses functional dependency statistics in PostgreSQL 10+.
7. How does a "bitmap AND" or "bitmap OR" plan node work internally?
8. What is "motion" in a Greenplum/Massively Parallel Processing execution plan?
9. How does the optimizer handle queries with UNION in terms of plan generation?
10. Explain the concept of "tuple routing" in a partitioned table query plan.

### System Design (11-20)

11. Design a system to collect and visualize execution plans from all databases in a fleet.
12. How would you build a query plan comparison tool to detect regressions in CI/CD?
13. Design a recommendation engine that suggests query rewrites based on plan analysis.
14. How would you implement automatic plan analysis in a database monitoring dashboard?
15. Design a system that captures and replays production query workloads in staging for plan testing.
16. How would you design a plan cache for a multi-tenant database proxy?
17. Design a system that automatically applies query hints based on historical plan analysis.
18. How would you implement plan-based query routing to different replica types?
19. Design a distributed query execution plan visualizer for a sharded database.
20. How would you build a plan-driven autoscaler for a database cluster?

## 12. Expert-Level Interview Questions (10)

1. Design an algorithm to automatically detect plan regressions using machine learning on EXPLAIN output features.
2. How would you implement a query optimizer that uses reinforcement learning from actual execution feedback?
3. Describe how to build a plan hashing and fingerprinting system that groups semantically equivalent SQL.
4. How would you design a system that predicts query execution time before running it, using plan features?
5. Explain the internals of how a hash join spills to disk when work_mem is insufficient. How does the database manage partial partitioning?
6. How does PostgreSQL's extended statistics (multi-column, expression, ndistinct) improve cardinality estimation?
7. Design a query plan explainability system that converts plan nodes into natural language explanations for non-experts.
8. How would you implement a cost model for a cloud database where storage and compute are decoupled (e.g., Amazon Aurora)?
9. Describe the algorithm for incremental plan recalculation when only some statistics change.
10. How would you design a plan that uses both row-store and column-store engines for different parts of the same query?

## 13. Debugging & Troubleshooting

### PostgreSQL Plan Diagnostics

```sql
-- Enable plan logging for a session
SET client_min_messages TO LOG;
SET log_planner_stats TO ON;
SET log_executor_stats TO ON;

-- Show most expensive plan nodes from pg_stat_statements
SELECT queryid, query, plans, total_plan_time,
       mean_plan_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_plan_time DESC
LIMIT 10;

-- Visualize plan as text tree
EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE status = 'PENDING';
```

### MySQL Plan Diagnostics

```sql
-- Traditional format
EXPLAIN SELECT * FROM orders WHERE status = 'PENDING';

-- JSON format (more detailed)
EXPLAIN FORMAT=JSON
SELECT * FROM orders o JOIN order_items oi ON o.id = oi.order_id;

-- Extended
EXPLAIN EXTENDED SELECT * FROM orders WHERE status = 'PENDING';
SHOW WARNINGS;
```

### Oracle Plan Diagnostics

```sql
-- Generate plan
EXPLAIN PLAN FOR
SELECT * FROM orders WHERE status = 'PENDING';

-- Display plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- With statistics
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT => 'ALLSTATS LAST'));
```

## 14. Comparison Section

| Database | EXPLAIN Syntax | Key Plan Feature |
|----------|---------------|------------------|
| PostgreSQL | EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) | Detailed cost model, parallel query, JIT |
| MySQL | EXPLAIN FORMAT=JSON | Using index/using where/using filesort |
| Oracle | EXPLAIN PLAN FOR / DBMS_XPLAN | Predicate information, query block names |
| SQL Server | SET STATISTICS PROFILE ON / SET SHOWPLAN_XML ON | Missing index suggestions |
| SQLite | EXPLAIN (bytecode, not cost-based) | Virtual machine opcodes |

| Plan Node | PostgreSQL | MySQL | Oracle | SQL Server |
|-----------|-----------|-------|--------|------------|
| Full scan | Seq Scan | ALL | TABLE ACCESS FULL | Table Scan |
| Index lookup | Index Scan | ref/const | INDEX UNIQUE SCAN | Index Seek |
| Multi-index | Bitmap Scan | index_merge | BITMAP CONVERSION | Index Intersection |
| Hash join | Hash Join | Hash Join | HASH JOIN | Hash Match |
| Sort merge | Merge Join | (no) | MERGE JOIN | Merge Join |
| Nested loop | Nested Loop | Nested Loop | NESTED LOOPS | Nested Loops |

## 15. Revision Notes

- Plans execute bottom-to-top, data flows upward
- Use EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) for production debugging
- Key metrics: actual rows vs estimated, buffers hit/read, loops
- Nested loops are fast for small datasets; hash joins for large unsorted; merge joins for pre-sorted
- Plan regression happens when statistics change or optimizer finds a different path
- Use plan caching for prepared statements but monitor for bloat
- JPA generates SQL — always inspect the actual SQL with hibernate.SQL=DEBUG
- Stale statistics are the most common cause of bad plans

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                   EXECUTION PLAN CHEAT SHEET                      |
+-------------------------------------------------------------------+
|                                                                   |
|  EXPLAIN SYNTAX:                                                  |
|                                                                   |
|  EXPLAIN (ANALYZE, BUFFERS, TIMING, FORMAT JSON) <query>         |
|                                                                   |
|  FORMAT options: TEXT, XML, JSON, YAML                            |
|  ANALYZE: actually executes the query                             |
|  BUFFERS: shows buffer hit/read counts                            |
|  TIMING: per-node timing (default on in PG 14+)                   |
|  VERBOSE: shows full output columns                               |
|                                                                   |
+-------------------------------------------------------------------+
|  PLAN NODE TYPES (PostgreSQL):                                    |
|                                                                   |
|  +---------------------+----------------------------------------+ |
|  | ACCESS METHODS:     |                                         | |
|  |  Seq Scan           | Full table scan (sequential read)       | |
|  |  Index Scan         | B-tree lookup + heap access             | |
|  |  Index Only Scan    | All data in index (no heap access)      | |
|  |  Bitmap Heap Scan   | Bitmap-based bulk heap access           | |
|  |  Bitmap Index Scan  | Build bitmap of matching pages          | |
|  |  Tid Scan           | Direct heap access by tuple ID           | |
|  +---------------------+----------------------------------------+ |
|  | JOIN METHODS:       |                                         | |
|  |  Nested Loop        | For each row in outer, scan inner       | |
|  |  Hash Join          | Build hash, probe                       | |
|  |  Merge Join         | Merge two sorted inputs                 | |
|  +---------------------+----------------------------------------+ |
|  | OTHER:             |                                         | |
|  |  Sort              | ORDER BY / DISTINCT                      | |
|  |  Aggregate         | GROUP BY / scalar aggregation            | |
|  |  Limit             | Stop after N rows                        | |
|  |  Materialize       | Cache intermediate result                | |
|  |  Gather / Gather Merge| Parallel query coordinator            | |
|  |  SubPlan/InitPlan  | Per-row / once subquery execution        | |
|  +---------------------+----------------------------------------+ |
|                                                                   |
+-------------------------------------------------------------------+
|  COMMON WARNING SIGNS:                                            |
|                                                                   |
|  WARNING                         | WHAT IT MEANS                  |
|  --------------------------------+------------------------------ |
|  Seq Scan on large_table         | Missing index                  |
|  actual rows >> plan rows        | Stale statistics (needs ANALYZE)|
|  actual rows << plan rows        | Over-optimistic estimate       |
|  loops = large_number            | Nested loop is expensive       |
|  Sort Method: external merge     | work_mem too small             |
|  Shared read > shared hit        | Low cache hit ratio            |
|  Rows Removed by Filter: high    | Index returns too many rows    |
|  SubPlan (not InitPlan)          | Correlated subquery            |
+-------------------------------------------------------------------+
|  PLAN READING GUIDE (bottom-to-top):                              |
|                                                                   |
|  1. Start at leaf nodes (bottom)                                  |
|  2. Check access method: Seq Scan? Need index?                   |
|  3. Check join method: Nested Loop with high loops? Bad.         |
|  4. Check actual vs estimated rows: >>10x difference? Stale stats.|
|  5. Check buffer info: shared_read > shared_hit? Cache issue.     |
|  6. Check sort method: external merge? Increase work_mem.         |
|  7. Follow data flow up to root node                              |
|  8. Identify highest actual_time node = bottleneck                |
|                                                                   |
+-------------------------------------------------------------------+
|  JPA/HIBERNATE PLAN DEBUGGING:                                    |
|                                                                   |
|  # application.yml                                                |
|  spring.jpa.properties.hibernate:                                 |
|    generate_statistics: true                                      |
|    session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 100             |
|  logging.level.org.hibernate.SQL: DEBUG                           |
|  logging.level.org.hibernate.type.descriptor.sql: TRACE           |
|                                                                   |
+-------------------------------------------------------------------+
