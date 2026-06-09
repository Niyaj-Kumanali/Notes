# Java Collections Framework

---

## Overview

- **Purpose** — The Java Collections Framework (JCF) provides a unified architecture for storing, retrieving, manipulating, and communicating groups of objects. It defines a hierarchy of interfaces like `List`, `Set`, `Map`, `Queue`, and `Deque` with battle-tested implementations and reusable algorithms in the `Collections` utility class.
- **Design Principle** — JCF separates contracts from implementations, enabling polymorphism and interchangeability across the collection ecosystem. Code written against an interface can swap implementations (e.g., `ArrayList` to `LinkedList`) without modifying a single line of application logic.
- **History** — Before JCF was introduced in Java 1.2, developers worked with fragmented APIs (`Vector`, `Hashtable`, `Stack`, raw arrays) with inconsistent naming and no common algorithmic support. The framework solved these problems by introducing standard interfaces and the `Iterable` interface for seamless integration with the for-each loop and Stream API.

  **Why not fix `Vector` and `Hashtable`?** Retrofitting interfaces onto concrete classes would break binary compatibility, and single inheritance meant a class could not be both a `List` and a `Deque` (something `LinkedList` achieves today). **Why does `Map` sit outside `Collection`?** A map is not a collection of elements — it is a collection of key-value associations. If `Map` extended `Collection`, every map would need `add(Object)`, which has no semantic meaning for a key-value structure. The `entrySet()`, `keySet()`, and `values()` view methods bridge the gap instead. **Why fail-fast iterators?** Concurrent modification during iteration is always a bug in single-threaded code. Fail-fast makes it visible immediately (crash early) rather than silently producing corrupted results.

---

## Core Concepts

- **Root Interface** — The JCF is rooted at the `Iterable` interface, which provides the `iterator()` method and enables the for-each loop. `Collection` extends `Iterable` and serves as the root interface for `List`, `Set`, and `Queue`.
- **List** — Represents an ordered, index-based sequence that allows duplicates and provides positional access. `Set` prohibits duplicate elements and enforces uniqueness through the `equals()` and `hashCode()` contract.
- **Queue and Deque** — Manage elements prior to processing, typically in FIFO order. `Deque` adds support for insertion and removal at both ends.
- **Map** — Sits separately from the `Collection` hierarchy (it does not extend `Collection`) and stores key-value pairs with unique keys.

### List Interface

- **ArrayList** — The most commonly used `List` implementation, backed by a resizable array that grows by 50% when full (`oldCapacity + (oldCapacity >> 1)`), starting from a default capacity of 10. The 50% factor is intentional: `Vector` used 100% (doubled), wasting memory; a smaller factor (25%) would increase resize frequency, making `add()` more expensive on average. It offers O(1) random access via `get()` — a single array load that brings ~16 adjacent references into the same cache line (64 bytes). This cache locality means iterating 100K elements may generate zero cache misses. Insertions/removals in the middle are O(n) due to `System.arraycopy()` element shifting.
- **LinkedList** — Backed by a doubly-linked list where each element is a `Node` object with `prev` and `next` pointers. On a 64-bit JVM with compressed OOPs, each Node consumes ~28 bytes (12-byte header + three 4-byte references + 4-byte padding), plus the element itself — roughly 7x the per-element memory of `ArrayList`. 1M elements = ~28 MB of Node overhead vs ~4 MB for an `ArrayList`. It offers O(1) insertions and deletions at either end or the middle when using a `ListIterator`, but O(n) for positional access (traverses from whichever end is closer, halving worst case but staying O(n)). `LinkedList` implements both `List` and `Deque`, making it usable as a queue or stack.
- **Usage Guidance** — `ArrayList` is the correct default for nearly all use cases due to CPU cache locality and contiguous memory. `LinkedList` should only be considered when doing frequent insertions and deletions at the beginning of the list and `ArrayDeque` cannot be used.

### Set Interface

- **HashSet** — Backed by a `HashMap`, offering O(1) average time complexity for `add()`, `remove()`, and `contains()` with no guarantees about iteration order. It relies on proper `hashCode()` and `equals()` implementations — failure to override both correctly is one of the most common bugs in Java.
- **LinkedHashSet** — Extends `HashSet` by maintaining a doubly-linked list running through all entries, preserving insertion order with only slightly worse performance than `HashSet`.
- **TreeSet** — Backed by a `TreeMap` (a Red-Black tree), maintaining elements in sorted order according to natural ordering or a provided `Comparator`. All operations are O(log n), and `TreeSet` does not allow null values.
- **Selection Criteria** — Choose `HashSet` for fastest performance with no ordering, `LinkedHashSet` for insertion-order iteration, and `TreeSet` for sorted iteration.

### Map Interface

