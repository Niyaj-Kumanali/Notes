# Stored Procedures

## 1. Executive Summary

Stored procedures are pre-compiled SQL code blocks stored in the database and executed on the server. They encapsulate business logic close to the data, reducing network round trips and enabling complex transactional operations. In modern architectures (microservices, Spring Boot), stored procedures are controversial: some advocate for logic in the application layer, while others leverage procedures for performance-critical paths. The right approach depends on your team's expertise, deployment model, and performance requirements.

## 2. Core Theory

### 2.1 What Stored Procedures Provide

- **Pre-compilation**: Procedure is parsed and optimized on first execution; subsequent calls reuse the plan
- **Server-side execution**: Logic runs close to data, minimizing network transfer
- **Security**: Direct table access can be revoked, exposing only procedure access
- **Transaction control**: Complex multi-step operations can be wrapped in a single transaction
- **Deterministic logic**: Consistent behavior regardless of client implementation

### 2.2 Stored Procedure Languages

| Language | Database | Notes |
|----------|----------|-------|
| PL/pgSQL | PostgreSQL | PostgreSQL's native procedural language |
| PL/SQL | Oracle | Oracle's proprietary procedural language |
| T-SQL | SQL Server | Microsoft's SQL Server procedural language |
| PL/Python | PostgreSQL | Python in the database |
| PL/v8 | PostgreSQL | JavaScript (V8 engine) |
| Java | Oracle | Java stored procedures |
| SQL/PSM | MySQL/DB2 | SQL Persisten Stored Modules standard |

### 2.3 Function vs Procedure

| Feature | Function | Procedure |
|---------|----------|-----------|
| Must return a value | Yes | No (but can have OUT params) |
| Used in SQL expressions | Yes (SELECT func()) | No |
| Transaction control | Limited (no COMMIT/ROLLBACK) | Full control |
| Output | Single value or table | Multiple OUT parameters |
| CALL syntax | SELECT func() | CALL proc() |

## 3. Under-the-Hood Deep Dive

### 3.1 Execution Model

1. **Parse**: Procedure text is parsed into AST
2. **Validate**: References to tables/columns are validated
3. **Optimize**: SQL within the procedure is optimized (plan caching)
4. **Execute**: Server executes the plan with current parameter values
5. **Cache**: Plan stored in procedure cache for reuse

### 3.2 Plan Caching for Procedures

Each SQL statement inside a procedure gets its own execution plan. These plans are cached and parameterized. In PostgreSQL, PL/pgSQL uses "generic plans" after 5 executions (parameter sniffing).

### 3.3 Security Context

```sql
-- SECURITY INVOKER (default): runs as caller's permissions
CREATE PROCEDURE delete_order(oid BIGINT)
LANGUAGE SQL
SECURITY INVOKER
AS $$
    DELETE FROM orders WHERE id = oid;
$$;

-- SECURITY DEFINER: runs as procedure owner's permissions
CREATE PROCEDURE delete_order(oid BIGINT)
LANGUAGE SQL
SECURITY DEFINER
AS $$
    DELETE FROM orders WHERE id = oid;
$$;
```

### 3.4 Exception Handling

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

## 4. Production Code Examples

### 4.1 Stored Procedure with JPA

```sql
-- PostgreSQL stored procedure
CREATE OR REPLACE PROCEDURE get_orders_by_date(
    p_start_date TIMESTAMP,
    p_end_date TIMESTAMP,
    p_limit INT,
    INOUT p_cursor REFCURSOR
)
LANGUAGE plpgsql
AS $$
BEGIN
    OPEN p_cursor FOR
        SELECT o.id, o.total, o.status, o.created_at, u.username
        FROM orders o
        JOIN users u ON u.id = o.user_id
        WHERE o.created_at BETWEEN p_start_date AND p_end_date
        ORDER BY o.created_at DESC
        LIMIT p_limit;
END;
$$;
```

