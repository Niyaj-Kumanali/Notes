# Java Concurrency

---

## 1. Executive Summary

### What Is It?
Java Concurrency is the set of APIs and language features for managing multiple threads accessing shared resources. It covers synchronization, locks, atomic variables, concurrent collections, and advanced coordination mechanisms in `java.util.concurrent` (JSR 166 — Doug Lea's contribution).

### Core java.util.concurrent Packages

| Package | Contents |
|---------|----------|
| `java.util.concurrent` | Executors, locks, atomic vars, concurrent collections, synchronizers |
| `java.util.concurrent.atomic` | AtomicInteger, AtomicReference, LongAdder, etc. |
| `java.util.concurrent.locks` | ReentrantLock, ReadWriteLock, StampedLock, Condition |

### Why Does It Exist?
Without proper concurrency tools, developers had to use low-level `synchronized` and `wait/notify` — error-prone, hard to reason about. The modern concurrency API provides:
- **Higher-level abstractions** — `ExecutorService`, `CompletableFuture`
- **Thread-safe collections** — `ConcurrentHashMap`
- **Efficient synchronization** — `ReentrantLock`, `StampedLock`
- **Lock-free operations** — `AtomicInteger`, `LongAdder`

---

## 2. Core Theory

### The Java Memory Model (JMM)
Defines how threads interact through memory and when changes by one thread are visible to others.

**Key Rules:**
1. **Program order rule** — within a thread, actions are ordered as in source code
2. **Monitor lock rule** — unlock on monitor happens-before every subsequent lock on that monitor
3. **Volatile variable rule** — write to volatile happens-before every subsequent read
4. **Thread start rule** — `Thread.start()` happens-before any action in started thread
5. **Thread join rule** — all actions in thread happen-before `Thread.join()` returns
6. **Transitivity** — if A happens-before B and B happens-before C, then A happens-before C

### Synchronization Mechanisms

#### synchronized
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

**How it works:**
- Each object has a **monitor** (intrinsic lock)
- Thread acquires monitor before entering synchronized block
- **Biased locking** — first thread to acquire lock gets bias (no CAS)
- **Lightweight locking** — if bias fails, CAS on lock word
- **Heavyweight locking** — if CAS fails, OS mutex
- Upgrade is one-way: biased → lightweight → heavyweight

#### volatile
```java
private volatile boolean running = true;
// Write to running → flush to main memory
// Read of running → read from main memory
```

**Guarantees:** visibility only, NOT atomicity.
`volatile int count; count++;` is still a read-modify-write race.

#### ReentrantLock
```java
private final ReentrantLock lock = new ReentrantLock();

public void doWork() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();
    }
}
```

Features: tryLock with timeout, fair/unfair fairness, Condition support.

#### ReadWriteLock
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

Multiple concurrent readers, exclusive writer.

#### StampedLock (Java 8+)
```java
private final StampedLock lock = new StampedLock();

public double getBalance() {
    long stamp = lock.tryOptimisticRead();
    double balance = this.balance;
    if (!lock.validate(stamp)) { // Optimistic read failed
        stamp = lock.readLock();
        try { balance = this.balance; }
        finally { lock.unlockRead(stamp); }
    }
    return balance;
}
```

Optimistic reads are lock-free — great for read-heavy, write-rare scenarios.

### Atomic Variables

```java
private final AtomicInteger counter = new AtomicInteger(0);

public int incrementAndGet() {
    return counter.incrementAndGet(); // CAS-based, no lock
}

// Compare-and-swap
int expected = counter.get();
int next = expected + 1;
counter.compareAndSet(expected, next); // Atomic: only updates if still expected
```

**How CAS works:** CPU instruction `compareAndSwap`:
1. Read memory location
2. Compare with expected value
3. If match, write new value
4. Return success/failure

**ABA Problem:** If value changes from A→B→A, CAS doesn't detect change.
Solution: `AtomicStampedReference` (with version counter).

### LongAdder (Java 8+)
Better than AtomicInteger for high-contention scenarios:
```java
private final LongAdder requests = new LongAdder();

public void recordRequest() { requests.increment(); }
public long getRequestCount() { return requests.sum(); }
```

Internally uses striped counters — multiple cells, update one at random, sum all on read.

### Concurrent Collections

| Collection | Design | Best For |
|-----------|--------|----------|
| `ConcurrentHashMap` | Per-bucket locking (JDK 8) | High-concurrency KV |
| `CopyOnWriteArrayList` | Snapshot array on write | Read-heavy, write-rare |
| `ConcurrentLinkedQueue` | Lock-free (CAS) | High-throughput queue |
| `LinkedBlockingQueue` | ReentrantLock + Conditions | Producer-consumer |
| `ArrayBlockingQueue` | Bounded, array-backed | Bounded producer-consumer |
| `ConcurrentSkipListMap` | Skip list | Sorted concurrent KV |
| `ConcurrentLinkedDeque` | Lock-free deque | Double-ended operations |

### Synchronizers

```java
// CountDownLatch — wait for N operations
CountDownLatch latch = new CountDownLatch(3);
// In thread 1, 2, 3: latch.countDown();
// In main thread: latch.await(); // blocks until count reaches 0

// CyclicBarrier — N threads meet at barrier
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All reached!"));
// barrier.await() in each thread — blocks until all 3 arrive

// Semaphore — control access count
Semaphore permits = new Semaphore(10); // 10 concurrent accesses
permits.acquire(); // blocks if none available
try { /* access resource */ }
finally { permits.release(); }

// Exchanger — exchange data between two threads
Exchanger<Data> exchanger = new Exchanger<>();
Data received = exchanger.exchange(myData); // blocks until other thread calls exchange

// Phaser — flexible barrier (replaces CountDownLatch + CyclicBarrier)
Phaser phaser = new Phaser(3); // 3 parties
phaser.arriveAndAwaitAdvance(); // phase 0 → 1
```

### CompletableFuture (Java 8+)
```java
CompletableFuture<User> future = CompletableFuture
    .supplyAsync(() -> fetchUser(id))         // async execution
    .thenApply(user -> enrichProfile(user))   // transform
    .thenAccept(user -> cache.put(id, user))  // consume
    .exceptionally(ex -> {
        log.error("Failed", ex);
        return User.getDefault();
    }); // error handling

// Combine two async operations
CompletableFuture<Order> orderFuture = CompletableFuture
    .supplyAsync(() -> fetchOrder(orderId))
    .thenCombine(
        CompletableFuture.supplyAsync(() -> fetchPayments(orderId)),
        (order, payments) -> order.applyPayments(payments)
    );

// Wait for all
CompletableFuture.allOf(f1, f2, f3).join();

// Wait for any
CompletableFuture.anyOf(f1, f2, f3).join();
```

---

## 3. Under-the-Hood Deep Dive

### synchronized Monitor Evolution (JDK 6+)
```
No contention:  Biased Locking (biased thread stores thread ID in object header)
    ↓ (bias revoked by another thread)
Light contention: Lightweight Locking (CAS on object header)
    ↓ (CAS fails)
Heavy contention: Heavyweight Locking (OS mutex — thread blocks)
```

### Lock Performance Comparison

| Mechanism | Contention | Throughput | Memory | Notes |
|-----------|-----------|-----------|--------|-------|
| `synchronized` | Low | Best (biased) | ~0 (object header) | JVM-optimized |
| `synchronized` | High | Moderate | ~0 | Inflates to OS mutex |
| `ReentrantLock` | Low | Good | ~32 bytes | More features |
| `ReentrantLock` | High | Good | ~32 bytes | Fairness option |
| `StampedLock` | Read-heavy | Best | ~40 bytes | Optimistic read |
| `AtomicInteger` | Low | Excellent | ~16 bytes | CAS, non-blocking |
| `LongAdder` | High contention | Best | Multiple cells | Striped counters |

### False Sharing
CPU cache lines (~64 bytes). If two atomic vars share a cache line, updating one invalidates the other on other cores — 10-100x slowdown.

**Solution:** Padding (add unused fields to 64 bytes) or `@Contended` annotation (Java 8+):
```java
@sun.misc.Contended
class PaddedCounter {
    volatile long count;
}
```

---

## 4. Production Code Examples

### 4.1 Thread-Safe Singleton with Double-Checked Locking

```java
public class ConfigManager {
    private static volatile ConfigManager instance;
    
    private ConfigManager() {}
    
    public static ConfigManager getInstance() {
        if (instance == null) {
            synchronized (ConfigManager.class) {
                if (instance == null) {
                    instance = new ConfigManager();
                }
            }
        }
        return instance;
    }
}
```

### 4.2 Producer-Consumer with BlockingQueue

```java
@Component
public class OrderProcessingPipeline {
    private final BlockingQueue<Order> queue = new LinkedBlockingQueue<>(1000);
    
    // Producer
    public void submitOrder(Order order) throws InterruptedException {
        queue.put(order); // Blocks if queue full (backpressure)
    }
    
    // Consumer (runs in dedicated thread)
    @PostConstruct
    public void startConsumer() {
        Executors.newSingleThreadExecutor().submit(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    Order order = queue.take(); // Blocks if queue empty
                    processOrder(order);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                } catch (Exception e) {
                    log.error("Failed to process order", e);
                }
            }
        });
    }
    
    private void processOrder(Order order) {
        // Business logic
    }
}
```

### 4.3 CompletableFuture — Parallel API Calls

```java
@Service
public class AggregatedDataService {
    
    public AggregatedResponse getAggregatedData(String userId) {
        CompletableFuture<UserProfile> profileFuture = 
            CompletableFuture.supplyAsync(() -> profileService.getProfile(userId));
        
        CompletableFuture<List<Order>> ordersFuture = 
            CompletableFuture.supplyAsync(() -> orderService.getOrders(userId));
        
        CompletableFuture<Analytics> analyticsFuture = 
            CompletableFuture.supplyAsync(() -> analyticsService.getMetrics(userId));
        
        CompletableFuture<Void> all = CompletableFuture.allOf(
            profileFuture, ordersFuture, analyticsFuture);
        
        return all.thenApply(v -> new AggregatedResponse(
            profileFuture.join(),
            ordersFuture.join(),
            analyticsFuture.join()
        )).exceptionally(ex -> {
            log.error("Failed to aggregate data for user {}", userId, ex);
            return AggregatedResponse.fallback(userId);
        }).join();
    }
}
```

### 4.4 StampedLock — Read-Optimized Cache

```java
public class OptimisticCache<K, V> {
    private final StampedLock lock = new StampedLock();
    private final Map<K, V> cache = new HashMap<>();
    
    public V get(K key) {
        long stamp = lock.tryOptimisticRead();
        V value = cache.get(key);
        if (!lock.validate(stamp)) { // Was written to during read
            stamp = lock.readLock();
            try {
                value = cache.get(key);
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return value;
    }
    
    public void put(K key, V value) {
        long stamp = lock.writeLock();
        try {
            cache.put(key, value);
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

---

## 5. Common Mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Double-checked locking without volatile | Publication race | Use `volatile` or holder class |
| 2 | Synchronized(this) — too coarse | Low concurrency | Narrow scope or use ConcurrentHashMap |
| 3 | Forgetting unlock in finally | Lock leak | Always try/finally unlock |
| 4 | busy-waiting with while(true) | 100% CPU | Use `wait/notify` or `BlockingQueue` |
| 5 | String as lock object | String interning leads to false sharing | `new Object()` as lock |
| 6 | Thread.stop() to kill thread | Corrupted shared state | Volatile flag or interrupt |
| 7 | Not handling InterruptedException | Lost interrupt | Restore interrupt flag |
| 8 | Immutable object published without volatile | Other thread sees stale reference | Use volatile or final |
| 9 | Deadlock from nested synchronized | Threads block forever | Consistent lock ordering |
| 10 | Not sizing thread pool | Thread starvation / OOM | Profile and size appropriately |

---

## 6. Cheat Sheet

```
═══ CONCURRENCY ═══════════════════════════════════════════════

┌─ SYNCHRONIZATION ──────────────────────────────────────────┐
│ synchronized — intrinsic lock, JVM-optimized                │
│ ReentrantLock — tryLock, fairness, Condition                │
│ ReadWriteLock — concurrent reads, exclusive writes          │
│ StampedLock — optimistic reads (lock-free!)                 │
└─────────────────────────────────────────────────────────────┘

┌─ ATOMICS ──────────────────────────────────────────────────┐
│ AtomicInteger   — CAS-based counter                         │
│ LongAdder       — striped counter (high contention)         │
│ AtomicReference — CAS on object ref                         │
│ AtomicStampedReference — CAS with version (solves ABA)      │
└─────────────────────────────────────────────────────────────┘

┌─ CONCURRENT COLLECTIONS ───────────────────────────────────┐
│ ConcurrentHashMap  — per-bucket locking                     │
│ CopyOnWriteArrayList — snapshot on write                    │
│ ConcurrentLinkedQueue — lock-free MPMC queue                │
│ LinkedBlockingQueue — bounded, blocking                     │
│ ConcurrentSkipListMap — sorted concurrent                    │
└─────────────────────────────────────────────────────────────┘

┌─ SYNCHRONIZERS ────────────────────────────────────────────┐
│ CountDownLatch — wait for N events (one-shot)               │
│ CyclicBarrier — N threads meet (reusable)                   │
│ Semaphore — control access count                            │
│ Exchanger — swap data between 2 threads                     │
│ Phaser — flexible multi-phase barrier                       │
└─────────────────────────────────────────────────────────────┘

┌─ COMPLETABLEFUTURE ─────────────────────────────────────────┐
│ supplyAsync() — async computation with result               │
│ thenApply() — transform result                              │
│ thenAccept() — consume result                               │
│ thenCombine() — combine two futures                         │
│ allOf() — wait for all                                      │
│ anyOf() — wait for first                                    │
│ exceptionally() — handle errors                             │
│ orTimeout() — add timeout                                   │
└─────────────────────────────────────────────────────────────┘

┌─ JMM RULES ────────────────────────────────────────────────┐
│ • synchronized provides visibility + atomicity               │
│ • volatile provides visibility only (no atomicity)          │
│ • final fields are safe-published in constructor             │
│ • Thread.start() happens-before first action in thread      │
│ • Thread.join() happens-before after all thread actions      │
└─────────────────────────────────────────────────────────────┘
```
