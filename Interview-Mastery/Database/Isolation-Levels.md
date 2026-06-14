# Database Isolation Levels

---

## What are Isolation Levels?

**Isolation levels** define how and when changes made by one transaction become visible to other concurrent transactions. They control the trade-off between **data consistency** and **concurrency**. The SQL standard defines four levels: **READ UNCOMMITTED**, **READ COMMITTED**, **REPEATABLE READ**, and **SERIALIZABLE**. Higher isolation levels prevent more anomalies but reduce throughput.

### Concurrency Anomalies

- **Dirty Read** — Reading uncommitted changes from another transaction. If that transaction rolls back, you have read data that never existed. Occurs only at READ UNCOMMITTED.
- **Non-Repeatable Read** — Reading the same row twice gives different values because another transaction updated and committed between the reads. Occurs at READ UNCOMMITTED and READ COMMITTED.
- **Phantom Read** — The same query returns different rows because another transaction inserted or deleted rows matching the filter. Occurs below SERIALIZABLE.
- **Lost Update** — Two transactions read the same value, both update it, and the last write overwrites the first. Can occur at any isolation level unless explicitly prevented.
- **Write Skew** — Two transactions read overlapping data and make individually correct but collectively inconsistent writes. Only prevented by SERIALIZABLE.

### Isolation Level Matrix

- **READ UNCOMMITTED** — Dirty reads, non-repeatable reads, and phantoms are all possible. Offers the lowest consistency but highest performance. Rarely used in practice.
- **READ COMMITTED** — Prevents dirty reads. Non-repeatable reads and phantoms are still possible. The default isolation level for PostgreSQL, Oracle, and SQL Server.
- **REPEATABLE READ** — Prevents dirty reads and non-repeatable reads. Phantoms are still possible (though MySQL/InnoDB prevents some via gap locks). Default for MySQL.
- **SERIALIZABLE** — Prevents all anomalies including write skew. Offers the highest consistency but lowest concurrency, requiring retry logic for serialization failures.

### Snapshot Isolation

Not in the original SQL standard but widely implemented:
- Each transaction sees a consistent snapshot from the transaction start time.
- Write-write conflicts are detected using a first-committer-wins strategy.
- Prevents dirty reads, non-repeatable reads, and phantoms.
- Does **NOT** prevent write skew, which is a key limitation.
- Used by PostgreSQL (REPEATABLE READ), Oracle (default), SQL Server (SNAPSHOT), and MySQL (REPEATABLE READ with MVCC).

### MVCC-Based Implementation

Most modern databases use **Multi-Version Concurrency Control (MVCC)**:
- Every write creates a new row version while old versions remain available for concurrent readers.
- Readers never block writers, and writers never block readers — a key advantage over lock-based concurrency.
- READ COMMITTED uses a statement-level snapshot where each query gets a fresh view of committed data.
- REPEATABLE READ uses a transaction-level snapshot where all queries see the same data from the first read.

---

## Core Concepts

### How READ COMMITTED Works

Each statement sees the latest committed data as of the statement start:

```
Tx A: BEGIN
Tx A: SELECT * FROM accounts WHERE id = 1 → balance = 100
Tx B: UPDATE accounts SET balance = 200 WHERE id = 1
Tx B: COMMIT
Tx A: SELECT * FROM accounts WHERE id = 1 → balance = 200 (different!)
Tx A: COMMIT
```

In MVCC databases, this is implemented via **statement-level snapshots** — each query within the transaction gets a fresh snapshot of committed data.

### How REPEATABLE READ Works

The transaction sees a consistent snapshot from its first read:

```
Tx A: BEGIN (xid = 100)
Tx A: SELECT * FROM accounts WHERE id = 1 → balance = 100
Tx B: UPDATE accounts SET balance = 200 WHERE id = 1
Tx B: COMMIT (xid = 101)
Tx A: SELECT * FROM accounts WHERE id = 1 → balance = 100 (same!)
Tx A: COMMIT
```

- PostgreSQL REPEATABLE READ: transaction snapshot from first query.
- MySQL REPEATABLE READ (InnoDB): transaction snapshot from first read plus gap locks for phantom prevention.

### How SERIALIZABLE Works

PostgreSQL uses **Serializable Snapshot Isolation (SSI)** — true serializable with predicate locks:

