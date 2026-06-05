# Collections

## 1. Executive Summary

Collections in C# provide type-safe, resizable data structures for storing and manipulating groups of objects. The `System.Collections.Generic` namespace houses List<T>, Dictionary<TKey,TValue>, HashSet<T>, Queue<T>, Stack<T>, LinkedList<T>, and more. Understanding their internal mechanics, algorithmic complexity, and memory layout is essential for writing performant and scalable applications.

## 2. Core Theory

Collections are broadly classified into:
- **Lists**: `List<T>`, `ArrayList` (non-generic) — ordered, indexable, dynamically resizable arrays.
- **Dictionaries**: `Dictionary<TKey,TValue>`, `SortedDictionary<TKey,TValue>`, `SortedList<TKey,TValue>` — key-value storage with fast lookup.
- **Sets**: `HashSet<T>`, `SortedSet<T>` — unique element storage.
- **Queues**: `Queue<T>`, `PriorityQueue<TElement,TPriority>` (.NET 6+) — FIFO and priority-based.
- **Stacks**: `Stack<T>` — LIFO access.
- **Linked Lists**: `LinkedList<T>` — doubly linked list.
- **Concurrent Collections**: `ConcurrentDictionary<TKey,TValue>`, `ConcurrentQueue<T>`, `ConcurrentStack<T>`, `BlockingCollection<T>`, `Channel<T>` — thread-safe variants.

Key interface hierarchy: `IEnumerable<T>` -> `ICollection<T>` -> `IList<T>`, `IDictionary<TKey,TValue>`, `ISet<T>`, `IReadOnlyCollection<T>`, `IReadOnlyList<T>`.

## 3. Under-the-Hood Deep Dive

### List<T> Internals

```csharp
// Conceptual internal layout of List<T>
internal struct List<T>
{
    private T[] _items;        // Internal array, allocated with default capacity (4)
    private int _size;         // Number of elements logically in the list
    private int _version;      // Incremented on every mutation (enumeration version check)

    public void Add(T item)
    {
        if (_size == _items.Length)
        {
            // Grow: 2x if < 64 items, then ~1.5x (internal logic varies by runtime)
            int newCapacity = _items.Length == 0 ? 4 : _items.Length * 2;
            T[] newItems = new T[newCapacity];
            Array.Copy(_items, newItems, _size);
            _items = newItems;
        }
        _items[_size++] = item;
        _version++;
    }
}
```

- Capacity doubling causes O(n) copy; amortized O(1) per Add.
- Avoid `Insert` at index 0 — causes O(n) shift.
- `TrimExcess()` reduces capacity to match size.

### Dictionary<TKey,TValue> Internals

```csharp
// Simplified internal layout
internal struct Dictionary<TKey, TValue>
{
    private struct Entry
    {
        public int hashCode;    // Lower 31 bits of hash code (or -1 for free slot)
        public int next;        // Index of next entry in chain (-1 for end)
        public TKey key;
        public TValue value;
    }

    private int[] _buckets;     // Hash buckets (index into _entries)
    private Entry[] _entries;   // Entry array
    private int _count;         // Number of entries used
    private int _freeList;      // Index of first free slot
    private int _freeCount;     // Number of free slots

    // Collision resolution: separate chaining via _buckets -> linked list in _entries
    // Resize when load factor exceeds ~0.75
}
```

- Open addressing is NOT used; it uses separate chaining with a single contiguous array.
- `EqualityComparer<T>.Default` uses `GetHashCode()` and `Equals()`.
- For value type keys, avoid boxing by using `EqualityComparer<T>.Default` which specializes for value types.

### HashSet<T> Internals

- Identical internal structure to `Dictionary<TKey,TValue>` but stores only keys (no values).
- Uses the same bucket/entry array pattern.

### Queue<T> Internals