```java
// Calling stored procedure with JPA
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

### 4.2 Spring Data JPA @Procedure

```sql
CREATE OR REPLACE FUNCTION calculate_order_discount(
    order_id BIGINT,
    customer_tier VARCHAR
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
public class Order {
    // entity fields
}
```

```java
// Repository using @Procedure
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Procedure(name = "calculateOrderDiscount")
    BigDecimal calculateDiscount(@Param("order_id") Long orderId,
                                  @Param("customer_tier") String customerTier);

    @Procedure(procedureName = "calculate_order_discount")
    BigDecimal calculateDiscountDirect(Long orderId, String customerTier);
}
```

### 4.3 Complex Reporting Procedure

```sql
CREATE OR REPLACE PROCEDURE generate_daily_sales_report(
    p_report_date DATE,
    INOUT p_report JSONB
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_summary JSONB;
    v_top_products JSONB;
    v_category_breakdown JSONB;
BEGIN
    -- Summary
    SELECT JSONB_BUILD_OBJECT(
        'total_revenue', SUM(total),
        'total_orders', COUNT(*),
        'avg_order_value', AVG(total),
        'total_items', SUM(quantity)
    ) INTO v_summary
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.id
    WHERE o.created_at::date = p_report_date
      AND o.status = 'COMPLETED';

    -- Top products
    SELECT JSONB_AGG(sub) INTO v_top_products
    FROM (
        SELECT p.name, SUM(oi.quantity) AS units_sold,
               SUM(oi.quantity * oi.unit_price) AS revenue
        FROM order_items oi
        JOIN products p ON p.id = oi.product_id
        JOIN orders o ON o.id = oi.order_id
        WHERE o.created_at::date = p_report_date AND o.status = 'COMPLETED'
        GROUP BY p.id, p.name
        ORDER BY revenue DESC
        LIMIT 10
    ) sub;

    -- Category breakdown
    SELECT JSONB_AGG(sub) INTO v_category_breakdown
    FROM (
        SELECT c.name AS category, COUNT(*) AS order_count,
               SUM(oi.quantity * oi.unit_price) AS revenue
        FROM order_items oi
        JOIN products p ON p.id = oi.product_id
        JOIN categories c ON c.id = p.category_id
        JOIN orders o ON o.id = oi.order_id
        WHERE o.created_at::date = p_report_date AND o.status = 'COMPLETED'
        GROUP BY c.id, c.name
    ) sub;

    p_report := JSONB_BUILD_OBJECT(
        'report_date', p_report_date,
        'summary', v_summary,
        'top_products', COALESCE(v_top_products, '[]'::JSONB),
        'category_breakdown', COALESCE(v_category_breakdown, '[]'::JSONB)
    );
END;
$$;
```

### 4.4 Batch Processing with Cursor

```sql
CREATE OR REPLACE PROCEDURE process_pending_orders(
    p_batch_size INT DEFAULT 100
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_order RECORD;
    v_processed INT := 0;
    v_cursor CURSOR FOR
        SELECT id, total, user_id
        FROM orders
        WHERE status = 'PENDING'
        ORDER BY created_at
        FOR UPDATE SKIP LOCKED
        LIMIT p_batch_size;
BEGIN
    OPEN v_cursor;
    LOOP
        FETCH v_cursor INTO v_order;
        EXIT WHEN NOT FOUND;

        UPDATE orders
        SET status = 'PROCESSING',
            updated_at = CURRENT_TIMESTAMP
        WHERE id = v_order.id;

        INSERT INTO order_processing_log(order_id, processed_at)
        VALUES (v_order.id, CURRENT_TIMESTAMP);

        v_processed := v_processed + 1;
    END LOOP;
    CLOSE v_cursor;

    RAISE NOTICE 'Processed % orders', v_processed;
END;
$$;
```

## 5. Real-World Scenarios

### 5.1 Database Migration Script (Stored Procedure for Schema Changes)

```sql
CREATE OR REPLACE PROCEDURE migrate_user_status()
LANGUAGE plpgsql
AS $$
BEGIN
    -- Add new column if not exists
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_name = 'users' AND column_name = 'account_tier'
    ) THEN
        ALTER TABLE users ADD COLUMN account_tier VARCHAR(20) DEFAULT 'STANDARD';
    END IF;

    -- Migrate data
    UPDATE users
    SET account_tier = CASE
        WHEN lifetime_value > 10000 THEN 'PREMIUM'
        WHEN lifetime_value > 5000 THEN 'GOLD'
        WHEN lifetime_value > 1000 THEN 'SILVER'
        ELSE 'STANDARD'
    END;

    -- Add index after migration
    CREATE INDEX IF NOT EXISTS idx_users_tier ON users(account_tier)
    WHERE account_tier IN ('PREMIUM', 'GOLD');
END;
$$;
```

### 5.2 Data Archival Procedure

```sql
CREATE OR REPLACE PROCEDURE archive_old_orders(p_cutoff_date DATE)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Insert into archive
    INSERT INTO orders_archive
    SELECT * FROM orders WHERE created_at < p_cutoff_date;

    -- Get count
    GET DIAGNOSTICS v_count = ROW_COUNT;

    -- Delete from main table in small batches to avoid lock contention
    LOOP
        DELETE FROM orders
        WHERE id IN (
            SELECT id FROM orders
            WHERE created_at < p_cutoff_date
            LIMIT 1000
        );
        EXIT WHEN NOT FOUND;
        COMMIT;
    END LOOP;

    RAISE NOTICE 'Archived % orders', v_count;
END;
$$;
```

### 5.3 Spring Boot Transaction Boundaries vs Stored Procedure Transactions

```java
@Service
@Transactional
public class OrderService {
    // Method-level transaction wraps calls to repositories
    // @Transactional ensures all-or-nothing
    public Order createOrder(OrderRequest request) {
        Order order = new Order();
        order.setTotal(request.getTotal());
        order.setStatus("PENDING");
        order = orderRepository.save(order);

        for (OrderItemRequest item : request.getItems()) {
            OrderItem orderItem = new OrderItem();
            orderItem.setOrderId(order.getId());
            orderItem.setProductId(item.getProductId());
            orderItem.setQuantity(item.getQuantity());
            orderItemRepository.save(orderItem);

            // Decrement inventory
            inventoryRepository.decrementStock(item.getProductId(), item.getQuantity());
        }

        return order;
    }
}
```

When to use stored procedures vs application-level transactions:
- **Application transaction**: Simple CRUD, across multiple services, complex business logic
- **Stored procedure**: Data-intensive operations, set-based processing, tight consistency requirements

## 6. Performance

### 6.1 When Stored Procedures Excel

- **Reduced network round trips**: Multiple operations in one call
- **Set-based processing**: SQL set operations are faster than row-by-row in application
- **Server-side filtering**: Move data-intensive computation to server
- **Pre-compiled plans**: No parse overhead for repeated calls (minimal benefit with modern plan caching)

### 6.2 When Stored Procedures Hurt

- **CPU-intensive logic**: Application servers scale horizontally better than databases
- **Complex branching**: Procedural code is harder to debug and profile in the database
- **Read-heavy workloads**: Caching in application (Redis) is more effective
- **Frequent deployments**: Changing procedures requires database schema changes (though Flyway/Liquibase help)

### 6.3 Performance Pitfalls

```sql
-- BAD: Row-by-row processing (slow)
CREATE OR REPLACE PROCEDURE slow_process()
LANGUAGE plpgsql
AS $$
DECLARE
    r RECORD;
BEGIN
    FOR r IN SELECT * FROM orders WHERE status = 'PENDING' LOOP
        UPDATE orders SET status = 'PROCESSED' WHERE id = r.id;
    END LOOP;
END;
$$;

-- GOOD: Set-based operation (fast)
CREATE OR REPLACE PROCEDURE fast_process()
LANGUAGE sql
AS $$
    UPDATE orders SET status = 'PROCESSED' WHERE status = 'PENDING';
$$;
```

## 7. Security

### 7.1 SQL Injection Protection

```sql
-- Use parameters, not string concatenation
CREATE OR REPLACE PROCEDURE find_user_by_email(p_email TEXT)
LANGUAGE SQL
AS $$
    -- SAFE: parameterized query
    SELECT * FROM users WHERE email = p_email;
$$;
```

### 7.2 SECURITY DEFINER for Privilege Escalation

```sql
-- Users have no direct table access; only through procedures
CREATE PROCEDURE get_own_profile(p_user_id INT)
LANGUAGE SQL
SECURITY DEFINER
AS $$
    SELECT id, username, email, created_at
    FROM users
    WHERE id = p_user_id;
$$;

-- Revoke direct table access
REVOKE ALL ON users FROM app_role;
GRANT EXECUTE ON PROCEDURE get_own_profile TO app_role;
```

## 8. Common Mistakes

1. **Procedural, not set-based thinking** — SQL is designed for set operations, not row-by-row loops
2. **Overusing stored procedures for simple CRUD** — application-level JPA is simpler and testable
3. **Ignoring error handling** — unhandled exceptions can leave partial transactions
4. **Hard to unit test** — stored procedures require database setup; consider integration tests
5. **Version control challenges** — ensure procedures are in migrations (Flyway/Liquibase)
6. **Debugging difficulty** — RAISE NOTICE / DBMS_OUTPUT for debugging, not breakpoints
7. **Hidden complexity** — business logic in two places (app + DB) creates maintenance burden
8. **No type safety** — weak typing compared to Java/Kotlin
9. **Stateless concerns** — procedures cannot access caches, external APIs, or services
10. **Performance estimation** — hard to assess cost without executing

## 9. Senior Engineer Perspective

### When to Use Stored Procedures

**Good fit for stored procedures:**
- Batch processing (nightly ETL, data archival, report generation)
- Complex reporting with multiple aggregation levels
- Data validation rules that must be enforced at database level
- Operations requiring tight transaction control (financial transfers)
- Operations on large datasets that would be slow to transfer to application

**Bad fit for stored procedures:**
- Simple CRUD operations (JPA handles these well)
- Business logic requiring external API calls
- Logic that changes frequently (slow deployment cycle)
- Compute-intensive operations (scales better in application tier)
- Operations requiring diverse testing scenarios (testability is harder in DB)

### Microservices and Stored Procedures

In microservices:
- Avoid cross-service stored procedures (database coupling)
- Each service owns its database; procedures should be service-scoped
- Consider event-driven architecture instead of cross-service coordination

## 10. Interview Questions (20)

### Easy (10)

1. What is a stored procedure?
2. What is the difference between a stored procedure and a function?
3. How do you call a stored procedure from SQL?
4. What languages can be used to write stored procedures?
5. What are the benefits of stored procedures?
6. What is a parameter in a stored procedure?
7. What does SECURITY DEFINER mean?
8. How do you handle errors in a stored procedure?
9. Can stored procedures return result sets?
10. What is the difference between IN and OUT parameters?

### Medium (10)

11. How do you call a stored procedure from Spring Data JPA?
12. Explain plan caching for stored procedures. How does it differ from ad-hoc queries?
13. How do you debug a stored procedure?
14. What is the difference between SQL and PL/pgSQL procedures?
15. How do you use cursors in a stored procedure? When are they appropriate?
16. Explain exception handling in PL/pgSQL with WHEN OTHERS.
17. How do you manage transactions inside a stored procedure?
18. What is the difference between SECURITY INVOKER and SECURITY DEFINER?
19. How would you write a stored procedure that returns paginated results?
20. Explain how stored procedures interact with connection pooling.

## 11. Advanced Interview Questions (20)

### Hard (10)

1. How does PostgreSQL implement plan caching for statements inside PL/pgSQL?
2. Explain the concept of "plan generic vs custom" and how the database chooses between them.
3. How do you implement retry logic inside a stored procedure for serialization failures?
4. Write a stored procedure that performs a bulk insert with conflict resolution (upsert).
5. How does the database handle nested procedure calls? What happens to transaction state?
6. Explain the difference between autonomous transactions and regular transactions in Oracle/PostgreSQL.
7. How would you implement a job queue using stored procedures?
8. Write a stored procedure that implements the saga pattern for distributed transactions.
9. How do you prevent stored procedure plan bloat in the procedure cache?
10. Explain the concept of "statement-level" vs "transaction-level" triggers and their interaction with procedures.

### System Design (11-20)

11. Design a system where stored procedures are versioned and deployed with zero downtime.
12. How would you design a testing framework for stored procedures in CI/CD?
13. Design a migration strategy from stored procedures to application-level logic (decomposing the monolith).
14. How would you implement role-based access control entirely through stored procedures?
15. Design a stored-procedure-based ETL pipeline for nightly data warehousing.
16. Design a monitoring system for stored procedure performance (execution time, frequency, plans).
17. How would you design a stored procedure framework that supports dynamic SQL safely?
18. Design a multi-tenant database where stored procedures handle tenant isolation.
19. How would you design a stored procedure that generates and executes dynamic pivot queries?
20. Design a system that migrates a subset of stored procedures to application-level code for scalability.

## 12. Expert-Level Interview Questions (10)

1. Design a stored procedure framework that implements event sourcing with automatic versioning.
2. How would you implement a distributed transaction coordinator using stored procedures across multiple databases?
3. Design a stored procedure that performs online DDL (schema changes) without locking, with progress reporting.
4. How would you implement a PL/pgSQL compiler that translates stored procedures into JVM bytecode for faster execution?
5. Design a stored procedure caching and invalidation strategy for a multi-node database cluster.
6. How would you implement a stored procedure debugger that supports breakpoints and step-through execution?
7. Design a system that automatically refactors procedural row-by-row processing into set-based operations.
8. How would you implement optimistic locking inside a stored procedure without using application-level version fields?
9. Design a stored procedure that implements multi-level sales commission calculation with recursion limits.
10. How would you implement a cost-based optimizer for PL/pgSQL that chooses between procedural and SQL execution paths?

## 13. Debugging & Troubleshooting

### PostgreSQL Procedure Debugging

```sql
-- Enable notices
SET client_min_messages TO DEBUG;
CALL process_pending_orders();

-- Use RAISE for debugging
CREATE OR REPLACE PROCEDURE debug_procedure()
LANGUAGE plpgsql
AS $$
DECLARE
    v_count INT;
BEGIN
    GET DIAGNOSTICS v_count = ROW_COUNT;
    RAISE NOTICE 'Rows affected: %', v_count;
    RAISE DEBUG 'Current user: %', current_user;
    RAISE LOG 'Procedure started at %', CURRENT_TIMESTAMP;
END;
$$;

-- Query current running procedures
SELECT pid, query, state, wait_event
FROM pg_stat_activity
WHERE query LIKE 'CALL%'
   OR query LIKE '%FROM%pg_proc%';
```

### Hibernate/JPA Stored Procedure Logging

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.proc: DEBUG
    org.springframework.jdbc.core: TRACE
```

## 14. Comparison Section

| Aspect | Stored Procedure | Application Logic (Java/Spring) |
|--------|-----------------|-------------------------------|
| Execution location | Database server | Application server |
| Network traffic | Minimal (just parameters) | Multiple round trips |
| Horizontal scaling | Difficult (vertically scale DB) | Easy (add application instances) |
| Testing | Complex (needs database) | Simple (mocks, embedded DB) |
| Deployment | Schema migration | Application deploy |
| Language | PL/pgSQL, T-SQL, PL/SQL | Java, Kotlin |
| Reusability | Only from SQL/database | Across services, APIs |
| Caching | Not typically cached | Redis, in-memory caches |
| Monitoring | pg_stat_statements, slow query log | APM tools, metrics |
| Version control | Migration scripts | Source code repos |

## 15. Revision Notes

- Stored procedures execute on database server, reducing network round trips
- Use set-based logic, not row-by-row (major performance difference)
- Prefer set-based SQL over explicit cursors in most cases
- Use parameterized queries inside procedures to prevent SQL injection
- Choose SECURITY DEFINER for controlled privilege escalation
- Spring Data JPA @Procedure provides clean integration
- Debug with RAISE NOTICE in PostgreSQL, DBMS_OUTPUT in Oracle
- Procedures in microservices should be single-service scoped
- Flyway + stored procedures: use V__ migration files for procedure definitions
- Monitor procedure performance via pg_stat_statements

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                 STORED PROCEDURES CHEAT SHEET                     |
+-------------------------------------------------------------------+
|                                                                   |
|  PL/pgSQL SYNTAX:                                                 |
|                                                                   |
|  CREATE OR REPLACE PROCEDURE name(param_list)                     |
|  LANGUAGE plpgsql                                                 |
|  [SECURITY DEFINER | SECURITY INVOKER]                            |
|  AS $$                                                            |
|  DECLARE                                                          |
|      variables;                                                    |
|  BEGIN                                                             |
|      -- logic here;                                               |
|  EXCEPTION                                                         |
|      WHEN condition THEN                                           |
|      -- error handler;                                            |
|  END;                                                              |
|  $$;                                                               |
|                                                                   |
+-------------------------------------------------------------------+
|  PARAMETER MODES:                                                 |
|                                                                   |
|  IN       - input parameter (default)                              |
|  OUT      - output parameter                                      |
|  INOUT    - input and output                                      |
|  REFCURSOR - cursor for result sets                               |
|                                                                   |
+-------------------------------------------------------------------+
|  FLOW CONTROL:                                                    |
|                                                                   |
|  IF cond THEN statements                                          |
|  [ELSIF cond THEN statements]                                     |
|  [ELSE statements]                                                |
|  END IF;                                                          |
|                                                                   |
|  CASE expr                                                        |
|      WHEN val THEN statements                                     |
|      ELSE statements                                              |
|  END CASE;                                                        |
|                                                                   |
|  LOOP                                                             |
|      statements;                                                  |
|      EXIT WHEN cond;                                              |
|  END LOOP;                                                        |
|                                                                   |
|  FOR i IN 1..10 LOOP                                              |
|      statements;                                                  |
|  END LOOP;                                                        |
|                                                                   |
|  FOR r IN SELECT... LOOP                                          |
|      -- process row r;                                            |
|  END LOOP;                                                        |
|                                                                   |
+-------------------------------------------------------------------+
|  ERROR HANDLING:                                                  |
|                                                                   |
|  EXCEPTION                                                        |
|      WHEN SQLSTATE '23505' THEN  -- unique violation              |
|          RAISE NOTICE 'Duplicate key';                            |
|      WHEN OTHERS THEN                                             |
|          RAISE;                                                    |
+-------------------------------------------------------------------+
|  SPRING BOOT JPA INTEGRATION:                                     |
|                                                                   |
|  @Procedure(procedureName = "proc_name")                          |
|  ReturnType methodName(@Param("p") ParamType param);              |
|                                                                   |
|  // Or programmatic:                                              |
|  StoredProcedureQuery q = em.createStoredProcedureQuery("proc")   |
|      .registerStoredProcedureParameter("p", type, mode)           |
|      .setParameter("p", value);                                   |
|  q.execute();                                                     |
|                                                                   |
+-------------------------------------------------------------------+
|  BEST PRACTICES:                                                  |
|                                                                   |
|  [ ] Use set-based operations, avoid row-by-row                   |
|  [ ] Always parameterize inputs                                   |
|  [ ] Use SECURITY DEFINER for admin operations                    |
|  [ ] Include RAISE NOTICE for debug logging                       |
|  [ ] Handle exceptions with WHEN OTHERS                          |
|  [ ] Keep procedures focused (single responsibility)              |
|  [ ] Test with both small and large datasets                      |
|  [ ] Version control via Flyway/Liquibase migrations              |
|  [ ] Monitor execution time in production                         |
+-------------------------------------------------------------------+
