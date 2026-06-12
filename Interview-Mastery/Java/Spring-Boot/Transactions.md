# Spring Transactions

---

## What is Spring Transaction Management?

**Spring Transaction Management** provides a declarative and programmatic way to manage database transactions. It abstracts away the underlying transaction API (JPA, JDBC, JTA) and provides a consistent, annotation-driven approach to transaction handling.

### `@Transactional` Annotation

The cornerstone of Spring's transaction management. Applied to methods or classes to define transaction boundaries:

```java
@Service
public class OrderService {

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        inventoryService.deductStock(order.getItems()); // Same transaction
        return order;
    }
}
```

When `createOrder()` is called, Spring starts a transaction before the method executes and commits it after the method completes. If any exception is thrown, the transaction is rolled back.

### Transaction Attributes

- **`propagation`** — Defines how transactions relate to each other. Default is `REQUIRED`, which joins an existing transaction or creates a new one.
- **`isolation`** — Defines how changes are visible to other concurrent transactions. Default is database-specific, typically `READ_COMMITTED`.
- **`timeout`** — Maximum seconds the transaction can run before automatic rollback. Default is -1 (no timeout), which can hold locks indefinitely.
- **`readOnly`** — Hint for read-optimized transactions. When `true`, Hibernate skips dirty checking for better performance, but does not guarantee write prevention.
- **`rollbackFor`** — Specific exception types that trigger rollback. Default is `RuntimeException` and `Error`, not checked exceptions.
- **`noRollbackFor`** — Specific exception types that do NOT trigger rollback. Use when you want to commit despite certain exceptions.

### Propagation Behaviors

- **`REQUIRED`** — Join the current transaction or create a new one if none exists. This is the default and most commonly used propagation level.
- **`SUPPORTS`** — Join if a transaction exists, run non-transactional otherwise. Useful for read-only helper methods that don't require a transaction.
- **`MANDATORY`** — Must join an existing transaction. Throws an exception if none exists. Use for methods that should only be called within a transaction.
- **`REQUIRES_NEW`** — Suspend the current transaction (if any) and create a new independent one. The inner transaction can commit or roll back independently.
- **`NOT_SUPPORTED`** — Suspend the current transaction and run non-transactionally. Use for operations that should not participate in the transaction.
- **`NEVER`** — Throw an exception if a current transaction exists. Use when a method must never be called within a transaction.
- **`NESTED`** — Create a savepoint within the current transaction. Allows partial rollback (JDBC drivers only, not JPA).

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditTrail(String action) {
    // Always commits independently of parent transaction
}
```

---

## Core Concepts

### Isolation Levels

Isolation levels determine how transaction changes are visible to other concurrent transactions:

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| `READ_UNCOMMITTED` | Possible | Possible | Possible |
| `READ_COMMITTED` | Prevented | Possible | Possible |
| `REPEATABLE_READ` | Prevented | Prevented | Possible |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

- **Dirty Read** — Reading uncommitted changes from another transaction. If that transaction rolls back, you have read invalid data that never existed.
- **Non-repeatable Read** — Reading the same row twice and getting different values because another transaction modified and committed it between the two reads.
- **Phantom Read** — Running the same query twice and getting different rows because another transaction inserted or deleted rows in between the two executions.

### Rollback Behavior

By default, Spring rolls back for `RuntimeException` and `Error`, but not for checked exceptions:

```java
@Transactional(rollbackFor = {OrderFailedException.class, DataIntegrityViolationException.class},
               noRollbackFor = {BusinessWarningException.class})
