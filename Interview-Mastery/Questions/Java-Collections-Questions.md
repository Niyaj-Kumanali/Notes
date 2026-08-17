# Java Collections Questions

## Questions

1. What is Java Collections Framework?
2. Difference between `List`, `Set`, and `Map`.
3. Difference between `ArrayList` and `LinkedList`.
4. Difference between `HashSet` and `TreeSet`.
5. Difference between `HashMap` and `Hashtable`.
6. Difference between `HashMap` and `ConcurrentHashMap`.
7. Difference between `HashMap` and `LinkedHashMap`.
8. Difference between `HashMap` and `TreeMap`.
9. How does `HashMap` work internally?
10. What happens during hash collision?
11. What is load factor?
12. What is rehashing?
13. Why should keys be immutable in a `HashMap`?
14. What happens if a mutable object is used as a key?
15. What is fail-fast iterator?
16. What is fail-safe iterator?
17. Difference between `Iterator` and `ListIterator`.
18. Difference between `Comparable` and `Comparator`.
19. What is priority queue?
20. What is blocking queue?
21. What is copy-on-write collection?
22. When would you use `ArrayList`?
23. When would you use `LinkedList`?
24. When would you use `ConcurrentHashMap`?
25. When would you use `TreeMap`?
26. How does `HashSet` work internally?

---

## Answers

1. What is Java Collections Framework?
   - **Answer:**
      - The Collections Framework provides interfaces and implementations for storing and manipulating groups of objects like List, Set, and Map
      - In my projects, I use ArrayList for ordered data, HashMap for lookups, and HashSet for deduplication of serial records in inventory validation
      - Collection hierarchy: Collection interface is the root, with List, Set, and Queue extending it; Map is a separate interface not part of the Collection hierarchy
      - Collections utility class provides static methods like sort(), reverse(), synchronized wrappers (synchronizedList/Map/Set), and unmodifiableList/Map/Set
      - Choosing wrong collection impacts performance — LinkedList for random access is O(n) vs ArrayList O(1); TreeMap for simple lookups is O(log n) vs HashMap O(1)
      - Arrays are preferred over collections for primitive-heavy data because they avoid autoboxing overhead and use less memory
2. Difference between `List`, `Set`, and `Map`.
   - **Answer:**
      - List allows duplicates and ordered access, Set ensures uniqueness, and Map stores key-value pairs
      - In my inventory system, I used List for partner records, Set for unique serial numbers, and Map for partner-to-count cache
      - ArrayList uses dynamic array for O(1) access; LinkedList uses doubly-linked nodes for O(1) insert/remove at ends. HashSet uses hash table for O(1) uniqueness checks; TreeSet uses Red-Black tree for sorted uniqueness. HashMap uses hash table for O(1) key-value lookups; TreeMap uses Red-Black tree for sorted key-value pairs
      - List preserves insertion order; Set has HashSet (no order), LinkedHashSet (insertion order), TreeSet (sorted); Map has HashMap (no order), LinkedHashMap (insertion/access order), TreeMap (sorted by key)
      - List allows any number of null elements; Set allows one null (HashSet/LinkedHashSet), TreeSet rejects null if Comparable doesn't handle it; Map allows one null key and multiple null values (HashMap/LinkedHashMap), TreeMap rejects null keys
      - Thread-safe variants include CopyOnWriteArrayList (snapshot iteration, write-heavy copy), Collections.synchronizedList/Map/Set (coarse-grained synchronization), and ConcurrentHashMap/CopyOnWriteArraySet for concurrent access
3. Difference between `ArrayList` and `LinkedList`.
   - **Answer:**
      - ArrayList uses a dynamic array with O(1) random access, LinkedList uses doubly-linked nodes with O(n) access but O(1) insertion at ends
      - I always default to ArrayList because memory locality and cache performance are better for typical iteration patterns in my APIs
      - LinkedList is useful when you need frequent insertions and deletions at both ends without random access, such as implementing a queue or deque
      - Each LinkedList node stores two pointers (next/prev) plus object header, adding ~24-32 bytes overhead per element vs ArrayList's compact array storage
      - ArrayList.subList() returns a view backed by the original list, so modifications to the sublist reflect in the parent list and vice versa
      - LinkedList is rarely chosen in backend code because ArrayList's cache-friendly contiguous memory outperforms it for nearly all real workloads, and ArrayDeque is better for queue/deque use cases
