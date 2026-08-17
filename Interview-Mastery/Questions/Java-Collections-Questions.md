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
   - **If asked more:**
      - I can explain the hierarchy (Collection -> List/Set/Queue, Map separately)
      - Utility class Collections
      - How choosing wrong collection impacts performance
      - When I prefer arrays over collections for primitive-heavy data
2. Difference between `List`, `Set`, and `Map`.
   - **Answer:**
      - List allows duplicates and ordered access, Set ensures uniqueness, and Map stores key-value pairs
      - In my inventory system, I used List for partner records, Set for unique serial numbers, and Map for partner-to-count cache
   - **If asked more:**
      - I can explain implementation differences (ArrayList vs LinkedList, HashSet vs TreeSet, HashMap vs TreeMap)
      - Ordering guarantees
      - Null handling
      - Thread-safe variants like CopyOnWriteArrayList
3. Difference between `ArrayList` and `LinkedList`.
   - **Answer:**
      - ArrayList uses a dynamic array with O(1) random access, LinkedList uses doubly-linked nodes with O(n) access but O(1) insertion at ends
      - I always default to ArrayList because memory locality and cache performance are better for typical iteration patterns in my APIs
   - **If asked more:**
      - I can explain when LinkedList is actually useful (frequent front insertion, queue-like operations)
      - Memory overhead per element
      - How subList works with ArrayList
      - Why LinkedList is rarely the right choice in backend code
4. Difference between `HashSet` and `TreeSet`.
   - **Answer:**
      - HashSet uses hashCode for O(1) operations, TreeSet uses Red-Black tree for sorted order
      - I use HashSet for deduplication of serial records during validation, and would choose TreeSet only if I needed sorted iteration without extra sorting step
   - **If asked more:**
      - I can explain internal structures
      - How equals/hashCode must be consistent for HashSet
      - How TreeSet requires Comparable or Comparator
      - Performance tradeoffs
      - LinkedHashSet as a middle ground
5. Difference between `HashMap` and `Hashtable`.
   - **Answer:**
      - HashMap is not synchronized and allows null keys/values, while Hashtable is synchronized and does not allow nulls
      - In my projects, I always use HashMap and handle synchronization externally with ConcurrentHashMap when needed for thread safety
   - **If asked more:**
      - I can explain Hashtable's legacy status
      - How HashMap's default capacity and load factor differ
      - Performance comparison
      - Why ConcurrentHashMap replaced Hashtable
      - Iteration behavior differences
6. Difference between `HashMap` and `ConcurrentHashMap`.
   - **Answer:**
      - HashMap is not thread-safe, while ConcurrentHashMap uses internal locking for concurrent access without blocking reads
      - In my Kafka consumer where multiple threads update a shared cache, I used ConcurrentHashMap over HashMap with external sync
   - **If asked more:**
      - I can explain ConcurrentHashMap's bucket-level locking in Java 7 vs CAS-based approach in Java 8
      - How computeIfAbsent works atomically
      - Performance scaling
      - Why Collections.synchronizedMap is less efficient
7. Difference between `HashMap` and `LinkedHashMap`.
   - **Answer:**
      - LinkedHashMap maintains insertion order (or access order) via a doubly-linked list, while HashMap does not guarantee order
      - I used LinkedHashMap in my cold-chain project when I needed to preserve the order of sensor readings as they arrived
   - **If asked more:**
      - I can explain how LinkedHashMap enables LRU cache with removeEldestEntry()
      - Memory overhead of the linked list
      - Performance vs HashMap
      - Iteration order guarantees
8. Difference between `HashMap` and `TreeMap`.
   - **Answer:**
      - TreeMap stores keys in sorted order using Red-Black tree, while HashMap is unordered
      - I would use TreeMap if I needed range queries or sorted iteration, like generating partner reports alphabetically sorted by partner name
   - **If asked more:**
      - I can explain O(log n) vs O(1) performance
      - How TreeMap implements NavigableMap for ceiling/floor/lower/higher methods
      - Comparable vs Comparator
      - When TreeMap's sorting is unnecessary overhead
9. How does `HashMap` work internally?
   - **Answer:**
      - HashMap stores entries in an array of buckets
      - Each bucket is a linked list or tree
      - When I call put(key, value), HashMap computes key's hashCode, finds the bucket index, and places the entry there
      - On collision, entries chained in that bucket
   - **If asked more:**
      - I can explain hash function (hashing, bit-masking for index)
      - Treeification threshold (8 -> tree, 6 -> untree)
      - Initial capacity (16)
      - Load factor (0.75)
      - Rehashing logic
      - How Java 8 improved collision handling with TreeNode
