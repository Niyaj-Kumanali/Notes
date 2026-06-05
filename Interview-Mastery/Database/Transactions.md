# Transactions

## 1. Executive Summary

A database transaction is a unit of work that is executed atomically, consistently, in isolation, and durably (ACID). Transactions are fundamental to data integrity in concurrent systems. In Spring Boot, transactions are managed declaratively via @Transactional annotations, programmatically via TransactionTemplate, or at the JDBC connection level. Understanding transaction boundaries, propagation behavior, and rollback rules is essential for building reliable enterprise applications.

## 2. Core Theory

### 2.1 ACID Properties

| Property | Description | Guarantee |
|----------|-------------|-----------|
| Atomicity | All or nothing execution | Transaction either fully commits or fully rolls back |
| Consistency | Database remains in valid state | Constraints, triggers, and invariants are maintained |
| Isolation | Concurrent transactions don't interfere | Each transaction appears to run alone |
| Durability | Committed changes persist | Survives crashes and restarts |

### 2.2 Transaction States

```
Active -> Partially Committed -> Committed
Active -> Failed -> Aborted
```

- **Active**: Initial state, statements executing
- **Partially Committed**: All statements executed, waiting for commit
- **Committed**: All changes made permanent
- **Failed**: Error occurred during execution
- **Aborted**: Changes rolled back

### 2.3 Transaction Lifecycle

```java
@Transactional
public void businessMethod() {
    // 1. Tx begins (or joins existing)
    // 2. Operations execute
    // 3. On success: tx commits
    // 4. On exception: tx rolls back
    // 5. Resources released
}
```

### 2.4 Savepoints

```sql
BEGIN;
INSERT INTO orders ...;
SAVEPOINT order_created;
INSERT INTO order_items ...;
-- If order_items fails:
ROLLBACK TO SAVEPOINT order_created;
-- Order still exists, can retry items or do something else
COMMIT;
```

## 3. Under-the-Hood Deep Dive

### 3.1 Transaction Manager in Spring

```
@Transactional
    -> TransactionInterceptor (AOP proxy)
        -> PlatformTransactionManager.getTransaction()
            -> DataSourceTransactionManager (or JpaTransactionManager)
                -> Connection.commit() / rollback()
```

### 3.2 Transaction Attributes

- **Propagation**: How transactions relate to each other (REQUIRED, REQUIRES_NEW, NESTED, etc.)
- **Isolation**: How transaction changes are visible (READ_COMMITTED, REPEATABLE_READ, etc.)
- **Timeout**: Maximum duration before automatic rollback
- **Read-only**: Hint for optimizations (no flush at commit, no dirty checking)
- **Rollback rules**: Which exceptions trigger rollback

### 3.3 Auto-Commit Mode

By default, JDBC connections are in auto-commit mode. Spring's @Transactional disables auto-commit and manages commit/rollback. With auto-commit, each SQL statement is its own transaction.

### 3.4 Transaction Log (Write-Ahead Log)

The transaction log (WAL) records all changes before they are written to the data files:
```
Transaction Start -> Write WAL records -> Write data pages -> Transaction Commit (WAL flush)
```

- **WAL flush**: On commit, WAL records are fsynced to disk (durability)
- **Redo**: On recovery, committed transactions in WAL are replayed
- **Undo**: On rollback or crash recovery, uncommitted changes are rolled back using the log

## 4. Production Code Examples

### 4.1 Basic @Transactional Usage

```java
@Service
@Transactional  // Class-level: all public methods are transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;

    @Transactional(readOnly = true)  // Method-level overrides class-level
    public Order getOrder(Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Order not found"));
    }

    @Transactional  // Default: readWrite, Propagation.REQUIRED
    public Order createOrder(OrderRequest request) {
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setStatus("PENDING");
        order = orderRepository.save(order);

        for (OrderItemRequest item : request.getItems()) {
            inventoryService.reserve(item.getProductId(), item.getQuantity());
        }

        return order;
    }
}
```

### 4.2 Transaction Propagation

