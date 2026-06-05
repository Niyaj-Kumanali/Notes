# Data Structures & Algorithms

---

## Overview

- **Definition:** Data structures organize and store data for efficient access. Algorithms are step-by-step procedures for solving problems. Together they form the foundation of efficient software.
- **Why It Exists:** Without DSA, software is slow (searching 1M records takes minutes), resource-heavy (memory usage scales linearly), and unscalable (what works for 100 users fails for 100K).
- **Key Concepts:** **Time Complexity** (how runtime grows — O(1), O(log n), O(n), O(n log n), O(n²)), **Space Complexity** (how memory grows), **Amortized Analysis** (average cost over a sequence), **Trade-offs** (time vs space, speed vs readability).

---

## Core Concepts

### Asymptotic Analysis & Big O

Big O describes the upper bound of runtime/memory growth as input size approaches infinity.

| Notation | Name | Example |
|----------|------|---------|
| O(1) | Constant | Array access, HashMap lookup |
| O(log n) | Logarithmic | Binary search, balanced BST |
| O(n) | Linear | Single loop, linked list traversal |
| O(n log n) | Linearithmic | Merge sort, heap sort |
| O(n²) | Quadratic | Nested loops over same array |
| O(2ⁿ) | Exponential | Recursive subsets, brute-force |

```cpp
// O(1) — array access
int get(int[] arr, int i) { return arr[i]; }

// O(n) — linear search
bool find(int[] arr, int target) {
    for (int n : arr) if (n == target) return true;
    return false;
}

// O(n²) — bubble sort
void sort(int[] arr) {
    for (int i = 0; i < arr.length; i++)
        for (int j = 0; j < arr.length - 1; j++)
            if (arr[j] > arr[j + 1]) swap(arr, j, j + 1);
}
```

### Linear Data Structures

**Arrays:** Contiguous memory, O(1) random access, O(n) insert/delete.

**Dynamic Arrays (ArrayList/Vec):** Amortized O(1) append, doubles capacity on overflow.

**Strings:** Immutable char arrays. `+=` in a loop creates O(n²) copies — use `StringBuilder`.

**Linked Lists:** Nodes with value + next pointer. O(1) prepend, O(n) access.

**Stacks & Queues:** LIFO (array/list) and FIFO (deque/linked list). Used for DFS/BFS.

### Hash-Based Structures

**HashMap / HashSet:** Key-value with O(1) average lookups.

```
Hash("apple") = 10 → arr[10 % 16] = "apple"
Hash("banana") = 200 → arr[200 % 16] = "banana"  // collides at index 8 → chaining
```

### Trees

- **Binary Tree:** Each node has ≤2 children; leaf nodes have no children.
- **BST:** Left child < parent < right child. O(log n) average, O(n) worst (skewed).
- **Balanced BST (AVL, Red-Black):** Self-balancing, guaranteed O(log n).
- **B-Tree:** Multi-child nodes, used in databases (reduces disk I/O).
- **Trie:** Prefix tree for strings. O(k) lookup, O(n) space. Used for autocomplete.
- **Heap:** Complete binary tree where parent ≥ children (max-heap) or ≤ children (min-heap).

### Graphs

- **Adjacency List:** `List<Integer>[] neighbors`. O(V+E) space. Most common.
- **Adjacency Matrix:** `boolean[][] adj`. O(V²) space, O(1) edge check.
- **DFS:** Recursive/stack. O(V+E). Good for connected components, cycles.
- **BFS:** Queue. O(V+E). Shortest path in unweighted graphs.
- **Dijkstra:** Priority queue + distances. O((V+E) log V). No negative edges.
- **Bellman-Ford:** Iterative relax. O(VE). Handles negative weights.

### Sorting

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|------|---------|-------|-------|--------|
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes |

---

## Common Mistakes

- **Bubble sort in production** — O(n²) for any dataset > trivial size
- **Not pre-sizing HashMaps** — Repeated resize storms for large datasets
- **Stack overflow** — Recursive DFS on deep tree; use iterative BFS/DFS
- **Mutable keys in HashSet** — Hash changes after insertion → object lost
- **Overflow in binary search** — `mid = (left + right) / 2` overflows for large ints
- **Off-by-one in binary search** — Wrong boundary updates cause infinite loops
- **Not handling duplicates** — Equal elements cause incorrect results

---

## Key Design Considerations

- **Depth-first vs. breadth-first:** DFS uses less memory (implicit stack), BFS finds shortest paths, DFS better for topological ordering.
- **Adjacency list vs. matrix:** List for sparse graphs, matrix for dense graphs with frequent edge checks.
- **Dynamic vs. static:** Static arrays for fixed sizes, dynamic lists for variable growth.
- **Memory hierarchy:** Sequential array access is cache-friendly; linked lists and trees cause cache misses.
- **Two-pointer vs. sliding window:** Two-pointer for sorted arrays and palindrome checks; sliding window for subarray/substring optimization.
- **Recursion vs. iteration:** Recursion is elegant for trees but may stack overflow; iteration is safer for deep structures.

