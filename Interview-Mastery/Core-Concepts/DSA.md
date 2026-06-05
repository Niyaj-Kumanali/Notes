# Data Structures & Algorithms (DSA)

---

## 1. Executive Summary

### What Is It?
Data Structures are ways to organize and store data for efficient access and modification. Algorithms are step-by-step procedures for solving problems. Together, they form the foundation of computer science and software engineering.

### Why Does It Exist?
Without efficient data structures and algorithms, software would be:
- **Slow** — Searching 1M records would take minutes instead of milliseconds
- **Resource-heavy** — Memory usage would be orders of magnitude higher
- **Unscalable** — What works for 100 users fails for 100,000

### Real-World Use Cases
| Domain | Data Structure | Algorithm |
|--------|---------------|-----------|
| **Database indexing** | B-Tree, Hash Index | B-Tree search, hash lookup |
| **GPS navigation** | Graph | Dijkstra's, A* |
| **Web search** | Inverted Index | PageRank, TF-IDF |
| **Social media feeds** | Graph | Topological sort, BFS |
| **Caching** | HashMap, LRU Cache | LRU eviction |
| **Compression** | Huffman Tree | Huffman coding |
| **Network routing** | Trie | Longest prefix match |
| **Job scheduling** | Priority Queue (Heap) | Round-robin, priority-based |
| **E-commerce recommendations** | Matrix | Matrix factorization, KNN |
| **Text autocomplete** | Trie | DFS traversal |

### When to Think About DSA
- Processing large datasets (10K+ records)
- Performance requirements (latency < 100ms, throughput > 1000 req/s)
- Search, sort, or filter operations on large collections
- Pathfinding, optimization, or constraint satisfaction
- System design interviews (caching, indexing, queue design)

### When NOT to Over-Optimize
- Simple CRUD apps with < 1000 users
- When a database query or library function already solves the problem efficiently
- Before profiling — premature optimization is the root of all evil
- When readability and maintainability matter more (most business applications)

---

## 2. Core Theory

### Data Structures Overview

#### Arrays
**Description:** Contiguous block of memory storing elements of the same type.

```
Memory: [ 10 | 25 | 33 | 47 | 52 | 68 | 71 | 89 ]
Index:    0    1    2    3    4    5    6    7
```

| Operation | Time Complexity |
|-----------|----------------|
| Access by index | O(1) |
| Search (unsorted) | O(n) |
| Search (sorted, binary) | O(log n) |
| Insert at end | O(1) amortized |
| Insert at middle | O(n) |
| Delete | O(n) |

**Advantages:** Fast access, cache-friendly (contiguous memory), low overhead.

**Disadvantages:** Fixed size (static arrays), expensive inserts/deletes.

#### Linked Lists
**Description:** Nodes where each node contains data and a pointer to the next node.

```
Singly: head → [data|next] → [data|next] → [data|null]
Doubly: head → [prev|data|next] ↔ [prev|data|next] ↔ [prev|data|null]
```

| Operation | Singly | Doubly |
|-----------|--------|--------|
| Access | O(n) | O(n) |
| Insert at head | O(1) | O(1) |
| Insert at tail | O(n) | O(1) |
| Delete (given node) | O(n) | O(1) |

**Advantages:** Dynamic size, O(1) inserts at head, no memory waste for resizing.

**Disadvantages:** No O(1) random access, extra memory per node (pointer), poor cache locality.

#### Stacks
**Description:** LIFO (Last-In-First-Out) structure.

```
Push ← [ 3 | 7 | 1 | 9 ] → Pop
        ↑     ↑     ↑   ↑
      bottom         top
```

| Operation | Time |
|-----------|------|
| Push | O(1) |
| Pop | O(1) |
| Peek | O(1) |

**Use cases:** Undo/redo, expression evaluation, backtracking, call stack.

#### Queues
**Description:** FIFO (First-In-First-Out) structure.

```
Enqueue → [ 3 | 7 | 1 | 9 ] → Dequeue
          ↑             ↑
        front          rear
```

**Variants:**
- **Circular queue** — efficient use of array space
- **Deque** — insert/remove from both ends
- **Priority queue** — elements ordered by priority (implemented via heap)
- **Blocking queue** — thread-safe, blocks on empty/full

| Operation | Queue | Deque | Priority Queue |
|-----------|-------|-------|----------------|
| Insert | O(1) | O(1) | O(log n) |
| Remove | O(1) | O(1) | O(log n) |
| Peek | O(1) | O(1) | O(1) |

#### Hash Tables (HashMap / Dictionary)
**Description:** Key-value store using a hash function to compute an index into an array of buckets.

```
Key "John" → hash() → index 3 → [bucket] → ("John", 25)
Key "Jane" → hash() → index 7 → [bucket] → ("Jane", 30)
Key "Bob"  → hash() → index 3 → [bucket] → ("John", 25) → ("Bob", 22)  [collision - chaining]
```

| Operation | Average | Worst Case |
|-----------|---------|------------|
| Insert | O(1) | O(n) (many collisions) |
| Search | O(1) | O(n) |
| Delete | O(1) | O(n) |

**Collision resolution methods:**
- **Chaining** — each bucket holds a linked list of entries
- **Open addressing** — linear probing, quadratic probing, double hashing

**Load factor and rehashing:** When load factor exceeds threshold (default 0.75 in Java), the table grows (~2x) and all entries are rehashed.

**Important for interviews:** Understand how Java's `HashMap` works, why `hashCode()` and `equals()` matter, how `ConcurrentHashMap` segments for thread safety.

#### Trees
**Binary Tree:**
```
        1
       / \
      2   3
     / \
    4   5
```

**Binary Search Tree (BST):** Left child < parent < right child
```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \
      4   7
```

| Operation | BST (balanced) | BST (skewed) |
|-----------|----------------|---------------|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |

**Tree Traversals:**
- **In-order** (LNR): 1, 3, 4, 6, 7, 8, 10, 14 → sorted order for BST
- **Pre-order** (NLR): 8, 3, 1, 6, 4, 7, 10, 14
- **Post-order** (LRN): 1, 4, 7, 6, 3, 14, 10, 8
- **Level-order** (BFS): 8, 3, 10, 1, 6, 14, 4, 7

**Balanced Trees:**
- **AVL Tree** — maintains height difference ≤ 1 via rotations
- **Red-Black Tree** — maintains approximate balance via color rules (used in Java `TreeMap`)
- **B-Tree** — optimized for disk I/O (database indexes)

**Heaps:**
```
Min Heap:       Max Heap:
    1                9
   / \              / \
  3   5            8   6
 / \ / \          / \ / \
4  8 7 9         4  5 3 1
```

| Operation | Heap |
|-----------|------|
| Find min/max | O(1) |
| Insert | O(log n) |
| Extract min/max | O(log n) |
| Heapify (build) | O(n) |

#### Graphs
**Types:** Directed, undirected, weighted, unweighted, cyclic, acyclic.

**Representations:**
```
Adjacency Matrix:       Adjacency List:
     A  B  C  D         A → [B, C]
A    0  1  1  0         B → [A, D]
B    1  0  0  1         C → [A, D]
C    1  0  0  1         D → [B, C]
D    0  1  1  0
```

| Operation | Matrix | List |
|-----------|--------|------|
| Space | O(V²) | O(V + E) |
| Add edge | O(1) | O(1) |
| Check edge | O(1) | O(degree) |
| Iterate neighbors | O(V) | O(degree) |

