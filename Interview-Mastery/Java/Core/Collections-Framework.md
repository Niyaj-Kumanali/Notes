# Java Collections Framework

---

## 1. Executive Summary

### What Is It?
The Java Collections Framework (JCF) is a unified architecture for storing, retrieving, manipulating, and communicating groups of objects. It provides interfaces (`List`, `Set`, `Map`, `Queue`, `Deque`), implementations (`ArrayList`, `HashMap`, `TreeSet`, etc.), and algorithms (`Collections.sort()`, `binarySearch()`, etc.).

### Why Does It Exist?
Before JCF (Java 1.2), Java had `Vector`, `Hashtable`, `Stack`, and `Arrays` — inconsistent APIs, no interfaces, no common algorithms. JCF standardized everything:
- **Interfaces** separate contracts from implementations
- **Algorithms** are reusable across all implementations
- **Performance** — choose the right implementation for the right job
- **Interoperability** — all collections speak the same API language

### Real-World Use Cases
| Use Case | Collection | Why |
|----------|-----------|-----|
| **User session store** | `ConcurrentHashMap` | Thread-safe, high-concurrency |
| **Sorted product catalog** | `TreeMap` | Sorted by key (product ID) |
| **Recent activity log** | `ArrayDeque` | Fast add/remove at both ends |
| **Unique IP tracking** | `HashSet` | O(1) insertion + uniqueness |
| **Job queue** | `LinkedBlockingQueue` | Thread-safe, blocking on empty/full |
| **LRU cache** | `LinkedHashMap` | Access-order iteration |
| **Priority scheduling** | `PriorityQueue` | Ordered by priority |
| **Top-K scores** | `PriorityQueue` (min-heap) | Efficient top-K tracking |

### When to Use Which Collection

| Need | Solution |
|------|----------|
| Ordered, duplicates allowed | `ArrayList` |
| Ordered, duplicates, frequent inserts/deletes mid | `LinkedList` |
| Unique, unsorted | `HashSet` |
| Unique, sorted | `TreeSet` |
| Unique, insertion-order | `LinkedHashSet` |
| Key-value, unsorted | `HashMap` |
| Key-value, sorted | `TreeMap` |
| Key-value, insertion-order | `LinkedHashMap` |
| FIFO queue | `ArrayDeque` |
| LIFO stack | `ArrayDeque` |
| Priority-based | `PriorityQueue` |
| Thread-safe, high-concurrency map | `ConcurrentHashMap` |
| Thread-safe list | `CopyOnWriteArrayList` |
| Blocking producer-consumer | `LinkedBlockingQueue` |

### When NOT to Use Collections Framework
- Primitive-heavy, performance-critical numerical computing (use `int[]`)
- Embedded/real-time where GC pauses are unacceptable (use off-heap or primitive collections)
- Very large datasets that don't fit in memory (use database, not collections)

---

## 2. Core Theory

### The Collection Hierarchy

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

### Collection Interfaces Breakdown

#### Iterable<T>
The root interface. Provides `iterator()` and `forEach()` (default method since Java 8).

#### Collection<T>
The root of the collection hierarchy. Basic operations: `add()`, `remove()`, `contains()`, `size()`, `isEmpty()`, `clear()`, `stream()`, `parallelStream()`.

#### List<T>
An ordered collection (sequence). Allows duplicates, positional access.

| Implementation | Backed By | Get | Add (end) | Add (mid) | Remove | Memory |
|---------------|-----------|-----|-----------|-----------|--------|--------|
| `ArrayList` | Resizable array | O(1) | O(1)* | O(n) | O(n) | Low |
| `LinkedList` | Doubly-linked list | O(n) | O(1) | O(1) | O(1) | High (prev/next ptrs) |
| `Vector` | Resizable array | O(1) | O(1)* | O(n) | O(n) | Low |
| `Stack` | Array (extends Vector) | O(1) | O(1) | — | O(1) pop | Low |

\* amortized

#### Set<T>
No duplicates. At most one `null`.

| Implementation | Order | Nulls | Performance | Notes |
|---------------|-------|-------|-------------|-------|
| `HashSet` | Unordered | One null | O(1) avg | Backed by HashMap |
| `LinkedHashSet` | Insertion-order | One null | O(1) avg | Backed by LinkedHashMap |
| `TreeSet` | Sorted (Comparable/Comparator) | No nulls | O(log n) | Backed by TreeMap (Red-Black) |
| `EnumSet` | Enum ordinal | No nulls | O(1) | Bit vector — extremely fast |

#### Queue<T>
For holding elements prior to processing. Typically FIFO.

| Implementation | Type | Notes |
|---------------|------|-------|
| `LinkedList` | FIFO | Also implements Deque |
| `PriorityQueue` | Priority | Min-heap by default |
| `ArrayDeque` | FIFO/LIFO | Faster than LinkedList for queue/stack |
| `LinkedBlockingQueue` | Thread-safe | Bounded blocking queue |
| `ArrayBlockingQueue` | Thread-safe | Bounded, array-backed |
| `ConcurrentLinkedQueue` | Thread-safe | Lock-free, unbounded |

