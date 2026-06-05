# Java Collections Framework

---

## Overview

- **Definition:** The Java Collections Framework (JCF) is a unified architecture for storing, retrieving, manipulating, and communicating groups of objects. It provides interfaces like `List`, `Set`, `Map`, `Queue`, and `Deque`, along with their implementations and utility algorithms in the `Collections` class.

- **Why It Exists:** Before JCF (Java 1.2), Java had `Vector`, `Hashtable`, `Stack`, and `Arrays` with inconsistent APIs and no common algorithms. JCF standardized everything with:
  - **Interfaces** that separate contracts from implementations
  - **Reusable algorithms** that work across all implementations
  - **Interoperability** so all collections speak the same API language

- **Core Interfaces:**
  - **Collection** — root interface for List, Set, Queue
  - **List** — ordered, allows duplicates, positional access
  - **Set** — no duplicates, at most one null
  - **Map** — key-value pairs, no duplicate keys
  - **Queue** — for holding elements prior to processing (typically FIFO)
  - **Deque** — double-ended queue, insert/remove at both ends

---

## Collection Hierarchy

```
                      Iterable
                          |
                     Collection
                    /    |    \
                 List   Set   Queue
                 / \   / \     |
          ArrayList  HashSet  Deque
          LinkedList TreeSet  ArrayDeque
                    LinkedHashSet

                     Map
                   /  |  \
              HashMap TreeMap
           LinkedHashMap
```

---

## List Interface

- **Definition:** An ordered collection (sequence) that allows duplicates and provides positional access.

- **ArrayList:**
  - Backed by a resizable array
  - **Get:** O(1), **Add (end):** O(1) amortized, **Add (mid):** O(n), **Remove:** O(n)
  - Grows by 1.5x when full (`oldCapacity + (oldCapacity >> 1)`)
  - Default initial capacity is 10
  - Good for: random access, iteration, adding/removing at end

- **LinkedList:**
  - Backed by a doubly-linked list
  - **Get:** O(n), **Add (end):** O(1), **Add (mid):** O(1), **Remove:** O(1)
  - Each element is a Node object with prev/next pointers (~40 bytes per element overhead)
  - Implements both List and Deque
  - Good for: frequent insert/delete at head or middle, implementing queue/stack

```java
List<String> list = new ArrayList<>();
list.add("Apple");
list.add("Banana");
list.add(0, "First");         // Insert at position
String fruit = list.get(1);   // Random access O(1)
list.remove(0);               // Remove by index
```

---

## Set Interface

- **Definition:** A collection with no duplicates. At most one null element.

- **HashSet:**
  - Backed by a HashMap
  - Unordered, O(1) average for add/remove/contains
  - Requires proper `hashCode()` and `equals()` implementation

- **LinkedHashSet:**
  - Extends HashSet, maintains insertion order via linked list
  - Slightly slower than HashSet, but predictable iteration order

- **TreeSet:**
  - Backed by a TreeMap (Red-Black tree)
  - Sorted order (natural or Comparator), O(log n) operations
  - Does not allow null values

```java
Set<String> hashSet = new HashSet<>();
hashSet.add("Java");
hashSet.add("Python");
hashSet.add("Java");           // Duplicate — not added

Set<String> treeSet = new TreeSet<>();
treeSet.add("C");
treeSet.add("A");
treeSet.add("B");              // Iterates as A, B, C (sorted)
```

---

## Map Interface

- **Definition:** An object that maps keys to values. No duplicate keys.

- **HashMap:**
  - Backed by an array of buckets (Node<K,V>[])
  - Default initial capacity: 16, load factor: 0.75
  - O(1) average for get/put
  - Treeifies buckets when they reach 8 nodes (and table >= 64)

- **LinkedHashMap:**
  - Maintains insertion order or access order (for LRU caches)
  - Slightly more memory than HashMap

- **TreeMap:**
  - Red-Black tree implementation
  - Sorted by keys (natural order or Comparator)
  - O(log n) for get/put

