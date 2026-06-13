# Database Transactions

---

## What is a Database Transaction?

- **Definition**
  - A **database transaction** is a unit of work that is executed **atomically**, **consistently**, **in isolation**, and **durably** — the **ACID** properties. Transactions are the foundation of data integrity in concurrent systems. In Spring Boot, transactions are managed declaratively via `@Transactional`, programmatically via `TransactionTemplate`, or at the JDBC connection level.

### ACID Properties

- **Atomicity** — All operations in the transaction complete successfully, or none of them do. It is all-or-nothing. If any operation fails, the entire transaction rolls back.
- **Consistency** — The database remains in a valid state before and after the transaction. All constraints, triggers, and data invariants are maintained throughout.
- **Isolation** — Concurrent transactions do not interfere with each other. Each transaction appears to run alone, even though many may be executing simultaneously.
- **Durability** — Once a transaction is committed, its changes persist even after a crash or restart. This guarantee is provided by the **Write-Ahead Log** (WAL).

### Transaction States

```
Active -> Partially Committed -> Committed
Active -> Failed -> Aborted
```

- **Active** — The initial state where SQL statements are executing within the transaction.
- **Partially Committed** — All statements have executed successfully, waiting for the final commit command.
- **Committed** — All changes have been made permanent and are visible to other transactions.
- **Failed** — An error occurred during execution, preventing normal completion.
- **Aborted** — All changes have been rolled back and the transaction has been undone completely.

### Spring Transaction Management

```java
@Service
@Transactional
public class OrderService {
    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        inventoryService.deductStock(order.getItems());
        return order;
    }
}
```

- Spring creates an AOP proxy around `@Transactional` classes. When a transactional method is called, the proxy starts a transaction before the method executes and commits or rolls back after it completes.

### Transaction Attributes in Spring

- **`propagation`** — Defines how transactions relate to each other (default: `REQUIRED`).
- **`isolation`** — Controls how changes are visible to other transactions (default: database-specific).
- **`timeout`** — Maximum seconds before the transaction automatically rolls back.
- **`readOnly`** — A hint for read optimization; Hibernate skips dirty checking when set to true.
- **`rollbackFor` / `noRollbackFor`** — Specifies which exception types should or should not trigger rollback.

### Propagation Behaviors

- **`REQUIRED`** — Joins the current transaction or creates a new one if none exists. This is the default and most common propagation behavior.
- **`REQUIRES_NEW`** — Suspends the current transaction and creates an independent new one. Each `REQUIRES_NEW` holds a separate database connection.
- **`NESTED`** — Creates a savepoint within the current transaction (JDBC only, not supported by JPA). Partial rollback to the savepoint is possible.
- **`MANDATORY`** — Must join an existing transaction; throws an exception if none exists.
- **`NEVER`** — Must not run within a transaction; throws an exception if one is active.
- **`NOT_SUPPORTED`** — Suspends the current transaction and runs non-transactionally.
- **`SUPPORTS`** — Joins a transaction if one exists, runs non-transactionally if not.

---

## Core Concepts

### Rollback Behavior

- By default, Spring rolls back for `RuntimeException` and `Error`, but **not** for checked exceptions:

```java
@Transactional(rollbackFor = {OrderFailedException.class},
               noRollbackFor = {BusinessWarningException.class})
public void processOrder(Order order) { ... }
```

- Checked exceptions represent business conditions that may not warrant a rollback. Always configure `rollbackFor` explicitly when a checked exception should trigger a rollback.

### How @Transactional Works Under the Hood

```
@Transactional method called
    → TransactionInterceptor intercepts (AOP proxy)
        → PlatformTransactionManager.getTransaction()
            → DataSourceTransactionManager (or JpaTransactionManager)
                → Connection.commit() / rollback()
```

- The AOP proxy is the key mechanism. When you call a `@Transactional` method from another bean, the proxy intercepts the call and manages the transaction lifecycle.

### Self-Invocation Problem