```
Tx A: BEGIN ISOLATION LEVEL SERIALIZABLE
Tx A: SELECT SUM(balance) FROM accounts WHERE type = 'CHECKING' → 1000
Tx B: BEGIN ISOLATION LEVEL SERIALIZABLE
Tx B: SELECT SUM(balance) FROM accounts WHERE type = 'SAVINGS' → 500
Tx A: UPDATE accounts SET balance = balance + 100 WHERE id = 1 AND type = 'CHECKING'
Tx A: COMMIT → succeeds
Tx B: UPDATE accounts SET balance = balance - 100 WHERE id = 2 AND type = 'SAVINGS'
Tx B: COMMIT → FAILS: "could not serialize access"
```

SSI detects serialization anomalies using **conflict detection** based on read-write and write-write dependencies between concurrent transactions. One transaction is aborted to ensure serializability.

### Setting Isolation Levels in Spring

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    Account from = accountRepository.findById(fromId).get();
    Account to = accountRepository.findById(toId).get();
    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));
}

@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferSerializable(Long fromId, Long toId, BigDecimal amount) {
    // Full isolation; may get serialization failures
    Account from = accountRepository.findById(fromId).get();
    Account to = accountRepository.findById(toId).get();
    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));
}
```

Choose the lowest isolation level that prevents the anomalies your application cannot tolerate. Higher isolation is not always better.

### Retry for Serialization Failures

```java
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
    throw new TransferFailedException("Transfer failed, please try again later");
}
```

Serialization failures are expected at SERIALIZABLE isolation. Retry with exponential backoff is essential for production use.

### Lost Update Prevention

Lost updates require explicit mechanisms even at REPEATABLE READ:
- **Pessimistic locking**: `SELECT ... FOR UPDATE`
- **Optimistic locking**: `@Version` column with retry
- **Atomic operations**: `UPDATE table SET col = col + 1`
- **Serializable isolation**: Prevents all anomalies including lost updates

### Write Skew Example

Two doctors both check if they are the only on-call, both see zero others, both go off-call:

```sql
-- Doctor A: SELECT count(*) FROM on_call WHERE doctor_id != 1 AND date = '2024-01-01';
-- Returns 0 (Doctor B is the only other on-call)
-- Doctor B: SELECT count(*) FROM on_call WHERE doctor_id != 2 AND date = '2024-01-01';
-- Returns 0 (Doctor A is the only other on-call)
-- Doctor A: DELETE FROM on_call WHERE doctor_id = 1 AND date = '2024-01-01';
-- Doctor B: DELETE FROM on_call WHERE doctor_id = 2 AND date = '2024-01-01';
-- Result: No doctors on call! (Write skew)
```

Prevention requires SERIALIZABLE isolation or explicit locking:
```sql
SELECT * FROM on_call WHERE date = '2024-01-01' FOR UPDATE;
```

Write skew is subtle because neither transaction reads or writes the same rows as the other, yet the combined effect is inconsistent.

---

## Common Mistakes

- **Using default isolation without understanding database defaults** — PostgreSQL defaults to READ COMMITTED, MySQL to REPEATABLE READ. Know your database's default behavior.
  - **Why it looks correct:** The application works on the developer's local database, and isolation anomalies only appear under concurrent load that most development environments never simulate.
- **Assuming REPEATABLE READ prevents all anomalies** — It prevents non-repeatable reads but not phantoms or write skew. Only SERIALIZABLE prevents the full set.
  - **Why it looks correct:** The name "REPEATABLE READ" sounds like it covers all consistency issues, and developers rarely encounter write skew until they run concurrent transactions on overlapping predicates.
- **Not handling serialization failures** — SERIALIZABLE requires retry logic. Without it, users see random "could not serialize access" errors.
  - **Why it looks correct:** The error looks like a transient database glitch rather than an expected concurrency control mechanism; developers assume a misconfiguration rather than a design requirement.
- **Using SERIALIZABLE everywhere** — Massive performance impact due to conflict detection and aborts. Use only where absolute correctness is required.
  - **Why it looks correct:** "Stronger isolation = better data safety" is intuitive; the throughput collapse under contention only becomes measurable under realistic load testing.
- **Assuming READ UNCOMMITTED provides dirty reads in PostgreSQL** — PostgreSQL treats READ UNCOMMITTED as READ COMMITTED because MVCC makes dirty reads impossible.
  - **Why it looks correct:** The SQL standard defines READ UNCOMMITTED as the lowest level with dirty reads, and developers expect the database to honor the standard rather than silently promoting to a higher isolation level.
- **Forgetting that READ COMMITTED allows non-repeatable reads** — The same query in the same transaction can return different results at READ COMMITTED.
  - **Why it looks correct:** Developers naturally assume that repeating a query within a single transaction returns consistent data; the statement-level snapshot behavior is not obvious from the transaction API.
- **Not testing for concurrency issues** — Isolation bugs only appear under realistic load. Write integration tests simulating concurrent access.
  - **Why it looks correct:** Unit tests pass, integration tests pass sequentially, and the application seems rock-solid until production traffic with concurrent transactions reveals the anomalies.
- **Confusing database isolation with application-level locking** — Isolation is a per-transaction property; application locking is orthogonal.
  - **Why it looks correct:** Both mechanisms prevent concurrent data access issues, and the boundary between database isolation guarantees and application-level synchronization is not immediately clear to developers new to transaction semantics.
- **Not understanding MVCC interaction with isolation** — MVCC provides snapshot isolation, which is not true serializable. PostgreSQL's SERIALIZABLE uses SSI on top of MVCC.
  - **Why it looks correct:** MVCC-based REPEATABLE READ prevents most visible anomalies, so developers assume it provides full serializability without understanding write skew.
- **Setting isolation at database level but overriding with different session/transaction settings** — Application-level settings override database defaults. Verify the effective isolation level in each context.
  - **Why it looks correct:** The DBA sets the database-level default and assumes all connections inherit it; the application's explicit `SET TRANSACTION ISOLATION LEVEL` silently overrides the DBA's setting without any warning.

---

## Real-World Scenarios

### Inventory Reservation Under READ COMMITTED

An e-commerce site checks stock before adding to cart. Under READ COMMITTED, two users both see 1 item in stock and both add to cart. The read is non-repeatable — between checking and updating, the stock changes:

```sql
-- Both transactions see stock = 1
SELECT stock FROM inventory WHERE product_id = 100; -- Returns 1
-- Both update, losing one update
UPDATE inventory SET stock = 0 WHERE product_id = 100;
```

Fix with `SELECT ... FOR UPDATE`:

```sql
SELECT stock FROM inventory WHERE product_id = 100 FOR UPDATE;
```

### Consistent Report with REPEATABLE READ

A financial report runs for 30 seconds scanning millions of transactions. Under READ COMMITTED, each query sees different committed data. REPEATABLE READ gives a transaction-level snapshot:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Report generateMonthlyReport(YearMonth period) {
    BigDecimal revenue = transactionRepo.sumByType("REVENUE", period);
    BigDecimal expenses = transactionRepo.sumByType("EXPENSE", period);
    return new Report(revenue, expenses);
}
```

