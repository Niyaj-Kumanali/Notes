# Java Concurrency

---

## Overview

- **Purpose**
  - Java Concurrency encompasses the APIs, language features, and best practices for managing multiple threads that access shared resources safely and efficiently.
  - Without proper concurrency tools, developers relied on low-level `synchronized` blocks and `wait()`/`notify()` pairs — mechanisms that are notoriously error-prone and difficult to reason about in complex systems.

- **Modern API**
  - The modern concurrency API in `java.util.concurrent` provides higher-level abstractions including thread pools, concurrent collections, atomic variables with lock-free semantics, and coordination primitives like `CountDownLatch`, `CyclicBarrier`, `Semaphore`, and `Phaser`.
  - The goal of these abstractions is to make concurrent code easier to write correctly while maximizing throughput and minimizing contention.

- **Core Packages**
  - `java.util.concurrent` — executors, locks, synchronizers, and concurrent collections.
  - `java.util.concurrent.atomic` — `AtomicInteger`, `AtomicReference`, `LongAdder`, and related CAS-based primitives.
  - `java.util.concurrent.locks` — `ReentrantLock`, `ReadWriteLock`, `StampedLock`, and `Condition`.

---

## The Java Memory Model (JMM)

- **Definition**
  - The Java Memory Model defines the formal rules for how threads interact through memory and when changes made by one thread become visible to others.
  - Without the JMM, compilers and CPUs could reorder instructions freely, making unsynchronized concurrent access completely unpredictable across different hardware platforms.

  **Why was the JMM formally specified (JSR 133, Java 5)?**
    - Before Java 5, the memory model was incomplete — it did not prevent common hardware reorderings on platforms like Alpha (which allowed reordering of normal loads).
    - Double-checked locking was widely used but broken because it relied on the assumption that the constructor completed before the reference was published — the old model allowed the compiler to reorder writes to the object's fields after the write of the reference to the shared variable.
    - JSR 133 fixed this by: strengthening the `volatile` semantics (volatile write → happens-before → volatile read), defining the `final` field guarantee (properly constructed object with final fields is safe to publish without synchronization), and establishing transitive happens-before ordering as the backbone of the model.

- **Happens-Before**
  - The JMM is built on the happens-before relationship: if action A happens-before action B, then A's results are visible to B, and A must appear before B in program order regardless of compiler or CPU optimizations.
  - The key happens-before rules are:
    - Program order within a single thread.
    - Monitor lock release before subsequent acquisition of the same lock.
    - Volatile write before subsequent read of the same field.
    - `Thread.start()` before any action in the started thread.
    - All actions in a thread before `Thread.join()` returns successfully.
    - Transitivity across the chain.

- **Importance**
  - Understanding the JMM is essential for diagnosing visibility bugs where one thread writes a value but another thread never sees it — the most common symptom of missing synchronization.

---

## Synchronization Mechanisms

### synchronized

- **Definition**
  - The `synchronized` keyword uses the intrinsic lock built into every Java object header, providing mutual exclusion and visibility guarantees in a single construct.
  - An instance method `public synchronized void increment()` locks on `this`, a static method locks on the `Class` object, and a synchronized block allows specifying an explicit lock object.

- **Monitor Evolution**
  - The JVM heavily optimizes `synchronized` through monitor evolution:
    - Under no contention, biased locking embeds the thread ID in the object header with zero CAS overhead.
    - Under light contention, lightweight locking uses CAS on the object header without blocking.
    - Under heavy contention, the lock inflates to an OS mutex that parks the thread.
  - The upgrade is one-way from biased through lightweight to heavyweight, and after Java 15 biased locking was deprecated due to its complexity in modern concurrent systems.

```java
// Instance method — locks on this
public synchronized void increment() { count++; }

// Static method — locks on Class object
public static synchronized void reset() { total = 0; }

// Block — explicit lock object
public void update() {
    synchronized(lock) {
        // critical section
    }
}
```

### volatile

- **Definition**
  - The `volatile` keyword ensures that every write to a field is immediately visible to all subsequent reads of that field across all threads by preventing CPU caching and instruction reordering.
  - However, `volatile` guarantees only visibility, not atomicity — an operation like `volatile int count; count++` is a read-modify-write sequence that remains a race condition because the increment itself is not atomic.

- **Usage**
  - Use `volatile` for flags, status indicators, and reference assignments where compound actions are not required.
  - For counters and accumulators, use `AtomicInteger`, `AtomicLong`, or `LongAdder` which provide atomic read-modify-write operations.

```java
private volatile boolean running = true;
```

### ReentrantLock

- **Definition**
  - `ReentrantLock` is a more flexible alternative to `synchronized` that provides features unavailable with intrinsic locks.
  - It supports `tryLock()` with a timeout to avoid indefinite blocking, a fairness parameter in the constructor to prevent thread starvation, and `Condition` objects for more sophisticated wait-and-notify patterns than `synchronized`'s single wait set.

- **Key Difference**
  - Unlike `synchronized` which automatically unlocks when the block exits, `ReentrantLock` requires explicit `unlock()` in a `finally` block — forgetting to unlock causes a lock leak that eventually hangs the system.
  - Performance-wise, `synchronized` and `ReentrantLock` are comparable in modern JVMs, so the choice should be based on feature requirements rather than performance assumptions.

```java
private final ReentrantLock lock = new ReentrantLock();

public void doWork() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();  // Always unlock in finally
    }
}
```

### ReadWriteLock

- **Definition**
  - `ReadWriteLock` maintains a pair of associated locks: a shared read lock that multiple threads can hold simultaneously, and an exclusive write lock that no other thread may hold when a writer is active.
  - This separation allows concurrent read operations to proceed without blocking each other, dramatically improving throughput in read-heavy, write-rare scenarios like caches, configuration stores, and lookup tables.

- **Implementation**
  - `ReentrantReadWriteLock` is the standard implementation, and it supports fairness and reentrancy on both locks.
  - The primary risk is writer starvation — under continuous read load, the writer may never acquire the lock because readers keep acquiring and releasing the read lock.

```java
private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

public V get(K key) {
    rwLock.readLock().lock();
    try { return cache.get(key); }
    finally { rwLock.readLock().unlock(); }
}

public void put(K key, V value) {
    rwLock.writeLock().lock();
    try { cache.put(key, value); }
    finally { rwLock.writeLock().unlock(); }
}
```

### StampedLock (Java 8+)

- **Definition**
  - `StampedLock` goes beyond `ReadWriteLock` by supporting optimistic reads that are entirely lock-free — no blocking or CAS overhead under the common case where no writer is active.
  - An optimistic read acquires a stamp via `tryOptimisticRead()`, performs the read operation, then validates the stamp with `validate(stamp)`.
  - If a concurrent write occurred between the read and validation, the stamp is invalidated and the read is retried with a regular read lock.