```csharp
internal struct Queue<T>
{
    private T[] _array;
    private int _head;          // Index of first element
    private int _tail;          // Index of next free slot
    private int _size;          // Number of elements

    public void Enqueue(T item)
    {
        _array[_tail] = item;
        _tail = (_tail + 1) % _array.Length;  // Circular wrap
        _size++;
    }

    public T Dequeue()
    {
        T item = _array[_head];
        _array[_head] = default;
        _head = (_head + 1) % _array.Length;
        _size--;
        return item;
    }
}
```

### Stack<T> Internals

- Simple single-direction array growth; `Push` writes at `_size`, `Pop` reads at `_size - 1`.

### LinkedList<T> Internals

```csharp
internal sealed class LinkedListNode<T>
{
    internal T _value;
    internal LinkedListNode<T> _next;  // Forward
    internal LinkedListNode<T> _prev;  // Backward
}

internal sealed class LinkedList<T>
{
    internal LinkedListNode<T> _head;   // Head node (circular: head.prev == last, last.next == head)
    internal int _count;
}
```

- Circular doubly linked list: `_head._prev` points to the last node.
- O(1) insert/remove at known nodes, O(n) lookup.

## 4. Production Code Examples

```csharp
// Thread-safe caching with ConcurrentDictionary
public class ProductCache
{
    private readonly ConcurrentDictionary<int, Product> _cache = new();
    private readonly TimeSpan _ttl = TimeSpan.FromMinutes(5);
    private readonly ConcurrentDictionary<int, DateTime> _timestamps = new();

    public async Task<Product> GetOrFetchAsync(int productId, Func<int, Task<Product>> factory)
    {
        if (_cache.TryGetValue(productId, out var cached) &&
            _timestamps.TryGetValue(productId, out var ts) &&
            DateTime.UtcNow - ts < _ttl)
        {
            return cached;
        }

        Product product = await factory(productId);
        _cache[productId] = product;
        _timestamps[productId] = DateTime.UtcNow;
        return product;
    }
}
```

```csharp
// High-performance list pooling to reduce GC pressure
public class ListPool<T>
{
    private readonly ConcurrentBag<List<T>> _pool = new();

    public List<T> Rent(int minCapacity = 0)
    {
        if (_pool.TryTake(out var list))
        {
            if (list.Capacity < minCapacity)
                list.Capacity = minCapacity;
            return list;
        }
        return new List<T>(minCapacity);
    }

    public void Return(List<T> list)
    {
        list.Clear();
        if (list.Capacity <= 1024)
            _pool.Add(list);
    }
}
```

```csharp
// PriorityQueue for job scheduling
public class JobScheduler
{
    private readonly PriorityQueue<Job, int> _queue = new();

    public void Enqueue(Job job, int priority) =>
        _queue.Enqueue(job, priority);

    public Job DequeueHighestPriority() =>
        _queue.Dequeue();  // Lowest int = highest priority

    public bool TryDequeue(out Job job, out int priority) =>
        _queue.TryDequeue(out job, out priority);
}
```

```csharp
// Using SortedSet for order maintenance (e.g., leaderboard)
public class Leaderboard
{
    private readonly SortedSet<PlayerScore> _scores = new(PlayerScoreComparer.Instance);

    public void AddOrUpdate(PlayerScore ps)
    {
        _scores.Remove(ps);
        _scores.Add(ps);
    }

    public IEnumerable<PlayerScore> GetTop(int n) =>
        _scores.Take(n);
}
```

## 5. Real-World Scenarios

**Scenario 1: Order Book for Trading System**
- Use `SortedDictionary<decimal, OrderBookLevel>` for price-level lookups.
- Use `ConcurrentDictionary<long, Order>` for order-by-ID lookups.
- Use `ConcurrentQueue<Order>` for FIFO order matching.

**Scenario 2: Web Request Rate Limiter**
- Use `ConcurrentDictionary<string, FixedSizeQueue<DateTime>>` per API key.
- `Queue<T>` for sliding window timestamps.

