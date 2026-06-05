# Database Transactions

---

## What is a Database Transaction?

A **database transaction** is a unit of work that is executed **atomically**, **consistently**, **in isolation**, and **durably** — the **ACID** properties. Transactions are the foundation of data integrity in concurrent systems. In Spring Boot, transactions are managed declaratively via `@Transactional`, programmatically via `TransactionTemplate`, or at the JDBC connection level.

### Key Concepts:

1. **ACID Properties**:

   - **Atomicity** — All operations in the transaction complete, or none do. It's all-or-nothing.
   - **Consistency** — The database remains in a valid state. Constraints, triggers, and invariants are maintained.
   - **Isolation** — Concurrent transactions do not interfere with each other. Each transaction appears to run alone.
   - **Durability** — Committed changes persist even after crashes or restarts (guaranteed by the **Write-Ahead Log**).

2. **Transaction States**:

   ```
   Active -> Partially Committed -> Committed
   Active -> Failed -> Aborted
   ```

   - **Active** — Initial state; statements are executing.
   - **Partially Committed** — All statements executed; waiting for the final commit.
   - **Committed** — All changes made permanent.
   - **Failed** — An error occurred during execution.
   - **Aborted** — Changes have been rolled back.

3. **Spring Transaction Management**:

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

   Spring creates an AOP proxy around `@Transactional` classes. When a transactional method is called, the proxy starts a transaction before the method executes and commits/rolls back after it completes.

4. **Transaction Attributes in Spring**:

   - **`propagation`** — How transactions relate to each other (default: `REQUIRED`).
   - **`isolation`** — How changes are visible to other transactions (default: database-specific).
   - **`timeout`** — Maximum seconds before automatic rollback.
   - **`readOnly`** — Hint for read optimization; Hibernate skips dirty checking.
   - **`rollbackFor`** / **`noRollbackFor`** — Which exceptions trigger rollback.

5. **Propagation Behaviors**:

   - **`REQUIRED`** — Join current transaction or create a new one. Default.
   - **`REQUIRES_NEW`** — Suspend current transaction, create an independent new one.
   - **`NESTED`** — Create a savepoint within the current transaction (JDBC only).
   - **`MANDATORY`** — Must join an existing transaction; throw if none exists.
   - **`NEVER`** — Must not run within a transaction.
   - **`NOT_SUPPORTED`** — Suspend current transaction, run non-transactionally.
   - **`SUPPORTS`** — Join if exists, run non-transactionally if not.

---

## Core Concepts

### 1. Rollback Behavior

   By default, Spring rolls back for `RuntimeException` and `Error`, but **not** for checked exceptions:

   ```java
   @Transactional(rollbackFor = {OrderFailedException.class},
                  noRollbackFor = {BusinessWarningException.class})
   public void processOrder(Order order) { ... }
   ```

### 2. How @Transactional Works Under the Hood

   ```
   @Transactional method called
       → TransactionInterceptor intercepts (AOP proxy)
           → PlatformTransactionManager.getTransaction()
               → DataSourceTransactionManager (or JpaTransactionManager)
                   → Connection.commit() / rollback()
   ```

### 3. Self-Invocation Problem

   Calling a `@Transactional` method from within the same class **bypasses the proxy**:

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

   **Fix**: Extract the method into a separate bean, inject a self-reference (`@Autowired`), or use `TransactionTemplate`.

### 4. Transactional Events

   Execute logic only after a transaction successfully commits:

   ```java
   @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
   public void handleOrderPlaced(OrderPlacedEvent event) {
       emailService.sendConfirmation(event.getOrderId());
   }
   ```

---

## Common Mistakes

1. **Self-invocation of `@Transactional` methods** — The proxy is bypassed, so no transaction is started.

2. **Catching and swallowing exceptions inside `@Transactional`** — Spring never sees the exception, so the transaction commits despite the error. Always re-throw or call `setRollbackOnly()`.

3. **`@Transactional` on private methods** — Ignored because AOP proxies cannot intercept private methods.

4. **Long-running transactions** — Hold database locks longer, increasing contention and reducing throughput. Keep transactions short.