- **Advantage**
  - This provides the best possible throughput for read-mostly workloads because optimistic reads never block writers, eliminating the writer starvation problem of `ReadWriteLock`.
  - Note that `StampedLock` is not reentrant, and its locks are not `synchronized`-block-compatible.

```java
private final StampedLock lock = new StampedLock();

public double getBalance() {
    long stamp = lock.tryOptimisticRead();
    double balance = this.balance;
    if (!lock.validate(stamp)) {  // Write occurred during read
        stamp = lock.readLock();
        try { balance = this.balance; }
        finally { lock.unlockRead(stamp); }
    }
    return balance;
}
```

---

## Atomic Variables

- **Purpose**
  - Atomic variables in `java.util.concurrent.atomic` provide lock-free, thread-safe operations on single variables using Compare-And-Swap (CAS) CPU instructions at the hardware level.
  - `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, and `AtomicReference` support atomic operations like `incrementAndGet()`, `decrementAndGet()`, `compareAndSet()`, and `getAndUpdate()` without any operating system thread blocking.

- **CAS Mechanism**
  - CAS works as a three-step CPU instruction: read a memory location, compare it with an expected value, and write the new value only if the comparison succeeds, returning success or failure atomically.
  - The ABA problem occurs when a value changes from A to B and back to A — CAS does not detect this change because the current value still matches the expected A.

- **ABA Solution**
  - `AtomicStampedReference` and `AtomicMarkableReference` solve the ABA problem by pairing the reference with an integer stamp or boolean mark that is updated on every change.

```java
private final AtomicInteger counter = new AtomicInteger(0);

public int incrementAndGet() {
    return counter.incrementAndGet();  // CAS-based, no lock
}
```

- **LongAdder (Java 8+)**
  - A specialized counter that outperforms `AtomicLong` under high write contention.
  - Instead of having all threads CAS on the same memory location (causing cache line bouncing), `LongAdder` maintains a base value plus an array of cells — each thread updates a randomly chosen cell with zero contention because different threads target different cells.
  - Reading the sum requires accumulating all cell values, which is O(n) where n is the number of cells (typically a power of two up to the number of CPU cores).

  **When to use which?**
    - Use `LongAdder` for write-heavy counters where the sum is read infrequently.
    - Use `AtomicLong` for read-heavy or low-contention counters.

```java
private final LongAdder requests = new LongAdder();
public void recordRequest() { requests.increment(); }
public long getRequestCount() { return requests.sum(); }
```

---

## Concurrent Collections

- **Overview**
  - Concurrent collections in `java.util.concurrent` are thread-safe data structures that provide significantly better performance than wrapping a regular collection with `Collections.synchronizedMap()`.
  - `ConcurrentHashMap` uses per-bucket locking (Java 8+) where only writes to the same hash bucket synchronize, while reads are entirely lock-free through volatile field access — multiple threads can read and write different buckets simultaneously without contention.
  - `CopyOnWriteArrayList` creates a fresh copy of the underlying array on every mutation, making reads completely lock-free at the cost of O(n) writes, ideal for read-heavy listener registries.
  - `ConcurrentLinkedQueue` is a lock-free unbounded queue using CAS operations with Michael-Scott algorithm, suitable for high-throughput producer-consumer patterns.
  - `LinkedBlockingQueue` and `ArrayBlockingQueue` are bounded blocking queues backed by `ReentrantLock` and `Condition`, providing backpressure for producer-consumer systems.
  - `ConcurrentSkipListMap` and `ConcurrentSkipListSet` provide sorted concurrent navigation with O(log n) operations using a skip-list data structure that supports lock-free reads.

| Collection | Design | Best For |
|-----------|--------|----------|
| `ConcurrentHashMap` | Per-bucket locking (JDK 8+) | High-concurrency KV |
| `CopyOnWriteArrayList` | Snapshot array on write | Read-heavy, write-rare |
| `ConcurrentLinkedQueue` | Lock-free (CAS) | High-throughput queue |
| `LinkedBlockingQueue` | ReentrantLock + Conditions | Producer-consumer |
| `ArrayBlockingQueue` | Bounded, array-backed | Bounded producer-consumer |
| `ConcurrentSkipListMap` | Skip list | Sorted concurrent KV |

---

## Synchronizers

- **Overview**
  - The `java.util.concurrent` package provides higher-level coordination primitives that solve common synchronization patterns without requiring manual `wait()`/`notify()` code.

- **CountDownLatch**
  - A one-shot barrier: one or more threads await until a count reaches zero via `countDown()` calls, after which the latch cannot be reset.

- **CyclicBarrier**
  - A reusable barrier where a fixed number of threads rendezvous at the barrier point, and when all arrive, they all proceed and the barrier automatically resets for the next phase.

- **Semaphore**
  - Controls access to a pool of resources by maintaining a set of permits — threads acquire permits to proceed and release them when done, naturally implementing resource pools with backpressure.

- **Phaser**
  - The most flexible synchronizer, supporting dynamic party registration and multi-phase barriers, effectively replacing both `CountDownLatch` and `CyclicBarrier` with a single, more powerful API.

```java
// CountDownLatch — wait for N operations (one-shot)
CountDownLatch latch = new CountDownLatch(3);
// In threads: latch.countDown();
// In main: latch.await();  // blocks until count reaches 0

// CyclicBarrier — N threads meet at barrier (reusable)
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All reached!"));
// barrier.await() in each thread — blocks until all 3 arrive

// Semaphore — control access count
Semaphore permits = new Semaphore(10);  // 10 concurrent accesses
permits.acquire();
try { /* access resource */ }
finally { permits.release(); }

// Phaser — flexible barrier (replaces CountDownLatch + CyclicBarrier)
Phaser phaser = new Phaser(3);
phaser.arriveAndAwaitAdvance();
```

---

## CompletableFuture (Java 8+)

- **Definition**
  - `CompletableFuture` is a `Future` that can be explicitly completed and chained with callback-based asynchronous operations, enabling declarative async pipelines without blocking or manual thread management.
  - Unlike the original `Future` (Java 5) which only supports blocking `get()` calls, `CompletableFuture` supports:
    - Non-blocking callbacks (`thenApply`, `thenAccept`, `thenRun`).
    - Composition (`thenCompose` for flat-mapping, `thenCombine` for joining two independent results).
    - Error handling (`exceptionally` to recover with a fallback, `handle` to process success or failure).
    - Timeouts (`orTimeout` in Java 9+).
  - It also provides `allOf()` and `anyOf()` for combining multiple futures, and `supplyAsync()` / `runAsync()` for executing tasks on configurable executors.
  - The framework uses the common `ForkJoinPool` by default, but a dedicated executor should always be provided for I/O-bound tasks.

```java
CompletableFuture<User> future = CompletableFuture
    .supplyAsync(() -> fetchUser(id))         // async execution
    .thenApply(user -> enrichProfile(user))   // transform
    .thenAccept(user -> cache.put(id, user))  // consume
    .exceptionally(ex -> {
        log.error("Failed", ex);
        return User.getDefault();
    });  // error handling