```java
Map<String, Integer> scores = new HashMap<>();
scores.put("Alice", 95);
scores.put("Bob", 87);
int score = scores.get("Alice");        // 95
for (Map.Entry<String, Integer> entry : scores.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
```

---

## Queue and Deque

- **Queue:** Typically FIFO order. Operations:
  - `add(e)` / `offer(e)` — insert (offer returns false if full)
  - `remove()` / `poll()` — retrieve and remove head (poll returns null if empty)
  - `element()` / `peek()` — retrieve head without removing (peek returns null if empty)

- **Deque:** Double-ended queue. Operations at both ends:
  - `addFirst(e)`, `addLast(e)` — throws if full
  - `offerFirst(e)`, `offerLast(e)` — returns boolean
  - `removeFirst()`, `removeLast()` — throws if empty
  - `pollFirst()`, `pollLast()` — returns null if empty

- **ArrayDeque:** Recommended implementation for both stack and queue. Faster than LinkedList, no nulls allowed, resizable array.

```java
Queue<String> queue = new ArrayDeque<>();
queue.offer("First");
queue.offer("Second");
String head = queue.poll();    // "First"

Deque<String> stack = new ArrayDeque<>();
stack.push("Bottom");
stack.push("Top");
String top = stack.pop();      // "Top"
```

---

## Concurrent Collections

- **Definition:** Thread-safe collections from `java.util.concurrent` package designed for concurrent access.

- **ConcurrentHashMap:** Per-bucket locking (JDK 8+), CAS for initialization, no null keys/values. Replaces synchronized HashMap.

- **CopyOnWriteArrayList:** Every mutation creates a new copy of the array. Read operations are lock-free. Best for read-heavy, write-rare scenarios.

- **ConcurrentLinkedQueue:** Lock-free queue using CAS operations. Unbounded, thread-safe.

- **LinkedBlockingQueue:** Bounded blocking queue backed by linked nodes. Supports producer-consumer pattern with backpressure.

- **ConcurrentSkipListMap:** Sorted concurrent map using skip-list data structure.

```java
Map<String, String> cache = new ConcurrentHashMap<>();
cache.put("key", "value");
String val = cache.get("key");  // Thread-safe without explicit synchronization
```

---

## Collections Utility Class

- **Definition:** `java.util.Collections` provides static helper methods that operate on or return collections.

- **Sorting and Searching:** `sort()`, `binarySearch()`, `reverse()`, `shuffle()`

- **Synchronization Wrappers:** `synchronizedList()`, `synchronizedMap()`, etc.

- **Unmodifiable Wrappers:** `unmodifiableList()`, `unmodifiableMap()`, etc.

- **Factories:** `singletonList()`, `emptyList()`

```java
List<String> list = new ArrayList<>(List.of("C", "A", "B"));
Collections.sort(list);                              // [A, B, C]
List<String> readOnly = Collections.unmodifiableList(list);
// readOnly.add("X");  // Throws UnsupportedOperationException
```

---

## Common Mistakes

- **Modifying collection during iteration** — causes `ConcurrentModificationException`. Use `Iterator.remove()` or `Collection.removeIf()` instead.
- **Mutable objects in HashSet** — if hash code changes after insertion, `contains()` fails. Use immutable keys.
- **Not overriding equals/hashCode** — HashMap and HashSet won't work correctly. Override both consistently.
- **LinkedList for most use cases** — ArrayList is better for 99% of list scenarios. LinkedList has high memory overhead.
- **Synchronized wrapper instead of ConcurrentHashMap** — `Collections.synchronizedMap()` locks entire map. Use `ConcurrentHashMap`.
- **HashMap without sizing** — default capacity 16 causes many resizes. Pre-size with `expectedSize / 0.75f + 1`.
- **PriorityQueue iteration expecting sorted order** — PriorityQueue only guarantees head is correct on `poll()`. Don't iterate directly.
- **Returning internal collection references** — callers can modify internals. Return `Collections.unmodifiableList()`.