4. Difference between `HashSet` and `TreeSet`.
   - **Answer:**
      - HashSet uses hashCode for O(1) operations, TreeSet uses Red-Black tree for sorted order
      - I use HashSet for deduplication of serial records during validation, and would choose TreeSet only if I needed sorted iteration without extra sorting step
      - HashSet uses a HashMap internally with elements as keys and a dummy PRESENT object as values for O(1) operations; TreeSet uses a TreeMap (Red-Black tree) internally for O(log n) sorted operations
      - HashSet relies on both hashCode() and equals() — if two objects are logically equal but have different hash codes, HashSet stores both as separate entries, breaking uniqueness
      - TreeSet requires elements to implement Comparable or a Comparator to be provided at construction time; without it, it throws ClassCastException when adding elements
      - HashSet offers O(1) add/remove/contains; TreeSet offers O(log n) for the same operations but provides sorted iteration and range queries (headSet, tailSet, subSet)
      - LinkedHashSet extends HashSet by maintaining a doubly-linked list for insertion-order iteration with O(1) operations, trading slight memory overhead for predictable ordering
5. Difference between `HashMap` and `Hashtable`.
   - **Answer:**
      - HashMap is not synchronized and allows null keys/values, while Hashtable is synchronized and does not allow nulls
      - In my projects, I always use HashMap and handle synchronization externally with ConcurrentHashMap when needed for thread safety
      - Hashtable is a legacy class from Java 1.0, synchronized on the entire map, and has been largely replaced by ConcurrentHashMap
      - HashMap defaults to initial capacity 16 and load factor 0.75, while Hashtable defaults to initial capacity 11 with load factor 0.75
      - HashMap is significantly faster because it avoids synchronization overhead; Hashtable locks the entire map for every operation even in single-threaded scenarios
      - ConcurrentHashMap replaced Hashtable because it provides fine-grained locking (bucket-level or CAS-based) instead of full-map synchronization, enabling true concurrent access
      - HashMap's iterator is fail-fast, while Hashtable's enumerator is legacy and not fail-fast — Hashtable uses Enumeration which does not detect concurrent modification
6. Difference between `HashMap` and `ConcurrentHashMap`.
   - **Answer:**
      - HashMap is not thread-safe, while ConcurrentHashMap uses internal locking for concurrent access without blocking reads
      - In my Kafka consumer where multiple threads update a shared cache, I used ConcurrentHashMap over HashMap with external sync
      - Java 7 ConcurrentHashMap used Segment-level locking (16 segments by default), while Java 8 replaced this with CAS operations on individual nodes plus synchronized blocks on specific buckets for finer granularity
      - computeIfAbsent computes the mapping function and inserts the result atomically under the bucket lock, preventing duplicate computation when multiple threads race on the same key
      - ConcurrentHashMap scales well under high concurrency because reads are lock-free (volatile writes), writes only lock the affected bucket, and size() uses a non-blocking algorithm (baseCount + CounterCells)
      - Collections.synchronizedMap wraps every method call with synchronized(mutex), serializing all access including reads, while ConcurrentHashMap allows concurrent reads and fine-grained write locking
7. Difference between `HashMap` and `LinkedHashMap`.
   - **Answer:**
      - LinkedHashMap maintains insertion order (or access order) via a doubly-linked list, while HashMap does not guarantee order
      - I used LinkedHashMap in my cold-chain project when I needed to preserve the order of sensor readings as they arrived
      - LinkedHashMap with access-order mode (constructor flag true) moves accessed entries to the end, and overriding removeEldestEntry() to return true when size > threshold creates a built-in LRU cache
      - LinkedHashMap adds memory overhead of two pointers (before/after) per entry for maintaining the doubly-linked list, increasing per-entry cost
      - LinkedHashMap is slightly slower than HashMap due to maintaining the linked list on every insertion and access, though the difference is negligible for most use cases
      - LinkedHashMap guarantees iteration in insertion order by default, or in access order if configured, making iteration predictable unlike HashMap's undefined order
8. Difference between `HashMap` and `TreeMap`.
   - **Answer:**
      - TreeMap stores keys in sorted order using Red-Black tree, while HashMap is unordered
      - I would use TreeMap if I needed range queries or sorted iteration, like generating partner reports alphabetically sorted by partner name
      - TreeMap operations (get, put, remove) are O(log n) due to Red-Black tree traversal, while HashMap operations are O(1) amortized through hash-based bucket lookup
      - TreeMap implements NavigableMap which provides navigation methods: ceilingKey(key) returns least key ≥ given, floorKey(key) returns greatest key ≤ given, lowerKey(key) and higherKey(key) for strict comparisons
      - Comparable defines natural ordering within the class via compareTo(), Comparator provides external ordering logic passed to TreeMap's constructor, enabling multiple sort strategies without modifying the key class
      - TreeMap's sorting overhead is unnecessary when you only need fast key lookups without iteration ordering — HashMap provides O(1) lookups vs TreeMap's O(log n), making HashMap preferred for pure lookup tables
