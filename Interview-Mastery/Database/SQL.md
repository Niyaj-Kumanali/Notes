# SQL

## 1. Executive Summary

Structured Query Language (SQL) is the standard language for relational database management systems. It enables the creation, manipulation, querying, and administration of relational databases. SQL is declarative: you specify WHAT you want, not HOW to get it. The SQL engine determines the execution plan. Mastery of SQL is essential for any backend or full-stack engineer, particularly in Java/Spring Boot ecosystems where JPA/Hibernate generate SQL automatically.

## 2. Core Theory

### 2.1 Relational Model Foundation

SQL is built on Codd's relational model:
- **Relation** = table
- **Tuple** = row
- **Attribute** = column
- **Domain** = allowed values for a column
- **Key** = unique identifier for a tuple
- **Foreign Key** = reference to another relation's key

### 2.2 SQL Sub-Languages

| Sublanguage | Category | Commands |
|-------------|----------|----------|
| DDL | Schema Definition | CREATE, ALTER, DROP, TRUNCATE |
| DML | Data Manipulation | SELECT, INSERT, UPDATE, DELETE |
| DCL | Access Control | GRANT, REVOKE |
| TCL | Transaction Control | BEGIN, COMMIT, ROLLBACK, SAVEPOINT |

### 2.3 SELECT Statement Execution Order (Logical)

```
FROM -> JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> DISTINCT -> ORDER BY -> LIMIT/OFFSET
```

This is the LOGICAL order. The database engine may reorder operations for optimization.

### 2.4 Set Operations

- UNION (distinct union)
- UNION ALL (duplicates retained)
- INTERSECT (common rows)
- EXCEPT / MINUS (rows in first not in second)

### 2.5 Predicate Types

- Comparison: =, <>, <, >, <=, >=
- Range: BETWEEN x AND y
- Set Membership: IN (subquery), NOT IN
- Pattern: LIKE ('%' wildcard, '_' single char)
- NULL Check: IS NULL, IS NOT NULL
- Quantified: ALL, ANY, SOME
- Existence: EXISTS, NOT EXISTS

### 2.6 Aggregate Functions

COUNT, SUM, AVG, MIN, MAX, STRING_AGG, ARRAY_AGG

## 3. Under-the-Hood Deep Dive

### 3.1 Query Processing Pipeline

```
SQL Text -> Parser -> Rewriter -> Planner/Optimizer -> Executor -> Result
```

1. **Parser**: Tokenizes, validates syntax, builds parse tree
2. **Rewriter**: Applies rules (view expansion, constant folding)
3. **Planner/Optimizer**: Generates execution plans, chooses lowest-cost plan
4. **Executor**: Executes plan operators (sequential scan, index scan, joins, etc.)

### 3.2 NULL Handling

NULL is not a value; it represents unknown/missing. NULL = NULL yields NULL (not TRUE). Use IS NULL. Three-valued logic: TRUE, FALSE, UNKNOWN.

### 3.3 Join Internals

- **Nested Loop Join**: O(n*m) — good for small tables or index lookups
- **Hash Join**: O(n+m) — builds hash table on one side, probes with the other — good for unsorted large datasets
- **Merge Join**: O(n+m) — requires sorted inputs — good for pre-indexed columns

### 3.4 Subquery Execution

- **Correlated subquery**: Executed once per outer row (expensive)
- **Uncorrelated subquery**: Executed once, result cached
- Database optimizers often rewrite subqueries as joins or semi-joins

### 3.5 Common Table Expression (CTE) Materialization

- Non-recursive CTEs may or may not be materialized depending on the DBMS
- Some databases materialize CTEs as temporary tables (acts as optimization fence)
- PostgreSQL 12+ can inline simple CTEs

## 4. Production Code Examples

### 4.1 Basic CRUD with Spring Data JPA

```sql
-- Table creation
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    username VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

```java
// JPA Entity
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String username;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }

    // getters, setters, constructors
}
```

```java
// Spring Data Repository
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<User> findByUsernameContainingIgnoreCase(String partial);
    boolean existsByEmail(String email);
}
```

### 4.2 Custom Queries with @Query

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmailCustom(@Param("email") String email);

    @Query(value = "SELECT * FROM users WHERE username ILIKE %:partial%", nativeQuery = true)
    List<User> searchByUsername(@Param("partial") String partial);

    @Modifying
    @Query("UPDATE User u SET u.username = :username WHERE u.id = :id")
    int updateUsername(@Param("id") Long id, @Param("username") String username);

    @Modifying
    @Query(value = "DELETE FROM users WHERE created_at < :cutoff", nativeQuery = true)
    int deleteOldUsers(@Param("cutoff") LocalDateTime cutoff);
}
```