// Combine two async operations
CompletableFuture<Order> orderFuture =
    CompletableFuture.supplyAsync(() -> fetchOrder(orderId))
    .thenCombine(
        CompletableFuture.supplyAsync(() -> fetchPayments(orderId)),
        (order, payments) -> order.applyPayments(payments)
    );

// Wait for all / any
CompletableFuture.allOf(f1, f2, f3).join();
CompletableFuture.anyOf(f1, f2, f3).join();
```

---

## Common Mistakes

- **Double-checked locking without `volatile`**
  - Without `volatile`, the compiler and CPU can reorder the writes to the field and the constructor, causing a thread to see a partially-constructed object.
  - This *looks correct* because the code checks if the field is `null` before and after synchronization — the double-check appears to guarantee safety, and the pattern was widely published in the 1990s before JSR 133 fixed the JMM.
  - The fix is either to declare the field `volatile` or use the initialize-on-demand holder class pattern.

- **Coarse `synchronized` scopes**
  - Making a `synchronized` scope too coarse destroys concurrency by serializing all operations that could run in parallel.
  - This *looks correct* because using `synchronized(this)` is the simplest way to make a class thread-safe — it covers all fields in one lock, and the code works correctly with no data races or lost updates.
  - The performance penalty only appears under load when multiple threads contend for unrelated fields.
  - If `synchronized(this)` protects unrelated fields, split into separate lock objects for unrelated data, or use `ConcurrentHashMap` and atomic variables to eliminate the lock entirely.

- **Forgetting `lock.unlock()` in `finally`**
  - If the critical section throws an exception, the lock is never released, and any thread subsequently waiting for that lock blocks forever.
  - This *looks correct* because the `unlock()` call after the critical section mirrors the structure of `synchronized` blocks where the lock is automatically released at the closing brace — the code compiles and runs successfully for the normal case.
  - The leak only manifests when an exception occurs in the protected code, which may be rare in testing.
  - Always structure `ReentrantLock` usage as `lock.lock(); try { ... } finally { lock.unlock(); }`.

- **Using `String` as a lock object**
  - String interning means two identical string literals may refer to the same `String` object, creating unintended lock sharing across unrelated code paths.
  - This *looks correct* because strings are natural identifiers — using a user ID or transaction ID as a lock makes the code readable and seems to correctly serialize operations on that specific entity.
  - The unintended sharing only surfaces when two unrelated features happen to use the same string literal, producing mysterious contention.
  - Always use `new Object()` as a lock for fine-grained locking.

- **Ignoring `InterruptedException`**
  - Catching `InterruptedException` and ignoring it loses the interrupt signal, leaving the thread unable to respond to cancellation requests.
  - This *looks correct* because the catch block satisfies the compiler's checked exception requirement, and the thread continues its work without any visible error — the cancellation remains pending silently.
  - The problem only surfaces when `shutdownNow()` is called and threads refuse to stop.
  - The correct pattern is either to propagate the exception or restore the interrupt flag with `Thread.currentThread().interrupt()`.

- **Publishing `this` from a constructor**
  - Starting a thread or registering a listener in a constructor exposes a partially-constructed object to other threads, violating the JMM's final-field safety guarantee.
  - This *looks correct* because the constructor logically sets up all fields before starting the thread — the code reads in execution order.
  - However, the JVM and CPU can reorder writes within the constructor, and the started thread may observe default values (null, 0) for fields that appear before the `thread.start()` call in source code.
  - Even if the field is `final`, the JMM guarantees safe publication only for objects whose constructors have completed.
  - Use a factory method (`create()`) or `@PostConstruct` that starts threads after the constructor returns.

- **Assuming `ConcurrentHashMap` stream is a consistent snapshot**
  - `ConcurrentHashMap` iterators reflect the state at the time of iteration and are weakly consistent — they may reflect some, all, or none of the concurrent modifications.
  - This *looks correct* because the word "Concurrent" in the class name implies thread safety, and the iterator does not throw `ConcurrentModificationException` — it silently traverses without errors.
  - The inconsistency only appears when the stream result is used for a critical decision that requires a point-in-time view.
  - For a truly consistent snapshot, use `new HashMap<>(concurrentMap)` which copies all entries at a point in time.

---

## Real-World Scenarios

### Scenario 1: Real-Time Order Matching Engine

- **Context**
  - A stock exchange matches buy and sell orders. Thousands of orders arrive per second from multiple trading firms.
  - The matching engine must maintain an order book per stock, match orders instantly by price-time priority, and publish trades — all with strict fairness guarantees and zero data loss.

```java
public class OrderBook {
    private final ConcurrentNavigableMap<Long, Queue<Order>> bids = new ConcurrentSkipListMap<>(Comparator.reverseOrder());
    private final ConcurrentNavigableMap<Long, Queue<Order>> asks = new ConcurrentSkipListMap<>();

    public List<Trade> match(Order order) {
        if (order.isBuy()) {
            NavigableMap<Long, Queue<Order>> matchingAsks = asks.headMap(order.getPrice(), true);
            return matchAgainst(order, matchingAsks);
        } else {
            NavigableMap<Long, Queue<Order>> matchingBids = bids.headMap(order.getPrice(), true);
            return matchAgainst(order, matchingBids);
        }
    }
}
```

- **What makes this work**
  - `ConcurrentSkipListMap` provides thread-safe sorted access with O(log n) insertion and query, maintaining price-time priority ordering without external synchronization.
  - `headMap()` returns a navigable view of entries with keys less than or equal to the incoming order's price, enabling efficient price-level matching.
  - Each price level uses a `ConcurrentLinkedQueue` for FIFO ordering among same-price orders, using lock-free CAS operations to avoid contention.
  - The skip-list's probabilistic balancing and lock-free reads allow multiple matching threads to operate on different price levels simultaneously, scaling with the number of CPU cores.

  **Why this approach?**
    - The alternative — a single `PriorityQueue` protected by `synchronized` — would serialize all order insertions and matching, capping throughput at roughly 50K orders/second regardless of CPU cores.
    - The concurrent skip-list + per-price-level queue design allows order matching at 1M+ orders/second because:
      - Multiple matching threads work on different price levels with zero contention.
      - The skip-list's lock-free reads let matching threads scan the book without blocking.
      - `ConcurrentLinkedQueue`'s CAS-based enqueue handles burst arrivals without lock contention.
    - The trade-off is higher per-element overhead (~80 bytes per node) and O(log n) reads vs O(1) for `HashMap` — acceptable because price levels rarely exceed 10K entries.

### Scenario 2: Distributed Tracing with CompletableFuture

- **Context**
  - A microservice receives a request and must fan out to five downstream services — user profile, order history, recommendations, inventory, and pricing.
  - The response aggregates data from all services with a 200ms deadline. If any service times out, the response must use cached fallback data.

```java
public class AggregationService {
    private final ExecutorService executor = Executors.newFixedThreadPool(20);