public void processOrder(Order order) {
    // Rollback for OrderFailedException and DataIntegrityViolationException
    // No rollback for BusinessWarningException
}
```

### How `@Transactional` Works Under the Hood

Spring creates a **CGLIB or JDK proxy** of the `@Transactional` class. When a `@Transactional` method is called:

1. The proxy intercepts the call.
2. `TransactionInterceptor` checks the transaction attributes.
3. It starts a transaction using `PlatformTransactionManager` (e.g., `JpaTransactionManager`).
4. The target method executes.
5. If successful, the transaction is committed. If an exception occurs, it is rolled back.

### Self-Invocation Problem

Calling a `@Transactional` method from within the same class bypasses the proxy:

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        validateOrder(order);
        saveOrder(order);
        sendNotification(order); // NOT transactional!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(Order order) {
        // REQUIRES_NEW won't work — self-invocation bypasses proxy
    }
}
```

**Why**: `this.sendNotification()` calls the method directly on the target object, not on the proxy.

**Fixes:**
- Extract `sendNotification` into a separate bean.
- Inject self-proxy: `@Autowired OrderService self; self.sendNotification(order);`
- Use `TransactionTemplate` programmatically.

### Programmatic Transactions with `TransactionTemplate`

When you need fine-grained control:

```java
@Service
public class PaymentService {
    private final TransactionTemplate transactionTemplate;

    public PaymentService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
        this.transactionTemplate.setIsolation(TransactionDefinition.ISOLATION_REPEATABLE_READ);
        this.transactionTemplate.setTimeout(30);
    }

    public Payment processPayment(Order order) {
        return transactionTemplate.execute(status -> {
            try {
                Payment payment = chargeCustomer(order);
                inventoryService.deduct(order.getItems());
                return payment;
            } catch (PaymentException e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

### Transactional Event Listener

Execute logic only after a transaction commits:

```java
@Component
public class OrderEventListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Only runs if the transaction COMMITS successfully
        emailService.sendConfirmation(event.getOrderId());
    }
}
```

---

## Common Mistakes

- **Self-invocation of `@Transactional` methods** — The transaction is not started because the proxy is bypassed when calling `this.method()`. This *looks correct* because the code compiles and runs without errors — the transaction simply doesn't exist, and no error or warning indicates the problem. Extract the transactional method to a separate bean to ensure the proxy intercepts the call.
- **Catching and swallowing exceptions in `@Transactional` methods** — The transaction does not roll back because Spring never sees the exception propagate. This *looks correct* because the catch block handles the error gracefully, logs it, and the method returns normally — the client gets a successful response despite the database being in an inconsistent state. Either re-throw the exception or call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` explicitly.
- **`@Transactional` on private methods** — Ignored because the proxy cannot intercept private methods. This *looks correct* because the `@Transactional` annotation is present and accepted by the compiler — there is no error, warning, or log message indicating it is being ignored. Only use `@Transactional` on public methods, as proxies can only intercept public method calls.
- **`REQUIRES_NEW` exhausting the connection pool** — Each `REQUIRES_NEW` holds a separate database connection, so nested usage multiplies connection consumption. This *looks correct* because during development with a single user and small connections, the pool never exhausts — the problem only emerges under production load with 50+ concurrent requests each nesting REQUIRES_NEW. Use sparingly in nested scenarios and monitor pool utilization.
- **Long-running transactions** — Hold database locks longer, increasing contention and reducing throughput. This *looks correct* because the transaction works correctly — data is consistent — and without monitoring lock contention, the performance impact is invisible. Keep transactions short by moving I/O operations (REST calls, file uploads) outside the transaction boundary.
- **Not setting `rollbackFor` for checked exceptions** — By default, checked exceptions do not trigger rollback, which can leave the database in an inconsistent state. This *looks correct* because the method compiles, runs, and the checked exception is properly declared — the database inconsistency is invisible until an audit reveals partial data. Always explicitly configure `rollbackFor` when your checked exception should cause a rollback.
- **Using `@Transactional(readOnly = true)` on writes** — Does not prevent writes in all databases, and Hibernate may skip dirty checking, leading to unexpected behavior. This *looks correct* because the annotation suggests the method is read-only, and during testing the write may or may not persist depending on the database — the inconsistency is hard to reproduce. Never rely on `readOnly` as a security mechanism.