---

## Real-World Scenarios

### Scenario 1: LRU Cache in a Web Application

A web server needs to cache the last 1000 recently viewed product pages per user session. The cache must evict the least-recently-viewed product when it exceeds capacity.

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

`LinkedHashMap` with access-order and `removeEldestEntry()` gives O(1) operations and automatic LRU eviction without external dependencies.

### Scenario 2: Priority-Based Task Scheduler

A job scheduler processes tasks with different priorities. High-priority tasks must execute before low-priority ones, but tasks with the same priority are processed FIFO.

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

`PriorityQueue` maintains the heap property so `poll()` always returns the highest-priority task. The FIFO tiebreaker ensures fairness among same-priority tasks.

### Scenario 3: High-Throughput Metrics Aggregator

A metrics service receives 100K events/second from multiple threads. Each event has a metric name and value. The system aggregates (sum/count) metrics per minute.

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

`ConcurrentHashMap` with `LongAdder` values provides per-bucket locking (not global), allowing true concurrent writes. `computeIfAbsent` is atomic.

---

## Scenario-Based Questions

1. **Q: You are building an inventory management system where multiple warehouse workers scan items simultaneously. Each scan updates stock counts. How do you prevent lost updates without locking the entire inventory?**
   A: Use `ConcurrentHashMap<String, AtomicInteger>` where each product's stock is an `AtomicInteger`. The key design: `ConcurrentHashMap` provides per-bucket concurrency, and `AtomicInteger` uses CAS for the increment — no global lock. For bulk operations, use `compute()` which is atomic per key: `inventory.compute("SKU-123", (k, v) -> v == null ? 0 : v.decrementAndGet())`. This avoids `put-if-absent` races and scales to hundreds of concurrent workers.

2. **Q: You have a REST API that returns paginated results from a leaderboard. Users can view any page. The leaderboard changes every few seconds. How do you ensure that a user viewing page 2 doesn't see items they saw on page 1 (duplicates) or miss items (skips)?**
   A: Use a `CopyOnWriteArrayList` for the leaderboard snapshot. Sort once per refresh interval, create a new list, and replace the reference. Pagination reads from a frozen snapshot — no concurrent modification. Each page request uses `subList(fromIndex, toIndex)` on the snapshot. The stale-snapshot problem is acceptable for a leaderboard that updates every 5-10 seconds. For stricter consistency, use `Collections.unmodifiableList()` and replace the entire list atomically with `volatile` reference.

3. **Q: You're designing a rate limiter that tracks requests per user in a 1-second sliding window. The system handles 50K QPS across 10K users. How do you store and expire the request timestamps efficiently?**
   A: Use `ConcurrentHashMap<String, ArrayDeque<Long>>` with per-key locking via `compute()`. For each request: `timestamps.compute(userId, (k, deque) -> { if (deque == null) deque = new ArrayDeque<>(); long now = System.currentTimeMillis(); while (!deque.isEmpty() && deque.peekFirst() < now - 1000) deque.pollFirst(); if (deque.size() >= RATE_LIMIT) throw RateExceededException(); deque.addLast(now); return deque; })`. The `compute()` method is atomic per key, so concurrent requests for different users don't contend. For memory cleanup, a scheduled task removes stale user entries from the map.

4. **Q: You are implementing an undo/redo system for a text editor. Users can perform thousands of operations. The undo stack must not grow unbounded. How do you design this with standard collections?**
   A: Use `ArrayDeque` as a circular buffer bounded by capacity (e.g., 100 operations). For undo: `ArrayDeque<EditOperation> undoStack = new ArrayDeque<>(100)`. When pushing a new operation: if the stack is full, poll the oldest entry first (`if (undoStack.size() >= 100) undoStack.pollFirst()`), then push. The redo stack is a separate `ArrayDeque` that is cleared whenever a new operation is performed (invalidating the redo history). `ArrayDeque` is preferred over `LinkedList` because it has no per-node memory overhead and is faster for both ends.