**Scenario 3: Event Sourcing Aggregate**
- Use `LinkedList<Event>` for append-heavy event streams with occasional replay.

**Scenario 4: Object Pool for Database Connections**
- Use `ConcurrentBag<DbConnection>` since order doesn't matter and each thread works on its own.

## 6. Performance

| Collection           | Access        | Search        | Insert/Add    | Delete        | Memory         |
|----------------------|---------------|---------------|---------------|---------------|----------------|
| T[]                  | O(1)          | O(n)          | O(n) (resize) | O(n)          | Lowest         |
| List<T>              | O(1)          | O(n)          | O(1)*         | O(n)          | Low + slack    |
| Dictionary<K,V>      | O(1)*         | O(1)*         | O(1)*         | O(1)*         | High (bucket)  |
| HashSet<T>           | -             | O(1)*         | O(1)*         | O(1)*         | High (bucket)  |
| SortedDictionary<K,V>| O(log n)      | O(log n)      | O(log n)      | O(log n)      | Medium (tree)  |
| SortedSet<T>         | -             | O(log n)      | O(log n)      | O(log n)      | Medium (tree)  |
| Stack<T>             | O(1) peak     | O(n)          | O(1)*         | O(1)*         | Low            |
| Queue<T>             | O(1) peek     | O(n)          | O(1)*         | O(1)*         | Low            |
| LinkedList<T>        | O(n)          | O(n)          | O(1)**        | O(1)**        | High (node)    |
| PriorityQueue<T,P>   | O(1) peek     | O(n)          | O(log n)      | O(log n)      | Medium         |

*Amortized; **At known node; ~Hash collisions degrade to O(n)

### Memory Layout Best Practices

```csharp
// Pre-allocate to avoid resizing
List<int> nums = new(capacity: 100_000);

// Use ArrayPool for large temporary buffers
byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
try { /* use buffer */ }
finally { ArrayPool<byte>.Shared.Return(buffer); }

// For struct enumerators, avoid boxing (List<T> already returns struct)
// But ArrayList (non-generic) boxes every element.
```

## 7. Security

```csharp
// Avoid exposing internal collections as mutable references
public class InsecureRepository
{
    public List<User> Users { get; } = new();  // Callers can add/remove!
}

public class SecureRepository
{
    private readonly List<User> _users = new();
    public IReadOnlyList<User> Users => _users.AsReadOnly();  // Defensive copy or wrapper
}

// Guard against hash-collision DoS attacks
var dict = new Dictionary<string, string>(
    StringComparer.Ordinal);  // Use ordinal, not invariant culture

// Always validate keys/indices
if (!_dict.TryGetValue(userInputKey, out var value))
    throw new KeyNotFoundException("Invalid key");
```

## 8. Common Mistakes

```csharp
// MISTAKE 1: Modifying collection during enumeration
foreach (var item in list)
{
    if (ShouldRemove(item))
        list.Remove(item);  // InvalidOperationException!
}
// FIX: Iterate backwards or use RemoveAll
list.RemoveAll(ShouldRemove);

// MISTAKE 2: Ignoring default comparer for custom types
class Person { public string Name; }
var set = new HashSet<Person>();  // Uses reference equality!
// FIX: Override Equals/GetHashCode or pass IEqualityComparer<Person>

// MISTAKE 3: Capacity fragmentation with List<T>
var list = new List<int>();
for (int i = 0; i < 100_000; i++) list.Add(i);  // ~17 resizes!
// FIX: new List<int>(capacity: 100_000);

// MISTAKE 4: Returning internal array reference
public int[] GetItems() => _items;  // Caller can mutate!
// FIX: return _items.ToArray();  // defensive copy

// MISTAKE 5: LINQ over LinkedList<T>
var middle = linkedList.Skip(count / 2).First();  // O(n) traversal
// FIX: Use Find(node) if you have a reference.

// MISTAKE 6: Concurrent write without synchronization
if (!_dict.ContainsKey(key)) _dict.Add(key, value);  // Race condition!
// FIX: _dict.GetOrAdd(key, _ => value);
```