5. **Not setting `rollbackFor` for checked exceptions** — Checked exceptions do not trigger rollback by default. Always explicitly configure `rollbackFor` when needed.

6. **`REQUIRES_NEW` exhausting the connection pool** — Each `REQUIRES_NEW` holds a separate database connection. Use sparingly.

---

## Real-World Scenarios

### 1. Money Transfer Between Accounts

A banking application must debit one account and credit another atomically. A single transaction wrapping both operations guarantees atomicity:

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

### 2. Order Processing with Independent Audit Logging

An e-commerce platform must save the order and send a confirmation email. The audit log must persist even if the main transaction rolls back:

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

### 3. Long-Running Batch with Checkpointing

A nightly batch processes 1M invoices. If it crashes at 90%, restarting from the beginning wastes hours. Flushing and clearing the session periodically enables resume:

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

## Scenario-Based Questions

1. **Q: You are building an order system where placing an order must deduct inventory, charge the customer, and send a confirmation. The payment gateway call takes 5 seconds. How do you structure transactions to avoid holding database locks for 5 seconds?**
   A: Split into multiple short transactions. Reserve inventory with a short transaction (`UPDATE inventory SET reserved = reserved + 1 WHERE id = ?`). Call the payment gateway outside any transaction. Confirm the order in a new transaction. If payment fails, release the reservation in a compensating transaction (Saga pattern).

2. **Q: A `@Transactional` method catches all exceptions and logs them, but the transaction still commits even when a database constraint is violated. Why?**
   A: Catching the exception prevents Spring's transaction interceptor from seeing it, so it assumes success. To roll back while handling the error, re-throw a RuntimeException or call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.

3. **Q: A `@Transactional(propagation = Propagation.REQUIRES_NEW)` method is called in a loop. After 50 iterations, the application hangs. What's happening?**
   A: Each `REQUIRES_NEW` suspends the current transaction and acquires a new database connection. If your pool has 50 connections, the 51st iteration deadlocks waiting for a connection held by a suspended transaction. Use `REQUIRES_NEW` sparingly or enlarge the pool.

4. **Q: A service method without `@Transactional` calls a `@Transactional` method on a different bean. Transaction A works, transaction B works, but combined, B's changes appear before A commits. Why?**
   A: Each `@Transactional` method runs in its own transaction (REQUIRED default). When called from a non-transactional context, each creates an independent transaction. The database may flush B's changes before A's. Use a single `@Transactional` on the entry point to wrap both.

5. **Q: Your app uses `@Transactional(readOnly = true)` for a reporting endpoint. Users can still successfully write data through this endpoint. Why isn't it preventing writes?**
   A: `readOnly = true` is a hint to Hibernate to skip dirty checking — it does not prevent writes at the database level. Most databases don't enforce read-only at the transaction level. Use database-level privileges or a read-only replica to truly prevent writes.

6. **Q: A transaction saves an entity and fires a `@TransactionalEventListener(phase = AFTER_COMMIT)`. The event handler queries the DB but doesn't see the saved entity. What's wrong?**
   A: The entity may not be flushed yet if FlushMode is AUTO and no query triggers a flush. Ensure `entityManager.flush()` is called before the transaction completes, or add `@Transactional(propagation = REQUIRES_NEW)` on the event handler.

7. **Q: Two concurrent transactions both read an account balance of $100, both add $50, and both write $150. The balance should be $200 but ends at $150. You're using READ COMMITTED. How do you prevent this?**
   A: This is a lost update. Use optimistic locking with `@Version` — the second commit fails with `OptimisticLockException`. Or use pessimistic locking with `@Lock(PESSIMISTIC_WRITE)` issuing `SELECT ... FOR UPDATE`.

8. **Q: A batch job runs in a single `@Transactional` and processes 100K records. It fails at 90K with OOM. All 90K inserts roll back. How do you fix this?**
   A: The single transaction holds all 90K entities in Hibernate's first-level cache. Flush and clear periodically: `entityManager.flush()` and `entityManager.clear()` every 1000 records. Only the last batch is lost on failure.