- **HashMap** — The workhorse implementation, backed by an array of buckets (`Node<K,V>[]`) with a default initial capacity of 16 and a load factor of 0.75. It provides O(1) average time for `get()` and `put()`, with performance degrading only under hash collisions.
- **HashMap Optimization (Java 8+)** — Buckets exceeding 8 entries are treeified into balanced Red-Black trees (provided the table has at least 64 buckets), improving worst-case from O(n) to O(log n). This prevents hash-collision DoS attacks that affected earlier versions. Why 8? Derived from the Poisson distribution — at the default 0.75 load factor, the probability of 8 collisions in one bucket by chance is < 1 in 10 million. If a bucket reaches 8 entries, it is almost certainly from malicious hash distribution, not normal usage. The untreeify threshold is 6 (not 8), creating a gap that prevents oscillation when elements are repeatedly added and removed near the boundary.
- **LinkedHashMap** — Extends `HashMap` with a doubly-linked list running through all entries, supporting both insertion-order and access-order iteration. Access-order mode is the foundation for LRU cache implementations.
- **TreeMap** — Uses a Red-Black tree internally, maintaining keys in sorted order with O(log n) operations. It is ideal for range queries, prefix scans, and navigational operations.
- **EnumMap** — A specialized Map implementation for enum keys that uses a plain array indexed by ordinal. It delivers 2-3x better performance than `HashMap` with zero memory waste.

### Queue and Deque

- **Queue Operations** — Provides two sets of operations: one set throws exceptions on failure (`add()`, `remove()`, `element()`), and the other returns special values (`offer()` returns false, `poll()` returns null, `peek()` returns null).
- **Deque** — Extends `Queue` to support insertion and removal at both ends with methods like `addFirst()`, `addLast()`, `offerFirst()`, `offerLast()`, `pollFirst()`, and `pollLast()`.
- **ArrayDeque** — The recommended implementation for both stack and queue use cases. It is backed by a resizable array, does not allow null elements, has no per-node memory overhead, and offers better performance than `LinkedList` due to CPU cache locality.
- **PriorityQueue** — A heap-based unbounded queue that orders elements according to natural ordering or a `Comparator`. It provides O(log n) insertion and O(1) retrieval of the head element, but iterating does not produce elements in sorted order — only `poll()` returns elements in priority order. This is because the heap only maintains the invariant that the parent is smaller than both children; the rest of the array is deliberately unsorted. Maintaining full sorted order would cost O(n log n) per insertion — the heap trades full sorting for O(log n) operations, which is the optimal trade-off for a priority queue.

### Concurrent Collections

- **ConcurrentHashMap** — Replaces `Collections.synchronizedMap()` with per-bucket locking (JDK 8+): writes synchronize on individual bucket heads (`synchronized (f)`) rather than a single global mutex, so threads writing to different buckets proceed in parallel. Reads are entirely lock-free — `get()` acquires no lock because `Node.val` is `volatile`, establishing a happens-before guarantee without synchronization cost. CAS-based initialization ensures only one thread allocates the internal table on first `put()`. `computeIfAbsent()` uses a temporary `ReservationNode` to guarantee the mapping function runs at most once per key, even under concurrent calls.
- **CopyOnWriteArrayList** — Creates a fresh copy of the underlying array on every mutative operation, making reads lock-free. It is ideal for read-heavy, write-rare scenarios like event listener registries.
- **ConcurrentLinkedQueue** — A lock-free, unbounded queue that uses CAS operations internally. It is suitable for high-throughput producer-consumer patterns.
- **LinkedBlockingQueue** — A bounded blocking queue that supports the producer-consumer pattern with backpressure. Threads block when the queue is full or empty, using internal `Condition` objects.
- **ConcurrentSkipListMap** — Provides a sorted, concurrent navigable map using a skip-list data structure. It offers O(log n) operations with probabilistic balancing and no locking or blocking for reads.

### Collections Utility Class

- **Sorting and Searching** — The `java.util.Collections` class provides `sort()`, `binarySearch()`, `reverse()`, `shuffle()`, `rotate()`, and `swap()` static helper methods.
- **Synchronization Wrappers** — Methods like `synchronizedList()`, `synchronizedMap()`, and `synchronizedSet()` wrap any collection with coarse-grained synchronization, though these are largely replaced by `java.util.concurrent` alternatives.
- **Unmodifiable Wrappers** — `unmodifiableList()`, `unmodifiableMap()`, and `unmodifiableSet()` create read-only views that throw `UnsupportedOperationException` on mutation attempts. The underlying collection can still change through its original reference.
- **Factory Methods** — `singletonList()`, `emptyList()`, and `emptySet()` provide memory-efficient singletons for edge cases. Since Java 9, `List.of()`, `Set.of()`, and `Map.of()` create deeply immutable collections that are often more compact and efficient.

---

## Common Mistakes