## 9. Senior Engineer Perspective

**Design Principles for Collection Choice:**

1. **Measure, don't guess.** The "theoretically better" collection may be slower due to memory locality or GC pressure. Profile with BenchmarkDotNet.

2. **Immutable collections for safety in concurrent read scenarios.** `ImmutableArray<T>`, `ImmutableDictionary<K,V>` from `System.Collections.Immutable` use persistent data structures (trees with sharing) for O(1) snapshot.

3. **Memory pooling reduces GC stalls.** ArrayPool<T>, ListPool<T> for hot paths in servers.

4. **Consider `Span<T>` and `Memory<T>` for slice operations** without allocation. These are not collections but views over contiguous memory, usable with `CollectionsMarshal` for direct access.

```csharp
// Low-allocation iteration using CollectionsMarshal
public static Span<T> AsSpan<T>(this List<T> list) =>
    CollectionsMarshal.AsSpan(list);

// Direct mutation without bounds checks
var span = CollectionsMarshal.AsSpan(myList);
span[0] = default;  // Mutates internal array, avoid resizing after
```

5. **For high-throughput scenarios, prefer struct-based collections.** `List<MyStruct>` stores structs inline in the array (dense memory, no GC references).

6. **Frozen collections** (.NET 8+): `FrozenSet<T>`, `FrozenDictionary<K,V>` for read-only lookups after construction — optimized down to minimal perfect hashing.

## 10. Interview Questions (Easy)

1. What is the difference between `List<T>` and `T[]`?
2. How does `Dictionary<TKey,TValue>` handle hash collisions?
3. What is the default capacity of `List<T>`?
4. What is the difference between `ICollection<T>` and `IEnumerable<T>`?
5. How do you remove elements from a collection during iteration?
6. What is `IReadOnlyList<T>` and when would you use it?
7. Explain the difference between `Stack<T>` and `Queue<T>`.
8. What is `HashSet<T>` used for?
9. How does `LinkedList<T>` differ from `List<T>` in lookup performance?
10. What does `TrimExcess()` do on `List<T>`?

## 11. Interview Questions (Medium)

1. Explain the amortized O(1) cost of `List<T>.Add()`. When does it degrade?
2. How does `Dictionary<TKey,TValue>` use `GetHashCode` and `Equals` internally?
3. Compare `SortedDictionary<TKey,TValue>` and `SortedList<TKey,TValue>` in terms of memory and performance.
4. When would you choose `ConcurrentDictionary<TKey,TValue>` over a regular `Dictionary<TKey,TValue>` with locking?
5. Explain how `PriorityQueue<TElement, TPriority>` works internally (binary heap).
6. What is the `_version` field in `List<T>` used for?
7. How does `Enumerable.Cast<T>` and `Enumerable.OfType<T>` differ in behavior with non-generic collections?
8. Explain the circular buffer pattern used in `Queue<T>`.
9. What is the difference between `ICollection<T>.IsReadOnly` and `IReadOnlyCollection<T>`?
10. How do you implement `IEquatable<T>` for optimal dictionary performance with value types?

## 12. Advanced Interview Questions (Hard)

1. Describe the internal resize strategy of `Dictionary<TKey,TValue>`. How does it rehash entries when growing?
2. Implement a thread-safe enumerator for a lock-free collection. What guarantees can you provide?
3. Design a memory-efficient `HashSet<T>` for 10 million struct keys.
4. Explain how `CollectionsMarshal.AsSpan()` works and why it's dangerous.
5. Implement an LRU cache using `LinkedList<T>` and `Dictionary<TKey, LinkedListNode<T>>`.
6. How would you design a concurrent `ObservableCollection<T>` that batches change notifications?
7. Explain `ImmutableArray<T>`'s internal structure and compare its performance to `List<T>`.
8. Design a collection that stores elements in sorted order with O(1) contains check.
9. How does .NET runtime specialize `EqualityComparer<T>.Default` for different types?
10. Write a lock-free `ConcurrentQueue<T>`-like structure from scratch.

