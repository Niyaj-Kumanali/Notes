# Java Concurrency

---

## Overview

- **Definition:** Java Concurrency is the set of APIs and language features for managing multiple threads accessing shared resources. It covers synchronization, locks, atomic variables, concurrent collections, and advanced coordination mechanisms in `java.util.concurrent`.

- **Why It Exists:** Without proper concurrency tools, developers had to use low-level `synchronized` and `wait/notify` — error-prone and hard to reason about. The modern concurrency API provides higher-level abstractions, thread-safe collections, efficient synchronization, and lock-free operations.

- **Core Packages:**
  - `java.util.concurrent` — Executors, locks, atomic vars, concurrent collections, synchronizers
  - `java.util.concurrent.atomic` — AtomicInteger, AtomicReference, LongAdder
  - `java.util.concurrent.locks` — ReentrantLock, ReadWriteLock, StampedLock, Condition

---

## The Java Memory Model (JMM)

- **Definition:** The JMM defines how threads interact through memory and when changes by one thread are visible to others. It's the formal specification that guarantees thread safety when using proper synchronization.

- **Key Happens-Before Rules:**
  - **Program order** — within a thread, actions are ordered as in source code
  - **Monitor lock** — unlock on a monitor happens-before every subsequent lock on that monitor
  - **Volatile** — write to a volatile field happens-before every subsequent read of that field
  - **Thread start** — `Thread.start()` happens-before any action in the started thread
  - **Thread join** — all actions in a thread happen-before `Thread.join()` returns
  - **Transitivity** — if A happens-before B and B happens-before C, then A happens-before C

---

## Synchronization Mechanisms

### synchronized

- **Definition:** The intrinsic lock mechanism built into every Java object. Simple and JVM-optimized.

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

- **Monitor Evolution (JDK 6+):**
  - **No contention:** Biased locking (thread ID in object header)
  - **Light contention:** Lightweight locking (CAS on object header)
  - **Heavy contention:** Heavyweight locking (OS mutex — thread blocks)
  - Upgrade is one-way: biased → lightweight → heavyweight

### volatile

- **Definition:** Ensures visibility of changes across threads. Writes to a volatile variable flush to main memory; reads come from main memory.

```java
private volatile boolean running = true;
```

- **Guarantees:** Visibility only, NOT atomicity. `volatile int count; count++` is still a read-modify-write race condition.

### ReentrantLock

- **Definition:** A more flexible alternative to `synchronized` with features like tryLock, fairness, and Condition support.

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

- **Features:** `tryLock(timeout)`, fair/unfair fairness, `Condition` for wait/notify patterns.

### ReadWriteLock

- **Definition:** Allows multiple concurrent readers but exclusive writer access. Best for read-heavy, write-rare scenarios.

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

- **Definition:** Supports optimistic reads that are lock-free. Falls back to read lock if optimistic read fails due to a concurrent write.

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

- **Definition:** Lock-free, thread-safe variables that use Compare-And-Swap (CAS) CPU instructions.

```java
private final AtomicInteger counter = new AtomicInteger(0);

public int incrementAndGet() {
    return counter.incrementAndGet();  // CAS-based, no lock
}
```

- **How CAS Works:**
  1. Read memory location
  2. Compare with expected value
  3. If match, write new value
  4. Return success/failure

- **ABA Problem:** Value changes A→B→A, CAS doesn't detect change. Solution: `AtomicStampedReference`.

- **LongAdder (Java 8+):** Better than AtomicInteger for high contention. Uses striped counters — multiple cells, update one at random, sum all on read.

```java
private final LongAdder requests = new LongAdder();
public void recordRequest() { requests.increment(); }
public long getRequestCount() { return requests.sum(); }
```

---

## Concurrent Collections

- **Definition:** Thread-safe collections that provide better performance than synchronized wrappers.

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

- **Definition:** Higher-level coordination primitives for thread synchronization.

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

- **Definition:** A future that can be manually completed and composed with callbacks for asynchronous programming.

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

- **Double-checked locking without volatile** — publication race. Use `volatile` or holder class pattern
- **synchronized(this) scope too coarse** — low concurrency. Narrow scope or use ConcurrentHashMap
- **Forgetting unlock in finally** — lock leak. Always try/finally unlock
- **Busy-waiting with while(true)** — 100% CPU. Use wait/notify, blocking queue, or LockSupport.park()
- **String as lock object** — string interning causes false sharing. Use `new Object()` as lock
- **Not handling InterruptedException** — lost interrupt signal. Restore interrupt flag: `Thread.currentThread().interrupt()`
- **Deadlock from nested synchronized** — threads block forever. Use consistent lock ordering
- **Not sizing thread pool** — thread starvation or OOM. Profile and size appropriately

