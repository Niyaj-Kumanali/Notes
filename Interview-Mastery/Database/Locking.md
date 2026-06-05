# Database Locking

---

## What is Database Locking?

**Database locking** is the mechanism that prevents concurrent transactions from interfering with each other while maintaining data consistency. Locks control access to database resources (rows, pages, tables) and implement isolation guarantees. Understanding locking is critical for building concurrent applications — too little locking causes data corruption, too much causes contention and deadlocks.

### Key Concepts:

1. **Lock Granularity**:

   - **Row-level** — Single row locked. Highest concurrency, highest overhead.
   - **Page-level** — Disk page (multiple rows) locked. Medium concurrency and overhead.
   - **Table-level** — Entire table locked. Lowest concurrency, lowest overhead.
   - **Database-level** — Entire database locked. No concurrency.

2. **Lock Modes**:

   - **Shared (S)** — Read lock. Allows other shared locks but blocks exclusive locks.
   - **Exclusive (X)** — Write lock. Blocks all other locks.
   - **Update (U)** — Intention to update. Prevents conversion deadlocks. Compatible with S, conflicts with X.
   - **Intention Shared (IS)** — Intention to read fine-grained resources.
   - **Intention Exclusive (IX)** — Intention to write fine-grained resources.

3. **Two-Phase Locking (2PL)**:

   For transaction isolation, databases use 2PL:
   - **Growing phase** — Locks are acquired but not released.
   - **Shrinking phase** — Locks are released but not acquired.
   - **Strict 2PL** — All exclusive locks held until commit.

4. **Deadlocks**:

   A **deadlock** occurs when two transactions each hold a lock the other needs:
   ```
   Tx 1: Locks row A → wants row B = blocked
   Tx 2: Locks row B → wants row A = blocked
   ```
   Databases detect deadlocks using **wait-for graphs** and resolve by aborting one transaction (the "victim").

5. **MVCC vs Locking**:

   MVCC allows readers to see a consistent snapshot without acquiring shared locks:
   - **Readers** see the latest committed version as of their transaction start.
   - **Writers** create new row versions; old versions remain for concurrent readers.
   - **Cleanup** is handled by VACUUM (PostgreSQL) or background threads.
   - This means: `SELECT` (without `FOR UPDATE`) never blocks and is never blocked by writers in PostgreSQL/Oracle.

---

## Core Concepts

### 1. PostgreSQL Row-Level Locking

   ```sql
   SELECT * FROM orders WHERE id = 100 FOR UPDATE;        -- Exclusive row lock
   SELECT * FROM orders WHERE id = 100 FOR NO KEY UPDATE; -- Weaker exclusive (doesn't block KEY SHARE)
   SELECT * FROM orders WHERE id = 100 FOR SHARE;         -- Shared row lock
   SELECT * FROM orders WHERE id = 100 FOR KEY SHARE;     -- Weakest shared (only blocks FK key changes)
   ```

### 2. Pessimistic Locking with JPA

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

### 3. Optimistic Locking with @Version

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

### 4. Handling OptimisticLockException

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

### 5. SKIP LOCKED for Queue Processing

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

   Available in PostgreSQL 9.5+, MySQL 8.0+, SQL Server 2019+.

### 6. Lock Timeout Configuration

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

### 7. Lock Monitoring

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

---

## Common Mistakes

1. **Not using @Version for optimistic locking** — silently overwriting concurrent updates.
2. **Using pessimistic locking when optimistic would suffice** — unnecessary contention.
3. **Forgetting lock_timeout** — transactions can block indefinitely.
4. **Inconsistent lock order across transactions** — leading to deadlocks.
5. **Holding locks across user interaction (long transactions)** — lock contention escalates.
6. **SELECT without FOR UPDATE before UPDATE in concurrent writes** — lost updates.
7. **Assuming SELECT always blocks** — MVCC means SELECT doesn't block in PostgreSQL/Oracle.
8. **Not handling OptimisticLockException** — silent data loss instead of retry.
9. **Lock escalation in MySQL/SQL Server** — row locks escalate to table locks unexpectedly.
10. **Not considering application-level transactions** — Hibernate session-per-request pattern.