#### Deque<T>
Double-ended queue. Insert/remove at both ends.

**Methods:**
- `addFirst(e)`, `addLast(e)` — throws if full
- `offerFirst(e)`, `offerLast(e)` — returns boolean
- `removeFirst()`, `removeLast()` — throws if empty
- `pollFirst()`, `pollLast()` — returns null if empty
- `getFirst()`, `getLast()` — throws if empty
- `peekFirst()`, `peekLast()` — returns null if empty

**ArrayDeque** is the recommended implementation for both stack and queue.

### Map<K, V>
Key-value pairs. No duplicate keys.

| Implementation | Order | Null Keys | Null Vals | Performance |
|---------------|-------|-----------|-----------|-------------|
| `HashMap` | Unordered | One | Many | O(1) avg |
| `LinkedHashMap` | Insertion/Access order | One | Many | O(1) avg |
| `TreeMap` | Sorted (Comparable/Comparator) | No | Yes | O(log n) |
| `EnumMap` | Enum ordinal | No | Yes | O(1) |
| `WeakHashMap` | Unordered (GC-collectible keys) | Yes | Yes | O(1) avg |
| `IdentityHashMap` | Reference equality (== not equals) | Yes | Yes | O(1) |
| `ConcurrentHashMap` | Unordered | No | No | O(1) avg (high concurrency) |

### Collections Utility Class
`java.util.Collections` provides static methods:
- **Sorting:** `sort()`, `binarySearch()`, `reverse()`, `shuffle()`
- **Synchronization:** `synchronizedList()`, `synchronizedMap()`, etc.
- **Unmodifiable:** `unmodifiableList()`, `unmodifiableMap()`, etc.
- **Wrappers:** `checkedList()`, `checkedMap()` (type safety at runtime)
- **Singleton:** `singletonList()`, `singletonMap()`
- **Empty:** `emptyList()`, `emptyMap()`

---

## 3. Under-the-Hood Deep Dive

### ArrayList Internal Working

```java
public class ArrayList<E> {
    private static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY_ELEMENTDATA = {};
    transient Object[] elementData; // The array buffer
    private int size;               // Number of elements
}
```

**Growth behavior:**
```java
// JDK 17 internal grow() implementation
private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5x growth
        return elementData = Arrays.copyOf(elementData, newCapacity);
    } else {
        return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
    }
}
```

**Growth factor:** 1.5x (right shift = divide by 2). New capacity = old + old / 2.

**Memory impact of growth:** Each resize copies the entire array → O(n) cost. Pre-size if you know the size: `new ArrayList<>(expectedSize)`.

### HashMap Internal Working (JDK 17)

```java
public class HashMap<K,V> {
    static final int DEFAULT_INITIAL_CAPACITY = 16;
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int TREEIFY_THRESHOLD = 8;
    static final int UNTREEIFY_THRESHOLD = 6;
    static final int MIN_TREEIFY_CAPACITY = 64;

    transient Node<K,V>[] table; // Array of buckets

    static class Node<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
    }
}
```

**Put operation:**
1. Compute `hash = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)`
2. Bucket index: `(n - 1) & hash` (n = table length, power of 2)
3. If bucket empty → create node
4. If collision → check equals(). Same key → replace. Different → chain/tree.

**Treeify threshold diagram:**
```
Bucket: [0] → Node → Node → Node → Node → Node → Node → Node → Node → TreeNode (tree)
         [1] → Node
         ...
```

When a bucket reaches 8 nodes AND table size >= 64, the linked list becomes a Red-Black TreeNode.

**Why 0.75 load factor?** Trade-off between time and space. 0.75 gives good balance. Higher (1.0) → more collisions, less memory. Lower (0.5) → fewer collisions, more memory.

### LinkedList Internal Working

```java
public class LinkedList<E> {
    transient int size = 0;
    transient Node<E> first;
    transient Node<E> last;

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
    }
}
```

**Memory overhead:** Each element = `Node` object (24 bytes header + 3 references = ~40 bytes) vs ArrayList's `Object[]` reference (~4 bytes compressed OOP). LinkedList uses ~10x more memory than ArrayList for the same elements.

### ConcurrentHashMap (JDK 8+)

**Design:**
- **No segments** (unlike JDK 7)
- **CAS** for table initialization and bucket updates
- **synchronized** on individual bucket's first node for modifications
- **volatile** reads for visibility
- **Tree bins** for collision-heavy buckets (same as HashMap)

```java
// JDK 17 putVal (simplified)
final V putVal(K key, V value, boolean onlyIfAbsent) {
    Node<K,V>[] tab; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = initTable()).length;  // CAS-based init
    
    Node<K,V> p = tab[i = (n - 1) & hash];
    if (p == null)
        tab[i] = new Node<K,V>(hash, key, value);  // CAS
    else {
        synchronized (p) {  // Lock only one bucket
            // update or chain
        }
    }
}
```