5. **Q: A microservice produces events, and consumers must process them in the order they were produced per partition (key). If consumer A dies, another consumer must take over from the last committed offset. Which collection model does this resemble?**
   A: This is a `ConcurrentLinkedDeque` per partition, but the actual implementation uses `ConcurrentSkipListMap` for offset tracking. The key insight: each partition is an ordered, thread-safe sequence. `ConcurrentSkipListMap<PartitionId, TreeMap<Offset, Record>>` provides O(log n) operations for offset-based lookups. The `ConcurrentNavigableMap` subMap view allows efficient range queries (e.g., records from offset X to Y). This is exactly how Kafka consumers track offsets internally — a sorted, concurrent map.

6. **Q: You are building a dependency resolver (like Maven or Gradle) that must detect circular dependencies. You have millions of nodes. Which collection do you use for the DFS visited set?**
   A: Use `HashSet<Node>` for the current-path set (to detect cycles) and `HashSet<Node>` for the fully-processed set (to avoid revisiting). The DFS is: when visiting a node, add to path set; if already in path set, circular dependency detected; after processing all children, remove from path set and add to processed set. Both sets must be `HashSet` with proper `hashCode()`/`equals()` on the node type — O(1) operations are critical for millions of nodes. If the graph is very large and memory is constrained, consider a `BitSet` or `RoaringBitmap` for integer-indexed nodes.

7. **Q: An ad-serving platform needs to find the top 50 ads by revenue from a stream of 10 million ad impressions per hour. Only one pass over the data is allowed. How do you maintain the top 50?**
   A: Use a min-heap via `PriorityQueue` with capacity 51. For each ad impression: add the ad's revenue to the heap; if the heap size exceeds 50, poll the minimum. After processing all data, the heap contains the top 50 items in O(n log 50) time — much better than O(n log n) for sorting all 10M items. For concurrent processing, each thread maintains its own top-50 heap and they are merged at the end using the same approach. This is the standard "top K" pattern in distributed systems.

8. **Q: You are designing a connection pool. Threads borrow and return connections. If all connections are in use, a thread must wait. When a connection is returned, waiting threads should be notified. What collection do you use for the pool and the wait queue?**
   A: Use `LinkedBlockingQueue<Connection>` for the pool itself with a fixed capacity. Threads call `poll(timeout, unit)` to borrow with a timeout, and `offer(conn)` to return. The blocking queue handles all the thread coordination internally — threads block via `Condition` objects (park/unpark) with zero busy-wait. For fairness, use `new LinkedBlockingQueue<>(true)` for fair ordering. This is far simpler than implementing wait/notify manually and handles interruption, timeout, and thread safety correctly.

9. **Q: A collaborative editing application (like Google Docs) must merge edits from multiple users on the same document. Each edit has a timestamp and a position. How do you track the edit history for Operational Transformation (OT) or CRDT?**
   A: Use a `TreeSet<Edit>` with a custom comparator by position and then by timestamp. Each user's edits are inserted at the correct position. For CRDT-based approaches, use `ConcurrentSkipListMap<Position, Edit>` which provides O(log n) insertion and iteration. The skip-list data structure is naturally concurrent and supports snapshot iterators. For the actual OT algorithm, a `LinkedList<Edit>` per user (append-only) tracks the edit history, and a `TreeMap<Integer, Edit>` maintains the current document state by character position.