```java
@Service
@Transactional
public class PaymentService {

    @Transactional(propagation = Propagation.REQUIRED)
    // Default: uses existing transaction or creates new one
    public void processPayment(Payment payment) {
        paymentRepository.save(payment);
        paymentGateway.charge(payment);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    // Always creates new transaction; suspends existing one if present
    public AuditRecord logAudit(AuditRecord audit) {
        // This will commit even if outer transaction rolls back
        return auditRepository.save(audit);
    }

    @Transactional(propagation = Propagation.NESTED)
    // JDBC savepoint; allows partial rollback within outer transaction
    public void processLineItem(LineItem item) {
        // If this fails, only this line item is rolled back
        lineItemRepository.save(item);
    }

    @Transactional(propagation = Propagation.MANDATORY)
    // Requires an existing transaction; throws if none exists
    public void criticalOperation() {
        // Must be called within a transaction
    }

    @Transactional(propagation = Propagation.NEVER)
    // Must NOT run within a transaction
    public void nonTransactionalOp() {
        // Throws if called within a transaction
    }

    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    // Suspends any existing transaction; runs non-transactionally
    public void batchProcess() {
        // Runs without transaction overhead
    }

    @Transactional(propagation = Propagation.SUPPORTS)
    // Uses existing transaction if present; runs without one otherwise
    public void optionalTransaction() {
        // Works either way
    }
}
```

### 4.3 Rollback Rules

```java
@Service
@Transactional(rollbackFor = Exception.class)
// Rollback for ALL exceptions (including checked)
public class TransferService {

    // Rollback for specific exceptions, noRollback for others
    @Transactional(rollbackFor = {DataAccessException.class, InsufficientFundsException.class},
                   noRollbackFor = {BusinessValidationException.class})
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // DataAccessException -> rollback
        // InsufficientFundsException -> rollback
        // BusinessValidationException -> no rollback (validation error, not data issue)
    }
}
```

### 4.4 Programmatic Transaction Management

```java
@Service
public class TransactionTemplateService {

    private final TransactionTemplate transactionTemplate;

    public TransactionTemplateService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
        this.transactionTemplate.setPropagationBehavior(
            TransactionDefinition.PROPAGATION_REQUIRES_NEW);
        this.transactionTemplate.setIsolationLevel(
            TransactionDefinition.ISOLATION_REPEATABLE_READ);
        this.transactionTemplate.setTimeout(30);
    }

    public Result performInTransaction() {
        return transactionTemplate.execute(status -> {
            try {
                // Transactional operations here
                Result result = doWork();
                return result;
            } catch (Exception e) {
                // Explicit rollback
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

### 4.5 Transactional Events

```java
@Service
public class OrderCreatedEventListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCreated(OrderCreatedEvent event) {
        // This runs AFTER the transaction commits
        // If the transaction rolls back, this never runs
        sendConfirmationEmail(event.getOrderId());
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleOrderFailed(OrderCreatedEvent event) {
        // Cleanup when order creation fails
        cleanup(event.getOrderId());
    }
}
```

### 4.6 Chained Transactions Across Repositories

```java
@Service
@Transactional
public class CompleteOrderFlow {

    private final OrderRepository orderRepository;
    private final PaymentRepository paymentRepository;
    private final InventoryRepository inventoryRepository;

    public Order executeCompleteFlow(OrderRequest request) {
        // All operations in ONE transaction
        Order order = createOrder(request);
        Payment payment = processPayment(order, request.getPaymentInfo());
        reserveInventory(order);

        if (payment.getStatus() == "FAILED") {
            throw new PaymentFailedException("Payment declined");
        }

        return order;
    }

    private Order createOrder(OrderRequest request) {
        return orderRepository.save(Order.from(request));
    }

    private Payment processPayment(Order order, PaymentInfo info) {
        Payment payment = new Payment(order.getId(), info);
        return paymentRepository.save(payment);
    }

    private void reserveInventory(Order order) {
        for (OrderItem item : order.getItems()) {
            inventoryRepository.decrementStock(item.getProductId(), item.getQuantity());
        }
    }
}
```

## 5. Real-World Scenarios

### 5.1 Transaction Across Multiple Microservices (Saga Pattern)

```java
// NOT ACID across services; use Saga pattern
@Service
public class OrderSagaOrchestrator {

    @Transactional
    public void createOrderSaga(OrderRequest request) {
        // Step 1: Create order (local transaction)
        Order order = orderRepository.save(Order.from(request));

        try {
            // Step 2: Reserve inventory (remote service call)
            inventoryClient.reserve(order.getId(), order.getItems());

            // Step 3: Process payment (remote service call)
            paymentClient.charge(order.getId(), order.getTotal());
        } catch (Exception e) {
            // Compensating transactions on failure
            inventoryClient.release(order.getId(), order.getItems());
            order.setStatus("FAILED");
            orderRepository.save(order);
            throw e;
        }
    }
}
```

### 5.2 Reading Uncommitted Data (Dirty Read Prevention)

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void readOnlyCheck() {
    // READ_COMMITTED ensures we never see uncommitted data
    // This is the default for most databases
    BigDecimal total = reportRepository.calculateTotalRevenue();
    return total;
}
```