**Graph Algorithms:**
- **BFS** — shortest path in unweighted graphs, level-order traversal
- **DFS** — topological sort, cycle detection, connected components
- **Dijkstra** — shortest path in weighted graphs (non-negative weights)
- **Bellman-Ford** — shortest path with negative weights
- **Floyd-Warshall** — all-pairs shortest path
- **Prim's / Kruskal's** — Minimum Spanning Tree
- **Topological Sort** — ordering for DAGs (Kahn's algorithm)

#### Tries (Prefix Trees)
```
        root
       / | \
      c  d  p
     /   |   \
    a    o    r
   / \   |    |
  t   r  g    e
     /        |
    d        fix → word
   /
  s → word
```

Used for: autocomplete, spell checking, IP routing, dictionary.

| Operation | Time |
|-----------|------|
| Insert | O(L) where L = word length |
| Search | O(L) |
| Prefix search | O(L + results) |

### Algorithm Paradigms

#### Brute Force
Try all possibilities. O(n!) or O(2ⁿ) typical.
- **Use when:** Input size is very small (< 10-15)
- **Example:** Traveling Salesman Problem for 8 cities

#### Divide and Conquer
Split problem into subproblems, solve recursively, combine results.
- **Examples:** Merge sort, Quick sort, Binary search, Strassen's matrix multiplication
- **Key:** Subproblems must be independent

#### Greedy
Make locally optimal choice at each step.
- **Examples:** Dijkstra's, Prim's, Huffman coding, coin change
- **Key:** Greedy choice property + optimal substructure

#### Dynamic Programming
Solve subproblems once, cache results (memoization/tabulation).
- **Key:** Optimal substructure + overlapping subproblems
- **Top-down:** Recursion + memoization
- **Bottom-up:** Iterative table filling

**Classic DP problems:**
- Fibonacci, Knapsack, Longest Common Subsequence, Longest Increasing Subsequence
- Edit Distance, Matrix Chain Multiplication, Coin Change
- Subset Sum, Rod Cutting, Palindromic Substrings

#### Backtracking
Try all possibilities, backtrack when a dead end is reached.
- **Examples:** N-Queens, Sudoku Solver, Permutations, Subsets
- **Pruning:** Cut off branches early to reduce search space

---

## 3. Under-the-Hood Deep Dive

### Memory Hierarchy and DSA Performance

```
Register    ~1 cycle     (few hundred bytes)
L1 Cache    ~3 cycles    (32KB per core)
L2 Cache    ~10 cycles   (256KB per core)
L3 Cache    ~30-50 cycles (8-32MB shared)
RAM         ~100-200 ns  (GB)
SSD         ~50-150 µs   (TB)
HDD         ~5-10 ms     (TB)
```

**Why arrays are fast:** Elements are contiguous in memory. When you access array[0], the CPU loads a cache line (64 bytes) containing array[0] through array[15] (for 4-byte ints). Sequential access → cache hits → fast.

**Why linked lists are slow:** Nodes are scattered in memory. Each node access is a potential cache miss. Traversing a linked list is 10-100x slower than iterating an array for large datasets.

### Java Collection Internals

| Collection | Internal Structure | Get | Insert | Delete | Memory |
|-----------|-------------------|-----|--------|--------|--------|
| ArrayList | Resizable array | O(1) | O(n) mid, O(1) end | O(n) | 4-8 bytes per element (more with resizing slack) |
| LinkedList | Doubly linked list | O(n) | O(1) head/tail | O(1) given node | 24-32 bytes per element (extra for prev/next) |
| HashMap | Array of buckets (trees for collisions) | O(1) avg | O(1) avg | O(1) avg | ~32 bytes per entry + key + value |
| TreeMap | Red-Black tree | O(log n) | O(log n) | O(log n) | ~40 bytes per entry |
| HashSet | HashMap-backed | O(1) avg | O(1) avg | O(1) avg | Same as HashMap |
| PriorityQueue | Binary heap (array) | O(1) peek | O(log n) | O(log n) | Array overhead |

### How HashMap Works in Java (JDK 17)

1. `hashCode()` generates int
2. `(n - 1) & hash` determines bucket index (n = table size, power of 2)
3. If bucket empty, store entry
4. If collision: check equals(). If same key, replace. If different key, chain or tree.

**Treeification:** When bucket has ≥ 8 entries AND table size ≥ 64, the linked list converts to a Red-Black tree. Treeification threshold = 8, detreeify threshold = 6.

**Resizing:** When load factor > 0.75, table doubles (to next power of 2). All entries rehashed. Cost: O(n) per resize, amortized O(1) per insert.

### Recursion vs Iteration: Stack Behavior

```java
// Recursive: O(n) stack frames
int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

// Iterative: O(1) stack
int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; i++) result *= i;
    return result;
}
```

**Stack depth limits:** Default Java thread stack ≈ 1MB. Each frame ~ 1-2KB. Max recursion ≈ 500-1000 calls. Beyond that → `StackOverflowError`.

### Big O Notation and Growth Rates

| Notation | Name | n = 10 | n = 100 | n = 1000 | n = 1M |
|----------|------|--------|---------|----------|--------|
| O(1) | Constant | 1 | 1 | 1 | 1 |
| O(log n) | Logarithmic | 3 | 7 | 10 | 20 |
| O(n) | Linear | 10 | 100 | 1000 | 1M |
| O(n log n) | Linearithmic | 10 | 700 | 10,000 | 20M |
| O(n²) | Quadratic | 100 | 10,000 | 1M | 1T |
| O(2ⁿ) | Exponential | 1,024 | 10³⁰ | impossible | impossible |
| O(n!) | Factorial | 3,628,800 | impossible | impossible | impossible |

**Calculating Big O:**
1. Drop constants: O(2n) → O(n)
2. Drop non-dominant terms: O(n² + n) → O(n²)
3. Different inputs? Use different variables: O(n + m)
4. Loop inside loop → multiply: O(n * m)
5. Recursive calls: O(branches^depth)

---

## 4. Production Code Examples (Java + Spring Boot)

### 4.1 LRU Cache Implementation

```java
import java.util.*;

public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head;
    private final Node<K, V> tail;

    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>(capacity);
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        moveToHead(node);
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> node = map.get(key);
        if (node != null) {
            node.value = value;
            moveToHead(node);
            return;
        }
        if (map.size() >= capacity) {
            Node<K, V> evicted = removeTail();
            map.remove(evicted.key);
        }
        node = new Node<>(key, value);
        map.put(key, node);
        addToHead(node);
    }

    private void addToHead(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void moveToHead(Node<K, V> node) {
        removeNode(node);
        addToHead(node);
    }

    private Node<K, V> removeTail() {
        Node<K, V> node = tail.prev;
        removeNode(node);
        return node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
}
```

### 4.2 Trie for Autocomplete (Spring Service)

```java
@Component
public class AutocompleteService {
    private final TrieNode root = new TrieNode();

    static class TrieNode {
        Map<Character, TrieNode> children = new HashMap<>();
        boolean isEnd;
    }

    @PostConstruct
    public void loadDictionary() {
        // Load from database or file
        List<String> words = wordRepository.findAllWords();
        words.forEach(this::insert);
    }

    public void insert(String word) {
        TrieNode current = root;
        for (char c : word.toLowerCase().toCharArray()) {
            current = current.children.computeIfAbsent(c, k -> new TrieNode());
        }
        current.isEnd = true;
    }

    public List<String> autocomplete(String prefix, int limit) {
        TrieNode current = root;
        for (char c : prefix.toLowerCase().toCharArray()) {
            current = current.children.get(c);
            if (current == null) return List.of();
        }
        List<String> results = new ArrayList<>();
        dfs(current, new StringBuilder(prefix.toLowerCase()), results, limit);
        return results;
    }

    private void dfs(TrieNode node, StringBuilder prefix, List<String> results, int limit) {
        if (results.size() >= limit) return;
        if (node.isEnd) results.add(prefix.toString());
        for (char c = 'a'; c <= 'z'; c++) {
            TrieNode child = node.children.get(c);
            if (child != null) {
                prefix.append(c);
                dfs(child, prefix, results, limit);
                prefix.deleteCharAt(prefix.length() - 1);
            }
        }
    }
}
```

### 4.3 Top K Frequent Elements (Production)

```java
@Service
public class TrendingTopicsService {
    private final Map<String, Long> topicCounts = new ConcurrentHashMap<>();
    private final PriorityQueue<Map.Entry<String, Long>> minHeap;

    public TrendingTopicsService(@Value("${trending.topics.limit:10}") int limit) {
        this.minHeap = new PriorityQueue<>(limit,
            Map.Entry.comparingByValue());
    }

    public void recordTopicView(String topic) {
        topicCounts.merge(topic, 1L, Long::sum);
    }

    public List<String> getTopTrending(int k) {
        return topicCounts.entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                Map.Entry::getValue,
                (a, b) -> a,
                () -> new PriorityQueue<>(k, Map.Entry.comparingByValue())
            ))
            .stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .limit(k)
            .map(Map.Entry::getKey)
            .toList();
    }
}
```

### 4.4 Binary Search for Search Optimization

```java
// BAD: Linear search for lookups
public Product findProduct(List<Product> products, Long id) {
    for (Product p : products) {
        if (p.getId().equals(id)) return p; // O(n)
    }
    return null;
}

// GOOD: Use HashMap for O(1) lookup
@Service
public class ProductService {
    private final Map<Long, Product> productCache = new ConcurrentHashMap<>();

    @PostConstruct
    public void warmCache() {
        productRepository.findAll()
            .forEach(p -> productCache.put(p.getId(), p));
    }

    public Product findProduct(Long id) {
        return productCache.get(id); // O(1)
    }
}
```

### 4.5 Sliding Window — Rate Limiter

```java
@Component
public class SlidingWindowRateLimiter {
    private final long windowSizeMillis;
    private final int maxRequests;
    private final Map<String, Deque<Long>> requestLogs = new ConcurrentHashMap<>();

    public SlidingWindowRateLimiter(
            @Value("${rate.limit.window:1000}") long windowSizeMillis,
            @Value("${rate.limit.max:10}") int maxRequests) {
        this.windowSizeMillis = windowSizeMillis;
        this.maxRequests = maxRequests;
    }

    public boolean allowRequest(String clientId) {
        long now = System.currentTimeMillis();
        Deque<Long> timestamps = requestLogs.computeIfAbsent(
            clientId, k -> new LinkedList<>());

        synchronized (timestamps) {
            // Remove expired timestamps
            while (!timestamps.isEmpty() &&
                   timestamps.peekFirst() < now - windowSizeMillis) {
                timestamps.pollFirst();
            }

            if (timestamps.size() >= maxRequests) {
                return false; // Rate limited
            }

            timestamps.addLast(now);
            return true;
        }
    }
}
```

### 4.6 Graph-Based Dependencies — Topological Sort

```java
@Service
public class TaskSchedulerService {
    public List<Task> scheduleTasks(List<Task> tasks) {
        Map<String, List<String>> graph = new HashMap<>();
        Map<String, Integer> inDegree = new HashMap<>();

        // Build graph
        for (Task task : tasks) {
            graph.putIfAbsent(task.getId(), new ArrayList<>());
            inDegree.putIfAbsent(task.getId(), 0);

            for (String depId : task.getDependencies()) {
                graph.computeIfAbsent(depId, k -> new ArrayList<>()).add(task.getId());
                inDegree.merge(task.getId(), 1, Integer::sum);
            }
        }

        // Kahn's algorithm
        Queue<String> queue = new LinkedList<>();
        for (var entry : inDegree.entrySet()) {
            if (entry.getValue() == 0) queue.add(entry.getKey());
        }

        List<Task> result = new ArrayList<>();
        while (!queue.isEmpty()) {
            String id = queue.poll();
            tasks.stream()
                .filter(t -> t.getId().equals(id))
                .findFirst()
                .ifPresent(result::add);

            for (String neighbor : graph.getOrDefault(id, List.of())) {
                int updated = inDegree.merge(neighbor, -1, Integer::sum);
                if (updated == 0) queue.add(neighbor);
            }
        }

        if (result.size() != tasks.size()) {
            throw new IllegalStateException("Circular dependency detected");
        }
        return result;
    }
}
```

### 4.7 Merge Sort for Large File Sorting (External Sorting)

```java
public class ExternalSorter {
    private static final int CHUNK_SIZE = 100_000; // Lines per chunk

    public void sortLargeFile(Path inputPath, Path outputPath) throws IOException {
        List<Path> chunks = splitIntoChunks(inputPath);
        mergeChunks(chunks, outputPath);
        chunks.forEach(p -> p.toFile().delete());
    }

    private List<Path> splitIntoChunks(Path inputPath) throws IOException {
        List<Path> chunks = new ArrayList<>();
        List<String> buffer = new ArrayList<>();

        try (BufferedReader reader = Files.newBufferedReader(inputPath)) {
            String line;
            int chunkIndex = 0;
            while ((line = reader.readLine()) != null) {
                buffer.add(line);
                if (buffer.size() >= CHUNK_SIZE) {
                    chunks.add(writeSortedChunk(buffer, chunkIndex++));
                    buffer.clear();
                }
            }
            if (!buffer.isEmpty()) {
                chunks.add(writeSortedChunk(buffer, chunkIndex));
            }
        }
        return chunks;
    }

    private Path writeSortedChunk(List<String> lines, int index) throws IOException {
        lines.sort(Comparator.naturalOrder());
        Path chunkPath = Path.of("chunk_" + index + ".tmp");
        Files.write(chunkPath, lines);
        return chunkPath;
    }

    private void mergeChunks(List<Path> chunks, Path outputPath) throws IOException {
        PriorityQueue<BufferedEntry> heap = new PriorityQueue<>(
            Comparator.comparing(e -> e.line));

        List<BufferedReader> readers = new ArrayList<>();
        for (Path chunk : chunks) {
            BufferedReader reader = Files.newBufferedReader(chunk);
            readers.add(reader);
            String line = reader.readLine();
            if (line != null) {
                heap.add(new BufferedEntry(line, reader));
            }
        }

        try (BufferedWriter writer = Files.newBufferedWriter(outputPath)) {
            while (!heap.isEmpty()) {
                BufferedEntry entry = heap.poll();
                writer.write(entry.line);
                writer.newLine();

                String next = entry.reader.readLine();
                if (next != null) {
                    heap.add(new BufferedEntry(next, entry.reader));
                }
            }
        }

        for (BufferedReader reader : readers) {
            reader.close();
        }
    }

    private record BufferedEntry(String line, BufferedReader reader) {}
}
```

---

## 5. Real-World Scenarios (10)

### Scenario 1: High-Availability Cache with Expiration
**Problem:** A social media feed needs cached for 5 minutes but evicted early if memory pressure is high.

**Analysis:** Need an eviction policy plus TTL. Simple HashMap doesn't handle this.

**Solution:** `LinkedHashMap` with LRU eviction + scheduled cleanup for expired entries.

**Why it works:** LRU evicts least recently used under memory pressure. TTL ensures staleness within bounds.

**Alternative:** Redis with `EXPIRE` and `maxmemory-policy allkeys-lru`.

### Scenario 2: Real-Time Leaderboard
**Problem:** A gaming app needs top 100 scores from 10M players, updated in real-time.

**Analysis:** Sorting 10M records for every request is O(n log n) — too slow.

**Solution:** Use a `ConcurrentSkipListSet` or Redis Sorted Set. Insert = O(log n). Get top 100 = O(log n + 100).

**Why it works:** Balanced tree keeps elements sorted at all times. No sorting needed on read.

### Scenario 3: Geospatial Queries (Find Nearby Restaurants)
**Problem:** Find all restaurants within 5km of the user, from 100K+ locations.

**Analysis:** O(n) distance calculation per query is too slow.

**Solution:** R-Tree or geohashing. Divide the world into grid cells. Only check restaurants in nearby cells.

**Why it works:** Spatial indexing reduces search space from O(n) to O(log n) with quadtree/R-tree.

### Scenario 4: URL Shortener (Bit.ly clone)
**Problem:** Generate short unique keys for URLs. Handle 1000 writes/sec, 10,000 reads/sec.

**Analysis:** Need fast encode/decode, collision-free generation, fast lookup.

**Solution:** Base62 encoding (a-z, A-Z, 0-9) of auto-increment ID or hash. HashMap/Redis for O(1) lookup.

**Why it works:** Base62 gives short keys (62⁷ ≈ 3.5 trillion combos for 7 chars). Hash/Redis gives O(1) reads.

### Scenario 5: Web Crawler URL Deduplication
**Problem:** Web crawler visits billions of URLs. Need to check if a URL was already visited.

**Analysis:** Storing all URLs in a HashSet is memory-prohibitive (~200 bytes per URL → 200GB for 1B URLs).

**Solution:** Bloom filter. Fast O(k) check, configurable false positive rate. Acceptable trade-off (miss 0.1% of unique URLs).

**Why it works:** Bloom filter uses ~2 bytes per entry at 1% false positive rate. 1B URLs → 2GB vs 200GB.

### Scenario 6: Scheduling Batch Jobs
**Problem:** 500 batch jobs with dependencies. Need to determine execution order and detect circular deps.

**Analysis:** DAG + topological sort.

**Solution:** Kahn's algorithm. Track in-degree. Process zero-in-degree nodes first. Detect cycle if not all nodes processed.

### Scenario 7: Autocomplete Across 1M Products
**Problem:** Search bar suggests products as user types. Need < 100ms response.

**Analysis:** Prefix search on 1M products. SQL `LIKE 'prefix%'` uses index for prefix but not for mid-word. Slow at scale.

**Solution:** Trie for prefix search. HashMap of prefix → suggestions (precomputed). Redis for caching.

**Why it works:** Trie search is O(L) where L = prefix length, independent of dictionary size.

### Scenario 8: Detecting Fraudulent Transactions
**Problem:** Detect if a credit card is being used simultaneously in two far-apart cities.

**Analysis:** Need to track recent transactions per card and detect distance anomalies.

**Solution:** HashMap<CardId, LastTransaction> with timestamp + geohash. Check if new transaction is impossible (distance / time > speed limit).

### Scenario 9: E-commerce Product Recommendations
**Problem:** "Customers who bought X also bought Y" for 10M products, 100M users.

**Analysis:** Naive approach O(n²) — impossible.

**Solution:** Collaborative filtering via matrix factorization. Offline batch computation. Online retrieval via HashMap (product → similar products).

### Scenario 10: Real-Time Analytics Dashboard
**Problem:** Count page views per URL in the last 5 minutes, updated every second.

**Analysis:** Counting per second with sliding window. Storing all timestamps is memory-heavy.

**Solution:** Sliding window counter (not per-event). Divide window into smaller buckets (e.g., per second). Sum last 300 buckets. Memory: 300 counters per URL, not 300 events.

---

## 6. Performance Considerations

### Sorting Algorithm Comparison

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|------|---------|-------|-------|--------|
| Bubble sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Insertion sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Selection sort | O(n²) | O(n²) | O(n²) | O(1) | No |
| Merge sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quick sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Heap sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Tim sort | O(n) | O(n log n) | O(n log n) | O(n) | Yes |

**Java's Arrays.sort() uses:** Dual-Pivot QuickSort for primitives, TimSort (merge + insertion) for objects.

### Memory Optimization Tips
1. **Use primitives** — `int[]` vs `Integer[]`: 4 bytes vs 16 bytes (plus boxing overhead)
2. **Pre-size collections** — `new HashMap<>(expectedSize)` to avoid repeated resizing
3. **Use arrays for fixed-size data** — ArrayList overhead is ~20% more
4. **String interning** — `String.intern()` for repeated strings (use with caution)
5. **Flyweight pattern** — Share immutable flyweight objects across many contexts
6. **Lazy initialization** — Don't compute/cache until needed

### Common Performance Patterns
- **Read-heavy:** HashMap, array, read-optimized tree
- **Write-heavy:** ArrayList (end inserts), LinkedList (head inserts), log-structured merge trees
- **Memory-bound:** Bloom filter, bitset, succinct data structures
- **Disk-bound:** B-Tree, LSM-Tree, external sorting

---

## 7. Security Considerations

### Data Structure Attack Vectors

| Attack | Target | Description | Mitigation |
|--------|--------|-------------|------------|
| Hash collision DoS | HashMap | Craft keys that collide → O(n) lookup | Random hash seed (Java 8+) |
| Algorithmic complexity | PriorityQueue | Insert many elements to trigger O(n log n) | Rate limit inputs |
| Stack overflow | Recursion | Deep recursion → StackOverflowError | Use iterative approach |
| Integer overflow | Arrays | `n * 2` for resize → negative size → crash | Check before arithmetic |
| Timing attack | Hash comparison | Use hash of password for timing leak | Constant-time comparison `MessageDigest.isEqual()` |
| Out of memory | Unlimited data | Attacker sends infinite data to unlimited structure | Bound all collections |

---

## 8. Common Mistakes (20)

| # | Mistake | Why It Happens | Consequence | Correct Approach |
|---|---------|---------------|-------------|------------------|
| 1 | Using `==` for string comparison | C/C++ habit | Wrong comparison | `.equals()` for string content |
| 2 | Forgetting `equals()`/`hashCode()` | "It compiles" | HashMap entries not found | Override both correctly |
| 3 | Using `ArrayList` when lots of inserts/deletes in middle | Default choice | O(n) per operation | `LinkedList` for head/mid inserts |
| 4 | Concurrent modification exception | Iterate + modify same collection | Runtime exception | `Iterator.remove()`, `ConcurrentHashMap` |
| 5 | HashMap without generics | Legacy code, lazy | Casts, heap pollution | `Map<String, Integer>` not `Map` |
| 6 | Using synchronized on HashMap | "Thread safety" | Per-method locking is too coarse | `ConcurrentHashMap` |
| 7 | Not handling tree balancing | "BST is fine" | O(n) degradation in worst case | AVL/Red-Black Tree |
| 8 | Off-by-one errors | Index confusion | ArrayIndexOutOfBounds | Inclusive/exclusive boundary discipline |
| 9 | Stack overflow from recursion | Not calculating depth | Crash | Use iteration or tail recursion |
| 10 | Assuming O(1) is always fast | Big O ignores constants | HashMap collisions → still O(1) but 10x slower | Measure, don't guess |
| 11 | Ignoring amortized cost | "ArrayList add is O(1)" | Occasional O(n) resize pauses | Pre-size if you know capacity |
| 12 | Using TreeMap when HashMap works | "TreeMap keeps order" | O(log n) vs O(1) per operation | Profile before choosing |
| 13 | Forgetting Comparator consistency | Natural ordering changes | TreeMap breaks | Consistent comparator with equals |
| 14 | Not considering memory | Data fits in memory assumption | OutOfMemoryError | Estimate memory before choosing |
| 15 | Copying entire collection | "Just to be safe" | Wasted memory + time | Use views, unmodifiable wrappers |
| 16 | Wrong sort algorithm | "QuickSort is always best" | O(n²) on nearly sorted data | TimSort (Java default), hybrid sorts |
| 17 | Not using try-with-resources | Old habit | Resource leak | `try (Stream s = ...)` |
| 18 | Mutable objects in HashSet | Hash changes after add | Lost objects | Immutable keys or defensive copy |
| 19 | Overusing recursion | Recursive thinking | Stack overflow | Iterative for large depth |
| 20 | Ignoring data locality | "LinkedList and ArrayList are just lists" | 10x slower due to cache misses | Use arrays for CPU-bound, cache-aware code |

---

## 9. Senior Engineer Perspective

### How a Senior Engineer Thinks About DSA

**1. Not Just About Algorithms**
A senior engineer knows that in most production code, the bottleneck is I/O (database, network, disk), not CPU. Optimizing an O(n²) to O(n log n) is irrelevant if the database query takes 500ms. Profile first, optimize where it matters.

**2. Readability > Theoretical Performance**
```java
// Senior engineer choice: clear and good enough
Set<String> processedIds = new HashSet<>();

// vs. "optimal" but unmaintainable
BitSet processedIds = new BitSet();
// Requires mapping strings to indices — fragile
```

**3. Choosing the Right Data Structure**
| Question | Data Structure |
|----------|---------------|
| Need fast lookups by key? | HashMap |
| Need sorted iteration? | TreeMap |
| Need FIFO processing? | Queue (ArrayDeque) |
| Need LIFO? | Stack / ArrayDeque |
| Need unique elements? | HashSet |
| Need priority-based processing? | PriorityQueue |
| Need bidirectional map? | BiMap (Guava) |
| Need fast prefix search? | Trie |

**4. Production vs Interview DSA**
- **Interview:** Solve the problem optimal asymptotically, handle edge cases
- **Production:** The problem is usually "find this in a database" or "cache this result." The "algorithm" is rarely novel — it's about wiring the right data structure with the right library.

**5. The Cost of Complexity**
Using a Red-Black tree when a HashMap works adds:
- 5x more code
- Harder to debug (tree structure in debugger)
- Harder to reason about
- More test cases needed

**"Make it work, make it right, make it fast — in that order."**

---

## 10. Interview Questions

### Beginner Questions (10)

**Q1: What's the difference between an array and a linked list?**
**A:** Array: contiguous memory, O(1) access, fixed size, cache-friendly. Linked list: scattered nodes, O(n) access, dynamic size, extra memory per node.

**Q2: What is Big O notation?**
**A:** Big O describes the upper bound of time/space complexity as input size grows. It drops constants and non-dominant terms. O(n) means linear growth.

**Q3: How does a HashMap work?**
**A:** Uses a hash function on keys → determines bucket index. Collisions resolved with chaining (linked list → tree after threshold). Load factor triggers resize.

**Q4: What's the difference between Stack and Queue?**
**A:** Stack: LIFO (Last-In-First-Out). Queue: FIFO (First-In-First-Out).

**Q5: What is recursion?**
**A:** A function that calls itself. Must have a base case to terminate. Uses the call stack.

**Q6: What's the difference between BFS and DFS?**
**A:** BFS uses a queue, explores level by level → shortest path in unweighted graphs, more memory. DFS uses recursion/stack, explores deep first → topological sort, less memory, may not find shortest path.

**Q7: What is a binary search tree?**
**A:** Tree where left < parent < right. O(log n) operations when balanced. O(n) when skewed.

**Q8: What is a heap?**
**A:** Complete binary tree where parent is min (min-heap) or max (max-heap) of its children. Used for priority queues, heapsort.

**Q9: What's the difference between ArrayList and LinkedList in Java?**
**A:** ArrayList: array-backed, O(1) get, O(n) insert/delete mid. LinkedList: doubly-linked, O(n) get, O(1) insert/delete at ends.

**Q10: What is dynamic programming?**
**A:** Solving problems by breaking into overlapping subproblems, caching results (memoization/tabulation). Requires optimal substructure.

### Intermediate Questions (20)

**Q11: Implement a function to detect cycle in a linked list.**
**A:** Floyd's cycle detection (tortoise and hare). Two pointers: slow (1 step) and fast (2 steps). If they meet, cycle exists.

**Q12: What's the time complexity of HashMap.get() in the worst case?**
**A:** O(n) — when all keys hash to the same bucket (hash collision DoS attack).

**Q13: Explain how to reverse a linked list in O(n).**
**A:** Three pointers: prev, current, next. Iteratively reverse each node's next pointer.

**Q14: How would you implement a queue using two stacks?**
**A:** Enqueue → push to stack1. Dequeue → if stack2 empty, pop all from stack1 to stack2, then pop from stack2. Amortized O(1) per operation.

**Q15: What's the difference between merge sort and quick sort?**
**A:** Merge sort: O(n log n) guaranteed, O(n) space, stable. Quick sort: O(n log n) average, O(n²) worst, O(log n) space, not stable, faster in practice.

**Q16: Explain how to find the middle of a linked list in one pass.**
**A:** Two pointers: slow moves 1 step, fast moves 2 steps. When fast reaches end, slow is at middle.

**Q17: What is a trie and when would you use it?**
**A:** Prefix tree. Each node represents a character. Used for autocomplete, spell check, dictionary lookups. Search is O(L) where L = word length.

**Q18: How do you find the Kth largest element in an array?**
**A:** QuickSelect (O(n) average, O(n²) worst) or min-heap of size K (O(n log K)).

**Q19: Explain load factor in HashMap. What happens when it's exceeded?**
**A:** Load factor (default 0.75) = (entries / capacity). When exceeded, HashMap doubles in size and rehashes all entries. Higher load factor = more collisions. Lower = more memory.

**Q20: What is a balanced tree? When do you need one?**
**A:** A tree that maintains O(log n) height via rotations or restructuring. Needed when worst-case O(n) performance is unacceptable (e.g., real-time systems, database indexes).

**Q21: Implement a binary search on a sorted array.**
**A:**
```java
int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

**Q22: What's the difference between a HashMap and a TreeMap?**
**A:** HashMap: O(1) average, unsorted, allows null. TreeMap: O(log n), sorted (Red-Black tree), comparable keys required.

**Q23: How does Java's PriorityQueue work?**
**A:** Binary heap (min-heap by default) backed by an array. Elements ordered by natural ordering or Comparator. Insert/extract-min are O(log n).

**Q24: Explain the sliding window technique.**
**A:** Maintain a window (subarray/substring) of variable/fixed size. Slide the window across the data. Used for substring problems, rate limiting. O(n) time.

**Q25: What is a Bloom filter?**
**A:** Probabilistic data structure for set membership. Can have false positives (says "yes" when actually "no") but never false negatives. Uses k hash functions + bit array. Space-efficient for large datasets.

**Q26: How would you detect if a string has all unique characters without additional data structures?**
**A:** Use a bit vector (int/long bitset) if character set is known (e.g., ASCII 128). Or compare each character (O(n²)).

**Q27: Explain LRU cache eviction policy.**
**A:** Least Recently Used: when cache is full, evict the item that was accessed least recently. Implement using HashMap + doubly linked list.

**Q28: What is topological sort and when is it used?**
**A:** Linear ordering of DAG vertices where for every edge u→v, u comes before v. Used for task scheduling, dependency resolution, build systems.

**Q29: Difference between B-Tree and Binary Search Tree?**
**A:** B-Tree can have more than 2 children, is self-balancing, optimized for disk I/O (higher branching factor → fewer page reads). Database indexes use B+Tree.

**Q30: How would you design a thread-safe counter?**
**A:** `AtomicInteger` (CAS-based, lock-free). Or `synchronized` increment. Or `LongAdder` for high-contention scenarios.

### Senior-Level Questions (20)

**Q31: Design a connection pool. What data structures would you use?**
**A:** BlockingQueue (ArrayBlockingQueue) for idle connections. Semaphore for max connections. HashMap<Connection, Timestamp> for lease tracking. Object pool pattern.

**Q32: How would you implement a highly concurrent rate limiter handling 100K req/s?**
**A:** Token bucket (Leaky bucket variant) per client. Use `LongAdder` for counters, striped locks per bucket. Better: Redis with Lua scripting for atomic operations across instances.

**Q33: Design a distributed ID generator (like Snowflake).**
**A:** 64-bit ID: timestamp (41 bits) + datacenter ID (5) + machine ID (5) + sequence (12). Local generation, no coordination. ~10K IDs/sec per machine.

**Q34: You have 1 billion integers. How do you find the median with limited memory?**
**A:** 1) Count sort by byte ranges (bucket pass), 2) Identify which bucket contains the median, 3) Second pass on that bucket. O(n) time, O(1) memory.

**Q35: Design a time-series database for metrics. How do you store and query 10M data points?**
**A:** LSM-Tree for writes (append-only, sorted on flush). B-Tree for reads. Downsampling for old data. Retention policies. Use separate compaction process.

**Q36: How does ConcurrentHashMap achieve high concurrency?**
**A:** JDK 8+: CAS on initialization, `synchronized` on specific bucket for modifications, `volatile` reads, `TreeNode` for high-collision buckets. No global lock — per-bucket locking.

**Q37: Design a near-real-time deduplication system for 50M events/day.**
**A:** Bloom filter per time window (hourly). Redis with TTL for recent events. For strict dedup: use database unique constraint with retry.

**Q38: How would you implement an autocomplete for 10M products with < 50ms response?**
**A:** Trie pre-built from product names. HashMap from prefix → top 10 results (precomputed). Redis caching for hot prefixes. Fallback to database for cold prefixes.

**Q39: How does Java's Arrays.sort() work and why does it use different algorithms?**
**A:** Dual-Pivot QuickSort for primitives (fast, in-place). TimSort (merge + insertion) for objects (stable, O(n) for nearly sorted). Different trade-offs.

**Q40: Design a leaderboard that updates in real-time for 10M players.**
**A:** Redis Sorted Set (ZADD O(log n), ZREVRANGE O(log n + k)). For write scaling: batch updates, merge periodically. For read scaling: cache top 100, update every 5s.

**Q41: How would you implement an LRU cache that's both thread-safe and high-performance?**
**A:** `ConcurrentHashMap` + `ConcurrentLinkedDeque`. CAS operations. Or use `LinkedHashMap` with `synchronizedMap` for lower contention scenarios. Guava Cache / Caffeine for production.

**Q42: Design a nearest-neighbor search for 1M locations.**
**A:** KD-Tree (balanced binary tree partitioning by dimension). R-Tree (bounding boxes). For approximate answers: locality-sensitive hashing. Geohashing for simple cases.

**Q43: You have an API that takes 500ms average, 99th percentile 5s. How do you debug this?**
**A:** Instrument each component. Plot latency distributions. Check for GC pauses. Look for queue buildup. The distribution suggests different execution paths — not just random noise.

**Q44: Design a message queue supporting at-least-once delivery.**
**A:** Queue (in-memory or persisted) + acknowledgment mechanism + retry with backoff + deduplication on consumer side. `LinkedBlockingQueue` for in-process, Kafka/Kinesis for distributed.

**Q45: How does Java's String.hashCode() work and what are its weaknesses?**
**A:** `s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]`. Multiplier 31 (prime, odd → better distribution). Weakness: predictable, collision DoS possible. Fixed in Java 8+ with random hash seed.

**Q46: Design a system to detect duplicate images.**
**A:** Perceptual hashing (pHash): resize → grayscale → DCT → hash. Store in HashMap<Hash, ImageId>. For scale: use locality-sensitive hashing (approximate nearest neighbor search).

**Q47: How would you implement a distributed counter with strong consistency?**
**A:** Redis with INCR (atomic, single-threaded). For stronger consistency: ZooKeeper (sequential znodes) or database with row-level locking and retry.

**Q48: Design a Dijkstra implementation for 1M nodes. Optimize it.**
**A:** Adjacency list for graph (sparse). PriorityQueue for frontier. Use indexed priority queue (Fibonacci heap could be faster but complex) for O((V+E) log V). Bidirectional Dijkstra for known start/end.

**Q49: How does garbage collection interact with data structure choice?**
**A:** Mutable objects survive longer → promoted to Old Gen → major GC overhead. Immutable objects create more garbage but die young → cheap minor GC. Object pooling reduces allocation but increases complexity.

**Q50: You have a 100TB dataset. What data structures and algorithms would you NOT use and why?**
**A:** Anything requiring random access on disk (BST, hash table) — too many seeks. Anything O(n²) — impossible. Anything requiring full data in memory — impossible. Use: B-Tree (sequential reads), external sort, map-reduce, streaming algorithms.

### Architect-Level Questions (10)

**Q51: Design a distributed key-value store with 99.999% availability and <10ms p99 latency.**
**A:** Consistent hashing for sharding, replication factor 3, quorum-based reads/writes (R=2, W=2), hinted handoff for failures, anti-entropy via Merkle trees, gossip protocol for membership.

**Q52: How would you choose between Cassandra, MongoDB, and PostgreSQL for a new system?**
**A:** Cassandra: write-heavy, time-series, multi-DC, no joins. MongoDB: flexible schema, nested documents, moderate writes. PostgreSQL: strong consistency, complex queries, ACID transactions, joins.

**Q53: Design a real-time fraud detection system processing 10K events/sec.**
**A:** Sliding window per user (HashMap<userId, Deque<Event>>). Stream processing (Kafka Streams/Flink) for stateful operations. ML model for scoring. Bloom filter for known-fraud patterns.

**Q54: How would you design Google Search's inverted index?**
**A:** HashMap<Term, List<DocId>> sorted by DocId. Postings lists compressed with variable-byte encoding. Skip pointers for faster OR operations. Index sharded across machines (doc ID range sharding).

**Q55: Design a recommendation system for 100M users and 10M items.**
**A:** Collaborative filtering offline (Spark on Hadoop → matrix factorization). Store user→factors vectors in Key-Value store. At runtime, compute dot product of user vector with candidate item vectors. ANN (HNSW) for finding similar vectors.

**Q56: How would you design a distributed queue with exactly-once semantics?**
**A:** Partitioned queue with unique message IDs. Deduplication by consumer (bloom filter + DB of processed IDs). Two-phase commit or Kafka transactional API. Idempotent consumers.

**Q57: Design a system that indexes 1B documents and supports full-text search from 100K+ queries/sec.**
**A:** Elasticsearch on sharded inverted indexes. Each shard = separate Lucene index. Query routing to relevant shards. Result merging at coordinator. Caching for hot queries. ASYNC replication.

**Q58: How would you design a globally distributed session store with < 5ms latency?**
**A:** Redis Cluster with read replicas in each region. Write to local primary → async replication. Sticky sessions (same user → same region). Fallback to local database cache.

**Q59: Design a time-bounded cache that supports eventual consistency across 5 data centers.**
**A:** Write-through cache (write to DB + local cache). Cross-DC replication via CDC (change data capture). Cache invalidation via pub/sub (Redis PubSub / Kafka). TTL-based fallback to prevent stale reads.

**Q60: How do you decide between a relational index (B+Tree) and a search index (inverted index) for a query feature?**
**A:** B+Tree for: range queries, point lookups, sorted results, joins. Inverted index for: full-text search, relevance ranking, fuzzy matching, faceted search. Hybrid: use both and route queries appropriately.

---

## 11. Scenario-Based Interview Questions (20)

### Scenario 1: Slow Product Search
**Problem:** Product search takes 5 seconds for 500K products.

**Analysis:** No index on search columns. Full table scan every query.

**Solution:** Add B-Tree index on search columns. Better: Elasticsearch for full-text search. Even better: precompute search results for common queries.

### Scenario 2: API Timeout Under Load
**Problem:** An API endpoint takes 30 seconds and times out when load increases.

**Analysis:** The endpoint loads all data into memory, sorts, then returns. O(n log n) sort on millions of records.

**Solution:** Paginate at the database level. Use LIMIT/OFFSET. Add proper indexes. If sorting by a calculated field, precompute it.

### Scenario 3: ConcurrentHashMap vs Hashtable
**Problem:** Developer uses `Hashtable` for a cache accessed by 50 threads. Throughput is terrible.

**Analysis:** `Hashtable` synchronizes every method — single-threaded access to the entire map. Contention is 100%.

**Solution:** `ConcurrentHashMap` — allows concurrent reads and segment-level locking for writes.

### Scenario 4: OOM in Production
**Problem:** Service crashes with OutOfMemoryError after running for 3 days.

**Analysis:** In-memory cache has no eviction policy. Growth is unbounded.

**Solution:** Add maximum size (LinkedHashMap with LRU eviction). Use Guava Cache/Caffeine with size-based eviction.

### Scenario 5: N+1 Query Problem
**Problem:** Loading 100 orders triggers 101 SQL queries.

**Analysis:** Default lazy loading. Each `order.getItems()` triggers a separate query.

**Solution:** `JOIN FETCH` or `@EntityGraph` to load associations in one query. DTO projections for read-only scenarios.

### Scenario 6: Wrong Index for Query Pattern
**Problem:** A query with `WHERE status = ? AND created_at < ?` is slow despite an index on `created_at`.

**Analysis:** The index on `created_at` is used but filters many rows by status after. Index scan is still large.

**Solution:** Create a composite index on `(status, created_at)`. The database can seek by status first, then scan by date within that status.

### Scenario 7: Recursive Comment Thread
**Problem:** Displaying nested comments recursively causes StackOverflowError for deep threads.

**Analysis:** Recursion depth > thread stack size. Default ~1000 frames.

**Solution:** Use iterative BFS/DFS with explicit stack. Or limit depth to 10 levels. Use a flat model with nesting handled by the frontend.

### Scenario 8: Rate Limiter Overhead
**Problem:** Custom rate limiter using `synchronized` blocks causing 50% CPU overhead.

**Analysis:** Contention on the rate limiter lock. Every request contends for the same lock.

**Solution:** Sliding window log per client. Striped locks. Better: Redis with Lua or token bucket with atomic counters.

### Scenario 9: Bloom Filter False Negatives
**Problem:** Dedup system using a Bloom filter starts producing false negatives.

**Analysis:** Initial Bloom filter size was too small. As more elements are added, false positive rate increases, and the filter can't be resized.

**Solution:** Use a scalable Bloom filter (adds new filters as needed). Or use Redis Bloom with auto-scaling.

### Scenario 10: Sort Order Inconsistency
**Problem:** Results appear in different order between paginated requests.

**Analysis:** `ORDER BY created_at` without secondary sort — ties can be sorted arbitrarily by the database.

**Solution:** Add a secondary sort column: `ORDER BY created_at DESC, id ASC`.

### Scenario 11: Index Not Used After ANALYZE
**Problem:** `EXPLAIN` shows full table scan despite having an index on the column.

**Analysis:** The query pattern uses functions or type casting on the indexed column. Example: `WHERE DATE(created_at) = '2024-01-01'` (function prevents index usage).

**Solution:** Use range query: `WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02'`.