---

## Real-World Scenarios

### Scenario 1: Real-Time Leaderboard
You need to display a live leaderboard for 10M players in an online game. Scores update every second. Players can view their rank and the top 100. **Design:** Use a Redis sorted set (`ZADD` to update scores, `ZREVRANGE` for top 100, `ZRANK` for individual rank). This gives O(log n) updates and O(1) rank lookups. For persistence, snapshot to a DB every 5 minutes. **Trade-off:** Redis is memory-bound; for 10M entries, estimate ~500MB RAM.

### Scenario 2: URL Shortener
Design bit.ly — generate short codes for long URLs, handle 100M+ URLs, redirect in <10ms. **Design:** Use a distributed ID generator (Snowflake) for unique 7-char base62 codes. Store in a distributed key-value store (Cassandra/DynamoDB) with URL as key, short code as value. Cache hot URLs in Redis. Handle collisions with retry logic. **Trade-off:** Base62 encoding uses more space than hashing but avoids collision complexity.

### Scenario 3: Search Autocomplete
Implement prefix-based autocomplete for a search engine handling 10M queries/day. **Design:** Build a Trie where each node stores the top 10 suggestions (precomputed via min-heap). Update suggestions nightly from search logs. For real-time trending, use a separate sliding window counter. **Trade-off:** Tries use O(n*m) memory (n=words, m=avg length). For large dictionaries, consider a bloom filter + inverted index hybrid.

---

## Scenario-Based Questions

1. **Q: You are building a real-time analytics dashboard that aggregates 1M events/second. Users query rolling 5-minute windows. What data structures do you use?**
   A: Use a ring buffer (circular array) per time window — append new events, evict expired. For aggregations (sum, avg, p99), maintain running counters that update in O(1). For top-K queries, use a min-heap. Trade-off: ring buffer uses fixed memory but loses precision on sparse buckets. Alternative: use Redis TimeSeries or ClickHouse for persistent storage.

2. **Q: You need to find the shortest path in a city's road network (1M nodes, 3M edges) with traffic updates every 5 minutes. How do you make Dijkstra practical?**
   A: Use A* with a heuristic (Manhattan distance). Use a binary heap or Fibonacci heap for the priority queue. Partition the graph into hierarchical levels (highways vs local roads) — route at highway level first, then refine locally. Trade-off: A* may not find the true shortest path with inadmissible heuristics.

3. **Q: Design a recommendation system where users rate items (1-5). You need "users who liked X also liked Y" queries under 50ms. How do you structure the data?**
   A: Build a collaborative filtering model offline (nightly batch). Store top-100 similar items per item in Redis as sorted sets. For online queries, look up the precomputed set in O(1). Trade-off: batch updates mean new items aren't recommended for up to 24h. Alternative: use Apache Spark MLlib for incremental updates.

4. **Q: Your web crawler needs to detect duplicate pages without storing all crawled URLs (10B pages). How?**
   A: Use a Bloom filter — a probabilistic data structure. Set multiple hash bits for each URL. If all bits are set, the page is likely a duplicate. Trade-off: false positives (missed pages) but zero false negatives and O(k) memory per insertion where k is hash count. For 10B URLs, a 1% false positive rate needs ~12GB.

5. **Q: You're implementing autocomplete for a mobile keyboard with 100K words. The user types a prefix — results must appear in <10ms. Memory is constrained (2MB). How?**
   A: Use a compressed Trie (Radix Tree) — merges single-child nodes into one. Store only leaf frequencies. Use a trie + ternary search tree hybrid for memory efficiency. Trade-off: insertion is more complex (node splitting/merging). Alternative: use a finite state transducer (FST) like Lucene's — more complex but minimal memory.

6. **Q: Design a rate limiter for a public API that handles 100K req/s with per-user limits (10/sec, 1000/hour). What data structures?**
   A: Sliding window log per user stored in Redis sorted sets (timestamp as score). Prune expired entries on each request (O(log n)). Better for production: sliding window counter — track current + previous window counts, estimate without storing all timestamps. Trade-off: sliding window counter has slight inaccuracy (±1 window boundary).

7. **Q: You have 1TB of log files on a single machine. Find the top-100 most frequent IP addresses. RAM is 4GB. How?**
   A: Use external sorting + map-reduce in memory. Split files into chunks, count frequencies per chunk in a HashMap, emit (count, IP) pairs, merge-sort with a min-heap of size 100. For higher accuracy: use Count-Min Sketch probabilistic data structure. Trade-off: external sorting is I/O bound; Count-Min Sketch has count overestimation but uses <1MB RAM.

8. **Q: Your e-commerce site needs to find all products within a 10km radius. How do you index 10M products by geolocation?**
   A: Use a Quadtree (2D spatial index) or Google S2 Geometry. Quadtree recursively divides space into quadrants; each leaf contains products in that region. Query: traverse nodes overlapping the 10km circle. Trade-off: Quadtrees are balanced by insertion order, not data density. Alternative: Geohash (Z-order curve) maps 2D to 1D for use in B-trees.