## 13. Interview Questions (System Design)

1. Design a multi-tenant cache with per-tenant eviction policies using concurrent collections.
2. Design a distributed rate limiter with consistent hashing and rolling window counters.
3. Design a real-time leaderboard system with 10M users using sorted sets.
4. Design an event sourcing store using append-only linked lists and snapshots.
5. Design an in-memory full-text search index using inverted indexes with `Dictionary<string, HashSet<int>>`.
6. Design a task scheduler with dependency graph using `ConcurrentDictionary` and `ConcurrentQueue`.
7. Design an object pool for a high-throughput game server.
8. Design a batch message processor that groups messages by key using `Channel<T>`.
9. Design a streaming windowed aggregation engine (sliding window of 60s).
10. Design a service mesh sidecar connection pool that reuses gRPC channels.

## 14. Expert-Level Interview Questions (Architect)

1. Design a distributed, consistent, in-memory data grid with partitioning and replication, using immutable collections for snapshot isolation.
2. Implement a garbage-collection-friendly ring buffer for a financial exchange with zero allocations on the hot path.
3. Design a collection library for real-time systems (no heap allocations after initialization) using `Span<T>` and stack-only structs.
4. Architect a versioned collection store that supports time-travel queries (efficient rollback/rollforward) using persistent data structures.
5. Design a hybrid dictionary that switches from hash-based to tree-based storage when hash collisions become pathological (like Java's HashMap).
6. Architect a concurrent b-tree for a database storage engine, with lock-free reads and fine-grained locking on writes.
7. Design a high-performance serialization framework that bypasses allocation by writing directly into a collection's internal buffer.
8. Design a collection-based event store that can replay 1M events/second with snapshotting and indexing.
9. Architect a zero-allocation log-structured merge-tree (LSM-tree) using sorted collections and tiered compaction.
10. Design an actor framework mailbox that uses a multi-producer, single-consumer concurrent queue with work stealing.

## 15. Debugging & Troubleshooting

```csharp
// Use DebuggerDisplay for collection inspection
[DebuggerDisplay("Count = {Count}")]
[DebuggerTypeProxy(typeof(ListDebugView<>))]
public class MyCollection<T> { /* ... */ }

// Watch for:
// - ConcurrentModificationException: use foreach on concurrent collection or capture snapshot
// - High GC allocations: profile List<T> resizes, use capacity hints
// - Dictionary key not found: always use TryGetValue

// SOS/WinDbg commands for dump analysis
// !dumpheap -type System.Collections.Generic.List`1
// !do <address>
// !dso
```

## 16. Comparison Section

```
+--------------------+------------+------------+-------------+-------------+
| Feature            |  List<T>   |  T[]       | LinkedList  | HashSet<T>  |
+--------------------+------------+------------+-------------+-------------+
| Indexed access     |  O(1)      |  O(1)      |  O(n)       |  N/A        |
| Add to end         |  O(1)*     |  N/A       |  O(1)       |  O(1)*      |
| Insert at start    |  O(n)      |  N/A       |  O(1)       |  N/A        |
| Remove by value    |  O(n)      |  O(n)      |  O(n)       |  O(1)*      |
| Contains           |  O(n)      |  O(n)      |  O(n)       |  O(1)*      |
| Memory overhead    |  Low+slack |  Minimal   |  High(3ptr) |  High(buck)  |
| Cache locality     |  Excellent |  Excellent |  Poor       |  Good        |
+--------------------+------------+------------+-------------+-------------+
*Amortized / average case

+---------------------------+---------------------+---------------------+
| Feature                   | Dictionary<K,V>     | SortedDict<K,V>     |
+---------------------------+---------------------+---------------------+
| Underlying structure      | Hash table (bucket) | Red-black tree      |
| Lookup                    | O(1)*               | O(log n)            |
| Ordered iteration         | No (insertion-ish)  | Yes (sorted by key) |
| Memory per entry          | ~12 bytes overhead  | ~40 bytes (node)    |
| Best for                  | Fast key lookup     | Range queries       |
+---------------------------+---------------------+---------------------+
```

## 17. Revision Notes

- List<T> uses an internal array that doubles; amortized O(1) Add.
- Dictionary uses separate chaining with a single Entry[] array.
- Queue is a circular buffer.
- LinkedList is circular doubly linked.
- ConcurrentDictionary uses striped locking (per-bucket).
- Immutable collections use tree-based persistent structures.
- FrozenDictionary (NET 8+) uses perfect hashing for O(1) read-only.
- Always pre-allocate capacity when size is known.
- Use `AsReadOnly()`, `ToArray()`, or `ImmutableArray<T>` to prevent mutation leaks.
- `PriorityQueue` uses a binary heap (min-heap by default, max via custom comparer).

## 18. Cheat Sheet

```
+------------------------------------------------------------------+
|                     C# COLLECTIONS CHEAT SHEET                    |
+------------------------------------------------------------------+
| GENERAL PURPOSE                                                    |
|  List<T>       = dynamic array, O(1) index, O(1)* add             |
|  Dictionary<K,V> = hash table, O(1)* key lookup                   |
|  HashSet<T>    = unique elements, O(1)* add/remove/contains       |
|  Queue<T>      = FIFO, circular buffer                            |
|  Stack<T>      = LIFO, single-direction array                     |
|  LinkedList<T> = doubly linked, O(1) insert at node, O(n) lookup  |
|  PriorityQueue<T,P> = binary heap, O(log n) enqueue/dequeue       |
+------------------------------------------------------------------+
| ORDERED                                                           |
|  SortedDictionary<K,V> = red-black tree, O(log n)                 |
|  SortedList<K,V>       = sorted array, O(log n) search, O(n) ins  |
|  SortedSet<T>         = red-black tree, O(log n)                  |
+------------------------------------------------------------------+
| CONCURRENT                                                        |
|  ConcurrentDictionary<K,V> = striped locking, fine-grained         |
|  ConcurrentQueue<T>       = lock-free (CAS), multi-producer       |
|  ConcurrentStack<T>       = lock-free (CAS), multi-producer       |
|  ConcurrentBag<T>         = thread-local storage, no ordering     |
|  BlockingCollection<T>    = bounded producer-consumer             |
|  Channel<T>               = async-first producer-consumer         |
+------------------------------------------------------------------+
| IMMUTABLE                                                        |
|  ImmutableArray<T>      = struct, O(1) snapshot                  |
|  ImmutableDictionary<K,V> = hash-based AVL tree                   |
|  ImmutableHashSet<T>    = hash-based AVL tree                    |
|  ImmutableList<T>       = AVL tree                               |
|  FrozenDictionary<K,V>  = perfect hash, O(1) read-only           |
+------------------------------------------------------------------+
| MEMORY TIPS                                                       |
|  Pre-allocate: new List<T>(n) avoids resizes                      |
|  ArrayPool<T>.Shared.Rent(n) for temp buffers                     |
|  Use structs in List<T> for inline storage                        |
|  TrimExcess() after bulk insert if reads >> writes                |
|  IReadOnlyList<T> > List<T> for public API surfaces               |
+------------------------------------------------------------------+
| COMMON PATTERNS                                                   |
|  RemoveAll(pred)  -> in-place removal                             |
|  ForEach(Action)  -> side-effect iteration (avoid in LINQ)        |
|  GetOrAdd(key, fn) -> concurrent dict lazy init                    |
|  AddOrUpdate(key, add, upd) -> upsert                             |
|  TryGetValue(key, out val) -> safe dict lookup                    |
+------------------------------------------------------------------+
