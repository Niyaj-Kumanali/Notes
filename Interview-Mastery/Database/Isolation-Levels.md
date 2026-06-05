# Database Isolation Levels

---

## What are Isolation Levels?

**Isolation levels** define how and when changes made by one transaction become visible to other concurrent transactions. They control the trade-off between **data consistency** and **concurrency**. The SQL standard defines four levels: **READ UNCOMMITTED**, **READ COMMITTED**, **REPEATABLE READ**, and **SERIALIZABLE**. Higher isolation levels prevent more anomalies but reduce throughput.

### Key Concepts:

1. **Concurrency Anomalies**:

   - **Dirty Read** — Reading uncommitted changes from another transaction. Occurs at READ UNCOMMITTED.
   - **Non-Repeatable Read** — Same row read twice gives different values (another tx updated and committed between reads). Occurs at READ UNCOMMITTED and READ COMMITTED.
   - **Phantom Read** — Same query returns different rows (another tx inserted/deleted rows). Occurs below SERIALIZABLE.
   - **Lost Update** — Two transactions read the same value, both update, the last write overwrites the first. Can occur at any level unless explicitly prevented.
   - **Write Skew** — Two transactions read overlapping data and make inconsistent writes. Only prevented by SERIALIZABLE.

2. **Isolation Level Matrix**:

   - **READ UNCOMMITTED** — Dirty reads, non-repeatable reads, and phantoms are all possible. Lowest consistency, highest performance.
   - **READ COMMITTED** — Prevents dirty reads. Non-repeatable reads and phantoms are still possible. Default for PostgreSQL, Oracle, SQL Server.
   - **REPEATABLE READ** — Prevents dirty and non-repeatable reads. Phantoms still possible. Default for MySQL/InnoDB.
   - **SERIALIZABLE** — Prevents all anomalies. Highest consistency, lowest concurrency. Requires retry logic for serialization failures.

3. **Snapshot Isolation**:

   Not in the original SQL standard but widely implemented:
   - Each transaction sees a consistent snapshot from the transaction start.
   - Write-write conflicts are detected (first committer wins).
   - Prevents dirty reads, non-repeatable reads, and phantoms.
   - Does **NOT** prevent write skew.
   - Used by PostgreSQL (REPEATABLE READ), Oracle (default), SQL Server (SNAPSHOT), and MySQL (REPEATABLE READ with MVCC).

4. **MVCC-Based Implementation**:

   Most modern databases use **Multi-Version Concurrency Control** (MVCC):
   - Every write creates a new row version; old versions remain for concurrent readers.
   - Readers never block writers, writers never block readers.
   - READ COMMITTED uses a statement-level snapshot.
   - REPEATABLE READ uses a transaction-level snapshot.

---

## Core Concepts

### 1. How READ COMMITTED Works

   Each statement sees the latest committed data as of the statement start:

   ```
   Tx A: BEGIN
   Tx A: SELECT * FROM accounts WHERE id = 1 → balance = 100
   Tx B: UPDATE accounts SET balance = 200 WHERE id = 1
   Tx B: COMMIT
   Tx A: SELECT * FROM accounts WHERE id = 1 → balance = 200 (different!)
   Tx A: COMMIT
   ```

   In MVCC databases, this is implemented via **statement-level snapshots** — each query gets a fresh snapshot.

### 2. How REPEATABLE READ Works

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
   - MySQL REPEATABLE READ (InnoDB): transaction snapshot from first read + gap locks for phantom prevention.

### 3. How SERIALIZABLE Works

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

### 4. Setting Isolation Levels in Spring

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

### 5. Retry for Serialization Failures

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

### 6. Lost Update Prevention

   Lost updates require explicit mechanisms even at REPEATABLE READ:
   - **Pessimistic locking**: `SELECT ... FOR UPDATE`
   - **Optimistic locking**: `@Version` column with retry
   - **Atomic operations**: `UPDATE table SET col = col + 1`
   - **Serializable isolation**: Prevents all anomalies including lost updates

### 7. Write Skew Example

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

---

## Common Mistakes