    public CompletableFuture<AggregatedResponse> getDashboard(String userId) {
        CompletableFuture<UserProfile> profile = fetchAsync("/users/" + userId);
        CompletableFuture<List<Order>> orders = fetchAsync("/orders?userId=" + userId);
        CompletableFuture<List<Product>> recs = fetchAsync("/recommendations?userId=" + userId);

        return CompletableFuture.allOf(profile, orders, recs)
            .applyToEither(timeoutAfter(200, MILLISECONDS), v ->
                new AggregatedResponse(
                    profile.getNow(fallbackProfile()),
                    orders.getNow(List.of()),
                    recs.getNow(List.of())
                ))
            .exceptionally(ex -> new AggregatedResponse(fallbackProfile(), List.of(), List.of()));
    }

    private <T> CompletableFuture<T> fetchAsync(String path) {
        return CompletableFuture.supplyAsync(() -> restClient.get(path), executor)
            .orTimeout(150, MILLISECONDS);
    }
}
```

- **What makes this work**
  - `CompletableFuture.allOf()` combines three independent async calls and completes when all three complete or any one fails.
  - The `applyToEither()` with `timeoutAfter()` implements a deadline race — if the 200ms aggregate deadline arrives before all services respond, the dashboard uses fallback data from `getNow()`.
  - Each individual call has a 150ms timeout via `orTimeout()`, preventing a single slow service from consuming the entire latency budget.
  - The dedicated `executor` with 20 threads isolates the I/O operations from the common ForkJoinPool, preventing thread starvation across the application.

  **Why this approach?**
    - The blocking alternative — launching three threads, calling `future.get(150ms)` on each, catching timeouts, and returning fallback — requires manual thread management and explicit timeout handling.
    - `CompletableFuture` composes these concerns declaratively: `orTimeout()` sets per-call deadlines, `allOf()` + `applyToEither()` with a 200ms global timeout implements the deadline race, and `getNow()` provides fallback values without checking individual completion states.
    - The dedicated executor (20 threads vs 8 cores) is sized for I/O — each thread spends most of its time waiting for HTTP responses, so more threads than cores improves throughput.

### Scenario 3: Database Migration with Phaser

- **Context**
  - A schema migration tool must run 8 migration scripts in parallel across 4 database shards.
  - After all migrations complete, the tool validates constraints and then switches traffic to the new schema.
  - The phases must be coordinated: all shards complete script 1 before any starts script 2.

```java
public class MigrationCoordinator {
    private final Phaser phaser = new Phaser(1); // register main thread
    private final List<Path> shards = List.of("shard1", "shard2", "shard3", "shard4");
    private final List<Path> scripts = List.of("v1.sql", "v2.sql", "v3.sql");

    public void migrate() {
        for (Path shard : shards) {
            phaser.register(); // register worker for this shard
            executor.submit(() -> {
                for (Path script : scripts) {
                    runScript(shard, script);
                    phaser.arriveAndAwaitAdvance(); // wait for all shards to finish this script
                }
                phaser.arriveAndDeregister();
            });
        }
        phaser.arriveAndAwaitAdvance(); // wait for script 1 on all shards
        phaser.arriveAndAwaitAdvance(); // wait for script 2 on all shards
        phaser.arriveAndAwaitAdvance(); // wait for script 3 on all shards
        phaser.arriveAndDeregister();
        validateAllShards();
        switchTraffic();
    }
}
```

- **What makes this work**
  - `Phaser` is the ideal choice for this multi-phase parallel workload because it supports dynamic party registration (each shard registers upon creation), reusable barriers across multiple phases, and automatic phase advancement without manual reset.
  - Each shard worker registers as a party, runs all three scripts — advancing through the phaser after each script to ensure all shards complete before any starts the next — and deregisters when finished.
  - The main thread participates as party 1, advancing through phases after each migration script and running validation and traffic switching after all shards have deregistered.
  - Compared to `CyclicBarrier`, `Phaser` avoids the need to know the exact party count at construction time.

  **Why this approach?**
    - A `CountDownLatch` per script would need three separate latches with manual wiring — one for each script phase, plus error handling if a shard fails mid-migration.
    - `CyclicBarrier` requires knowing the party count at construction (4 shards + 1 main = 5), but if a shard goes offline, the barrier waits forever.
    - `Phaser` handles both problems: dynamic registration (`phaser.register()`) means shards can join as they start, and `arriveAndDeregister()` lets failed shards be cleanly removed without blocking the remaining shards.
    - The main thread participates as an ordinary party, so it naturally advances through phases alongside the shards.

---

## Use Cases

The `java.util.concurrent` package provides production-ready building blocks for most concurrent programming needs, from thread management to coordination.

- **Thread pools for request handling** — Manage a fixed or cached set of worker threads to avoid the cost of per-task thread creation.
  - Use `Executors.newFixedThreadPool(n)` or `newCachedThreadPool()` for web server request processing. Example: accepting HTTP connections with a bounded pool to prevent resource exhaustion.
  - **Avoid when:** tasks are long-lived CPU-bound and the pool is sized larger than available cores — oversubscription degrades throughput.

- **Concurrent collections** — Share data across threads without external synchronization when access patterns fit the collection's design.
  - Use `ConcurrentHashMap` for high-read, moderate-write maps. Example: an in-memory cache of user sessions updated by one thread and read by many.
  - **Avoid when:** compound operations (iterate-then-modify) are needed — use `compute` or `merge` atomic methods instead.

- **Atomic counters and accumulators** — Safely increment or update a single numeric value without locks.
  - Use `AtomicInteger`, `LongAdder`, or `AtomicReference` for metrics, sequence generators, or status flags. Example: counting requests per second across dozens of threads.
  - **Avoid when:** multiple related fields must change atomically — use a lock or `StampedLock` for multi-field invariants.

- **Coordination primitives** — Orchestrate multi-thread workflows where threads must wait for each other or for external conditions.
  - Use `CountDownLatch` for one-shot barriers, `CyclicBarrier` for repeated synchronization, `Phaser` for dynamic party sets, `Semaphore` for resource access limits. Example: a batch processor that starts all workers after data is fully loaded.
  - **Avoid when:** a simpler `CompletableFuture` chain or structured concurrency (`StructuredTaskScope`) can express the same orchestration more declaratively.

## Scenario-Based Questions

**Q: You are designing a real-time chat server that handles 100K concurrent connections. Each user can be in multiple chat rooms. Messages must be delivered to all members of a room within 100ms. The server runs on 8 cores. How do you structure the concurrency?**

- **Solution**
  - Use the reactor pattern with a small number of event loop threads (one per core) and non-blocking I/O, assigning each chat room to a single event loop thread to eliminate locking on room state:

```java
public class ChatServer {
    private final EventLoopGroup group = new EventLoopGroup(Runtime.getRuntime().availableProcessors());
    private final ConcurrentHashMap<String, EventLoop> roomAssignments = new ConcurrentHashMap<>();