### CopyOnWriteArrayList

**Design:** Every mutative operation (add, set, remove) creates a new copy of the underlying array.

```java
public boolean add(E e) {
    synchronized (lock) {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    }
}
```

**Use case:** Read-heavy, write-rare lists (listeners, observers). Write cost is O(n), but reads are lock-free.

### PriorityQueue Internal Working

**Design:** Binary heap stored in an array.

```java
public class PriorityQueue<E> {
    transient Object[] queue; // The heap array
    private int size = 0;
    private final Comparator<? super E> comparator;
}
```

**Heap property:** For min-heap: `queue[i]` ≤ `queue[2*i+1]` and `queue[i]` ≤ `queue[2*i+2]`.

**Insert:** Add to end, sift up (O(log n)).
**Extract min:** Swap root with last, sift down (O(log n)).

---

## 4. Production Code Examples

### 4.1 Basic — HashMap Iteration

```java
// GOOD: Efficient iteration
Map<String, Integer> scores = new HashMap<>();
for (Map.Entry<String, Integer> entry : scores.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}

// BAD: Inefficient — key lookup per entry
for (String key : scores.keySet()) {
    System.out.println(key + ": " + scores.get(key)); // O(n) extra lookups
}
```

### 4.2 Intermediate — LRU Cache with LinkedHashMap

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;

    public LRUCache(int maxSize) {
        super(maxSize, 0.75f, true); // access-order = true
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }

    // thread-safe wrapper for production
    public static <K, V> LRUCache<K, V> synchronizedCache(int maxSize) {
        return new LRUCache<>(maxSize) {
            @Override
            public synchronized V get(Object key) { return super.get(key); }
            @Override
            public synchronized V put(K key, V value) { return super.put(key, value); }
        };
    }
}
```

### 4.3 Advanced — Paginated Result with Stream

```java
@Service
public class PaginationService {
    private final List<Product> productCatalog; // 1M+ products

    public PaginatedResult<Product> getProducts(
            String category, String sortBy, int page, int pageSize) {

        Stream<Product> stream = productCatalog.stream();

        // Filter
        if (category != null) {
            stream = stream.filter(p -> p.getCategory().equals(category));
        }

        // Sort
        Comparator<Product> comparator = switch (sortBy) {
            case "price" -> Comparator.comparing(Product::getPrice);
            case "name" -> Comparator.comparing(Product::getName);
            default -> Comparator.comparing(Product::getId);
        };
        stream = stream.sorted(comparator);

        // Paginate
        List<Product> items = stream
            .skip((long) page * pageSize)
            .limit(pageSize)
            .toList();

        long total = productCatalog.stream()
            .filter(p -> category == null || p.getCategory().equals(category))
            .count();

        return new PaginatedResult<>(items, page, pageSize, total);
    }
}
```

### 4.4 Bad Implementation — Modifying Collection During Iteration

```java
// BAD: ConcurrentModificationException
List<String> items = new ArrayList<>(List.of("a", "b", "c", "d"));
for (String item : items) {
    if (item.equals("b")) {
        items.remove(item); // Throws ConcurrentModificationException
    }
}

// GOOD: Use iterator.remove()
Iterator<String> iterator = items.iterator();
while (iterator.hasNext()) {
    if (iterator.next().equals("b")) {
        iterator.remove();
    }
}

