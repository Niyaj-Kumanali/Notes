# Collections

---

## Overview

- **Definition:** Type-safe, resizable data structures in `System.Collections.Generic` for storing and manipulating groups of objects.
- **Why It Exists:** Provides standardized, tested implementations of fundamental data structures (dynamic arrays, hash tables, queues, stacks, linked lists) with consistent APIs and well-understood performance characteristics.
- **Key Concepts:** **`List<T>`** (dynamic array), **`Dictionary<TKey,TValue>`** (hash table), **`HashSet<T>`** (unique elements), **`Queue<T>`** (FIFO), **`Stack<T>`** (LIFO), **`LinkedList<T>`** (doubly linked), **`PriorityQueue<TElement,TPriority>`** (binary heap), **concurrent collections** (`ConcurrentDictionary`, `ConcurrentQueue`, `BlockingCollection`, `Channel<T>`), and **immutable collections**.

---

## Core Concepts

- **List\<T\> Internals:** Wraps a `T[]` internal array with default capacity 4. Grows by 2x when full (amortized O(1) per Add). `Insert` at index 0 causes O(n) shift. `_version` field incremented on every mutation to detect modification during enumeration.

```csharp
public void Add(T item)
{
    if (_size == _items.Length)
    {
        int newCapacity = _items.Length == 0 ? 4 : _items.Length * 2;
        T[] newItems = new T[newCapacity];
        Array.Copy(_items, newItems, _size);
        _items = newItems;
    }
    _items[_size++] = item;
    _version++;
}
```

- **Dictionary\<TKey,TValue\> Internals:** Uses separate chaining with a single contiguous `Entry[]` array. Hash buckets index into the entry array. Collisions resolved via linked list within the entries. Resizes when load factor exceeds ~0.75. Uses `EqualityComparer<T>.Default` for hash/equality — specialized for value types to avoid boxing.

- **HashSet\<T\> Internals:** Identical internal structure to `Dictionary<TKey,TValue>` but stores only keys (no values). Same bucket/entry array pattern with separate chaining.

- **Queue\<T\> Internals:** Circular buffer with `_head` and `_tail` indices that wrap around using modulo arithmetic. Enqueue writes at `_tail`, Dequeue reads from `_head`. Amortized O(1) for both operations.

```csharp
public void Enqueue(T item)
{
    _array[_tail] = item;
    _tail = (_tail + 1) % _array.Length;
    _size++;
}
```

- **LinkedList\<T\> Internals:** Circular doubly linked list — `_head._prev` points to the last node. O(1) insert/remove at known nodes, O(n) lookup. Each node carries forward and backward pointers, resulting in high memory overhead (3 pointers per element).

- **PriorityQueue\<TElement,TPriority\> (.NET 6+):** Binary heap (min-heap by default). `Enqueue` is O(log n), `Dequeue` is O(log n), `Peek` is O(1). Lower priority value = higher priority.

---

## Common Mistakes

- **Modifying collection during enumeration** — `foreach` over a `List<T>` while calling `Remove` throws `InvalidOperationException`. Use `RemoveAll(predicate)` or iterate backwards.
- **Ignoring default comparer for custom types** — `HashSet<Person>` uses reference equality unless `Equals`/`GetHashCode` are overridden or `IEqualityComparer<T>` is provided.
- **Capacity fragmentation** — Growing `List<T>` from default 0 to 100,000 causes ~17 resizes. Pre-allocate with `new List<T>(capacity: 100_000)`.
- **Returning internal array reference** — Exposing `_items` directly lets callers mutate internal state. Return `_items.ToArray()` (defensive copy) or `AsReadOnly()`.
- **LINQ over LinkedList\<T\>** — `linkedList.Skip(count/2).First()` is O(n) traversal. Use `Find(node)` if you have a node reference.
- **Concurrent write without synchronization** — Using `if (!dict.ContainsKey(key)) dict.Add(key, value)` has a race condition. Use `dict.GetOrAdd(key, _ => value)`.
- **Dictionary hash-collision DoS** — Using culture-sensitive string comparers can make dictionaries vulnerable to hash-collision attacks. Use `StringComparer.Ordinal`.

```csharp
// Safe: defensive copy for public API
public IReadOnlyList<User> Users => _users.AsReadOnly();
```

---

## Key Design Considerations

