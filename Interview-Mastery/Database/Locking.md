# Database Locking

---

## What is Database Locking?

**Database locking** is the mechanism that prevents concurrent transactions from interfering with each other while maintaining data consistency. Locks control access to database resources (rows, pages, tables) and implement isolation guarantees. Understanding locking is critical for building concurrent applications — too little locking causes data corruption, too much causes contention and deadlocks.

### Lock Granularity

- **Row-level** — A single row is locked. Offers the highest concurrency but the highest overhead due to managing many individual locks.
- **Page-level** — A disk page (containing multiple rows) is locked. Provides medium concurrency and overhead, balancing row and table locking.
- **Table-level** — The entire table is locked. Offers the lowest concurrency but the lowest overhead for lock management.
- **Database-level** — The entire database is locked. Provides no concurrency and is rarely used outside of administrative operations.

### Lock Modes

- **Shared (S)** — A read lock that allows other shared locks but blocks exclusive locks. Multiple transactions can hold shared locks on the same resource simultaneously.
- **Exclusive (X)** — A write lock that blocks all other locks, both shared and exclusive. Only one transaction can hold an exclusive lock on a resource.
- **Update (U)** — An intention to update that prevents conversion deadlocks. Compatible with shared locks but conflicts with exclusive locks.
- **Intention Shared (IS)** — An intention to read fine-grained resources at a higher level (e.g., table-level lock indicating intention to lock rows with shared locks).
- **Intention Exclusive (IX)** — An intention to write fine-grained resources at a higher level (e.g., table-level lock indicating intention to lock rows with exclusive locks).

### Two-Phase Locking (2PL)

For transaction isolation, databases use 2PL:
- **Growing phase** — Locks are acquired but not released during this phase.
- **Shrinking phase** — Locks are released but not acquired during this phase.
- **Strict 2PL** — All exclusive locks are held until the transaction commits or rolls back, ensuring serializability.

### Deadlocks

A **deadlock** occurs when two transactions each hold a lock the other needs:
```
Tx 1: Locks row A → wants row B = blocked
Tx 2: Locks row B → wants row A = blocked
```
Databases detect deadlocks using **wait-for graphs** and resolve them by aborting one transaction (the "victim"). The victim's transaction is rolled back and must be retried.

### MVCC vs Locking

MVCC allows readers to see a consistent snapshot without acquiring shared locks:
- **Readers** see the latest committed version as of their transaction start, without blocking.
- **Writers** create new row versions; old versions remain available for concurrent readers.
- **Cleanup** of old versions is handled by VACUUM (PostgreSQL) or background purge threads.
- This means `SELECT` (without `FOR UPDATE`) never blocks and is never blocked by writers in PostgreSQL and Oracle.

---

## Core Concepts

### PostgreSQL Row-Level Locking

```sql
SELECT * FROM orders WHERE id = 100 FOR UPDATE;        -- Exclusive row lock
SELECT * FROM orders WHERE id = 100 FOR NO KEY UPDATE; -- Weaker exclusive (doesn't block KEY SHARE)
SELECT * FROM orders WHERE id = 100 FOR SHARE;         -- Shared row lock
SELECT * FROM orders WHERE id = 100 FOR KEY SHARE;     -- Weakest shared (only blocks FK key changes)
```

Each lock mode offers a different balance of protection and concurrency. `FOR NO KEY UPDATE` is useful when you need to lock a row for update but want to allow foreign key references to the locked column.

### Pessimistic Locking with JPA

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

`PESSIMISTIC_WRITE` acquires an exclusive lock, while `PESSIMISTIC_READ` acquires a shared lock. Choose the mode based on whether you plan to modify the entity.

### Optimistic Locking with @Version

```java
@Entity
@Table(name = "accounts")
public class Account {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

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
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();
        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
        // version check happens on save; if version changed → OptimisticLockException
        accountRepository.save(from);
        accountRepository.save(to);
    }
}
```