### Scenario 12: Thread Pool Exhaustion
**Problem:** Application stops responding. Thread dump shows all threads in a blocking queue.

**Analysis:** Thread pool is full. Tasks are waiting for downstream services that are slow.

**Solution:** Add timeout to all network calls. Use `newCachedThreadPool` for short tasks. Implement circuit breaker pattern.

### Scenario 13: HashMap Resize Storm
**Problem:** Application pauses for 2 seconds every few minutes.

**Analysis:** GC logs show major GC. Heap dump shows large HashMap resizing (copying all entries to new buckets).

**Solution:** Pre-size HashMap with expected capacity. `new HashMap<>(expectedSize / 0.75f + 1)`.

### Scenario 14: Slow UUID Primary Key
**Problem:** Writes to a table with UUID primary key are slow.

**Analysis:** Random UUID → B+Tree index fragmentation → page splits → poor cache utilization.

**Solution:** Use UUID v7 (time-ordered) or ULID for sequential UUIDs. Or use auto-increment + externally exposed UUID.

### Scenario 15: Binary Search on Unsorted Array
**Problem:** Binary search returns -1 even though the target exists.

**Analysis:** Binary search requires a sorted array. The array is unsorted.

**Solution:** Sort the array first, or use linear search, or maintain sorted order during insert.