---

## Real-World Scenarios

### Scenario 1: Real-Time Order Matching Engine

A stock exchange matches buy and sell orders. Thousands of orders arrive per second from multiple trading firms. The matching engine must maintain an order book per stock, match orders instantly, and publish trades — all with strict fairness guarantees.

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

`ConcurrentSkipListMap` provides thread-safe sorted access — O(log n) for insertion and query. `ConcurrentNavigableMap.headMap()` returns a view of entries with price ≤ the order price. Each price level uses a `ConcurrentLinkedQueue` for FIFO ordering among same-price orders. The per-bucket design of `ConcurrentHashMap` would not work here because we need sorted access by price. The skip-list's lock-free implementation allows multiple matching threads to operate on different price levels simultaneously.

### Scenario 2: Distributed Tracing with CompletableFuture

A microservice receives a request and must fan out to 5 downstream services (user profile, order history, recommendations, inventory, pricing). The response aggregates data from all services. If any service fails, the entire response must use cached fallback data within 200ms.

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

`CompletableFuture.allOf()` combines three independent async calls. `.applyToEither()` with `timeoutAfter()` implements a race — if the 200ms deadline arrives before all services respond, we use fallback data. Each individual call has a 150ms timeout to prevent slow services from consuming the entire budget. The `executor` with 20 threads handles the thread fan-out without using the common ForkJoinPool.

### Scenario 3: Database Migration with Phaser

A schema migration tool must run 8 migration scripts in parallel across 4 shards. After all migrations complete, it should validate constraints and then switch traffic to the new schema. The tool uses a `Phaser` to coordinate the phases.

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

`Phaser` is a reusable barrier that supports dynamic party registration. Each shard is a party that registers, runs through 3 phases (one per script), and deregisters. The main thread also participates, advancing through phases after each script. The key advantage over `CountDownLatch` and `CyclicBarrier`: `Phaser` supports an arbitrary number of parties that can change dynamically, and it's reusable across phases without reset.

---

## Scenario-Based Questions

1. **Q: You are designing a real-time chat server that handles 100K concurrent connections. Each user can be in multiple chat rooms. Messages must be delivered to all members of a room within 100ms. The server runs on 8 cores. How do you structure the concurrency?**
   A: Use the reactor pattern with a small number of event loop threads (one per core) and non-blocking I/O. Each room's state is single-threaded — all operations on a room are handled by the same event loop thread to avoid locking:
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
   Each room is pinned to a specific event loop thread — no locks needed for room state. `ConcurrentHashMap` for room-to-thread mapping handles concurrent room creation. The event loop threads handle I/O (Netty/NIO) which scales to 100K connections with 8 threads. Thread-per-connection would require 100K threads and collapse under the memory overhead (1GB+ just for thread stacks).

2. **Q: A leader-election system uses ZooKeeper/Etcd to pick one node as the leader. When the leader fails, another node must take over. During the handoff, two nodes briefly believe they are both leaders. How do you prevent this split-brain scenario?**
   A: Use a fencing token — a monotonically increasing epoch number that the leader must include in all writes. When a new leader is elected, it increments the epoch. Any write from the old leader (with a lower epoch) is rejected:
   ```java
   public class FencedLeader {
       private volatile int epoch;

       public boolean executeWithFence(Runnable action) {
           int currentEpoch = this.epoch;
           return storage.writeWithEpoch(currentEpoch, action); // storage rejects stale epochs
       }

       public void onElected() {
           this.epoch = storage.incrementAndGetEpoch(); // atomically increment
       }
   }
   ```
   The lock-based approach (`synchronized` or `ReentrantLock`) prevents interleaving within a JVM but doesn't help across JVMs. The fencing token is the only reliable way to prevent split-brain. Combined with a lease (leader must renew every N seconds), the window of double-leadership is limited to the lease duration.

3. **Q: A payment processing system uses `synchronized` blocks to ensure idempotency — only one thread processes a given payment ID. Under load, throughput drops and threads pile up waiting for the lock. How do you scale beyond a single-threaded bottleneck?**
   A: Replace the single monolithic lock with striped locking — one lock per payment ID hash bucket:
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
   `Striped.lock(1024)` creates 1024 locks. Payments with the same ID hash to the same lock (ensuring per-ID serialization), but different IDs use different locks (allowing parallelism up to 1024 concurrent processors). This is the idea behind sharded locking — used by `ConcurrentHashMap` internally. The stripe count should be a power of 2 and tuned so that lock contention is <5%.