9. How does `HashMap` work internally?
   - **Answer:**
      - HashMap stores entries in an array of buckets
      - Each bucket is a linked list or tree
      - When I call put(key, value), HashMap computes key's hashCode, finds the bucket index, and places the entry there
      - On collision, entries chained in that bucket
      - HashMap computes hashCode(), applies supplemental hash (XOR with upper 16 bits), then uses bit-masking (n-1) where n is array length to map the hash to a bucket index
      - When a bucket's linked list exceeds 8 entries (TREEIFY_THRESHOLD), it converts to a balanced Red-Black tree (TreeNode) for O(log n) lookup; when tree size drops below 6 (UNTREEIFY_THRESHOLD), it converts back to a linked list
      - HashMap starts with an internal array of 16 buckets, which is always rounded up to the nearest power of 2 for efficient bit-masking index computation
      - Load factor of 0.75 balances memory usage and collision probability — the map resizes when entries exceed capacity × load factor (e.g., 16 × 0.75 = 12)
      - Rehashing creates a new array of double the capacity, then iterates all entries, recomputing each bucket index with the new capacity mask to redistribute entries
      - Java 8 introduced treeification so that buckets with many collisions use a Red-Black tree instead of a long linked list, improving worst-case lookup from O(n) to O(log n)
10. What happens during hash collision?
   - **Answer:**
      - When two keys have the same bucket index, HashMap stores both entries as a linked list in that bucket
      - In Java 8, if the list exceeds 8 entries, it converts to a balanced tree for O(log n) lookup instead of O(n)
      - When two keys hash to the same bucket, HashMap walks the chain and uses equals() to find the exact matching key, storing or updating the value for that specific key
      - Treeify threshold is 8 (linked list converts to tree when chain length exceeds 8), untreeify threshold is 6 (tree reverts to linked list when resized down to 6 or fewer entries)
      - A poorly designed hashCode that returns the same value for many keys causes all entries to land in one bucket, degrading performance from O(1) to O(n) linked list traversal
      - Good hashCode implementations distribute keys uniformly across the hash space using prime numbers or bit manipulation, and are consistent with equals() to ensure correct behavior
11. What is load factor?
   - **Answer:**
      - Load factor (default 0.75) determines when HashMap resizes: when size exceeds capacity * load factor
      - A lower factor wastes memory but reduces collisions; higher factor saves memory but increases lookup time
      - To avoid resizing, set initial capacity to expected_entries / load_factor + 1, e.g., for 1000 entries: capacity = 1000 / 0.75 + 1 = 1334, which gets rounded up to the nearest power of 2 (2048)
      - Resizing allocates a new array (double size), rehashes every entry to find its new bucket, and copies everything over — this is O(n) and causes a temporary spike in CPU and GC pressure
      - When building a cache of 10,000+ inventory records, I pre-sized the HashMap with `new HashMap<>(13334)` to avoid multiple expensive resize cycles during bulk loading
12. What is rehashing?
   - **Answer:**
      - Rehashing is the process of resizing the HashMap's bucket array and redistributing all entries when the load factor threshold is crossed
      - It involves creating a new array (2x size), recomputing bucket indices, and moving entries, which is O(n) and expensive
      - Java 8 rehashing checks if the old hash bit (e.hash & oldCap) is 0 or 1: if 0, the entry stays at the same index; if 1, it moves to (old index + old capacity), avoiding full hashCode recomputation
      - ConcurrentHashMap uses a sizeCtl volatile variable as a resize barrier — only one thread can win the CAS to set sizeCtl to negative (resizing state), forcing others to help transfer entries rather than start a new resize
      - Pre-sizing the map to the expected number of entries eliminates or minimizes resize operations, which is critical for performance in large caches where rehashing 100K+ entries causes noticeable latency spikes