### Scenario 16: Recursive CTE Performance
**Problem:** Recursive CTE for tree traversal takes 30 seconds for a depth of 5.

**Analysis:** No index on `parent_id`. Recursive CTE does sequential scan per level.

**Solution:** Index on `parent_id`. Consider using nested sets or closure tables for tree storage.

### Scenario 17: PriorityQueue with Variable Priority
**Problem:** Task priority changes after insertion, but PriorityQueue doesn't reorder.

**Analysis:** PriorityQueue doesn't support dynamic priority changes. The heap property is violated.

**Solution:** Use a custom heap that supports `decreaseKey`. Or remove and re-insert the task. For small sizes, scan and rebuild.

### Scenario 18: Linked List Traversal in Cache-Miss Mode
**Problem:** Iterating 100K linked list nodes is 10x slower than iterating an array of same size.

**Analysis:** Nodes are allocated at different times/places → scattered in memory → cache misses on every access.

**Solution:** Use array-backed structures (ArrayList, ArrayDeque) for sequential access patterns.

### Scenario 19: Too Many Database Connections
**Problem:** Application runs out of database connections under moderate load.

**Analysis:** Each request opens a new connection without pooling.

**Solution:** Use connection pool (HikariCP — default Spring Boot). Set pool size based on: `(max_threads * (1 + block_factor))`. Max connections for typical app: 10-20 per instance.