- **ConcurrentModificationException** — Modifying a collection directly while iterating over it throws `ConcurrentModificationException` because the iterator checks a `modCount` field on each `next()` call. Use the iterator's own `remove()` method or `Collection.removeIf()` (Java 8+) to remove elements during iteration. This mistake is especially insidious in multi-threaded code where one thread modifies a collection while another iterates. The fail-fast behavior is deliberate — the alternative would be silently reading corrupted data. By crashing immediately on structural modification during iteration, the JVM prevents subtle memory corruption bugs that would be much harder to diagnose.
- **Mutable Keys in HashMap/HashSet** — Using mutable objects as keys causes hard-to-find bugs because if a key's `hashCode()` changes after insertion, the map loses track of that entry entirely — `get()` returns `null` even though the key is logically present. Always use immutable keys (`String`, `Integer`, `UUID`) or ensure `hashCode()` depends only on immutable fields.
- **Inconsistent equals() and hashCode()** — Failing to override both `equals()` and `hashCode()` consistently breaks `HashMap`, `HashSet`, and `HashTable`. The contract states that if two objects are equal according to `equals()`, they must have the same hash code. Modern IDEs and `java.util.Objects` make generating correct implementations trivial.
- **LinkedList as Default** — Choosing `LinkedList` as a default list implementation is a common anti-pattern with significant memory and performance costs. Each element requires a `Node` object (~40 bytes overhead), breaking CPU cache locality and increasing GC pressure. `ArrayList` operations at the end are O(1) amortized.
- **synchronizedMap() vs ConcurrentHashMap** — Using `Collections.synchronizedMap()` instead of `ConcurrentHashMap` degrades concurrency by locking the entire map on every read and write. `ConcurrentHashMap` achieves scalability through per-bucket locking (Java 8+), lock-free reads via `volatile` semantics, and atomic compound operations like `computeIfAbsent()` and `merge()`.
- **Capacity vs Size Confusion** — `new ArrayList<>(100).size()` is 0, not 100. The constructor parameter sets the internal array's initial capacity, not the element count. This leads to bugs where code checks `list.size() == 100` expecting the list to be "full" immediately.
- **TreeSet / TreeMap and hashCode()** — These do not use `hashCode()` or `equals()` at all. They use `compareTo()` (or `Comparator.compare()`) exclusively for element identity: two elements are "equal" when `compare(a, b) == 0`. If `compareTo` is inconsistent with `equals` (as with `BigDecimal`, where `new BigDecimal("2.0").equals(new BigDecimal("2.00"))` is false but `compareTo` returns 0), the set may silently reject elements you consider distinct.

---

## Design Considerations

- **Access Patterns** — Random access patterns favor `ArrayList` for its O(1) positional `get()` and CPU cache line efficiency. Sequential access with frequent head insertions favors `ArrayDeque` or `LinkedList` depending on memory constraints.
- **Collection Size** — For small collections (under 100 elements), algorithmic complexity is often irrelevant compared to memory overhead and allocation cost. The total number of elements matters greatly when selecting an implementation.
- **Thread Safety Granularity** — Wrapping a `HashMap` with `synchronizedMap()` provides thread safety at the cost of serializing all access, acceptable only for low-contention scenarios. For high-concurrency environments, `ConcurrentHashMap` with per-bucket locking and lock-free reads is the correct choice. Thread-safe individual operations do not compose into thread-safe compound operations — use atomic methods like `compute()`.
- **Memory Footprint** — `ArrayList` with 1 million `Integer` elements consumes roughly 4 MB while `LinkedList` consumes 40+ MB due to per-node object overhead. For numeric data, specialized primitive collections like `Int2ObjectOpenHashMap` (fastutil) can reduce memory by 50-70% compared to boxed `HashMap<Integer, V>`.
- **Collection Sizing** — Starting with default capacities forces resizing multiple times — a `HashMap` growing to 100 entries undergoes resizes at 16, 32, 64, and 128, each requiring a full rehash. The correct initial capacity is `expectedSize / loadFactor + 1`. For `ArrayList`, simply initialize with `new ArrayList<>(expectedSize)`.

**When not to use each implementation:**
- `ArrayList` for frequent head insertions — use `ArrayDeque` instead.
- `LinkedList` almost always — `ArrayDeque` or `ArrayList` is better in practice.
- `HashMap` when sorted iteration or range queries are needed — `TreeMap` provides `subMap()`.
- `HashMap` for enum keys — `EnumMap` is 2-3x faster and memory-zero waste.
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

A web server needs to cache the last 1000 recently viewed product pages per user session. The cache must evict the least-recently-viewed product when it exceeds capacity, and every cache hit must refresh the access timestamp so hot items remain in the cache longer. The implementation must be efficient enough to handle hundreds of requests per second without introducing noticeable latency.

```java
public class ProductCache<K, V> {
    private final LinkedHashMap<K, V> cache;

    public ProductCache(int maxSize) {
        this.cache = new LinkedHashMap<>() {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > maxSize;
            }
        };
    }

    public V get(K key) {
        V value = cache.remove(key);
        if (value != null) cache.put(key, value);
        return value;
    }

    public void put(K key, V value) {
        cache.put(key, value);
    }
}
```

`LinkedHashMap` with access-order enabled and `removeEldestEntry()` provides O(1) operations and automatic LRU eviction without requiring any external dependencies like Caffeine or Guava. The access order mode causes every `get()` and `put()` to reorder the internal linked list, moving the accessed entry to the tail. When `removeEldestEntry()` returns `true` (i.e., the map exceeds capacity), the eldest entry — the least-recently-accessed one at the head of the linked list — is automatically removed. This pattern is a textbook use of `LinkedHashMap` that interviewers frequently reference.

Why this approach? The example uses the default `LinkedHashMap()` constructor, which sets `accessOrder=false` (insertion order). This means calling `get()` will NOT reorder the entry — the LRU behavior relies on access-order, so the code has a subtle bug. The correct instantiation is `new LinkedHashMap<>(16, 0.75f, true)` where the third parameter enables access-order mode. The explicit `remove(key)` / `put(key)` in `get()` is an attempt to work around this, but it is unnecessary with access-order enabled — `LinkedHashMap.get()` already moves the accessed entry to the tail. The anonymous subclass to override `removeEldestEntry` is the intended extension point; `LinkedHashMap` was designed with this `protected` method specifically for subclassing. For production caches with expiration or statistics, Caffeine or Guava are better choices, but `LinkedHashMap` requires zero external dependencies.

