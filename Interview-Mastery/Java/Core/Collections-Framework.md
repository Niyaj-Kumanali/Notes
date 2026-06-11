# Java Collections Framework

---

## Overview

- **Purpose** — The Java Collections Framework (JCF) provides a unified architecture for storing, retrieving, manipulating, and communicating groups of objects. It defines a hierarchy of interfaces like `List`, `Set`, `Map`, `Queue`, and `Deque` with battle-tested implementations and reusable algorithms in the `Collections` utility class.

- **Design Principle** — JCF separates contracts from implementations, enabling polymorphism and interchangeability across the collection ecosystem. Code written against an interface can swap implementations (e.g., `ArrayList` to `LinkedList`) without modifying a single line of application logic.

- **History** — Before JCF was introduced in Java 1.2, developers worked with fragmented APIs (`Vector`, `Hashtable`, `Stack`, raw arrays) with inconsistent naming and no common algorithmic support. The framework solved these problems by introducing standard interfaces and the `Iterable` interface for seamless integration with the for-each loop and Stream API.

  **Why not fix `Vector` and `Hashtable`?** Retrofitting interfaces onto concrete classes would break binary compatibility, and single inheritance meant a class could not be both a `List` and a `Deque` — something `LinkedList` achieves today precisely because it implements multiple interfaces rather than extending a concrete class.

  **Why does `Map` sit outside `Collection`?** A map is not a collection of elements — it is a collection of key-value associations. If `Map` extended `Collection`, every map would need `add(Object)`, which has no semantic meaning for a key-value structure. The `entrySet()`, `keySet()`, and `values()` view methods bridge the gap instead.

  **Why fail-fast iterators?** Concurrent modification during iteration is always a bug in single-threaded code. Fail-fast makes it visible immediately rather than silently producing corrupted results. The alternative — letting iteration continue over a structurally modified collection — would produce results that are wrong in ways that are nearly impossible to reproduce or diagnose.

---

## Core Concepts

- **Root Interface** — The JCF is rooted at `Iterable`, which provides the `iterator()` method and enables the for-each loop. `Collection` extends `Iterable` and serves as the root interface for `List`, `Set`, and `Queue`.
- **List** — An ordered, index-based sequence that allows duplicates and provides positional access.
- **Set** — Prohibits duplicate elements and enforces uniqueness through the `equals()` and `hashCode()` contract.
- **Queue and Deque** — Manage elements prior to processing, typically in FIFO order. `Deque` adds support for insertion and removal at both ends.
- **Map** — Sits separately from the `Collection` hierarchy and stores key-value pairs with unique keys.

---

### List Interface

- **ArrayList** — The most commonly used `List` implementation, backed by a resizable array that grows by 50% when full (`oldCapacity + (oldCapacity >> 1)`), starting from a default capacity of 10.

  The 50% growth factor is a deliberate trade-off. `Vector` used 100% (doubling), which wastes memory by over-allocating. A smaller factor like 25% would reduce waste but increase resize frequency, making `add()` more expensive on average. 50% sits between those extremes. The more important advantage of `ArrayList` is CPU cache locality: a single `get()` loads a 64-byte cache line containing ~16 adjacent references, which means iterating 100K elements can generate zero cache misses. Insertions and removals in the middle are O(n) due to `System.arraycopy()` shifting, but that cost is usually acceptable because the cache-friendly access pattern makes the shift itself fast.

- **LinkedList** — Backed by a doubly-linked list where each element is a `Node` object with `prev` and `next` pointers. On a 64-bit JVM with compressed OOPs, each `Node` consumes ~28 bytes (12-byte header + three 4-byte references + 4-byte padding), plus the element itself — roughly 7× the per-element memory of `ArrayList`. At 1M elements, that is ~28 MB of Node overhead vs ~4 MB for an `ArrayList`.

  It offers O(1) insertions and deletions at either end or the middle when using a `ListIterator`, but O(n) for positional access because it must traverse from whichever end is closer. `LinkedList` implements both `List` and `Deque`, which is often the only legitimate reason to reach for it — and even then, `ArrayDeque` is usually the better `Deque` implementation.

- **Usage Guidance** — `ArrayList` is the correct default for nearly all use cases. `LinkedList` should only be considered for frequent insertions and deletions at the beginning of the list when `ArrayDeque` cannot be used.

---

### Set Interface

- **HashSet** — Backed by a `HashMap`, offering O(1) average time for `add()`, `remove()`, and `contains()` with no ordering guarantees. Correctness depends entirely on proper `hashCode()` and `equals()` implementations — failing to override both is one of the most common bugs in Java.
- **LinkedHashSet** — Extends `HashSet` by maintaining a doubly-linked list through all entries, preserving insertion order with only slightly worse performance.
- **TreeSet** — Backed by a `TreeMap` (Red-Black tree), maintaining elements in sorted order via natural ordering or a `Comparator`. All operations are O(log n) and null values are not allowed.

  One subtlety worth noting: `TreeSet` uses `compareTo()` exclusively for element identity, not `equals()`. Two elements are considered equal when `compare(a, b) == 0`. This creates a silent trap with types like `BigDecimal`, where `new BigDecimal("2.0").compareTo(new BigDecimal("2.00"))` returns 0 but `equals()` returns false. The set will silently reject the second value as a duplicate even though your application logic may consider them distinct.