### Scenario 20: Inner Join vs Subquery Performance
**Problem:** Subquery takes 10 seconds while equivalent JOIN takes 50ms.

**Analysis:** Older MySQL versions optimize subqueries poorly (materialized subquery without index). JOIN is better for correlated subqueries.

**Solution:** Rewrite subquery as JOIN. Use EXPLAIN to verify query plan. Modern databases (PostgreSQL, MySQL 8+) handle both well.

---

## 12. Debugging & Troubleshooting

### Issue 1: HashMap Infinite Loop in Java 7
- **Symptoms:** CPU at 100%, thread dump shows thread stuck in HashMap.get()
- **Root cause:** Concurrent resize without synchronization → circular linked list in bucket
- **Investigation:** Thread dump → `HashMap.transfer()` method
- **Resolution:** Use `ConcurrentHashMap` or synchronized access

### Issue 2: StackOverflowError in Recursive Functions
- **Symptoms:** Application crashes with StackOverflowError
- **Root cause:** Recursive call depth > thread stack size (~500-1000 frames default)
- **Investigation:** Check stack trace for recursion path. Increase -Xss if needed.
- **Resolution:** Convert to iteration. Or increase stack size: `-Xss2m`

### Issue 3: OutOfMemoryError: Java Heap Space
- **Symptoms:** Application crashes with OOM. GC logs show heap exhaustion.
- **Root cause:** Unbounded data structure growth (cache, queue, session store)
- **Investigation:** Heap dump → analyze biggest objects. Look for HashMap/ArrayList with millions of entries.
- **Resolution:** Add max size to collections. Use weak references. Implement eviction.