    public void joinRoom(String roomId, Channel channel) {
        EventLoop eventLoop = roomAssignments.computeIfAbsent(roomId, id -> group.next());
        eventLoop.execute(() -> roomRegistry.get(roomId).addMember(channel));
    }
}
```

  - Each room is pinned to a specific event loop thread via `computeIfAbsent`, so all operations on that room's state happen on the same thread with no locks needed.
  - The `ConcurrentHashMap` for room-to-thread mappings handles concurrent room creation from multiple event loop threads.
  - With 8 event loop threads using NIO (via Netty or similar), the server scales to 100K connections — the thread-per-connection model would require 100K threads consuming over 1GB of stack memory alone.

> **Interview follow-up:** The candidate pinned each room to a single event loop thread to eliminate locks. If a room has 10K members and broadcasting a message to all of them takes 200ms on the event loop thread, all other rooms assigned to that same event loop are delayed. How would you prevent a large room from starving smaller rooms on the same event loop?

---

**Q: A payment processing system uses `synchronized` blocks to ensure idempotency — only one thread processes a given payment ID. Under load, throughput drops and threads pile up waiting for the lock. How do you scale beyond a single-threaded bottleneck?**

- **Solution**
  - Replace the single monolithic lock with striped locking — one lock per payment ID hash bucket, allowing parallel processing of different payment IDs:

```java
public class IdempotentProcessor {
    private final Striped<Lock> locks = Striped.lock(1024); // Guava's Striped

    public void process(Payment payment) {
        Lock lock = locks.get(payment.getId()); // deterministic lock per ID
        lock.lock();
        try {
            if (alreadyProcessed(payment.getId())) return;
            processPayment(payment);
        } finally {
            lock.unlock();
        }
    }
}
```

  - `Striped.lock(1024)` creates 1024 distinct locks, and `locks.get(payment.getId())` deterministically maps each payment ID to one of those locks.
  - Payments with the same ID always map to the same lock (ensuring per-ID serialization for idempotency), but different IDs naturally distribute across different locks, allowing up to 1024 concurrent payments to be processed in parallel.
  - This is the same sharded-locking principle that `ConcurrentHashMap` uses internally.
  - The stripe count should be a power of two and tuned so that lock contention stays below 5%.

> **Interview follow-up:** The candidate chose 1024 stripes. If payment IDs are sequential integers (1, 2, 3...), the modulo distribution maps consecutive IDs to consecutive stripes. How would you handle the case where one stripe becomes a hot spot because a single payment ID accounts for 50% of all requests?

---

**Q: A caching system stores computed results in a `ConcurrentHashMap<K, CompletableFuture<V>>`. Multiple threads may request the same key simultaneously. Only one should compute; others should wait for the result. The computation may fail — future callers should retry, not get the failed result. How do you handle this?**

- **Solution**
  - Use `computeIfAbsent()` for atomic lazy initialization combined with automatic entry removal on computation failure:

```java
public class ComputingCache<K, V> {
    private final ConcurrentHashMap<K, CompletableFuture<V>> cache = new ConcurrentHashMap<>();

    public V get(K key) throws ExecutionException, InterruptedException {
        CompletableFuture<V> future = cache.computeIfAbsent(key, k -> CompletableFuture
            .supplyAsync(() -> compute(k), executor)
            .whenComplete((result, ex) -> {
                if (ex != null) cache.remove(k); // remove on failure → retry
            }));
        try {
            return future.get();
        } catch (ExecutionException e) {
            cache.remove(key, future); // only remove if still the same future
            throw e;
        }
    }
}
```

  - `computeIfAbsent` guarantees that only the first caller executes the computation — subsequent callers receive the same `CompletableFuture` and block on `future.get()`.
  - The `whenComplete` callback removes the map entry if the computation fails, so the next caller retries the computation rather than receiving the failed result.
  - The `cache.remove(key, future)` in the catch block handles the edge case where the future completed exceptionally — if the `whenComplete` already removed the entry, this second remove is a no-op.
  - This pattern is the basis for cache deduplication in libraries like Caffeine and Spring's `@Cacheable`.

> **Interview follow-up:** The candidate used `cache.remove(key, future)` with the identity check. If two callers request the same key simultaneously and the first computation fails, the second caller receives a failed future, calls `get()`, gets `ExecutionException`, and then retries. Could the second caller's retry race with the third caller's concurrent request for the same key, causing duplicate computation? How would you prevent this?

---

**Q: A background job processes 1M records using a fixed thread pool. Each record processing holds a database connection. The pool has 10 threads and the connection pool has 10 connections. Occasionally, the system deadlocks — threads are waiting for connections, but connections are held by threads waiting for the queue to process more records. How do you diagnose and fix this thread pool deadlock?**

- **Solution**
  - This is a classic thread starvation deadlock where parent tasks submit subtasks that require the same thread pool, exhausting all threads because parents are blocked waiting for children's futures:

```java
// Deadlock scenario
executor.submit(() -> {
    List<Future<Result>> futures = data.stream()
        .map(item -> executor.submit(() -> processWithDb(item))) // children need threads too
        .toList();
    futures.forEach(f -> f.get()); // blocks the parent thread
});
```

  - The fix involves several strategies:
    - Use `CompletableFuture` with asynchronous chaining to avoid blocking `get()` calls.
    - Use separate thread pools for parent and child tasks so they never compete.
    - Use `ForkJoinPool` with its work-stealing and compensating thread mechanism where blocked tasks are automatically detected and compensated with new threads.
  - The debugging technique is always a thread dump — look for threads in `WAITING` state with stack traces showing `Future.get()` or `LinkedBlockingQueue.put()` as the blocking point.

> **Interview follow-up:** The candidate mentioned using `CompletableFuture` to avoid blocking. If the parent task spawns 1000 child subtasks via `CompletableFuture.allOf()`, all 1000 children execute concurrently on the same thread pool. How would you limit the concurrency of child subtasks to avoid overwhelming downstream systems like the database connection pool?

---

**Q: A data pipeline reads events from Kafka (1K events/s), enriches them with data from a REST API (50ms per call), and writes to Elasticsearch. The current implementation processes events sequentially — 50s latency. How do you parallelize while preserving per-partition ordering?**

- **Solution**
  - Use a striped executor where events from the same Kafka partition are always routed to the same single-threaded executor, preserving per-partition ordering while allowing different partitions to be processed in parallel:

```java
public class PartitionedProcessor {
    private final ExecutorService[] executors;

