# Isolation Levels

## 1. Executive Summary

Isolation levels define how and when changes made by one transaction become visible to other concurrent transactions. They control the trade-off between data consistency and concurrency. The SQL standard defines four isolation levels: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, and SERIALIZABLE. Higher isolation levels prevent more concurrency anomalies but reduce throughput. In Spring Boot/JPA, isolation is configured via @Transactional(isolation = ...) and maps to the underlying database's implementation. Understanding isolation is critical for building correct concurrent applications.

## 2. Core Theory

### 2.1 Concurrency Anomalies

| Anomaly | Description | Occurs At |
|---------|-------------|-----------|
| Dirty Read | Read uncommitted changes from another transaction | READ UNCOMMITTED |
| Non-Repeatable Read | Same row read twice gives different values (row was updated by another tx) | READ UNCOMMITTED, READ COMMITTED |
| Phantom Read | Same query gives different set of rows (rows were inserted/deleted by another tx) | READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ |
| Lost Update | Two transactions read same value, both update, last write overwrites first | All levels (unless explicitly prevented) |
| Dirty Write | Write to uncommitted data | None (prevented at all levels) |
| Read Skew | Inconsistent state due to reading different versions of related data | Below SNAPSHOT/SERIALIZABLE |
| Write Skew | Two transactions read overlapping data and write inconsistently | Below SERIALIZABLE |

### 2.2 SQL Standard Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|-------------------|-------------|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Prevented | Possible | Possible |
| REPEATABLE READ | Prevented | Prevented | Possible |
| SERIALIZABLE | Prevented | Prevented | Prevented |

### 2.3 Snapshot Isolation

Snapshot isolation (not in the original SQL standard but widely implemented):
- Each transaction sees a consistent snapshot of the database as of the transaction start
- Write-write conflicts are detected (first committer wins)
- Prevents dirty reads, non-repeatable reads, and phantoms
- Does NOT prevent write skew
- Used by: PostgreSQL (REPEATABLE READ), Oracle (default), SQL Server (snapshot isolation), MySQL (REPEATABLE READ with MVCC)

## 3. Under-the-Hood Deep Dive

### 3.1 How READ COMMITTED Works

Implementation in MVCC databases:
```
Transaction A: BEGIN
Transaction A: SELECT * FROM accounts WHERE id = 1
    -> Returns current committed version
Transaction B: UPDATE accounts SET balance = 100 WHERE id = 1
Transaction B: COMMIT
Transaction A: SELECT * FROM accounts WHERE id = 1
    -> Returns NEW committed version (balance = 100)
Transaction A: COMMIT
```

- Each statement sees the latest committed data as of statement start
- Row versions: Transaction B creates a new row version on UPDATE; Transaction A sees the new version on its next SELECT

### 3.2 How REPEATABLE READ Works

```
Transaction A: BEGIN (xid = 100)
Transaction A: SELECT * FROM accounts WHERE id = 1
    -> Returns version as of xid=100 snapshot
Transaction B: UPDATE accounts SET balance = 100 WHERE id = 1
Transaction B: COMMIT (xid = 101)
Transaction A: SELECT * FROM accounts WHERE id = 1
    -> Returns SAME version as first SELECT (ignores xid 101 changes)
Transaction A: COMMIT
```

- PostgreSQL REPEATABLE READ: Transaction sees snapshot from first query in transaction
- MySQL REPEATABLE READ (InnoDB): Transaction sees snapshot from first read in transaction
- Both prevent non-repeatable reads using MVCC

### 3.3 How SERIALIZABLE Works

PostgreSQL implements true SERIALIZABLE using Serializable Snapshot Isolation (SSI):
```
Transaction A: BEGIN ISOLATION LEVEL SERIALIZABLE
Transaction A: SELECT SUM(balance) FROM accounts WHERE type = 'CHECKING'
    -> Reads snapshot
Transaction B: BEGIN ISOLATION LEVEL SERIALIZABLE
Transaction B: SELECT SUM(balance) FROM accounts WHERE type = 'SAVINGS'
    -> Reads snapshot
Transaction A: UPDATE accounts SET balance = balance + 100 WHERE id = 1 AND type = 'CHECKING'
    -> Works
Transaction A: COMMIT
    -> Succeeds
Transaction B: UPDATE accounts SET balance = balance - 100 WHERE id = 2 AND type = 'SAVINGS'
    -> Works
Transaction B: COMMIT
    -> FAILS: "could not serialize access due to read/write dependencies"
```