- **Selection Criteria** — `HashSet` for fastest performance with no ordering, `LinkedHashSet` for insertion-order iteration, `TreeSet` for sorted iteration or range queries (`subSet()`, `headSet()`, `tailSet()`).

---

### Map Interface

- **HashMap** — The workhorse implementation, backed by an array of buckets (`Node<K,V>[]`) with a default initial capacity of 16 and a load factor of 0.75. Provides O(1) average time for `get()` and `put()`, degrading only under hash collisions.

  **Java 8+ treeification:** Buckets exceeding 8 entries convert to balanced Red-Black trees (provided the table has at least 64 buckets), improving worst-case from O(n) to O(log n). This directly addresses a pre-Java-8 security vulnerability where malicious input engineered to produce hash collisions could force O(n) map operations — a well-known DoS vector in web frameworks at the time.

  The threshold of 8 is not arbitrary. Derived from the Poisson distribution: at the 0.75 load factor, the probability of 8 collisions in a single bucket by chance is less than 1 in 10 million. So if a bucket reaches 8 entries in practice, it is almost certainly the result of a bad `hashCode()` implementation or deliberate attack, not normal usage. The untreeify threshold is 6 rather than 8, creating a hysteresis gap that prevents oscillation when elements are repeatedly added and removed near the boundary.

- **LinkedHashMap** — Extends `HashMap` with a doubly-linked list running through all entries, supporting insertion-order or access-order iteration. Access-order mode is the foundation for LRU cache implementations via the `removeEldestEntry()` hook.

- **TreeMap** — Uses a Red-Black tree internally, maintaining keys in sorted order with O(log n) operations. Ideal for range queries, prefix scans, and navigational operations like `subMap()`, `headMap()`, and `tailMap()`.

- **EnumMap** — A specialized `Map` for enum keys that uses a plain array indexed by ordinal. It delivers 2–3× better performance than `HashMap` with zero memory waste — there are no `Node` objects, no load factor, and no hashing at all. If your keys are enums, there is no reason to use `HashMap`.

---

### Queue and Deque

- **Queue Operations** — Two sets of operations: one throws exceptions on failure (`add()`, `remove()`, `element()`), the other returns sentinel values (`offer()` → false, `poll()` → null, `peek()` → null). The sentinel-value variants are generally preferred in application code because they allow normal control flow without exception handling.

- **Deque** — Extends `Queue` with `addFirst()`, `addLast()`, `offerFirst()`, `offerLast()`, `pollFirst()`, and `pollLast()`.

- **ArrayDeque** — The recommended implementation for both stack and queue use cases. Backed by a resizable array, no null elements allowed, no per-node overhead, and better performance than `LinkedList` due to CPU cache locality. The fact that Java's own `Stack` class exists is mostly historical — `ArrayDeque` is the correct replacement.

- **PriorityQueue** — A heap-based unbounded queue ordering elements by natural ordering or a `Comparator`. Provides O(log n) insertion and O(1) retrieval of the head element. A common misconception is that iterating a `PriorityQueue` produces elements in sorted order — it does not. Only `poll()` is guaranteed to return elements in priority order. The heap only maintains the invariant that each parent is smaller than its children; the rest of the array is deliberately unsorted. Maintaining full sorted order would cost O(n log n) per insertion. The heap trades that for O(log n), which is the entire point.

---

### Concurrent Collections

- **ConcurrentHashMap** — Replaces `Collections.synchronizedMap()` with per-bucket locking (JDK 8+): writes synchronize only on individual bucket heads, so threads writing to different buckets proceed in parallel. Reads are entirely lock-free — `get()` acquires no lock because `Node.val` is `volatile`, establishing a happens-before guarantee without synchronization cost.

  The two methods most worth understanding deeply are `computeIfAbsent()` and `compute()`. `computeIfAbsent()` uses a temporary `ReservationNode` as a placeholder to guarantee the mapping function runs at most once per key, even under concurrent calls — this is what makes it safe to use for lazy initialization without an external lock. `compute()` provides atomic read-modify-write semantics per key, which is the correct tool for any operation that reads a value and writes it back conditionally. Using `get()` + `put()` instead introduces a race window between the two calls.

- **CopyOnWriteArrayList** — Creates a fresh copy of the underlying array on every mutative operation, making reads lock-free. Ideal for read-heavy, write-rare scenarios like event listener registries. The trade-off is that writes are O(n) and the old array sits in memory until GC collects it — never use this for write-heavy workloads.

- **ConcurrentLinkedQueue** — A lock-free, unbounded queue using CAS operations internally. Suitable for high-throughput producer-consumer patterns where backpressure is not needed.

- **LinkedBlockingQueue** — A bounded blocking queue with backpressure. Threads block when the queue is full or empty, coordinated via internal `Condition` objects. Uses separate `ReentrantLock` instances for take and put operations, so borrowers and returners rarely contend with each other.

- **ConcurrentSkipListMap** — A sorted, concurrent navigable map using a skip-list data structure. O(log n) operations with probabilistic balancing, lock-free reads, and snapshot iterators that remain consistent during concurrent writes. Unlike `TreeMap` backed by a Red-Black tree, the skip-list does not need rebalancing locks, which is why it appears in systems like Kafka for offset tracking where concurrent reads and writes are constant.

---

### Collections Utility Class