The `@Version` field is incremented on every update. If another transaction modified the same row concurrently, the version mismatch prevents the update and throws `OptimisticLockException`.

### Handling OptimisticLockException

```java
@Service
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
                entityManager.clear(); // Re-read fresh state
            }
        }
        return TransferResult.failure("Max retries exceeded");
    }
}
```

Retry with exponential backoff is essential for optimistic locking under moderate contention. Clearing the persistence context ensures fresh data is read on each retry.

### SKIP LOCKED for Queue Processing

```sql
-- Skip already-locked rows for concurrent workers
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

Available in PostgreSQL 9.5+, MySQL 8.0+, SQL Server 2019+. SKIP LOCKED enables horizontal scaling of workers by avoiding lock contention on shared queue tables.

### Lock Timeout Configuration

```sql
-- PostgreSQL
SET lock_timeout = '5s';       -- Statement-level timeout
SET deadlock_timeout = '1s';   -- How long before deadlock check

-- MySQL
SET innodb_lock_wait_timeout = 50; -- Seconds
```

```java
// JPA lock timeout
@QueryHints({
    @QueryHint(name = "jakarta.persistence.lock.timeout", value = "5000")
})
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdWithLock(@Param("id") Long id);
```

Always set `lock_timeout` to fail fast instead of blocking indefinitely. A transaction waiting on a lock blocks a connection pool thread.

### Lock Monitoring

```sql
-- PostgreSQL: current locks
SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks WHERE NOT granted;

-- MySQL: InnoDB lock status
SHOW ENGINE INNODB STATUS;
SELECT * FROM performance_schema.data_locks;

-- SQL Server
SELECT * FROM sys.dm_tran_locks;
```

Monitoring active locks helps identify blocking chains and contention points. The `NOT granted` filter shows only waiting transactions.

---

## Common Mistakes

- **Not using @Version for optimistic locking** — Without versioning, concurrent updates silently overwrite each other, causing lost updates.
  - **Why it looks correct:** The application reads a value, modifies it in memory, and saves it — the classic read-check-write pattern that looks safe but misses the concurrency window.
- **Using pessimistic locking when optimistic would suffice** — Pessimistic locking always acquires locks, adding overhead even without conflicts. Optimistic locking is cheaper when contention is low.
  - **Why it looks correct:** Pessimistic locking guarantees safety upfront and seems like "stronger" protection; the performance cost of always acquiring locks is invisible when there are no conflicts to compare against.
- **Forgetting lock_timeout** — Transactions waiting on a lock can block indefinitely, consuming connection pool threads and causing cascading failures.
  - **Why it looks correct:** The transaction eventually completes during testing; the indefinite block only becomes visible when another transaction holds the lock longer than expected under production load.
- **Inconsistent lock order across transactions** — Transaction A locks table X then Y, while Transaction B locks Y then X. This creates deadlock conditions. Always enforce a consistent lock order.
  - **Why it looks correct:** Each transaction's individual lock order seems natural given its specific operation; the circular wait only emerges when both transactions run concurrently.
- **Holding locks across user interaction (long transactions)** — Lock contention escalates when transactions span slow operations like user input or HTTP calls. Keep transactions short.
  - **Why it looks correct:** Wrapping the entire request in a transaction is the simplest approach, and the lock escalation is invisible during development where there is no concurrent load.
- **SELECT without FOR UPDATE before UPDATE in concurrent writes** — Two transactions read the same value, both decide to update, and the second write overwrites the first. Always lock rows you plan to update.
  - **Why it looks correct:** The read-then-write pattern is standard in application code and works perfectly in single-threaded or low-concurrency environments.
- **Assuming SELECT always blocks** — In MVCC databases like PostgreSQL and Oracle, plain SELECT never blocks and is never blocked by writers.
  - **Why it looks correct:** Other databases (MySQL with MyISAM, early SQL Server) use locking reads, so the assumption carries over from past experience with different database systems.
- **Not handling OptimisticLockException** — Without retry logic, the user sees a random "save failed" error instead of the operation being retried transparently.
  - **Why it looks correct:** The exception seems like a rare edge case; developers don't anticipate how frequently concurrent updates trigger it under normal load.
- **Lock escalation in MySQL/SQL Server** — When a query scans too many rows without an efficient index, row locks escalate to table locks, severely reducing concurrency.
  - **Why it looks correct:** The query uses row-level locking and seems fine; the escalation to table-level locking happens silently inside the database engine without any warning.
- **Not considering application-level transactions** — The Hibernate session-per-request pattern means a single HTTP request can hold database resources for its entire duration.
  - **Why it looks correct:** The Open Session in View pattern is Spring Boot's default and makes development convenient; the resource retention is invisible until connection pool exhaustion under load.

---

## Real-World Scenarios

### Ticket Booking with Pessimistic Locking

A concert ticket system must prevent overselling. Two users see the same available seat. `SELECT ... FOR UPDATE` locks the seat row until the transaction completes:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT s FROM Seat s WHERE s.id = :id")
Optional<Seat> findByIdWithLock(@Param("id") Long id);

@Transactional
public Booking bookSeat(Long seatId, Long userId) {
    Seat seat = seatRepository.findByIdWithLock(seatId)
        .orElseThrow(() -> new SeatNotFoundException());
    if (!seat.isAvailable()) throw new SeatBookedException();
    seat.setAvailable(false);
    return bookingRepository.save(new Booking(seat, userId));
}
```

