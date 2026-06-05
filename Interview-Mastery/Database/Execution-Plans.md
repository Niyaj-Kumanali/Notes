# Execution Plans

---

## What is an Execution Plan?

An **execution plan** (query plan) is the sequence of operations a database engine uses to execute a SQL query. The query optimizer generates multiple candidate plans, estimates their cost, and selects the cheapest. Understanding how to read and analyze execution plans is the most important skill for query performance tuning. Every production engineer must master plan analysis to diagnose slow queries and verify optimizations.

### Key Concepts:

1. **Plan Node Types**:

   - **Seq Scan** — Full table scan (sequential read of all pages).
   - **Index Scan** — B-tree lookup + heap access to fetch row data.
   - **Index Only Scan** — All needed columns are in the index; no heap access.
   - **Bitmap Scan** — Build a bitmap of matching page locations, then fetch heap pages in order. Used for multi-condition queries.
   - **Nested Loop** — For each row in outer, scan inner table.
   - **Hash Join** — Build hash table on inner, probe with outer.
   - **Merge Join** — Merge two sorted inputs.
   - **Sort** — Explicit sorting for ORDER BY, DISTINCT, or merge join prep.
   - **Aggregate** — GROUP BY or scalar aggregation.
   - **Limit** — Stop after N rows.
   - **Gather** — Parallel query coordinator node.

2. **Plan Tree Structure**:

   Plans are tree-shaped:
   - Inner nodes are join/aggregation/sort operations (internal nodes).
   - Leaf nodes are table/index access operations.
   - Data flows upward — from leaves (bottom) to root (top).
   - Each node's output becomes the input to its parent.

3. **Cost Metrics**:

   - **startup cost** — Cost before first row is produced.
   - **total cost** — Cost to produce all rows.
   - **plan rows** — Estimated number of rows.
   - **plan width** — Estimated average row width in bytes.
   - **actual time** — Actual timing (requires ANALYZE).
   - **actual rows** — Actual row count (requires ANALYZE).
   - **loops** — How many times this node executed (critical for nested loop costs).
   - **buffers** — Buffer hit/read/dirtied/written counts.

---

## Core Concepts

### 1. Reading an EXPLAIN ANALYZE Output

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
   - Nested Loop with **20 loops** on the inner scan (one per order).
   - Index Scan Backward avoids explicit sort.
   - All buffers are **"hit"** (cached), no disk reads.
   - Estimate vs actual: 2345 vs 20 rows (LIMIT propagates estimate).

### 2. Detecting Cardinality Mismatch

   If `actual rows` differs from `plan rows` by >10x, statistics are stale or distribution is skewed:

   ```
   Index Scan using idx_orders_status on orders
     (cost=0.29..456.34 rows=1000 width=200)
     actual rows=50000 loops=1
     → This 50x mismatch suggests stale statistics!
   ```

   Fix:
   ```sql
   ANALYZE orders;
   ```

### 3. Identifying Plan Regression

   ```java
   @Service
   public class PlanRegressionDetector {
       private final Map<String, Double> baselineCosts = new ConcurrentHashMap<>();

       public boolean checkForRegression(String queryHash, String sql) {
           double currentCost = analyzeCost(sql);
           double baseline = baselineCosts.getOrDefault(queryHash, Double.MAX_VALUE);
           if (currentCost > baseline * 2) {
               log.warn("Plan regression detected: {} -> {}", baseline, currentCost);
               return true;
           }
           return false;
       }
   }
   ```

### 4. work_mem Tuning

   Operations that use memory:
   - **Sort** (ORDER BY, DISTINCT, merge joins)
   - **Hash tables** (hash joins, hash aggregates)
   - **Bitmap operations**

   Sign of insufficient `work_mem`:
   ```
   Sort Method: external merge  Disk: 26152kB
   ```