### Scenario 2: Priority-Based Task Scheduler

A job scheduler processes tasks with different priorities. High-priority tasks must execute before low-priority ones, but tasks with the same priority must be processed in FIFO order to ensure fairness. The scheduler receives tasks from multiple concurrent producers and a single consumer thread drains them for execution. The collection must support efficient insertion and head removal without requiring full sorting.

```java
public class PriorityTaskScheduler {
    private final PriorityQueue<ScheduledTask> queue = new PriorityQueue<>(
        Comparator.comparingInt(ScheduledTask::priority)
            .thenComparing(ScheduledTask::enqueuedAt)
    );

    public void submit(ScheduledTask task) {
        queue.offer(task);
    }

    public ScheduledTask next() {
        return queue.poll();
    }
}
```

`PriorityQueue` maintains the heap property internally so `poll()` always returns the highest-priority task in O(log n) time. The composite comparator first orders by priority and then breaks ties by `enqueuedAt` timestamp, ensuring FIFO ordering within the same priority level. This approach avoids the O(n log n) cost of full sorting each time a task is submitted — instead, each insertion is O(log n) and each poll is O(log n), making it suitable for real-time scheduling. For concurrent access, this specific implementation would need to be wrapped with `PriorityBlockingQueue` or external synchronization.

Why this approach? The composite comparator is the key design choice — `thenComparing` breaks priority ties deterministically, which `PriorityQueue` alone does not guarantee (equal elements can appear in any heap order). `offer()`/`poll()` are chosen over `add()`/`remove()` because they return sentinel values (false/null) instead of throwing exceptions, which is appropriate for normal control flow in a scheduler. A sorted `ArrayList` would be simpler but would cost O(n) per insertion due to element shifting, making it unsuitable for real-time scheduling at scale.

### Scenario 3: High-Throughput Metrics Aggregator

A metrics service receives 100K events per second from multiple threads. Each event carries a metric name (string) and a numeric value. The system must aggregate these values (sum and count) per metric and produce a snapshot every minute. The implementation must scale with the number of producer threads without introducing contention on the data structure itself.

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

`ConcurrentHashMap` with `LongAdder` values provides per-bucket locking for updates, allowing true concurrent writes from hundreds of threads without contention. The `LongAdder` class uses striped counters internally, reducing CAS contention under high write loads — it is specifically designed for scenarios where the sum is read less frequently than individual values are incremented. `computeIfAbsent` is atomic, ensuring that each metric name gets exactly one `LongAdder` instance created, even when multiple threads attempt to record the same new metric simultaneously. The `snapshotAndReset()` method uses `forEachKey(1, ...)` with a parallelism threshold of 1 to traverse the map, removes each entry atomically, and captures the aggregated sum in a snapshot map that can be sent to an external monitoring system.

Why this approach? `computeIfAbsent` is chosen over `putIfAbsent` because the latter would pre-allocate a `LongAdder` on every call (even when the key exists), wasting allocation on every cache hit, and two concurrent threads could each create a `LongAdder` with only one surviving. `computeIfAbsent` only creates the value when the key is absent, and exactly once across all threads. `LongAdder` is preferred over `AtomicLong` because under high contention, `AtomicLong.incrementAndGet()` uses CAS that fails repeatedly when many threads write simultaneously, while `LongAdder` stripes writes across a cell array, reducing CAS contention by an order of magnitude. The `remove(key)` in snapshot ensures the counter is drained atomically — without it, the same counter would persist across snapshots, producing cumulative (not interval) values.

---

## Scenario-Based Questions

**Q: You are building an inventory management system where multiple warehouse workers scan items simultaneously. Each scan updates stock counts. How do you prevent lost updates without locking the entire inventory?**

A: Use `ConcurrentHashMap<String, AtomicInteger>` where each product's stock is an `AtomicInteger` that uses compare-and-swap (CAS) at the hardware level, eliminating the need for synchronized blocks or locks during single-value updates. The `ConcurrentHashMap` provides per-bucket concurrency, meaning multiple workers scanning different products can proceed in parallel without contending on the same lock. For operations involving multiple steps, use the `compute()` method which is atomic per key: `inventory.compute("SKU-123", (k, v) -> v == null ? 0 : v.decrementAndGet())`. This approach avoids the put-if-absent race condition where two threads might read the same value and overwrite each other's update, and it scales to hundreds of concurrent workers. For batch operations like end-of-day reconciliation, the `forEachKey()` or `reduceKeys()` methods allow parallel bulk processing without external synchronization.

**Q: You have a REST API that returns paginated results from a leaderboard. Users can view any page. The leaderboard changes every few seconds. How do you ensure that a user viewing page 2 does not see items they saw on page 1 (duplicates) or miss items (skips)?**