- Calling a `@Transactional` method from within the same class **bypasses the proxy**:

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        sendNotification(order); // NOT transactional — self-invocation!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(Order order) { ... }
}
```

- **Fix**: Extract the method into a separate Spring bean, inject a self-reference (`@Autowired`), or use `TransactionTemplate` programmatically.

### Transactional Events

- Execute logic only after a transaction successfully commits:

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleOrderPlaced(OrderPlacedEvent event) {
    emailService.sendConfirmation(event.getOrderId());
}
```

- `@TransactionalEventListener` binds the listener to a specific transaction phase. The event is only dispatched when the transaction reaches that phase, preventing actions on rolled-back data.

---

## Common Mistakes

- **Self-invocation of `@Transactional` methods**
  - The AOP proxy is bypassed, so no transaction is started. Extract the method to a separate bean to fix this.
  - **Why it looks correct:** Both methods are in the same service class and the annotation is clearly present; the proxy bypass is invisible at the source code level.

- **Catching and swallowing exceptions inside `@Transactional`**
  - Spring never sees the exception, so the transaction commits despite the error. Always re-throw or call `setRollbackOnly()`.
  - **Why it looks correct:** Standard exception handling patterns (try/catch/log) work everywhere else in Java; the silent commit happens because Spring's interceptor never receives the exception signal.

- **`@Transactional` on private methods**
  - AOP proxies cannot intercept private method calls, so the annotation is silently ignored on private methods.
  - **Why it looks correct:** The annotation compiles and the IDE doesn't warn; the silent failure only surfaces when the database shows uncommitted partial changes.

- **Long-running transactions**
  - Holding database locks across slow operations increases contention and reduces throughput. Keep transactions as short as possible.
  - **Why it looks correct:** A single transaction seems safer (atomicity guarantees), and the contention cost is invisible until concurrent requests pile up waiting for the same locked rows.

- **Not setting `rollbackFor` for checked exceptions**
  - Checked exceptions do not trigger rollback by default. Always explicitly configure `rollbackFor` when a checked exception should cause a rollback.
  - **Why it looks correct:** An exception being thrown intuitively should roll back the transaction; the Spring default of only rolling back on RuntimeException is a design decision that surprises most developers.

- **`REQUIRES_NEW` exhausting the connection pool**
  - Each `REQUIRES_NEW` holds a separate database connection. Using them in a loop can exhaust a 50-connection pool after 50 iterations.
  - **Why it looks correct:** Each individual `REQUIRES_NEW` call works fine; the exhaustion only manifests when the loop count exceeds the pool size, which may never happen in test environments.

---

## Real-World Scenarios

### Money Transfer Between Accounts

- A banking application must debit one account and credit another atomically. A single transaction wrapping both operations guarantees atomicity:

```java
@Service
@Transactional
public class TransferService {
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepo.findById(fromId).orElseThrow();
        Account to = accountRepo.findById(toId).orElseThrow();
        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
    }
}
```

- If either operation fails, the entire transaction rolls back — no money is lost.

### Order Processing with Independent Audit Logging

- An e-commerce platform must save the order and send a confirmation email. The audit log must persist even if the main transaction rolls back:

```java
@Service
public class OrderService {
    @Transactional
    public Order placeOrder(OrderRequest req) {
        Order order = orderRepo.save(new Order(req));
        inventoryService.deductStock(req.getItems());
        auditService.logAudit(req); // REQUIRES_NEW — persists independently
        return order;
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAudit(OrderRequest req) {
        auditRepo.save(new AuditEntry("ORDER", req.getUserId()));
    }
}
```

- The audit entry commits independently, surviving any rollback in the calling transaction.

### Long-Running Batch with Checkpointing

- A nightly batch processes 1M invoices. If it crashes at 90%, restarting from the beginning wastes hours. Flushing and clearing the session periodically enables resume:

```java
@Transactional
public void processInvoices() {
    for (int i = 0; i < 1000; i++) {
        List<Invoice> batch = invoiceRepo.findPendingBatch(1000);
        if (batch.isEmpty()) break;
        batch.forEach(inv -> inv.setStatus("PROCESSED"));
        invoiceRepo.saveAll(batch);
        entityManager.flush();
        entityManager.clear();
    }
}
```

- Only the current batch is lost on failure, not the entire 1M records.