    public void process(ConsumerRecord<String, byte[]> record) {
        int partition = record.partition();
        executors[partition % executors.length].submit(() -> {
            EnrichedEvent enriched = restClient.enrich(record);
            elasticsearch.index(enriched);
        });
    }
}
```

  - Events from Kafka partition 0 always go to executor 0, partition 1 to executor 1, and so on.
  - Each executor is a `newSingleThreadExecutor()` so tasks within a partition are processed sequentially, preserving Kafka's per-partition ordering guarantee.
  - With the number of executors matching the number of partitions (typically 10-50), throughput scales linearly — from 20 events/sec to 200+ events/sec — while the ordering guarantee is maintained within each partition.
  - Kafka's consumer groups handle the rebalancing when partitions are reassigned.

> **Interview follow-up:** The candidate used one single-threaded executor per partition. During a Kafka rebalance, partitions are reassigned to different consumers, and the partition-to-executor mapping changes. The new consumer may process events from a partition while the old consumer still has in-flight events in its executor's queue, causing out-of-order processing. How would you handle this reordering during rebalancing?

---

**Q: A thread-safe counter using `synchronized` is the bottleneck in a high-throughput system. You try replacing it with `AtomicInteger`, but under contention the CAS spin-loop burns CPU. How do you design a counter that is both fast under low contention and doesn't burn CPU under high contention?**

- **Solution**
  - Use `LongAdder` (Java 8+) which employs cell striping — under low contention it behaves like a single `AtomicLong` with one CAS, but under high contention it expands to multiple cells where each thread updates a randomly chosen cell with no CAS retries:

```java
// LongAdder — striped counters, no spin-loop
private final LongAdder count = new LongAdder();
count.increment(); // fast — updates a random cell
// reading:
long total = count.sum(); // slower — sums all cells
```

  - The key insight is that `AtomicLong` causes cache line bouncing — every CAS invalidates the cache line on all other cores, causing the core owning the line to write it back.
  - With `LongAdder`, each thread writes to a different cell on a different cache line, eliminating the invalidation traffic.
  - The trade-off is that `sum()` is O(n) where n is the number of cells (up to the number of CPU cores, typically powers of two).
  - Use `LongAdder` for write-heavy counters (metrics, request counts, statistics) and `AtomicLong` for read-heavy or latency-sensitive counters.

> **Interview follow-up:** The candidate recommended `LongAdder` for write-heavy counters. If the `sum()` is called on every request to report the current count (e.g., a live dashboard showing concurrent connections), the O(n) cell accumulation adds latency to every request. How would you design a counter that supports both high-write throughput and fast reads?

---

**Q: A web server uses `ThreadLocal` to store request context (user ID, trace ID). A thread pool reuses threads for multiple requests. After the first request completes, the second request sees the first request's context. How do you prevent this context leak?**

- **Solution**
  - Clear the `ThreadLocal` in a `finally` block using a servlet filter or middleware that ensures cleanup regardless of whether the request succeeds or fails:

```java
public class ContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        try {
            RequestContext.set(request);
            chain.doFilter(request, response);
        } finally {
            RequestContext.clear(); // always clean up
        }
    }
}
```

  - Without explicit cleanup, the `ThreadLocal` value persists on the thread after the request completes, and when the thread pool reuses that thread for the next request, the old context is visible.
  - The same issue applies to SLF4J's MDC (Mapped Diagnostic Context).
  - For asynchronous processing where callbacks execute on different threads, the context must be captured before the async call and restored in the callback — use a library like `TransmittableThreadLocal` (Alibaba) that automatically propagates context across executor and `CompletableFuture` boundaries.

> **Interview follow-up:** The candidate recommended a servlet filter for cleanup. If the application also uses `@Async` Spring methods that execute on a different thread, the `ThreadLocal` context from the original HTTP request is lost in the async method. The MDC context for logging would show no trace ID in the async task's log output. How would you propagate the trace ID to the async thread without modifying every `@Async` method?

---

**Q: A microservice has a health check endpoint that calls downstream services. Each downstream call is 100ms. Under normal load, the health check returns in 100ms. Under load, health checks pile up — 50 concurrent health checks times 100ms equals 5 seconds response time. How do you ensure health checks remain fast regardless of load?**

- **Solution**
  - Cache the health status with a short TTL and use a single background thread for the actual health check, so the health endpoint always reads a `volatile` field with zero latency:

```java
public class HealthChecker {
    private volatile HealthStatus cached = HealthStatus.UNKNOWN;
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

    @PostConstruct
    public void start() {
        scheduler.scheduleAtFixedRate(this::check, 0, 5, SECONDS);
    }

    private void check() {
        HealthStatus status = checkDownstream(); // takes ~100ms
        cached = status;
        alertIfDown(status);
    }