A: Use a `CopyOnWriteArrayList` for the leaderboard snapshot — sort it once per refresh interval and atomically replace the reference so that pagination always reads from a frozen snapshot without concurrent modification exceptions. Each page request uses `subList(fromIndex, toIndex)` on the current snapshot, and since the snapshot is immutable, users never see duplicates or experience shifting rankings mid-pagination that would cause items to appear on multiple pages. The stale-snapshot problem (a user might see an older leaderboard state) is acceptable for a leaderboard that updates every 5-10 seconds and is a deliberate trade-off between consistency and complexity. For stricter consistency, use a `volatile` reference to an unmodifiable list and replace the entire list atomically — this gives you a happen-before guarantee between the writer thread and all reader threads.

**Q: You are designing a rate limiter that tracks requests per user in a 1-second sliding window. The system handles 50K QPS across 10K users. How do you store and expire the request timestamps efficiently?**

A: Use `ConcurrentHashMap<String, ArrayDeque<Long>>` with per-key atomicity provided by the `compute()` method, which executes the remapping function under an internal lock scoped to the specific key's bucket. On each request, the compute function checks the current time, evicts timestamps older than 1 second from the deque's head, checks whether the remaining count exceeds the limit, and pushes the current timestamp onto the deque's tail — all within a single atomic operation. This design ensures that concurrent requests for different users proceed in parallel without contention, while concurrent requests for the same user are serialized only for that specific key. For periodic memory cleanup of idle users, schedule a background task that traverses the map and removes entries whose deques are empty, preventing unbounded growth from users who hit the rate limiter once and never return.

**Q: You are implementing an undo/redo system for a text editor. Users can perform thousands of operations. The undo stack must not grow unbounded. How do you design this with standard collections?**

A: Use `ArrayDeque` as a bounded stack by manually enforcing a capacity limit — when pushing a new operation, check if the deque has reached capacity (e.g., 100 operations), and if so, poll the oldest entry from the front of the deque before pushing the new operation onto the back. The redo stack is a separate `ArrayDeque` that is cleared on every new operation that is not an undo or redo, because any new action invalidates the current redo history and creates a new branch in the operation timeline. `ArrayDeque` is the correct choice here because it has no per-node memory overhead (unlike `LinkedList`) and offers O(1) amortized time for both `addLast()` and `pollFirst()`, with excellent CPU cache locality from its contiguous underlying array. For undo, you pop from the back of the undo stack, invert the operation, push it onto the redo stack, and apply the inverse — the cost is O(1) per operation with no garbage collection pressure from node allocation.

**Q: A microservice produces events, and consumers must process them in the order they were produced per partition (key). If consumer A dies, another consumer must take over from the last committed offset. Which collection model does this resemble?**

A: This maps directly to a sorted, concurrent map per partition, and the actual production implementation in systems like Kafka uses `ConcurrentSkipListMap` for offset tracking with O(log n) operations for offset-based lookups and range scans. Each partition maps to a sorted sequence of records indexed by monotonically increasing offsets, and the `ConcurrentNavigableMap` subMap view allows efficient range queries — for example, retrieving all records from offset X to Y without scanning the entire partition. The skip-list data structure is naturally concurrent, supporting lock-free reads and fine-grained locking for writes, which mirrors how Kafka consumers track offset positions: a sorted structure where the current offset advances sequentially and any consumer can seek to an arbitrary offset. The `ConcurrentSkipListMap` provides snapshot iterators that allow consistent traversal even while concurrent writes are happening, which is exactly what a consumer rebalancing scenario requires.

**Q: You are building a dependency resolver (like Maven or Gradle) that must detect circular dependencies. You have millions of nodes. Which collection do you use for the DFS visited set?**

A: Use `HashSet<Node>` for both the current-path set (to detect cycles by tracking the recursion stack) and the fully-processed set (to avoid revisiting subgraphs that have already been resolved). In the DFS traversal, when visiting a node, add it to the path set — if it is already present, a circular dependency exists, and the specific cycle can be reconstructed from the path set contents. After processing all children of a node, remove it from the path set (backtracking) and add it to the processed set, so that any subsequent dependency reaching this node through a different path finds it already resolved and skips the subtree. Both sets must be `HashSet` with proper `hashCode()` and `equals()` implementations, because O(1) operations are essential for a traversal involving millions of nodes — using `TreeSet` with its O(log n) operations would add tens of millions of comparisons. For extremely large graphs where memory is constrained, consider representing nodes as sequential integer identifiers and using a `BitSet` or `RoaringBitmap`, which can store millions of visited states in a few megabytes.

**Q: An ad-serving platform needs to find the top 50 ads by revenue from a stream of 10 million ad impressions per hour. Only one pass over the data is allowed. How do you maintain the top 50?**

A: Use a min-heap via `PriorityQueue` with a maximum capacity of 51 elements — for each ad impression, add the ad's revenue to the heap, and if the heap size exceeds 50, poll the minimum element to evict it. After processing all 10 million impressions, the heap contains exactly the top 50 ads by revenue, and the total time complexity is O(n log 50), which is dramatically better than O(n log n) sorting of the full dataset. For parallel processing across multiple worker threads, each thread maintains its own local top-50 heap, and these heaps are merged at the end using the same min-heap approach: iterate through all local heaps, adding each element to a global heap, and evicting the minimum when the global heap exceeds 50. This pattern — known as the "top K" or "heap with eviction" pattern — is a fundamental building block in distributed data processing frameworks like MapReduce, Spark, and Flink.