4. **Q: A caching system stores computed results in a `ConcurrentHashMap<K, CompletableFuture<V>>`. Multiple threads may request the same key simultaneously. Only one should compute; others should wait for the result. The computation may fail — future callers should retry, not get the failed result. How do you handle this?**
   A: Use `computeIfAbsent()` with atomic removal on failure:
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
   `computeIfAbsent` guarantees that only the first caller creates the future. Other callers get the same future. On failure, `whenComplete` removes the entry so future callers retry. The `cache.remove(key, future)` in the catch block handles the edge case where the future completed exceptionally and was already removed — the second `remove` is a no-op. This pattern is commonly used in libraries like Caffeine and Spring's `Cacheable`.

5. **Q: A background job processes 1M records using a fixed thread pool. Each record processing holds a database connection. The pool has 10 threads and the connection pool has 10 connections. Occasionally, the system deadlocks — threads are waiting for connections, but connections are held by threads waiting for the queue to process more records. How do you diagnose and fix this thread pool deadlock?**
   A: This is a classic thread starvation deadlock. The task spawns subtasks that also need thread pool threads, but all threads are blocked waiting for subtasks that can't run:
   ```java
   // Deadlock scenario
   executor.submit(() -> {
       List<Future<Result>> futures = data.stream()
           .map(item -> executor.submit(() -> processWithDb(item))) // children need threads too
           .toList();
       futures.forEach(f -> f.get()); // blocks the parent thread
   });
   ```
   Fixes: (1) Use `CompletableFuture` with async chaining instead of blocking `get()`: `CompletableFuture.supplyAsync(() -> process(data), executor).thenComposeAsync(...)`. (2) Use a larger pool or separate pools for parent and child tasks. (3) Use `ForkJoinPool` which uses work-stealing — blocked tasks are automatically compensated. The debugging technique: take a thread dump and look for threads in `WAITING` state (parking) waiting on `Future.get()` or `LinkedBlockingQueue.put()`.

6. **Q: A data pipeline reads events from Kafka (1K events/s), enriches them with data from a REST API (50ms per call), and writes to Elasticsearch. The current implementation processes events sequentially — 50s latency. How do you parallelize while preserving per-partition ordering (Kafka guarantee)?**
   A: Use a `Striped` executor that ensures events from the same partition are processed by the same thread:
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
   Events from partition 0 always go to executor 0, partition 1 to executor 1, etc. Within each executor, tasks are processed sequentially (single-threaded executor), preserving Kafka's per-partition ordering. With 10 executors, throughput scales 10x (from 20 events/s to 200 events/s). The ordering-per-partition is maintained while different partitions are processed in parallel. Use `Executors.newSingleThreadExecutor()` for each partition group.

7. **Q: A thread-safe counter using `synchronized` is the bottleneck in a high-throughput system. You try replacing it with `AtomicInteger`, but under contention the CAS spin-loop burns CPU. How do you design a counter that is both fast under low contention and doesn't burn CPU under high contention?**
   A: Use `LongAdder` (Java 8+) which uses cell striping:
   ```java
   // AtomicInteger — CAS spin-loop under contention, burns CPU
   private final AtomicInteger count = new AtomicInteger();
   count.incrementAndGet();

   // LongAdder — striped counters, no spin-loop
   private final LongAdder count = new LongAdder();
   count.increment(); // fast — updates a random cell
   // reading:
   long total = count.sum(); // slower — sums all cells
   ```
   Under low contention, `LongAdder` behaves like `AtomicInteger` (single cell). Under high contention, it creates additional cells and distributes updates across them. A thread picks a random cell to update — no CAS retries. The trade-off: `sum()` is O(n) where n is the number of cells (typically powers of 2, max `CPU cores`). Use `LongAdder` for write-heavy counters and `AtomicLong` for read-heavy or low-contention counters.

8. **Q: A web server uses `ThreadLocal` to store request context (user ID, trace ID). A thread pool reuses threads for multiple requests. After the first request completes, the second request sees the first request's context. How do you prevent this context leak?**
   A: Clear the `ThreadLocal` in a finally block or use a servlet filter:
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
   Without cleanup, thread-pool reuse causes context leaking between requests. The same applies to MDC (Mapped Diagnostic Context) in logging frameworks. For async processing, the context must be captured before the async call and restored in the callback. Use a library like `TransmittableThreadLocal` (Alibaba) that automatically propagates context across `CompletableFuture` and executor boundaries.