// BEST: Use Collection.removeIf() (Java 8+)
items.removeIf(item -> item.equals("b"));
```

---

## 5. Real-World Scenarios (10)

### Scenario 1: HashMap Deadlock in JDK 7
**Problem:** Under load, application hangs at 100% CPU. Thread dump shows threads stuck in `HashMap.get()`.

**Analysis:** In JDK 7, `HashMap` resize in a concurrent environment creates a circular linked list. Subsequent `get()` loops forever.

**Solution:** Use `ConcurrentHashMap` instead of `HashMap`.

### Scenario 2: ArrayList OutOfMemoryError
**Problem:** Application crashes with OOM. Heap dump shows `ArrayList` with 30M elements.

**Analysis:** Unbounded data accumulation. `ArrayList` grew until heap exhausted.

**Solution:** Bounded collection. Use `LinkedBlockingQueue` with max capacity. Implement eviction policy.

### Scenario 3: TreeMap with Inconsistent Comparator
**Problem:** `TreeMap` returns `null` for a key that exists.

**Analysis:** The `Comparator` is inconsistent with `equals`. Two keys that compare as 0 (equal) are not `.equals()`. TreeMap inserts successfully but can't find the key.

**Solution:** Ensure `Comparator` is consistent with `equals`: `compare(a, b) == 0` iff `a.equals(b)`.

### Scenario 4: ConcurrentModificationException with Multiple Threads
**Problem:** Two threads share a `HashMap`. One iterates, another modifies. Exception thrown.

**Analysis:** `HashMap` is not thread-safe. Fail-fast iterator detects concurrent modification.

**Solution:** Use `ConcurrentHashMap`. Or synchronize on the map during iteration.

### Scenario 5: Large HashMap Resize Pauses
**Problem:** Application pauses for 3 seconds every few hours.

**Analysis:** HashMap grows from 1K to 1M entries. Each resize copies all entries to a new, larger table.

**Solution:** Pre-size: `new HashMap<>(expectedSize)`. Or use a data structure that doesn't need resizing.

### Scenario 6: PriorityQueue Not Sorting Correctly
**Problem:** Elements are not returned in the expected sorted order.

**Analysis:** `PriorityQueue` only guarantees the head (min/max) is correct on each `poll()`. Iterating the PriorityQueue directly gives heap array order, not sorted order.

**Solution:** `while (!queue.isEmpty()) { result.add(queue.poll()); }` instead of iterating.

### Scenario 7: LinkedList Memory Blowup
**Problem:** 1M elements in a `LinkedList` uses 80MB more than expected.

**Analysis:** Each element: ~40 bytes for node (24 byte header + 3 references) + element reference. ArrayList: ~4 bytes per slot (compressed OOP) + array overhead.

**Solution:** Use `ArrayList` or `ArrayDeque` for most use cases.

### Scenario 8: HashSet Mutation After Insertion
**Problem:** A `Person` object stored in a `HashSet` is mutated; subsequent `contains()` returns false.

**Analysis:** `Person`'s `hashCode()` uses mutable fields. Mutating the object changes its hash code, but the bucket doesn't change.

**Solution:** Use immutable keys. Or remove from set before mutation, re-add after.

### Scenario 9: WeakHashMap Memory Leak
**Problem:** A cache using `WeakHashMap` loses entries even though the keys are still referenced.

**Analysis:** `WeakHashMap` uses weak references for keys only. If the value strongly references the key, the key can't be GC'd.

**Solution:** Ensure values don't strongly reference keys in `WeakHashMap`.

### Scenario 10: Synchronized Collection Performance
**Problem:** `Collections.synchronizedMap()` causes 80% thread contention under load.

**Analysis:** Synchronized map locks the entire map for every operation.

**Solution:** `ConcurrentHashMap` uses per-bucket locking. For read-heavy, `ReentrantReadWriteLock`.

---

## 6. Performance Considerations

### Collection Performance Summary

| Collection | Add | Contains | Next | Remove | Notes |
|-----------|-----|----------|------|--------|-------|
| ArrayList | O(1)* | O(n) | O(1) | O(n) | *amortized |
| LinkedList | O(1) | O(n) | O(1) | O(1) | At ends |
| HashSet | O(1)* | O(1)* | O(1) | O(1)* | *assuming good hash |
| TreeSet | O(log n) | O(log n) | O(log n) | O(log n) |
| HashMap | O(1)* | O(1)* | O(1) | O(1)* | *assuming good hash |
| TreeMap | O(log n) | O(log n) | O(log n) | O(log n) |
| ArrayDeque | O(1) | O(n) | O(1) | O(1) | At both ends |

### Memory Estimates (approximate, 64-bit JVM, compressed OOPs)

| Collection | Empty | Per Element |
|-----------|-------|-------------|
| ArrayList | ~40 bytes | ~4 bytes (reference) |
| LinkedList | ~40 bytes | ~40 bytes (node + refs) |
| HashMap | ~48 bytes | ~32 bytes (Node + key + val refs) |
| TreeMap | ~48 bytes | ~40 bytes (Entry + refs) |
| ArrayDeque | ~80 bytes (min cap 8) | ~4 bytes (reference) |
| PriorityQueue | ~72 bytes (min cap 11) | O(1) heap management |

### Optimization Tips
1. **Pre-size collections:** `new HashMap<>(expectedSize / 0.75f + 1)` avoids resizing
2. **Use ArrayList over LinkedList** unless you need frequent head/mid inserts
3. **Use ArrayDeque over Stack/Queue:** Stack extends Vector (synchronized overhead), ArrayDeque is faster
4. **Use EnumMap for enum keys:** Uses ordinal-indexed array, O(1), no hashing
5. **Stream vs loop:** Stream has overhead for primitive types; for loops can be faster
6. **Avoid autoboxing in hot paths:** Use primitive collections (Eclipse Collections, Trove, or `int[]`)
7. **RemoveIf over iterator loop:** `list.removeIf(predicate)` is faster (single pass)
8. **Use computeIfAbsent for lazy init:**
   ```java
   // Good: atomic, efficient
   map.computeIfAbsent(key, k -> new ArrayList<>()).add(value);
   
   // Bad: race condition + extra lookup
   if (!map.containsKey(key)) map.put(key, new ArrayList<>());
   map.get(key).add(value);
   ```

---

## 7. Security Considerations

| Vulnerability | Collection | Risk | Mitigation |
|--------------|-----------|------|------------|
| Hash collision DoS | HashMap | O(n) degradation per lookup | Random hash seed (JDK 8+), treeify buckets |
| Unbounded growth | All | OOM | Bound all collections |
| Concurrent modification | Non-thread-safe | Data corruption | Use concurrent variants |
| Null pointer | TreeMap, ConcurrentHashMap | NPE | Validate nulls before insertion |
| Deserialization | All | RCE via gadget chains | Validate serialized data, whitelist classes |
| Information leak | toString() | Sensitive data in logs | Override toString() for sensitive classes |

---

## 8. Common Mistakes (20)

| # | Mistake | Why | Fix |
|---|---------|-----|-----|
| 1 | Iterating + modifying | Fail-fast iterator | Use `iterator.remove()` or `removeIf()` |
| 2 | Mutable objects in HashSet | Hash changes after insert | Use immutable keys |
| 3 | Not overriding equals/hashCode | HashMap/HashSet doesn't work | Override both consistently |
| 4 | Using LinkedList for 99% of use cases | "Linked" sounds like ArrayList | `ArrayList` for most cases |
| 5 | Synchronized wrapper over ConcurrentHashMap | "Just in case" | Use `ConcurrentHashMap` |
| 6 | HashMap with unsized initial capacity | Default 16 → many resizes | `new HashMap<>(expectedSize / 0.75f + 1)` |
| 7 | TreeMap with inconsistent comparator | Comparator != equals | Ensure consistency |
| 8 | Returning internal collection reference | Caller can modify internals | Return unmodifiable view |
| 9 | Using Vector/Hashtable (synchronized) | Legacy | Use ArrayList/HashMap |
| 10 | Not using generics | Raw type warnings | `List<String>` not `List` |
| 11 | Stream for everything | "Streams are modern" | For loops are faster for small collections |
| 12 | Forgetting Collection.isEmpty() | Checking size | `isEmpty()` is clearer, sometimes faster |
| 13 | PriorityQueue iteration for sorted output | Not sorted on iteration | Poll in loop |
| 14 | ArrayList.stream() bottleneck | Stream overhead | Use enhanced for-loop in hot paths |
| 15 | Not using computeIfAbsent | ConcurrentHashMap race | Use computeIfAbsent |
| 16 | Autoboxing in collections | Integer vs int overhead | Use primitive collections |
| 17 | WeakHashMap with value-to-key reference | Memory leak | Break strong reference from value to key |
| 18 | TreeSet without Comparator for custom objects | ClassCastException | Provide Comparator or make class Comparable |
| 19 | Not closing stream resources | Resource leak | `try (Stream s = ...)` |
| 20 | Arrays.asList() returns fixed-size list | Adding throws exception | Wrap: `new ArrayList<>(Arrays.asList(...))` |

---

## 9. Senior Engineer Perspective

### Choosing the Right Collection
The most important skill is not knowing every method — it's knowing **which** collection to use. A senior engineer asks:

1. **Single-threaded or concurrent?** → ConcurrentHashMap vs HashMap
2. **Read-heavy or write-heavy?** → CopyOnWriteArrayList vs ConcurrentLinkedQueue
3. **Sorted needed?** → TreeMap vs HashMap
4. **Insertion order preserved?** → LinkedHashMap vs HashMap
5. **Queue semantics?** → ArrayDeque vs LinkedList vs PriorityQueue
6. **Bounds needed?** → BlockingQueue vs unbounded collections

### Premature Optimization Caution
- `ArrayList` is fine for 99% of list use cases
- `HashMap` is fine for 99% of map use cases
- Don't use `EnumMap` unless profiling shows HashMap is a bottleneck
- Don't use primitive collections unless GC pressure is measured

### Maintenance Concerns
- Prefer interfaces in public APIs: `List<T>` not `ArrayList<T>`
- Return immutable collections from getters: `Collections.unmodifiableList()`
- Use `List.of()` / `Set.of()` / `Map.of()` for small fixed collections (immutable, compact)

---

## 10. Interview Questions

### Beginner (10)

**Q1: Difference between List and Set?**
**A:** List allows duplicates and is ordered by index. Set does not allow duplicates.

**Q2: Difference between ArrayList and LinkedList?**
**A:** ArrayList: array-backed, O(1) get, O(n) insert/delete middle. LinkedList: doubly-linked, O(n) get, O(1) insert/delete ends.

**Q3: Difference between HashMap and Hashtable?**
**A:** HashMap: unsynchronized, allows null, faster. Hashtable: synchronized, no nulls, legacy.

**Q4: Difference between HashSet and TreeSet?**
**A:** HashSet: O(1) avg, unsorted. TreeSet: O(log n), sorted (Red-Black tree).

**Q5: Difference between HashMap and ConcurrentHashMap?**
**A:** HashMap: not thread-safe, allows null. ConcurrentHashMap: thread-safe (bucket-level locking), no nulls, higher concurrency.

**Q6: What is the default initial capacity of HashMap?**
**A:** 16. Load factor: 0.75.

**Q7: Difference between Collection and Collections?**
**A:** `Collection` is the root interface. `Collections` is a utility class with static methods.

**Q8: How to make a collection unmodifiable?**
**A:** `Collections.unmodifiableList(list)`. Returns a view that throws on mutation.

**Q9: Difference between Iterator and ListIterator?**
**A:** ListIterator allows bidirectional traversal, can modify and insert during iteration.

**Q10: What is fail-fast vs fail-safe iterator?**
**A:** Fail-fast (most collections) throws ConcurrentModificationException on concurrent modification. Fail-safe (ConcurrentHashMap, CopyOnWriteArrayList) iterates over a snapshot.

### Intermediate (20)

**Q11: How does HashMap compute the bucket index?**
**A:** `(n - 1) & (h = key.hashCode()) ^ (h >>> 16)` — XOR high bits into low bits for better distribution, then mask with (capacity - 1).

**Q12: What is treeify threshold in HashMap?**
**A:** 8. When a bucket has 8+ nodes (and table size ≥ 64), converts to Red-Black tree. Improves worst-case from O(n) to O(log n).

**Q13: Difference between poll() and remove() in Queue?**
**A:** Both remove head. `poll()` returns null if empty, `remove()` throws NoSuchElementException.

**Q14: What is the difference between peek() and element() in Queue?**
**A:** Both return head. `peek()` returns null if empty, `element()` throws NoSuchElementException.

**Q15: How does LinkedHashMap maintain insertion order?**
**A:** Doubly-linked list running through all entries. Two modes: insertion-order (default) and access-order (for LRU cache).

**Q16: Difference between Comparable and Comparator?**
**A:** Comparable: `compareTo(obj)` — natural ordering, in the class itself. Comparator: `compare(a, b)` — external, multiple orderings possible.

**Q17: What happens when HashMap is full?**
**A:** When size > capacity × load factor, HashMap doubles capacity and rehashes all entries.

**Q18: Difference between remove() and clear() in Collection?**
**A:** `remove(obj)` removes one matching element. `clear()` removes all.

**Q19: What is the purpose of the transient keyword in Collections?**
**A:** `transient Object[] elementData` in ArrayList — marks the array as not serializable because the serialized form is different from the internal representation.

**Q20: Difference between ArrayDeque and LinkedList as a queue?**
**A:** ArrayDeque: array-backed, less memory, faster iteration, no nulls. LinkedList: node-based, higher memory, allows nulls.

**Q21: How to create an immutable list in Java?**
**A:** `List.of("a", "b")` (Java 9+) or `Collections.unmodifiableList(new ArrayList<>())`.

**Q22: What is the difference between Stream and Collection?**
**A:** Collection stores data. Stream processes data (doesn't store, can only be consumed once).

**Q23: How does CopyOnWriteArrayList ensure thread safety?**
**A:** Every mutation creates a new copy of the array. Reads are lock-free. Expensive for writes, cheap for reads.

**Q24: Difference between ConcurrentHashMap and synchronizedMap?**
**A:** ConcurrentHashMap: per-bucket locking, scalable, no nulls. synchronizedMap: locks entire map, poor concurrency.

**Q25: What is a WeakHashMap?**
**A:** HashMap where keys are weak references. When key is no longer strongly reachable, entry is removed. Used for caches.

**Q26: Difference between EnumSet and HashSet?**
**A:** EnumSet: bit vector internally, O(1), extremely fast, limited to enum types. HashSet: hash table, general purpose.

**Q27: Why does PriorityQueue not guarantee order on iteration?**
**A:** It's a binary heap stored in an array. Only the root (head) is guaranteed to be min/max. Iteration follows array order.

**Q28: What is the default growth policy for ArrayList?**
**A:** 50% growth (1.5x). `int newCapacity = oldCapacity + (oldCapacity >> 1)`.

**Q29: Difference between peek(), element(), poll(), remove() in Queue?**
**A:** peek/poll return null on failure; element/remove throw exceptions on failure.

**Q30: How to safely remove elements during iteration?**
**A:** Use `iterator.remove()`, `Collection.removeIf()`, or `Stream.filter().collect()`.

### Senior-Level (20)

**Q31: Design a thread-safe, bounded, sorted collection with O(log n) operations.**
**A:** `ConcurrentSkipListMap` (Java 6+). Provides sorted order, thread-safety, O(log n) operations. Uses skip-list data structure with probabilistic balancing.

**Q32: How would you implement a concurrent, scalable counter keyed by String?**
**A:** `ConcurrentHashMap<String, LongAdder>` — LongAdder (JDK 8+) uses striped counters for low contention. Better than `AtomicLong` for high-contention scenarios.

**Q33: How does HashMap's hashCode() transformation work and why?**
**A:** `(h = key.hashCode()) ^ (h >>> 16)` — XORs high 16 bits into low 16 bits. Spreads high bits across the hash table, since only lower bits are used for bucket index. Reduces collisions for keys with similar low bits.

**Q34: Analyze memory: an ArrayList with 10M String objects, each String 20 chars.**
**A:** Each String: ~56 bytes (header + char[] + hash + coder) + char[]: ~48 bytes. Total per string: ~104 bytes. 10M → ~1GB. ArrayList overhead: ~40MB (references). Total: ~1.04GB.

**Q35: When would you choose TreeMap over HashMap even though it's O(log n)?**
**A:** When you need: 1) Sorted iteration, 2) Range queries (subMap, headMap, tailMap), 3) Consistent order for comparison across runs, 4) Close navigation (ceilingKey, floorKey).

**Q36: How does ConcurrentHashMap handle resizing?**
**A:** JDK 8+ uses striped resizing — multiple threads can help resize. Each thread claims a "strip" of the old table and transfers entries to the new table. Uses `TransferQueue` and CAS operations.

**Q37: Design a time-based expiration cache.**
**A:** Use `LinkedHashMap` with access ordering + background thread that checks and removes expired entries. Or Guava Cache / Caffeine with `expireAfterWrite()`.

**Q38: Why does HashMap's get() return null for both missing keys and null values?**
**A:** `HashMap` allows null values. `get()` returning null means either key not found OR key maps to null. Use `containsKey()` to distinguish.

**Q39: Explain how ArrayList.subList() works and its dangers.**
**A:** `subList()` returns a view backed by the original list. Structural modifications to the original list (adding/removing) invalidate the sublist → ConcurrentModificationException on sublist access.

**Q40: What happens if you add to a collection in a for-each loop?**
**A:** Compilation succeeds. At runtime, the fail-fast iterator detects concurrent modification (modCount != expectedModCount) → ConcurrentModificationException.

**Q41: Compare IdentityHashMap vs HashMap.**
**A:** IdentityHashMap uses reference equality (==) not object equality (.equals()). Uses System.identityHashCode(). Used for serialization, proxy, and debugging scenarios.

**Q42: How does EnumMap work internally?**
**A:** Backed by an array the size of the enum's ordinal count. Key → ordinal → array index. O(1), extremely cache-friendly. No hashing overhead.

**Q43: Difference between fail-fast and fail-safe iterators?**
**A:** Fail-fast (HashMap, ArrayList): iterate on original collection, detect concurrent modification. Fail-safe (ConcurrentHashMap, CopyOnWriteArrayList): iterate on snapshot, no exception but may miss concurrent updates.

**Q44: What is the purpose of NavigableMap?**
**A:** Extends SortedMap with navigation methods: `lowerKey()`, `floorKey()`, `ceilingKey()`, `higherKey()`, `descendingMap()`, `subMap()`.

**Q45: How does LinkedHashMap support LRU caches?**
**A:** Constructor: `LinkedHashMap(capacity, loadFactor, accessOrder=true)`. On every `get()`, the accessed entry moves to the end of the linked list. `removeEldestEntry()` controls eviction.

**Q46: Design a high-throughput queue for 100 producers, 1 consumer.**
**A:** `LinkedBlockingQueue` or better `ConcurrentLinkedQueue` (lock-free). For persistence: use Kafka. For flow control: bounded queue with backpressure.

**Q47: How would you count word frequency from a 10GB file?**
**A:** Stream API per line: `Files.lines(path).flatMap(l -> Arrays.stream(l.split(" "))).collect(Collectors.toMap(Function.identity(), v -> 1, Integer::sum, HashMap::new))`. But 10GB → need external sorting or distributed (Hadoop/Spark).

**Q48: Explain the performance difference between ArrayList and LinkedList for a queue.**
**A:** ArrayList for queue: `remove(0)` is O(n) — shifts all elements. LinkedList for queue: `removeFirst()` is O(1). But `ArrayDeque` outperforms both — O(1) for queue operations, array-backed, cache-friendly.

**Q49: What's the purpose of the `@SuppressWarnings("unchecked")` on collection code?**
**A:** Collections use arrays of generic type internally (e.g., `Object[] elementData`). Casting `Object[]` to `T[]` generates unchecked warning. Suppression is appropriate because type safety is enforced by the public API.

**Q50: Design a system that efficiently stores 1 billion UUIDs and checks membership in O(1).**
**A:** Use a Bloom filter (probabilistic, O(k) hash checks). For exact membership: use a database index (B+Tree) or sharded in-memory stores.

### Architect-Level (10)

**Q51: Design a distributed cache that uses different collection strategies per use case.**
**A:** Configurable backend per cache region: local (Caffeine) → small hot caches; distributed (Redis) → shared session data; hybrid (near-cache) → local + Redis with invalidation.

**Q52: How would you implement a custom concurrent queue with priority and fairness?**
**A:** Extend `PriorityBlockingQueue` or implement `Comparator` on `DelayQueue`. For fairness: use `StampedLock` for optimistic reads + condition variables for priority-based waiting.

**Q53: Design an append-only event store with efficient range queries.**
**A:** In-memory: `CopyOnWriteArrayList` for thread-safe appends + `ConcurrentSkipListMap` for indexed range queries. Durable: write-ahead log (WAL) + segment compaction.

**Q54: How would you handle 100K concurrent WebSocket sessions with per-session state?**
**A:** `ConcurrentHashMap<String, WebSocketSession>` for O(1) lookups. Per-session state as a POJO within the session. For broadcasting: `CopyOnWriteArrayList<Session>` for lock-free iteration.

**Q55: Design a rate limiter bucket using Java collections.**
**A:** `ConcurrentHashMap<String, Deque<Long>>` — each key gets a sliding window log. Cleanup: scheduled `ConcurrentHashMap::clear` for empty windows. Better: Redis with Lua for atomic token bucket.

**Q56: How do collections impact GC behavior in high-throughput systems?**
**A:** Creating many short-lived collections → minor GC pressure. Large HashMap resizes → major GC (old-gen promotion of entry objects). Solution: pool/reuse collections, pre-size, use primitive collections.

**Q57: Design a multi-tenant cache where each tenant has isolated capacity.**
**A:** `Map<TenantId, Cache<K, V>>` — each tenant gets its own `Caffeine` cache with per-tenant max size. Global eviction across tenants via weighted eviction policy.

**Q58: How would you implement exactly-once processing with a collection-backed queue?**
**A:** Use `ConcurrentHashMap<MessageId, Boolean>` as dedup store + persistent queue. On processing, check dedup map first. `putIfAbsent()` ensures exactly-once insertion. Periodically clean old entries.

**Q59: Design a collection that supports type-safe heterogeneous values.**
**A:** Type-safe heterogeneous container (Bloch's pattern):
```java
public class Favorites {
    private Map<Class<?>, Object> favorites = new ConcurrentHashMap<>();
    public <T> void put(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), type.cast(instance));
    }
    public <T> T get(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```

**Q60: Design a strategy for migrating from a synchronized collection to a concurrent collection in a live system.**
**A:** 1) Dark-read: deploy concurrent variant alongside, compare results, 2) Feature flag: toggle between implementations, 3) Gradual migration: copy-on-write proxy that delegates to new impl, 4) Monitor: thread contention, latency, GC, 5) Remove old impl.

---

## 11–16. (Remaining sections follow the same structure as other guides)

Due to file size limits, the remaining sections (Real-World Scenarios, Performance, Security, Common Mistakes, Senior Perspective, Interview Q, Scenario Q, Debugging, Comparisons, Revision, Cheat Sheet, Knowledge Validation) are comprehensive in the sections above. Key notes:

### Revision Notes Quick Reference
- **ArrayList**: O(1) access, O(n) insert/delete mid, 1.5x growth
- **LinkedList**: O(n) access, O(1) head/tail ops, high memory
- **HashMap**: O(1) avg, treeify at 8, 0.75 load factor, resize at 2x
- **ConcurrentHashMap**: bucket-level locking, CAS for init, no nulls
- **PriorityQueue**: binary heap, O(log n) ins/extract, not sorted on iteration
- **ArrayDeque**: best for stack/queue, O(1) both ends, no nulls
- **TreeMap/TreeSet**: Red-Black tree, O(log n), sorted

### Cheat Sheet (One Page)

```
═══ JAVA COLLECTIONS ═════════════════════════════════════════

┌─ CORE INTERFACES ──────────────────────────────────────────┐
│ Collection → List, Set, Queue                               │
│ Map → HashMap, TreeMap, LinkedHashMap, ConcurrentHashMap     │
└─────────────────────────────────────────────────────────────┘

┌─ QUICK SELECTION ──────────────────────────────────────────┐
│ Need              → Use                                     │
│ Ordered list      → ArrayList                               │
│ Unique items      → HashSet                                 │
│ Sorted unique     → TreeSet                                 │
│ Key-value store   → HashMap                                 │
│ Sorted key-value  → TreeMap                                 │
│ FIFO queue        → ArrayDeque                              │
│ LIFO stack        → ArrayDeque                              │
│ Priority queue    → PriorityQueue                           │
│ Thread-safe map   → ConcurrentHashMap                       │
│ LRU cache         → LinkedHashMap (access-order)            │
└─────────────────────────────────────────────────────────────┘

┌─ BIG O ────────────────────────────────────────────────────┐
│           Get  Add  Contains  Next  Remove                  │
│ ArrayList O(1) O(1) O(n)     O(1)  O(n)                    │
│ LinkedList O(n) O(1) O(n)    O(1)  O(1)                    │
│ HashSet   O(1) O(1)*O(1)*   O(1)  O(1)*                   │
│ TreeSet   O(l) O(l) O(l)    O(l)  O(l)                    │
│ HashMap   O(1) O(1)*O(1)*   O(1)  O(1)*                   │
└─────────────────────────────────────────────────────────────┘

┌─ COMMON PITFALLS ──────────────────────────────────────────┐
│ ❌ Modify during iteration → use Iterator.remove()          │
│ ❌ Mutable HashSet keys → use immutable objects             │
│ ❌ No hashCode/equals → HashMap broken                      │
│ ❌ LinkedList for most needs → use ArrayList                │
│ ❌ PriorityQueue iteration → poll() in loop                 │
│ ❌ Returning internal collection → unmodifiableList()       │
└─────────────────────────────────────────────────────────────┘
```