- **Measure, don't guess** — Theoretical complexity often loses to memory locality and GC pressure. Profile with BenchmarkDotNet. A `List<T>` of structs can outperform a `Dictionary` for small lookup sets due to cache locality.
- **Prefer immutable collections for concurrent read scenarios** — `ImmutableArray<T>`, `ImmutableDictionary<K,V>` use persistent data structures with O(1) snapshots and structural sharing. Zero contention for readers.
- **Memory pooling reduces GC stalls** — Use `ArrayPool<T>.Shared.Rent(n)` for large temporary buffers and consider `ListPool<T>` for hot paths in servers.
- **`Span<T>` and `CollectionsMarshal`** — `CollectionsMarshal.AsSpan(list)` provides direct access to a `List<T>`'s internal array without allocation. Dangerous if the list resizes after obtaining the span.
- **Struct-based collections** — `List<MyStruct>` stores structs inline (dense memory, no GC references per element). Better cache locality and less GC pressure.
- **Frozen collections (.NET 8+)** — `FrozenSet<T>` and `FrozenDictionary<K,V>` optimize read-only lookups after construction using minimal perfect hashing. Ideal for configuration data and lookup tables.

```csharp
// Low-allocation iteration using CollectionsMarshal
var span = CollectionsMarshal.AsSpan(myList);
for (int i = 0; i < span.Length; i++) span[i] = default;
```

---

## Real-World Scenarios

### Scenario 1: High-Performance Session Store
**Context:** A web server manages 500K concurrent sessions with O(1) lookups, periodic cleanup of expired sessions, and thread-safe access.

```csharp
public class SessionStore
{
    private readonly ConcurrentDictionary<string, Session> _sessions = new();
    private readonly ConcurrentDictionary<string, DateTime> _expiryTracker = new();
    private readonly PriorityQueue<string, DateTime> _expiryHeap = new();

    public void AddSession(string id, Session session, TimeSpan ttl)
    {
        var expiry = DateTime.UtcNow.Add(ttl);
        _sessions[id] = session;
        _expiryTracker[id] = expiry;
        lock (_expiryHeap) _expiryHeap.Enqueue(id, expiry);
    }

    public bool TryGetSession(string id, out Session session)
    {
        if (_sessions.TryGetValue(id, out session) && _expiryTracker.TryGetValue(id, out var expiry) && expiry > DateTime.UtcNow)
            return true;
        
        _sessions.TryRemove(id, out _); // Lazy cleanup on miss
        return false;
    }

    public async Task CleanupExpiredAsync()
    {
        while (true)
        {
            await Task.Delay(TimeSpan.FromSeconds(30));
            lock (_expiryHeap)
            {
                while (_expiryHeap.Count > 0 && _expiryHeap.Peek().Peek() < DateTime.UtcNow)
                {
                    var (id, _) = _expiryHeap.Dequeue();
                    _sessions.TryRemove(id, out _);
                    _expiryTracker.TryRemove(id, out _);
                }
            }
        }
    }
}
```

### Scenario 2: In-Memory Search Index with Inverted Lookup
**Context:** A document search feature needs indexed term lookup across 1M documents. Must handle concurrent indexing and searching.

```csharp
public class InMemorySearchIndex
{
    private readonly Dictionary<string, HashSet<int>> _invertedIndex = new();
    private readonly ReaderWriterLockSlim _rwLock = new();

    public void IndexDocument(int docId, string text)
    {
        var terms = text.Split(' ').Select(t => t.ToLowerInvariant()).Distinct();
        _rwLock.EnterWriteLock();
        try
        {
            foreach (var term in terms)
            {
                if (!_invertedIndex.TryGetValue(term, out var postings))
                    _invertedIndex[term] = postings = new HashSet<int>();
                postings.Add(docId);
            }
        }
        finally { _rwLock.ExitWriteLock(); }
    }

    public IEnumerable<int> Search(string query)
    {
        var terms = query.Split(' ').Select(t => t.ToLowerInvariant());
        _rwLock.EnterReadLock();
        try
        {
            IEnumerable<int>? results = null;
            foreach (var term in terms)
            {
                if (!_invertedIndex.TryGetValue(term, out var postings)) return Enumerable.Empty<int>();
                results = results == null ? postings : results.Intersect(postings);
            }
            return results?.ToList() ?? Enumerable.Empty<int>();
        }
        finally { _rwLock.ExitReadLock(); }
    }
}
```

### Scenario 3: Real-Time Analytics Aggregator
**Context:** A dashboard aggregates event counts per minute across 10K dimensions. Must support atomic updates from multiple threads and consistent snapshots.