9. **Q: You're building a distributed job queue. Workers pull jobs, process them, and acknowledge. Jobs must be processed exactly-once. What data structures do you use?**
   A: Use a Redis list (LPUSH/BRPOP) for the queue. For exactly-once: move jobs to an in-flight set with a lease TTL. A worker that crashes before acknowledging has its job returned to the queue after TTL expiry. Use a Redis sorted set for delayed/scheduled jobs. Trade-off: Redis is single-threaded for commands; shard across Redis clusters for scale.

10. **Q: Design the data structure for a version control system (like Git). How do you store 100K files with branching, history, and efficient diffs?**
    A: Use a Merkle DAG (Directed Acyclic Graph). Each commit is a tree node pointing to a parent. Files are content-addressed blobs (SHA-1 hash). Trees map filenames → blob hashes. Diffs compare tree hashes. Trade-off: content-addressed storage causes fragmentation; git gc compresses. For large binaries, Git LFS stores pointers in the tree, blobs externally.

---

## Interview Questions

1. **What is Big O notation?**
   A: A mathematical notation describing the upper bound of runtime/memory growth as input size approaches infinity. It ignores constants and lower-order terms.

2. **What is the time complexity of binary search?**
   A: O(log n). Each step halves the search space. Requires sorted data.

3. **What is the difference between an array and a linked list?**
   A: Arrays have O(1) random access but O(n) insert/delete. Linked lists have O(n) access but O(1) prepend/insert at known position.

4. **What is a hash collision and how is it resolved?**
   A: When two keys hash to the same index. Resolved via chaining (linked list per bucket) or open addressing (probing for next empty slot).

5. **When would you use a BFS vs DFS?**
   A: BFS finds shortest path in unweighted graphs and uses more memory (queue). DFS uses less memory (stack) and is better for topological sorting, cycle detection, and exhaustive search.

6. **What is the difference between a min-heap and a max-heap?**
   A: In a min-heap, parent ≤ children (root is minimum). In a max-heap, parent ≥ children (root is maximum). Both support O(log n) insert and extract.

7. **How does quicksort work and what is its worst case?**
   A: Choose a pivot, partition elements into less-than and greater-than, recursively sort. Worst case O(n²) occurs when the pivot is always the smallest/largest element (e.g., already sorted array with first-element pivot).

8. **What is dynamic programming and when do you use it?**
   A: Solving problems by breaking them into overlapping subproblems and storing results to avoid recomputation. Use when the problem has optimal substructure and overlapping subproblems.

9. **Explain the two-pointer technique with an example.**
   A: Two pointers traverse a data structure at different speeds or from different ends. Example: checking if a string is a palindrome — one pointer from start, one from end, compare characters moving inward.

10. **What is the space complexity of a recursive algorithm?**
    A: O(depth of recursion) due to the call stack. Each recursive call adds a frame. Deep recursion can cause stack overflow.

---

## Developer Recommendations

- **Know your data structures' time complexities cold** — Choosing a LinkedList when you need O(1) random access leads to O(n) production performance. Memorize the Big O table for Array, List, HashMap, TreeSet, PriorityQueue, and HashSet. The trade-off: HashMap is O(1) average but uses more memory and has poor cache locality vs arrays.

- **Start with a brute force solution, then optimize** — During interviews, get a working solution first (even O(n²)), then discuss trade-offs and improve. Premature optimization leads to buggy code and wasted time. Use the BUD (Bottlenecks, Unnecessary work, Duplicated work) framework to find optimization opportunities.

- **Use hash-based structures for lookup-heavy problems** — If your algorithm repeatedly searches for elements (contains, indexOf on lists), insert them into a HashSet or HashMap first. The trade-off: O(n) memory for O(1) lookups. This is almost always worth it for non-trivial n.

- **Prefer iterative over recursive for production code** — Recursion is elegant for trees but risks stack overflow for deep structures (thousands of levels). Iterative solutions with explicit stacks are safer. The trade-off: iterative solutions are often more verbose and harder to reason about.

- **Benchmark before optimizing** — O(n log n) may outperform O(n) for small n due to constants and cache effects. Quicksort (O(n log n) average) often beats merge sort in practice because of in-place memory access patterns. Profile first, optimize second.

- **Practice the sliding window pattern** — Many subarray/substring problems (max sum, longest unique, smallest window) reduce to a single sliding window template. Recognizing this pattern saves 30 minutes of problem-solving. Trade-off: not all window-like problems fit — validate that the window can shrink/grow monotonically.

- **Test with edge cases before considering done** — Empty input, single element, duplicates, negative numbers, overflow, null values. 90% of bugs in DSA code come from edge cases, not the core algorithm. Write test assertions for these cases during practice.

- **Use divide and conquer for complex problems** — Split the problem into smaller independent subproblems, solve each, combine. Merge sort, quicksort, binary search, and many tree algorithms follow this pattern. Trade-off: recursion overhead and stack depth limits.