### 5. Parallel Query Plans

   ```sql
   EXPLAIN (ANALYZE) SELECT COUNT(*) FROM orders WHERE status = 'SHIPPED';
   ```

   Parallel plan indicators:
   - `Gather` or `Gather Merge` node.
   - Multiple workers.
   - Partial aggregates combined at Gather node.

### 6. Key Plan Warning Signs

   | Warning | What It Means |
   |---------|---------------|
   | Seq Scan on large table | Missing index |
   | actual rows >> plan rows | Stale statistics (needs ANALYZE) |
   | actual rows << plan rows | Over-optimistic cardinality estimate |
   | loops = large number | Nested loop is expensive |
   | Sort Method: external merge | work_mem too small |
   | Shared read > shared hit | Low cache hit ratio |
   | Rows Removed by Filter: high | Index returns too many rows |
   | SubPlan (not InitPlan) | Correlated subquery |

### 7. Plan Diagnostics Commands

   ```sql
   -- PostgreSQL: top queries by total execution time
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

   -- Find queries waiting for locks
   SELECT pid, wait_event_type, wait_event, query, state
   FROM pg_stat_activity
   WHERE wait_event IS NOT NULL;
   ```

---

## Common Mistakes

1. **Not using ANALYZE** — EXPLAIN without ANALYZE shows estimates, not reality.
2. **Ignoring the loops column** — a nested loop with 100K loops is very expensive.
3. **Focusing only on total cost** — actual time matters more.
4. **Not checking buffer stats** — high shared_read indicates insufficient caching.
5. **Reading plans top-to-bottom** — plans execute bottom-to-top (leaf to root).
6. **Comparing costs across different databases** — costs are optimizer-specific.
7. **Forgetting that ANALYZE actually executes the query** — be careful in production.
8. **Not using FORMAT JSON** — JSON format preserves more detail and is easier to parse programmatically.
9. **Analyzing plans without realistic data** — small datasets produce unrealistic plans.
10. **Trusting the all-in-one cost number** — dig into individual node costs.

---

## Real-World Scenarios

### 1. Diagnosing a Slow E-Commerce Query

A dashboard query joining orders, customers, and products takes 30 seconds instead of 200ms. EXPLAIN ANALYZE reveals the root cause:

```
Nested Loop  (cost=0.29..45678.90 rows=500 width=120)
  actual rows=50000 loops=1
  ->  Seq Scan on orders  (cost=0.00..12345.60 rows=50000 width=40)
        actual rows=50000 loops=1
        Filter: (status = 'PENDING'::text)
  ->  Index Scan using users_pkey on users
        (cost=0.29..0.67 rows=1 width=44)
        actual rows=1 loops=50000
```

Key findings: estimate says 500 rows, actual says 50000 (stale statistics). Loops=50000 on the inner index scan confirms the nested loop is the bottleneck.

### 2. Detecting Disk Spills in Reporting

A monthly report sorts 10M rows. The execution plan shows a sort spilling to disk:

```
Sort  (cost=456789.01..478901.23 rows=8848891 width=247)
  Sort Key: created_at DESC, total DESC
  Sort Method: external merge  Disk: 523472kB
  ->  Seq Scan on orders  (cost=0.00..234567.89 rows=8848891 width=247)
```

The 500MB disk spill indicates `work_mem` is too low. Increasing it from 4MB to 256MB keeps the sort in memory, reducing query time from 5 minutes to 15 seconds.

### 3. Plan Regression Detection

A daily batch query suddenly takes 2x longer. The deployment log shows no code changes. EXPLAIN ANALYZE reveals a different plan:

```
-- Current plan: Nested Loop (cost=5000...)
-- Baseline plan: Hash Join (cost=300...)
-- The optimizer switched from hash join to nested loop due to changed statistics
```

Fix: run ANALYZE to refresh statistics. Use pg_hint_plan to pin the hash join plan if the issue persists.

## Scenario-Based Questions