9. **Q: A microservice has a health check endpoint that calls downstream services. Each downstream call is 100ms. Under normal load, the health check returns in 100ms. Under load, health checks pile up — 50 concurrent health checks × 100ms = 5s response time. How do you ensure health checks remain fast regardless of load?**
   A: Cache the health status with a short TTL and use a single background thread to refresh:
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
   All health check requests read the `volatile cached` field — O(1), non-blocking. A single background thread performs the actual check every 5 seconds. This pattern (observed-replicated state) ensures the health endpoint returns in microseconds regardless of load. The trade-off: health status may be up to 5 seconds stale, which is acceptable for load balancer health checks.

10. **Q: A service uses `synchronized` blocks that are nested — `synchronized(A) { synchronized(B) { ... } }`. Another method has `synchronized(B) { synchronized(A) { ... } }`. Occasionally the service hangs. What is happening and how do you fix it?**
    A: This is a classic deadlock — thread 1 holds A and waits for B; thread 2 holds B and waits for A:
    ```java
    // Thread 1                         // Thread 2
    synchronized(lockA) {               synchronized(lockB) {
        synchronized(lockB) {               synchronized(lockA) {
            // ...                           // ...
        }                               }
    }                                   }
    ```
    Fixes: (1) Always acquire locks in the same global order (A then B): `synchronized(A) { synchronized(B) { } }` in both methods. (2) Use `ReentrantLock.tryLock()` with timeout to detect and recover from deadlock:
    ```java
    if (lockA.tryLock(100, MILLISECONDS)) {
        try {
            if (lockB.tryLock(100, MILLISECONDS)) {
                try { /* critical section */ }
                finally { lockB.unlock(); }
            }
        } finally { lockA.unlock(); }
    }
    ```
    (3) Use higher-level abstractions like `StampedLock` or `Phaser` that avoid manual lock ordering. For prevention, use a lock ordering checker (like `jstack` or `ThreadMXBean.findDeadlockedThreads()`) in CI tests.

---

## Interview Questions

1. **What is the Java Memory Model (JMM) and why is it important?**
   A: The JMM defines how threads interact through memory and when one thread's changes are visible to others. It specifies happens-before rules: synchronized unlock before subsequent lock, volatile write before subsequent read, thread start/join, and transitivity. Without the JMM, compiler and CPU reorderings could make unsynchronized access unpredictable. The JMM guarantees thread safety when proper synchronization is used.

2. **What is the difference between `synchronized` and `ReentrantLock`?**
   A: `synchronized` is simpler — automatic lock release, no manual unlock. `ReentrantLock` offers: `tryLock()` with timeout, fair/unfair fairness, `Condition` for wait/notify, interruptible locking, and ability to check if held. Performance is similar (both are JVM-optimized). Use `synchronized` by default; use `ReentrantLock` when you need timeout, fairness, or multiple condition variables.

3. **What is `volatile` and what does it guarantee?**
   A: `volatile` guarantees visibility — a write to a volatile variable happens-before any subsequent read of that variable. It prevents CPU caching and instruction reordering. `volatile` does NOT guarantee atomicity — `volatile int count; count++` is still a read-modify-write race condition. Use `volatile` for flags and status variables where atomic compound actions aren't needed. For counters, use `AtomicInteger` or `LongAdder`.

4. **What is CAS (Compare-And-Swap) and how does it work?**
   A: CAS is a CPU-level instruction that atomically compares a memory location with an expected value and updates it to a new value if the comparison succeeds. It returns success/failure. Java exposes CAS via `AtomicInteger`, `AtomicReference`, etc. CAS is lock-free — no OS thread blocking. Under high contention, CAS retries (spin-loop) can burn CPU. Java 9+ provides `VarHandle` for direct CAS operations on any field.

5. **What is the difference between `CountDownLatch` and `CyclicBarrier`?**
   A: `CountDownLatch` is one-shot — it counts down to zero and cannot be reset. Threads `await()` until the count reaches zero. `CyclicBarrier` is reusable — N threads `await()` until all N arrive, then they all proceed and the barrier resets. Use `CountDownLatch` for one-time synchronization (e.g., wait for services to start). Use `CyclicBarrier` for multi-phase computations (e.g., iterative algorithm convergence).

6. **What is a `CompletableFuture` and how does it differ from a `Future`?**
   A: `Future` (Java 5) represents an async result with blocking `get()` — no composition, no callbacks. `CompletableFuture` (Java 8) supports: non-blocking callbacks (`thenApply`, `thenAccept`), composition (`thenCompose`, `thenCombine`), error handling (`exceptionally`, `handle`), timeouts (`orTimeout`), and combining multiple futures (`allOf`, `anyOf`). Use `CompletableFuture` for async pipelines; use `Future` only when interfacing with legacy `ExecutorService.submit()`.