### 4.3 Batch Operations

```java
// Batch insert with JPA
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

```sql
-- SQL batch insert
INSERT INTO users (email, username) VALUES
('alice@example.com', 'alice'),
('bob@example.com', 'bob'),
('charlie@example.com', 'charlie');
```

### 4.4 Pagination

```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    Page<Post> findByPublishedTrue(Pageable pageable);

    @Query("SELECT p FROM Post p WHERE p.author.id = :authorId")
    Slice<Post> findByAuthor(@Param("authorId") Long authorId, Pageable pageable);
}

// Service usage
public Page<Post> getPublishedPosts(int page, int size) {
    Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
    return postRepository.findByPublishedTrue(pageable);
}
```

## 5. Real-World Scenarios

### 5.1 Soft Delete Pattern

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP NULL;
CREATE INDEX idx_users_active ON users(deleted_at) WHERE deleted_at IS NULL;
```

```java
@Entity
@Table(name = "users")
@SQLDelete(sql = "UPDATE users SET deleted_at = CURRENT_TIMESTAMP WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
public class User {
    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;
}
```

### 5.2 Versioning / Optimistic Locking

```java
@Entity
public class Product {
    @Version
    private Long version;
}
```

### 5.3 Audit Logging with Triggers

```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    row_id BIGINT NOT NULL,
    operation TEXT NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_by TEXT,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 6. Performance

### 6.1 SELECT * Anti-Pattern

Always select only needed columns. SELECT * forces full table scan, increases I/O, and prevents index-only scans.

### 6.2 N+1 Query Problem

```java
// BAD: N+1 queries
List<Author> authors = authorRepository.findAll();
for (Author author : authors) {
    System.out.println(author.getBooks().size()); // triggers query per author
}

// FIX: Join fetch
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();

// Or use EntityGraph
@EntityGraph(attributePaths = {"books"})
@Query("SELECT a FROM Author a")
List<Author> findAllWithBooks();
```

### 6.3 Connection Pooling

```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1200000
```

## 7. Security

### 7.1 SQL Injection Prevention

```java
// NEVER do this:
String sql = "SELECT * FROM users WHERE email = '" + userInput + "'";