- **Sorting and Searching** — `sort()`, `binarySearch()`, `reverse()`, `shuffle()`, `rotate()`, `swap()`.
- **Synchronization Wrappers** — `synchronizedList()`, `synchronizedMap()`, `synchronizedSet()` wrap collections with coarse-grained synchronization. Largely superseded by `java.util.concurrent` — see the design considerations section for why these fall short under contention.
- **Unmodifiable Wrappers** — `unmodifiableList()`, `unmodifiableMap()`, `unmodifiableSet()` create read-only views that throw `UnsupportedOperationException` on mutation. Important caveat: the underlying collection can still change through its original reference. This is a view, not a copy.
- **Factory Methods** — `singletonList()`, `emptyList()`, `emptySet()` for edge cases. Since Java 9, `List.of()`, `Set.of()`, and `Map.of()` create deeply immutable collections — no reference to a mutable backing collection exists at all, making them safer as API return types than the unmodifiable wrappers.

---

## Common Mistakes

- **ConcurrentModificationException** — Modifying a collection directly while iterating throws `CME` because the iterator checks a `modCount` field on each `next()` call. The `modCount` is incremented on every structural modification — add, remove, or resize. The iterator captures this value at creation time and compares on every step. Use `iterator.remove()` or `Collection.removeIf()` (Java 8+) instead. In multi-threaded code, this is especially insidious because the exception may appear intermittently and only under load, long after the offending modification.

- **Mutable Keys in HashMap/HashSet** — If a key's `hashCode()` changes after insertion, the map cannot locate that entry anymore — `get()` returns `null` even though the entry exists. The map looks in the wrong bucket because the bucket index is derived from the hash. The entry is not gone; it is simply unreachable. Always use immutable keys (`String`, `Integer`, `UUID`) or ensure `hashCode()` depends only on `final` fields.

- **Inconsistent equals() and hashCode()** — The contract: if `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must hold. Violating this silently breaks `HashMap`, `HashSet`, and `Hashtable`. The reverse is not required — two unequal objects can have the same hash (collision) — but the forward direction is mandatory. Use `java.util.Objects.hash()` and `Objects.equals()` for correct, boilerplate-free implementations.

- **LinkedList as Default** — Each element requires a `Node` object (~40 bytes overhead), breaking CPU cache locality and increasing GC pressure. The GC must traverse every `Node` as a separate heap object. `ArrayList` operations at the end are O(1) amortized with no allocation per element.

- **synchronizedMap() vs ConcurrentHashMap** — `synchronizedMap()` locks the entire map on every read and write. Under contention with many threads, all reads block each other even though reads could safely proceed in parallel. `ConcurrentHashMap`'s lock-free reads mean concurrent `get()` calls never block, regardless of how many threads are reading simultaneously.

- **Capacity vs Size Confusion** — `new ArrayList<>(100).size()` is 0, not 100. The constructor argument sets the internal array's initial capacity to avoid early resizes — it does not populate the list. The list is still empty until elements are added.

- **TreeSet / TreeMap and hashCode()** — These do not use `hashCode()` or `equals()` at all. Identity is determined entirely by `compareTo()` or `Comparator.compare()`. Two elements are considered equal when `compare(a, b) == 0`. The `BigDecimal` trap: `new BigDecimal("2.0").equals(new BigDecimal("2.00"))` is false, but `compareTo` returns 0, so `TreeSet` silently treats them as duplicates. If your application considers them distinct, `TreeSet` is the wrong data structure.

---

## Design Considerations

- **Access Patterns** — Random access favors `ArrayList` for O(1) positional `get()` and cache line efficiency. Sequential access with frequent head insertions favors `ArrayDeque`. The distinction matters most at scale — an algorithm that looks like O(n) with `ArrayList` can become O(n²) in practice with `LinkedList` due to cache miss overhead on every pointer dereference.

- **Collection Size** — For small collections (under ~100 elements), algorithmic complexity is often irrelevant compared to memory overhead and allocation cost. A `TreeSet` with 10 elements is not meaningfully slower than a `HashSet` with 10 elements — the overhead of object creation and GC pressure can dominate.

- **Thread Safety Granularity** — Thread-safe individual operations do not compose into thread-safe compound operations. The classic mistake: checking `if (!map.containsKey(k))` and then calling `map.put(k, v)` — another thread can insert between the two calls even with `synchronizedMap`. The fix is `computeIfAbsent()` or `putIfAbsent()`, which are atomic at the map level.

- **Memory Footprint** — `ArrayList` with 1 million `Integer` elements consumes roughly 4 MB. `LinkedList` with the same data consumes 40+ MB due to per-node object overhead. At scale, this is the difference between fitting in L3 cache and forcing heap allocations. For numeric-heavy workloads, specialized primitive collections like `Int2ObjectOpenHashMap` (fastutil) can reduce memory by 50–70% compared to a boxed `HashMap<Integer, V>`.

- **Collection Sizing** — A `HashMap` growing from default capacity to 100 entries resizes at 16, 32, 64, and 128 — four full rehashes. Each rehash re-indexes every entry in the new bucket array. Pre-size with `expectedSize / loadFactor + 1` to eliminate all of this. For `ArrayList`, `new ArrayList<>(expectedSize)` avoids incremental array copying during population.

**When not to use each implementation:**
- `ArrayList` for frequent head insertions — use `ArrayDeque` instead.
- `LinkedList` almost always — `ArrayDeque` or `ArrayList` is better in practice.
- `HashMap` when sorted iteration or range queries are needed — `TreeMap` provides `subMap()`.
- `HashMap` for enum keys — `EnumMap` is 2–3× faster with zero memory waste.
- `ConcurrentHashMap` when null keys/values are required — not supported; fall back to `synchronizedMap`.
- `ConcurrentHashMap` for cross-key atomicity — only per-key atomicity is guaranteed.

**What a senior engineer evaluates before choosing a collection:**
1. **Access pattern** — read-to-write ratio; positional vs value-based access.
2. **Concurrency** — single-threaded, low-contention, or high-contention? Are stale reads acceptable?
3. **Size bounds** — 10 vs 10K vs 10M elements changes the decision completely.
4. **Memory budget** — per-element overhead matters at scale; boxing costs for primitives.
5. **Ordering** — is insertion order, sorted order, or no order required?
6. **Operational exposure** — is the collection returned from an API? If so, immutable wrapper needed.
7. **Library policy** — can external dependencies (Caffeine, fastutil) be used, or must it be JDK-only?

---

## Real-World Scenarios

### Scenario 1: LRU Cache in a Web Application

A web server needs to cache the last 1000 recently viewed product pages per user session. The cache must evict the least-recently-viewed product when it exceeds capacity, and every cache hit must refresh the access timestamp so hot items remain in the cache longer.

```java
public class ProductCache<K, V> {
    private final LinkedHashMap<K, V> cache;