---

## Real-World Scenarios

### Scenario 1: Funds Transfer Between Accounts with Rollback

A banking service transfers money between accounts. The debit succeeds, but the credit fails due to a database constraint. Without a transaction, the debit persists and money disappears.

```java
@Service
public class TransferService {
    private final AccountRepository accountRepository;
    private final AuditService auditService;

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();
        from.debit(amount);
        to.credit(amount);
        accountRepository.save(from);
        accountRepository.save(to);
        // If credit fails, both saves are rolled back atomically
    }
}
```

With `@Transactional`, if `to.credit(amount)` throws an exception (e.g., account closed), the `from.debit()` is also rolled back. The caller gets an error, and no money is lost.

### Scenario 2: Order Processing with Independent Audit Logging

An e-commerce order service must save an audit log entry even if the order processing fails. The audit cannot be rolled back — it's a permanent record.

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final AuditService auditService;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        try {
            inventoryService.deductStock(order.getItems());
            paymentService.charge(order.getTotal());
            auditService.log("ORDER_CREATED", order.getId()); // REQUIRES_NEW
            return order;
        } catch (Exception e) {
            auditService.log("ORDER_FAILED", order.getId(), e.getMessage()); // Saves even though order rolls back
            throw e;
        }
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String action, Long entityId, String... details) {
        auditRepository.save(new AuditEntry(action, entityId, details));
    }
}
```

### Scenario 3: Batch Job with Per-Record Transaction Isolation

A nightly batch job processes 50K records. If one record fails, the entire batch should not roll back — only the failed record should be skipped.

```java
@Service
public class BatchProcessor {
    private final TransactionTemplate transactionTemplate;

    public BatchProcessor(PlatformTransactionManager txManager) {
        this.transactionTemplate = new TransactionTemplate(txManager);
        this.transactionTemplate.setPropagationBehavior(
            TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    }

    public void processBatch(List<Record> records) {
        int success = 0, failure = 0;
        for (Record record : records) {
            try {
                transactionTemplate.execute(status -> {
                    processRecord(record);
                    return null;
                });
                success++;
            } catch (Exception e) {
                log.error("Record {} failed: {}", record.getId(), e.getMessage());
                failure++;
            }
        }
        log.info("Batch complete: {} succeeded, {} failed", success, failure);
    }

    private void processRecord(Record record) {
        // Each record gets its own transaction
    }
}
```

---

## Scenario-Based Questions

1. **Q: Your `@Transactional` service method catches a `DataIntegrityViolationException` and logs it but does not rethrow. A customer reports that their order was not created. The database shows no order record. But the method returned normally without an error to the client. What happened?**
   A: This is the classic "swallowed exception" anti-pattern. The `@Transactional` proxy monitors the transaction. When `DataIntegrityViolationException` is thrown, Spring marks the transaction for rollback. But the catch block swallows the exception. The method returns normally, but when the transaction commits, Spring sees it's marked for rollback and throws `UnexpectedRollbackException`. The client gets a 500 despite the method appearing to succeed. Never swallow exceptions in `@Transactional` methods — either rethrow or call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` explicitly.