SSI detects serialization anomalies using predicate locks and conflict detection. A transaction that would produce a non-serializable execution is aborted.

### 3.4 Lock-Based vs MVCC Implementations

| Database | READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE |
|----------|-----------------|----------------|-----------------|--------------|
| PostgreSQL | Like RC | MVCC (statement snapshot) | MVCC (tx snapshot) | SSI (true serializable) |
| MySQL/InnoDB | Dirty reads possible | MVCC (statement snapshot) | MVCC (tx snapshot) | Locks + gap locks |
| Oracle | N/A (default RC) | MVCC (statement snapshot) | N/A (use Serializable) | MVCC + ORA-08177 |
| SQL Server | Dirty reads | Lock-based or SNAPSHOT | Lock-based or SNAPSHOT | Lock-based |

### 3.5 Lost Update Prevention

Lost updates require explicit mechanisms:
1. **Pessimistic locking**: `SELECT ... FOR UPDATE`
2. **Optimistic locking**: `@Version` column with retry
3. **Atomic operations**: `UPDATE table SET col = col + 1`
4. **Serializable isolation**: Prevents all anomalies including lost updates

## 4. Production Code Examples

### 4.1 Setting Isolation Levels in Spring

```java
@Service
@Transactional
public class AccountService {

    // Default: database's default (usually READ COMMITTED)
    public BigDecimal getBalance(Long accountId) {
        return accountRepository.findById(accountId)
            .map(Account::getBalance)
            .orElse(BigDecimal.ZERO);
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    // Prevents dirty reads but allows non-repeatable reads
    public void transferReadCommitted(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).get();
        Account to = accountRepository.findById(toId).get();
        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
    }

    @Transactional(isolation = Isolation.REPEATABLE_READ)
    // Prevents dirty reads and non-repeatable reads
    // In PostgreSQL: snapshot isolation (no phantoms in practice)
    public void transferRepeatableRead(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).get();
        Account to = accountRepository.findById(toId).get();
        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    // Full isolation; may get serialization failures requiring retry
    public void transferSerializable(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).get();
        Account to = accountRepository.findById(toId).get();
        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
    }
}
```

### 4.2 Retry for Serialization Failures

```java
@Service
public class RetryableTransferService {

    @Retryable(
        value = CannotSerializeTransactionException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).get();
        Account to = accountRepository.findById(toId).get();
        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
    }

    @Recover
    public void recover(CannotSerializeTransactionException e,
                        Long fromId, Long toId, BigDecimal amount) {
        log.error("Transfer failed after retries: {} -> {} amount {}",
                fromId, toId, amount, e);
        throw new TransferFailedException("Transfer failed, please try again later");
    }
}
```

### 4.3 READ UNCOMMITTED for Reporting

```java
@Service
public class ReportingService {

    @Transactional(isolation = Isolation.READ_UNCOMMITTED)
    // Accepts dirty reads for performance; reporting data needn't be perfectly accurate
    public ReportSummary generateSummary() {
        // These reads may see uncommitted data
        // But it's OK for approximate reports
        long totalOrders = orderRepository.count();
        BigDecimal revenue = orderRepository.calculateTotalRevenue();
        return new ReportSummary(totalOrders, revenue);
    }
}
```

### 4.4 Using SNAPSHOT Isolation (SQL Server)

```java
// SQL Server with Snapshot Isolation
@Transactional(isolation = Isolation.REPEATABLE_READ)
// This maps to SQL Server's SNAPSHOT isolation level if enabled
public List<Order> getOrdersForReporting() {
    // Consistent snapshot without blocking writers
    return orderRepository.findAll();
}
```

### 4.5 Checking Database Isolation Level

```java
@Component
public class IsolationLevelChecker {

    private final JdbcTemplate jdbcTemplate;

    public String getCurrentIsolationLevel() {
        // PostgreSQL
        return jdbcTemplate.queryForObject(
            "SHOW transaction_isolation", String.class);
    }

    public void setSessionIsolationLevel(String level) {
        jdbcTemplate.execute(
            "SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL " + level);
    }
}
```

## 5. Real-World Scenarios