10. What happens during hash collision?
   - **Answer:**
      - When two keys have the same bucket index, HashMap stores both entries as a linked list in that bucket
      - In Java 8, if the list exceeds 8 entries, it converts to a balanced tree for O(log n) lookup instead of O(n)
   - **If asked more:**
      - I can explain how equals() differentiates collided entries
      - The treeify threshold and untreeify threshold
      - How a bad hashCode implementation floods one bucket
      - How to design keys to minimize collisions
11. What is load factor?
   - **Answer:**
      - Load factor (default 0.75) determines when HashMap resizes: when size exceeds capacity * load factor
      - A lower factor wastes memory but reduces collisions; higher factor saves memory but increases lookup time
   - **If asked more:**
      - I can explain how to choose initial capacity if I know the expected size (capacity = expected / load factor + 1)
      - How resizing is expensive
      - How I pre-sized HashMaps when caching 10,000+ inventory records
12. What is rehashing?
   - **Answer:**
      - Rehashing is the process of resizing the HashMap's bucket array and redistributing all entries when the load factor threshold is crossed
      - It involves creating a new array (2x size), recomputing bucket indices, and moving entries, which is O(n) and expensive
   - **If asked more:**
      - I can explain how Java 8 avoids rehashing all keys (just tests bit for new index)
      - How concurrent resizing is avoided in ConcurrentHashMap
      - Why tuning initial capacity reduces rehashing overhead in large caches
13. Why should keys be immutable in a `HashMap`?
   - **Answer:**
      - Immutable keys guarantee hash code stability
      - If a key's hashCode changes after insertion, the HashMap cannot find it in the correct bucket, causing memory leaks and lookup failures
      - I always use String or Integer keys which are inherently immutable
   - **If asked more:**
      - I can explain precisely what happens when a mutable key's hash changes (the entry is orphaned)
      - How to detect such issues in production
      - How Java records make ideal keys with their built-in equals/hashCode
14. What happens if a mutable object is used as a key?
   - **Answer:**
      - If I modify a key after inserting it into a HashMap, the hash code changes but the entry stays in the original bucket
      - The entry becomes permanently lost — get() looks in the new bucket and finds nothing, while the old entry stays in memory
   - **If asked more:**
      - I can explain a real bug scenario I've seen where entity modification after map insertion caused data loss
      - How defensive copying helps
      - How to use immutable wrappers or records to prevent this
15. What is fail-fast iterator?
   - **Answer:**
      - A fail-fast iterator throws ConcurrentModificationException if the collection is structurally modified after the iterator is created
      - In my single-threaded code, modifying a list while iterating with for-each triggers this, so I use Iterator.remove() instead
   - **If asked more:**
      - I can explain the modCount field mechanism
      - Why fail-fast is a bug-detection feature not a guarantee
      - How to safely modify during iteration (CopyOnWriteArrayList or collect then remove)
      - Race conditions in concurrent scenarios
16. What is fail-safe iterator?
   - **Answer:**
      - A fail-safe iterator works on a snapshot of the collection, so modifications after creation do not throw exceptions
      - Iterators of ConcurrentHashMap and CopyOnWriteArrayList are fail-safe, which I rely on when iterating shared caches in Kafka consumers
   - **If asked more:**
      - I can explain the snapshot vs live-data approach
      - How ConcurrentHashMap's iterator reflects some concurrent updates but not all
      - Memory overhead of CopyOnWriteArrayList
      - Why fail-safe does not mean full thread safety
17. Difference between `Iterator` and `ListIterator`.
   - **Answer:**
      - ListIterator extends Iterator with bidirectional traversal, index access, and element modification/replacement
      - I use Iterator for general collection iteration and ListIterator when I need to traverse backwards or insert during iteration in my data processing
   - **If asked more:**
      - I can explain available methods (hasPrevious, previousIndex, set, add)
      - Which collections support ListIterator (List implementors)
      - Why there is no SetIterator — Set has no positional access
18. Difference between `Comparable` and `Comparator`.
   - **Answer:**
      - Comparable defines natural ordering inside the class (compareTo), Comparator is external sorting logic
      - In my projects, I implement Comparable in entity classes for default sorting, and create custom Comparators for specific report ordering requirements
   - **If asked more:**
      - I can explain how compareTo contract mirrors equals consistency
      - Lambda-based Comparator construction
      - Comparator.comparing() chaining
      - Null handling with nullsFirst/nullsLast
      - TreeSet/TreeMap requirements