### Job Queue with SKIP LOCKED

A job processing system has 10 workers polling the same `job_queue`. Without SKIP LOCKED, all workers block on the same rows. With SKIP LOCKED, each worker picks up only unlocked rows:

```sql
BEGIN;
SELECT * FROM job_queue
WHERE status = 'PENDING'
ORDER BY priority DESC
LIMIT 5
FOR UPDATE SKIP LOCKED;
UPDATE job_queue SET status = 'DONE' WHERE id IN (...);
COMMIT;
```

### Social Media Likes with Optimistic Locking

A social media post receives 1000 concurrent likes. Each like reads the count, increments, and writes back. Optimistic locking ensures no updates are silently lost:

```java
@Entity
public class Post {
    @Id private Long id;
    private int likeCount;
    @Version private Long version;
}

@Service
public class LikeService {
    @Transactional
    public void likePost(Long postId) {
        Post post = postRepo.findById(postId).orElseThrow();
        post.setLikeCount(post.getLikeCount() + 1);
    }
}
```

## Scenario-Based Questions

- **Q:** You are building a concert ticket booking system with 10,000 concurrent users. Seats are limited. How do you prevent overselling while maintaining throughput?
  - **A:** Use pessimistic locking with `FOR UPDATE SKIP LOCKED` on the seat reservation. This locks only the seats being booked and skips already-locked rows. Keep the reservation window short (5 minutes with a background job to release expired holds). For the waitlist, use optimistic locking since contention is lower.
  - **Interview follow-up:** With `SKIP LOCKED`, a user never gets an error — they just see fewer available seats. How do you differentiate between "seat is booked" and "seat is locked by someone else's pending reservation" in the UI?
- **Q:** Your app uses `@Version` for optimistic locking. During a flash sale with 5000 concurrent users for 100 items, almost every request fails with OptimisticLockException and retries. Retries also fail. What's happening?
  - **A:** Retry storm — all 5000 users read version 1, but only the first commit succeeds. The other 4999 retry reading version 2, but only one succeeds, and so on. Fix: switch to pessimistic locking for reservation, or use atomic `UPDATE items SET stock = stock - 1 WHERE id = ? AND stock > 0`.
  - **Interview follow-up:** You switch to atomic `UPDATE ... WHERE stock > 0` but need to show the user which specific item they reserved. The atomic update doesn't return which row was updated. How do you handle this?