10. **Q: You need to implement a multi-level cache (L1: local heap, L2: Redis, L3: database) with the following semantics: if L1 has the key, return immediately; if not, check L2; if not, check L3 and populate L1 and L2. Concurrent requests for the same key must not cascade — only one thread should populate. How do you coordinate?**
    A: Use `ConcurrentHashMap<K, CompletableFuture<V>>` as the L1 cache. On request: `cache.computeIfAbsent(key, k -> fetchFromL2(k))`. The `computeIfAbsent` is atomic — only the first caller executes the fetching function. Subsequent callers get the same `CompletableFuture` and `join()` on it. The fetching function checks L2, then L3 if needed, and completes the future. This pattern (sometimes called "future-based deduplication" or "coalescing cache") prevents the thundering-herd problem without explicit locking. For eviction, wrap with Caffeine or use scheduled computation of stale keys.

---

## Interview Questions

1. **What is the difference between fail-fast and fail-safe iterators?**
   A: Fail-fast iterators (e.g., `ArrayList`, `HashMap`) throw `ConcurrentModificationException` if the collection is structurally modified after the iterator is created. They track changes via a `modCount` field. Fail-safe iterators (e.g., `ConcurrentHashMap`, `CopyOnWriteArrayList`) operate on a snapshot or a copy of the underlying data, so modifications don't affect iteration. Fail-safe iterators trade memory/consistency for safety.

2. **When would you use LinkedList over ArrayList?**
   A: `LinkedList` is better when you need frequent insertions/deletions at the beginning or middle of the list, or when implementing a queue/deque (it implements both `List` and `Deque`). In practice, `LinkedList` is rarely the right choice because the per-node memory overhead (~40 bytes) is high and modern `ArrayList` operations are fast due to CPU cache locality. Use `ArrayDeque` for queue/stack scenarios instead.

3. **How does HashMap handle collisions?**
   A: In Java 8+, `HashMap` uses separate chaining with linked lists that convert to balanced trees (red-black) when a bucket exceeds 8 entries (provided the table has at least 64 buckets). Tree nodes use `compareTo()` (if keys implement `Comparable`) or `hashCode()` ordering. This prevents O(n) degradation from hash collisions — worst case becomes O(log n) instead of O(n). Before Java 8, only linked lists were used.

4. **What is the difference between HashMap and ConcurrentHashMap?**
   A: `HashMap` is not thread-safe — concurrent modifications cause race conditions or `ConcurrentModificationException`. `ConcurrentHashMap` is thread-safe using per-bucket locking (Java 8+) or segment-based locking (Java 7). `ConcurrentHashMap` uses CAS for lock-free reads and synchronized blocks only for writes that affect the same bucket. It also provides atomic `computeIfAbsent()`, `merge()`, and `forEachKey()` operations. `ConcurrentHashMap` does not allow null keys or values.

5. **How does LinkedHashMap maintain insertion order?**
   A: `LinkedHashMap` extends `HashMap` and adds a doubly-linked list connecting the entries. Each `Entry` has `before` and `after` references. The linked list preserves either insertion order or access order (configured via constructor). Access order is used for LRU cache implementation via the `removeEldestEntry()` method. The linked list adds O(1) overhead per operation and uses slightly more memory than plain `HashMap`.

6. **What is the difference between TreeSet and HashSet?**
   A: `HashSet` is backed by `HashMap` and offers O(1) average operations but no ordering guarantees. `TreeSet` is backed by `TreeMap` (Red-Black tree) and maintains sorted order (natural or `Comparator`), with O(log n) operations. `HashSet` allows a null element; `TreeSet` does not. `TreeSet` requires elements to implement `Comparable` or a `Comparator` to be provided. `HashSet` requires proper `hashCode()`/`equals()`.

7. **What is the initial capacity and load factor of HashMap? Why are they important?**
   A: Default initial capacity is 16; default load factor is 0.75. The load factor controls when the map resizes — when `size > capacity × loadFactor`, the capacity doubles. A higher load factor (0.9) saves memory but increases collision probability. A lower load factor (0.5) reduces collisions but wastes memory. The 0.75 default is a trade-off between time and space. If you know the expected size, pre-size: `expectedSize / 0.75f + 1` to avoid resizing.