// ALWAYS use parameterized queries:
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// For native queries, use ? binding
@Query(value = "SELECT * FROM users WHERE email = ?1", nativeQuery = true)
Optional<User> findByEmailNative(String email);
```

### 7.2 Least Privilege Principle

```sql
CREATE USER app_user WITH PASSWORD 'secure';
GRANT SELECT, INSERT, UPDATE, DELETE ON all_tables TO app_user;
REVOKE ALL ON sensitive_table FROM app_user;
```

## 8. Common Mistakes

1. **Using SELECT DISTINCT to hide missing JOIN conditions** — indicates a Cartesian product
2. **Forgetting NULL handling** — NULL IN (1,2,3) is NULL, not FALSE
3. **Missing indexes on foreign keys** — leads to sequential scans on JOINs
4. **Using functions in WHERE clauses** — `WHERE YEAR(date) = 2024` prevents index usage
5. **Assuming implicit row ordering** — without ORDER BY, order is undefined
6. **Overusing subqueries when JOINs suffice** — correlated subqueries are especially slow
7. **Ignoring transaction boundaries** — each JPA repository call is a separate transaction by default

## 9. Senior Engineer Perspective

Production SQL considerations:
- **Idempotency**: Use UPSERT (INSERT ... ON CONFLICT DO UPDATE) for retry-safe operations
- **Idempotency keys**: Store idempotency_key with UNIQUE constraint
- **Schema migrations**: Use Flyway or Liquibase — never manual DDL
- **Read replicas**: Route read queries to replicas, writes to primary
- **Database per service**: In microservices, avoid shared databases
- **Connection management**: Always return connections to pool, use connection timeout

## 10. Interview Questions (20)

### Easy (10)

1. What is the difference between WHERE and HAVING?
2. Explain the difference between INNER JOIN and LEFT JOIN.
3. What is a PRIMARY KEY?
4. What does NULL mean in SQL?
5. Explain the difference between UNION and UNION ALL.
6. What is a foreign key constraint?
7. How do you count rows in a table?
8. What is the difference between DELETE and TRUNCATE?
9. Explain LIKE and its wildcard characters.
10. What is the purpose of DISTINCT?

### Medium (10)

11. Write a query to find duplicate email addresses in a users table.
12. Explain the difference between `WHERE` and `ON` in a JOIN.
13. What is a correlated subquery? Give an example.
14. How would you paginate results in SQL? Write the syntax.
15. What is the difference between CHAR and VARCHAR?
16. Write a query to find the nth highest salary.
17. Explain GROUP BY and why non-aggregated columns must appear in it.
18. What is a self-join? When would you use it?
19. What is the difference between `HAVING COUNT(*) > 1` and putting the condition in WHERE?
20. How does NULL behave with aggregate functions like COUNT, SUM, AVG?

## 11. Advanced Interview Questions (20)

### Hard (10)

1. Write a recursive CTE to generate a date range or traverse a tree.
2. Explain the difference between `EXISTS` and `IN` with a subquery. When is each faster?
3. How do you implement intersection in a database that doesn't support INTERSECT?
4. What is a window function? Write a query with ROW_NUMBER() and RANK().
5. How would you implement a pivot in SQL (rows to columns)?
6. Explain the difference between CROSS APPLY and INNER JOIN (SQL Server).
7. Write a query to find gaps in a sequence of numbers.
8. What is the difference between `WITH TIES` and `WITHOUT TIES` in ORDER BY?
9. How does the MERGE (UPSERT) statement work? Write an example.
10. Explain SQL injection and demonstrate safe parameterized query patterns.

### System Design (10)

11. Design a database schema for a URL shortener.
12. Design the schema for a social media feed with likes and comments.
13. How would you store hierarchical data (org chart, tree) in SQL?
14. Design a multi-tenant database schema (shared vs isolated approaches).
15. Design the schema for a booking system with availability checking.
16. How would you implement a queue using SQL tables?
17. Design a schema for an e-commerce product catalog with varying attributes.
18. How would you design a rate-limiting system using a relational database?
19. Design the schema for a chat/messaging application.
20. How would you implement full-text search with SQL (without external search engines)?

## 12. Expert-Level Interview Questions (10)

1. You have a table with 500 million rows. How do you add a new column with a default value without downtime?
2. Describe how you would migrate from a monolithic database to microservice-owned databases.
3. How do you implement a distributed transaction across multiple databases without 2PC?
4. Design a change-data-capture (CDC) system using triggers, audit tables, or logical replication.
5. How would you implement optimistic offline locking in a REST API backed by SQL?
6. Design a multi-region active-active database topology with conflict resolution.
7. How do you handle schema migrations in a zero-downtime deployment pipeline?
8. Describe the internals of how a B-tree maintains balance during concurrent inserts.
9. Design a system to generate globally unique, time-sortable IDs (like Snowflake) using SQL sequences.
10. How would you detect and resolve deadlocks in a high-throughput transaction processing system?

## 13. Debugging & Troubleshooting

### Common Issues and Diagnostics

```sql
-- Find long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Check for blocking locks
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query,
       blocking_activity.query AS blocking_query
FROM pg_locks blocked_locks
JOIN pg_locks blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
JOIN pg_stat_activity blocked_activity ON blocked_locks.pid = blocked_activity.pid
JOIN pg_stat_activity blocking_activity ON blocking_locks.pid = blocking_activity.pid
WHERE NOT blocked_locks.granted;

-- MySQL SHOW PROCESSLIST equivalent
SHOW FULL PROCESSLIST;
```

### JPA/Hibernate Debugging

```yaml
# application.yml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
    org.springframework.transaction: TRACE