- **Q:** Two transactions always lock tables A then B. A third transaction locks B then A. The app experiences periodic deadlocks. How do you prevent this?
  - **A:** Enforce global lock ordering — always acquire locks in the same order (by resource ID numerically, or table name alphabetically). For transfers, lock the smaller account ID first. Use `ORDER BY id` when selecting rows to lock. Set `lock_timeout` to fail fast instead of blocking.
  - **Interview follow-up:** You enforce lock ordering by account ID. The application has 8 microservices, each with its own database, and each maintains its own lock ordering convention. A cross-service saga now creates a distributed deadlock. How do you handle this?
- **Q:** A reporting query running for 3 minutes blocks all order updates. The report uses default settings. How do you fix this without a replica?
  - **A:** In PostgreSQL, `SELECT` without `FOR UPDATE` doesn't block writes. If it blocks, the report may use `FOR SHARE` or SERIALIZABLE. In MySQL, ensure the WHERE clause uses an index to avoid row-to-table lock escalation. Set `SET TRANSACTION READ ONLY` to avoid acquiring locks.
- **Q:** A `SELECT ... FOR UPDATE` on a parent table prevents INSERTs on a child table with a foreign key. Why?
  - **A:** The FK constraint acquires a shared lock (`FOR KEY SHARE`) on the parent row. Locking the parent with `FOR UPDATE` blocks child INSERTs. Use `FOR NO KEY UPDATE` (PostgreSQL) which doesn't block KEY SHARE locks.
- **Q:** Your PostgreSQL app uses `SELECT ... FOR UPDATE` for inventory. Locked rows are held for only 50ms, but CPU spikes and throughput drops under load. What's going wrong?
  - **A:** Even short lock holds cause contention when thousands compete for the same rows. The CPU spike is from context switching and deadlock detection. Use `SKIP LOCKED` so transactions skip locked rows instead of waiting. Queue requests in application memory and batch process. Use atomic `UPDATE inventory SET reserved = reserved + 1 WHERE id = ? AND reserved + 1 <= stock`.
- **Q:** A MySQL table with 10M rows frequently escalates row locks to table locks under heavy load. The WHERE clause uses `WHERE status = 'PENDING'`. What's the root cause?
  - **A:** MySQL escalates to table locks when a query scans too many rows without an efficient index. If `status` has low selectivity, MySQL may choose a full table scan, locking every row. Add an index on `status` and verify with `EXPLAIN` that the query uses an index scan.
- **Q:** An OptimisticLockException is logged but the application doesn't retry. The user sees "save failed". How do you handle this properly?
  - **A:** Implement retry with exponential backoff at the service layer. Catch `OptimisticLockException`, clear the persistence context (`entityManager.clear()`), re-read fresh data, and retry. After max retries, throw a meaningful business exception. Use `@Retryable` from Spring Retry.
- **Q:** A database deadlock is detected and one transaction is chosen as the victim. The victim's transaction rolls back. What happens to the connection?
  - **A:** The connection returns to the pool in a clean state (rollback resets it). Catch the deadlock exception and retry the entire operation. Never retry on the same connection without a fresh transaction. Configure `deadlock_timeout` for faster detection.
- **Q:** A long transaction holds a lock for 30 seconds while the app processes data in memory. Other transactions block. Connection pool threads are consumed by waiters. How do you diagnose and fix this?
  - **A:** Check `pg_locks` for blocked processes and `pg_stat_activity` for the blocking query. Fix: never hold locks across slow application processing. Read in one short transaction, process in memory, write in another. Set `lock_timeout` to fail fast.

## Interview Questions

- **What is the difference between shared and exclusive locks?**
  - **A:** Shared (read) locks allow other shared locks but block exclusive locks. Exclusive (write) locks block all other locks — both shared and exclusive.