## Scenario-Based Questions

**Q: You are building an order system where placing an order must deduct inventory, charge the customer, and send a confirmation. The payment gateway call takes 5 seconds. How do you structure transactions to avoid holding database locks for 5 seconds?**

- Split into multiple short transactions. Reserve inventory with a short transaction (`UPDATE inventory SET reserved = reserved + 1 WHERE id = ?`). Call the payment gateway outside any transaction. Confirm the order in a new transaction. If payment fails, release the reservation in a compensating transaction (Saga pattern).
- **Interview follow-up:** Between the inventory reservation and the payment confirmation, another request times out the reservation. How do you handle expired reservations without overselling?

---

**Q: A `@Transactional` method catches all exceptions and logs them, but the transaction still commits even when a database constraint is violated. Why?**

- Catching the exception prevents Spring's transaction interceptor from seeing it, so it assumes success. To roll back while handling the error, re-throw a RuntimeException or call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.
- **Interview follow-up:** The method calls `setRollbackOnly()` and then proceeds to call an external API. The external API succeeds. Now the database rollback contradicts the external side effect. How do you prevent this?

---

**Q: A `@Transactional(propagation = Propagation.REQUIRES_NEW)` method is called in a loop. After 50 iterations, the application hangs. What's happening?**

- Each `REQUIRES_NEW` suspends the current transaction and acquires a new database connection. If your pool has 50 connections, the 51st iteration deadlocks waiting for a connection held by a suspended transaction. Use `REQUIRES_NEW` sparingly or enlarge the pool.
- **Interview follow-up:** You enlarge the pool to 200 connections. Now the database server's `max_connections` is reached, and all other services lose connectivity. How do you bound the total connections across all application instances?

---

**Q: A service method without `@Transactional` calls a `@Transactional` method on a different bean. Transaction A works, transaction B works, but combined, B's changes appear before A commits. Why?**

- Each `@Transactional` method runs in its own transaction when called from a non-transactional context. The database may flush B's changes before A's. Use a single `@Transactional` on the entry point to wrap both operations.

---

**Q: Your app uses `@Transactional(readOnly = true)` for a reporting endpoint. Users can still successfully write data through this endpoint. Why isn't it preventing writes?**

- `readOnly = true` is a hint to Hibernate to skip dirty checking — it does not prevent writes at the database level. Most databases don't enforce read-only at the transaction level. Use database-level privileges or a read-only replica to truly prevent writes.

---

**Q: A transaction saves an entity and fires a `@TransactionalEventListener(phase = AFTER_COMMIT)`. The event handler queries the DB but doesn't see the saved entity. What's wrong?**

- The entity may not be flushed yet if FlushMode is AUTO and no query triggers a flush. Ensure `entityManager.flush()` is called before the transaction completes, or add `@Transactional(propagation = REQUIRES_NEW)` on the event handler.

---

**Q: Two concurrent transactions both read an account balance of $100, both add $50, and both write $150. The balance should be $200 but ends at $150. You're using READ COMMITTED. How do you prevent this?**

- This is a lost update. Use optimistic locking with `@Version` — the second commit fails with `OptimisticLockException`. Or use pessimistic locking with `@Lock(PESSIMISTIC_WRITE)` issuing `SELECT ... FOR UPDATE`.

---

**Q: A batch job runs in a single `@Transactional` and processes 100K records. It fails at 90K with OOM. All 90K inserts roll back. How do you fix this?**

- The single transaction holds all 90K entities in Hibernate's first-level cache. Flush and clear periodically: `entityManager.flush()` and `entityManager.clear()` every 1000 records. Only the last batch is lost on failure.

---

**Q: Your Spring Boot test uses `@Transactional` on the test method, and the test passes. But against a real DB, the scenario fails. Why?**

- `@Transactional` on a test rolls back after the test, but the test runs in a single transaction where Hibernate's first-level cache returns the same entity for repeated reads — masking non-repeatable read issues. Remove `@Transactional` from tests that verify transaction behavior.

---

**Q: A method annotated with `@Transactional` calls another method in the same class also `@Transactional(propagation = REQUIRES_NEW)`. The inner method's transaction does not start independently. Why?**

