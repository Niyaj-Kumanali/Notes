# Spring Transactions

---

## 1. Executive Summary

### What Is It?
Spring Transaction Management provides a declarative (annotation-based) and programmatic way to manage database transactions. It abstracts away the underlying transaction API (JPA, JDBC, JTA) and provides consistent transaction handling.

### Transaction Attributes

| Attribute | Description | Default |
|-----------|-------------|---------|
| `propagation` | How transactions relate to each other | `REQUIRED` |
| `isolation` | How changes are visible to other transactions | `DEFAULT` (DB-specific) |
| `timeout` | Max seconds before rollback | -1 (no timeout) |
| `readOnly` | Hint for read-optimized transaction | `false` |
| `rollbackFor` | Exceptions that trigger rollback | Runtime exceptions only |
| `noRollbackFor` | Exceptions that DON'T trigger rollback | — |

---

## 2. Core Theory

### @Transactional

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

### Propagation

| Propagation | Behavior |
|-------------|----------|
| `REQUIRED` | Join current transaction or create new one |
| `SUPPORTS` | Join if exists, run non-transactional if not |
| `MANDATORY` | Join current tx, throw if none exists |
| `REQUIRES_NEW` | Suspend current tx, create new one |
| `NOT_SUPPORTED` | Suspend current tx, run non-transactional |
| `NEVER` | Throw if current tx exists |
| `NESTED` | Savepoint within current tx (JDBC) |

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditTrail(String action) {
    // Always commits independently of parent transaction
    // If parent rolls back, this audit entry persists
}
```

### Isolation Levels

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| `READ_UNCOMMITTED` | ✅ Possible | ✅ Possible | ✅ Possible |
| `READ_COMMITTED` | ❌ Prevented | ✅ Possible | ✅ Possible |
| `REPEATABLE_READ` | ❌ | ❌ | ✅ Possible |
| `SERIALIZABLE` | ❌ | ❌ | ❌ |

- **Dirty Read**: Read uncommitted changes from another transaction
- **Non-repeatable Read**: Same row read twice, different values (changed by another tx)
- **Phantom Read**: Same query returns different rows (inserted by another tx)

### Rollback Behavior

```java
@Transactional(rollbackFor = {OrderFailedException.class, DataIntegrityViolationException.class},
               noRollbackFor = {BusinessWarningException.class})
public void processOrder(Order order) {
    // Rollback for OrderFailedException and DataIntegrityViolationException
    // No rollback for BusinessWarningException
}
```

**Default:** Runtime exceptions → rollback. Checked exceptions → no rollback.

---

## 3. Under-the-Hood

### How @Transactional Works

1. Spring creates a **CGLIB/JDK proxy** of the `@Transactional` class
2. When `@Transactional` method is called, the proxy intercepts the call
3. Transaction interceptor (`TransactionInterceptor`) manages begin/commit/rollback
4. Uses `PlatformTransactionManager` (e.g., `JpaTransactionManager`)

### Self-Invocation Problem

```java
@Service
public class OrderService {
    
    @Transactional
    public void placeOrder(Order order) {
        validateOrder(order);
        saveOrder(order);
        sendNotification(order); // Not transactional!
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(Order order) {
        // REQUIRES_NEW won't work when called from within the same class!
    }
}
```

**Why:** Self-invocation calls `this.sendNotification()` — bypasses the proxy. No transaction management.

**Fixes:**
1. Extract into separate bean
2. Inject self-proxy: `@Autowired OrderService self; self.sendNotification(order);`
3. Use `TransactionTemplate` programmatically

### TransactionTemplate (Programmatic)

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

---

## 4. Production Code

### 4.1 Transactional Event Listener

```java
@Component
public class OrderEventListener {
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Only runs if the transaction COMMITS successfully
        // If transaction rolls back, this event handler never runs
        emailService.sendConfirmation(event.getOrderId());
    }
}
```

### 4.2 Chained Transactions (REQUIRES_NEW)

```java
@Service
public class OrderService {
    private final AuditService auditService;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        // Even if this service's transaction rolls back,
        // audit entry is saved (independent transaction)
        auditService.log("ORDER_CREATED", order.getId());
        return order;
    }
}

@Service
public class AuditService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String action, Long entityId) {
        auditRepository.save(new AuditEntry(action, entityId));
    }
}
```

---

## 5. Common Mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Self-invocation bypassing proxy | Transaction not started | Extract to separate bean |
| 2 | Swallowing exceptions in @Transactional | No rollback | Don't catch runtime exceptions in transactional methods |
| 3 | @Transactional on private methods | Ignored (proxy can't intercept) | Use public methods only |
| 4 | REQUIRES_NEW exhausts connections | Connection pool starves | Limit nested REQUIRES_NEW |
| 5 | Long transactions (many operations) | Lock contention, stale data | Keep transactions short |
| 6 | Not setting rollbackFor for checked exceptions | Transaction commits despite error | Add rollbackFor = Exception.class |
| 7 | Reading in one tx, writing in another | Non-repeatable reads | Use REPEATABLE_READ or @Transactional on read |
| 8 | Spring test without @Rollback | Test data persists | @Transactional on tests auto-rolls back |

---

## 6. Cheat Sheet

```
═══ SPRING TRANSACTIONS ═══════════════════════════════════════

┌─ PROPAGATION ──────────────────────────────────────────────┐
│ REQUIRED       — join or create (default)                   │
│ REQUIRES_NEW   — suspend, create new (independent)          │
│ NESTED         — savepoint (partial rollback)               │
│ MANDATORY      — must exist                                 │
│ SUPPORTS       — join if exists                             │
│ NOT_SUPPORTED  — non-transactional                          │
│ NEVER          — throw if exists                            │
└─────────────────────────────────────────────────────────────┘

┌─ ISOLATION ────────────────────────────────────────────────┐
│ READ_UNCOMMITTED   — dirty, non-repeatable, phantom         │
│ READ_COMMITTED     — no dirty, yes non-repeatable, phantom  │
│ REPEATABLE_READ    — no dirty, no non-repeatable, phantom   │
│ SERIALIZABLE       — all prevented (slowest)               │
└─────────────────────────────────────────────────────────────┘

┌─ ROLLBACK ─────────────────────────────────────────────────┐
│ Default: RuntimeExceptions → rollback                       │
│ Checked exceptions → commit                                 │
│ override: rollbackFor / noRollbackFor                       │
└─────────────────────────────────────────────────────────────┘

┌─ KEY ANNOTATIONS ──────────────────────────────────────────┐
│ @Transactional(propagation, isolation, timeout,             │
│                readOnly, rollbackFor, noRollbackFor)        │
│ @TransactionalEventListener(phase = AFTER_COMMIT)           │
│ @EnableTransactionManagement (auto in Spring Boot)          │
└─────────────────────────────────────────────────────────────┘

┌─ BEST PRACTICES ───────────────────────────────────────────┐
│ • Keep transactions short                                  │
│ • Avoid self-invocation of @Transactional methods           │
│ • Use @Transactional(readOnly = true) for read operations   │
│ • Don't catch and swallow exceptions in tx methods           │
│ • Use REQUIRES_NEW sparingly (consumes connections)         │
│ • Test rollback behavior explicitly                        │
└─────────────────────────────────────────────────────────────┘
```