13. Why should keys be immutable in a `HashMap`?
   - **Answer:**
      - Immutable keys guarantee hash code stability
      - If a key's hashCode changes after insertion, the HashMap cannot find it in the correct bucket, causing memory leaks and lookup failures
      - I always use String or Integer keys which are inherently immutable
      - When a key's hashCode changes after insertion, the entry remains in the old bucket but future get() calls compute the new hash and look in a different bucket, orphaning the entry in memory and causing a logical memory leak
      - Detect orphaned entries by monitoring heap usage for unexplained growth, using profilers to inspect bucket chains, or comparing map.size() vs iteration count to find unreachable entries
      - Java records generate symmetric, consistent equals() and hashCode() based on all declared components, and are immutable by design, making them ideal HashMap keys
14. What happens if a mutable object is used as a key?
   - **Answer:**
      - If I modify a key after inserting it into a HashMap, the hash code changes but the entry stays in the original bucket
      - The entry becomes permanently lost — get() looks in the new bucket and finds nothing, while the old entry stays in memory
      - An entity used as a map key had its id field updated after insertion; the get() call with the same object failed because the hash bucket changed, silently returning null and causing downstream NullPointerExceptions
      - Defensive copying involves inserting a copy of the key into the map and storing the original as part of the value, ensuring the map key's state cannot be accidentally mutated after insertion
      - Wrap mutable keys in unmodifiable classes or use Java records which are immutable by construction, guaranteeing hash code stability throughout the key's lifetime in the map
15. What is fail-fast iterator?
   - **Answer:**
      - A fail-fast iterator throws ConcurrentModificationException if the collection is structurally modified after the iterator is created
      - In my single-threaded code, modifying a list while iterating with for-each triggers this, so I use Iterator.remove() instead
      - Each ArrayList and similar collection maintains a modCount variable that increments on structural modification; the iterator snapshots modCount at creation and throws ConcurrentModificationException if it detects a mismatch during next()/remove()
      - Fail-fast does not guarantee detection in all cases — concurrent modifications may slip through, and the documentation explicitly states it makes no guarantees, only best-effort detection
      - Safely modify by iterating over a copy (new ArrayList<>(list)), using CopyOnWriteArrayList which iterates a snapshot, or collecting items to remove and deleting them after iteration
      - In concurrent scenarios, fail-fast provides no protection — one thread may modify while another iterates without ConcurrentModificationException, leading to inconsistent state without any error
16. What is fail-safe iterator?
   - **Answer:**
      - A fail-safe iterator works on a snapshot of the collection, so modifications after creation do not throw exceptions
      - Iterators of ConcurrentHashMap and CopyOnWriteArrayList are fail-safe, which I rely on when iterating shared caches in Kafka consumers
      - CopyOnWriteArrayList iterates a snapshot array (copy at iterator creation), while ConcurrentHashMap iterates a weakly consistent view that may reflect some but not all concurrent modifications
      - ConcurrentHashMap's iterator is weakly consistent — it reflects the state of the map at or since iterator creation, but may not show modifications made after the iterator was created
      - CopyOnWriteArrayList copies the entire underlying array on every add/set/remove, making writes O(n) in both time and memory, which is acceptable only for read-heavy scenarios
      - Fail-safe iterators avoid ConcurrentModificationException but do not synchronize writes — concurrent modifications to the underlying collection still require external synchronization or appropriate concurrent collections for data consistency
17. Difference between `Iterator` and `ListIterator`.
   - **Answer:**
      - ListIterator extends Iterator with bidirectional traversal, index access, and element modification/replacement
      - I use Iterator for general collection iteration and ListIterator when I need to traverse backwards or insert during iteration in my data processing
      - ListIterator adds hasPrevious(), previous(), nextIndex(), previousIndex(), set(element), and add(element) beyond Iterator's hasNext(), next(), and remove()
      - ListIterator is returned only by List implementations (ArrayList, LinkedList, Vector, CopyOnWriteArrayList) via listIterator() method; it is not available for Set or Map
      - Set has no positional concept (no index), so bidirectional traversal with position-based operations like set() and add() at a specific position has no meaning for Set elements
18. Difference between `Comparable` and `Comparator`.
   - **Answer:**
      - Comparable defines natural ordering inside the class (compareTo), Comparator is external sorting logic
      - In my projects, I implement Comparable in entity classes for default sorting, and create custom Comparators for specific report ordering requirements
      - compareTo() should be consistent with equals() — if compareTo returns 0, equals should return true — otherwise sorted collections like TreeSet behave unpredictably when combined with equals-based operations
      - Comparators can be created inline with lambdas: `(a, b) -> a.getName().compareTo(b.getName())`, providing concise custom sorting without creating a separate class
      - Comparator.comparing(keyExtractor) creates a Comparator from a function, and .thenComparing() chains secondary sorts: `Comparator.comparing(Person::getLastName).thenComparing(Person::getFirstName)`
      - Comparator.nullsFirst(comparator) and Comparator.nullsLast(comparator) handle null values by placing them at the beginning or end of sorted order, preventing NullPointerException
      - TreeSet and TreeMap require either the key type to implement Comparable or a Comparator to be provided at construction, otherwise they throw ClassCastException at runtime
