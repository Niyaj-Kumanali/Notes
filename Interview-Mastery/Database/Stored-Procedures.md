# Stored Procedures

---

## What are Stored Procedures?

**Stored procedures** are pre-compiled SQL code blocks stored in the database and executed on the server. They encapsulate business logic close to the data, reducing network round trips and enabling complex transactional operations. In modern architectures, stored procedures are best used for data-intensive, set-based operations rather than simple CRUD, which is better handled by application-level ORMs like JPA.

### Benefits

- **Pre-compilation** — The procedure is parsed and optimized on first execution; subsequent calls reuse the cached plan, reducing overhead.
- **Server-side execution** — Logic runs close to the data, minimizing network transfer between the application and database servers.
- **Security** — Direct table access can be revoked, exposing only procedure access to application users. This provides a controlled API for data operations.
- **Transaction control** — Complex multi-step operations can be wrapped in a single transaction with full COMMIT and ROLLBACK control.
- **Deterministic logic** — Consistent behavior regardless of client implementation language, ensuring business rules are enforced uniformly.

### Procedure vs Function

- **Function** — Must return a value and can be used in SQL expressions like `SELECT func()`. Has limited transaction control — cannot COMMIT or ROLLBACK.
- **Procedure** — May not return a value and is called via `CALL proc()`. Has full transaction control including COMMIT and ROLLBACK, enabling more complex transactional logic.

### Languages

- **PL/pgSQL** — PostgreSQL's native procedural language, similar to Oracle's PL/SQL. Supports variables, cursors, exception handling, and control structures.
- **PL/SQL** — Oracle's proprietary procedural language with extensive features for enterprise database development.
- **T-SQL** — SQL Server's procedural language, integrated with Microsoft's ecosystem and tools.
- **SQL/PSM** — MySQL and DB2's standard procedural language based on the SQL/PSM standard.

### Security Contexts

- **SECURITY INVOKER** (default) — The procedure runs with the caller's permissions. PostgreSQL 15+ defaults to this mode for better security.
- **SECURITY DEFINER** — The procedure runs with the owner's permissions, enabling controlled privilege escalation for specific operations.

---

## Core Concepts

### Exception Handling

```sql
CREATE OR REPLACE PROCEDURE transfer_funds(
    from_id INT, to_id INT, amount DECIMAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Account % not found', from_id;
    END IF;

    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Account % not found', to_id;
    END IF;

    INSERT INTO transfer_log(from_id, to_id, amount) VALUES (from_id, to_id, amount);
EXCEPTION
    WHEN OTHERS THEN
        RAISE;
END;
$$;
```

Exception handling in PL/pgSQL uses `EXCEPTION` blocks to catch errors. The `WHEN OTHERS` clause catches all exceptions, and `RAISE` re-throws the error after cleanup.

### Calling Stored Procedures with JPA

```java
// Programmatic approach
@Repository
public class OrderRepository {
    @PersistenceContext
    private EntityManager entityManager;

    @SuppressWarnings("unchecked")
    public List<Object[]> getOrdersByDate(LocalDateTime start, LocalDateTime end, int limit) {
        StoredProcedureQuery query = entityManager
            .createStoredProcedureQuery("get_orders_by_date")
            .registerStoredProcedureParameter("p_start_date", LocalDateTime.class, ParameterMode.IN)
            .registerStoredProcedureParameter("p_end_date", LocalDateTime.class, ParameterMode.IN)
            .registerStoredProcedureParameter("p_limit", Integer.class, ParameterMode.IN)
            .registerStoredProcedureParameter("p_cursor", void.class, ParameterMode.REF_CURSOR)
            .setParameter("p_start_date", start)
            .setParameter("p_end_date", end)
            .setParameter("p_limit", limit);
        query.execute();
        return query.getResultList();
    }
}
```

### Spring Data JPA @Procedure

```sql
CREATE OR REPLACE FUNCTION calculate_order_discount(
    order_id BIGINT, customer_tier VARCHAR
) RETURNS DECIMAL(10,2)
LANGUAGE plpgsql
AS $$
DECLARE
    order_total DECIMAL(10,2);
    discount DECIMAL(10,2);
BEGIN
    SELECT total INTO order_total FROM orders WHERE id = order_id;
    discount := CASE customer_tier
        WHEN 'GOLD' THEN order_total * 0.20
        WHEN 'SILVER' THEN order_total * 0.10
        ELSE order_total * 0.05
    END;
    RETURN discount;
END;
$$;
```