    public ProductCache(int maxSize) {
        // Third parameter true = access-order mode
        this.cache = new LinkedHashMap<>(16, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > maxSize;
            }
        };
    }

    public V get(K key) { return cache.get(key); }
    public void put(K key, V value) { cache.put(key, value); }
}
```

`LinkedHashMap` with access-order enabled moves every accessed entry to the tail of the internal doubly-linked list. When `removeEldestEntry()` returns true, the entry at the head — the least recently accessed — is automatically evicted. The `protected` method is an intentional extension point; `LinkedHashMap` was designed for subclassing in exactly this pattern.

The critical detail is the third constructor argument `true`. The default constructor sets `accessOrder=false` (insertion order), which means `get()` does not reorder the entry — and the LRU behavior silently breaks. A common workaround is calling `remove(key)` followed by `put(key, value)` in the `get()` method to manually move the entry to the tail, but this is unnecessary and error-prone when access-order is enabled correctly.

For production caches that also need TTL expiry, access statistics, or soft reference eviction, Caffeine is the correct choice. `LinkedHashMap` is appropriate when you want zero external dependencies and can accept manual thread-safety handling.

---

### Scenario 2: Priority-Based Task Scheduler

A job scheduler processes tasks with different priorities. High-priority tasks must execute before low-priority ones, but tasks with the same priority must be processed in FIFO order to ensure fairness.

```java
public class PriorityTaskScheduler {
    private final PriorityQueue<ScheduledTask> queue = new PriorityQueue<>(
        Comparator.comparingInt(ScheduledTask::priority)
            .thenComparing(ScheduledTask::enqueuedAt)
    );

    public void submit(ScheduledTask task) { queue.offer(task); }
    public ScheduledTask next() { return queue.poll(); }
}
```

The composite comparator is the key design decision. `PriorityQueue` alone does not guarantee ordering between equal-priority elements — the heap only guarantees the minimum is at the head. The `thenComparing(ScheduledTask::enqueuedAt)` clause breaks ties deterministically by insertion timestamp, achieving FIFO within each priority tier. Without it, equal-priority tasks can be returned in any order.

`offer()`/`poll()` are used over `add()`/`remove()` because they return false/null on failure instead of throwing exceptions — appropriate for control flow in a scheduler where an empty queue is a normal state, not an error.

For concurrent access, wrap with `PriorityBlockingQueue`. A sorted `ArrayList` is a tempting alternative but costs O(n) per insertion due to element shifting, making it unsuitable at scale.

---

### Scenario 3: High-Throughput Metrics Aggregator

A metrics service receives 100K events per second from multiple threads. The system must aggregate values per metric name and snapshot every minute.

```java
public class MetricsAggregator {
    private final ConcurrentHashMap<String, LongAdder> counters = new ConcurrentHashMap<>();

    public void record(String metric, long value) {
        counters.computeIfAbsent(metric, k -> new LongAdder()).add(value);
    }