8. **How do you make a collection immutable or unmodifiable?**
   A: `Collections.unmodifiableList()`, `unmodifiableSet()`, `unmodifiableMap()`, etc. These create a view that throws `UnsupportedOperationException` on mutation. In Java 9+, `List.of()`, `Set.of()`, `Map.of()` create immutable collections directly. The difference: unmodifiable wrappers are views — the underlying collection can still change; `List.of()` is deeply immutable. For deep immutability, always return copies or use `List.copyOf()`.

9. **What is the difference between poll() and remove() in Queue?**
   A: Both retrieve and remove the head of the queue. `remove()` throws `NoSuchElementException` if the queue is empty; `poll()` returns `null`. The same distinction applies to `element()` vs `peek()` — `element()` throws, `peek()` returns null. The `offer()`/`add()` pair has a similar pattern: `add()` throws `IllegalStateException` if the queue is full (bounded queues), `offer()` returns `false`.

10. **How does CopyOnWriteArrayList achieve thread safety, and when should you use it?**
    A: `CopyOnWriteArrayList` achieves thread safety by creating a new copy of the underlying array on every mutative operation (add, set, remove). Reads are lock-free and never blocked — they operate on the current array reference. This makes it ideal for read-heavy, write-rare scenarios like listener lists in event systems. The trade-off is that writes are O(n) and memory-intensive (the old array is garbage collected). Never use it for write-heavy workloads.

---

## Developer Recommendations

- **Prefer `ConcurrentHashMap` over `Collections.synchronizedMap()`** — `ConcurrentHashMap` uses per-bucket locking, allowing concurrent reads and writes to different buckets. `synchronizedMap()` locks the entire map, serializing all access and reducing throughput on multi-core systems. For a map with 10 concurrent writers, `ConcurrentHashMap` can be 10x faster.

- **Use `ArrayDeque` instead of `LinkedList` for stack/queue** — `ArrayDeque` is backed by a resizable array with no per-node overhead. `LinkedList` creates a `Node` object for every element (~40 bytes overhead). `ArrayDeque` also has better CPU cache locality since elements are contiguous in memory. For stacks and queues, `ArrayDeque` is always the better choice.

- **Pre-size collections when the size is known** — Starting with default capacity (e.g., 16 for `HashMap`) causes multiple resizes: 16 → 32 → 64 → 128 for 100 elements. Each resize is O(n). Use `new HashMap<>(expectedSize / 0.75f + 1)` to avoid resizing entirely. For `ArrayList`, `new ArrayList<>(expectedSize)` avoids incremental array copying.

- **Override `equals()` and `hashCode()` consistently for map/set keys** — If two objects are logically equal but have different hash codes, they'll be stored as separate entries. If an object's hash code changes after insertion (mutable key), the map/set will lose track of it. Use immutable keys (e.g., `String`, `Integer`, `UUID`) or ensure `hashCode()` depends only on immutable fields.

- **Use `EnumMap` and `EnumSet` over `HashMap`/`HashSet` for enum keys** — `EnumMap` is backed by a plain array indexed by the enum ordinal. It's faster (no hashCode computation), more memory-efficient, and maintains natural enum order. For `Enum` keys, `EnumMap` outperforms `HashMap` by 2-3x with zero memory waste.

- **Return immutable or unmodifiable collections from API methods** — Returning a reference to an internal `ArrayList` allows callers to modify your internal state. Always return `Collections.unmodifiableList(internalList)` or `List.copyOf(internalList)`. This prevents callers from adding/removing elements and makes the API contract clear. The cost is a single wrapper object creation.

- **Use `computeIfAbsent()` over `putIfAbsent()` for atomic lazy initialization** — `map.computeIfAbsent(key, k -> new Value())` is atomic — the function runs at most once per key even with concurrent callers. `putIfAbsent()` followed by `get()` is not atomic — multiple threads may create redundant instances. This is critical in caching scenarios to prevent the thundering-herd problem.