### 5.3 Optimistic Locking with Transactions

```java
@Transactional
public void updateWithOptimisticLock(Long id, String newData) {
    // Read
    Entity entity = repository.findById(id).get();
    entity.setData(newData);
    // Write at commit: JPA checks @Version hasn't changed
    // If changed since read: OptimisticLockException thrown
}
```

## 6. Performance

### 6.1 Transaction Duration Guidelines

| Duration | Classification | Risk |
|----------|---------------|------|
| < 10ms | Ideal | Minimal contention |
| 10-100ms | Normal | Acceptable |
| 100ms-1s | Long | Increased contention risk |
| > 1s | Very long | High risk of blocking/lock escalation |
| > 10s | Danger | Likely timeout, deadlock |

### 6.2 Transaction Overhead

Each transaction adds:
- **Connection acquisition** from pool
- **Transaction begin** (WAL log sequence number advance)
- **WAL flush on commit** (disk I/O)
- **Lock acquisition and release**
- **Transaction manager overhead** (AOP interception in Spring)

### 6.3 Reducing Transaction Overhead

```java
// BAD: separate transactions for each item
public void processItemsBad(List<Item> items) {
    items.forEach(this::processItem);  // N transactions
}

// GOOD: single transaction for batch
@Transactional
public void processItemsGood(List<Item> items) {
    items.forEach(this::processItem);  // 1 transaction
}

// GOOD: read-only optimization
@Transactional(readOnly = true)
public List<Data> readBatch() {
    // Hibernate skips dirty checking, no flush at commit
}
```

## 7. Security

### 7.1 Transaction Rollback and Data Leakage

When a transaction rolls back, the application may have already used the data in subsequent operations. Ensure:
- Rolled-back data is not sent to external systems
- Generated IDs (from sequences) are not reused after rollback (PostgreSQL sequences increment even on rollback)
- Temp tables created in a transaction are cleaned up on rollback

### 7.2 Timeout-Based DoS Prevention

Set transaction timeouts to prevent slow transactions from blocking resources:
```java
@Transactional(timeout = 30)  // seconds
public void potentiallySlowOperation() {
    // Will be rolled back after 30 seconds
}
```

## 8. Common Mistakes

1. **Self-invocation bypasses @Transactional** — calling a @Transactional method from the same class bypasses the proxy
2. **Catching exceptions within @Transactional** — if you catch the exception, Spring doesn't know to rollback
3. **Long transactions holding locks** — keep transactions short
4. **@Transactional on private methods** — ignored (AOP works on public methods)
5. **Mixing transaction managers** — JPA + JDBC can conflict without proper configuration
6. **Ignoring transaction propagation defaults** — REQUIRED propagates across nested service calls
7. **Read-write operations in read-only transactions** — may cause unexpected behavior
8. **Not considering @Transactional rollback rules** — checked exceptions don't trigger rollback by default
9. **Overusing REQUIRES_NEW** — each REQUIRES_NEW creates a separate connection
10. **No timeout on long operations** — transactions can block resources indefinitely

## 9. Senior Engineer Perspective

### Transaction Management Best Practices

1. **@Transactional at service layer** — not at controller or repository
2. **Keep transactions as short as possible** — acquire resources late, release early
3. **Read-only transactions on queries** — @Transactional(readOnly=true) for all SELECT operations
4. **Never pass the persistence context to the view** — lazy loading exceptions or large sessions
5. **Explicit rollback rules** — always configure rollbackFor for checked exceptions
6. **Monitor transaction metrics** — commit rate, rollback rate, duration
7. **Use @TransactionalEventListener** — for operations that should happen after successful commit
8. **Consider ChainedTransactionManager** — for distributed transactions across multiple datasources

### Transaction Manager Selection