```

## 14. Comparison Section

| Feature | SQL | NoSQL (MongoDB) |
|---------|-----|-----------------|
| Schema | Fixed (relational) | Flexible (document) |
| Joins | Native | $lookup (aggregation) |
| Transactions | ACID | Limited (multi-doc) |
| Scaling | Vertical (sharding complex) | Horizontal (native) |
| Query | Declarative (SQL) | JSON-based API |
| Consistency | Strong | Tunable (eventual) |
| Best for | Complex relationships, reporting | High volume, rapid iteration |

| Statement | DDL vs DML vs DCL |
|-----------|-------------------|
| CREATE, ALTER, DROP | DDL (changes structure) |
| SELECT, INSERT, UPDATE, DELETE | DML (manages data) |
| GRANT, REVOKE | DCL (controls access) |

## 15. Revision Notes

- SQL is declarative; the optimizer decides HOW to execute
- Logical order: FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY
- NULL is unknown; use IS NULL/IS NOT NULL, never = NULL
- Always parameterize queries to prevent SQL injection
- Understand the difference between JOIN types: INNER, LEFT, RIGHT, FULL, CROSS, SEMI
- Window functions (ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD) are powerful for analytics
- CTEs improve readability; recursive CTEs traverse hierarchies
- DISTINCT is not a free operation — it causes a sort or hash
- Indexes speed up reads but slow down writes
- Use EXPLAIN ANALYZE to understand query performance

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                        SQL CHEAT SHEET                            |
+-------------------------------------------------------------------+
|                                                                   |
|  SELECT [DISTINCT] columns                                        |
|  FROM table                                                       |
|  [JOIN other ON condition]                                        |
|  [WHERE condition]                                                |
|  [GROUP BY columns]                                               |
|  [HAVING condition]                                               |
|  [ORDER BY columns [ASC|DESC]]                                    |
|  [LIMIT n [OFFSET m]]                                             |
|                                                                   |
+-------------------------------------------------------------------+
|  JOIN TYPES:                                                      |
|                                                                   |
|  INNER JOIN  -> Only matching rows                                |
|  LEFT JOIN   -> All from left + matches from right                |
|  RIGHT JOIN  -> All from right + matches from left                |
|  FULL JOIN   -> All from both sides                               |
|  CROSS JOIN  -> Cartesian product                                 |
|  SEMI JOIN   -> Rows in left that have match in right (EXISTS)    |
+-------------------------------------------------------------------+
|  WINDOW FUNCTIONS:                                                |
|                                                                   |
|  ROW_NUMBER() OVER (PARTITION BY col ORDER BY col)                |
|  RANK()       OVER (PARTITION BY col ORDER BY col)                |
|  LAG(col, n)  OVER (ORDER BY col)                                 |
|  LEAD(col, n) OVER (ORDER BY col)                                 |
|  SUM(col)     OVER (PARTITION BY col ORDER BY col)                |
+-------------------------------------------------------------------+
|  AGGREGATE FUNCTIONS:                                             |
|                                                                   |
|  COUNT(*), COUNT(col)     -> number of rows/non-null               |
|  SUM(col)                 -> total (ignores NULLs)                |
|  AVG(col)                 -> average (ignores NULLs)              |
|  MIN(col), MAX(col)       -> min/max value                        |
|  STRING_AGG(col, delim)   -> concatenate values                   |
+-------------------------------------------------------------------+
|  DATA TYPES (COMMON):                                             |
|                                                                   |
|  Character:  CHAR(n), VARCHAR(n), TEXT                            |
|  Numeric:    INTEGER, BIGINT, DECIMAL(p,s), FLOAT, DOUBLE         |
|  Date/Time:  DATE, TIME, TIMESTAMP, TIMESTAMPTZ, INTERVAL         |
|  Binary:     BYTEA, BLOB                                          |
|  JSON:       JSON, JSONB (PostgreSQL)                             |
+-------------------------------------------------------------------+
|  CONSTRAINTS:                                                     |
|                                                                   |
|  PRIMARY KEY (col)          -> unique + not null                  |
|  FOREIGN KEY (col) REFERENCES t(col) [ON DELETE CASCADE]          |
|  UNIQUE (col)               -> all values distinct                |
|  NOT NULL                   -> column cannot be NULL              |
|  CHECK (condition)          -> row must satisfy condition         |
|  DEFAULT value              -> default when not specified         |
+-------------------------------------------------------------------+
|  SET OPERATIONS:                                                  |
|                                                                   |
|  query1 UNION [ALL] query2   -> combine results (distinct/all)   |
|  query1 INTERSECT query2      -> common rows                      |
|  query1 EXCEPT query2         -> rows in q1 not in q2             |
+-------------------------------------------------------------------+
|  TRANSACTION CONTROL:                                             |
|                                                                   |
|  BEGIN / START TRANSACTION    -> start transaction                |
|  COMMIT                       -> save changes                     |
|  ROLLBACK                     -> undo changes                      |
|  SAVEPOINT name               -> named rollback point             |
+-------------------------------------------------------------------+
|  CONDITIONAL EXPRESSIONS:                                         |
|                                                                   |
|  COALESCE(val1, val2, ...)  -> first non-NULL value               |
|  NULLIF(a, b)               -> NULL if a = b, else a              |
|  CASE WHEN c THEN r ELSE d END -> if-else logic                   |
+-------------------------------------------------------------------+