1. **Using default isolation without understanding database defaults** — PostgreSQL defaults to READ COMMITTED, MySQL to REPEATABLE READ.
2. **Assuming REPEATABLE READ prevents all anomalies** — it doesn't prevent phantoms or write skew.
3. **Not handling serialization failures** — SERIALIZABLE requires retry logic.
4. **Using SERIALIZABLE everywhere** — massive performance impact; use only where needed.
5. **Assuming READ UNCOMMITTED provides dirty reads in PostgreSQL** — PostgreSQL treats RU as RC.
6. **Forgetting that READ COMMITTED allows non-repeatable reads** — same query, different results in same transaction.
7. **Not testing for concurrency issues** — isolation bugs only appear under load.
8. **Confusing database isolation with application-level locking** — isolation is per-transaction.
9. **Not understanding MVCC interaction with isolation** — MVCC provides snapshot isolation, not true serializable.
10. **Setting isolation at database level but overriding with different session/transaction settings.**

---

## Real-World Scenarios

### 1. Inventory Reservation Under READ COMMITTED

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

### 2. Consistent Report with REPEATABLE READ

A financial report runs for 30 seconds scanning millions of transactions. Under READ COMMITTED, each query sees different committed data. REPEATABLE READ gives a transaction-level snapshot:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Report generateMonthlyReport(YearMonth period) {
    BigDecimal revenue = transactionRepo.sumByType("REVENUE", period);
    BigDecimal expenses = transactionRepo.sumByType("EXPENSE", period);
    return new Report(revenue, expenses);
}
```

### 3. Doctor On-Call Write Skew

Two on-call doctors check the schedule. Each sees exactly one other doctor on call, so each goes off call. Under REPEATABLE READ, neither sees the other's DELETE. Result: no doctor on call (write skew):

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

## Scenario-Based Questions

1. **Q: Your booking system uses READ COMMITTED. Two users simultaneously book the last seat. Both see "available" and both book successfully — overselling by 1. How do you prevent this?**
   A: Use pessimistic locking: `SELECT ... FOR UPDATE` locks the row until the transaction commits, forcing the second user to wait. Or use SERIALIZABLE isolation with retry logic. With optimistic locking (`@Version`), the second commit fails with `OptimisticLockException` and you retry, but under high contention retries degrade throughput.

2. **Q: A reporting job at REPEATABLE READ produces inconsistent counts between users and orders tables. The report shows order counts that don't match the users table. What's happening?**
   A: REPEATABLE READ prevents non-repeatable reads within a single table but doesn't prevent phantoms across tables. If the report queries users first, then orders, and new orders are inserted between the queries, counts are inconsistent. Use SERIALIZABLE or take a snapshot timestamp and filter by `created_at <= snapshot_time`.

3. **Q: Your app uses `@Transactional(isolation = Isolation.SERIALIZABLE)`. Under high load, 30% of transactions fail with "could not serialize access". How do you fix this?**
   A: Add retry logic with exponential backoff using `@Retryable`. If contention is intrinsic, consider relaxing to REPEATABLE READ with explicit locking on critical paths. Redesign transactions to be shorter — read in SERIALIZABLE, compute, retry on conflict.

4. **Q: Two transactions both read the same set of rows, then make decisions based on what they read. Neither updates the same rows, yet the final state is inconsistent. What anomaly is this?**
   A: Write skew — each transaction reads an overlapping data set and makes writes that are individually correct but collectively inconsistent. Example: two doctors going off call. Only SERIALIZABLE or `SELECT ... FOR UPDATE` on the overlapping predicate prevents this.

5. **Q: Your PostgreSQL app uses READ UNCOMMITTED expecting dirty reads, but they don't occur. Why?**
   A: PostgreSQL does not support dirty reads — READ UNCOMMITTED behaves identically to READ COMMITTED. MVCC makes dirty reads impossible because readers always see a consistent snapshot from the statement start.

6. **Q: A heavy analytics query at READ COMMITTED blocks OLTP writes for several seconds. You cannot change the query. How do you fix this?**
   A: The analytics query may acquire shared locks. Use a read-only replica. Without replicas, set `SET TRANSACTION READ ONLY` and `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE` in PostgreSQL — SSI doesn't block writes. Alternatively, lower `lock_timeout` so the analytics query fails fast.

7. **Q: You need to choose an isolation level for a financial trading system where every trade must be accurate, but SERIALIZABLE causes too many conflicts. What do you do?**
   A: Use REPEATABLE READ for reads. For critical writes (order placement, balance updates), use `SELECT ... FOR UPDATE`. For balance updates, use atomic SQL: `UPDATE accounts SET balance = balance + ? WHERE id = ? AND balance + ? >= 0`.

8. **Q: An app at REPEATABLE READ has a long-running transaction (5 minutes) that blocks VACUUM from cleaning dead tuples. What happens?**
   A: MVCC retains old row versions visible to the long-running transaction, causing table bloat. Autovacuum cannot remove dead tuples visible to any active transaction. Keep transactions short, or set `old_snapshot_threshold` in PostgreSQL to forcibly terminate long snapshots.

9. **Q: Your PostgreSQL at READ COMMITTED has a transaction that reads a row, processes for 2 seconds, then writes. Under high concurrency, writes fail with "could not serialize access" even though you're not using SERIALIZABLE. Why?**
   A: A trigger or function may use SERIALIZABLE internally, or a deferred constraint check causes the failure. Check for serializable functions in the call stack. In rare cases, pgBouncer transaction mode can cause unexpected serialization errors.

10. **Q: Your MySQL REPEATABLE READ transaction sometimes gets duplicate key errors on INSERT from concurrent transactions. How is this possible at REPEATABLE READ?**
    A: MySQL's REPEATABLE READ uses MVCC where INSERTs don't see each other's uncommitted data due to gap locks, but the unique constraint is checked at commit time. The second committer fails. Use SERIALIZABLE to prevent it entirely, or handle unique violation errors in application code.

## Interview Questions

1. **What are the four SQL standard isolation levels?**
   A: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, and SERIALIZABLE. Higher levels prevent more anomalies but reduce concurrency.

2. **What is a dirty read?**
   A: Reading uncommitted changes from another transaction. If that transaction rolls back, you've read data that never existed. Prevented by READ COMMITTED and above.

3. **What is the difference between non-repeatable read and phantom read?**
   A: Non-repeatable read: same row read twice gives different values (row updated). Phantom read: same query returns different rows (rows inserted). REPEATABLE READ prevents non-repeatable but allows phantoms.

4. **What is snapshot isolation?**
   A: Each transaction gets a consistent snapshot at start time. Implemented via MVCC in PostgreSQL, Oracle, SQL Server, and MySQL. Prevents dirty reads, non-repeatable reads, and phantoms, but not write skew.

5. **What is write skew?**
   A: Two transactions read overlapping data and make individually correct but collectively inconsistent writes. Example: two doctors go off call leaving no coverage. Only SERIALIZABLE prevents write skew.

6. **How does MVCC implement REPEATABLE READ?**
   A: Each transaction gets a snapshot at its first read. Queries see row versions committed before snapshot time. Later writes by other transactions are invisible.

7. **Does READ COMMITTED prevent lost updates?**
   A: No. Lost updates can occur at any isolation level unless explicitly prevented with `SELECT FOR UPDATE`, `@Version`, or atomic updates.

8. **What is SSI (Serializable Snapshot Isolation)?**
   A: PostgreSQL's SERIALIZABLE implementation. Uses predicate locks to detect read-write conflicts that would produce non-serializable behavior, including write skew. One conflicting transaction is aborted.

9. **Why would you avoid SERIALIZABLE everywhere?**
   A: Highest overhead — more conflicts, more aborts, requires retry logic. Use where absolute correctness is needed (financial) and READ COMMITTED/REPEATABLE READ for everything else.

10. **What is the default isolation level in PostgreSQL vs MySQL?**
    A: PostgreSQL defaults to READ COMMITTED. MySQL (InnoDB) defaults to REPEATABLE READ. Know your database's default.

## Developer Recommendations

- **Use READ COMMITTED as the default isolation level** — It balances consistency and concurrency for most workloads. PostgreSQL uses it by default. Only escalate when data anomalies are identified and unacceptable.

- **Prefer explicit locking (SELECT FOR UPDATE) over SERIALIZABLE** — SERIALIZABLE has global overhead and requires retry logic. `SELECT FOR UPDATE` locks only specific rows, providing the same guarantee with much lower contention.

- **Always handle serialization failures with retry logic** — SERIALIZABLE transactions can abort at any time. Use `@Retryable` with exponential backoff for `CannotSerializeTransactionException`.

- **Use REPEATABLE READ for reporting queries** — Reports running for seconds should use REPEATABLE READ so all queries see a consistent snapshot. Under READ COMMITTED, different queries may see different data.

- **Test with realistic concurrency** — Isolation bugs only appear under load. Write integration tests simulating concurrent transactions using testcontainers.

- **Use atomic UPDATE instead of read-then-write** — `UPDATE accounts SET balance = balance + 100 WHERE id = 1` is immune to lost updates. The read-then-write pattern is vulnerable regardless of isolation level.

- **Know your database's MVCC behavior** — PostgreSQL's REPEATABLE READ allows phantoms via snapshots. MySQL's uses gap locks preventing some phantoms at higher contention cost.