**Q: You are designing a connection pool. Threads borrow and return connections. If all connections are in use, a thread must wait. When a connection is returned, waiting threads should be notified. What collection do you use for the pool and the wait queue?**

A: Use `LinkedBlockingQueue<Connection>` with a fixed capacity for the connection pool itself — threads call `poll(timeout, unit)` to borrow a connection with a configurable timeout, preventing indefinite blocking if all connections are deadlocked or slow. Returning a connection is a simple `offer(conn)` call, which internally notifies one waiting thread via a `Condition` object (backed by `LockSupport.park()` and `unpark()`), eliminating busy-waiting and CPU waste. The `LinkedBlockingQueue` handles all the underlying thread coordination internally: it uses separate `ReentrantLock` instances for take and put operations, allowing concurrent borrowers and returners to proceed with minimal contention. For fairness, construct the queue with `true` for the fairness parameter, which uses a FIFO wait queue and prevents thread starvation under high contention — this is critical in production systems where some threads might otherwise wait indefinitely.

**Q: A collaborative editing application (like Google Docs) must merge edits from multiple users on the same document. Each edit has a timestamp and a position. How do you track the edit history for Operational Transformation (OT) or CRDT?**

A: Use a `ConcurrentSkipListMap<Position, Edit>` for the current document state, providing O(log n) insertion and ordered iteration that is naturally concurrent and supports snapshot iterators for consistent reads during live editing sessions. For the Operational Transformation algorithm, a `LinkedList<Edit>` per user (append-only) tracks each user's edit history in chronological order, and these per-user histories are merged to compute the transformation function when concurrent edits conflict. The skip-list structure is particularly well-suited here because it provides probabilistic balancing without the need for rebalancing locks (unlike a Red-Black tree), making it ideal for the highly concurrent environment of collaborative editing where dozens of users may be editing simultaneously. The `subMap()` and `headMap()` navigational methods allow efficient range queries to compute the visible document region and to determine which edits overlap spatially for conflict resolution.

**Q: You need to implement a multi-level cache (L1: local heap, L2: Redis, L3: database) with the following semantics: if L1 has the key, return immediately; if not, check L2; if not, check L3 and populate L1 and L2. Concurrent requests for the same key must not cascade — only one thread should populate. How do you coordinate?**

A: Use `ConcurrentHashMap<K, CompletableFuture<V>>` as the L1 cache — on a cache miss, call `cache.computeIfAbsent(key, k -> fetchFromL2(k))`, which atomically ensures that only the first caller executes the fetch function while subsequent callers receive the same `CompletableFuture` and block on `join()`. The fetching function checks L2 (Redis), and if that is also a miss, checks L3 (database) and populates both L2 and L1 by completing the future with the fetched value. This pattern, known as "future-based deduplication" or "coalescing cache", prevents the thundering-herd problem where thousands of concurrent requests for the same uncached key would all cascade to the database simultaneously, potentially causing an outage. For cache eviction, wrap the `ConcurrentHashMap` with a scheduled task that removes stale entries or use a library like Caffeine that provides time-based and size-based eviction on top of the same `computeIfAbsent` pattern.

**Q: A production service uses `ConcurrentHashMap.computeIfAbsent()` to lazily load configuration from a database. After a deployment, database load spikes 100x and the service becomes unresponsive. What happened?**

A: The mapping function in `computeIfAbsent()` threw an exception (likely a database timeout). When the mapping function throws, `computeIfAbsent()` does NOT cache the result — the next caller tries again. If the database is down, every caller's mapping function fires, each hitting the database and failing, amplifying load exponentially. Fix: wrap the mapping function with error handling that caches a sentinel value (e.g., `Optional.empty()`), and add a circuit breaker to stop cascading retries.

**Q: After migrating from JDK 8 to JDK 11, your application's memory usage increases noticeably for `HashMap`-heavy workloads. What changed?**

A: JDK 11 uses compact strings (Latin-1 encoding) by default, which changes `hashCode()` distribution for ASCII strings. More likely: if `-Djdk.map.althashing.threshold` was set in JDK 8 for hash-collision DoS protection, it was removed in JDK 11 (the feature was deprecated). This can increase collision rates. Investigate with a heap dump comparing bucket distribution before and after migration.

**Q: A developer replaced `ConcurrentHashMap` with `HashMap` wrapped in `Collections.synchronizedMap()` because "they are equivalent." After deployment, p99 latency increases from 10ms to 500ms. Why?**

A: `synchronizedMap()` serializes ALL read and write access on the same intrinsic lock. With 100 concurrent requests, 99 queue up on the mutex. `ConcurrentHashMap` reads are lock-free (no blocking), and writes only lock individual buckets. The synchronized map multiplies waiting time proportionally to the number of competing threads. Additionally, `synchronizedMap` lacks atomic compound operations (`computeIfAbsent`, `merge`), so the replacement likely also introduced race conditions in read-modify-write patterns.

---

## Interview Questions

**What is the difference between fail-fast and fail-safe iterators?** Fail-fast iterators (used by `ArrayList`, `HashMap`, `HashSet`) throw `ConcurrentModificationException` when the collection is structurally modified after the iterator is created, detecting changes through a `modCount` field that the collection increments on every structural modification. Fail-safe iterators (used by `ConcurrentHashMap`, `CopyOnWriteArrayList`) operate on a snapshot or a copy of the underlying data, so concurrent modifications do not affect iteration — the trade-off is memory overhead from the snapshot and potentially stale data.