```java
@Configuration
@EnableTransactionManagement
public class TransactionConfig {

    @Primary
    @Bean
    public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
        JpaTransactionManager tm = new JpaTransactionManager();
        tm.setEntityManagerFactory(emf);
        tm.setDefaultTimeout(30);
        return tm;
    }

    @Bean
    public DataSourceTransactionManager dataSourceTransactionManager(
            DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

## 10. Interview Questions (20)

### Easy (10)

1. What is a database transaction?
2. What does ACID stand for?
3. What is the purpose of commit and rollback?
4. What does the @Transactional annotation do?
5. What is auto-commit mode?
6. What happens when an exception is thrown in a @Transactional method?
7. What is a savepoint?
8. What is the default transaction propagation in Spring?
9. What does @Transactional(readOnly = true) do?
10. What is a transaction timeout?

### Medium (10)

11. Explain the difference between Propagation.REQUIRED and Propagation.REQUIRES_NEW.
12. How does @Transactional work internally in Spring (AOP proxy mechanism)?
13. What is the default rollback behavior of @Transactional?
14. How do you configure rollback for checked exceptions?
15. What is the difference between PROGRAMMATIC and DECLARATIVE transaction management?
16. Explain the ChainedTransactionManager pattern.
17. How do transactions work in a microservices architecture?
18. What is the Saga pattern for distributed transactions?
19. How does Hibernate's persistence context relate to transactions?
20. What is the difference between flush and commit in JPA?

## 11. Advanced Interview Questions (20)

### Hard (10)

1. How does Spring's @Transactional annotation work with AOP proxies? Explain the self-invocation problem.
2. Explain the concept of transaction suspension and how it's implemented.
3. How does the transaction log (WAL) guarantee durability?
4. What is a phantom read and how does SERIALIZABLE isolation prevent it?
5. Explain how distributed transactions work with XA protocol and 2PC.
6. What is the difference between container-managed and bean-managed transactions in Java EE?
7. How does the write-ahead log protocol ensure atomicity and durability?
8. Explain the concept of "compensating transaction" in the Saga pattern.
9. How does transaction isolation interact with connection pooling?
10. What is the difference between logical and physical transactions in Spring?

### System Design (11-20)

11. Design a transaction management system for a microservices architecture (no distributed transactions).
12. How would you design a retry mechanism for transaction failures?
13. Design a system that supports long-running transactions without holding database locks.
14. How would you implement a transaction log reader for change-data-capture (CDC)?
15. Design a transaction monitoring and alerting system.
16. How would you design transaction isolation for a multi-tenant SaaS application?
17. Design a system that supports XA transactions across PostgreSQL, RabbitMQ, and MongoDB.
18. How would you design a system with eventual consistency and compensation for failed transactions?
19. Design a transaction timeout management system that prevents resource leaks.
20. How would you design a distributed transaction tracer across microservices?

## 12. Expert-Level Interview Questions (10)

1. Design a transaction manager that supports both optimistic and pessimistic concurrency control, choosing per-entity.
2. How would you implement multi-level transactions (nested transactions with independent rollback)?
3. Design a system that handles distributed transactions across shards using the TCC (Try-Confirm/Cancel) pattern.
4. How would you implement a transactionally consistent message queue using database transaction logs?
5. Design a system that detects and resolves heuristic (indoubt) transactions in a 2PC setup.
6. How would you design a transaction processing system that can handle 100K+ concurrent transactions per second?
7. Design a horizontally scalable transaction ID generator that preserves causal ordering.
8. How would you implement transaction-scoped caching that invalidates on rollback?
9. Design a system that supports multi-database transactions using the Outbox pattern and CDC.
10. How would you implement a transaction scheduler that optimizes for minimal lock contention?

## 13. Debugging & Troubleshooting

### Transaction Logging

```yaml
logging:
  level:
    org.springframework.transaction: TRACE
    org.springframework.orm.jpa.JpaTransactionManager: DEBUG
    org.hibernate.engine.transaction: DEBUG
    org.hibernate.resource.transaction: DEBUG
```

### Monitoring Transactions

```sql
-- PostgreSQL: find long-running transactions
SELECT pid, age(now(), xact_start) AS duration,
       state, query,
       application_name, client_addr
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND state NOT IN ('idle')
ORDER BY duration DESC;

