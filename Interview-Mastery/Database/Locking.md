# Locking

## 1. Executive Summary

Database locking is the mechanism that prevents concurrent transactions from interfering with each other while maintaining data consistency. Locks control access to database resources (rows, pages, tables) and implement isolation guarantees. Understanding locking is critical for building concurrent applications — too little locking causes data corruption, too much causes contention and deadlocks. In Spring Boot/JPA applications, the ORM and transaction manager interact with database locks in ways that can surprise developers who don't understand the underlying mechanisms.

## 2. Core Theory

### 2.1 Lock Types by Granularity

| Granularity | Resource Locked | Concurrency | Overhead |
|-------------|----------------|-------------|----------|
| Row-level | Single row | Highest | High |
| Page-level | Disk page (multiple rows) | Medium | Medium |
| Table-level | Entire table | Lowest | Low |
| Database-level | Entire database | None | Lowest |

### 2.2 Lock Modes

| Mode | Symbol | Description | Compatibility |
|------|--------|-------------|---------------|
| Shared | S | Read lock; allows concurrent reads | Compatible with S |
| Exclusive | X | Write lock; blocks all other locks | Incompatible with all |
| Update | U | Intention to update; prevents conversion deadlock | S compatible, conflicts with X |
| Intention Shared | IS | Intention to read at finer granularity | Compatible with IS, IX, S, X (at higher level) |
| Intention Exclusive | IX | Intention to write at finer granularity | Compatible with IS, IX; conflicts with S, X |

### 2.2 Lock Compatibility Matrix

```
    S   X   IS  IX  SIX
S   Y   N   Y   N   N
X   N   N   N   N   N
IS  Y   N   Y   Y   Y
IX  N   N   Y   Y   N
SIX N   N   Y   N   N

Y = Compatible, N = Conflict
```

### 2.3 Two-Phase Locking (2PL)

For transaction isolation, databases use 2PL:
- **Growing phase**: Locks are acquired but not released
- **Shrinking phase**: Locks are released but not acquired
- Strict 2PL: All exclusive locks held until commit

### 2.4 Deadlock

A deadlock occurs when two transactions each hold a lock the other needs:

```
Transaction 1: Locks row A -> wants row B = blocked
Transaction 2: Locks row B -> wants row A = blocked
```

Databases detect deadlocks using wait-for graphs and resolve them by aborting one transaction (the "victim").

## 3. Under-the-Hood Deep Dive

### 3.1 Lock Manager Internals

The lock manager maintains:
- **Lock table**: Hash table mapping resource IDs to lock structures
- **Lock structure**: Holders (granted), Waiters (pending), Mode
- **Wait-for graph**: Tracks which transactions wait for which

### 3.2 Row-Level Locking in PostgreSQL

PostgreSQL implements row-level locking using a combination of:
- **Tuple header flags**: Indicate if row is locked
- **Multi-version concurrency control (MVCC)**: Readers don't block writers; writers don't block readers
- **Row locks**: Implemented via `ctid` and transaction ID arrays

```sql
-- PostgreSQL row locks explicitly:
SELECT * FROM orders WHERE id = 100 FOR UPDATE;      -- Exclusive row lock
SELECT * FROM orders WHERE id = 100 FOR NO KEY UPDATE; -- Weaker exclusive
SELECT * FROM orders WHERE id = 100 FOR SHARE;         -- Shared row lock
SELECT * FROM orders WHERE id = 100 FOR KEY SHARE;     -- Weakest shared
```

### 3.3 MVCC vs Locking

MVCC allows readers to see a consistent snapshot without acquiring shared locks:
- **Readers**: See the latest committed version as of their transaction start
- **Writers**: Create new row versions; old versions remain for concurrent readers
- **Cleanup**: Dead row versions are cleaned by VACUUM (PostgreSQL) or background threads

This means: SELECT (without FOR UPDATE) never blocks and never is blocked by writers in PostgreSQL/Oracle.

### 3.4 Lock Escalation

Databases may escalate row locks to table locks when a transaction locks many rows:
- SQL Server: Automatically at ~5000 locks per session
- PostgreSQL: No lock escalation (uses memory-efficient lock structures)
- MySQL/InnoDB: No lock escalation (uses bitmap for page-level lock tracking)

### 3.5 Lock Timeout Configuration

```sql
-- PostgreSQL
SET lock_timeout = '5s';  -- Statement-level timeout
SET deadlock_timeout = '1s'; -- How long before deadlock check

-- MySQL
SET innodb_lock_wait_timeout = 50; -- Seconds
```