19. What is priority queue?
   - **Answer:**
      - PriorityQueue orders elements by their natural order or a custom Comparator, not insertion order
      - The head is always the smallest element
      - I would use it for task scheduling scenarios, like processing inventory alerts by severity priority
   - **If asked more:**
      - I can explain heap-based implementation
      - O(log n) offer/poll
      - O(1) peek
      - Why it is not thread-safe
      - PriorityBlockingQueue for concurrent use
      - How Iterator does not guarantee priority order
20. What is blocking queue?
   - **Answer:**
      - BlockingQueue is a queue that blocks when taking from an empty queue or adding to a full queue
      - In my cold-chain project, we conceptually used this pattern for Kafka consumer message processing where consumers block until new sensor data arrives
   - **If asked more:**
      - I can explain implementations like ArrayBlockingQueue (bounded, fair)
      - LinkedBlockingQueue
      - How producers/consumers coordinate
      - Delay queues for scheduled processing
      - How thread pools internally use BlockingQueue
21. What is copy-on-write collection?
   - **Answer:**
      - CopyOnWriteArrayList creates a new underlying array on every modification, so iterators never see stale data and never throw ConcurrentModificationException
      - I would use it for read-heavy, write-rare scenarios like configuration lists read by multiple threads
   - **If asked more:**
      - I can explain why it is expensive for writes (O(n) array copy)
      - Snapshot iterator semantics
      - Memory implications
      - How it differs from ConcurrentHashMap's design
      - When the cost is justified (frequent reads, rare writes)
22. When would you use `ArrayList`?
   - **Answer:**
      - I use ArrayList almost always for ordered data because of O(1) random access, cache-friendly memory layout, and no per-element memory overhead
      - In my inventory system, I store lists of serial records in ArrayList for fast indexed access during validation
   - **If asked more:**
      - I can explain the dynamic growth (newSize = old + old >> 1)
      - Why toArray() with size zero is fastest
      - SubList view behavior
      - When to use Arrays.asList vs ArrayList constructor
23. When would you use `LinkedList`?
   - **Answer:**
      - I rarely use LinkedList in backend code
      - It makes sense when I need frequent insertions at both ends (deque operations) and do not need random access
      - Even then, ArrayDeque usually outperforms LinkedList for stack/queue use cases in my experience
   - **If asked more:**
      - I can explain why LinkedList has poor cache locality
      - Higher memory per element (2 references + object header)
      - How it implements both List and Deque
      - Why ArrayList + ArrayDeque cover 99% of backend needs
24. When would you use `ConcurrentHashMap`?
   - **Answer:**
      - I use ConcurrentHashMap whenever a HashMap is accessed by multiple threads, such as my cache of partner serial counts that was read/updated by multiple Kafka consumer threads
      - It gives me thread safety without explicit synchronization overhead
   - **If asked more:**
      - I can explain the CAS + synchronized segment design in Java 8
      - Why computeIfAbsent is atomic
      - How to safely iterate and update concurrently
      - When to use ConcurrentHashMap vs Collections.synchronizedMap
      - Performance tradeoffs
25. When would you use `TreeMap`?
   - **Answer:**
      - I use TreeMap when I need keys sorted automatically, like maintaining a time-sorted map of sensor events by timestamp for dashboard display
      - TreeMap's NavigableMap features (ceilingKey, subMap) are useful for range queries on sorted data
   - **If asked more:**
      - I can explain O(log n) performance for all operations
      - Memory overhead of tree nodes
      - When to prefer TreeMap over sorting HashMap entries after retrieval
      - How ConcurrentSkipListMap is the thread-safe alternative
26. How does `HashSet` work internally?
   - **Answer:**
      - HashSet is internally backed by a HashMap
      - When I `add(element)`, Java stores the element as the *key* of the HashMap and a constant dummy object (`PRESENT`) as the value
      - Uniqueness is enforced by the HashMap's key contract: two elements are considered duplicates when their `hashCode()` is equal AND `equals()` returns true
      - `contains()` just calls `map.containsKey()`, so lookup is O(1) on average using the same hashing, bucket, and treeification logic as HashMap
   - **If asked more:**
      - I can explain that this is why a HashSet requires proper `equals()`/`hashCode()` — if two objects are logically equal but have different hash codes, both get stored
      - I can mention that HashSet uses the HashMap's capacity (16), load factor (0.75), resizing, and collision handling
      - I'd also mention that `LinkedHashSet` extends HashSet to add insertion-order iteration, and `TreeSet` is backed by a TreeMap for sorted iteration
      - In my inventory dedup, I used HashSet on serial numbers to drop duplicate records before validation