19. What is priority queue?
   - **Answer:**
      - PriorityQueue orders elements by their natural order or a custom Comparator, not insertion order
      - The head is always the smallest element
      - I would use it for task scheduling scenarios, like processing inventory alerts by severity priority
      - PriorityQueue is backed by a binary heap (a complete binary tree stored in an array), where the parent at index i has children at 2i+1 and 2i+2, maintaining the heap property for efficient min extraction
      - offer() inserts at the end of the heap array and bubbles up to restore heap order in O(log n); poll() removes the root, moves the last element to root, and sifts down in O(log n)
      - peek() returns the root element (smallest for min-heap) in O(1) without modifying the heap structure
      - PriorityQueue is not synchronized, so concurrent offer/poll operations can corrupt the internal array or violate the heap invariant — use PriorityBlockingQueue for concurrent access
      - PriorityBlockingQueue is the thread-safe version of PriorityQueue using ReentrantLock for blocking offer() and take() operations, blocking when empty or bounded
      - PriorityQueue's iterator does not guarantee elements are returned in priority (sorted) order — it traverses the internal array in heap structure order, which is neither sorted nor insertion order
20. What is blocking queue?
   - **Answer:**
      - BlockingQueue is a queue that blocks when taking from an empty queue or adding to a full queue
      - In my cold-chain project, we conceptually used this pattern for Kafka consumer message processing where consumers block until new sensor data arrives
      - ArrayBlockingQueue is a bounded blocking queue backed by a circular array with ReentrantLock, and can be configured with fairness policy (fair=true gives waiting threads FIFO access)
      - LinkedBlockingQueue is an optionally bounded blocking queue backed by linked nodes, using separate locks for put and take to allow higher throughput than ArrayBlockingQueue in producer-consumer scenarios
      - Producers call put() which blocks when the queue is full, consumers call take() which blocks when the queue is empty — this naturally coordinates producer-consumer workflows without manual wait/notify
      - DelayQueue stores Delayed elements, where take() blocks until the element's delay expires, useful for scheduling tasks like expired session cleanup or delayed job execution
      - ThreadPoolExecutor uses BlockingQueue as its work queue — submitted Runnable/Callable tasks are queued, and the executor polls from the queue when threads are available
21. What is copy-on-write collection?
   - **Answer:**
      - CopyOnWriteArrayList creates a new underlying array on every modification, so iterators never see stale data and never throw ConcurrentModificationException
      - I would use it for read-heavy, write-rare scenarios like configuration lists read by multiple threads
      - Every add, set, or remove creates a full copy of the underlying array, making each write O(n) in both time and memory, which is prohibitive for write-heavy workloads
      - CopyOnWriteArrayList's iterator holds a reference to the array snapshot at creation time, so it sees a consistent view even if other threads modify the list concurrently
      - Each modification allocates a new array of the same or larger size, causing significant GC pressure under write-heavy scenarios — the old array is retained until all active iterators release it
      - ConcurrentHashMap uses fine-grained locking (CAS + synchronized per bucket) to allow concurrent reads and writes without copying, while CopyOnWriteArrayList copies the entire collection on every write
      - Copy-on-write is justified for read-heavy, write-rare scenarios like configuration lists, listener registries, or routing tables where iteration consistency outweighs write cost
22. When would you use `ArrayList`?
   - **Answer:**
      - I use ArrayList almost always for ordered data because of O(1) random access, cache-friendly memory layout, and no per-element memory overhead
      - In my inventory system, I store lists of serial records in ArrayList for fast indexed access during validation
      - ArrayList grows by 50% each time (newCapacity = oldCapacity + oldCapacity >> 1), balancing between excessive resizing and wasted memory
      - `toArray(new T[0])` is fastest because JVM (HotSpot) uses an intrinsic optimized path for zero-length arrays, avoiding unnecessary reflection overhead of determining array type at runtime
      - ArrayList.subList() returns a lightweight view backed by the parent list's internal array — structural modifications to either the sublist or parent are reflected in both and may throw ConcurrentModificationException
      - Arrays.asList returns a fixed-size list backed by the array (no structural changes allowed), while new ArrayList<>(collection) creates a mutable copy — use Arrays.asList for quick varargs list creation, ArrayList constructor for a modifiable copy