   > **Interview follow-up:** The candidate identified the swallowed exception correctly. A junior developer on the team suggests fixing this by removing the catch block entirely, letting the exception propagate. If the `@Transactional` method is called from a REST controller, and the exception is a `DataIntegrityViolationException`, what HTTP status code does the client receive, and what error format should the `@RestControllerAdvice` return to ensure the client can parse it programmatically?

2. **Q: Your service method `createOrder()` calls `sendNotification()` in the same class. `sendNotification()` has `@Transactional(propagation = REQUIRES_NEW)` but it runs in the same transaction as `createOrder()`. Why?**
   A: Self-invocation — calling `this.sendNotification()` bypasses the Spring AOP proxy. The `@Transactional` annotation on `sendNotification()` is never processed. Fix by extracting `sendNotification()` into a separate bean:
   ```java
   @Service
   public class OrderService {
       private final NotificationService notificationService;
       @Transactional
       public void createOrder(Order order) {
           orderRepository.save(order);
           notificationService.sendNotification(order); // Goes through proxy — works!
       }
   }
   ```

3. **Q: Your transaction uses `REQUIRES_NEW` for an audit log inside a parent transaction. During load testing, the database connection pool is exhausted and the application hangs. What's happening?**
   A: Each `REQUIRES_NEW` holds a separate database connection — the parent transaction holds one, and the inner transaction acquires another. Under high concurrency, nested `REQUIRES_NEW` calls multiply the connection usage. If 50 concurrent requests each create 3 nested `REQUIRES_NEW` transactions, you need 150 connections even with a pool size of 10. Fix by: (a) reducing nested `REQUIRES_NEW` usage, (b) increasing the connection pool, or (c) using an alternative like `TransactionTemplate` with a separate data source for audit logs.