- **What is a deadlock and how does the database resolve it?**
  - **A:** A deadlock occurs when two transactions each hold a lock the other needs (circular wait). The database uses wait-for graphs to detect deadlocks and aborts one transaction (the victim).
- **What is the difference between pessimistic and optimistic locking?**
  - **A:** Pessimistic acquires locks upfront (`SELECT FOR UPDATE`), preventing conflicts. Optimistic detects conflicts at commit time (`@Version`), retrying on failure. Pessimistic is better for high contention.
- **What does SKIP LOCKED do?**
  - **A:** Skips rows already locked by other transactions instead of waiting. Used for job queues and work distribution. Available in PostgreSQL 9.5+, MySQL 8.0+.
- **How does MVCC affect locking?**
  - **A:** MVCC allows readers to see a consistent snapshot without shared locks. `SELECT` without `FOR UPDATE` never blocks and is never blocked by writers in PostgreSQL and Oracle.
- **What is lock escalation?**
  - **A:** Converting many fine-grained locks (row-level) into a coarse-grained lock (table-level). Common in MySQL when a query scans many rows without an efficient index.
- **What is two-phase locking (2PL)?**
  - **A:** Transactions acquire locks in a growing phase and release in a shrinking phase. Strict 2PL holds all exclusive locks until commit or rollback, ensuring serializability.
- **What happens with @Lock(PESSIMISTIC_WRITE) in JPA?**
  - **A:** Hibernate issues `SELECT ... FOR UPDATE` on the entity. The row is locked until the transaction commits. Other transactions trying to read (with FOR UPDATE) or write to that row block.
- **How does @Version work?**
  - **A:** Hibernate increments the version column on every UPDATE. The UPDATE includes `WHERE version = :oldVersion`. If the version changed, no rows are updated and `OptimisticLockException` is thrown.
- **How do you monitor active locks in PostgreSQL?**
  - **A:** Query `pg_locks` for current locks and `pg_stat_activity` for blocked queries. Use `SELECT blocked.pid, blocking.pid FROM pg_locks ... WHERE NOT blocked.granted` to find blocking chains.

## Developer Recommendations

- **Use optimistic locking (@Version) for low-contention scenarios** — Optimistic locking has zero overhead when there is no conflict. Pessimistic locking always acquires locks, adding overhead even without conflict.
  - **Production story:** A social media app used pessimistic locking for all like operations. Under 10K likes/second on a viral post, the database CPU hit 100% from lock management overhead. Switching to optimistic locking with `UPDATE posts SET like_count = like_count + 1 WHERE id = ?` eliminated all locking overhead.
- **Always acquire locks in a consistent order across transactions** — Inconsistent lock ordering is the #1 cause of deadlocks. For transfers, lock the smaller account ID first. For multi-table operations, use alphabetical order.
  - **Production story:** A payment service locked `accounts` then `transactions`, while a report service locked `transactions` then `accounts`. Under daily batch processing, deadlocks killed both jobs every night for 3 months until the inconsistent ordering was identified and fixed.
- **Use SKIP LOCKED for work queue tables** — `SELECT ... FOR UPDATE` without `SKIP LOCKED` causes workers to block on locked rows. SKIP LOCKED enables horizontal scaling of workers.
- **Keep lock durations as short as possible** — Hold locks only for the critical section (read-check-write), not for slow operations. Extract heavy computation outside the transaction.
- **Set lock_timeout to fail fast instead of blocking indefinitely** — A transaction waiting on a lock blocks a connection pool thread. `SET lock_timeout = '5s'` ensures the wait fails fast.
- **Use SELECT FOR NO KEY UPDATE when you don't need to block FK checks** — `FOR UPDATE` blocks both writes and FK shared locks. `FOR NO KEY UPDATE` (PostgreSQL) blocks writes but allows FK references.
- **Handle OptimisticLockException with retry logic** — Without retry, the user sees a random failure. Implement retry with exponential backoff and clear the persistence context between attempts.