-- MySQL: InnoDB transaction list
SELECT * FROM information_schema.INNODB_TRX;
SELECT * FROM performance_schema.events_transactions_current;
```

### JPA Transaction Debugging

```java
// Programmatic transaction monitoring
@Component
public class TransactionLogger {
    @EventListener
    public void onTransactionEvent(TransactionPhaseEvent event) {
        log.info("Transaction {}: {}", event.getTransactionId(), event.getPhase());
    }
}
```

## 14. Comparison Section

| Propagation | Behavior | Use Case |
|-------------|----------|----------|
| REQUIRED | Join or create | Default; most service methods |
| REQUIRES_NEW | Suspend existing, create new | Audit logging (commit even if outer fails) |
| NESTED | Savepoint in existing | Sub-operations with partial rollback |
| MANDATORY | Must exist | Internal service calls |
| NEVER | Must not exist | Testing, non-tx operations |
| NOT_SUPPORTED | Suspend, run non-tx | Batch operations |
| SUPPORTS | Optional | Read operations |

| Isolation | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|-----------|-------------------|-------------|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Prevented | Possible | Possible |
| REPEATABLE READ | Prevented | Prevented | Possible |
| SERIALIZABLE | Prevented | Prevented | Prevented |

## 15. Revision Notes

- ACID: Atomicity, Consistency, Isolation, Durability
- @Transactional uses AOP proxy — self-invocation bypasses it
- Default: rollback for RuntimeException/Error, NOT for checked exceptions
- REQUIRED: reuse existing transaction; REQUIRES_NEW: always new
- readOnly=true optimizes read-only transactions
- Keep transactions short to reduce lock contention
- TransactionalEventListener for post-commit actions
- WAL (Write-Ahead Log) guarantees durability
- Saga pattern for distributed transactions
- Always access resources in consistent order to prevent deadlocks

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                    TRANSACTIONS CHEAT SHEET                       |
+-------------------------------------------------------------------+
|                                                                   |
|  ACID PROPERTIES:                                                 |
|                                                                   |
|  A - Atomicity:     All or nothing                                |
|  C - Consistency:   Valid state maintained                         |
|  I - Isolation:     Concurrent transactions don't interfere        |
|  D - Durability:    Committed changes persist                      |
|                                                                   |
+-------------------------------------------------------------------+
|  @Transactional ATTRIBUTES:                                       |
|                                                                   |
|  propagation    -> REQUIRED, REQUIRES_NEW, NESTED, ...            |
|  isolation      -> READ_COMMITTED, REPEATABLE_READ, ...           |
|  timeout        -> seconds before rollback                        |
|  readOnly       -> true/false (optimization hint)                  |
|  rollbackFor    -> exception classes that trigger rollback        |
|  noRollbackFor  -> exception classes that DON'T trigger rollback  |
|                                                                   |
+-------------------------------------------------------------------+
|  PROPAGATION BEHAVIOR:                                            |
|                                                                   |
|  REQUIRED       [   Tx1   ]                                       |
|                 [   Tx1   ]  (same tx)                            |
|                                                                   |
|  REQUIRES_NEW   [   Tx1   ]                                       |
|                   [ Tx2 ]     (suspended)                         |
|                                                                   |
|  NESTED         [   Tx1   ]                                       |
|                 [ [Tx2]  ]  (savepoint within Tx1)                |
|                                                                   |
+-------------------------------------------------------------------+
|  ROLLBACK RULES:                                                  |
|                                                                   |
|  Default:                                                        |
|    RuntimeException -> ROLLBACK                                   |
|    Error           -> ROLLBACK                                    |
|    Checked Exception -> COMMIT (does NOT roll back)               |
|                                                                   |
|  Custom:                                                          |
|    @Transactional(rollbackFor = Exception.class)                  |
|    @Transactional(noRollbackFor = {ValidationException.class})    |
|                                                                   |
+-------------------------------------------------------------------+
|  SPRING TX EVENT PHASES:                                          |
|                                                                   |
|  BEFORE_COMMIT     -> Before transaction commits                  |
|  AFTER_COMMIT      -> After successful commit                     |
|  AFTER_ROLLBACK    -> After rollback                              |
|  AFTER_COMPLETION  -> After commit or rollback                    |
|                                                                   |
+-------------------------------------------------------------------+
|  JPA FLUSH MODES:                                                 |
|                                                                   |
|  AUTO        -> Flush before query and commit (default)           |
|  COMMIT      -> Flush only at commit                              |
|  ALWAYS      -> Flush before every query                          |
|  MANUAL      -> Only explicit flush()                             |
|                                                                   |
+-------------------------------------------------------------------+
|  COMMON ANTI-PATTERNS:                                            |
|                                                                   |
|  [ ] @Transactional on private methods (ignored)                  |
|  [ ] Catching exception in @Transactional (prevents rollback)    |
|  [ ] Long transactions (>1s) holding locks                       |
|  [ ] Mixing REQUIRES_NEW with nested propagation without thought |
|  [ ] Not setting timeout for potentially long operations         |
|  [ ] Reading in one tx, writing in another (non-atomic)          |
|                                                                   |
+-------------------------------------------------------------------+