```java
@Entity
@NamedStoredProcedureQuery(
    name = "calculateOrderDiscount",
    procedureName = "calculate_order_discount",
    parameters = {
        @StoredProcedureParameter(mode = ParameterMode.IN, name = "order_id", type = Long.class),
        @StoredProcedureParameter(mode = ParameterMode.IN, name = "customer_tier", type = String.class),
        @StoredProcedureParameter(mode = ParameterMode.OUT, name = "discount", type = BigDecimal.class)
    }
)
public class Order { /* entity fields */ }
```

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Procedure(procedureName = "calculate_order_discount")
    BigDecimal calculateDiscount(Long orderId, String customerTier);
}
```

### Batch Processing with Cursor

```sql
CREATE OR REPLACE PROCEDURE process_pending_orders(p_batch_size INT DEFAULT 100)
LANGUAGE plpgsql
AS $$
DECLARE
    v_order RECORD;
    v_processed INT := 0;
    v_cursor CURSOR FOR
        SELECT id, total, user_id
        FROM orders WHERE status = 'PENDING'
        ORDER BY created_at
        FOR UPDATE SKIP LOCKED
        LIMIT p_batch_size;
BEGIN
    OPEN v_cursor;
    LOOP
        FETCH v_cursor INTO v_order;
        EXIT WHEN NOT FOUND;
        UPDATE orders SET status = 'PROCESSING' WHERE id = v_order.id;
        v_processed := v_processed + 1;
    END LOOP;
    CLOSE v_cursor;
    RAISE NOTICE 'Processed % orders', v_processed;
END;
$$;
```

### Performance: Set-Based vs Row-by-Row

```sql
-- BAD: Row-by-row processing (slow)
CREATE PROCEDURE slow_process()
LANGUAGE plpgsql
AS $$
DECLARE r RECORD;
BEGIN
    FOR r IN SELECT * FROM orders WHERE status = 'PENDING' LOOP
        UPDATE orders SET status = 'PROCESSED' WHERE id = r.id;
    END LOOP;
END;
$$;

-- GOOD: Set-based operation (fast)
CREATE PROCEDURE fast_process()
LANGUAGE sql
AS $$
    UPDATE orders SET status = 'PROCESSED' WHERE status = 'PENDING';
$$;
```

Set-based operations are orders of magnitude faster than explicit cursor loops. SQL is designed for set operations — avoid FOR loops over large result sets.

### Security with SECURITY DEFINER

```sql
-- Users have no direct table access; only through procedures
CREATE PROCEDURE get_own_profile(p_user_id INT)
LANGUAGE SQL
SECURITY DEFINER
AS $$
    SELECT id, username, email, created_at FROM users WHERE id = p_user_id;
$$;