## 4. Production Code Examples

### 4.1 Pessimistic Locking with JPA

```java
@Repository
public interface AccountRepository extends JpaRepository<Account, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdWithLock(@Param("id") Long id);

    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdWithSharedLock(@Param("id") Long id);
}
```

### 4.2 Optimistic Locking with @Version

```java
@Entity
@Table(name = "accounts")
public class Account {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String ownerName;

    @Column(nullable = false)
    private BigDecimal balance;

    @Version
    private Long version;
}
```

```java
@Service
@Transactional
public class TransferService {
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // Optimistic locking: version check happens on update/save
        Account from = accountRepository.findById(fromId)
            .orElseThrow(() -> new EntityNotFoundException("From account not found"));
        Account to = accountRepository.findById(toId)
            .orElseThrow(() -> new EntityNotFoundException("To account not found"));

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));

        accountRepository.save(from);  // version++ if version matches
        accountRepository.save(to);    // version++ if version matches
    }
}
```

### 4.3 Pessimistic Locking in Service Layer

```java
@Service
@Transactional
public class InventoryService {
    public OrderItem reserveInventory(Long productId, int quantity) {
        // Pessimistic lock prevents overselling
        Product product = productRepository.findByIdWithLock(productId)
            .orElseThrow(() -> new EntityNotFoundException("Product not found"));

        if (product.getStockQuantity() < quantity) {
            throw new InsufficientInventoryException(
                "Only " + product.getStockQuantity() + " available");
        }

        product.setStockQuantity(product.getStockQuantity() - quantity);
        productRepository.save(product);

        return new OrderItem(productId, quantity);
    }
}
```

### 4.4 Lock Timeout Configuration in JPA

```java
// Set lock timeout at query level
@QueryHints({
    @QueryHint(name = "jakarta.persistence.lock.timeout", value = "5000")
})
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdWithLock(@Param("id") Long id);

// Global configuration
spring.jpa.properties.jakarta.persistence.lock.timeout: 5000
spring.jpa.properties.hibernate.connection.isolation: 2  # READ_COMMITTED
```

### 4.5 Handling OptimisticLockException

```java
@Service
@Transactional
public class RetryTransferService {
    private static final int MAX_RETRIES = 3;

    public TransferResult transferWithRetry(Long fromId, Long toId, BigDecimal amount) {
        for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try {
                transfer(fromId, toId, amount);
                return TransferResult.success();
            } catch (OptimisticLockException e) {
                if (attempt == MAX_RETRIES) {
                    return TransferResult.failure("Concurrent update conflict, please retry");
                }
                // Re-read fresh state and retry
                entityManager.clear();
            }
        }
        return TransferResult.failure("Max retries exceeded");
    }
}
```

## 5. Real-World Scenarios

### 5.1 Inventory Reservation (Pessimistic Lock)

```sql
-- BEGIN TX
SELECT * FROM inventory WHERE product_id = 100 FOR UPDATE;
-- Check stock, decrement, decision to fulfill
UPDATE inventory SET reserved = reserved + 1 WHERE product_id = 100;
-- COMMIT / ROLLBACK
```

Without `FOR UPDATE`, two concurrent requests could both see 1 item in stock and both fulfill, causing overselling.

### 5.2 Non-Blocking Read with Optimistic Lock

```java
// Read current state (no lock)
Article article = articleRepository.findById(id).get();
article.setTitle(newTitle);
article.setContent(newContent);
// At save time, DB checks version hasn't changed
articleRepository.save(article);
// If another user updated between read and save -> OptimisticLockException
```