**When would you use LinkedList over ArrayList?** `LinkedList` is appropriate when you need frequent insertions or deletions at the beginning or middle of the list using a `ListIterator`, or when implementing a queue or deque through its dual `List` and `Deque` interface implementations. In practice, `LinkedList` is rarely the optimal choice because each element carries roughly 40 bytes of per-node overhead from the `Node` object's `prev`, `next`, `item`, and object header fields, and the non-contiguous memory layout destroys CPU cache locality.

**How does HashMap handle collisions?** In Java 8+, `HashMap` uses separate chaining with linked lists that convert into balanced Red-Black trees when a bucket exceeds 8 entries and the table has at least 64 buckets, preventing the worst-case O(n) degradation from hash collisions and improving it to O(log n). The tree uses either `Comparable.compareTo()` (if keys implement `Comparable`) or `System.identityHashCode()` ordering, and the tree is converted back to a linked list when the bucket shrinks below 6 entries. Before Java 8, only linked lists were used and malicious input causing hash collisions could trigger O(n) map operations, making this a security vulnerability (hash-collision DoS).

**What is the difference between HashMap and ConcurrentHashMap?** `HashMap` is not thread-safe and concurrent modifications lead to race conditions, data corruption, or `ConcurrentModificationException`. `ConcurrentHashMap` uses per-bucket locking (Java 8+) where only writes to the same bucket synchronize, while reads are entirely lock-free through volatile semantics on the `Node` array reference and value fields. `ConcurrentHashMap` provides atomic compound operations like `computeIfAbsent()`, `merge()`, and `forEachKey()`, and it does not allow null keys or values.

**How does LinkedHashMap maintain insertion order?** `LinkedHashMap` extends `HashMap` and adds a doubly-linked list connecting all entries, where each entry has `before` and `after` references that thread through the map. The linked list preserves either insertion order (the order in which keys were first added) or access order (each `get()` or `put()` moves the entry to the end), configurable through the constructor's `accessOrder` parameter. Access-order mode is the foundation for LRU cache implementations via the `removeEldestEntry()` callback, which is called on every `put()` and `putAll()` to determine whether the eldest entry should be evicted.

**What is the difference between TreeSet and HashSet?** `HashSet` is backed by `HashMap` and provides O(1) average operations with no ordering guarantees, allowing a single null element. `TreeSet` is backed by `TreeMap` (a Red-Black tree) and maintains elements in sorted order using either natural ordering (elements must implement `Comparable`) or a provided `Comparator`, with O(log n) operations and no null support. Choosing between them requires knowing whether sorted iteration or raw performance is more important — `TreeSet` is the right choice when you need range queries (`subSet()`, `headSet()`, `tailSet()`), while `HashSet` is preferable when you only need uniqueness.

**What are the initial capacity and load factor of HashMap? Why are they important?** The default initial capacity is 16 and the default load factor is 0.75 — the load factor controls when the map resizes, doubling the capacity when `size > capacity * loadFactor`. A higher load factor (0.9) saves memory but increases collision probability, degrading lookup performance; a lower load factor (0.5) reduces collisions but wastes memory with empty buckets. The 0.75 default represents a time-space trade-off that works well for most workloads. For a known expected size, pre-size the map with `(expectedSize / 0.75f) + 1` to avoid all resizing overhead.

**How do you make a collection immutable or unmodifiable?** `Collections.unmodifiableList()`, `unmodifiableSet()`, and `unmodifiableMap()` create wrapper views that throw `UnsupportedOperationException` on mutation attempts, though the underlying collection can still change if another reference to it is modified. In Java 9+, `List.of()`, `Set.of()`, and `Map.of()` create deeply immutable collections with compact internal representations that cannot be changed by any means, making them preferable for API return types. For defensive copies, use `List.copyOf()` which returns an unmodifiable copy of a collection without copying if the source is already immutable.

**What is the difference between poll() and remove() in Queue?** Both methods retrieve and remove the head of the queue, but `remove()` throws `NoSuchElementException` when the queue is empty while `poll()` returns `null`. The same pattern applies to `element()` (throws) versus `peek()` (returns null) for retrieval without removal, and `add()` (throws `IllegalStateException` for bounded queues) versus `offer()` (returns false) for insertion. These two sets of operations give the caller the choice between exception-based error handling and sentinel-value-based control flow.

**How does CopyOnWriteArrayList achieve thread safety, and when should you use it?** `CopyOnWriteArrayList` achieves thread safety by creating a fresh copy of the underlying array on every mutative operation — `add()`, `set()`, and `remove()` all produce a new array — while reads operate on the current array reference without any synchronization or locking. This design is optimal for read-heavy, write-rare scenarios such as listener registries in event-driven systems where the listener list seldom changes but is iterated frequently on every event. The trade-off is that writes are O(n) with memory overhead from the old array, which persists until garbage collected, so `CopyOnWriteArrayList` must never be used for write-heavy workloads.