   > **Interview follow-up:** The candidate suggested reducing nested `REQUIRES_NEW` usage. If the business requirement demands that the audit log commits independently of the parent transaction (cannot use `REQUIRED`), and increasing the connection pool is not an option (database connection limits), which of the remaining fixes would you prioritize, and what production monitoring metric would you use to alert before the pool is exhausted again?

4. **Q: A developer annotates a `private` method with `@Transactional`. The transaction never starts. The developer argues that "the method is only called from within the class, so it's fine." How do you convince them this is a bug?**
   A: Spring creates a proxy that intercepts calls to `@Transactional` methods. Proxies can only intercept `public` methods (CGLIB can proxy protected/package-private but not private). A `private` method is never proxied. The fix is to make the method `public` if it needs transaction management. If it's only called internally, either: (a) make it `public` anyway and document that it's for internal use, or (b) move the transactional logic to a public method that the private method calls, or (c) use `TransactionTemplate` for programmatic transactions within the private method.

5. **Q: You have a `@Transactional(readOnly = true)` method that calls a repository method. The repository method should only read data, but a developer accidentally added a `save()` call inside it. What happens? Will the data be saved?**
   A: With `readOnly = true`, Hibernate sets the flush mode to `MANUAL` and disables dirty checking. The `save()` call may not throw an exception (depending on the database and provider) but the changes will NOT be flushed to the database. However, this behavior is provider-specific — PostgreSQL ignores `readOnly` for DDL but respects it for DML, while MySQL may silently accept writes. Never assume `readOnly = true` prevents writes. Use it as an optimization hint, not a security mechanism.

6. **Q: Your application intermittently gets `Transaction rolled back because it has been marked as rollback-only` even though no exception is thrown. How do you debug this?**
   A: This occurs when a nested method marks the transaction as rollback-only (by throwing an exception or calling `setRollbackOnly()`), but the outer method catches it and continues normally. Enable transaction logging to see which method marks it:
   ```yaml
   logging:
     level:
       org.springframework.transaction: DEBUG
       org.springframework.orm.jpa: DEBUG
   ```
   Check for: (a) A catch block that swallows an exception inside a `@Transactional` method, (b) A `REQUIRES_NEW` method called from a `REQUIRED` transaction where the inner method rolls back but the outer catches it, (c) A `@TransactionalEventListener` that throws an exception.

7. **Q: Your service processes a batch of 10K records in a single `@Transactional` method. The database transaction log fills up and performance degrades. How do you split this into smaller transactions?**
   A: Process in chunks with periodic flushing:
   ```java
   @Service
   public class BulkImportService {
       private final EntityManager entityManager;

       @Transactional
       public void importInBatches(List<Record> records) {
           for (int i = 0; i < records.size(); i++) {
               entityManager.persist(records.get(i));
               if (i > 0 && i % 100 == 0) {
                   entityManager.flush();
                   entityManager.clear(); // Detaches all entities to free memory
               }
           }
       }
   }
   ```
   For truly independent batches, use `TransactionTemplate` in a loop — each batch gets its own transaction, preventing any single transaction from being too large.

8. **Q: You deploy a new version and users report that data updates are not visible for several minutes. The old version used `@Transactional` on service methods. The new version wraps `@Transactional` at the controller level instead. What changed?**
   A: Moving `@Transactional` from the service layer to the controller layer keeps the transaction open for the entire HTTP request, including JSON serialization. This causes: (a) long-held database connections reducing pool availability, (b) lazy-loading queries triggered during serialization running inside the transaction (masking N+1 problems), (c) transaction timeouts on slow serialization. Keep transaction boundaries at the service layer — short and focused. The controller should be transactional only if you explicitly need OSIV-like behavior.

9. **Q: Your `@Transactional` method calls an external REST API. The external API call takes 30 seconds. During this time, the database connection is held idle. How do you prevent this?**
   A: Never perform blocking I/O (REST calls, file uploads, message sending) inside a transaction. Restructure to separate the transaction from the external call:
   ```java
   @Service
   public class OrderService {
       @Transactional
       public Order createOrder(CreateOrderRequest request) {
           Order order = orderRepository.save(new Order(request));
           // Do NOT call external API here — just save the order
           return order;
       }

       public void notifyExternalSystem(Order order) {
           // External API call — no transaction
           restTemplate.postForEntity("https://external.com/notify", order, Void.class);
       }
   }
   ```
   The controller calls `createOrder()` (wrapped in a short transaction) and then `notifyExternalSystem()` (no transaction). If the external call fails, the order is already saved and can be retried later.

10. **Q: You have a `@Transactional` method that uses `Propagation.MANDATORY`. The caller does not have a transaction. What happens?**
    A: `MANDATORY` requires an existing transaction — if none exists, it throws `IllegalTransactionStateException`. This is useful for enforcing that a method is always called within a transaction context. If your method must always participate in an existing transaction and should never start its own, use `MANDATORY`. This is common for internal DAO methods that should only be called from transactional service methods.

---

## Interview Questions

1. **What is `@Transactional` and how does it work?** 
   A: `@Transactional` is a Spring annotation that declaratively manages transactions. Spring creates a proxy around the annotated class. When the method is called, the proxy starts a transaction before execution and commits/rolls back after. It uses a `PlatformTransactionManager` (e.g., `JpaTransactionManager`) to interact with the underlying transaction system.

2. **What are the propagation levels in Spring transactions?** 
   A: `REQUIRED` (default) — join or create a new transaction. `REQUIRES_NEW` — suspend current and create new. `SUPPORTS` — join if exists, run non-transactional otherwise. `MANDATORY` — must join existing, throw otherwise. `NOT_SUPPORTED` — suspend current, run non-transactional. `NEVER` — throw if transaction exists. `NESTED` — savepoint within existing transaction (JDBC only).

3. **What is the default rollback behavior of `@Transactional`?** 
   A: By default, Spring rolls back for `RuntimeException` and `Error`, but NOT for checked exceptions. Use `rollbackFor` to customize: `@Transactional(rollbackFor = {CheckedException.class})`. Use `noRollbackFor` to exclude specific exceptions from rollback.

4. **What is the self-invocation problem and how do you fix it?** 
   A: Self-invocation occurs when a method in the same class calls another `@Transactional` method — the call bypasses the proxy, so the annotation is ignored. Fix by: (a) extracting the method to a separate bean, (b) injecting a self-reference, or (c) using `TransactionTemplate` programmatically.

5. **What is `TransactionTemplate` and when do you use it?** 
   A: `TransactionTemplate` provides programmatic transaction management. Use it when you need fine-grained control (e.g., per-record transactions in a batch loop) or when you need transactions inside `@PostConstruct` methods where AOP proxies are not yet available.

6. **What are the isolation levels in Spring transactions?** 
   A: `READ_UNCOMMITTED` — dirty reads possible. `READ_COMMITTED` — dirty reads prevented. `REPEATABLE_READ` — dirty and non-repeatable reads prevented. `SERIALIZABLE` — all concurrency issues prevented. Higher isolation = better consistency but lower concurrency. Default is database-specific (typically `READ_COMMITTED`).

7. **What is the difference between `NESTED` and `REQUIRES_NEW`?** 
   A: `NESTED` uses a savepoint within the current transaction — if it rolls back, only changes since the savepoint are undone, and the outer transaction can continue. `REQUIRES_NEW` suspends the current transaction entirely and creates a completely independent one. `NESTED` is only supported by JDBC (not JPA).

8. **What is OSIV (Open Session in View) and why is it problematic?** 
   A: OSIV keeps the Hibernate session open for the entire HTTP request, allowing lazy loading in the view layer. It is problematic because: (a) it holds database connections longer than necessary, (b) it masks N+1 problems by silently executing lazy queries, and (c) it can cause transactions to span across service and view layers. Disabled by default in Spring Boot 3.x.

9. **How does `@Transactional(readOnly = true)` improve performance?** 
   A: It tells Hibernate to skip dirty checking and set flush mode to `MANUAL`, reducing overhead. Some JPA providers also optimize read-only queries (e.g., no need to acquire write locks). It also serves as documentation — developers know the method should not modify data.

10. **How do you test `@Transactional` behavior?** 
    A: Use `@SpringBootTest` with `@Transactional` on the test method (auto-rollback after each test). Assert database state before and after method calls. Use `assertThrows` to verify rollback on exceptions. For testing `REQUIRES_NEW`, inject the inner service separately. Verify no data was persisted after a rollback scenario.

---

## Developer Recommendations

- **Use `@Transactional` at the service layer, not the controller or repository layer** — Service-layer transactions group multiple repository calls into a single atomic unit. Repository-level transactions are too granular (each save is a separate transaction). Controller-level transactions hold connections too long (through JSON serialization).
- **Never perform blocking I/O (REST calls, file uploads) inside a transaction** — Transactions hold database connections and locks. An external API call inside a transaction keeps the connection idle for seconds, exhausting the pool. Always separate I/O operations from transactional logic.
- **Never swallow exceptions in `@Transactional` methods** — Catching an exception and not rethrowing it leaves the transaction marked for rollback. Spring throws `UnexpectedRollbackException` on commit. Always rethrow or call `setRollbackOnly()` explicitly if you must catch the exception.
- **Use `REQUIRES_NEW` sparingly and be aware of connection pool impact** — Each `REQUIRES_NEW` acquires a separate connection. Nested `REQUIRES_NEW` in a loop multiplies connection usage and can exhaust the pool. Prefer `TransactionTemplate` for batch processing.
- **Extract `@Transactional` methods that need separate transaction boundaries into separate beans** — Self-invocation bypasses the proxy. Calling `this.transactionalMethod()` ignores the annotation. Always call transactional methods on a different bean instance.
- **Use `@Transactional(readOnly = true)` for read-only queries** — It enables Hibernate optimizations (skip dirty checking, set flush mode to MANUAL) and documents the method's intent. However, do not rely on it as a security mechanism — some databases ignore the hint.
- **Set explicit timeouts on long-running transactions** — `@Transactional(timeout = 30)` prevents a transaction from running indefinitely due to database contention or deadlock. Default is no timeout, which can hold locks forever.
- **Use `@TransactionalEventListener` for logic that should run after a transaction commits** — This ensures side effects (email, events, notifications) only happen if the transaction succeeds. `TransactionPhase.AFTER_COMMIT` runs only after a successful commit, avoiding the "email sent but order failed" scenario.