- Self-invocation bypasses the AOP proxy, so the inner `@Transactional` is ignored. Extract the inner method to a separate Spring bean, inject a self-reference, or use `TransactionTemplate` programmatically.

## Interview Questions

- **What are the ACID properties?**
  - Atomicity — all or nothing. Consistency — database remains valid. Isolation — concurrent transactions don't interfere. Durability — committed changes persist after crash.

- **What happens when a RuntimeException is thrown inside a @Transactional method?**
  - Spring rolls back for RuntimeException and Error by default. Checked exceptions do NOT trigger rollback. Use `rollbackFor` to customize this behavior.

- **What is the self-invocation problem?**
  - Calling a `@Transactional` method from within the same class bypasses the AOP proxy, so the annotation is ignored. Fix by extracting to a separate bean or using TransactionTemplate.

- **What is propagation REQUIRES_NEW?**
  - Suspends the current transaction and creates an independent new one. The suspended transaction resumes after the new one commits. Each REQUIRES_NEW holds a separate DB connection.

- **What does @Transactional(readOnly = true) actually do?**
  - It is a hint to Hibernate to skip dirty checking and set FlushMode to MANUAL. It does NOT prevent writes at the database level. Improves performance by not tracking entity changes.

- **What is the difference between NESTED and REQUIRES_NEW?**
  - NESTED uses a savepoint within the current transaction and can roll back partially. REQUIRES_NEW suspends and creates a fully independent transaction. NESTED is JDBC-only, not supported by JPA.

- **What is a transactional event listener?**
  - `@TransactionalEventListener` binds a listener to a transaction phase (AFTER_COMMIT, AFTER_ROLLBACK). The event fires only when the transaction reaches that phase.

- **What causes a long-running transaction to be problematic?**
  - Long transactions hold locks longer, increasing contention and deadlock probability. They also delay WAL cleanup and can exhaust connection pools.

- **How does OSIV interact with transactions?**
  - OSIV keeps the Hibernate session open for the entire HTTP request, enabling lazy loading in views. It does NOT create a transaction. Disabled by default in Spring Boot 3.x.

- **How do you ensure a checked exception triggers a transaction rollback?**
  - Use `@Transactional(rollbackFor = {CheckedException.class})`. By default, only RuntimeException and Error trigger rollback because checked exceptions represent business conditions.

## Developer Recommendations

- **Keep transactions as short as possible**
  - Holding locks across slow operations (HTTP calls, file I/O, user input) increases contention and deadlock risk. Read in one transaction, process in application code, write in another.
  - **Production story:** A team wrapped an entire REST endpoint in `@Transactional`, including a slow file upload. Under load, all concurrent requests queued behind the upload transaction, causing a 30-second p99 latency and exhausting the connection pool within minutes of deployment.

- **Never catch and swallow exceptions inside @Transactional methods**
  - Swallowing an exception prevents Spring from detecting the failure. Always re-throw or call `setRollbackOnly()` if you must handle the exception locally.
  - **Production story:** A payment processing service caught all exceptions in a `@Transactional` method to log them. A constraint violation on the audit log caused silent commit of incomplete financial records, leading to a $50K reconciliation effort and a 3-day incident.

- **Use REQUIRES_NEW sparingly**
  - Each REQUIRES_NEW holds a separate database connection. In a loop of 50 iterations, this can exhaust a 50-connection pool. Prefer short, independent transactions.

- **Configure explicit rollbackFor for checked exceptions**
  - Checked exceptions don't trigger rollback by default. If `InsufficientFundsException` should roll back, declare it in `rollbackFor`.

- **Avoid @Transactional on private methods**
  - AOP proxies cannot intercept private method calls. The annotation is silently ignored on private methods.

- **Flush and clear the Hibernate session in batch operations**
  - Processing 100K entities without flushing fills the first-level cache, causing OOM. Flush and clear every 1000 records.

- **Extract @Transactional methods to separate beans to avoid self-invocation**
  - Self-invocation bypasses the proxy. Inject a self-reference or extract to a service with its own `@Transactional` boundary.