### Doctor On-Call Write Skew

Two on-call doctors check the schedule. Each sees exactly one other doctor on call, so each goes off call. Under REPEATABLE READ, neither sees the other's DELETE — neither read the same rows:

```java
@Transactional
public void goOffCall(Long doctorId, LocalDate date) {
    int onCallCount = onCallRepo.countByDateExcludingDoctor(date, doctorId);
    if (onCallCount >= 1) { // Both see 1
        onCallRepo.deleteByDoctorIdAndDate(doctorId, date);
    }
}
```

Fix with `SELECT ... FOR UPDATE` or SERIALIZABLE isolation.

## Use Cases

Reach for isolation levels to control the consistency-vs-concurrency trade-off — higher levels prevent more anomalies but reduce throughput.

- **Financial transactions (banking, payments)** — Use SERIALIZABLE to prevent write skew and ensure account invariants (e.g., total balance never negative).
  - **Avoid when:** maximum throughput is needed and the specific anomaly is tolerable.

- **E-commerce inventory management** — Use REPEATABLE READ with `SELECT FOR UPDATE` to prevent phantom reads and overselling.
  - The `FOR UPDATE` lock serializes access to the inventory row.
  - **Avoid when:** read-only reporting — a consistent snapshot (MVCC) is sufficient.

- **Social media feeds (read-heavy)** — READ COMMITTED is sufficient — an occasional non-repeatable read on a like count is harmless.
  - Default in PostgreSQL, Oracle, and SQL Server for good reason.
  - **Avoid when:** displaying critical financial or medical data where every read must be consistent.

- **Reporting and analytics** — Snapshot isolation (MVCC) gives each report a consistent point-in-time view without blocking concurrent writers.
  - **Avoid when:** the report must reflect the absolute latest committed state — use READ COMMITTED or lock the relevant rows.