---

## Real-World Scenarios

### 1. Ticket Booking with Pessimistic Locking

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

### 2. Job Queue with SKIP LOCKED

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

### 3. Social Media Likes with Optimistic Locking

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

1. **Q: You are building a concert ticket booking system with 10,000 concurrent users. Seats are limited. How do you prevent overselling while maintaining throughput?**
   A: Use pessimistic locking with `FOR UPDATE SKIP LOCKED` on the seat reservation. This locks only the seats being booked and skips already-locked rows. Keep the reservation window short (5 minutes with a background job to release expired holds). For the waitlist, use optimistic locking since contention is lower.

2. **Q: Your app uses `@Version` for optimistic locking. During a flash sale with 5000 concurrent users for 100 items, almost every request fails with OptimisticLockException and retries. Retries also fail. What's happening?**
   A: Retry storm — all read version 1, only the first commit succeeds. The other 4999 retry reading version 2, but only one succeeds, etc. Fix: switch to pessimistic locking for reservation, or use atomic `UPDATE items SET stock = stock - 1 WHERE id = ? AND stock > 0`.

3. **Q: Two transactions always lock tables A then B. A third transaction locks B then A. The app experiences periodic deadlocks. How do you prevent this?**
   A: Enforce global lock ordering — always acquire locks in the same order (by resource ID numerically, or table name alphabetically). For transfers, lock the smaller account ID first. Use `ORDER BY id` when selecting rows to lock. Set `lock_timeout` to fail fast instead of blocking.

4. **Q: A reporting query running for 3 minutes blocks all order updates. The report uses default settings. How do you fix this without a replica?**
   A: In PostgreSQL, `SELECT` without `FOR UPDATE` doesn't block writes. If it blocks, the report may use `FOR SHARE` or SERIALIZABLE. In MySQL, ensure the WHERE clause uses an index to avoid row-to-table lock escalation. Set `SET TRANSACTION READ ONLY` to avoid acquiring locks.

5. **Q: A `SELECT ... FOR UPDATE` on a parent table prevents INSERTs on a child table with a foreign key. Why?**
   A: The FK constraint acquires a shared lock (`FOR KEY SHARE`) on the parent row. Locking the parent with `FOR UPDATE` blocks child INSERTs. Use `FOR NO KEY UPDATE` (PostgreSQL) which doesn't block KEY SHARE locks.

6. **Q: Your PostgreSQL app uses `SELECT ... FOR UPDATE` for inventory. Locked rows are held for only 50ms, but CPU spikes and throughput drops under load. What's going wrong?**
   A: Even short lock holds cause contention when thousands compete for the same rows. The CPU spike is from context switching and deadlock detection. Use `SKIP LOCKED` so transactions skip locked rows instead of waiting. Queue requests in application memory and batch. Use atomic `UPDATE inventory SET reserved = reserved + 1 WHERE id = ? AND reserved + 1 <= stock`.

7. **Q: A MySQL table with 10M rows frequently escalates row locks to table locks under heavy load. The WHERE clause uses `WHERE status = 'PENDING'`. What's the root cause?**
   A: MySQL escalates to table locks when a query scans too many rows without an efficient index. If `status` has low selectivity, MySQL may choose a full table scan, locking every row. Add an index on `status`. Verify with `EXPLAIN` that the query uses an index scan.

8. **Q: An OptimisticLockException is logged but the application doesn't retry. The user sees "save failed". How do you handle this properly?**
   A: Implement retry with exponential backoff at the service layer. Catch `OptimisticLockException`, clear the persistence context (`entityManager.clear()`), re-read fresh data, and retry. After max retries, throw a meaningful business exception. Use `@Retryable` from Spring Retry.

9. **Q: A database deadlock is detected and one transaction is chosen as the victim. The victim's transaction rolls back. What happens to the connection?**
   A: The connection returns to the pool in a clean state (rollback resets it). Catch the deadlock exception and retry the entire operation. Never retry on the same connection without a fresh transaction. Configure `deadlock_timeout` for faster detection.