    public HealthStatus getHealth() {
        return cached; // never blocks, always fast
    }
}
```

  - All health check requests read the `volatile cached` field — an O(1) memory read that returns in microseconds regardless of how many concurrent callers exist.
  - A single background thread performs the actual downstream health checks every 5 seconds, updating the cached value.
  - This pattern — observed-replicated state — sacrifices 5 seconds of staleness for O(1) response time, which is the correct trade-off for load balancer health checks that typically use 10-30 second intervals.

> **Interview follow-up:** The candidate cached health status with a 5-second TTL. If a critical downstream service goes down, the health check endpoint continues reporting "healthy" for up to 5 seconds, directing traffic to an unhealthy node. The load balancer's health check interval is 10 seconds — matching the cache TTL means up to 15 seconds of traffic to a dead node. How would you minimize this detection lag while still keeping the health check endpoint fast?

---

**Q: A service uses `synchronized` blocks that are nested — `synchronized(A) { synchronized(B) { ... } }`. Another method has `synchronized(B) { synchronized(A) { ... } }`. Occasionally the service hangs. What is happening and how do you fix it?**

- **Solution**
  - This is a classic deadlock — thread 1 holds lock A and waits for lock B; thread 2 holds lock B and waits for lock A — a cycle that can never resolve:

```java
// Thread 1                         // Thread 2
synchronized(lockA) {               synchronized(lockB) {
    synchronized(lockB) {               synchronized(lockA) {
        // ...                           // ...
    }                               }
}                                   }
```

  - Three fixes exist:
    - Enforce consistent lock ordering across the entire codebase (always acquire A then B).
    - Use `ReentrantLock.tryLock()` with a timeout so the lock acquisition backs off rather than blocking forever.
    - Use higher-level abstractions like `StampedLock` or `Phaser` that eliminate the need for manual lock ordering.
  - For prevention, use `ThreadMXBean.findDeadlockedThreads()` in automated tests to detect lock cycles before they reach production.

> **Interview follow-up:** The candidate suggested `tryLock()` with a timeout to avoid deadlock. If both threads back off when they cannot acquire the second lock, they release the first lock and retry — this avoids deadlock but can cause livelock where both threads keep retrying in lockstep. How would you add jitter or exponential backoff to the retry logic to break the livelock cycle?

**Q: A real-time trading system processes buy/sell orders using multiple threads that access a shared order book. Under peak load, traders report that their orders are taking 10x longer to execute even though the system is nowhere near its theoretical capacity. Thread dumps show many threads in `BLOCKED` state on `ReentrantLock` associated with the order book's bid-ask spread calculation. How would you diagnose and resolve this contention?**
  - The thread dumps reveal `BLOCKED` state — threads are not doing useful work; they are waiting to acquire the spread calculation lock.
  - The spread calculation is a read-mostly operation (it only changes when market conditions shift, which is far less frequent than order placement).
  - Solution: replace `ReentrantLock` with `ReadWriteLock` or `StampedLock`. The spread calculation acquires a read lock (shared), while order mutations acquire a write lock (exclusive).
  - Even better: use `StampedLock.tryOptimisticRead()` for the spread calculation — it never blocks writers and only validates the stamp after reading. If a write occurred during the read, the stamp invalidates and the read retries with a full read lock.
  - Profile with JFR before and after — expected improvement: 5-10x throughput under contention.

---

## Interview Questions

- **What is the Java Memory Model (JMM) and why is it important?**
  - The JMM defines the formal rules for how threads interact through memory and when one thread's changes are visible to others.
  - It establishes happens-before relationships including program order, monitor lock unlock before subsequent lock, volatile write before subsequent read, thread start, thread join, and transitivity.
  - Without the JMM, compiler and CPU reorderings would make unsynchronized concurrent access unpredictable and platform-dependent.

- **What is the difference between `synchronized` and `ReentrantLock`?**
  - `synchronized` is simpler — it automatically releases the lock when the block exits and is JVM-optimized with biased locking.
  - `ReentrantLock` offers advanced features: `tryLock()` with timeout, fair/unfair modes, `Condition` for multiple wait sets, interruptible locking, and the ability to query lock state.
  - Performance is similar.
  - Use `synchronized` by default; use `ReentrantLock` when you need timeout-based locking, fairness guarantees, or multiple condition variables.

- **What is `volatile` and what does it guarantee?**
  - `volatile` guarantees visibility: every write to a `volatile` field happens-before every subsequent read of that field across all threads, preventing CPU caching and instruction reordering.
  - It does NOT guarantee atomicity — `volatile int count; count++` is still a read-modify-write race condition.
  - Use `volatile` for flags and status variables. For atomic counters, use `AtomicInteger` or `LongAdder`.

- **What is CAS (Compare-And-Swap) and how does it work?**
  - CAS is a CPU-level instruction that atomically reads a memory location, compares it with an expected value, and writes a new value only if the comparison succeeds.
  - It returns a boolean indicating success.
  - Java exposes CAS through `AtomicInteger`, `AtomicLong`, `AtomicReference`, and `VarHandle` (Java 9+).
  - CAS is lock-free — it never parks the OS thread — but under high contention, repeated CAS failures cause spin-loop CPU usage.

- **What is the difference between `CountDownLatch` and `CyclicBarrier`?**
  - `CountDownLatch` is one-shot — it counts down from N to zero and cannot be reset. Threads await until the count reaches zero.
  - `CyclicBarrier` is reusable — N threads await until all N arrive, then they all proceed and the barrier automatically resets.
  - Use `CountDownLatch` for one-time synchronization; use `CyclicBarrier` for multi-phase parallel computations.

- **What is a `CompletableFuture` and how does it differ from a `Future`?**
  - `Future` (Java 5) represents an asynchronous result with only a blocking `get()` — no composition, no callbacks.
  - `CompletableFuture` (Java 8) supports non-blocking callbacks (`thenApply`, `thenAccept`), composition (`thenCompose`, `thenCombine`), error handling (`exceptionally`, `handle`), timeouts (`orTimeout`), and combining multiple futures (`allOf`, `anyOf`).
  - Use `CompletableFuture` for async pipelines; use `Future` only when interfacing with legacy `ExecutorService.submit()`.

- **What is thread starvation and how do you prevent it?**
  - Thread starvation occurs when a thread never gets CPU time to make progress, typically from priority inversion, unfair lock acquisition, or bounded thread pools that cannot accommodate all submitter threads.
  - Prevention strategies include using fair locks (`new ReentrantLock(true)`), avoiding thread priority changes, sizing thread pools appropriately with profiling, using `ForkJoinPool` for work-stealing, and adding timeouts to all blocking operations.

- **What is the ABA problem in CAS and how do you solve it?**
  - The ABA problem occurs when a value changes from A to B and back to A between a CAS read and write — CAS sees the expected A, succeeds, but the data structure may be in an unexpected state.
  - This is critical in lock-free data structures like stacks.
  - Solutions: `AtomicStampedReference` pairs the reference with an integer stamp incremented on every write, and `AtomicMarkableReference` pairs it with a boolean mark.

- **What is the difference between `submit()` and `execute()` in ExecutorService?**
  - `execute(Runnable)` returns void — fire-and-forget with no result or exception tracking.
  - `submit(Runnable)` returns `Future<?>` allowing completion checking and exception capture via `future.get()`.
  - `submit(Callable<T>)` returns `Future<T>` with the callable's return value.
  - Exceptions from `execute()` go to the uncaught exception handler; exceptions from `submit()` are stored in the `Future`.
  - Use `execute()` for fire-and-forget; use `submit()` when you need the result or error handling.

- **What is the difference between `shutdown()` and `shutdownNow()` in ExecutorService?**
  - `shutdown()` prevents new task submission but allows already-submitted tasks to complete normally.
  - `shutdownNow()` interrupts actively executing tasks and returns the list of queued (not yet started) tasks.
  - The best practice pattern is: `shutdown()`, then `awaitTermination(timeout)`, then if not terminated, `shutdownNow()`, then `awaitTermination(timeout)` again for the forced shutdown.

- **What is a `Phaser` and when would you use it?**
  - `Phaser` is a reusable synchronization barrier similar to `CyclicBarrier` and `CountDownLatch`, but supports dynamic party registration and deregistration.
  - Parties can register mid-execution via `register()` or `bulkRegister()`, and arrive without waiting via `arrive()`.
  - Use for multi-phase computations where the number of participating threads changes over time — e.g., a staged data pipeline where some workers join only for specific phases.

- **What is `Exchanger<V>` and what is its use case?**
  - `Exchanger<V>` provides a synchronization point where two threads can exchange objects — each thread calls `exchange(V)` and blocks until the other thread arrives, receiving the other's object.
  - Use for pairwise data exchange patterns like a producer-consumer with buffers: one thread hands off a filled buffer and receives an empty one.
  - The exchange is atomic — both threads must arrive for either to proceed.

- **What is the difference between `Runnable` and `Callable` in a concurrency context?**
  - `Runnable.run()` returns `void` and cannot throw checked exceptions; its failures propagate to `UncaughtExceptionHandler` if submitted via `execute()`.
  - `Callable.call()` returns a value and can throw checked exceptions; failures are captured in the `Future` and surfaced when `get()` is called.
  - `Runnable` is for fire-and-forget tasks; `Callable` for tasks where the caller needs the result or exception.

- **What is `ForkJoinPool` and how does work stealing work?**
  - `ForkJoinPool` is a specialized thread pool for divide-and-conquer algorithms, where tasks can recursively fork subtasks and join their results.
  - Work stealing: each worker thread has a double-ended queue (deque) of tasks. A thread that exhausts its own deque can "steal" work from the tail of another thread's deque.
  - This minimizes contention — the owner pushes/pops from the head, while stealers take from the tail.
  - Use `ForkJoinPool` for recursive parallel tasks like merge sort, matrix multiplication, and tree traversal.

- **What is `CompletableFuture.allOf()` and `CompletableFuture.anyOf()`?**
  - `allOf()` returns a `CompletableFuture<Void>` that completes when all provided futures complete. Use for parallel fan-out where you need all subtasks to finish.
  - `anyOf()` returns a `CompletableFuture<Object>` that completes when any one of the provided futures completes. Use for race patterns (e.g., query multiple replicas, take the fastest response).
  - Neither `allOf()` nor `anyOf()` propagate results — you must combine them manually with `thenApply()` or `thenAccept()`.

- **What is `ThreadLocalRandom` and why use it over `Random` in multithreaded code?**
  - `ThreadLocalRandom` provides per-thread instances of `Random`, eliminating contention on the shared seed update.
  - `Random` uses an `AtomicLong` seed — under high concurrency, CAS failures cause retries and cache line bouncing.
  - `ThreadLocalRandom` is up to 10x faster than `Random` under contention. Use `ThreadLocalRandom.current().nextInt()`.

- **What is the concept of happens-before in the JMM?**
  - Happens-before is a partial ordering of operations that guarantees memory visibility — if operation X happens-before Y, then X's effects are visible to Y.
  - Sources of happens-before: program order (within a thread), monitor unlock before next lock on same monitor, `volatile` write before subsequent read, `Thread.start()` before the started thread's actions, `Thread.join()` after the joined thread's actions, and transitivity.
  - Without happens-before, compiler reordering and CPU out-of-order execution can produce surprising results.

- **What is the difference between `notify()` and `notifyAll()`?**
  - `notify()` wakes up a single arbitrarily chosen thread waiting on the monitor — efficient but can cause missed signals if the chosen thread is not the right one.
  - `notifyAll()` wakes up all waiting threads — safer but may cause unnecessary context switching as all threads contend for the lock.
  - Use `notifyAll()` unless you can prove that only one thread type waits on the condition and any thread can handle the notification.

- **What is the role of `ThreadMXBean` in diagnosing concurrency issues?**
  - `ThreadMXBean` (via `ManagementFactory.getThreadMXBean()`) provides thread CPU time, thread contention monitoring, and deadlock detection.
  - `findDeadlockedThreads()` detects cycles in lock ownership, including `ReentrantLock` and other `java.util.concurrent` locks (not just `synchronized`).
  - `findMonitorDeadlockedThreads()` only finds `synchronized`-based deadlocks.
  - Use in automated tests — if `findDeadlockedThreads()` returns non-null, the test fails immediately.

---

## Developer Recommendations

- **Prefer `synchronized` over `ReentrantLock`** unless you specifically need advanced features like `tryLock()` with timeout, fairness, `Condition` variables, or interruptible locking.
  - `synchronized` is simpler (no manual unlock required), JVM-optimized with biased locking and lock coarsening, and less error-prone — you cannot forget to unlock.
  - For 90% of concurrency needs, `synchronized` is the correct choice.
  - A team once replaced all `synchronized` blocks with `ReentrantLock` based on a blog post about performance, but missed `unlock()` in one `catch` block — the resulting lock leak caused a production outage where database update threads silently stopped after the first exception, taking 3 hours to diagnose.

- **Use `LongAdder` over `AtomicLong` for write-heavy counters** where the sum is read infrequently.
  - Under high contention, `AtomicLong` causes cache line bouncing as each CAS invalidates the line on all other cores.
  - `LongAdder` distributes updates across multiple cells with zero contention — each thread updates a random cell.
  - This makes writes scale with the number of cores.
  - Use `AtomicLong` for read-heavy counters where `sum()` is called frequently, because `LongAdder.sum()` must accumulate all cells and is O(n).

- **Always use `ConcurrentHashMap` for maps accessed by multiple threads.**
  - A `HashMap` used concurrently can cause infinite loops during resize (JDK 7), data loss, and `ConcurrentModificationException`.
  - `ConcurrentHashMap` provides lock-free reads and per-bucket synchronization for writes.
  - Use `computeIfAbsent()` for atomic lazy initialization — the mapping function runs at most once per key even with thousands of concurrent callers.
  - Avoid `putIfAbsent()` followed by `get()` which is not atomic.
  - A production incident involved a JDK 7 `HashMap` used as a cache in a multi-threaded server — during a resize under concurrent `put()` calls, the `HashMap` entered an infinite loop in `transfer()`, consuming 100% CPU on all cores and requiring a full cluster restart.

- **Use `Semaphore` to limit access to bounded resources** like database connections, file handles, or rate-limited API calls.
  - Unlike thread pools which limit threads, semaphores limit concurrent operations regardless of thread count.
  - Always call `release()` in a `finally` block to prevent permit leaks.

- **Always clear `ThreadLocal` values in `finally` blocks** when using thread pools because threads are reused.
  - A `ThreadLocal` set during one request persists for the next request on the same thread, causing context leakage that is extremely hard to debug.
  - The pattern is `try { ... } finally { contextHolder.remove(); }`.

- **Add timeouts to all blocking operations** — `future.get()`, `queue.take()`, `latch.await()`, `lock.tryLock()`, and `completableFuture.join()` can all block indefinitely.
  - Always use the timeout overload: `future.get(5, SECONDS)`.
  - For `CompletableFuture`, use `.orTimeout(5, SECONDS)`.
  - When a timeout expires, clean up by cancelling futures and interrupting threads.

- **Use `StampedLock` for read-mostly workloads** where `ReadWriteLock` causes writer starvation.
  - `StampedLock` with `tryOptimisticRead()` never blocks writers — if a writer arrives during an optimistic read, the stamp validation fails and the read falls back to a regular read lock.
  - This gives superior throughput for read-heavy data structures like configuration registries and lookup tables.

- **Profile lock contention before optimizing** — many teams prematurely replace `synchronized` with `ReentrantLock` or `StampedLock` based on intuition rather than data.
  - Use `jstack` thread dumps, async profiler's lock profiling, or JFR (Java Flight Recorder) to measure actual contention.
  - If lock acquisition takes less than 1% of CPU time, optimizing the lock is premature — the performance bottleneck lies elsewhere.
  - Contention optimization should follow a priority order: eliminate shared state → reduce critical section size → use lock striping → use lock-free algorithms → replace locking mechanism.