REVOKE ALL ON users FROM app_role;
GRANT EXECUTE ON PROCEDURE get_own_profile TO app_role;
```

SECURITY DEFINER allows limited privilege escalation — users can access their own data through the procedure without having direct table access.

---

## Common Mistakes

- **Procedural, not set-based thinking** — SQL is designed for set operations, not row-by-row loops. A cursor loop processing 50K rows is 100x slower than a single UPDATE.
  - **Why it looks correct:** Procedural languages (PL/pgSQL, T-SQL) use familiar loop constructs that feel intuitive to developers coming from general-purpose programming languages.
- **Overusing stored procedures for simple CRUD** — Application-level JPA is simpler, more testable, and provides better type safety for basic create-read-update-delete operations.
  - **Why it looks correct:** Stored procedures execute close to the data and seem inherently faster; the network overhead of sending SQL from the application is visible while the maintenance cost of dual-location logic is invisible.
- **Ignoring error handling** — Unhandled exceptions can leave partial transactions in an inconsistent state. Always include EXCEPTION blocks.
  - **Why it looks correct:** The procedure works correctly in the happy path during testing; the partial failure scenario only manifests when a specific step fails mid-execution.
- **Hard to unit test** — Stored procedures require a database setup for testing, making them harder to test than application-layer code. Prefer integration tests.
  - **Why it looks correct:** The procedure is tested once during development and appears to work; the lack of automated regression testing is invisible until a schema change breaks it silently in production.
- **Version control challenges** — Without proper tooling, stored procedures can drift from application code. Ensure procedures are in Flyway or Liquibase migrations.
  - **Why it looks correct:** The procedure is modified directly in the database console and works immediately; the drift between the production database and the codebase is invisible until a new environment is provisioned with the old procedure definition.
- **Debugging difficulty** — No debugger exists for most procedural SQL. Use `RAISE NOTICE` or `DBMS_OUTPUT` for debugging output.
  - **Why it looks correct:** The print-statement approach mirrors how developers debugged code before graphical debuggers; the lack of stack traces, breakpoints, and variable inspection makes finding the exact line of failure much harder than in application code.
- **Hidden complexity** — Business logic ends up in two places (application code and database), creating a maintenance burden and making system behavior harder to reason about.
  - **Why it looks correct:** Each piece of logic makes sense in isolation; the cognitive overhead of tracking which logic lives where only becomes apparent during debugging when you must trace execution across both layers.
- **No type safety** — Procedural SQL has weaker typing compared to Java or Kotlin, making certain bugs harder to catch at compile time.
  - **Why it looks correct:** The procedure compiles and runs; type mismatches that would be caught by a Java compiler manifest as runtime errors that only surface when the specific code path executes.
- **Stateless concerns** — Stored procedures cannot access caches, external APIs, or services. Any operation requiring external resources must live in the application layer.
  - **Why it looks correct:** The procedure is designed to run inside the database; the limitation is invisible until a requirement to call an external service or cache emerges and the procedure cannot be extended.
- **Performance estimation** — It is hard to assess the cost of a procedure without executing it on realistic data volumes.
  - **Why it looks correct:** The procedure runs instantly during development on small datasets; the cost difference between 100-row and 10M-row execution is a scale inflection point that cannot be extrapolated linearly.

---

## Real-World Scenarios

### Funds Transfer with Full Transaction Control

A banking funds transfer must debit one account, credit another, and create an audit log — all or nothing. A stored procedure wraps everything in a single transaction:

```sql
CREATE OR REPLACE PROCEDURE transfer_funds(
    p_from_id INT, p_to_id INT, p_amount DECIMAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_from_id;
    IF NOT FOUND THEN RAISE EXCEPTION 'Account % not found', p_from_id; END IF;

    UPDATE accounts SET balance = balance + p_amount WHERE id = p_to_id;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Account % not found, reversing debit', p_to_id;
    END IF;

    INSERT INTO audit_log(entity_type, entity_id, action, amount)
    VALUES ('TRANSFER', p_from_id, 'DEBIT', -p_amount),
           ('TRANSFER', p_to_id, 'CREDIT', p_amount);
END;
$$;
```

### Batch Order Processing with Cursor

An order fulfillment system processes 100K orders nightly. A stored procedure with cursor and SKIP LOCKED handles this:

```sql
CREATE OR REPLACE PROCEDURE process_order_batch(p_batch_size INT DEFAULT 1000)
LANGUAGE plpgsql
AS $$
DECLARE
    r RECORD;
    v_count INT := 0;
BEGIN
    FOR r IN SELECT id FROM orders WHERE status = 'PENDING'
             ORDER BY created_at LIMIT p_batch_size FOR UPDATE SKIP LOCKED
    LOOP
        UPDATE orders SET status = 'PROCESSING', updated_at = NOW() WHERE id = r.id;
        INSERT INTO order_log(order_id, status, logged_at)
        VALUES (r.id, 'PROCESSING', NOW());
        v_count := v_count + 1;
    END LOOP;
    RAISE NOTICE 'Processed % orders', v_count;
END;
$$;
```

### Row-Level Access Control via SECURITY DEFINER

A multi-tenant SaaS app ensures users only access their own data via SECURITY DEFINER:

```sql
CREATE FUNCTION get_own_invoices(p_user_id INT, p_tenant_id INT)
RETURNS TABLE(invoice_id INT, total DECIMAL, status VARCHAR)
LANGUAGE SQL
SECURITY DEFINER
AS $$
    SELECT id, total, status FROM invoices
    WHERE user_id = p_user_id AND tenant_id = p_tenant_id;
$$;

REVOKE ALL ON invoices FROM app_role;
GRANT EXECUTE ON FUNCTION get_own_invoices TO app_role;
```

## Scenario-Based Questions

- **Q:** You are designing a payment processing system. The team proposes putting all business logic in stored procedures for "performance." The system needs to call external payment APIs, send emails, and integrate with fraud detection. What do you advise?
  - **A:** Stored procedures cannot make HTTP calls or access external services. Put orchestration in the application layer. Use procedures only for data-intensive operations (batch updates, complex aggregations, transactional validation). This keeps logic testable and extensible.
  - **Interview follow-up:** The team insists on stored procedures for the core transaction logic because "the database guarantees atomicity." The application layer calls the procedure and then calls the external payment API. If the payment API fails after the procedure commits, the transaction is already committed. How do you handle this inconsistency?
- **Q:** A stored procedure processes 50,000 rows using a FOR loop with individual UPDATEs. Each iteration takes 2ms, total 100 seconds. How do you make it 100x faster?
  - **A:** Row-by-row processing is orders of magnitude slower than set-based operations. Rewrite as a single UPDATE: `UPDATE orders SET status = 'PROCESSED' WHERE status = 'PENDING' AND created_at < cutoff`. If per-row logic is needed, use a single UPDATE with a FROM clause.
  - **Interview follow-up:** The per-row logic requires calling a calculation function that references a second table and the result depends on the current row's data. A single UPDATE with FROM still works, but how do you handle cases where the calculation requires calling an external HTTP API per row?
- **Q:** A stored procedure was deployed via Flyway but continues to exhibit old behavior. The new definition is in pg_proc. Why is the old behavior cached?
  - **A:** PostgreSQL caches execution plans. The new definition exists but the cached plan is stale. Run `DISCARD PLANS` or wait for cache invalidation. For functions affected by schema changes, use `ALTER FUNCTION ...` to force recompilation.
  - **Interview follow-up:** The procedure is called thousands of times per second. Running `DISCARD PLANS` drops all cached plans, causing a temporary CPU spike as plans are regenerated. How do you invalidate only this specific procedure's plan without affecting other queries?
- **Q:** Your team needs to grant a reporting user access to see order data but prevent them from seeing `credit_card_last_four`. You don't want a view. How do you use stored procedures for this?
  - **A:** Use `SECURITY DEFINER` procedures returning only permitted columns. Revoke direct table access and grant EXECUTE on the procedures: `CREATE FUNCTION get_orders_for_reporting() RETURNS TABLE(id INT, total DECIMAL, status VARCHAR) LANGUAGE SQL SECURITY DEFINER AS $$ SELECT id, total, status FROM orders; $$; REVOKE ALL ON orders FROM reporting_role; GRANT EXECUTE ON FUNCTION get_orders_for_reporting TO reporting_role;`.
- **Q:** A nightly ETL stored procedure takes 3 hours. If it fails at 2.5 hours, all changes roll back. How do you make it resumable?
  - **A:** Add checkpointing with incremental COMMITs. Process in batches (10K rows), committing after each batch. Track progress: `UPDATE etl_control SET last_processed_id = ?, status = 'IN_PROGRESS'`. On restart, resume from the checkpoint. Only the current batch is lost on failure.
- **Q:** A stored procedure with SECURITY INVOKER fails with permission errors when called by the application user, even though the user has EXECUTE privilege. What's wrong?
  - **A:** With SECURITY INVOKER (PostgreSQL 15+ default), the procedure runs with the caller's permissions. The caller needs permissions on underlying tables, not just the procedure. Either grant table permissions or change to SECURITY DEFINER.
- **Q:** A microservice needs data from another service's database. The team wants a cross-service stored procedure. Is this a good idea?
  - **A:** No — this creates tight coupling between services. Each service owns its database. Call the other service's API instead. For cross-database queries, use foreign data wrappers (FDW) or a reporting database fed by an event-driven pipeline.
- **Q:** Your stored procedure's EXPLAIN ANALYZE shows different plans in dev (100 rows) vs prod (10M rows). The prod plan is suboptimal. How do you fix it?
  - **A:** The optimizer chooses different plans based on data distribution. Test with prod-sized data. Use `pg_hint_plan` to pin the good plan, break the procedure into steps with intermediate temp tables, or update statistics with extended statistics for correlated columns.
- **Q:** A stored procedure must run in a transaction but internally calls COMMIT (necessary in PostgreSQL maintenance). How does this affect the calling app's transaction?
  - **A:** COMMIT inside a procedure (CALL) changes the calling app's transaction context. The outer savepoint or transaction is affected. Avoid COMMIT inside procedures called from app transactions. Use batched calls instead.
- **Q:** A stored procedure with ROLLBACK in the EXCEPTION block is migrated to Spring Boot with JPA. The procedure is called via @Procedure and the app's @Transactional also tries to roll back on error. What happens?
  - **A:** Double rollback — the procedure rolls back inside the database, then Spring tries to roll back the already-rolled-back connection. This can cause `RollbackException` or connection state issues. Either let the procedure manage transactions (no `@Transactional`) or let Spring manage (remove ROLLBACK from procedure).

## Interview Questions

- **What is a stored procedure?**
  - **A:** A pre-compiled SQL code block stored in the database, executed server-side. Encapsulates business logic close to data, reducing network round trips.
- **What is the difference between a stored procedure and a function?**
  - **A:** A function must return a value and can be used in SQL expressions. A procedure may not return a value and supports full transaction control (COMMIT/ROLLBACK).
- **What are the benefits of stored procedures?**
  - **A:** Pre-compilation (cached plans), server-side execution (close to data), security (table access via procedures only), and multi-step atomic operations.
- **What are the drawbacks of stored procedures?**
  - **A:** Hard to unit test, difficult to version control, limited debugging, no external service access, weak typing, and logic spread across app and database.
- **What is SECURITY DEFINER vs SECURITY INVOKER?**
  - **A:** SECURITY DEFINER runs with the owner's permissions (privilege escalation). SECURITY INVOKER runs with the caller's permissions. PostgreSQL 15+ defaults to SECURITY INVOKER.
- **When would you use a stored procedure over application code?**
  - **A:** For data-intensive set-based operations, complex multi-step transactions requiring atomicity, batch ETL, and operations minimizing network round trips.
- **How do you call a stored procedure from Spring Data JPA?**
  - **A:** Use `@Procedure(procedureName = "my_proc")` on a repository method, or `EntityManager.createStoredProcedureQuery()` for REF_CURSOR results.
- **What is the performance difference between set-based and row-by-row?**
  - **A:** Set-based operations are orders of magnitude faster than explicit cursor loops. SQL is designed for sets — avoid FOR loops over large result sets.
- **How do you handle errors in stored procedures?**
  - **A:** Use EXCEPTION blocks in PL/pgSQL to catch errors, roll back partial changes, log, and re-raise. Always handle "row not found" with explicit checks.
- **Can stored procedures call external APIs?**
  - **A:** No — procedures run inside the database process and cannot make HTTP calls. This logic belongs in the application layer.

## Developer Recommendations

- **Use set-based SQL, not row-by-row loops** — PL/pgSQL FOR loops processing thousands of rows are 100x slower than a single UPDATE. Write set-based queries first; use cursors only when per-row logic is unavoidable.
  - **Production story:** A nightly ETL procedure processed 200K rows in a FOR loop with individual UPDATEs. The job ran for 45 minutes and frequently timed out during peak season. Rewriting to a single `UPDATE ... FROM` with a CASE expression reduced execution to 12 seconds and eliminated the timeout incidents.
- **Keep simple CRUD in the application layer** — JPA repositories handle basic CRUD with better testability and type safety. Reserve procedures for complex data-intensive operations.
  - **Production story:** A team migrated all `INSERT` and `SELECT` operations to stored procedures "for consistency." When a schema migration added a column, every procedure had to be updated individually — 47 procedures needed changes for a single column addition. The 3-day migration effort was entirely avoidable with JPA's automatic column mapping.
- **Version control stored procedures in database migrations** — Store definitions in Flyway or Liquibase migrations, ensuring procedures are versioned and deployed consistently across environments.
- **Use SECURITY DEFINER sparingly** — It grants elevated privileges to the caller. Validate inputs and restrict operations. Prefer SECURITY INVOKER with fine-grained table permissions.
- **Test with realistic data volumes** — A procedure that is fast on 100 rows may be catastrophically slow on 10M rows. Always test with production-sized datasets.
- **Add checkpoints to long-running ETL procedures** — Processing 1M rows in a single transaction is risky. Commit in batches and track progress for checkpoint-based resumability.
- **Avoid mixing procedure and application transaction management** — Either let the procedure manage transactions (COMMIT/ROLLBACK internally) or let Spring manage. Mixing causes double-rollback issues.
