# SQL

---

## What is SQL?

**Structured Query Language (SQL)** is the standard language for relational database management systems. It is **declarative** — you specify **what** you want, not **how** to get it. The database engine's optimizer determines the execution plan. SQL mastery is essential for any backend engineer, particularly when using ORMs like Hibernate that generate SQL automatically.

### Key Concepts:

1. **SQL Sub-Languages**:

   - **DDL (Data Definition Language)** — Schema definition: `CREATE`, `ALTER`, `DROP`, `TRUNCATE`.
   - **DML (Data Manipulation Language)** — Data operations: `SELECT`, `INSERT`, `UPDATE`, `DELETE`.
   - **DCL (Data Control Language)** — Access control: `GRANT`, `REVOKE`.
   - **TCL (Transaction Control)** — Transaction management: `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`.

2. **SELECT Statement Logical Execution Order**:

   ```
   FROM -> JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> DISTINCT -> ORDER BY -> LIMIT/OFFSET
   ```

   This is the **logical** order. The optimizer may reorder operations for performance, but understanding this sequence helps you write correct queries.

3. **Join Types**:

   - **INNER JOIN** — Returns only rows with matches in both tables.
   - **LEFT JOIN** — All rows from the left table, plus matches from the right (NULLs for non-matches).
   - **RIGHT JOIN** — All rows from the right table, plus matches from the left.
   - **FULL JOIN** — All rows from both tables.
   - **CROSS JOIN** — Cartesian product of both tables.
   - **SELF JOIN** — Joining a table with itself (used for hierarchical data).

4. **NULL Handling**:

   NULL is not a value — it represents **unknown** or **missing**. Three-valued logic: `TRUE`, `FALSE`, `UNKNOWN`.

   - `NULL = NULL` yields `NULL` (not `TRUE`). Always use `IS NULL` / `IS NOT NULL`.
   - `NULL IN (1, 2, 3)` yields `NULL`.
   - Aggregate functions like `SUM`, `AVG`, `COUNT(col)` ignore NULLs. `COUNT(*)` counts all rows.

5. **Window Functions**:

   Powerful for analytics without grouping:

   ```sql
   SELECT name, department, salary,
          ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
          AVG(salary) OVER (PARTITION BY department) AS dept_avg,
          LAG(salary) OVER (ORDER BY hire_date) AS prev_salary
   FROM employees;
   ```

---

## Core Concepts

### 1. Query Processing Pipeline

   ```
   SQL Text -> Parser -> Rewriter -> Planner/Optimizer -> Executor -> Result
   ```

   - **Parser** — Tokenizes input, validates syntax, builds a parse tree.
   - **Rewriter** — Applies rules like view expansion and constant folding.
   - **Planner/Optimizer** — Generates candidate execution plans and chooses the lowest-cost one.
   - **Executor** — Executes the plan using operators (sequential scan, index scan, joins, etc.).

### 2. Join Algorithms

   - **Nested Loop Join** — O(n × m). For each row in the outer table, scan the inner table. Good for small tables or when the inner table has an index.
   - **Hash Join** — O(n + m). Build a hash table on one side, probe with the other. Good for large, unsorted datasets.
   - **Merge Join** — O(n + m). Requires sorted inputs. Good for pre-indexed or pre-sorted columns.

### 3. Common Table Expressions (CTEs)

   CTEs improve query readability and can be recursive:

   ```sql
   WITH RECURSIVE org_chart AS (
       SELECT id, name, manager_id, 1 AS level
       FROM employees WHERE manager_id IS NULL
       UNION ALL
       SELECT e.id, e.name, e.manager_id, oc.level + 1
       FROM employees e
       JOIN org_chart oc ON e.manager_id = oc.id
   )
   SELECT * FROM org_chart;
   ```

### 4. Pagination

   - **OFFSET pagination** — Simple but slow for large offsets (the database still reads all skipped rows).
   - **Keyset (cursor-based) pagination** — Uses `WHERE (created_at, id) < (?, ?)` for efficient, constant-time pagination.

### 5. Batch Operations in Spring Boot

   ```java
   @Transactional
   public void batchInsertUsers(List<User> users) {
       int batchSize = 50;
       for (int i = 0; i < users.size(); i++) {
           entityManager.persist(users.get(i));
           if (i > 0 && i % batchSize == 0) {
               entityManager.flush();
               entityManager.clear();
           }
       }
   }
   ```

---

## Common Mistakes

1. **Using `SELECT DISTINCT` to hide missing JOIN conditions** — Often indicates a Cartesian product. Always verify JOIN conditions are correct before using DISTINCT.

2. **Forgetting NULL handling** — `WHERE col NOT IN (1, 2, 3)` excludes rows where `col IS NULL` because `NULL NOT IN (1, 2, 3)` yields `NULL`, not `TRUE`.

3. **Missing indexes on foreign keys** — JOINs on foreign key columns without indexes cause sequential scans.