### 5.1 Financial Transfer (SERIALIZABLE or Pessimistic Locking)

```sql
-- SERIALIZABLE transaction
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- Check balance >= amount
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SELECT balance FROM accounts WHERE id = 2 FOR UPDATE;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Without SERIALIZABLE or FOR UPDATE, concurrent transfers could cause lost updates or inconsistent reads.

### 5.2 E-Commerce Cart (READ COMMITTED + Optimistic Lock)

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public boolean addToCart(Long cartId, Long productId, int quantity) {
    // Check stock (may be slightly stale - acceptable)
    Product product = productRepository.findById(productId).get();
    if (product.getStockQuantity() < quantity) {
        return false;
    }
    // Add to cart
    cartItemRepository.save(new CartItem(cartId, productId, quantity));
    return true;
}

// Actual inventory reservation uses pessimistic lock in separate transaction
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void reserveInventory(Long orderId, Long productId, int quantity) {
    Product product = productRepository.findByIdWithLock(productId);
    product.setStockQuantity(product.getStockQuantity() - quantity);
}
```

### 5.3 Reporting with READ UNCOMMITTED

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- Or in PostgreSQL:
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- PostgreSQL treats this as READ COMMITTED
SELECT COUNT(*), SUM(amount) FROM orders WHERE created_at > NOW() - INTERVAL '1 hour';
COMMIT;
```

### 5.4 Write Skew Scenario

-- Write skew: Two doctors both check if they are on call, both see they're the only one, both go off call
```sql
-- Doctor A checks: SELECT count(*) FROM on_call WHERE doctor_id != 1 AND date = '2024-01-01';
-- Returns 0 (Doctor B is the only other on call)
-- Doctor B checks: SELECT count(*) FROM on_call WHERE doctor_id != 2 AND date = '2024-01-01';
-- Returns 0 (Doctor A is the only other on call)
-- Doctor A: DELETE FROM on_call WHERE doctor_id = 1 AND date = '2024-01-01';
-- Doctor B: DELETE FROM on_call WHERE doctor_id = 2 AND date = '2024-01-01';
-- Result: No doctors on call! (Write skew)
```

Prevention requires SERIALIZABLE isolation or explicit locking:
```sql
-- SELECT FOR UPDATE on the relevant rows
SELECT * FROM on_call WHERE date = '2024-01-01' FOR UPDATE;
```

## 6. Performance

### 6.1 Isolation Level Performance Impact

| Level | Read Overhead | Write Overhead | Contention |
|-------|--------------|---------------|------------|
| READ UNCOMMITTED | Lowest | Lowest | Lowest |
| READ COMMITTED | Low | Low | Low |
| REPEATABLE READ | Medium | Medium | Medium |
| SERIALIZABLE | High | High | High (retries) |

### 6.2 Throughput vs Consistency Trade-off

- **READ UNCOMMITTED**: Highest throughput, risky data quality
- **READ COMMITTED**: Good balance; default for most databases
- **REPEATABLE READ**: Lower throughput, needed for consistent snapshots
- **SERIALIZABLE**: Lowest throughput, maximum consistency

### 6.3 Connection Pool Impact

Higher isolation levels hold transactions longer (waiting for locks), which ties up connection pool resources. Monitor:
- Connection wait time
- Active vs idle connections
- Transaction duration distribution

## 7. Security

### 7.1 Isolation Level and Data Exposure

Lower isolation levels can expose sensitive data:
- **Dirty reads**: User could see another user's uncommitted (potentially incorrect) data
- **Non-repeatable reads**: Reporting inconsistency could lead to incorrect security decisions

Set minimum isolation level based on data sensitivity:
```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public SensitiveData readSensitiveData() {
    // Stronger isolation for sensitive operations
}
```

### 7.2 Anomaly-Based Attacks

Attackers could exploit isolation anomalies:
- **Read skew**: Exploit inconsistent state for fraud
- **Write skew**: Bypass business rules (e.g., both withdraw from last shared funds)

Use SERIALIZABLE for security-critical operations.

## 8. Common Mistakes

1. **Using default isolation without understanding what it means** — database defaults vary
2. **Assuming REPEATABLE READ prevents all anomalies** — doesn't prevent phantoms or write skew
3. **Not handling serialization failures** — SERIALIZABLE requires retry logic
4. **Confusing database isolation with application-level locking** — isolation is per-transaction
5. **Using SERIALIZABLE everywhere** — massive performance impact; use only where needed
6. **Not testing for concurrency issues** — isolation bugs only appear under load
7. **Assuming READ UNCOMMITTED provides no dirty reads in PostgreSQL** — PostgreSQL treats RU as RC
8. **Forgetting that READ COMMITTED allows non-repeatable reads** — same query, different results in same tx
9. **Not understanding MVCC interaction with isolation** — MVCC provides snapshot isolation, not true serializable
10. **Setting isolation at database level but overriding with different session/transaction settings**

## 9. Senior Engineer Perspective

### Isolation Level Decision Matrix

| Application Type | Recommended Isolation | Rationale |
|-----------------|----------------------|-----------|
| Financial transactions | SERIALIZABLE or READ COMMITTED + FOR UPDATE | Must prevent all anomalies |
| E-commerce | READ COMMITTED (carts) + lower for browsing | Balance consistency vs performance |
| Social media | READ COMMITTED | Inconsistencies acceptable |
| Reporting/analytics | READ UNCOMMITTED / SNAPSHOT | Avoid blocking; approximate OK |
| Inventory management | REPEATABLE READ + optimistic lock | Prevent overselling without serialization overhead |
| Audit trailing | SERIALIZABLE | Must have consistent view |

### Isolation Across Microservices

Each service has its own database with its own isolation. Cross-service consistency requires:
- Saga patterns with compensating transactions
- Distributed tracing to detect anomalies
- Event-driven eventual consistency

### PostgreSQL Isolation Behavior

PostgreSQL's isolation behavior:
- READ UNCOMMITTED -> treated as READ COMMITTED
- READ COMMITTED -> statement-level snapshot
- REPEATABLE READ -> transaction-level snapshot (no phantoms in practice due to MVCC)
- SERIALIZABLE -> SSI with predicate locks; may abort with serialization failure

## 10. Interview Questions (20)

### Easy (10)

1. What is a transaction isolation level?
2. What is a dirty read?
3. What is a non-repeatable read?
4. What is a phantom read?
5. List the four standard isolation levels from lowest to highest.
6. Which isolation level prevents dirty reads?
7. Which isolation level prevents all anomalies?
8. What is the default isolation level in PostgreSQL?
9. What is the default isolation level in MySQL/InnoDB?
10. How do you set isolation level in Spring @Transactional?

### Medium (10)

11. Explain the difference between READ COMMITTED and REPEATABLE READ.
12. What is snapshot isolation and how does it differ from SERIALIZABLE?
13. What is a write skew anomaly? How does SERIALIZABLE prevent it?
14. How does MVCC enable READ COMMITTED without read locks?
15. What happens when two concurrent transactions use SERIALIZABLE and both modify the same data?
16. Explain the "first committer wins" rule in snapshot isolation.
17. How do you handle serialization failures in a Spring Boot application?
18. What is the difference between pessimistic and optimistic concurrency control?
19. How does REPEATABLE READ prevent non-repeatable reads?
20. What is the performance impact of using SERIALIZABLE vs READ COMMITTED?

## 11. Advanced Interview Questions (20)

### Hard (10)

1. Explain how PostgreSQL's Serializable Snapshot Isolation (SSI) detects serialization anomalies.
2. What are predicate locks and how do they prevent phantoms in SERIALIZABLE isolation?
3. How does MySQL/InnoDB REPEATABLE READ prevent phantoms using gap locks?
4. Explain the difference between "read skew" and "write skew" anomalies.
5. How does Oracle's snapshot isolation differ from PostgreSQL's REPEATABLE READ?
6. What is the "lost update" problem and why isn't it prevented by REPEATABLE READ?
7. How does the ANSI SQL standard definition of isolation levels differ from practical implementations?
8. Explain the concept of "anomaly-strong" vs "anomaly-weak" isolation and the Adya classification.
9. How does SQL Server's SNAPSHOT isolation level compare to PostgreSQL's REPEATABLE READ?
10. What is the "SI anomaly" that can occur with snapshot isolation but not with serializable?

### System Design (11-20)

11. Design a system where different operations use different isolation levels.
12. How would you design a multi-region database system supporting serializable isolation?
13. Design a financial system that uses READ COMMITTED but prevents all anomalies programmatically.
14. How would you implement a monitoring system that tracks isolation-level-related anomalies?
15. Design a system that dynamically adjusts isolation level based on query type and system load.
16. How would you design a testing framework for detecting isolation-related bugs?
17. Design a multi-tenant database where each tenant can choose their isolation level.
18. How would you design a system where reports run at READ UNCOMMITTED without impacting OLTP workloads?
19. Design a system that provides serializable isolation without the performance cost using application-level locking.
20. How would you implement a distributed isolation level across shards?

## 12. Expert-Level Interview Questions (10)

1. Design a concurrency control system that automatically chooses between optimistic and pessimistic strategies based on contention probability prediction.
2. How would you implement a true serializable isolation level on top of a database that only supports snapshot isolation?
3. Design a system that detects phantom reads in production and alerts when they occur.
4. How would you implement a distributed serializable snapshot isolation protocol across a multi-region database?
5. Design a transaction scheduler that uses machine learning to predict and prevent serialization anomalies before they happen.
6. How would you implement an isolation level that is stronger than REPEATABLE READ but weaker than SERIALIZABLE (e.g., "repeatable read with no phantoms but allows write skew")?
7. Design a system that can transparently upgrade isolation level for certain operations based on learned query patterns.
8. How would you implement a correctness checker that verifies isolation guarantees are being met in production?
9. Design a cost model that quantifies the dollar value of isolation anomalies (how much should we spend to prevent them?).
10. How would you implement a linearizable isolation level on top of a Non-Volatile Memory (NVM) storage engine?

## 13. Debugging & Troubleshooting

### Check Current Isolation Level

```sql
-- PostgreSQL
SHOW transaction_isolation;