23. When would you use `LinkedList`?
   - **Answer:**
      - I rarely use LinkedList in backend code
      - It makes sense when I need frequent insertions at both ends (deque operations) and do not need random access
      - Even then, ArrayDeque usually outperforms LinkedList for stack/queue use cases in my experience
      - LinkedList nodes are scattered across the heap, so traversing the list causes frequent CPU cache misses compared to ArrayList's contiguous array that benefits from prefetching
      - Each LinkedList node requires two object references (next/prev) plus the Node object header (12-16 bytes), adding ~24-32 bytes overhead per element beyond the element's own memory
      - LinkedList implements both List and Deque interfaces, so it can function as a list with index access or as a double-ended queue with addFirst/addLast/removeFirst/removeLast
      - ArrayList provides efficient random access and iteration, ArrayDeque provides efficient stack/queue operations — together they cover virtually all collection use cases in backend development without LinkedList's overhead
24. When would you use `ConcurrentHashMap`?
   - **Answer:**
      - I use ConcurrentHashMap whenever a HashMap is accessed by multiple threads, such as my cache of partner serial counts that was read/updated by multiple Kafka consumer threads
      - It gives me thread safety without explicit synchronization overhead
      - Java 8 ConcurrentHashMap uses CAS for empty bucket insertions and synchronized blocks only on the specific bucket node being modified, eliminating the segment-level locking from Java 7
      - computeIfAbsent uses CAS or synchronized on the bucket node to ensure the mapping function is applied exactly once per key, even when multiple threads call it simultaneously for the same key
      - ConcurrentHashMap iterators are weakly consistent and safe to use while other threads modify the map — reads are lock-free and writes only lock the affected bucket
      - Use ConcurrentHashMap for high-concurrency scenarios requiring fine-grained locking; use Collections.synchronizedMap for low-contention cases where simple synchronized wrapping suffices
      - ConcurrentHashMap has slightly higher per-operation overhead than HashMap due to volatile reads and CAS, but scales linearly under concurrency where Collections.synchronizedMap becomes a bottleneck
25. When would you use `TreeMap`?
   - **Answer:**
      - I use TreeMap when I need keys sorted automatically, like maintaining a time-sorted map of sensor events by timestamp for dashboard display
      - TreeMap's NavigableMap features (ceilingKey, subMap) are useful for range queries on sorted data
      - TreeMap provides O(log n) for get, put, remove, and containsKey using Red-Black tree balancing, compared to HashMap's O(1) amortized but without ordering guarantees
      - Each TreeMap entry stores a TreeNode with parent, left, right, and color references, adding ~40 bytes overhead per entry compared to HashMap's simpler node structure
      - Prefer TreeMap when you frequently need sorted access or range queries (subMap, headMap, tailMap) — sorting HashMap entries repeatedly via Collections.sort() is O(n log n) each time vs TreeMap's built-in sorted structure
      - ConcurrentSkipListMap provides the same sorted map and NavigableMap functionality as TreeMap but with lock-free concurrent access using skip list data structure
26. How does `HashSet` work internally?
   - **Answer:**
      - HashSet is internally backed by a HashMap
      - When I `add(element)`, Java stores the element as the *key* of the HashMap and a constant dummy object (`PRESENT`) as the value
      - Uniqueness is enforced by the HashMap's key contract: two elements are considered duplicates when their `hashCode()` is equal AND `equals()` returns true
      - `contains()` just calls `map.containsKey()`, so lookup is O(1) on average using the same hashing, bucket, and treeification logic as HashMap
      - Proper equals()/hashCode() is critical for HashSet because equality checks use hashCode() first to locate the bucket, then equals() to confirm — if logically equal objects have different hashes, they end up in different buckets and both are stored, breaking uniqueness
      - HashSet inherits HashMap's defaults: initial capacity 16, load factor 0.75, power-of-2 capacity rounding, linked list to tree conversion at 8 collisions, and O(n) rehashing on resize
      - LinkedHashSet extends HashSet by adding a doubly-linked list through entries for predictable insertion-order iteration. TreeSet is backed by a TreeMap for sorted iteration with O(log n) operations
      - In inventory deduplication, I created a HashSet of serial numbers and filtered incoming records by checking contains(), dropping duplicates before validation logic