9. **Q: Your Spring Boot test uses `@Transactional` on the test method, and the test passes. But against a real DB, the scenario fails. Why?**
   A: `@Transactional` on a test rolls back after the test, but the test runs in a single transaction where Hibernate's first-level cache returns the same entity for repeated reads — masking non-repeatable read issues. Remove `@Transactional` from tests that verify transaction behavior.

10. **Q: A method annotated with `@Transactional` calls another method in the same class also `@Transactional(propagation = REQUIRES_NEW)`. The inner method's transaction does not start independently. Why?**
    A: Self-invocation bypasses the AOP proxy. Extract the inner method to a separate Spring bean, inject a self-reference, or use `TransactionTemplate` programmatically.

## Interview Questions

1. **What are the ACID properties?**
   A: Atomicity — all or nothing. Consistency — database remains valid. Isolation — concurrent transactions don't interfere. Durability — committed changes persist after crash.

2. **What happens when a RuntimeException is thrown inside a @Transactional method?**
   A: Spring rolls back for RuntimeException and Error by default. Checked exceptions do NOT trigger rollback. Use `rollbackFor` to customize.

3. **What is the self-invocation problem?**
   A: Calling a `@Transactional` method from within the same class bypasses the AOP proxy, so the annotation is ignored. Fix by extracting to a separate bean or using TransactionTemplate.

4. **What is propagation REQUIRES_NEW?**
   A: Suspends the current transaction and creates an independent new one. The suspended transaction resumes after the new one commits. Each REQUIRES_NEW holds a separate DB connection.

5. **What does @Transactional(readOnly = true) actually do?**
   A: It's a hint to Hibernate to skip dirty checking and set FlushMode to MANUAL. It does NOT prevent writes at the database level. Improves performance by not tracking entity changes.

6. **What is the difference between NESTED and REQUIRES_NEW?**
   A: NESTED uses a savepoint within the current transaction. REQUIRES_NEW suspends and creates an independent one. NESTED is JDBC-only, not supported by JPA.

7. **What is a transactional event listener?**
   A: `@TransactionalEventListener` binds a listener to a transaction phase (AFTER_COMMIT, AFTER_ROLLBACK). The event fires only when the transaction reaches that phase.

8. **What causes a long-running transaction to be problematic?**
   A: Long transactions hold locks longer, increasing contention and deadlock probability. They also delay WAL cleanup and can exhaust connection pools.

9. **How does OSIV interact with transactions?**
   A: OSIV keeps the Hibernate session open for the entire HTTP request, enabling lazy loading in views. It does NOT create a transaction. Disabled by default in Spring Boot 3.x.

10. **How do you ensure a checked exception triggers a transaction rollback?**
    A: Use `@Transactional(rollbackFor = {CheckedException.class})`. By default, only RuntimeException and Error trigger rollback because checked exceptions represent business conditions.

## Developer Recommendations

- **Keep transactions as short as possible** — Holding locks across slow operations (HTTP calls, file I/O, user input) increases contention and deadlock risk. Read in one transaction, process in application code, write in another.

- **Never catch and swallow exceptions inside @Transactional methods** — Swallowing an exception prevents Spring from detecting the failure. Always re-throw or call `setRollbackOnly()` if you must handle the exception locally.

- **Use REQUIRES_NEW sparingly** — Each REQUIRES_NEW holds a separate database connection. In a loop of 50 iterations, this can exhaust a 50-connection pool. Prefer short, independent transactions.

- **Configure explicit rollbackFor for checked exceptions** — Checked exceptions don't trigger rollback by default. If `InsufficientFundsException` should roll back, declare it in `rollbackFor`.

- **Avoid @Transactional on private methods** — AOP proxies cannot intercept private method calls. The annotation is silently ignored on private methods.

- **Flush and clear the Hibernate session in batch operations** — Processing 100K entities without flushing fills the first-level cache, causing OOM. Flush and clear every 1000 records.

- **Extract @Transactional methods to separate beans to avoid self-invocation** — Self-invocation bypasses the proxy. Inject a self-reference or extract to a service with its own `@Transactional` boundary.
