# SQL

---

## What is SQL?

- **Definition**
  - **Structured Query Language (SQL)** is the standard language for relational database management systems. It is **declarative** — you specify **what** you want, not **how** to get it, and the database engine's optimizer determines the execution plan. SQL mastery is essential for any backend engineer, particularly when using ORMs like Hibernate that generate SQL automatically.

### SQL Sub-Languages

- **DDL (Data Definition Language)** — Handles schema definition with commands like `CREATE`, `ALTER`, `DROP`, and `TRUNCATE`. These statements define the structure of database objects.
- **DML (Data Manipulation Language)** — Handles data operations with `SELECT`, `INSERT`, `UPDATE`, and `DELETE`. These are the most frequently used SQL commands in application code.
- **DCL (Data Control Language)** — Manages access control with `GRANT` and `REVOKE`. Controls which users can read or modify database objects.
- **TCL (Transaction Control)** — Manages transaction boundaries with `BEGIN`, `COMMIT`, `ROLLBACK`, and `SAVEPOINT`. These commands ensure data integrity during multi-step operations.

### SELECT Statement Logical Execution Order

```
FROM -> JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> DISTINCT -> ORDER BY -> LIMIT/OFFSET
```

- This is the **logical** order that defines which data is visible at each stage. The optimizer may reorder operations for performance, but understanding this sequence helps you write correct and predictable queries.

### Join Types

- **INNER JOIN** — Returns only rows with matches in both tables. Rows without a match on either side are excluded from the result set.
- **LEFT JOIN** — Returns all rows from the left table, with matching values from the right table (or NULLs where no match exists). The most commonly used outer join type.
- **RIGHT JOIN** — Returns all rows from the right table, with matching values from the left table. Functionally equivalent to a LEFT JOIN with the table order reversed.
- **FULL JOIN** — Returns all rows from both tables, with NULLs where no match exists on either side. Less common but useful for comparing two complete datasets.
- **CROSS JOIN** — Produces the Cartesian product of both tables, combining every row from the first table with every row from the second. Use cautiously as it can generate enormous result sets.
- **SELF JOIN** — Joining a table with itself, typically using aliases to distinguish roles. Used for hierarchical data like employee-manager relationships.

### NULL Handling

- **NULL Semantics** — NULL is not a value but represents **unknown** or **missing**. Three-valued logic applies: comparisons with NULL yield `UNKNOWN`, not `TRUE` or `FALSE`.
- **NULL Comparisons** — `NULL = NULL` yields `NULL` (not `TRUE`). Always use `IS NULL` or `IS NOT NULL` for NULL checks. `NULL IN (1, 2, 3)` also yields `NULL`.
- **NULL in Aggregates** — Aggregate functions like `SUM`, `AVG`, and `COUNT(col)` ignore NULLs. `COUNT(*)` counts all rows regardless of NULLs. Be aware of this when computing averages over sparse data.

### Window Functions

- Window functions are powerful for analytics without collapsing rows:

```sql
SELECT name, department, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg,
       LAG(salary) OVER (ORDER BY hire_date) AS prev_salary
FROM employees;
```

- Window functions perform calculations across related rows without grouping them into a single output row. The `OVER` clause defines the window frame, and `PARTITION BY` creates independent groups.

---

## Core Concepts

### Query Processing Pipeline

```
SQL Text -> Parser -> Rewriter -> Planner/Optimizer -> Executor -> Result
```

- **Parser** — Tokenizes the SQL input, validates syntax, and builds a parse tree representing the query structure. Syntax errors are detected at this stage.
- **Rewriter** — Applies semantic rules like view expansion (replacing view references with their defining queries) and constant folding (evaluating constant expressions at plan time).
- **Planner/Optimizer** — Generates multiple candidate execution plans and estimates their costs using table statistics. The cheapest plan is selected for execution.
- **Executor** — Executes the chosen plan using physical operators (sequential scan, index scan, various join algorithms). The executor processes data bottom-up through the plan tree.

### Join Algorithms

- **Nested Loop Join** — O(n × m) complexity. For each row in the outer table, scan the inner table. Best for small tables or when the inner table has an efficient index for lookups.
- **Hash Join** — O(n + m) complexity. Builds a hash table on one side of the join and probes it with the other side. Ideal for large, unsorted datasets where no index exists.
- **Merge Join** — O(n + m) complexity with sorted inputs. Combines two sorted lists by advancing through both simultaneously. Excellent for pre-indexed or pre-sorted columns.

### Common Table Expressions (CTEs)

- CTEs improve query readability and support recursion:

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

- Recursive CTEs consist of a base case (non-recursive term) and a recursive step (referencing the CTE itself). They enable tree and graph traversal in a single query.