- **Booking and reservation systems** — Use SERIALIZABLE or `SELECT FOR UPDATE` to prevent double-booking under high contention.
  - Be prepared to retry serialization failures with exponential backoff.
  - **Avoid when:** the system can tolerate eventual consistency and reconcile conflicts later (e.g., waitlists).

---

## Scenario-Based Questions

- **Q:** Your booking system uses READ COMMITTED. Two users simultaneously book the last seat. Both see "available" and both book successfully — overselling by 1. How do you prevent this?
  - **A:** Use pessimistic locking: `SELECT ... FOR UPDATE` locks the row until the transaction commits, forcing the second user to wait. Or use SERIALIZABLE isolation with retry logic. With optimistic locking (`@Version`), the second commit fails with `OptimisticLockException`, but under high contention retries may degrade throughput.
  - **Interview follow-up:** With `SELECT ... FOR UPDATE`, the second user waits for the first to complete. If the first user abandons the booking without committing, how long does the second user wait, and how do you prevent indefinite blocking?
- **Q:** A reporting job at REPEATABLE READ produces inconsistent counts between users and orders tables. The report shows order counts that don't match the users table. What's happening?
  - **A:** REPEATABLE READ prevents non-repeatable reads within a single table but doesn't prevent phantoms across tables. If new orders are inserted between querying users and orders, counts become inconsistent. Use SERIALIZABLE or take a snapshot timestamp and filter by `created_at <= snapshot_time`.
  - **Interview follow-up:** Using a snapshot timestamp with `created_at <= snapshot_time` misses orders that were created before the snapshot but committed after you read the users table. How do you handle orders in-flight at snapshot time?
- **Q:** Your app uses `@Transactional(isolation = Isolation.SERIALIZABLE)`. Under high load, 30% of transactions fail with "could not serialize access". How do you fix this?
  - **A:** Add retry logic with exponential backoff using `@Retryable`. If contention is intrinsic, consider relaxing to REPEATABLE READ with explicit locking on critical paths. Redesign transactions to be shorter — read in SERIALIZABLE, compute, retry on conflict.
  - **Interview follow-up:** You add retry with exponential backoff, but under peak load, retries also fail because the conflicting transaction is still running. Now the p99 latency exceeds the API gateway timeout. How do you break this cycle?
- **Q:** Two transactions both read the same set of rows, then make decisions based on what they read. Neither updates the same rows, yet the final state is inconsistent. What anomaly is this?
  - **A:** Write skew — each transaction reads an overlapping data set and makes writes that are individually correct but collectively inconsistent. Example: two doctors going off call. Only SERIALIZABLE or `SELECT ... FOR UPDATE` on the overlapping predicate prevents this.
- **Q:** Your PostgreSQL app uses READ UNCOMMITTED expecting dirty reads, but they don't occur. Why?
  - **A:** PostgreSQL does not support dirty reads because MVCC makes them impossible — readers always see a consistent snapshot from the statement start. READ UNCOMMITTED behaves identically to READ COMMITTED in PostgreSQL.
- **Q:** A heavy analytics query at READ COMMITTED blocks OLTP writes for several seconds. You cannot change the query. How do you fix this?
  - **A:** The analytics query may acquire shared locks. Use a read-only replica. Without replicas, set `SET TRANSACTION READ ONLY` and `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE` in PostgreSQL — SSI doesn't block writes. Alternatively, lower `lock_timeout` so the analytics query fails fast.
- **Q:** You need to choose an isolation level for a financial trading system where every trade must be accurate, but SERIALIZABLE causes too many conflicts. What do you do?
  - **A:** Use REPEATABLE READ for reads. For critical writes (order placement, balance updates), use `SELECT ... FOR UPDATE`. For balance updates, use atomic SQL: `UPDATE accounts SET balance = balance + ? WHERE id = ? AND balance + ? >= 0`.
- **Q:** An app at REPEATABLE READ has a long-running transaction (5 minutes) that blocks VACUUM from cleaning dead tuples. What happens?
  - **A:** MVCC retains old row versions visible to the long-running transaction, causing table bloat. Autovacuum cannot remove dead tuples visible to any active transaction. Keep transactions short, or set `old_snapshot_threshold` in PostgreSQL to forcibly terminate long snapshots.
- **Q:** Your PostgreSQL at READ COMMITTED has a transaction that reads a row, processes for 2 seconds, then writes. Under high concurrency, writes fail with "could not serialize access" even though you're not using SERIALIZABLE. Why?
  - **A:** A trigger or function may use SERIALIZABLE internally, or a deferred constraint check causes the failure. Check for serializable functions in the call stack. In rare cases, pgBouncer transaction mode can cause unexpected serialization errors.