### Issue 4: ConcurrentModificationException
- **Symptoms:** Exception during iteration over a collection while another thread modifies it
- **Root cause:** Fail-fast iterator detects structural modification
- **Investigation:** Check which threads are reading and writing. Stack trace shows the iteration point.
- **Resolution:** Use `ConcurrentHashMap`, `CopyOnWriteArrayList`, or synchronize iteration.

### Issue 5: Slow HashMap.get() Despite O(1)
- **Symptoms:** HashMap.get() takes 100ms
- **Root cause:** Poor hash function → all keys in one bucket → O(n) scan
- **Investigation:** Check key hashCode() distribution. Check for hash collision attack.
- **Resolution:** Improve hashCode(). Use random hash seed (Java 8+). Use TreeMap as fallback.

### Issue 6: Index Not Used Despite Having Index
- **Symptoms:** Full table scan on indexed column
- **Root cause:** Query uses function on indexed column: `WHERE UPPER(name) = 'JOHN'`
- **Investigation:** `EXPLAIN ANALYZE` shows seq scan
- **Resolution:** Create functional index: `CREATE INDEX ON users (UPPER(name))`. Or use `= 'John'` directly.

### Issue 7: PriorityQueue Not Returning Sorted Elements
- **Symptoms:** Iterating PriorityQueue returns unsorted elements
- **Root cause:** PriorityQueue guarantees min on poll(), not on iteration. Iteration order is heap array order.
- **Investigation:** Check how elements are consumed. If iterating, order is not guaranteed.
- **Resolution:** Use poll() in a loop. Or use TreeSet for sorted iteration.