**Design a data structure supporting insert(key), delete(key), and getRandom() in O(1) average time.** The solution uses `HashMap<K, Integer>` (key-to-index mapping) composed with an `ArrayList<K>` (keys by index). Insert: append to list, store index in map. Delete: swap the deleted element with the last element in the list (O(1)), update the map with the swapped element's new index, remove the last entry. `getRandom()`: generate a random index within the list size. This composition pattern — using a map for O(1) lookups and a list for O(1) indexed access — is a common system design building block.

**Your service shows 100% CPU in `HashMap.get()`. Your application code does not use HashMap directly. What happened?** Many libraries (Hibernate, Spring, Tomcat) and the JDK itself (URL caching, class metadata, string interning) use `HashMap` internally. A thread-safety violation in a library — or a `ConcurrentModificationException` caught and swallowed somewhere — can create a cycle in the internal linked list, causing `get()` to loop forever. Take a thread dump, identify which `HashMap` is involved by examining the stack trace's object reference, then check the owning library's version for known concurrency bugs.

**What happens internally during HashMap resize for treeified buckets?** The resize doubles the bucket array. For treeified buckets, the tree is split into two chains based on `(hash & oldCap) == 0` — this single bit test determines whether the entry stays at the same index or moves to `index + oldCapacity`. Each resulting chain is either kept as a Red-Black tree (if chain length ≥ 6) or converted back to a linked list (if < 6). Non-tree buckets use the same bit test for redistribution.

---

## Developer Recommendations

- **Prefer ConcurrentHashMap over synchronizedMap()** — `ConcurrentHashMap` uses per-bucket locking (Java 8+) that allows concurrent reads and writes to different buckets without any contention, while `Collections.synchronizedMap()` serializes all access on the same mutex. In a benchmark with ten concurrent writers, `ConcurrentHashMap` can be 10x faster because threads rarely contend on the same bucket, and reads are entirely lock-free. The atomic `computeIfAbsent()`, `merge()`, and `forEachKey()` methods also eliminate entire categories of thread-safety bugs.
- **Use ArrayDeque over LinkedList** — `ArrayDeque` is backed by a resizable array with no per-node object overhead, whereas `LinkedList` creates a `Node` object for every single element adding ~40 bytes overhead per element. The contiguous memory layout of `ArrayDeque` provides excellent CPU cache locality — iterating 100K elements may generate zero cache misses, while the same iteration on `LinkedList` causes a cache miss on nearly every node.
- **Pre-size collections when size is known** — A `HashMap` starting at default capacity 16 and growing to 100 elements undergoes resizes at 16, 32, 64, and 128, each requiring a full rehash. The formula `new HashMap<>(expectedSize / 0.75f + 1)` pre-allocates the correct capacity, ensuring zero resizes during population. For `ArrayList`, use `new ArrayList<>(expectedSize)` to avoid incremental array copying.
- **Override equals() and hashCode() consistently** — The contract is strict: if `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must hold. Using mutable objects as keys is even more dangerous — if a key's hash code changes after insertion, the map entry becomes permanently unreachable. Modern IDEs and `java.util.Objects` (with `Objects.hash()` and `Objects.equals()`) make generating correct implementations straightforward.
- **Use EnumMap and EnumSet for enum keys** — These enum-based implementations are backed by plain arrays indexed by the enum's ordinal, completely eliminating hash code computation and maintaining insertion order. `EnumMap` outperforms `HashMap` by 2-3x in practice and uses a fraction of the memory since it stores values in a simple object array with no `Node` objects or load factor.
- **Return immutable collections from API methods** — Returning a direct reference to an internal `ArrayList` gives callers the ability to silently mutate internal state. Use `Collections.unmodifiableList(internalList)` to create a view that throws `UnsupportedOperationException` on mutation, or `List.copyOf(internalList)` (Java 10+) to create a truly independent immutable copy.
- **Use computeIfAbsent() over putIfAbsent()** — `computeIfAbsent(key, k -> new Value())` is an atomic operation where the mapping function executes at most once per key, and all concurrent callers receive the same value. The `putIfAbsent()` pattern requires a subsequent `get()` to retrieve the value, and two concurrent threads can easily execute `putIfAbsent()` with two different instances before either thread reads the result, wasting memory and computational resources.

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
| Memory grows unbounded | HashMap used as cache without eviction strategy |
| High GC pause times | `LinkedList` per-node overhead — each Node is a GC root traversal target |
| p99 latency spike after collection change | `synchronizedMap` replacing `ConcurrentHashMap` (all threads serialize on one lock) |
| Duplicate entries in `Set` | Inconsistent `equals()` / `hashCode()` |
| Intermittent test failures | `HashMap` iteration order is non-deterministic; tests assumed a specific order |

### Common Fixes

| Bug | Fix |
|-----|-----|
| ConcurrentModificationException | Use `list.removeIf(predicate)` or `iterator.remove()` |
| HashMap infinite loop (JDK 7) | Upgrade to JDK 8+ or switch to `ConcurrentHashMap` |
| Mutable key lost in HashMap | Make key immutable; ensure `hashCode()` uses only final fields |
| LinkedList O(n²) access in loop | Replace with `ArrayList` or restructure to use `ListIterator` |
| synchronizedMap contention | Replace with `ConcurrentHashMap` |
| Unbounded cache OOM | Add eviction via `LinkedHashMap.removeEldestEntry()` or switch to Caffeine |
| Stale data in concurrent map | Use `compute()` for atomic read-modify-write instead of `get`+`put` |