- **Q:** Your MySQL REPEATABLE READ transaction sometimes gets duplicate key errors on INSERT from concurrent transactions. How is this possible at REPEATABLE READ?
  - **A:** MySQL's REPEATABLE READ uses MVCC where INSERTs don't see each other's uncommitted data due to gap locks, but the unique constraint is checked at commit time. The second committer fails. Use SERIALIZABLE to prevent it entirely, or handle unique violation errors in application code.

## Interview Questions

- **What are the four SQL standard isolation levels?**
  - **A:** READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, and SERIALIZABLE. Higher levels prevent more anomalies but reduce concurrency.
- **What is a dirty read?**
  - **A:** Reading uncommitted changes from another transaction. If that transaction rolls back, you have read data that never existed. Prevented by READ COMMITTED and above.
- **What is the difference between non-repeatable read and phantom read?**
  - **A:** Non-repeatable read: same row read twice gives different values (row updated). Phantom read: same query returns different rows (rows inserted). REPEATABLE READ prevents non-repeatable but allows phantoms.
- **What is snapshot isolation?**
  - **A:** Each transaction gets a consistent snapshot at start time, implemented via MVCC. Prevents dirty reads, non-repeatable reads, and phantoms, but not write skew.
- **What is write skew?**
  - **A:** Two transactions read overlapping data and make individually correct but collectively inconsistent writes. Example: two doctors go off call leaving no coverage. Only SERIALIZABLE prevents write skew.
- **How does MVCC implement REPEATABLE READ?**
  - **A:** Each transaction gets a snapshot at its first read. Queries see row versions committed before the snapshot time. Later writes by other transactions are invisible.
- **Does READ COMMITTED prevent lost updates?**
  - **A:** No. Lost updates can occur at any isolation level unless explicitly prevented with `SELECT FOR UPDATE`, `@Version`, or atomic update statements.
- **What is SSI (Serializable Snapshot Isolation)?**
  - **A:** PostgreSQL's SERIALIZABLE implementation that uses predicate locks to detect read-write conflicts producing non-serializable behavior, including write skew. One conflicting transaction is aborted.
- **Why would you avoid SERIALIZABLE everywhere?**
  - **A:** Highest overhead — more conflicts, more aborts, requires retry logic. Use where absolute correctness is needed (financial) and READ COMMITTED or REPEATABLE READ for everything else.
- **What is the default isolation level in PostgreSQL vs MySQL?**
  - **A:** PostgreSQL defaults to READ COMMITTED. MySQL (InnoDB) defaults to REPEATABLE READ. Know your database's default.

## Developer Recommendations

- **Use READ COMMITTED as the default isolation level** — It balances consistency and concurrency for most workloads. PostgreSQL uses it by default. Only escalate when data anomalies are identified and unacceptable.
  - **Production story:** A financial reconciliation system used SERIALIZABLE everywhere "to be safe." Under 200 concurrent users, 60% of transactions aborted and retried, causing a cascading failure that took the system down for 2 hours. Switching to READ COMMITTED with targeted `SELECT FOR UPDATE` resolved the outage.
- **Prefer explicit locking (SELECT FOR UPDATE) over SERIALIZABLE** — SERIALIZABLE has global overhead and requires retry logic. `SELECT FOR UPDATE` locks only specific rows, providing the same guarantee with much lower contention.
- **Always handle serialization failures with retry logic** — SERIALIZABLE transactions can abort at any time. Use `@Retryable` with exponential backoff for `CannotSerializeTransactionException`.
  - **Production story:** A team deployed SERIALIZABLE without retry logic. Users saw "could not serialize access" errors on every concurrent operation. The on-call engineer spent 4 hours debugging before realizing the missing retry was the root cause.
- **Use REPEATABLE READ for reporting queries** — Reports running for seconds should use REPEATABLE READ so all queries see a consistent snapshot. Under READ COMMITTED, different queries may see different data.
- **Test with realistic concurrency** — Isolation bugs only appear under load. Write integration tests simulating concurrent transactions using testcontainers.
- **Use atomic UPDATE instead of read-then-write** — `UPDATE accounts SET balance = balance + 100 WHERE id = 1` is immune to lost updates. The read-then-write pattern is vulnerable regardless of isolation level.
- **Know your database's MVCC behavior** — PostgreSQL's REPEATABLE READ allows phantoms via snapshots. MySQL's uses gap locks preventing some phantoms at higher contention cost.