### Issue 8: Recursive Query Hits Database Timeout
- **Symptoms:** CTE with `WITH RECURSIVE` times out after 30s
- **Root cause:** No cycle detection → infinite recursion or very deep tree
- **Investigation:** Check depth of recursion. Check for cyclic references.
- **Resolution:** Add cycle detection: `CYCLE id SET is_cycle USING path`. Limit depth: `WHERE depth < 10`.

### Issue 9: Memory Leak From HashMap With Mutable Keys
- **Symptoms:** Memory grows over time, objects not garbage collected
- **Root cause:** Mutable objects used as HashMap keys. Object mutates → hash changes but object stays in old bucket. Can never be found or removed.
- **Investigation:** Heap dump → find objects in HashMap with unnatural hashCode
- **Resolution:** Use immutable keys. Or remove/re-insert when key changes.

### Issue 10: ThreadPoolExecutor Rejection
- **Symptoms:** `RejectedExecutionException` under load
- **Root cause:** Thread pool queue is full. Pool max threads reached. Work queue full.
- **Investigation:** Check thread pool settings. Monitor queue size.
- **Resolution:** Increase pool size. Use unbounded queue with caution. Implement rejection handler (CallerRunsPolicy). Add backpressure.

---

## 13. Comparison Section

### Data Structure Comparison

| Structure | Access | Search | Insert | Delete | Memory | Use Case |
|-----------|--------|--------|--------|--------|--------|----------|
| Array | O(1) | O(n) | O(n) | O(n) | Low | Fixed-size, random access |
| ArrayList | O(1) | O(n) | O(n)* | O(n) | Low+ | Dynamic array |
| LinkedList | O(n) | O(n) | O(1) | O(1) | Medium | Queue/Deque |
| HashMap | O(1)* | O(1)* | O(1)* | O(1)* | Medium | Key-value cache |
| TreeMap | O(log n) | O(log n) | O(log n) | O(log n) | Medium | Sorted KV |
| HashSet | O(1)* | O(1)* | O(1)* | O(1)* | Medium | Uniqueness |
| PriorityQueue | O(1) peek | O(n) | O(log n) | O(log n) | Medium | Priority-based |
| ArrayDeque | O(1) | O(n) | O(1) | O(1) | Low | Stack/Queue |

\* Amortized average case

### Which Data Structure for the Job?

| Need | Data Structure |
|------|---------------|
| Fast lookups by key | HashMap / ConcurrentHashMap |
| Sorted unique elements | TreeSet |
| Sorted key-value | TreeMap |
| Queue (FIFO) | ArrayDeque |
| Stack (LIFO) | ArrayDeque |
| Double-ended queue | ArrayDeque |
| Priority queue | PriorityQueue |
| Concurrent cache | Caffeine / Guava Cache |
| Thread-safe map with high concurrency | ConcurrentHashMap |
| Copy-on-read, write-rarely list | CopyOnWriteArrayList |
| Unique elements, insertion order | LinkedHashSet |
| Insertion-ordered key-value | LinkedHashMap |
| LRU cache | LinkedHashMap (access-order) |

---

## 14. Revision Notes

### Big O Quick Reference
```
O(1)       Constant      HashMap get, array access
O(log n)   Logarithmic   Binary search, balanced tree
O(n)       Linear        Iterate array, linked list search
O(n log n) Linearithmic  Merge sort, heap sort, quick sort avg
O(n²)      Quadratic     Bubble sort, nested loops
O(2ⁿ)      Exponential   Fibonacci naive recursion
O(n!)      Factorial     Traveling salesman brute force
```

### Algorithm Design Techniques
- **Divide & Conquer:** Merge sort, Quick sort, Binary search
- **Greedy:** Dijkstra, Prim's, Huffman, coin change (canonical)
- **DP:** Fibonacci, Knapsack, LCS, Edit distance, subset sum
- **Backtracking:** N-Queens, Sudoku, permutations
- **Sliding window:** Subarray sum, longest substring
- **Two pointers:** Pair sum, palindrome, linked list cycle
- **BFS/DFS:** Graph traversal, shortest path, connected components

### Must-Know Algorithms
1. Binary search
2. Merge sort / Quick sort
3. BFS / DFS
4. Dijkstra's shortest path
5. Floyd's cycle detection
6. Topological sort (Kahn's)
7. LRU cache implementation
8. Trie operations
9. Union-Find (Disjoint Set)
10. KMP string matching

### Interview Checkpoints
1. Always clarify input size, constraints, edge cases
2. Start with brute force → optimize
3. Think out loud: "What if I use a HashMap here?"
4. Test with: empty input, single element, duplicates, negative values
5. Space-time trade-off: "I can use more memory to make it faster"
6. Handle overflow: `int mid = left + (right - left) / 2` not `(left + right) / 2`

---

## 15. Cheat Sheet

```
═══ DSA QUICK REFERENCE ═══════════════════════════════════════

┌─ BIG O COMPLEXITIES ────────────────────────────────────────┐
│        Access  Search  Insert  Delete                        │
│ Array   O(1)    O(n)    O(n)    O(n)                        │
│ Stack   O(n)    O(n)    O(1)    O(1)                        │
│ Queue   O(n)    O(n)    O(1)    O(1)                        │
│ LL      O(n)    O(n)    O(1)    O(1)                        │
│ BST     O(l)*   O(l)*   O(l)*   O(l)*    *balanced          │
│ HashMap O(1)†   O(1)†   O(1)†   O(1)†    †avg               │
│ Heap    O(1)    O(n)    O(log)  O(log)                      │
└─────────────────────────────────────────────────────────────┘

┌─ SORTING ───────────────────────────────────────────────────┐
│ Algorithm   Best    Avg     Worst   Space   Stable           │
│ Bubble      O(n)    O(n²)   O(n²)   O(1)    Yes             │
│ Insertion   O(n)    O(n²)   O(n²)   O(1)    Yes             │
│ Selection   O(n²)   O(n²)   O(n²)   O(1)    No              │
│ Merge       O(nl)   O(nl)   O(nl)   O(n)    Yes             │
│ Quick       O(nl)   O(nl)   O(n²)   O(l)    No              │
│ Heap        O(nl)   O(nl)   O(nl)   O(1)    No              │
└─────────────────────────────────────────────────────────────┘

┌─ GRAPH ─────────────────────────────────────────────────────┐
│ BFS: Queue, short path (unweighted), O(V+E)                  │
│ DFS: Stack/Recursion, topo sort, cycle detect, O(V+E)       │
│ Dijkstra: PQ, +weights, O((V+E) log V)                      │
│ Bellman-Ford: -weights, O(VE)                                │
│ Floyd-Warshall: all-pairs, O(V³)                             │
│ Prim/Kruskal: MST, O(E log V)                                │
└─────────────────────────────────────────────────────────────┘

┌─ COMMON TECHNIQUES ─────────────────────────────────────────┐
│ Two pointers: pair sum, palindrome, cycle detection          │
│ Sliding window: subarray sum, substring problems             │
│ Prefix sum: range sum queries                                │
│ Hashing: dedup, frequency, cache                             │
│ Heap: top K, median, merge K sorted                          │
│ DP: overlapping subs + memoization → tabulation              │
│ Backtracking: generate all, prune early                      │
└─────────────────────────────────────────────────────────────┘

┌─ JAVA COLLECTIONS ──────────────────────────────────────────┐
│ ArrayList    → dynamic array                                │
│ LinkedList   → doubly linked list                           │
│ HashMap      → hash table (chaining + treeify)              │
│ TreeMap      → Red-Black tree                               │
│ LinkedHashMap→ hash + linked list (insertion/access order)  │
│ HashSet      → HashMap<E, PRESENT>                          │
│ PriorityQueue→ binary heap                                  │
│ ArrayDeque   → resizable array (Stack + Queue)              │
│ ConcurrentHashMap→ segmented locks (JDK 8 CAS + sync)      │
│ CopyOnWriteArrayList→ snapshot iterator, write copy         │
└─────────────────────────────────────────────────────────────┘
```

---

## 16. Knowledge Validation

### 20 Multiple Choice Questions

**Q1: What is the time complexity of binary search on a sorted array?**
- A) O(1)
- B) **O(log n)**
- C) O(n)
- D) O(n log n)

**Q2: Which data structure provides O(1) average-time insertion, deletion, and access?**
- A) TreeMap
- B) **HashMap**
- C) ArrayList
- D) PriorityQueue