### 5.3 Avoiding Locks with READ UNCOMMITTED or READ ONLY

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- Or for reporting queries that shouldn't lock
SELECT * FROM orders WHERE status = 'COMPLETED';
-- These never lock and never are blocked
```

In PostgreSQL:
```sql
-- READ UNCOMMITTED behaves like READ COMMITTED (no dirty reads)
-- Equivalent to:
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT * FROM orders WHERE status = 'COMPLETED';
COMMIT;
```

### 5.4 Deadlock Detection and Resolution

```java
@ControllerAdvice
public class DeadlockHandler {
    @ExceptionHandler(DeadlockLoserDataAccessException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleDeadlock(DeadlockLoserDataAccessException e) {
        // Log and return retry suggestion
        return new ErrorResponse("CONCURRENT_MODIFICATION",
            "Request failed due to concurrent modification. Please retry.");
    }
}
```

## 6. Performance

### 6.1 Lock Contention Impact

- **Row locks**: Minimal contention for distinct rows; high contention for same row
- **Page/table locks**: High contention, should be avoided in OLTP
- **Lock overhead**: Memory for lock structures + CPU for lock management

### 6.2 Reducing Lock Contention

1. **Keep transactions short** — minimize lock hold time
2. **Access resources in consistent order** — prevent deadlocks (always update accounts in ID order)
3. **Use appropriate isolation level** — READ COMMITTED instead of SERIALIZABLE when possible
4. **Use optimistic locking** — for low-contention scenarios
5. **Batch operations** — reduce number of transactions
6. **Index for update** — FOR UPDATE uses index to lock only matching rows

### 6.3 Lock Monitoring

```sql
-- PostgreSQL: current locks
SELECT locktype, relation::regclass, mode, granted,
       pid, transactionid, virtualtransaction
FROM pg_locks
WHERE NOT granted;

-- MySQL: InnoDB lock status
SHOW ENGINE INNODB STATUS;
SELECT * FROM performance_schema.data_locks;

-- SQL Server
EXEC sp_who2;
SELECT * FROM sys.dm_tran_locks;
```

## 7. Security

### 7.1 Lock-Based Denial of Service

An attacker can intentionally acquire locks on critical rows to block legitimate users. Mitigations:
- Set lock_timeout to prevent indefinite blocking
- Monitor for sessions holding locks for extended periods
- Use row-level security with read-committed isolation
- Kill long-running idle-in-transaction sessions

## 8. Common Mistakes

1. **Not using @Version for optimistic locking** — silently overwriting concurrent updates
2. **Using pessimistic locking when optimistic would suffice** — unnecessary contention
3. **Forgetting lock_timeout** — transactions can block indefinitely
4. **Inconsistent lock order across transactions** — leading to deadlocks
5. **Holding locks across user interaction (long transactions)** — lock contention escalates
6. **SELECT without FOR UPDATE before UPDATE in concurrent writes** — lost updates
7. **Assuming SELECT always blocks** — MVCC means SELECT doesn't block in PostgreSQL/Oracle
8. **Not handling OptimisticLockException** — silent data loss instead of retry
9. **Lock escalation in MySQL/SQL Server** — row locks escalate to table locks unexpectedly
10. **Not considering application-level transactions** — Hibernate session-per-request pattern

## 9. Senior Engineer Perspective

### Locking Strategy Selection

| Scenario | Recommended Strategy | Why |
|----------|---------------------|-----|
| Low contention, simple reads/writes | Optimistic locking (@Version) | Lowest overhead, no blocking |
| High contention (single resource, many users) | Pessimistic locking | Prevents retry storms |
| Financial transactions | Pessimistic locking with ordered access | Must prevent overspend |
| Reporting / analytics | Read-only transaction (no locking) | Avoids locking OLTP rows |
| Queue processing (work table) | SELECT ... FOR UPDATE SKIP LOCKED | Non-blocking work distribution |
| High-volume counter | Atomic UPDATE without read-first | Reduces round trips |

### SKIP LOCKED (PostgreSQL 9.5+, MySQL 8.0+)

```sql
-- Skip already-locked rows (queue processing)
BEGIN;
SELECT * FROM job_queue
WHERE status = 'PENDING'
ORDER BY priority DESC
LIMIT 10
FOR UPDATE SKIP LOCKED;
-- Process jobs...
UPDATE job_queue SET status = 'DONE' WHERE id IN (...);
COMMIT;
```

## 10. Interview Questions (20)

### Easy (10)

1. What is a database lock?
2. What is the difference between shared and exclusive locks?
3. What is a deadlock?
4. How does MVCC avoid locking for read operations?
5. What is the difference between optimistic and pessimistic locking?
6. What does the @Version annotation do in JPA?
7. What is lock contention?
8. What happens when a deadlock is detected?
9. What is the difference between row-level and table-level locking?
10. What is lock escalation?

### Medium (10)

11. Explain the difference between `FOR UPDATE` and `FOR SHARE`.
12. How does JPA's `@Lock(LockModeType.PESSIMISTIC_WRITE)` work?
13. What is two-phase locking (2PL)?
14. How does `SELECT ... FOR UPDATE SKIP LOCKED` work? When would you use it?
15. What is a lock timeout and how do you configure it in PostgreSQL/MySQL?
16. Explain how optimistic locking detects concurrent modifications.
17. What is lock granularity and how does it affect performance?
18. How would you diagnose a deadlock in PostgreSQL?
19. What is the difference between `LockModeType.OPTIMISTIC` and `LockModeType.OPTIMISTIC_FORCE_INCREMENT`?
20. How does MVCC eliminate the need for read locks?

## 11. Advanced Interview Questions (20)

### Hard (10)

1. Explain how PostgreSQL implements row-level locking using the tuple header's `t_infomask` bits.
2. What is predicate locking and how does it prevent phantoms in SERIALIZABLE isolation?
3. Explain the difference between blocking locks, deadlocks, and livelocks.
4. How does the wait-die or wound-wait scheme prevent deadlocks?
5. Describe the internal structure of a lock manager (lock table, lock structures, wait queues).
6. How does lock escalation work in SQL Server vs PostgreSQL?
7. Explain the concept of "lock key range" in MySQL/InnoDB for preventing phantoms.
8. How does the database detect deadlocks in a distributed (sharded) environment?
9. What is the difference between a "lock" and a "latch" in database internals?
10. How does PostgreSQL implement "SELECT ... FOR UPDATE" under the hood (tuple locking + multixact)?

### System Design (11-20)

11. Design a distributed lock service using a relational database.
12. How would you design a job queue with `SKIP LOCKED` for concurrent workers?
13. Design an inventory reservation system that handles high contention on popular products.
14. How would you implement a pessimistic locking strategy across microservices?
15. Design a deadlock detection and resolution system for a distributed database.
16. How would you design a ticket booking system that prevents overselling?
17. Design a locking strategy for a financial ledger that must maintain audit integrity.
18. How would you implement a non-blocking read-optimized system with occasional writes?
19. Design a lock-free queue using PostgreSQL advisory locks.
20. How would you design a multi-tenant system with tenant-level locking isolation?

## 12. Expert-Level Interview Questions (10)

1. Design a database concurrency control system that uses optimistic CC for all operations but automatically falls back to pessimistic on conflict.
2. How would you implement serializable snapshot isolation (SSI) from scratch?
3. Design a distributed deadlock detection algorithm for a globally distributed database spanning 10+ regions.
4. How would you implement a lock manager that supports both row-level and predicate locks with zero overhead for the common case of no concurrency?
5. Design a system that automatically detects lock contention hotspots and suggests schema/query changes.
6. How would you implement MVCC-garbage-collection (VACUUM) that doesn't interfere with concurrent readers?
7. Design a hybrid concurrency control system that uses different strategies for different tables in the same transaction.
8. How would you implement optimistic locking in a sharded database without a global version counter?
9. Design a system that supports both snapshot isolation and serializable isolation concurrently, choosing per-transaction.
10. How would you implement a lock-free read of a row that is being concurrently updated by another transaction?

## 13. Debugging & Troubleshooting

### PostgreSQL Lock Diagnostics

```sql
-- Blocked queries
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query,
       now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.database = blocking_locks.database
    AND NOT blocked_locks.granted
JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- Long-running transactions (potential lock holders)
SELECT pid, age(now(), xact_start) AS transaction_duration,
       state, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
   OR (state = 'active' AND xact_start < now() - interval '5 minutes');
```

### MySQL Lock Diagnostics

```sql
-- Current locks
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;

-- InnoDB status (verbose)
SHOW ENGINE INNODB STATUS;
```

### Spring Boot Lock Diagnostics

```yaml
logging:
  level:
    org.springframework.orm.jpa: DEBUG
    org.springframework.transaction: TRACE
    com.zaxxer.hikari: DEBUG
```

## 14. Comparison Section

| Feature | PostgreSQL | MySQL/InnoDB | Oracle | SQL Server |
|---------|-----------|--------------|--------|------------|
| Row locking | FOR UPDATE/KEY SHARE | FOR UPDATE | FOR UPDATE | UPDLOCK/ROWLOCK |
| MVCC | Yes | Yes | Yes | Yes (snapshot isolation) |
| Lock escalation | None | None | None | Yes (row->table) |
| Deadlock detection | Immediate (wait-for graph) | Immediate (wait-for graph) | Immediate | Periodic check |
| SKIP LOCKED | Yes (9.5+) | Yes (8.0+) | No (use NOWAIT) | Yes (2019+) |
| Advisory locks | Yes (pg_advisory_lock) | GET_LOCK() | DBMS_LOCK | sp_getapplock |
| Default isolation | READ COMMITTED | REPEATABLE READ | READ COMMITTED | READ COMMITTED |
| Predicate locks | Yes (SERIALIZABLE) | Yes (gap locks) | Yes | Yes |

| Approach | Optimistic Locking | Pessimistic Locking |
|----------|-------------------|---------------------|
| Read | No lock (version read) | No lock (or shared lock) |
| Write | Version check at update | Exclusive lock at read |
| Conflict detection | At commit (may fail) | At read (blocks immediately) |
| Retry needed | Yes (OptimisticLockException) | No (waits and succeeds) |
| Best for | Low contention, read-heavy | High contention, write-heavy |
| Scalability | Better (no locking overhead) | Worse (lock contention) |
| Implementation | @Version annotation | @Lock(PESSIMISTIC_WRITE) |

## 15. Revision Notes

- Locks prevent concurrent access conflicts; MVCC reduces need for read locks
- Lock granularity: row < page < table (fine = more concurrency, more overhead)
- Two-phase locking: acquire all locks before releasing any
- Deadlock: cyclic wait; database detects and aborts one transaction
- Pessimistic locking: lock at read time; use for high contention
- Optimistic locking: check at write time; use for low contention
- SKIP LOCKED: skip locked rows in queue processing
- Always access resources in consistent order to prevent deadlocks
- Keep transactions short to minimize lock hold time
- Monitor pg_locks / performance_schema.data_locks for contention

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                     LOCKING CHEAT SHEET                           |
+-------------------------------------------------------------------+
|                                                                   |
|  LOCK MODES:                                                      |
|                                                                   |
|  S  (Shared)       -> Read lock, allows concurrent reads          |
|  X  (Exclusive)    -> Write lock, no concurrent access            |
|  U  (Update)       -> Intention to update (prevents conversion    |
|                        deadlock on SQL Server)                     |
|  IS (Int Shared)   -> Intention to read at fine granularity       |
|  IX (Int Exclusive)-> Intention to write at fine granularity      |
|                                                                   |
+-------------------------------------------------------------------+
|  LOCK COMPATIBILITY (Y=compatible, N=conflict):                   |
|                                                                   |
|           S    X    IS   IX   SIX                                 |
|     S     Y    N    Y    N    N                                   |
|     X     N    N    N    N    N                                   |
|     IS    Y    N    Y    Y    Y                                   |
|     IX    N    N    Y    Y    N                                   |
|     SIX   N    N    Y    N    N                                   |
|                                                                   |
+-------------------------------------------------------------------+
|  POSTGRESQL ROW LOCKS:                                            |
|                                                                   |
|  FOR UPDATE         -> Exclusive (prevents all writes + key       |
|                         updates on referenced FKs)                 |
|  FOR NO KEY UPDATE  -> Like UPDATE but doesn't block KEY SHARE    |
|  FOR SHARE          -> Shared (prevents writes, allows reads)     |
|  FOR KEY SHARE      -> Weak shared (only blocks FK key changes)   |
|                                                                   |
+-------------------------------------------------------------------+
|  JPA LOCK MODES:                                                  |
|                                                                   |
|  LockModeType.OPTIMISTIC           -> Check @Version at commit     |
|  LockModeType.OPTIMISTIC_FORCE_INCREMENT -> Force version inc     |
|  LockModeType.PESSIMISTIC_READ     -> Shared lock (FOR SHARE)     |
|  LockModeType.PESSIMISTIC_WRITE    -> Exclusive lock (FOR UPDATE) |
|  LockModeType.PESSIMISTIC_FORCE_INCREMENT -> Exclusive + ver inc |
|                                                                   |
+-------------------------------------------------------------------+
|  DEADLOCK PREVENTION:                                             |
|                                                                   |
|  1. Always access resources in the same order                     |
|     (e.g., always update account 1 before account 2)              |
|  2. Keep transactions short                                       |
|  3. Use appropriate isolation level (not SERIALIZABLE unless needed)|
|  4. Use lock_timeout to prevent indefinite waiting                |
|  5. Manage locks at database level, not application level         |
|  6. Use READ COMMITTED + optimistic locking when possible         |
|                                                                   |
+-------------------------------------------------------------------+
|  MONITORING COMMANDS:                                             |
|                                                                   |
|  PostgreSQL:   SELECT * FROM pg_locks WHERE NOT granted;          |
|  MySQL:        SHOW ENGINE INNODB STATUS;                        |
|  SQL Server:   SELECT * FROM sys.dm_tran_locks;                  |
|  Oracle:       SELECT * FROM v$lock WHERE block > 0;             |
|                                                                   |
+-------------------------------------------------------------------+