4. **Using functions in WHERE clauses** — `WHERE YEAR(date) = 2024` prevents index usage. Use `WHERE date BETWEEN '2024-01-01' AND '2024-12-31'` instead.

5. **Assuming implicit row ordering** — Without `ORDER BY`, the order of results is undefined and may vary between executions.

6. **Using correlated subqueries when JOINs suffice** — Correlated subqueries execute once per outer row. A JOIN is almost always more efficient.

---

## Real-World Scenarios

### 1. Analytics Dashboard with Window Functions

A SaaS platform needs a monthly report showing each employee's salary, department average, and rank. Without window functions, this requires multiple subqueries. A single query handles it efficiently:

```sql
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees WHERE active = TRUE;
```

### 2. Keyset Pagination for Social Media Feed

A social media app loads 20 posts at a time. `OFFSET 100000 LIMIT 20` reads and discards 100K rows. Keyset pagination uses a WHERE clause on the last cursor for constant-time pagination:

```sql
SELECT id, content, created_at FROM posts
WHERE (created_at, id) < ('2024-06-01T00:00:00', 5000)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

### 3. Recursive CTE for Organizational Hierarchy

An HR system needs the full reporting chain for any employee. Recursive CTEs traverse the tree in a single query handling arbitrary depth:

```sql
WITH RECURSIVE org_chain AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees WHERE id = 123
    UNION ALL
    SELECT e.id, e.name, e.manager_id, oc.depth + 1
    FROM employees e
    JOIN org_chain oc ON e.manager_id = oc.id
)
SELECT * FROM org_chain ORDER BY depth;
```

## Scenario-Based Questions

1. **Q: You are building a billing system that generates monthly invoices. A single customer can have thousands of transactions. You need to compute the total per customer, apply tiered discounts, and insert results into an invoices table. How do you write this efficiently?**
   A: Use a single INSERT...SELECT with aggregations and a CASE expression for tiered discounts: `WITH customer_totals AS (SELECT customer_id, SUM(amount) AS total FROM transactions WHERE date >= $1 AND date < $2 GROUP BY customer_id) INSERT INTO invoices SELECT customer_id, total * CASE WHEN total > 10000 THEN 0.9 ELSE 1.0 END FROM customer_totals`. Process in batches to avoid long-running transactions.

2. **Q: Your reporting query joins 6 tables and takes 45 seconds. The CEO needs a dashboard that refreshes in under 5 seconds. You cannot change the schema. What do you do?**
   A: Check the execution plan for full table scans, nested loops with many loops, and sort spills. Add missing indexes on join and filter columns. Use a materialized view that pre-joins the data and refreshes periodically. If data can be slightly stale, use a read-only replica. As a last resort, cache results in Redis with a TTL.

3. **Q: A DELETE removing 5M rows from a 50M row table takes 30 minutes and blocks other queries. How do you make it non-blocking and faster?**
   A: Batch the delete in chunks of 1000 rows with `LIMIT` inside a loop, adding `pg_sleep(0.1)` between batches: `DELETE FROM table WHERE id IN (SELECT id FROM table WHERE condition LIMIT 1000)`. This holds short-lived locks per batch. Consider partitioning if you can drop entire partitions.

4. **Q: Your application team uses `SELECT * FROM users WHERE email = $1` and it's fast. But a new query `SELECT * FROM users WHERE email LIKE '%@gmail.com'` is extremely slow despite an index on email. Why?**
   A: B-Tree indexes only support prefix patterns (`LIKE 'pattern%'`). `LIKE '%pattern'` cannot use a B-Tree. Use a trigram GIN index: `CREATE INDEX ON users USING GIN (email gin_trgm_ops)`. This enables efficient wildcard searches on both sides.

5. **Q: A `LEFT JOIN` returns 3x more rows than expected for a one-row-per-customer report. What happened?**
   A: The right table has multiple matching rows per left row, causing row multiplication. Use `DISTINCT ON (customer.id)` with proper ORDER BY to pick the desired match, or aggregate the right side: `LEFT JOIN (SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id)`.

6. **Q: Your yearly cleanup runs `DELETE FROM logs WHERE created_at < NOW() - INTERVAL '1 year'`. It works for months but suddenly fails with "out of disk space" on the WAL. Why?**
   A: The DELETE generates massive WAL in a single transaction. Batch in smaller transactions (10K rows each), committing after each batch. Each commit recycles WAL. Add `pg_sleep(0.1)` between batches to allow autovacuum to keep up.

7. **Q: A query with multiple CTEs runs slower than an equivalent subquery version. You expected CTEs to be faster. What's happening?**
   A: In PostgreSQL, CTEs are optimization fences — the optimizer cannot push predicates through CTEs. The CTE is materialized fully before the outer query runs. Use `NOT MATERIALIZED` (PostgreSQL 12+) to inline the CTE, or rewrite as subqueries.

8. **Q: A table has 100 columns. `SELECT *` for a paginated list takes 200ms. Selecting only 3 columns takes 20ms. Why such a big difference?**
   A: `SELECT *` reads all 100 columns from disk, using more I/O and bandwidth. The 10x difference suggests the table is wider than a single page per row. Always select only needed columns. In JPA, use DTO projections for read-only queries.

9. **Q: Your query uses `WHERE status IN (SELECT status FROM allowed_statuses)`. The subquery returns 3 values but is slow. How would you rewrite it?**
   A: The subquery may be evaluated per row (correlated). Rewrite as a JOIN: `SELECT t.* FROM table t JOIN allowed_statuses a ON t.status = a.status`. If `allowed_statuses` is small, use a hardcoded IN list. Check the plan for SubPlan vs InitPlan.

10. **Q: A batch INSERT of 10K rows into a table with 5 indexes takes 10 seconds. A single-row INSERT takes 2ms. Why doesn't it scale linearly?**
    A: Each index has O(log n) insert cost. Without batching, each INSERT is a separate transaction with commit overhead. Use multi-row INSERT (`INSERT INTO t VALUES (...), (...), (...)`), batch in a single transaction, and configure `hibernate.jdbc.batch_size` in JPA.

## Interview Questions

1. **What is the logical execution order of a SELECT statement?**
   A: FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET. The optimizer may reorder operations physically.

2. **What is the difference between INNER JOIN and LEFT JOIN?**
   A: INNER JOIN returns only rows with matches in both tables. LEFT JOIN returns all rows from the left table, with NULLs where the right has no match.

3. **How does NULL behave in SQL?**
   A: NULL represents unknown. `NULL = NULL` yields NULL, not TRUE. Use `IS NULL` not `= NULL`. Aggregate functions ignore NULLs except `COUNT(*)`. `WHERE col NOT IN (1,2,3)` excludes NULL rows.

4. **What is the difference between UNION and UNION ALL?**
   A: UNION removes duplicates (requires a sort), UNION ALL keeps all rows. UNION ALL is faster when duplicates are acceptable.

5. **What is a correlated subquery?**
   A: A subquery that references columns from the outer query and executes once per outer row. Usually less efficient than a JOIN with GROUP BY.

6. **How do window functions differ from GROUP BY?**
   A: Window functions perform calculations across related rows without collapsing them. GROUP BY collapses groups into single rows. Window functions use OVER (PARTITION BY ...).

7. **What is the difference between HAVING and WHERE?**
   A: WHERE filters rows before GROUP BY. HAVING filters groups after GROUP BY. HAVING can use aggregate functions; WHERE cannot.

8. **How do you paginate efficiently in SQL?**
   A: Keyset pagination uses `WHERE id > :last_id ORDER BY id LIMIT :size` — O(1) per page. OFFSET pagination is O(n) because skipped rows are read internally.

9. **What is a CTE and when is it useful?**
   A: A Common Table Expression (WITH clause) defines a named temporary result set. Useful for recursive queries, breaking complex queries into steps, and reusing subquery results.

10. **How do you safely interpolate user input in SQL?**
    A: Use parameterized queries. Never concatenate user input into SQL strings. In JPA use `:param` with @Param. In JDBC use PreparedStatement placeholders.

## Developer Recommendations

- **Always use explicit column lists instead of SELECT *** — `SELECT *` reads all columns, increasing I/O, network transfer, and memory. Changes to schema with `SELECT *` may break application code. Explicit columns enable index-only scans.

- **Prefer JOINs over correlated subqueries** — Correlated subqueries execute once per outer row (O(n × m)). A JOIN with GROUP BY does the same work in a single pass. Exception: when the subquery returns a single row and the filter is highly selective.

- **Use keyset pagination instead of OFFSET for deep pages** — OFFSET 100000 reads 100K rows internally. Keyset pagination uses a WHERE clause on the last-seen cursor for constant time. Trade-off: requires a unique sort column and cannot jump to arbitrary pages.

- **Batch DML operations in transactions** — Each individual DML outside a transaction issues an implicit commit. Batching 1000 rows in a single transaction improves throughput 10-100x. In JPA, configure `hibernate.jdbc.batch_size`.

- **Avoid functions on indexed columns in WHERE clauses** — `WHERE YEAR(date) = 2024` prevents index usage. Use `WHERE date >= '2024-01-01' AND date < '2025-01-01'`. For LOWER(), create a functional index.

- **Use DISTINCT sparingly** — DISTINCT often masks a missing JOIN condition creating a Cartesian product. Verify JOIN correctness before adding DISTINCT.

- **Handle NULLs explicitly in WHERE clauses** — `WHERE col NOT IN (1,2,3)` silently excludes rows where col IS NULL. Use `WHERE (col NOT IN (1,2,3) OR col IS NULL)`.