```csharp
public class RollingMetricsAggregator
{
    private readonly Dictionary<string, long> _counters = new();
    private readonly ImmutableArray<string> _dimensionKeys;
    private readonly object _lock = new();

    public void RecordEvent(string dimension, long value = 1)
    {
        Interlocked.Add(ref Unsafe.As<long>(_counters.GetOrAdd(dimension, _ => 0L)), value);
    }

    public IReadOnlyDictionary<string, long> GetSnapshot()
    {
        lock (_lock) // Snapshot consistency
        {
            return _counters.ToDictionary(kvp => kvp.Key, kvp => Volatile.Read(ref kvp.Value));
        }
    }

    public void Reset()
    {
        lock (_lock) _counters.Clear();
    }
}
```

---

## Scenario-Based Questions

1. **Q: You are building an in-memory cache that stores 10M key-value pairs with automatic LRU eviction. How do you design it for O(1) operations?**
   A: Combine `Dictionary<TKey, LinkedListNode<(TKey Key, TValue Value)>>` for O(1) lookups with a `LinkedList<(TKey, TValue)>` for LRU ordering. On access, move the node to the head of the list (remove + add first). On eviction, remove the tail node and its dictionary entry. For thread safety, use `ConcurrentDictionary` + `lock` on the linked list operations, or implement a striped approach. Trade-off: memory overhead from LinkedListNode (3 pointers per entry) vs. predictable eviction behavior.

2. **Q: You have a high-throughput telemetry processing pipeline that receives 1M events/sec. Each event must be routed to one of 100 subscribers based on a key. What collection do you use?**
   A: Use a `ConcurrentDictionary<TKey, Channel<TEvent>>` where each subscriber has a dedicated channel. For the routing table itself, use `FrozenDictionary<TKey, int>` (.NET 8+) — it uses minimal perfect hashing for O(1) lookups with zero allocation once built. For dynamic routing changes, maintain a `ConcurrentDictionary` for the active mapping and periodically rebuild the `FrozenDictionary`. Avoid `List<Channel>[]` as it requires complex resizing logic.

3. **Q: You need to store a sorted list of 100K stock quotes that update frequently. `SortedSet` inserts are O(log n), but you need O(1) by ID lookup too. How do you design this?**
   A: Maintain two structures in sync: a `Dictionary<int, StockQuote>` for O(1) ID lookup and a `SortedSet<StockQuote>` (with custom `IComparer<StockQuote>` by price) for sorted iteration. On update, remove the old entry from the `SortedSet`, update the `Dictionary`, and re-insert. For thread safety, wrap all mutations in a single `lock` or use `ReaderWriterLockSlim` for read-heavy workloads. Trade-off: O(1) lookups + O(log n) sorted access at the cost of double storage.

4. **Q: You are processing a stream where you need to track the top 100 most frequent items seen so far (streaming Top-K). How do you implement this efficiently?**
   A: Use a `PriorityQueue<string, int>` (min-heap) to maintain the top K. Also maintain a `Dictionary<string, int>` for frequencies. For each item: increment its frequency, if it's in the heap, update (re-insert); if not and heap size < K, add it; if heap size == K and frequency > heap min frequency, dequeue min and enqueue the new item. Use `MinHeap` behavior (lower priority = higher priority). Memory is O(K + distinct items in the heap). This is the "lossy count" variation.

5. **Q: You are building a collection that needs to support undo/redo operations. What collection pattern do you use?**
   A: Use two `Stack<T>` (stacks) — one for undo, one for redo. Each stack stores a `Memento` (snapshot or command). On each mutation, push the previous state onto the undo stack and clear the redo stack. On undo, pop from undo and push current state onto redo. For memory efficiency, store commands (inverse operations) instead of full snapshots. Use `ImmutableStack<T>` for snapshot isolation. Trade-off: full snapshots are O(n) memory per undo but simpler; command-based is O(1) but complex.

6. **Q: You need to implement a custom collection that is structurally identical to `List<T>` but with O(1) removal of arbitrary elements. How?**
   A: Use a dynamic array but with a "hole" approach: maintain a `FreeList` of removed indices. When removing an element at index `i`, push `i` onto a stack of free slots. When adding, reuse a free slot if available; otherwise append. Track count with `_size`. Add a `_version` field for enumeration safety. For O(1) removal by value, combine with a `Dictionary<T, List<int>>` mapping values to their indices. Trade-off: slightly slower iteration (skipping holes) vs. O(1) remove.