10. **Q: A long transaction holds a lock for 30 seconds while the app processes data in memory. Other transactions block. Connection pool threads are consumed by waiters. How do you diagnose and fix this?**
    A: Check `pg_locks` for blocked processes and `pg_stat_activity` for the blocking query. Fix: never hold locks across slow application processing. Read in one short transaction, process in memory, write in another. Set `lock_timeout` to fail fast.

## Interview Questions

1. **What is the difference between shared and exclusive locks?**
   A: Shared (read) locks allow other shared locks but block exclusive locks. Exclusive (write) locks block all other locks.

2. **What is a deadlock and how does the database resolve it?**
   A: A deadlock occurs when two transactions each hold a lock the other needs (circular wait). The database uses wait-for graphs to detect and aborts one transaction (the victim).

3. **What is the difference between pessimistic and optimistic locking?**
   A: Pessimistic acquires locks upfront (`SELECT FOR UPDATE`), preventing conflicts. Optimistic detects conflicts at commit time (`@Version`), retrying on failure. Pessimistic is better for high contention.

4. **What does SKIP LOCKED do?**
   A: Skips rows already locked by other transactions instead of waiting. Used for job queues and work distribution. Available in PostgreSQL 9.5+, MySQL 8.0+.

5. **How does MVCC affect locking?**
   A: MVCC allows readers to see a consistent snapshot without shared locks. `SELECT` without `FOR UPDATE` never blocks and is never blocked by writers in PostgreSQL/Oracle.

6. **What is lock escalation?**
   A: Converting many fine-grained locks (row-level) into a coarse-grained lock (table-level). Common in MySQL when a query scans many rows without an index.

7. **What is two-phase locking (2PL)?**
   A: Transactions acquire locks in a growing phase and release in a shrinking phase. Strict 2PL holds all exclusive locks until commit/rollback, ensuring serializability.

8. **What happens with @Lock(PESSIMISTIC_WRITE) in JPA?**
   A: Hibernate issues `SELECT ... FOR UPDATE` on the entity. The row is locked until the transaction commits. Others trying to read (with FOR UPDATE) or write to that row block.

9. **How does @Version work?**
   A: Hibernate increments the version column on every UPDATE. The UPDATE includes `WHERE version = :oldVersion`. If the version changed, no rows are updated and `OptimisticLockException` is thrown.

10. **How do you monitor active locks in PostgreSQL?**
    A: Query `pg_locks` for current locks, `pg_stat_activity` for blocked queries. Use `SELECT blocked.pid, blocking.pid FROM pg_locks ... WHERE NOT blocked.granted` to find blocking chains.

## Developer Recommendations

- **Use optimistic locking (@Version) for low-contention scenarios** — Optimistic locking has zero overhead when there's no conflict. Pessimistic locking always acquires locks, adding overhead even without conflict.

- **Always acquire locks in a consistent order across transactions** — Inconsistent lock ordering is the #1 cause of deadlocks. For transfers, lock smaller account ID first. For multi-table ops, use alphabetical order.

- **Use SKIP LOCKED for work queue tables** — `SELECT ... FOR UPDATE` without `SKIP LOCKED` causes workers to block on locked rows. SKIP LOCKED enables horizontal scaling of workers.

- **Keep lock durations as short as possible** — Hold locks only for the critical section (read-check-write), not for slow operations. Extract heavy computation outside the transaction.

- **Set lock_timeout to fail fast instead of blocking indefinitely** — A transaction waiting on a lock blocks a connection pool thread. `SET lock_timeout = '5s'` ensures the wait fails fast.

- **Use SELECT FOR NO KEY UPDATE when you don't need to block FK checks** — `FOR UPDATE` blocks both writes and FK shared locks. `FOR NO KEY UPDATE` (PostgreSQL) blocks writes but allows FK references.

- **Handle OptimisticLockException with retry logic** — Without retry, the user sees a random failure. Implement retry with exponential backoff and clear the persistence context between attempts.