1. **Q: You are debugging a query that became 10x slower after a data load. EXPLAIN ANALYZE shows `actual rows=1000000` but `plan rows=5000`. The query uses an index scan. How do you fix it?**
   A: 200x cardinality mismatch — statistics are stale after the bulk load. Run `ANALYZE orders`. If the issue recurs, increase `default_statistics_target` from 100 to 500-1000. Create extended statistics (`CREATE STATISTICS`) for correlated columns where the optimizer assumes independence.

2. **Q: An EXPLAIN ANALYZE shows a nested loop with `loops=200000` on the inner index scan. Each loop returns 1 row in 0.05ms. Total time is 10 seconds. How do you optimize?**
   A: 200K index probes at 0.05ms each = 10,000ms. The optimizer chose nested loop because per-loop cost is low, but iteration count is high. Force a hash join by increasing `work_mem` or using `enable_nestloop = off`. A hash join does one build (O(n)) + probe (O(m)).

3. **Q: A plan shows `Sort Method: external merge Disk: 256MB`. The query runs hourly. Increasing work_mem globally could cause memory issues. How do you fix this specific query?**
   A: Use per-query work_mem: `SET work_mem = '256MB'` before the query, `RESET work_mem` after. In PostgreSQL 13+, use `pg_hint_plan` for per-query settings. Better: add an index on the sort columns to avoid the sort entirely.

4. **Q: A parallel query plan shows `Workers Planned: 4` but `Workers Launched: 0`. The query runs on a 64-core server with plenty of I/O. Why aren't workers being used?**
   A: Check `max_parallel_workers_per_gather` (may be too low). The query may be too short for parallelism to be beneficial (startup overhead). The query may involve non-parallelizable operations (writes, some aggregates). Check the table's `parallel_workers` storage parameter.

5. **Q: A plan shows `Seq Scan on orders (cost=0.00..450000.00 rows=10000000 width=200)` with `WHERE status = 'PENDING'`. The `status` column has a B-Tree index. Why does PostgreSQL choose a sequential scan?**
   A: If 30%+ of rows have status='PENDING', the optimizer estimates random I/O from an index scan exceeds seq scan cost. Check actual selectivity: `SELECT status, COUNT(*) FROM orders GROUP BY status`. If 'PENDING' is common, the optimizer is correct. If rare but statistics don't show it, run ANALYZE.

6. **Q: An index scan shows `Rows Removed by Filter: 90000` and `actual rows: 10`. The query returns 10 rows out of 90010 index entries. How do you make this more efficient?**
   A: The index isn't selective enough. Create a composite index covering all filter columns. If WHERE has `status = 'ACTIVE' AND created_at > '2024-01-01'`, create `(status, created_at)` so both conditions are evaluated during the index scan, not as post-filter.

7. **Q: EXPLAIN ANALYZE shows `Shared read: 50000` and `Shared hit: 100`. The query runs every 5 minutes on the same data. Why is the cache hit ratio so low?**
   A: 0.2% cache hit ratio means the working set doesn't fit in `shared_buffers`. Increase to 25% of total RAM. The data may be cold — ensure frequent enough access to stay cached. Consider a covering index to reduce data volume read from disk.

8. **Q: Two identical queries on the same database produce different plans on different days. Query A uses hash join (500ms), Query B uses nested loop (30s). How do you stabilize this?**
   A: Plan regression from changing statistics. Save the good plan's cost and compare. Create extended statistics on correlated columns. Use `pg_hint_plan` to pin the hash join plan. Use `plan_hash` from `pg_stat_statements` for automated regression detection.

9. **Q: EXPLAIN (ANALYZE, BUFFERS) shows every node has `Buffers: shared hit=0 shared read=N`. What does this imply and is it always bad?**
   A: Zero cache hits means data isn't cached. For the first query after restart or on cold data, this is normal. For frequent queries, this indicates `shared_buffers` is too small. However, `shared read` doesn't always mean physical I/O — the OS filesystem cache may serve it. Check `pg_stat_database` for the full picture.