7. **Q: You are designing a configuration system where settings are read 1000x/sec and updated once/hour. What collection do you use?**
   A: Use `ImmutableDictionary<string, object>` for lock-free reads. On update, create a new `ImmutableDictionary` via `AddRange` and atomically swap the reference with `Interlocked.Exchange`. Readers always see a consistent snapshot without any blocking. Alternatively, use `FrozenDictionary` (.NET 8+) rebuilt on each update — it trades build cost for faster reads. Trade-off: immutable collections allocate on every write, but writes are rare so this is acceptable.

8. **Q: You need to process items from a queue with priority — higher priority items should be processed first, but items of the same priority should be FIFO. What do you use?**
   A: Use `PriorityQueue<QueueItem, (int Priority, long Sequence)>` with a custom comparer that sorts by `Priority` descending, then `Sequence` ascending. Maintain a `long _sequenceCounter` (use `Interlocked.Increment`) to assign sequence numbers on enqueue. This naturally gives priority-based ordering with FIFO within the same priority level. For thread safety, wrap operations in a lock or use a channel-based approach with separate queues per priority level.

9. **Q: You are processing a graph with 1M nodes. You need to perform BFS, traversing neighbors. What collection gives the best performance?**
   A: Use a `Queue<int>` for the frontier and a `HashSet<int>` (or `BitArray` for dense IDs) for visited tracking. The adjacency list should be stored as `List<int>[]` (array of lists) for cache-friendly iteration — arrays of `List<T>` have excellent locality. For the visited set, `HashSet<int>` is O(1) but has overhead; `BitArray` uses 1 bit per node (125KB for 1M nodes) and is extremely fast. Memory: `BitArray` wins for dense graphs; `HashSet` wins for sparse ones.

10. **Q: You have a `List<T>` that 50 threads read concurrently and one thread writes. How do you make this safe without using `ConcurrentCollection`?**
    A: Use `ImmutableArray<T>` with `Volatile.Read`/`Interlocked.Exchange` for lock-free snapshots. Readers capture a reference to the current `ImmutableArray<T>` (atomic on x64), then iterate safely — the array is immutable. The writer builds a new `ImmutableArray<T>` from the current one (e.g., `current.Add(item)`), then `Interlocked.Exchange(ref _items, newArray)`. This is the copy-on-write pattern. Trade-off: writes allocate a new array, but for read-heavy workloads this is ideal.

---

## Interview Questions

1. **What is the difference between `List<T>` and `ArrayList`?**
   A: `List<T>` is a generic, type-safe collection that avoids boxing and unboxing. `ArrayList` is non-generic (stores `object`), causing boxing for value types and requiring casting. `List<T>` has value type-specific optimizations in the JIT.

2. **What is `HashSet<T>` used for?**
   A: A collection that contains unique elements with O(1) add, remove, and lookup. It uses the same internal structure as `Dictionary<TKey,TValue>` but stores only keys. Unlike `List<T>`, it does not preserve insertion order.

3. **Explain the difference between `SortedSet<T>` and `SortedList<TKey,TValue>`.**
   A: `SortedSet<T>` is a red-black tree (O(log n) for all operations). `SortedList<TKey,TValue>` is a sorted array (O(log n) search via binary search, O(n) insert/delete due to shifting). `SortedSet` has better insert/delete for large collections; `SortedList` has better cache locality and lower memory per entry.

4. **What is `FrozenDictionary<TKey,TValue>` and when should you use it?**
   A: Added in .NET 8, it's an immutable dictionary optimized for read-only lookups using minimal perfect hashing. Build once, use forever. Ideal for configuration data, lookup tables, and static mappings that are defined at startup and never change.

5. **How does `PriorityQueue<TElement, TPriority>` work internally?**
   A: It uses a binary min-heap stored as an array. The heap property ensures each parent has higher priority (lower numeric value) than its children. `Enqueue` is O(log n), `Dequeue` is O(log n), `Peek` is O(1).

6. **What is the difference between `IReadOnlyList<T>` and `ReadOnlyCollection<T>`?**
   A: `IReadOnlyList<T>` is an interface that guarantees no mutation methods — a contract. `ReadOnlyCollection<T>` is a wrapper class that implements `IReadOnlyList<T>` by wrapping an `IList<T>` and throwing on mutation attempts. The interface avoids allocation; the wrapper adds a small overhead.

7. **Explain the `_version` field in `List<T>`.**
   A: It's an `int` field incremented on every mutation (Add, Insert, Remove, Clear). The `List<T>.Enumerator` checks it on each `MoveNext()` call. If it changed during enumeration, `InvalidOperationException` is thrown, preventing undetected concurrent modification.