### Pagination

- **OFFSET Pagination** — Simple to implement but slow for large offsets because the database still reads and discards all skipped rows internally. Performance degrades linearly with page depth.
- **Keyset (Cursor-Based) Pagination** — Uses `WHERE (created_at, id) < (?, ?)` for efficient, constant-time pagination. Requires a unique sort column and cannot jump to arbitrary pages.

### Batch Operations in Spring Boot

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

- Flushing and clearing the persistence context periodically prevents OOM and reduces transaction size. Each batch of 50 is flushed to the database while the remaining entities stay managed.

---

## Common Mistakes

- **Using `SELECT DISTINCT` to hide missing JOIN conditions**
  - Often indicates an unintended Cartesian product from a missing join condition. Always verify JOIN conditions are correct before resorting to DISTINCT.
  - **Why it looks correct:** The result set appears deduplicated, and the underlying row multiplication is invisible without examining the row count before and after DISTINCT.

- **Forgetting NULL handling**
  - `WHERE col NOT IN (1, 2, 3)` excludes rows where `col IS NULL` because `NULL NOT IN (1, 2, 3)` yields `NULL`, not `TRUE`. Always consider NULL behavior in exclusion queries.
  - **Why it looks correct:** The query returns results for the non-NULL values, and the silent exclusion of NULL rows only surfaces when someone counts the total.

- **Missing indexes on foreign keys**
  - JOINs on foreign key columns without indexes cause sequential scans on the referenced table. This is one of the most common and impactful query performance issues.
  - **Why it looks correct:** The query works and returns correct results — the sequential scan is invisible until the table grows or concurrent load increases.

- **Using functions in WHERE clauses**
  - `WHERE YEAR(date) = 2024` prevents index usage on the `date` column. Use sargable predicates like `WHERE date BETWEEN '2024-01-01' AND '2024-12-31'` instead.
  - **Why it looks correct:** The query returns the right results; the full table scan is hidden from the developer who never checks the execution plan.

- **Assuming implicit row ordering**
  - Without an explicit `ORDER BY`, the order of results is undefined and may vary between executions. Never rely on insertion order for query results.
  - **Why it looks correct:** Results often appear in insertion order during development on small datasets, creating a false sense of predictability that breaks under parallel inserts or after a VACUUM.

- **Using correlated subqueries when JOINs suffice**
  - Correlated subqueries execute once per outer row, leading to O(n × m) complexity. A JOIN with GROUP BY is almost always more efficient.
  - **Why it looks correct:** The subquery is concise and performs acceptably on small datasets; the O(n×m) explosion only becomes visible when the outer table grows into the thousands of rows.

---

## Real-World Scenarios

### Analytics Dashboard with Window Functions

- A SaaS platform needs a monthly report showing each employee's salary, department average, and rank. Without window functions, this requires multiple subqueries. A single query handles it efficiently:

```sql
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees WHERE active = TRUE;
```

### Keyset Pagination for Social Media Feed

- A social media app loads 20 posts at a time. `OFFSET 100000 LIMIT 20` reads and discards 100K rows. Keyset pagination uses a WHERE clause on the last cursor for constant-time pagination:

```sql
SELECT id, content, created_at FROM posts
WHERE (created_at, id) < ('2024-06-01T00:00:00', 5000)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

### Recursive CTE for Organizational Hierarchy

- An HR system needs the full reporting chain for any employee. Recursive CTEs traverse the tree in a single query handling arbitrary depth:

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

## Use Cases

Reach for SQL whenever you need to interact with relational data — it is the universal language for structured data storage and retrieval.

- **Data retrieval via SELECT** — Fetch, filter, join, and aggregate data from one or more tables with WHERE, JOIN, GROUP BY, and HAVING.
  - The most frequent operation in backend applications.
  - **Avoid when:** you need simple key-value lookups — NoSQL databases may be simpler.

- **Data modification via INSERT/UPDATE/DELETE** — Add, change, or remove rows while maintaining referential integrity through foreign keys.
  - Always wrap multi-step modifications in transactions.
  - **Avoid when:** the schema is highly dynamic or unstructured — consider a document or wide-column store.

- **Schema design and DDL** — CREATE, ALTER, and DROP define tables, indexes, constraints, and relationships that enforce data integrity at the database level.
  - **Avoid when:** the schema changes multiple times daily — schema-less databases reduce migration overhead.

- **Reporting and analytical queries** — Window functions, CTEs, and aggregation produce business reports, dashboards, and data exports.
  - **Avoid when:** data volume is in petabytes — dedicated OLAP engines (e.g., ClickHouse, Snowflake) may be more appropriate.

- **Backend API data access** — ORMs like Hibernate/JPA generate SQL from object mappings; understanding SQL is essential for debugging generated queries and fixing N+1 problems.
  - **Avoid when:** the endpoint does simple CRUD on a single table — ORM abstraction alone may suffice.

---

## Scenario-Based Questions

**Q: You are building a billing system that generates monthly invoices. A single customer can have thousands of transactions. You need to compute the total per customer, apply tiered discounts, and insert results into an invoices table. How do you write this efficiently?**

- Use a single `INSERT...SELECT` with aggregations and a CASE expression for tiered discounts: `WITH customer_totals AS (SELECT customer_id, SUM(amount) AS total FROM transactions WHERE date >= $1 AND date < $2 GROUP BY customer_id) INSERT INTO invoices SELECT customer_id, total * CASE WHEN total > 10000 THEN 0.9 ELSE 1.0 END FROM customer_totals`. Process in batches to avoid long-running transactions.
- **Interview follow-up:** The query locks the transactions table for the entire aggregation. How would you prevent this from blocking new transaction inserts during month-end processing?

---

**Q: Your reporting query joins 6 tables and takes 45 seconds. The CEO needs a dashboard that refreshes in under 5 seconds. You cannot change the schema. What do you do?**

- Check the execution plan for full table scans, nested loops with many iterations, and sort spills. Add missing indexes on join and filter columns. Use a materialized view that pre-joins the data and refreshes periodically. If data can be slightly stale, use a read-only replica or cache results in Redis with a TTL.
- **Interview follow-up:** The materialized view takes 30 seconds to refresh. During refresh, users see stale data. The CEO demands it be fresh every minute. How do you reconcile the refresh latency with the freshness requirement?

---

**Q: A DELETE removing 5M rows from a 50M row table takes 30 minutes and blocks other queries. How do you make it non-blocking and faster?**

- Batch the delete in chunks of 1000 rows with `LIMIT` inside a loop, adding `pg_sleep(0.1)` between batches: `DELETE FROM table WHERE id IN (SELECT id FROM table WHERE condition LIMIT 1000)`. This pattern holds short-lived locks per batch rather than one long exclusive lock.
- **Interview follow-up:** The batched DELETE triggers a foreign key check on each row that cascades to a child table with 100M rows. How do you avoid the cascade becoming the new bottleneck?

---

**Q: Your application team uses `SELECT * FROM users WHERE email = $1` and it's fast. But a new query `SELECT * FROM users WHERE email LIKE '%@gmail.com'` is extremely slow despite an index on email. Why?**

- B-Tree indexes only support prefix patterns (`LIKE 'pattern%'`). `LIKE '%pattern'` cannot use a B-Tree index. Create a trigram GIN index: `CREATE INDEX ON users USING GIN (email gin_trgm_ops)` for efficient wildcard searches on both sides.

---

**Q: A `LEFT JOIN` returns 3x more rows than expected for a one-row-per-customer report. What happened?**

- The right table has multiple matching rows per left row, causing row multiplication. Use `DISTINCT ON (customer.id)` with proper ORDER BY to pick the desired match, or aggregate the right side: `LEFT JOIN (SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id)`.

---

**Q: Your yearly cleanup runs `DELETE FROM logs WHERE created_at < NOW() - INTERVAL '1 year'`. It works for months but suddenly fails with "out of disk space" on the WAL. Why?**

- The DELETE generates massive Write-Ahead Log data in a single transaction. Batch in smaller transactions (10K rows each), committing after each batch. Each commit recycles WAL segments and keeps disk usage manageable.

---

**Q: A query with multiple CTEs runs slower than an equivalent subquery version. You expected CTEs to be faster. What's happening?**

- In PostgreSQL, CTEs are optimization fences — the optimizer cannot push predicates through CTEs, so they are materialized fully before the outer query runs. Use `NOT MATERIALIZED` (PostgreSQL 12+) to inline the CTE, or rewrite as subqueries for better optimization.

---

**Q: A table has 100 columns. `SELECT *` for a paginated list takes 200ms. Selecting only 3 columns takes 20ms. Why such a big difference?**

- `SELECT *` reads all 100 columns from disk, consuming more I/O and network bandwidth. The 10x difference suggests the table is wider than a single page per row. Always select only needed columns and use DTO projections for read-only queries in JPA.

---

**Q: Your query uses `WHERE status IN (SELECT status FROM allowed_statuses)`. The subquery returns 3 values but is slow. How would you rewrite it?**

- The subquery may be evaluated per row (correlated subquery). Rewrite as a JOIN: `SELECT t.* FROM table t JOIN allowed_statuses a ON t.status = a.status`. If `allowed_statuses` is small, use a hardcoded IN list instead.

---

**Q: A batch INSERT of 10K rows into a table with 5 indexes takes 10 seconds. A single-row INSERT takes 2ms. Why doesn't it scale linearly?**

- Each index has O(log n) insert cost, and without batching, each INSERT is a separate transaction with commit overhead. Use multi-row INSERT (`INSERT INTO t VALUES (...), (...), (...)`), batch in a single transaction, and configure `hibernate.jdbc.batch_size` in JPA.

## Interview Questions

- **What is the logical execution order of a SELECT statement?**
  - FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET. The optimizer may reorder operations physically but the logical order defines data visibility.

- **What is the difference between INNER JOIN and LEFT JOIN?**
  - INNER JOIN returns only rows with matches in both tables. LEFT JOIN returns all rows from the left table, with NULLs where the right table has no match.

- **How does NULL behave in SQL?**
  - NULL represents unknown. `NULL = NULL` yields NULL, not TRUE — always use `IS NULL`. Aggregate functions ignore NULLs except `COUNT(*)`. `WHERE col NOT IN (1,2,3)` excludes rows where col is NULL.

- **What is the difference between UNION and UNION ALL?**
  - UNION removes duplicates by performing an implicit sort, while UNION ALL keeps all rows. UNION ALL is faster when duplicates are acceptable or impossible.

- **What is a correlated subquery?**
  - A subquery that references columns from the outer query and executes once per outer row. It is usually less efficient than a JOIN with GROUP BY for the same result.

- **How do window functions differ from GROUP BY?**
  - Window functions perform calculations across related rows without collapsing them into a single output row. GROUP BY reduces groups to one row per group. Window functions use `OVER (PARTITION BY ...)`.

- **What is the difference between HAVING and WHERE?**
  - WHERE filters individual rows before GROUP BY. HAVING filters groups after GROUP BY. HAVING can reference aggregate functions; WHERE cannot.

- **How do you paginate efficiently in SQL?**
  - Keyset pagination uses `WHERE id > :last_id ORDER BY id LIMIT :size` for O(1) per page. OFFSET pagination is O(n) because skipped rows are still read internally by the database.

- **What is a CTE and when is it useful?**
  - A Common Table Expression (WITH clause) defines a named temporary result set within a query. Useful for recursive queries, breaking complex queries into steps, and reusing subquery results.

- **How do you safely interpolate user input in SQL?**
  - Use parameterized queries (prepared statements) with placeholders. Never concatenate user input into SQL strings. In JPA use `:param` with `@Param`; in JDBC use `PreparedStatement`.

## Developer Recommendations

- **Always use explicit column lists instead of SELECT ***
  - `SELECT *` reads all columns, increasing I/O, network transfer, and memory usage. Schema changes with `SELECT *` may also break application code. Explicit columns enable index-only scans.
  - **Production story:** A team added a 10KB `avatar_data` BLOB column to the users table. Their `SELECT *` user listing query, previously 50ms at 100 QPS, jumped to 800ms and saturated the network interface. The fix was listing only the 3 needed columns.

- **Prefer JOINs over correlated subqueries**
  - Correlated subqueries execute once per outer row (O(n × m)). A JOIN with GROUP BY does the same work in a single pass. Exception: when the subquery returns a single row and the filter is highly selective.

- **Use keyset pagination instead of OFFSET for deep pages**
  - OFFSET 100000 reads 100K rows internally before discarding them. Keyset pagination uses a WHERE clause on the last-seen cursor for constant-time access. Trade-off: requires a unique sort column and cannot jump to arbitrary pages.
  - **Production story:** A production admin panel using OFFSET 50000 LIMIT 20 caused a 15-second query that blocked the database connection pool. The 5 concurrent admin users exhausted all 20 connections, cascading into a full-site outage.

- **Batch DML operations in transactions**
  - Each individual DML outside a transaction issues an implicit commit. Batching 1000 rows in a single transaction improves throughput 10-100x. In JPA, configure `hibernate.jdbc.batch_size`.

- **Avoid functions on indexed columns in WHERE clauses**
  - `WHERE YEAR(date) = 2024` prevents index usage. Use `WHERE date >= '2024-01-01' AND date < '2025-01-01'`. For `LOWER()`, create a functional index.

- **Use DISTINCT sparingly**
  - DISTINCT often masks a missing JOIN condition that creates a Cartesian product. Verify JOIN correctness before adding DISTINCT as a fix.

- **Handle NULLs explicitly in WHERE clauses**
  - `WHERE col NOT IN (1,2,3)` silently excludes rows where `col IS NULL`. Use `WHERE (col NOT IN (1,2,3) OR col IS NULL)` for correct behavior.