7. **What is thread starvation and how do you prevent it?**
   A: Thread starvation occurs when a thread never gets CPU time to make progress. Causes: thread priority misuse, unfair locks, CPU-bound threads monopolizing cores, and bounded thread pools rejecting tasks. Prevention: use fair locks (`new ReentrantLock(true)`), avoid thread priority changes, size thread pools appropriately, use work-stealing (`ForkJoinPool`), and add timeouts to all blocking operations.

8. **What is the ABA problem in CAS and how do you solve it?**
   A: ABA occurs when a value changes A → B → A, and CAS doesn't detect the change (it sees A matches the expected A). This can corrupt data structures like lock-free stacks. Solutions: use `AtomicStampedReference` or `AtomicMarkableReference` which pair the reference with a version stamp. The stamp is incremented on every write, so even if the reference is the same, the stamp differs.

9. **What is the difference between `submit()` and `execute()` in ExecutorService?**
   A: `execute(Runnable)` returns void — fire-and-forget, no result or exception tracking. `submit(Runnable)` returns `Future<?>` — allows checking completion and catching exceptions via `future.get()`. `submit(Callable<T>)` returns `Future<T>` — the callable returns a result. Use `execute()` for fire-and-forget tasks; use `submit()` when you need the result or exception handling. Exceptions from `execute()` go to the uncaught exception handler; exceptions from `submit()` are stored in the Future.

10. **What is the difference between `shutdown()` and `shutdownNow()` in ExecutorService?**
    A: `shutdown()` prevents new tasks from being submitted but allows already-submitted tasks to complete. Returns gracefully. `shutdownNow()` attempts to stop all actively executing tasks (sends `interrupt()`) and returns a list of queued (not yet started) tasks. Use `shutdown()` for normal shutdown; use `shutdownNow()` when you need to stop quickly (e.g., application crash). Best practice: `shutdown()` → `awaitTermination(timeout)` → `shutdownNow()` → `awaitTermination(timeout)`.

---

## Developer Recommendations

- **Prefer `synchronized` over `ReentrantLock` unless you need advanced features** — `synchronized` is simpler (no manual unlock), JVM-optimized (biased locking, lock coarsening), and less error-prone. Use `ReentrantLock` only when you need `tryLock()` with timeout, fairness, `Condition` variables, or interruptible locking. The `synchronized` block is the default choice for 90% of concurrency needs.

- **Use `LongAdder` over `AtomicLong` for high-contention counters** — `AtomicLong` uses CAS which causes cache line bouncing under contention — every thread's CAS invalidates other cores' cache lines. `LongAdder` uses cell striping: each thread updates a random cell with zero contention. Writes scale with the number of cores. Reads (`.sum()`) are slower (sum all cells) — use `AtomicLong` for read-heavy counters and `LongAdder` for write-heavy counters.

- **Always use `ConcurrentHashMap` for shared maps** — `HashMap` in concurrent access causes infinite loops (JDK 7 resize), data loss, and `ConcurrentModificationException`. `ConcurrentHashMap` provides lock-free reads and per-bucket synchronization for writes. Use `computeIfAbsent()` for atomic lazy initialization — the function runs at most once per key even with concurrent callers.

- **Use `Semaphore` to limit access to bounded resources, not thread pools** — A semaphore controls access count: `semaphore.acquire()` before using a resource, `release()` after. Unlike thread pools which limit threads, semaphores limit concurrent operations regardless of thread count. Use semaphores for connection pools, file handles, or rate-limited API calls. Always release in `finally` block to prevent leaks.

- **Always clear `ThreadLocal` in finally blocks when using thread pools** — Thread pool threads are reused. A `ThreadLocal` set during one request persists for the next request on the same thread, causing context leakage. Always clear: `try { ... } finally { contextHolder.remove(); }`. For async callbacks, capture context before submitting and restore in the callback using `CompletableFuture` with context propagation.

- **Add timeouts to all blocking operations** — `future.get()`, `queue.take()`, `latch.await()`, `lock.tryLock()`, `completableFuture.join()` can block indefinitely. Always use the overloaded version with a timeout: `future.get(5, SECONDS)`. For `CompletableFuture`, use `.orTimeout(5, SECONDS)`. If timeout expires, clean up (cancel futures, interrupt threads) to prevent resource leaks.

- **Use `StampedLock` for read-heavy, write-rare scenarios** — `ReentrantReadWriteLock` causes writer starvation under high read load — the writer may wait indefinitely while readers keep acquiring the read lock. `StampedLock` with optimistic reads (`tryOptimisticRead()`) doesn't block writers at all. If a write occurs during an optimistic read, validate returns false and you fall back to a regular read lock. This gives better throughput for read-mostly workloads.