8. **What is the difference between `Dictionary` and `ConcurrentDictionary`?**
   A: `Dictionary<TKey,TValue>` is not thread-safe — concurrent reads are OK only if no writes occur. `ConcurrentDictionary<TKey,TValue>` uses striped locking for thread-safe reads and writes, and provides atomic operations like `GetOrAdd` and `AddOrUpdate`.

9. **How does `Queue<T>` implement its circular buffer?**
   A: It uses a circular array with `_head` and `_tail` indices that wrap via modulo: `_tail = (_tail + 1) % _array.Length`. `Enqueue` writes at `_tail`, `Dequeue` reads from `_head`. Both operations are O(1). When capacity is reached, the array grows (typically 2x) and elements are copied to the new array with `_head` reset to 0.

10. **What is the difference between `Cast<T>` and `OfType<T>` in LINQ?**
    A: `Cast<T>` attempts to cast every element to `T` and throws `InvalidCastException` on mismatch. `OfType<T>` filters to only elements of type `T`, silently skipping incompatible elements. Use `Cast` when all elements are guaranteed to be of type `T`; use `OfType` when some elements may be of a different type.

---

## Developer Recommendations

- **Prefer `IReadOnlyList<T>` over `List<T>` for public APIs** — Exposing `List<T>` allows callers to modify the collection, breaking encapsulation. `IReadOnlyList<T>` signals intent and prevents mutation. Internally you can still use `List<T>` for performance. Use `AsReadOnly()` to create a wrapper without copying.

- **Use `ConcurrentDictionary.GetOrAdd` instead of check-then-add pattern** — The naive `if (!dict.ContainsKey(key)) dict.Add(key, value)` has a race condition in multithreaded code. `GetOrAdd(key, _ => value)` atomically adds only if the key doesn't exist, returning the existing or newly added value.

- **Pre-size collections when capacity is known** — `new List<T>(100_000)` pre-allocates the internal array, avoiding ~17 resizes when growing from the default capacity of 4. Each resize copies all elements. Pre-allocating reduces GC pressure and improves throughput for large collections.

- **Use `CollectionsMarshal.AsSpan` for performance-critical `List<T>` access** — In hot paths, `CollectionsMarshal.AsSpan(list)` returns a `Span<T>` over the list's internal array, avoiding bounds checking and enabling zero-allocation iteration. Dangerous: the span becomes invalid if the list resizes. Only use in controlled, non-mutating scopes.

- **Prefer `ImmutableArray<T>` over `List<T>` for lock-free read concurrency** — `ImmutableArray<T>` is a struct wrapping a `T[]` with copy-on-write semantics. Multiple threads can read concurrently without any synchronization. Writes create a new array. Ideal for configuration data, cached lookups, and snapshot patterns.

- **Return `IEnumerable<T>` with caution for API contracts** — Callers may enumerate multiple times, causing repeated execution. Use `IReadOnlyCollection<T>` or `IReadOnlyList<T>` to signal materialized data. If returning a LINQ query, document that it's lazily evaluated.

- **Prefer `ArrayPool<T>` over allocating temporary arrays** — `ArrayPool<T>.Shared.Rent(n)` reuses arrays from a pool, reducing GC allocations. Return with `Return(array, clearArray: true)` if the array contains sensitive data. Ideal for `byte[]` buffers in high-throughput scenarios.

- **Use `FrozenSet<T>` / `FrozenDictionary<TKey,TValue>` for static lookup tables (.NET 8+)** — These use minimal perfect hashing for the fastest possible read performance at the cost of expensive construction. Build once during startup, use for the application's lifetime.

---

## Performance by Collection

| Collection | Access | Search | Insert | Delete |
|---|---|---|---|---|
| `T[]` | O(1) | O(n) | O(n) | O(n) |
| `List<T>` | O(1) | O(n) | O(1)* | O(n) |
| `Dictionary<K,V>` | O(1)* | O(1)* | O(1)* | O(1)* |
| `HashSet<T>` | — | O(1)* | O(1)* | O(1)* |
| `SortedDictionary<K,V>` | O(log n) | O(log n) | O(log n) | O(log n) |
| `SortedSet<T>` | — | O(log n) | O(log n) | O(log n) |
| `Stack<T>` | O(1) peak | O(n) | O(1)* | O(1)* |
| `Queue<T>` | O(1) peek | O(n) | O(1)* | O(1)* |
| `LinkedList<T>` | O(n) | O(n) | O(1)** | O(1)** |
| `PriorityQueue<T,P>` | O(1) peek | O(n) | O(log n) | O(log n) |

\*Amortized; \*\*At known node; ~Hash collisions degrade to O(n)