-- MySQL
SELECT @@transaction_isolation;

-- SQL Server
SELECT CASE transaction_isolation_level
    WHEN 0 THEN 'Unspecified'
    WHEN 1 THEN 'Read Uncommitted'
    WHEN 2 THEN 'Read Committed'
    WHEN 3 THEN 'Repeatable Read'
    WHEN 4 THEN 'Serializable'
    WHEN 5 THEN 'Snapshot'
END AS isolation_level
FROM sys.dm_exec_sessions
WHERE session_id = @@SPID;
```

### Detect Isolation Anomalies

```sql
-- Check for serialization failures
SELECT datname, xact_commit, xact_rollback,
       ROUND(100.0 * xact_rollback / NULLIF(xact_commit + xact_rollback, 0), 2) AS rollback_pct
FROM pg_stat_database;

-- PostgreSQL: count serialization failures
SELECT count(*) AS serialization_failures
FROM pg_stat_database
WHERE xact_rollback > 0
  AND datname = current_database();
```

### Spring Boot Isolation Logging

```yaml
logging:
  level:
    org.springframework.transaction: TRACE
    org.springframework.orm.jpa.JpaTransactionManager: DEBUG
```

## 14. Comparison Section

| Aspect | READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE |
|--------|-----------------|----------------|-----------------|--------------|
| Dirty Read | X | OK | OK | OK |
| Non-Repeatable Read | X | X | OK | OK |
| Phantom Read | X | X | X | OK |
| Write Skew | X | X | X | OK |
| Lost Update | X | X | X | OK |
| Typical Use | Reporting, dashboard | OLTP (default) | Inventory, billing | Financial, critical |
| Lock Usage | None | Short read locks | Read locks held | Full locking/predicate locks |
| MVCC | Statement snapshot | Statement snapshot | Tx snapshot | SSI / lock-based |
| PostgreSQL Impl | Like RC | Statement snapshot | Tx snapshot | SSI (true serializable) |
| MySQL Impl | Dirty reads possible | Statement snapshot | Tx snapshot + gap locks | Lock-based + gap locks |
| Performance | Best | Good | Fair | Worst |

## 15. Revision Notes

- Four levels: READ UNCOMMITTED < READ COMMITTED < REPEATABLE READ < SERIALIZABLE
- Dirty read: seeing uncommitted data (prev at RC+)
- Non-repeatable read: same row, different values in same tx (prev at RR+)
- Phantom read: same query, different rows in same tx (prev at SERIALIZABLE)
- Write skew: inconsistent writes based on stale snapshot (only SERIALIZABLE prevents)
- PostgreSQL MVCC provides snapshot isolation at REPEATABLE READ
- PostgreSQL SERIALIZABLE uses SSI (Serializable Snapshot Isolation) — true serializable
- Higher isolation = more consistency, less concurrency
- Always handle serialization failures with retry logic
- Default isolation varies by database (PG=RC, MySQL=RR, Oracle=RC, SQL Server=RC)

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                  ISOLATION LEVELS CHEAT SHEET                     |
+-------------------------------------------------------------------+
|                                                                   |
|  ANOMALY PROTECTION MATRIX:                                       |
|                                                                   |
|  +------------------+------+------+------+-----------+          |
|  |                  | RU   | RC   | RR   | SERIAL    |          |
|  +------------------+------+------+------+-----------+          |
|  | Dirty Read       | FAIL | PASS | PASS | PASS      |          |
|  | Non-Repeat Read  | FAIL | FAIL | PASS | PASS      |          |
|  | Phantom Read     | FAIL | FAIL | FAIL | PASS      |          |
|  | Lost Update      | FAIL | FAIL | FAIL | PASS      |          |
|  | Write Skew       | FAIL | FAIL | FAIL | PASS      |          |
|  +------------------+------+------+------+-----------+          |
|                                                                   |
|  RU = READ UNCOMMITTED                                            |
|  RC = READ COMMITTED                                              |
|  RR = REPEATABLE READ                                             |
|  SERIAL = SERIALIZABLE                                            |
|                                                                   |
+-------------------------------------------------------------------+
|  DATABASE DEFAULTS AND BEHAVIOR:                                  |
|                                                                   |
|  PostgreSQL:       RC (default), RR=Snapshot, SERIAL=SSI          |
|  MySQL/InnoDB:     RR (default), uses MVCC + gap locks            |
|  Oracle:           RC (default), RR unavailable, SERIAL=Snapshot  |
|  SQL Server:       RC (default), offers SNAPSHOT isolation        |
|                                                                   |
+-------------------------------------------------------------------+
|  SPRING BOOT CONFIGURATION:                                       |
|                                                                   |
|  @Transactional(isolation = Isolation.READ_COMMITTED)             |
|  @Transactional(isolation = Isolation.REPEATABLE_READ)            |
|  @Transactional(isolation = Isolation.SERIALIZABLE)               |
|                                                                   |
|  Global default:                                                   |
|  spring.jpa.properties.hibernate.connection.isolation: 2          |
|  # 1=RU, 2=RC, 4=RR, 8=SERIAL                                    |
|                                                                   |
+-------------------------------------------------------------------+
|  ANOMALY EXAMPLES:                                                |
|                                                                   |
|  Dirty Read:                                                      |
|    Tx1: UPDATE account SET balance=200                             |
|    Tx2: SELECT balance -> sees 200 (uncommitted!)                  |
|    Tx1: ROLLBACK (balance back to 100)                            |
|    Tx2: used wrong value!                                         |
|                                                                   |
|  Non-Repeatable Read:                                             |
|    Tx1: SELECT balance -> 100                                     |
|    Tx2: UPDATE balance=200; COMMIT                                |
|    Tx1: SELECT balance -> 200 (different!)                        |
|                                                                   |
|  Phantom Read:                                                    |
|    Tx1: SELECT count(*) WHERE status='PENDING' -> 5               |
|    Tx2: INSERT order (status='PENDING'); COMMIT                  |
|    Tx1: SELECT count(*) WHERE status='PENDING' -> 6 (phantom!)   |
|                                                                   |
+-------------------------------------------------------------------+
|  SERIALIZABLE RETRY PATTERN:                                      |
|                                                                   |
|  @Retryable(value = CannotSerializeTransactionException.class,    |
|             maxAttempts = 3, backoff = @Backoff(delay = 100))     |
|  @Transactional(isolation = Isolation.SERIALIZABLE)              |
|  public void criticalOperation() { ... }                          |
|                                                                   |
+-------------------------------------------------------------------+
|  LOST UPDATE PREVENTION (without SERIALIZABLE):                   |
|                                                                   |
|  1. SELECT ... FOR UPDATE (pessimistic lock)                     |
|  2. @Version column (optimistic lock)                             |
|  3. Atomic SQL: UPDATE table SET col = col + 1                   |
|  4. LAST_UPDATED timestamp check                                 |
|                                                                   |
+-------------------------------------------------------------------+