**Q3: What data structure is used to implement a priority queue?**
- A) **Heap**
- B) Queue
- C) Stack
- D) HashMap

**Q4: Which sorting algorithm is O(n log n) in the worst case and stable?**
- A) Quick sort
- B) **Merge sort**
- C) Heap sort
- D) Insertion sort

**Q5: What is the worst-case time complexity of HashMap.get()?**
- A) O(1)
- B) O(log n)
- C) **O(n)**
- D) O(n log n)

**Q6: Breadth-First Search uses which data structure?**
- A) Stack
- B) **Queue**
- C) Heap
- D) HashMap

**Q7: Depth-First Search uses which data structure?**
- A) **Stack** (or recursion via call stack)
- B) Queue
- C) Heap
- D) HashMap

**Q8: What is the time complexity of inserting into a heap?**
- A) O(1)
- B) **O(log n)**
- C) O(n)
- D) O(n log n)

**Q9: Which data structure is best for an LRU cache?**
- A) ArrayList
- B) **LinkedHashMap** (access-order) + manual eviction
- C) TreeMap
- D) PriorityQueue

**Q10: What algorithm finds the shortest path in a graph with non-negative weights?**
- A) BFS
- B) **Dijkstra's**
- C) Bellman-Ford
- D) Floyd-Warshall

**Q11: What data structure would you use to detect a cycle in a linked list?**
- A) HashMap
- B) **Two pointers (tortoise and hare)**
- C) Stack
- D) Binary search

**Q12: Which tree property does an AVL tree guarantee?**
- A) **Height difference ≤ 1 between subtrees**
- B) O(n) search in worst case
- C) All leaves at the same depth
- D) No more than 2 children

**Q13: What is the space complexity of storing a graph using an adjacency matrix?**
- A) O(V + E)
- B) **O(V²)**
- C) O(E)
- D) O(V)

**Q14: What algorithm is used to detect a cycle in a directed graph?**
- A) BFS
- B) **DFS with recursion stack tracking**
- C) Dijkstra's
- D) Kruskal's

**Q15: What is the time complexity of building a heap from an array?**
- A) O(log n)
- B) O(n log n)
- C) **O(n)**
- D) O(n²)

**Q16: Which data structure is best for a concurrent, high-throughput key-value store?**
- A) Hashtable
- B) **ConcurrentHashMap**
- C) HashMap
- D) TreeMap

**Q17: What is the time complexity of finding the median of a stream using two heaps?**
- A) **O(log n) per insertion**
- B) O(1) per insertion
- C) O(n) per insertion
- D) O(n log n)

**Q18: What is a Bloom filter?**
- A) **A probabilistic data structure for set membership with possible false positives**
- B) A type of hash table with perfect hashing
- C) A data structure for sorting
- D) A balanced binary tree

**Q19: Which traversal visits nodes in sorted order for a BST?**
- A) Pre-order
- B) **In-order**
- C) Post-order
- D) Level-order

**Q20: What is the purpose of the load factor in HashMap?**
- A) To determine when to shrink the table
- B) **To determine when to resize the table (rehash)**
- C) To balance the BST
- D) To sort the entries

### 10 Coding Questions

**Q1:** Implement a function to check if a string is a palindrome.

```java
public boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) return false;
        left++;
        right--;
    }
    return true;
}
```

**Q2:** Reverse a linked list.

```java
public ListNode reverseList(ListNode head) {
    ListNode prev = null;
    ListNode current = head;
    while (current != null) {
        ListNode next = current.next;
        current.next = prev;
        prev = current;
        current = next;
    }
    return prev;
}
```

**Q3:** Find the first non-repeating character in a string.

```java
public char firstNonRepeating(String s) {
    Map<Character, Integer> counts = new LinkedHashMap<>();
    for (char c : s.toCharArray()) counts.merge(c, 1, Integer::sum);
    for (var entry : counts.entrySet()) {
        if (entry.getValue() == 1) return entry.getKey();
    }
    return '_'; // none found
}
```

**Q4:** Merge two sorted arrays.

```java
public int[] merge(int[] a, int[] b) {
    int[] result = new int[a.length + b.length];
    int i = 0, j = 0, k = 0;
    while (i < a.length && j < b.length) {
        result[k++] = a[i] <= b[j] ? a[i++] : b[j++];
    }
    while (i < a.length) result[k++] = a[i++];
    while (j < b.length) result[k++] = b[j++];
    return result;
}
```

**Q5:** Implement a binary search.

```java
public int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

**Q6:** Check if two strings are anagrams.

```java
public boolean areAnagrams(String a, String b) {
    if (a.length() != b.length()) return false;
    int[] counts = new int[26];
    for (char c : a.toCharArray()) counts[c - 'a']++;
    for (char c : b.toCharArray()) {
        if (--counts[c - 'a'] < 0) return false;
    }
    return true;
}
```

**Q7:** Find the maximum subarray sum (Kadane's algorithm).

```java
public int maxSubArraySum(int[] nums) {
    int maxSoFar = nums[0];
    int maxEndingHere = nums[0];
    for (int i = 1; i < nums.length; i++) {
        maxEndingHere = Math.max(nums[i], maxEndingHere + nums[i]);
        maxSoFar = Math.max(maxSoFar, maxEndingHere);
    }
    return maxSoFar;
}
```

**Q8:** Level-order traversal of a binary tree (BFS).

```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.add(root);
    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null) queue.add(node.left);
            if (node.right != null) queue.add(node.right);
        }
        result.add(level);
    }
    return result;
}
```

**Q9:** Determine if a linked list has a cycle and find its start.

```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) { // Cycle detected
            slow = head;
            while (slow != fast) {
                slow = slow.next;
                fast = fast.next;
            }
            return slow; // Start of cycle
        }
    }
    return null; // No cycle
}
```

**Q10:** Coin change problem (minimum coins) — DP.

```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);
    dp[0] = 0;
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

### 10 Scenario Questions

**Scenario 1:** Your API takes 2 seconds to return top 10 products by sales. The `orders` table has 10M rows. How do you optimize?

**Answer:** Create a materialized view or summary table: `product_sales_summary(product_id, total_sales)`. Update via batch job every 5 minutes. Query: `SELECT * FROM product_sales_summary ORDER BY total_sales DESC LIMIT 10`. Add index on `total_sales`.

**Scenario 2:** A HashMap with 10K entries is taking 500ms per get() call. What's wrong?

**Answer:** Likely a terrible hashCode() causing all entries in one bucket. Check the key class. If using `String` keys, verify that they're not crafted for collision attacks. Solution: improve hashCode(). With JDK 8+, treeify after 8 collisions → O(log n) for that bucket.

**Scenario 3:** You need to process 10K concurrent WebSocket connections. Each connection maintains state. What data structures?

**Answer:** `ConcurrentHashMap<ConnectionId, SessionState>` for O(1) lookups. For broadcasting: `CopyOnWriteArrayList<Session>` for safe iteration. For per-connection message queues: `LinkedBlockingQueue<Message>`.

**Scenario 4:** Design a cache for a busy REST API. Requirements: < 5ms response, 50K req/s, 1M entries max.

**Answer:** Caffeine cache (Java) with: maximum size = 1M, expireAfterWrite = 5 min, recordStats(). Uses W-TinyLFU eviction — better than LRU for skewed access patterns.

**Scenario 5:** You have two sorted arrays of 100K integers. Find the median of the combined array in O(log(min(n,m))).

**Answer:** Binary search on the smaller array. Partition both arrays such that left side max ≤ right side min. Adjust partition based on comparison. O(log(min(n,m))).

**Scenario 6:** Find all duplicate records in a 10GB CSV file with 200M rows and 50 columns.

**Answer:** External sorting: split into chunks, sort each, merge and detect duplicates. Or use a Bloom filter as first pass (fast, approximate), then exact comparison for candidates.

**Scenario 7:** Design a notification system where users get pushed notifications in real-time. 10M users, 100K events/sec.

**Answer:** Event → Kafka → stream processor. Per-user notification queue in Redis (sorted by time). Fan-out via topic. Push via WebSocket/SSE. Use `ConcurrentHashMap<UserId, WebSocketSession>` for active connections.

**Scenario 8:** Your search autocomplete shows results < 10ms for most queries but 2s for rare queries.

**Answer:** Trie + precomputed top 10 per prefix. Hot prefixes cached in Redis. Cold prefixes compute on demand (trie traversal) and cache for next time. Background job to precompute top queries from logs.

**Scenario 9:** A function processes social media feeds using recursion. It StackOverflows for users with 10K+ friends.

**Answer:** Recursion depth = friend count. Convert to iterative BFS using explicit Queue. Or iterative DFS with Stack.

**Scenario 10:** Design a rate limiter for a free-tier API: 10 req/sec per API key. Must handle 500K API keys, 50K req/sec total.

**Answer:** Sliding window counter per key. Store counters in Redis sorted set or hash. Use Lua scripting for atomic operations. Memory estimate: 500K keys × ~100 bytes = 50MB — fits in Redis comfortably. Batch key cleanup every hour.