    public Map<String, Long> snapshotAndReset() {
        Map<String, Long> snapshot = new HashMap<>();
        counters.forEachKey(1, key -> {
            LongAdder adder = counters.remove(key);
            if (adder != null) snapshot.put(key, adder.sum());
        });
        return snapshot;
    }
}
```

Two independent decisions here are both worth understanding. First, `computeIfAbsent` over `putIfAbsent`: `putIfAbsent` allocates a `LongAdder` on every call regardless of whether the key already exists, and under concurrent calls, multiple `LongAdder` instances can be created with only one surviving. `computeIfAbsent` runs the factory function exactly once, under the internal bucket lock, so no wasted allocation occurs.

Second, `LongAdder` over `AtomicLong`: under high contention, `AtomicLong.incrementAndGet()` retries its CAS in a spin loop whenever another thread writes concurrently. `LongAdder` stripes writes across a cell array — each thread updates its own cell, and the sum is only aggregated when `sum()` is called. This trades slightly more memory for dramatically less CAS contention at high write rates.

The `remove(key)` in `snapshotAndReset()` is not incidental — it ensures counters drain atomically so the same value cannot be counted in two consecutive snapshots. Without it, the snapshot would return cumulative totals instead of per-interval values.

---

## Scenario-Based Questions

**Q: You are building an inventory management system where multiple warehouse workers scan items simultaneously. Each scan updates stock counts. How do you prevent lost updates without locking the entire inventory?**

Use `ConcurrentHashMap<String, AtomicInteger>` where each product's stock is an `AtomicInteger` using compare-and-swap at the hardware level, eliminating synchronized blocks for single-value updates. The `ConcurrentHashMap` provides per-bucket concurrency, so workers scanning different products proceed in parallel. For multi-step operations, use `compute()` which is atomic per key: `inventory.compute("SKU-123", (k, v) -> v == null ? 0 : v.decrementAndGet())`. This eliminates the put-if-absent race condition where two threads read the same value and overwrite each other's update. For batch operations like end-of-day reconciliation, `forEachKey()` or `reduceKeys()` enable parallel bulk processing without external synchronization.

> **Interview follow-up:** The candidate mentioned `compute()` — but `compute()` locks the bucket, not just the key. What happens if two unrelated keys hash to the same bucket? How would you prove contention is negligible before putting this in production?

**Q: You have a REST API that returns paginated results from a leaderboard. Users can view any page. The leaderboard changes every few seconds. How do you ensure that a user viewing page 2 does not see duplicates or miss items?**

Use a `CopyOnWriteArrayList` for the leaderboard snapshot — sort it once per refresh interval and atomically replace the reference. Each page request reads `subList(fromIndex, toIndex)` from the current snapshot, so users never experience rankings shifting mid-pagination. The stale-snapshot problem (a user may see an older leaderboard state) is a deliberate and acceptable trade-off for a leaderboard updating every 5–10 seconds. For stricter consistency, use a `volatile` reference to an `unmodifiableList` and replace atomically — this provides a happens-before guarantee between the writer and all reader threads.

> **Interview follow-up:** The `CopyOnWriteArrayList` creates an array copy on every sort. With 10K leaderboard entries sorted every 5 seconds, what is the per-minute allocation rate, and at what point does this become a GC problem?

**Q: You are designing a rate limiter that tracks requests per user in a 1-second sliding window. The system handles 50K QPS across 10K users. How do you store and expire request timestamps efficiently?**

Use `ConcurrentHashMap<String, ArrayDeque<Long>>` with per-key atomicity via `compute()`. The remapping function runs under a bucket-level lock scoped to that specific key: it evicts timestamps older than 1 second from the deque's head, checks whether the count exceeds the limit, and pushes the current timestamp to the tail — all atomically. Concurrent requests for different users proceed without contention; concurrent requests for the same user serialize only at that bucket. A background task should periodically clean up entries whose deques are empty to prevent memory growth from users who hit the limiter once and never return.

> **Interview follow-up:** The `ArrayDeque` grows unbounded for a user who never stops sending requests. How would you bound per-user memory without breaking the sliding-window semantics?

**Q: You are implementing an undo/redo system for a text editor. Users can perform thousands of operations. The undo stack must not grow unbounded. How do you design this with standard collections?**

Use `ArrayDeque` as a bounded stack by manually enforcing a capacity limit — when pushing a new operation, if the deque is at capacity, poll the oldest entry from the front before pushing the new one to the back. The redo stack is a separate `ArrayDeque` that is cleared on every new user action that is not an undo or redo, because any new action invalidates the forward history. `ArrayDeque` is correct here: no per-node allocation means no GC pressure on every keystroke, and O(1) amortized `addLast()`/`pollFirst()` means the cost is constant regardless of how deep the history grows.

> **Interview follow-up:** The candidate mentioned clearing the redo stack on new actions. What happens if the user does undo → edit → undo? Should the re-applied operation appear in the original undo history, or is that a separate branch?

**Q: A microservice produces events and consumers must process them in order per partition. If consumer A dies, another consumer must resume from the last committed offset. Which collection model does this resemble?**

A sorted, concurrent map per partition — which is exactly what systems like Kafka use internally. `ConcurrentSkipListMap` fits the model: monotonically increasing offsets as keys, O(log n) lookups, and `subMap()` range queries to retrieve records from offset X to Y without scanning the full partition. The skip-list's naturally concurrent structure supports lock-free reads and fine-grained write locking, which mirrors how consumers track positions: a sorted map where the current offset advances sequentially and any consumer can seek to an arbitrary position. Snapshot iterators allow consistent traversal during consumer rebalancing, even while producers continue writing new offsets.

> **Interview follow-up:** The candidate mentioned snapshot iterators as a benefit. What happens to the snapshot's memory when the producer keeps writing new offsets while a consumer is iterating a large range? Does the snapshot grow, or is it fixed at creation time?

**Q: You are building a dependency resolver that must detect circular dependencies. You have millions of nodes. Which collection do you use for the DFS visited set?**

Use `HashSet<Node>` for both the current-path set (cycle detection via recursion stack tracking) and the fully-processed set (avoid revisiting resolved subgraphs). On entering a node, add it to the path set — if it is already present, a cycle exists. After processing all children, remove it from the path set (backtracking) and add it to the processed set so future paths reaching this node skip the subtree. Both sets must be `HashSet` because O(1) membership checks are essential at millions of nodes — `TreeSet`'s O(log n) would add tens of millions of comparisons. For memory-constrained environments with sequential integer node IDs, a `BitSet` or `RoaringBitmap` stores millions of visited states in a few megabytes.

**Q: An ad-serving platform needs the top 50 ads by revenue from a stream of 10 million impressions per hour. Only one pass is allowed.**

Use a min-heap via `PriorityQueue` capped at 51 elements. For each impression, add the ad to the heap; if size exceeds 50, evict the minimum. After the full stream, the heap contains exactly the top 50. Time complexity is O(n log 50) — effectively O(n) — compared to O(n log n) for full sorting. For parallel processing, each thread maintains its own local top-50 heap, and the heaps are merged at the end using the same eviction approach. This "top-K with heap eviction" pattern is the basis for top-K aggregation in MapReduce, Spark, and Flink.

**Q: You are designing a connection pool. Threads borrow and return connections. If all connections are in use, a thread must wait. When a connection is returned, waiting threads should be notified. What collection do you use?**

Use `LinkedBlockingQueue<Connection>` with a fixed capacity. Borrowing calls `poll(timeout, unit)` — returns null on timeout rather than blocking indefinitely. Returning calls `offer(conn)`, which internally wakes one waiting thread via a `Condition` object backed by `LockSupport.park()`/`unpark()`. Separate `ReentrantLock` instances for take and put allow concurrent borrowers and returners with minimal contention. Constructing with `fairness=true` enforces FIFO waiting, preventing starvation under high load — in production systems without fairness, some threads can wait indefinitely while others acquire connections repeatedly.

**Q: A collaborative editing application must merge edits from multiple users. Each edit has a timestamp and a position. How do you track edit history for OT or CRDT?**

Use `ConcurrentSkipListMap<Position, Edit>` for the current document state. It provides O(log n) insertion, ordered iteration, and snapshot iterators for consistent reads during live sessions. Per-user edit histories are `LinkedList<Edit>` (append-only), merged to compute the OT transformation function when concurrent edits conflict. The skip-list is particularly well-suited here because it achieves probabilistic balancing without rebalancing locks — unlike a Red-Black tree, no exclusive lock is acquired during restructuring. The `subMap()` and `headMap()` views allow efficient range queries to identify spatially overlapping edits for conflict resolution.

**Q: You need a multi-level cache (L1: heap, L2: Redis, L3: database). Concurrent requests for the same missing key must not cascade. How do you coordinate?**

Use `ConcurrentHashMap<K, CompletableFuture<V>>` as L1. On a cache miss, call `cache.computeIfAbsent(key, k -> fetchFromL2(k))` — atomically ensures only the first caller triggers the fetch while all concurrent callers for the same key block on the same `CompletableFuture`. The fetch checks L2, then L3 on L2 miss, and populates both before completing the future. This "future-based deduplication" or "coalescing cache" pattern prevents the thundering-herd problem where thousands of concurrent requests for an uncached key simultaneously hit the database. Error handling must cache a sentinel value (e.g., `Optional.empty()`) on failure — if the future throws and is not cached, every subsequent caller triggers a new fetch, amplifying load exponentially.

**Q: A production service uses `ConcurrentHashMap.computeIfAbsent()` to lazily load configuration. After a deployment, database load spikes 100× and the service becomes unresponsive. What happened?**

The mapping function threw an exception — likely a database timeout. When the mapping function throws, `computeIfAbsent()` does not cache the result. Every subsequent caller retries, each hitting the already-overloaded database and failing, creating a feedback loop. Fix: wrap the mapping function to catch exceptions and cache a sentinel value (`Optional.empty()`), then add a circuit breaker to stop retrying after a threshold of consecutive failures.

**Q: After migrating from JDK 8 to JDK 11, memory usage increases for `HashMap`-heavy workloads. What changed?**

Most likely: if `-Djdk.map.althashing.threshold` was set in JDK 8 for hash-collision DoS protection, it was removed in JDK 11 (deprecated and dropped). This can increase collision rates for certain key distributions. The JDK 11 compact strings change (Latin-1 encoding by default) also alters `hashCode()` distribution for ASCII strings. Diagnose with a heap dump comparing bucket fill distributions before and after migration.

**Q: A developer replaced `ConcurrentHashMap` with `HashMap` wrapped in `Collections.synchronizedMap()` because "they are equivalent." After deployment, p99 latency increases from 10ms to 500ms. Why?**

`synchronizedMap()` acquires a single intrinsic lock on every read and write. With 100 concurrent threads, 99 queue up on that mutex — and reads block each other even though reads could safely proceed in parallel. `ConcurrentHashMap` reads are entirely lock-free. Additionally, `synchronizedMap` provides no atomic compound operations, so the replacement likely introduced `get()`+`put()` patterns with race windows where `computeIfAbsent()` or `merge()` were previously used.

---

## Interview Questions

**What is the difference between fail-fast and fail-safe iterators?**
Fail-fast iterators (`ArrayList`, `HashMap`, `HashSet`) throw `ConcurrentModificationException` when the collection is structurally modified after iterator creation. They detect this via a `modCount` field incremented on every structural modification — the iterator captures the value at creation and compares on every `next()` call. Fail-safe iterators (`ConcurrentHashMap`, `CopyOnWriteArrayList`) operate on a snapshot or a copy of the underlying data, so concurrent modifications do not affect ongoing iteration. The trade-off is memory overhead from the snapshot and the possibility of reading stale data.

**When would you use LinkedList over ArrayList?**
`LinkedList` is appropriate when you need frequent insertions or deletions at the beginning or middle using a `ListIterator`, or when implementing a queue or deque via its dual `List`/`Deque` interface implementation. In practice it is rarely optimal: ~40 bytes of per-node overhead from `Node.prev`, `Node.next`, `Node.item`, and the object header, plus non-contiguous memory that destroys CPU cache locality.

**How does HashMap handle collisions?**
Java 8+ uses separate chaining: linked lists that convert into balanced Red-Black trees when a bucket exceeds 8 entries and the table has at least 64 buckets, improving worst-case from O(n) to O(log n). The tree falls back to a linked list when the bucket shrinks below 6 entries. Before Java 8, only linked lists were used — malicious hash collisions could force O(n) operations, making hash maps a real DoS attack surface in web frameworks.

**What is the difference between HashMap and ConcurrentHashMap?**
`HashMap` is not thread-safe — concurrent modifications cause race conditions, data corruption, or `ConcurrentModificationException`. `ConcurrentHashMap` uses per-bucket locking for writes and lock-free reads via volatile semantics. It also provides atomic compound operations (`computeIfAbsent()`, `merge()`, `forEachKey()`) that `HashMap` + `synchronizedMap` cannot replicate safely. It does not allow null keys or values.

**How does LinkedHashMap maintain insertion order?**
Each entry has `before` and `after` references threading a doubly-linked list through the entire map. The list preserves insertion order or access order, configurable via the `accessOrder` constructor parameter. Access-order mode moves each accessed entry to the list tail on every `get()` or `put()`. The `removeEldestEntry()` hook is called on every `put()` and `putAll()` to determine whether the head entry (eldest) should be evicted — the designed extension point for LRU caches.

**What is the difference between TreeSet and HashSet?**
`HashSet` is backed by `HashMap` with O(1) average operations, no ordering, and one null allowed. `TreeSet` is backed by `TreeMap` with O(log n) operations, sorted order via `Comparable` or `Comparator`, and no null support. `TreeSet` is the right choice when you need range queries (`subSet()`, `headSet()`, `tailSet()`). `HashSet` when you only need uniqueness and raw performance.

**What are the initial capacity and load factor of HashMap? Why do they matter?**
Default capacity 16, default load factor 0.75. The map resizes — doubling capacity — when `size > capacity × loadFactor`. A higher load factor saves memory but increases collisions and degrades lookup performance. A lower load factor reduces collisions but wastes memory. Pre-sizing with `(expectedSize / 0.75f) + 1` eliminates all resize overhead when the eventual size is known upfront.

**How do you make a collection immutable or unmodifiable?**
`Collections.unmodifiableList()` and similar methods create a view that throws `UnsupportedOperationException` on mutation — but the underlying collection can still mutate through its original reference. `List.of()`, `Set.of()`, `Map.of()` (Java 9+) create deeply immutable collections with no backing mutable reference, making them safer for API return types. `List.copyOf()` (Java 10+) creates an immutable independent copy.

**What is the difference between poll() and remove() in Queue?**
Both retrieve and remove the head element. `remove()` throws `NoSuchElementException` on an empty queue; `poll()` returns null. The same pattern applies to `element()` vs `peek()` for non-destructive retrieval, and `add()` vs `offer()` for insertion. The sentinel-value variants (`poll`, `peek`, `offer`) are preferred when an empty queue is a normal expected state rather than an error.

**How does CopyOnWriteArrayList achieve thread safety, and when should you use it?**
Every mutative operation produces a fresh copy of the underlying array while reads operate on the current reference without any locking. This is optimal for read-heavy, write-rare scenarios like event listener registries where the list seldom changes but is iterated on every event. Writes are O(n) with memory pressure from the old array pending GC — never appropriate for write-heavy workloads.

**Design a data structure supporting insert(key), delete(key), and getRandom() in O(1) average time.**
Compose `HashMap<K, Integer>` (key → index) with `ArrayList<K>` (index → key). Insert: append to list, store index in map. Delete: swap the deleted element with the last element in the list (O(1)), update the swapped element's index in the map, remove the last entry. `getRandom()`: generate a random index within the list size. This composition — a map for O(1) lookup and a list for O(1) indexed access — is a recurring system design building block.

**Your service shows 100% CPU in HashMap.get(). Your code does not use HashMap directly. What happened?**
Many libraries (Hibernate, Spring, Tomcat) and the JDK itself use `HashMap` internally. A thread-safety violation or a swallowed `ConcurrentModificationException` can create a cycle in the internal linked list, causing `get()` to loop forever. Take a thread dump, identify the specific `HashMap` instance from the object reference in the stack trace, and check the owning library's version for known concurrency bugs.

**What happens internally during HashMap resize for treeified buckets?**
Resize doubles the bucket array. Treeified buckets split into two chains based on `(hash & oldCap) == 0` — this single bit determines whether an entry stays at the same index or moves to `index + oldCapacity`. Each resulting chain either remains a Red-Black tree (if length ≥ 6) or converts back to a linked list (if < 6). Non-tree buckets use the same bit test for redistribution.

---

## Developer Recommendations

- **Prefer `ConcurrentHashMap` over `synchronizedMap()`** — Per-bucket locking allows concurrent reads and writes to different buckets without contention. Reads are entirely lock-free. In a benchmark with ten concurrent writers, `ConcurrentHashMap` can be 10× faster. The atomic compound operations (`computeIfAbsent()`, `merge()`, `forEachKey()`) also eliminate entire categories of thread-safety bugs that `synchronizedMap` cannot address.

- **Use `ArrayDeque` over `LinkedList`** — No per-node allocation, contiguous memory, excellent cache locality. Iterating 100K elements in `ArrayDeque` may generate zero cache misses; the same iteration on `LinkedList` causes a cache miss on nearly every node dereference.

- **Pre-size collections when size is known** — A `HashMap` growing from default to 100 entries rehashes four times. `new HashMap<>(expectedSize / 0.75f + 1)` eliminates all of that. For `ArrayList`, `new ArrayList<>(expectedSize)` avoids incremental array copying during population.

- **Override `equals()` and `hashCode()` consistently** — If `a.equals(b)` is true, `a.hashCode() == b.hashCode()` must hold. Mutable keys are especially dangerous — a changed hash after insertion makes the entry permanently unreachable without removing it. Use `Objects.hash()` and `Objects.equals()` for straightforward correct implementations.

- **Use `EnumMap` and `EnumSet` for enum keys** — Backed by plain arrays indexed by ordinal. No hash computation, no `Node` objects, no load factor. `EnumMap` outperforms `HashMap` by 2–3× in practice with a fraction of the memory.

- **Return immutable collections from API methods** — A direct `ArrayList` reference lets callers silently mutate internal state. Use `Collections.unmodifiableList(internalList)` for a view, or `List.copyOf(internalList)` (Java 10+) for a truly independent immutable copy.

- **Use `computeIfAbsent()` over `putIfAbsent()`** — `computeIfAbsent()` is atomic: the factory runs at most once per key, and all concurrent callers receive the same value. `putIfAbsent()` requires a separate `get()` to retrieve the inserted value, and two concurrent threads can allocate two instances before either reads the result — wasting both allocation and initialization cost.

---

## Production Patterns

**Thundering herd prevention:** `ConcurrentHashMap.computeIfAbsent()` + `CompletableFuture` deduplicates concurrent cache misses — only one thread fetches from the database; N-1 threads block on the same future. Without this, 1000 concurrent requests for the same uncached key all hit the database simultaneously.

**Event batching:** An `ArrayList` accumulates events from a producer thread. When the batch reaches a threshold (e.g., 500 items) or a time window expires (e.g., 5 seconds), the list is handed off to a `LinkedBlockingQueue` and a new list is created. This amortizes I/O cost and reduces per-event overhead.

**Rate limiting:** `ConcurrentHashMap<String, ArrayDeque<Long>>` with `compute()` provides atomic per-key sliding-window eviction. The `compute()` function atomically evicts expired timestamps, checks the limit, and pushes the current timestamp — all under a single bucket lock scoped to that key.

**Connection pooling:** `LinkedBlockingQueue` with fixed capacity enforces backpressure. Borrowers call `poll(timeout, unit)` with a configurable timeout; returners call `offer()`. The queue uses separate `ReentrantLock`s for take and put, so borrowers and returners rarely contend.

---

## Debugging Quick Reference

### Symptom → Root Cause

| Symptom | Likely Root Cause |
|---------|-------------------|
| `ConcurrentModificationException` | Collection modified while iterating without `iterator.remove()` |
| 100% CPU in `HashMap.get()` | Concurrent modification created a cycle in the internal linked list (JDK 7 bug or unsynchronized access) |
| `get()` returns null for existing key | Mutable key's `hashCode()` changed after insertion |
| Memory grows unbounded | `HashMap` used as cache without eviction strategy |
| High GC pause times | `LinkedList` per-node overhead — each `Node` is a GC root traversal target |
| p99 latency spike after collection change | `synchronizedMap` replacing `ConcurrentHashMap` (all threads serialize on one lock) |
| Duplicate entries in `Set` | Inconsistent `equals()` / `hashCode()` |
| Intermittent test failures | `HashMap` iteration order is non-deterministic; tests assumed a specific order |

### Common Fixes

| Bug | Fix |
|-----|-----|
| `ConcurrentModificationException` | Use `list.removeIf(predicate)` or `iterator.remove()` |
| `HashMap` infinite loop (JDK 7) | Upgrade to JDK 8+ or switch to `ConcurrentHashMap` |
| Mutable key lost in `HashMap` | Make key immutable; ensure `hashCode()` uses only `final` fields |
| `LinkedList` O(n²) access in loop | Replace with `ArrayList` or restructure to use `ListIterator` |
| `synchronizedMap` contention | Replace with `ConcurrentHashMap` |
| Unbounded cache OOM | Add eviction via `LinkedHashMap.removeEldestEntry()` or switch to Caffeine |
| Stale data in concurrent map | Use `compute()` for atomic read-modify-write instead of `get` + `put` |