10. **Q: A plan shows `SubPlan 1 -> Seq Scan on audit_log` inside a nested loop. The outer query returns 100K rows. The query runs for 5 minutes. What is this pattern and how do you fix it?**
    A: This is a correlated subquery — SubPlan executes once per outer row (100K times). Rewrite as a JOIN: `SELECT o.*, COALESCE(a.cnt, 0) FROM orders o LEFT JOIN (SELECT order_id, COUNT(*) AS cnt FROM audit_log GROUP BY order_id) a ON a.order_id = o.id`. One aggregation instead of 100K subqueries.

## Interview Questions

1. **What is an execution plan?**
   A: The sequence of operations a database engine uses to execute a SQL query. The optimizer generates candidate plans and selects the cheapest based on cost estimates.

2. **What is the difference between EXPLAIN and EXPLAIN ANALYZE?**
   A: EXPLAIN shows estimated costs only (doesn't execute). EXPLAIN ANALYZE executes and shows actual timings, row counts, and buffer usage.

3. **What does a Seq Scan indicate?**
   A: Full table scan. Appropriate when: table is small, query matches large % of rows (>5-10%), or no suitable index exists. Can also indicate a missing index.

4. **What does the loops column mean?**
   A: How many times a node was executed. A nested loop with 50,000 loops means the inner node ran 50,000 times. High loop counts make nested loops expensive.

5. **How do you interpret a cardinality mismatch?**
   A: Compare `plan rows` (estimated) to `actual rows`. Mismatch >10x indicates stale statistics or correlated columns. Fix with ANALYZE or extended statistics.

6. **What is the difference between Index Scan and Index Only Scan?**
   A: Index Scan reads index then fetches row from heap (2 I/Os). Index Only Scan reads all data from the index alone. Covering indexes enable Index Only Scans.

7. **What does Sort Method: external merge mean?**
   A: The sort exceeded `work_mem` and spilled to disk. External merge is ~100x slower than in-memory. Increase `work_mem` or add an index to avoid the sort.

8. **How do bitmap scans differ from regular index scans?**
   A: Bitmap scans build a page-level bitmap of matching rows, then fetch heap pages in order. Reduces random I/O. PostgreSQL combines multiple bitmap scans with AND/OR for complex conditions.

9. **What is a parallel query plan?**
   A: Uses multiple worker processes to execute parts of a query in parallel. Identified by `Gather` or `Gather Merge` nodes. Workers appear in `Workers Planned`/`Workers Launched`.

10. **How do you detect plan regression?**
    A: Compare EXPLAIN ANALYZE output over time. Use `pg_stat_statements` to track changes in execution time and row estimates. Tools like `pg_plan_advice` or custom baselines flag regressions.

## Developer Recommendations

- **Always use EXPLAIN ANALYZE (with BUFFERS) for real diagnosis** — Plain EXPLAIN shows estimates that can be wildly inaccurate. ANALYZE shows actual rows, times, and buffer usage. Never optimize based on estimates alone.

- **Read plans bottom-to-top, not top-to-bottom** — Data flows from leaf nodes (table/index access) to the root node (final result). Start at the bottom to understand data access, then work up.

- **Pay attention to the loops column** — A nested loop with 100K loops is almost always the bottleneck, even if each loop is fast. Total cost = loops × per-loop cost.

- **Compare actual rows to plan rows for cardinality mismatches** — Mismatch >10x means the optimizer works with bad estimates. This is the most common cause of bad plans.

- **Monitor shared_hit vs shared_read for cache efficiency** — Low cache hit ratio indicates the working set doesn't fit in memory. Increase `shared_buffers` or optimize queries to access less data.

- **Save EXPLAIN ANALYZE baselines for critical queries** — When a query runs well, save its plan. Compare when it degrades to identify plan regression. Automate with monitoring tools.

- **Use FORMAT JSON for programmatic plan analysis** — JSON format preserves all plan detail and is easier to parse. Tools like explain.depesz.com and pgMustard accept JSON format.
