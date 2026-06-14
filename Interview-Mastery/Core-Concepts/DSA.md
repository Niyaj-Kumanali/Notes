# Data Structures & Algorithms

---

## Overview

- **Definition**
  - Data structures organize and store data for efficient access.
  - Algorithms are step-by-step procedures for solving problems.
  - Together they form the foundation of efficient software.

  **Why It Exists**
    - Without DSA, software is slow (searching 1M records takes minutes), resource-heavy (memory usage scales linearly), and unscalable (what works for 100 users fails for 100K).

  **Historical Context**
    - Before formal complexity analysis (pre-1960s), programmers relied on intuition and hand-tuned assembly.
    - Big O notation, popularized by Donald Knuth in *The Art of Computer Programming* (1968), gave the field a language to reason about scalability independent of hardware.

  **Key Concepts**
    - **Time Complexity** — how runtime grows as input scales: O(1), O(log n), O(n), O(n log n), O(n²).
    - **Space Complexity** — how memory usage grows with input size.
    - **Amortized Analysis** — average cost over a sequence of operations, not per-operation worst case.
    - **Trade-offs** — time vs space, speed vs readability, exactness vs approximation.

---

## Core Concepts

### Asymptotic Analysis & Big O

- Big O describes the upper bound of runtime or memory growth as input size approaches infinity.
- Constants and lower-order terms are dropped because at large n, they become irrelevant — an O(n) algorithm will always eventually outperform an O(n²) algorithm regardless of the constant factor.

| Notation | Name | Example |
|----------|------|---------|
| O(1) | Constant | Array access, HashMap lookup |
| O(log n) | Logarithmic | Binary search, balanced BST |
| O(n) | Linear | Single loop, linked list traversal |
| O(n log n) | Linearithmic | Merge sort, heap sort |
| O(n²) | Quadratic | Nested loops over same array |
| O(2^n) | Exponential | Recursive subsets, brute-force |

```java
// O(1) — array access
int get(int[] arr, int i) { return arr[i]; }

// O(n) — linear search
boolean find(int[] arr, int target) {
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

- **Amortized analysis** matters most with dynamic arrays. A single `append` that triggers a resize is O(n), but averaged over all appends, the cost is O(1) because the resize doubles capacity and becomes increasingly rare. Reporting O(n) per append would be technically correct but misleadingly pessimistic.
- **Space-time trade-off** is the fundamental tension in DSA. Caching computed results (memoization) spends memory to save time. A sorted array enables O(log n) search but costs O(n log n) to build. Every algorithmic choice is implicitly a choice about which resource is more scarce.

---

### Linear Data Structures

- **Arrays**
  - Store elements in contiguous memory, enabling O(1) random access because the address of element `i` is `base + i * element_size` — a single arithmetic operation.
  - Inserting or deleting in the middle requires shifting all subsequent elements, O(n).
  - Arrays are maximally cache-friendly: sequential access loads adjacent elements into the same cache line.

- **Dynamic Arrays (`ArrayList`/`Vec`)**
  - Wrap a fixed array and resize when full. The resize strategy — typically doubling capacity — is what makes append amortized O(1).
  - If you doubled capacity on every single insert instead, each insert would be O(n). Doubling ensures the total work across all inserts is O(n), so the per-insert average is O(1).
  - This is the same reasoning behind `ArrayList`'s 50% growth factor: doubling wastes memory; too-small factors increase resize frequency.

- **Strings**
  - Immutable character arrays in most languages.
  - The classic mistake is concatenating in a loop:

```java
String result = "";
for (String s : list) result += s;  // O(n²) — creates a new String on every iteration
```

  - Each `+=` allocates a new String object of length `result.length + s.length`. Over n iterations, total work is 1 + 2 + 3 + ... + n = O(n²).
  - `StringBuilder` avoids this by maintaining a mutable buffer, making the loop O(n).

- **Linked Lists**
  - Each element is stored in a separate `Node` object with a pointer to the next node.
  - O(1) prepend is possible because inserting at the head requires only changing one pointer.
  - O(n) access is unavoidable because there is no arithmetic shortcut to element `i` — you must follow pointers from the head.
  - Each pointer dereference is also a potential cache miss, since nodes are allocated independently on the heap.

- **Stacks and Queues**
  - Abstract data types that constrain access to one or two ends.
  - Stack (LIFO): `push`/`pop` from the same end. Used for DFS, call stacks, expression parsing, undo history.
  - Queue (FIFO): `enqueue` at back, `dequeue` from front. Used for BFS, job scheduling, buffering.
  - Both are typically implemented with a dynamic array or doubly-linked list.
  - `ArrayDeque` is the correct choice in Java — no per-node overhead, O(1) amortized at both ends.

---

### Hash-Based Structures

- A hash map works by computing an index from the key: `bucket = hash(key) % capacity`.
- If hash values are uniformly distributed, lookups require examining only one bucket rather than scanning the entire structure.

```
hash("apple")  = 10  → arr[10 % 16] = "apple"
hash("banana") = 200 → arr[200 % 16] = "banana"  // index 8 — collision handled by chaining
```

- **Collision handling** is unavoidable because hash functions map an infinite key space to a finite array. Two strategies exist:
  - **Chaining** — each bucket holds a linked list of all keys that hashed there. Degrades to O(n) if too many keys collide.
  - **Open addressing** — on collision, probe for the next empty slot (linear, quadratic, or double hashing). More cache-friendly than chaining but sensitive to load factor.

- Java's `HashMap` uses chaining and converts chains to Red-Black trees when a bucket exceeds 8 entries — improving worst-case from O(n) to O(log n). This was added in Java 8 specifically to mitigate hash-collision DoS attacks.

- **Load factor** controls the balance between memory and performance.
  - At load factor 0.75 (the Java default), the map resizes when 75% of buckets are occupied.
  - Higher load = more collisions but less memory waste.
  - Lower load = fewer collisions but more empty buckets.
  - The 0.75 default represents an empirically good balance for general workloads.

- **Scale inflection point:** HashMap performance degrades sharply when the load factor exceeds ~0.9 under open addressing. Up to 0.75, average probe length grows slowly. Past 0.85, probe length grows exponentially — a map that averages 1.5 probes per lookup at 0.75 load may require 10+ probes at 0.95 load. The same map passes unit tests (small n, few collisions) and looks fine in monitoring (average probe length is hidden) until the production traffic pushes it past the inflection point.

---

### Trees

- All tree-based structures share a single core property: they reduce search from O(n) linear scan to something smaller by organizing data hierarchically.

- **Binary Tree** — each node has at most 2 children. No ordering constraint. Used for expression parsing, Huffman coding.

- **BST (Binary Search Tree)** — left child < parent < right child. Enables O(log n) search on average by eliminating half the tree on each comparison. Degrades to O(n) on a skewed tree (e.g., inserting already-sorted data builds a linked list structure).

- **Balanced BST (AVL, Red-Black)** — self-balancing variants that guarantee O(log n) by performing rotations after insertions and deletions to maintain height ≤ O(log n). AVL trees are more strictly balanced (better for read-heavy workloads), Red-Black trees allow slightly more imbalance in exchange for fewer rotations (better for write-heavy workloads). Java's `TreeMap` and `HashMap` (for treeified buckets) use Red-Black trees.

- **B-Tree** — generalizes BST to nodes with many children (not just 2). Each node stores multiple keys and fits within a single disk page. Minimizes disk I/O by maximizing data per page read. Used in every major relational database (PostgreSQL, MySQL, SQLite) and filesystem for index storage.

- **Trie (Prefix Tree)** — each node represents a character; a path from root to a node spells a prefix. O(k) lookup where k is the key length — independent of how many keys are stored. Used for autocomplete, spell checking, and IP routing tables. The memory cost is O(n × m) where n is the number of words and m is average length. Compressed tries (Radix Trees) merge single-child chains to reduce this.

- **Heap** — a complete binary tree where every parent satisfies the heap property (max-heap: parent ≥ children; min-heap: parent ≤ children). Stored as an array where children of node `i` are at `2i+1` and `2i+2` — no pointers needed. Insert: place at the end, sift up — O(log n). Extract min/max: swap root with last element, sift down — O(log n). The root is always the min or max, which is why heaps back `PriorityQueue`. Iterating a heap does not produce sorted order — only successive `poll()` calls do.

---

### Graphs

- A graph is a set of nodes (vertices) connected by edges. Trees are a special case — a connected acyclic graph.

- **Representation trade-off:**
  - **Adjacency List** (`List<Integer>[] neighbors`) — O(V+E) space. Efficient for sparse graphs (most real-world graphs: social networks, road networks). Iterating neighbors is O(degree).
  - **Adjacency Matrix** (`boolean[][] adj`) — O(V²) space. Efficient for dense graphs where edge existence needs O(1) checking. Wastes memory on sparse graphs.

- **Traversal:**
  - **DFS** — uses a stack (recursive or explicit). O(V+E). Explores as deep as possible before backtracking. Natural for: topological sort, cycle detection, connected components, maze solving.
  - **BFS** — uses a queue. O(V+E). Explores all neighbors at the current depth before going deeper. Natural for: shortest path in unweighted graphs, level-order traversal, bipartiteness testing.

- The choice between DFS and BFS is not stylistic — it determines what the algorithm naturally produces. BFS levels correspond to distances from the source, so shortest-path guarantees are structural. DFS post-order naturally produces reverse topological order.

- **Shortest Path:**
  - **Dijkstra** — greedy approach using a priority queue. Processes nodes in order of their current known distance. O((V+E) log V). Fails on negative-weight edges because it assumes that once a node is processed, its shortest path is finalized — a negative edge discovered later could undercut that.
  - **Bellman-Ford** — relaxes all edges V-1 times. O(VE). Handles negative weights. Detects negative cycles (a cycle whose total weight is negative — no shortest path exists in such a graph).
  - **A\*** — Dijkstra with a heuristic that estimates remaining distance. Explores more promising directions first. Optimal when the heuristic is admissible (never overestimates).

---

### Sorting

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|------|---------|-------|-------|--------|
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes |

- **Why Quicksort dominates in practice** despite its O(n²) worst case: it is in-place (no auxiliary memory), and its cache access pattern is sequential within each partition — adjacent elements are compared and swapped, maximizing cache line reuse. Merge sort's O(n) auxiliary array means every merge step writes to a separate allocation, incurring more cache misses. Java's `Arrays.sort()` uses Dual-Pivot Quicksort for primitives and Timsort for objects.

- **Stability** matters when you sort by multiple keys. If you sort a list of employees first by department, then by name, a stable sort preserves the department ordering within each name group. An unstable sort may not. This is why Merge Sort (stable) is used for objects in Java but Quicksort (unstable) is acceptable for primitives — primitives have no identity beyond their value, so stability is meaningless.

- **Counting Sort** works in O(n+k) by skipping comparisons entirely — it counts occurrences of each value and reconstructs the sorted array from counts. It requires the values to be bounded integers (range k). For k >> n, it wastes memory and time. For k ≈ n (sorting scores 0–100, sorting characters), it beats O(n log n) comparison sorts.

---

## Common Mistakes

- **Bubble sort in production** — Implementing your own sort instead of calling `Arrays.sort()` or `Collections.sort()`.
  - **Why it looks correct:** It produces the right output on small test data and the code is trivially simple. In production, a 100K-element list turns a sub-millisecond sort into 10 seconds of CPU time — the service passes integration tests with 100 elements and only shows the symptom under real load.

- **Not pre-sizing HashMaps** — Using `new HashMap<>()` when the final number of entries is known to be large (e.g., loading 100K database records into a map).
  - **Why it looks correct:** The default constructor works fine for small maps and does not require a manual size estimate. In production, the map grows from 16 to 32 to 64... up to 131K, performing 14 full rehashes that each re-index every entry. At 100K entries, these rehashes add 100ms+ of latency to a cold-start path that was expected to complete in 20ms. `new HashMap<>(expectedSize / 0.75f + 1)` eliminates all resize overhead when the final size is known.

- **Stack overflow from recursive DFS** — Recursive DFS on a tree or graph with thousands of levels consumes one call stack frame per level. JVM default stack depth is ~500–1000 frames depending on frame size. Convert to iterative DFS with an explicit `Deque` stack.

- **Mutable keys in HashSet** — If a key's `hashCode()` changes after insertion, the map looks in the wrong bucket on retrieval and finds nothing. The entry still exists — it is just unreachable. Always use immutable keys.

- **Overflow in binary search** — `mid = (left + right) / 2` overflows when `left + right > Integer.MAX_VALUE`. Use `mid = left + (right - left) / 2` instead. This is a real production bug — it was present in Java's `Arrays.binarySearch` until 2006.

- **Off-by-one in binary search** — The wrong boundary update (`left = mid` instead of `left = mid + 1`) causes an infinite loop when `left` and `right` are adjacent. The boundary update must strictly shrink the search space on every iteration.

- **Not handling duplicates** — Algorithms that assume distinct elements (binary search returning a single index, two-pointer techniques) silently produce wrong results when duplicates are present. Consider what "equal" means for your specific problem before writing comparisons.

- **Interview follow-up:** In a sorted array with duplicates, binary search can return any index where the target exists. How would you modify it to always return the first occurrence?

---

## Key Design Considerations

- **DFS vs BFS** — DFS uses O(depth) memory (the implicit or explicit stack), BFS uses O(width) memory (the queue holding all nodes at the current level). For trees where width >> depth (dense graphs, complete trees), BFS can exhaust memory while DFS stays lean. For trees where depth >> width (linked-list-like structures), DFS risks stack overflow while BFS stays shallow.

- **Adjacency list vs matrix** — Use adjacency list for sparse graphs (most real-world networks where E << V²). Use adjacency matrix when you need O(1) edge existence checks or are working with dense graphs. For a 1M-node sparse graph, a matrix would require 10¹² bits — impossible.

- **Static vs dynamic structures** — Static arrays allocate a fixed block of contiguous memory at creation. Dynamic lists allocate incrementally. Static arrays are faster (no pointer dereferencing, better cache behavior) but require knowing size upfront. The common pattern is to use dynamic structures during construction, then convert to a static array when the size is finalized.

- **Memory hierarchy** — Sequential array access is cache-friendly because hardware prefetchers detect the access pattern and load ahead. Pointer-following structures (linked lists, trees) cause cache misses on almost every access because nodes are scattered in heap memory. At scale, this difference can be the primary performance factor regardless of algorithmic complexity.

- **Two-pointer vs sliding window:**
  - Two-pointer — two indices moving toward each other or at different speeds. Best for sorted arrays, palindrome checks, and cycle detection.
  - Sliding window — a range `[left, right]` that expands and contracts. Best for subarray/substring problems where you need the optimal contiguous range satisfying a condition.
  - The key distinction: two-pointer typically shrinks the problem from both ends; sliding window expands from one end and contracts from the other.

- **Recursion vs iteration** — Recursion is structurally natural for problems defined recursively (trees, divide-and-conquer). Iteration is safer for production code on deep structures. When converting, use an explicit stack to simulate the call stack rather than rewriting the logic from scratch.

---

## Real-World Scenarios

### Scenario 1: Real-Time Leaderboard

- **Context**
  - You need to display a live leaderboard for 10M players. Scores update every second. Players view their rank and the top 100.

- **Design**
  - Use a Redis sorted set (`ZADD` to update scores, `ZREVRANGE` for top 100, `ZRANK` for individual rank). Redis sorted sets are backed by both a hash map (O(1) score lookup by member) and a skip list (O(log n) ordered traversal). This gives O(log n) updates and O(log n + k) range queries. For persistence, snapshot to a database every 5 minutes.

- **Trade-off**
  - Redis is memory-bound. At 10M entries with ~50 bytes per entry, expect ~500MB RAM. The skip list also carries more memory overhead than a pure array structure. If memory is the primary constraint and exact real-time rank is not required, approximate rank via a sampled sorted structure trades precision for memory.

- **Interview follow-up:** What happens to the leaderboard if Redis goes down for 30 seconds — how do you prevent score loss during that window?

### Scenario 2: URL Shortener

- **Context**
  - Design bit.ly — 100M+ URLs, redirect in < 10ms.

- **Design**
  - Use a distributed ID generator (Snowflake) for unique 7-char base62 codes. Store in a distributed key-value store (Cassandra/DynamoDB) with the short code as key and long URL as value. Cache hot URLs in Redis (a small fraction of URLs receive the vast majority of traffic — LRU cache handles this effectively).

- **Trade-off**
  - Base62 encoding is deterministic per ID — no collision risk — but the ID generator becomes a potential bottleneck and single point of failure. Hash-based approaches (MD5 truncated to 7 chars) risk collisions and require retry logic but distribute generation across nodes. The ID generator approach trades centralization for predictability.

- **Interview follow-up:** How do you handle the case where two different long URLs produce the same short code after a hash collision — what does your retry logic look like at 10K writes/second?

### Scenario 3: Search Autocomplete

- **Context**
  - Prefix-based autocomplete for 10M queries/day, results in < 10ms.

- **Design**
  - Build a Trie where each node stores the top 10 suggestions precomputed via a min-heap during index construction. Update suggestions nightly from search logs. For real-time trending queries, maintain a separate sliding window counter that feeds into a hot-prefix cache.

- **Trade-off**
  - Tries use O(n × m) memory where n is the number of words and m is average length. A naive Trie for a 1M-word dictionary can consume hundreds of MB. Compressed Tries (Radix Trees) reduce this by merging single-child chains. For truly memory-constrained environments, a Finite State Transducer (FST) — used by Lucene — provides minimal-memory prefix lookup at the cost of construction complexity.

- **Interview follow-up:** How do you handle trending or new queries that were not in last night's training data — what data structure supports real-time suggestion additions without rebuilding the entire index?

## Use Cases

- Reach for DSA knowledge whenever you need to make software predictable in time and memory under load. The right data structure can turn an O(n²) operation into O(log n), and the right algorithm determines whether a feature works at 10 users or 10 million.

- **Efficient lookups** — choose the right map or set
  - Use a hash map (`HashMap`, `Dictionary`) for O(1) average lookups when keys are unordered. Use a tree map (`TreeMap`, `std::map`) when you need sorted iteration or range queries. For ~1M+ entries, measure the hash function quality — a poorly distributed hash degrades to O(n) buckets.
  - **Avoid when:** a simple array or list suffices (fewer than 100 items, dense integer keys) — the overhead of hashing and boxing is wasted.

- **Fast insertion and deletion** — list vs. linked list vs. array
  - Use `ArrayList` for index-based access and when the size is known ahead of time. Use `LinkedList` or a `Deque` when you need constant-time inserts/removals at both ends (queue, stack, sliding window). For arbitrary insertions in the middle, consider a balanced tree or a skip list.
  - **Avoid when:** you need both fast random access and fast middle insertions — no single structure excels at both; consider a hybrid approach (e.g., a B-tree or an array of chunks).

- **Priority processing** — heaps and priority queues
  - Use a min-heap or max-heap when you repeatedly need the smallest (or largest) element from a dynamic set: task scheduling, Dijkstra's algorithm, top-K queries. Insertion and extraction are both O(log n).
  - **Avoid when:** you need to frequently update the priority of existing elements — a Fibonacci heap or a bucket queue may be better, or a sorted list if updates are rare.

- **Graph traversal** — BFS vs. DFS
  - Use BFS (breadth-first) for shortest path in unweighted graphs and level-order processing. Use DFS (depth-first) for topological sorting, cycle detection, and exploring all paths. For weighted shortest paths, use Dijkstra (non-negative weights) or Bellman-Ford (negative weights).
  - **Avoid when:** the graph is implicit and infinite (e.g., a game tree for chess) — iterative deepening DFS or A* with a good heuristic limits exploration.

- **String searching** — pattern matching at scale
  - Use KMP or Boyer-Moore for searching a pattern in a long text (log parsing, DNA sequence matching). Use a Trie (prefix tree) for autocomplete, spell-check, or IP routing tables. Use Rabin-Karp for multi-pattern search in a single pass.
  - **Avoid when:** simple `indexOf` or regex suffices (text < 10K characters, simple pattern) — the implementation cost of advanced algorithms outweighs the gain.

---

## Scenario-Based Questions

**Q: You are building a real-time analytics dashboard aggregating 1M events/second. Users query rolling 5-minute windows. What data structures do you use?**

- Use a ring buffer (circular array) per time bucket — new events overwrite the oldest as time advances. For aggregations (sum, average, p99), maintain running counters that update in O(1) per event. For top-K queries, use a min-heap of size K. The ring buffer's fixed size gives predictable memory usage regardless of event volume, which is critical for a high-throughput system. Trade-off: fixed bucket granularity loses precision for queries that don't align with bucket boundaries. Redis TimeSeries or ClickHouse are better choices when persistence and ad-hoc query flexibility matter more than latency.
- **Non-obvious constraint:** The ring buffer only supports fixed-resolution queries — asking for a 3-minute window within a system that buckets by 1-minute intervals requires merging at query time, which adds latency.
- **Interview follow-up:** How do you handle clock skew between multiple servers producing events in this ring buffer — what happens if server A's clock is 10 seconds ahead of server B's?

---

**Q: Find the shortest path in a city road network (1M nodes, 3M edges) with traffic updates every 5 minutes.**

- Vanilla Dijkstra on 1M nodes is too slow for real-time routing. Use A* with a geographic heuristic (straight-line or Manhattan distance) to prioritize exploration toward the destination. Combine with Contraction Hierarchies — precompute shortcut edges that skip intermediate nodes on known highway-level routes. This reduces query time from seconds to milliseconds by routing at the highway level first, then refining locally. Trade-off: A* is only optimal when the heuristic is admissible. Contraction Hierarchies require expensive preprocessing when the graph changes — traffic updates invalidate precomputed shortcuts for affected edges.
- **Non-obvious constraint:** The straight-line heuristic works for road networks but is not admissible in all graphs (e.g., a mountain road that requires a long detour looks close on a straight line). The heuristic must never overestimate for A* to remain optimal.
- **Interview follow-up:** How do you update Contraction Hierarchies when a traffic incident changes edge weights on 5% of the road network — do you rebuild from scratch?

---

**Q: Design a recommendation system where "users who liked X also liked Y" queries must run under 50ms.**

- Build a collaborative filtering model offline in a nightly batch job. Store the top-100 similar items per item in Redis sorted sets (item similarity score as the sort key). Online queries look up the precomputed set in O(1). The key insight is that freshness matters less than latency here — a recommendation that is 12 hours stale is almost always acceptable. Trade-off: new items are not recommended for up to 24 hours after they appear. For platforms where new item discovery is critical (news, trending content), supplement with a real-time signal layer using approximate nearest neighbor search (Faiss, ScaNN) over recent interaction embeddings.
- **Non-obvious constraint:** The assumption that "freshness matters less than latency" breaks down during viral events — a new item that gets 10K interactions in an hour is invisible for up to 24 hours.
- **Interview follow-up:** How do you handle the cold-start problem for a user who just signed up and has no interaction history — what do you recommend?

---

**Q: Your web crawler needs to detect duplicate pages without storing all 10B crawled URLs. How?**

- Use a Bloom filter — a bit array with k hash functions. To insert a URL, set k bits at positions determined by the k hash functions. To check, verify all k bits are set. If any bit is unset, the URL is definitely not seen before. If all bits are set, it is probably a duplicate. The false positive rate is tunable: for 10B URLs with 1% false positive rate, a Bloom filter needs approximately 12GB — far less than storing URLs as strings (~800GB). The trade-off is accepting occasional re-crawls of already-visited pages (false positives) in exchange for massive memory savings. There are zero false negatives — a URL that was inserted will always be detected as seen.
- **Non-obvious constraint:** The Bloom filter cannot be resized or have entries removed. If the crawl set grows beyond the planned capacity, the false positive rate spikes — you must rebuild it with a larger bit array and re-insert all known URLs.
- **Interview follow-up:** A Bloom filter cannot delete URLs. How would you handle re-crawling a site that may have changed — how do you mark a URL as "needs re-check" without false negatives on re-insertion?

---

**Q: Implement autocomplete for a mobile keyboard with 100K words in under 10ms. Memory limit: 2MB.**

- A full Trie for 100K words exceeds 2MB. Use a compressed Trie (Radix Tree) — chains of single-child nodes are merged into one edge — to reduce node count. Store only leaf-level frequencies, not intermediate counts. For further compression, a Finite State Transducer (FST) encodes all words as a minimal acyclic automaton, achieving near-theoretical minimum memory. Trade-off: FST construction is expensive and complex; it is built once offline and loaded into memory at startup. Insertion after construction requires a full rebuild. For a mobile keyboard where the dictionary is static between app updates, this is acceptable.
- **Non-obvious constraint:** The 10ms latency budget includes serialization and IPC between the keyboard process and the autocomplete service — the actual data structure lookup must complete in under 1ms.
- **Interview follow-up:** How do you handle the first few characters of a prefix before the user finishes typing — at what point in the keystroke sequence do you trigger the autocomplete query?

---

**Q: Design a rate limiter for 100K req/s with per-user limits (10/sec, 1000/hour).**

- For the per-second limit, use a sliding window counter per user in Redis: track the count in the current second and the previous second, then estimate the current rate as `prev_count × (1 - elapsed_fraction) + curr_count`. This avoids storing individual timestamps while approximating a true sliding window with negligible error. For the per-hour limit, use token buckets: each user has a bucket refilled at 1000/3600 tokens per second. Token buckets handle bursts gracefully — a user who has been idle can send a burst up to the bucket capacity. Trade-off: sliding window counters have a ~±1 window error at boundaries; token buckets allow short bursts above the stated rate. For strict per-second enforcement, store a sorted set of timestamps and prune on each request — accurate but O(log n) per request and higher memory.
- **Non-obvious constraint:** The per-second and per-hour limits are checked independently — a user could send 10 requests right before the second boundary and 10 more right after, effectively doubling the per-second rate within a 2-second window.
- **Interview follow-up:** How would you implement a distributed rate limiter that works correctly even when a user's requests are spread across 5 backend servers?

---

**Q: You have 1TB of log files on a single machine with 4GB RAM. Find the top-100 most frequent IP addresses.**

- External merge sort approach: split the log into 4GB chunks, count IP frequencies per chunk with a `HashMap`, write `(count, IP)` pairs to disk, then merge using a min-heap across all chunk outputs — keeping only the top 100 globally. Time complexity is O(n log n) dominated by the sort phase with heavy I/O. For an approximate answer in much less time, use Count-Min Sketch: a 2D array of counters with multiple hash functions per row. Each IP increments k counters; the frequency estimate is the minimum across those counters. Count-Min Sketch fits in kilobytes of RAM and processes the full 1TB in a single pass. Trade-off: counts are overestimates (never underestimates), with error bounded by `ε × total_events` for a sketch of width `1/ε`.
- **Non-obvious constraint:** The external merge sort approach does not handle 1TB in a single sorted pass — if the IPs are uniformly distributed across files, each chunk requires a full HashMap fit in 4GB, which limits the number of distinct IPs you can count per chunk.
- **Interview follow-up:** How would you extend this to handle streaming log data where you cannot store all chunks on disk — what if the logs are arriving in real time?

---

**Q: Find all products within a 10km radius from 10M products indexed by geolocation.**

- Use a Geohash index. Geohash encodes a 2D coordinate as a 1D string (Z-order curve) where shared prefixes correspond to nearby regions. Products sharing a Geohash prefix are geographically close. Store products in a B-tree indexed by Geohash — a range query over nearby prefixes retrieves candidate products, then filter by exact distance. The Z-order curve does not perfectly preserve 2D proximity (edge cases near region boundaries), so query slightly larger Geohash cells and filter. Trade-off: Geohash boundary artifacts require querying 8 neighboring cells to guarantee no misses near boundaries, multiplying query work by 9. A Quadtree avoids this by splitting on actual data density but is harder to store in a relational database index.
- **Non-obvious constraint:** The 10km radius is computed on a sphere, not a flat plane. Near the equator, 1 degree of longitude ≈ 111km; near the poles, it approaches 0. A fixed-size Geohash prefix does not represent the same physical area at all latitudes.
- **Interview follow-up:** How would you satisfy a "sort by distance" query while still using a spatial index — what data structure supports both range filtering and distance ordering in the same query?

---

**Q: You are building a distributed job queue. Jobs must be processed exactly-once. What data structures do you use?**

- Use a Redis List (`LPUSH`/`BRPOP`) for the queue. For exactly-once semantics: on dequeue, atomically move the job to an in-flight sorted set with `timestamp + lease_TTL` as the score (`BRPOPLPUSH`). A worker that processes and acknowledges removes the job from the in-flight set. A worker that crashes leaves the job in the in-flight set; a reaper process polls for jobs whose score (expiry time) has passed and returns them to the queue. For delayed/scheduled jobs, a separate Redis sorted set with scheduled execution time as score is checked periodically. Trade-off: Redis is single-threaded for commands — at extreme queue depths, pipeline commands and shard across multiple Redis instances. At-least-once delivery is easy; exactly-once requires the worker to implement idempotent processing so that re-delivered jobs produce the same result.
- **Non-obvious constraint:** The reaper's poll interval determines the minimum visibility timeout — if the reaper polls every 5 seconds, a crashed worker's job will not be retried for up to 5 seconds. Setting the reaper interval too low wastes CPU on empty polls.
- **Interview follow-up:** What happens if the reaper process itself crashes while returning jobs from the in-flight set back to the queue — how do you prevent job loss or double-processing?

---

**Q: Design the data structure for a version control system like Git.**

- Use a Merkle DAG (Directed Acyclic Graph). Each commit node points to its parent commit(s) — multiple parents for merges. Each commit points to a tree object that maps filenames to blob hashes. Blobs are content-addressed: the blob's name is the SHA-1 of its content, so identical file content across branches or history is stored only once. Diffs compare tree objects — only subtrees whose root hash changed need examination, making diff O(changed files) rather than O(all files). Trade-off: content-addressed storage fragments the object store over time. `git gc` (garbage collection) packs loose object files into compressed packfiles using delta encoding (storing diffs between similar blobs rather than full copies). Git LFS handles large binaries by storing a pointer blob in the tree and the actual content on an external server — the tree structure stays intact, but objects that would bloat the repository are externalized.
- **Non-obvious constraint:** The SHA-1 hash collision resistance (now deprecated in favor of SHA-256) is the only guarantee that two different file contents never produce the same blob hash — a collision silently merges the two files into one.
- **Interview follow-up:** How does `git merge` resolve conflicts between two branches that both modified the same file — what data structure represents a conflicted state?

---

## Interview Questions

- **What is Big O notation?**
  - A mathematical notation describing the upper bound of runtime or memory growth as input size approaches infinity. Constants and lower-order terms are dropped because they become irrelevant as n grows. O(2n) and O(n) are both O(n).

- **What is the time complexity of binary search?**
  - O(log n). Each step eliminates half the remaining search space. After k steps, the remaining space is n/2^k — it reaches 1 when k = log₂(n). Requires the data to be sorted. The log₂(n) bound assumes O(1) random access — on a linked list, binary search is O(n log n) because the mid-point access itself costs O(n).

- **What is the difference between an array and a linked list?**
  - Arrays have O(1) random access (direct address arithmetic) but O(n) insert/delete due to element shifting. Linked lists have O(1) prepend and O(1) insert at a known position, but O(n) access because there is no address shortcut — you must follow pointers from the head.

- **What is a hash collision and how is it resolved?**
  - A collision occurs when two keys hash to the same bucket index. Resolved via chaining (each bucket holds a linked list or tree of all colliding keys) or open addressing (probe for the next available slot using linear, quadratic, or double-hashing). Chaining degrades gracefully under high load; open addressing is more cache-friendly but requires careful load factor management. Open addressing with linear probing also suffers from primary clustering — a collision at one bucket increases the probability of collision at the next bucket, creating dense runs that degrade performance to near O(n) under high load.

- **When would you use BFS vs DFS?**
  - BFS finds the shortest path in unweighted graphs and processes nodes level by level — natural for problems about distance or reachability within a bounded number of steps. DFS uses less memory (O(depth) vs O(width)), and its post-order traversal naturally produces reverse topological order, making it better for dependency resolution, cycle detection, and exhaustive search of all paths.

- **What is the difference between a min-heap and a max-heap?**
  - In a min-heap, every parent is ≤ its children, so the root is always the minimum. In a max-heap, every parent is ≥ its children, so the root is the maximum. Both support O(log n) insert and extract-min/max, and O(1) peek at the root. Internally, both store elements in an array using the formula: children of index `i` are at `2i+1` and `2i+2`.

- **How does quicksort work and what is its worst case?**
  - Choose a pivot, partition the array into elements less than and greater than the pivot, then recursively sort each partition. Average O(n log n) because each partition roughly halves the problem. Worst case O(n²) occurs when the pivot is always the minimum or maximum (e.g., sorted input with first-element pivot) — each partition step reduces the problem by only one element. Randomized pivot selection or median-of-three selection makes worst case extremely unlikely in practice.

- **What is dynamic programming and when do you use it?**
  - Solving optimization or counting problems by breaking them into overlapping subproblems and storing results to avoid recomputation. Apply when two conditions hold: optimal substructure (the global optimum can be built from local optima) and overlapping subproblems (the same subproblem is solved multiple times in a naive recursive approach). Classic examples: Fibonacci, 0/1 knapsack, longest common subsequence, shortest path with DP relaxation.

- **Explain the two-pointer technique with an example.**
  - Two indices traverse the data structure simultaneously, either from both ends or at different speeds. Example: detect a cycle in a linked list — slow pointer advances one step, fast pointer advances two. If there is a cycle, fast eventually laps slow and they meet. Example: check if a string is a palindrome — left pointer from start, right from end, advance inward until they cross or a mismatch is found.

- **What is the space complexity of a recursive algorithm?**
  - O(depth of recursion) due to call stack frames. Each recursive call adds a frame containing local variables and the return address. A recursive DFS on a tree of depth d uses O(d) stack space. For a balanced tree this is O(log n); for a degenerate (linked-list) tree this is O(n), risking stack overflow on large inputs.

- **What is the difference between a HashMap and a TreeMap?**
  - HashMap provides O(1) average lookup/insert using hash tables but does not maintain any order. TreeMap provides O(log n) operations using a Red-Black tree and maintains keys in sorted order (natural ordering or custom Comparator).
  - Choose HashMap when order does not matter and you need the fastest access. Choose TreeMap when you need sorted iteration, range queries (subMap, headMap, tailMap), or floor/ceiling operations.

- **Explain the sliding window technique and when to use it.**
  - A window `[left, right]` that expands and contracts across the data. Used for subarray/substring problems where you need the optimal contiguous range satisfying a condition.
  - Example (maximum sum subarray of size k): expand right to add elements, contract left when the window exceeds size k. Each element enters once and leaves once — O(n) total. The technique works when the condition is monotonic (expanding the window never makes a valid window invalid for the property you care about).

- **What is the difference between a stack and a queue?**
  - Stack (LIFO) — last element pushed is the first popped. Used for DFS, expression evaluation, undo history. Queue (FIFO) — first element enqueued is the first dequeued. Used for BFS, job scheduling, buffering.
  - In Java, `ArrayDeque` implements both interfaces — use `Deque` with `push`/`pop` for stack semantics, `add`/`poll` for queue semantics.

- **What is the time complexity of merge sort and why is it stable?**
  - O(n log n) in all cases (best, average, worst). Divide the array into halves recursively (log n levels), then merge each level (O(n) per level). Stability comes from the merge step: when two elements are equal, the element from the left subarray is placed first, preserving the original relative order.
  - The O(n) auxiliary space is the main disadvantage — merging requires a temporary array of size n.

- **What is a trie and what problems does it solve?**
  - A tree where each node represents a character; paths from root to nodes spell prefixes. O(k) lookup where k is the key length — independent of dictionary size.
  - Solves: autocomplete (find all words with a given prefix), spell checking (is this a valid word?), longest common prefix, and IP routing (longest prefix match). Trade-off: O(n × m) memory where n is the number of words and m is average length.

- **How does Java's HashMap handle collisions after Java 8?**
  - Java 8+ converts a collision bucket from a linked list to a Red-Black tree when a bucket exceeds 8 entries (TREEIFY_THRESHOLD). This improves worst-case from O(n) to O(log n) — a defense against hash-collision DoS attacks.
  - When the bucket shrinks below 6 entries (UNTREEIFY_THRESHOLD), it converts back to a linked list. The threshold gap prevents oscillation between list and tree on edge operations.

- **What is the difference between counting sort and comparison-based sorts?**
  - Counting sort does not compare elements — it counts occurrences of each value and reconstructs the sorted array from those counts. O(n + k) where k is the value range. Works only for integers or objects that can be mapped to integer keys.
  - Comparison sorts (quicksort, mergesort) are O(n log n) on average and work on any comparable type. Counting sort beats comparison sorts when k is not much larger than n (e.g., sorting 1M scores in range 0-100).

- **What is amortized analysis and why does it matter for dynamic arrays?**
  - Amortized analysis averages the cost of operations over a sequence, reporting the per-operation average rather than the worst case of a single operation.
  - For ArrayList: a single `add` that triggers resize is O(n), but resize happens only when the array is full. After each resize (doubling capacity), n/2 more O(1) adds follow before the next resize. Total work across n adds is ~3n, making the amortized cost O(1). Without amortized analysis, a dynamic array's `add` would be reported as O(n) — technically true per operation but misleadingly pessimistic.

- **What is the difference between Bellman-Ford and Dijkstra's algorithms?**
  - Dijkstra's greedily processes nodes by shortest known distance using a priority queue — O((V+E) log V). Fails on negative-weight edges because it assumes processed nodes are finalized.
  - Bellman-Ford relaxes all edges V-1 times — O(VE). Handles negative weights and detects negative cycles (a cycle whose total weight is negative, making shortest paths undefined). Use Dijkstra for positive-weight graphs; use Bellman-Ford when negative edges may exist or you need cycle detection.

---

## Advanced Topics

### Dynamic Programming: Patterns and Intuition

- DP is not a single technique — it is a family of patterns. Recognizing which pattern applies is the actual skill.

- **Memoization vs tabulation:**
  - Memoization (top-down) — write the recursive solution naturally, then add a cache. Only computes subproblems that are actually needed.
  - Tabulation (bottom-up) — fill a table from the smallest subproblem upward. No recursion overhead, better cache locality, easier to optimize space.
  - Both produce the same result. Use memoization when the subproblem space is sparse (not all subproblems are needed); use tabulation when the space is dense and the order of computation is clear.

- **Common DP patterns:**

  1. **Linear DP** — subproblem depends on the previous 1–2 states. Fibonacci, house robber, climbing stairs. Often O(n) time, reducible to O(1) space if only the last k states are needed.

  2. **Interval DP** — subproblem is defined over a range `[i, j]`. Matrix chain multiplication, burst balloons, palindrome partitioning. O(n²) states, O(n³) total.

  3. **Knapsack (Bounded selection)** — choose a subset of items under a constraint to maximize value. O(n × W) where W is the weight limit. The key insight: `dp[i][w] = max(dp[i-1][w], dp[i-1][w-weight[i]] + value[i])` — either skip item i or include it.

  4. **LCS / Edit Distance** — two-sequence problems. `dp[i][j]` represents the answer for the first i characters of one string and j of the other. O(n × m) time and space, reducible to O(min(n,m)) space.

  5. **DP on trees** — subproblem is defined per subtree. Compute children first (post-order), pass results up. Used in maximum path sum, tree diameter, subtree counting.

```java
// Longest Common Subsequence — tabulation
int lcs(String a, String b) {
    int n = a.length(), m = b.length();
    int[][] dp = new int[n+1][m+1];
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= m; j++)
            dp[i][j] = a.charAt(i-1) == b.charAt(j-1)
                ? dp[i-1][j-1] + 1
                : Math.max(dp[i-1][j], dp[i][j-1]);
    return dp[n][m];
}
```

- **Space optimization:** Many DP problems only require the previous row of the table. Rolling arrays reduce O(n × m) space to O(m). When doing this, be careful about the direction of iteration — filling right-to-left or left-to-right changes whether you read from the current or previous row.

---

### Graph Algorithms: Beyond BFS/DFS

- **Topological Sort** orders nodes of a DAG (Directed Acyclic Graph) such that every edge points from an earlier node to a later one. Two implementations:
  - Kahn's algorithm (BFS-based): repeatedly remove nodes with in-degree 0. If all nodes are removed, the sort is valid. If nodes remain, a cycle exists.
  - DFS-based: run DFS and append each node to a stack in post-order. Reverse the stack.
  - Applications: build systems (Maven, Gradle compile ordering), course scheduling, spreadsheet cell evaluation.

- **Union-Find (Disjoint Set Union)** tracks connected components dynamically as edges are added.

```java
int[] parent, rank;

int find(int x) {
    if (parent[x] != x) parent[x] = find(parent[x]);  // path compression
    return parent[x];
}

void union(int x, int y) {
    int px = find(x), py = find(y);
    if (px == py) return;
    if (rank[px] < rank[py]) { int t = px; px = py; py = t; }
    parent[py] = px;
    if (rank[px] == rank[py]) rank[px]++;
}
```

  - With path compression and union by rank, both `find` and `union` run in amortized O(α(n)) — effectively constant. Used in Kruskal's MST algorithm, cycle detection, and network connectivity problems.

- **Minimum Spanning Tree (MST):**
  - Kruskal's — sort all edges by weight, greedily add the cheapest edge that does not create a cycle (use Union-Find for cycle detection). O(E log E).
  - Prim's — grow the MST from a starting node, always adding the cheapest edge connecting the current tree to a new node (use a priority queue). O((V+E) log V).
  - Use Kruskal's for sparse graphs (few edges to sort), Prim's for dense graphs (fewer priority queue operations than sorting all edges).

- **Strongly Connected Components (SCC):**
  - A subset of a directed graph where every node is reachable from every other.
  - Kosaraju's algorithm runs two DFS passes — one on the original graph to get finish-order, one on the reversed graph in reverse finish-order.
  - Tarjan's algorithm finds SCCs in a single DFS using a low-link value per node.
  - SCCs appear in compiler optimization, social network analysis, and dependency cycle detection.

---

### Advanced Sorting and Searching

- **Timsort** — Java's `Arrays.sort` for objects and Python's `list.sort`. Hybrid of merge sort and insertion sort. Detects naturally sorted "runs" in the input and merges them. Performs at O(n) on nearly-sorted data and O(n log n) worst case. The key insight: real-world data is rarely random — it typically has partial order (logs sorted by time, names partially alphabetized). Timsort exploits this.

- **Radix Sort** — processes digits from least significant to most significant (LSD) or most to least (MSD). O(d × (n + k)) where d is digit count and k is the digit range. For fixed-length integers, d is constant, making this O(n) — faster than any comparison sort for large n. Used in integer sorting, string sorting, and as a subroutine in larger algorithms.

- **Binary search variations** — the pattern `left + (right - left) / 2` is the template, but the boundary conditions vary:

```java
// Find first position where condition is true (leftmost true)
int lo = 0, hi = n;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (condition(mid)) hi = mid;
    else lo = mid + 1;
}
// lo is the answer

// Find last position where condition is true (rightmost true)
int lo = -1, hi = n - 1;
while (lo < hi) {
    int mid = lo + (hi - lo + 1) / 2;  // upper-mid to avoid infinite loop
    if (condition(mid)) lo = mid;
    else hi = mid - 1;
}
```

  - The template unifies all binary search variants: define what "true" means for the condition, decide whether you want the leftmost or rightmost true, and apply the appropriate boundary update. Getting this wrong is the most common source of off-by-one bugs.

---

### Probabilistic Data Structures

- These structures trade exactness for dramatic reductions in memory and computation. They are appropriate when approximate answers are acceptable — which is often true for analytics, monitoring, and deduplication at scale.

- **Bloom Filter** — tests set membership. Never has false negatives. Has tunable false positive rate. Uses k hash functions over a bit array of size m.
  - For n expected insertions at false positive rate p:
    - Optimal m = `-(n × ln(p)) / (ln(2))²`
    - Optimal k = `(m/n) × ln(2)`
  - At 1% false positive rate: ~10 bits per element. At 0.1%: ~15 bits per element. Storing 10B URLs as strings needs ~800GB; a Bloom filter needs ~12GB at 1%.
  - Applications: Google BigTable uses Bloom filters to avoid disk reads for non-existent keys. Chrome uses them to check safe-browsing lists without sending URLs to Google's servers.

- **Count-Min Sketch** — estimates frequencies. Stores a 2D array of w × d counters with d independent hash functions. To insert: increment one counter per row at the column `hash_i(item) % w`. To query: return the minimum across all d rows. The minimum is an overestimate — it can only be inflated by collisions with other items, never reduced.
  - Error bound: with probability 1 - δ, the estimated count is within `ε × total_count` of the true count, using `w = e/ε` and `d = ln(1/δ)`.
  - Applications: tracking heavy hitters in network traffic, per-IP request counting at ISP scale, approximate frequency in streaming data pipelines.

- **HyperLogLog** — estimates the cardinality of a set (count distinct). Uses ~1.5KB of memory regardless of set size. Error is typically ±2%. The core insight: the maximum number of leading zeros in any hash value in a stream is correlated with log₂(distinct elements). By tracking this statistic across multiple hash buckets and applying harmonic mean correction, cardinality is estimated with small relative error.
  - Applications: Redis `PFCOUNT`, counting unique visitors in web analytics, estimating distinct queries in a search engine — any scenario where `SELECT COUNT(DISTINCT ...)` would be too expensive.

---

### Segment Trees and Fenwick Trees

- Both answer range queries over an array and support point updates efficiently. The choice depends on what operations you need.

- **Fenwick Tree (Binary Indexed Tree)** — answers prefix sum queries and supports point updates. O(log n) per operation, O(n) space. Conceptually simpler and smaller constant than a segment tree.

```java
int[] bit = new int[n + 1];

void update(int i, int delta) {
    for (; i <= n; i += i & (-i)) bit[i] += delta;
}

int query(int i) {
    int sum = 0;
    for (; i > 0; i -= i & (-i)) sum += bit[i];
    return sum;
}

int rangeQuery(int l, int r) { return query(r) - query(l - 1); }
```

  - The `i & (-i)` extracts the lowest set bit of `i`, which determines exactly which cells this index is responsible for. The pattern is elegant but opaque — understanding why it works requires knowing how the BIT maps responsibilities to indices.

- **Segment Tree** — a binary tree where each node stores the aggregate (sum, min, max, GCD) of a subarray range. Supports both range queries and range updates (with lazy propagation). O(log n) per operation, O(n) space.

```java
void build(int[] arr, int node, int start, int end) {
    if (start == end) { tree[node] = arr[start]; return; }
    int mid = (start + end) / 2;
    build(arr, 2*node, start, mid);
    build(arr, 2*node+1, mid+1, end);
    tree[node] = tree[2*node] + tree[2*node+1];
}

int query(int node, int start, int end, int l, int r) {
    if (r < start || end < l) return 0;
    if (l <= start && end <= r) return tree[node];
    int mid = (start + end) / 2;
    return query(2*node, start, mid, l, r)
         + query(2*node+1, mid+1, end, l, r);
}
```

- **Lazy propagation** defers range updates: instead of updating all leaves immediately, mark internal nodes with a pending update and push it down only when the subtree is accessed. This reduces range-update + range-query from O(n) to O(log n).

- Use a Fenwick Tree when you only need prefix sums or simple point updates — it is simpler to implement and faster in practice. Use a Segment Tree when you need range updates, custom aggregation functions (max, min, GCD), or lazy propagation.

---

### String Algorithms

- **KMP (Knuth-Morris-Pratt)** — finds a pattern of length m in a text of length n in O(n + m). The key is the failure function: a precomputed array that tells the algorithm how far to shift the pattern when a mismatch occurs, without re-examining already-matched characters. Naive string matching is O(n × m) because it restarts from the beginning of the pattern on every mismatch. KMP avoids this by recognizing that a partial match tells you something about the next possible alignment — the failure function encodes the longest proper prefix of the pattern that is also a suffix of the matched portion.

- **Rabin-Karp** — uses a rolling hash to find pattern matches. Compute the hash of the pattern and the hash of each text window of the same length. Advance the window in O(1) by subtracting the outgoing character's contribution and adding the incoming character's. Match the hash first; confirm with character comparison only on matches. O(n + m) average, O(n × m) worst case due to hash collisions. Well-suited for multi-pattern search — hash all patterns into a set and check each window against the set.

- **Suffix Arrays and LCP Arrays** — a suffix array is the sorted array of all suffixes of a string. Combined with an LCP (Longest Common Prefix) array, it enables O(log n) substring search, O(n) longest repeated substring, and O(n) longest common substring between two strings. More memory-efficient than suffix trees (which give the same asymptotic bounds) and easier to implement correctly.

---

### Bit Manipulation

- Bit operations are O(1) and operate directly on the integer's binary representation. They appear in competitive programming and performance-critical systems code.

```java
// Check if bit k is set
boolean isSet(int n, int k) { return (n & (1 << k)) != 0; }

// Set bit k
int set(int n, int k) { return n | (1 << k); }

// Clear bit k
int clear(int n, int k) { return n & ~(1 << k); }

// Toggle bit k
int toggle(int n, int k) { return n ^ (1 << k); }

// Check if n is a power of 2
boolean isPow2(int n) { return n > 0 && (n & (n-1)) == 0; }

// Count set bits (Brian Kernighan)
int popcount(int n) {
    int count = 0;
    while (n != 0) { n &= (n-1); count++; }
    return count;
}
```

- `n & (n-1)` clears the lowest set bit of `n`. This is why the Kernighan popcount loop runs in O(set bits) rather than O(32).

- **Bitmask DP** — when the state space can be represented as a subset of a small set (n ≤ 20), represent each subset as an integer bitmask. The number of subsets is 2^n, and each fits in a 32-bit integer. Iterating over subsets of a bitmask `mask` is `for (int sub = mask; sub > 0; sub = (sub-1) & mask)`. Classic application: Travelling Salesman Problem with DP over subsets of visited cities — O(n² × 2^n), feasible for n ≤ 20.

---

## Developer Recommendations

- **Know your data structures' time complexities cold**
  - Choosing a `LinkedList` when you need O(1) random access results in O(n) production performance.
  - Memorize the Big O table for Array, List, HashMap, TreeSet, PriorityQueue, and HashSet.
  - The underlying memory model matters as much as the asymptotic bound — cache-friendly structures consistently outperform theoretically equivalent ones at scale.
  - **Production story:** A team building an in-memory message buffer used a `LinkedList` for 10K elements and polled by index every 100ms — the O(n) access on each poll turned a 1μs operation into 50μs, which cascaded into thread-pool starvation under 500 concurrent consumers because the CPU spent all its time walking pointers instead of processing messages.

- **Start with brute force, then optimize**
  - Get a working solution first, even at O(n²).
  - Then apply the BUD framework: find Bottlenecks (the slowest step), eliminate Unnecessary work (redundant computation), and remove Duplicated work (overlapping subproblems).
  - Premature optimization before the brute force is correct leads to buggy, hard-to-debug solutions.
  - **Production story:** A team once spent a week implementing a concurrent multi-level cache to avoid O(n²) in a report generator, only to discover the real bottleneck was a forgotten debug log that wrote 10MB per report on a throttled disk — the O(n²) loop processed 200 records.

- **Use hash-based structures for lookup-heavy problems**
  - If your algorithm repeatedly calls `contains()` or `indexOf()` on a list, insert elements into a `HashSet` or `HashMap` first.
  - The O(n) memory cost to build the set is almost always worth the O(1) per-lookup improvement.
  - A common failure: a nested loop checking whether any element of list A exists in list B was O(n × m) and was "fast enough" during development with 50-element lists — in staging with 10K elements each, it took 45 seconds and caused a connection timeout on the API gateway above it.

- **Prefer iterative over recursive for production code**
  - Recursion is elegant for tree and divide-and-conquer problems but risks stack overflow for deep structures (thousands of levels).
  - Convert to iterative using an explicit `Deque` as the stack. The logic is identical — you are just managing the stack yourself rather than relying on the call stack.
  - **Production story:** A production outage: a directory-walking service used recursive DFS to compute disk usage. On a deeply nested auto-generated directory (a test harness produced 4000 levels of nesting), every worker thread hit a `StackOverflowError` simultaneously, bringing the service down — and because the error was uncaught in the thread pool, the outage went undetected for hours.

- **Benchmark before optimizing**
  - O(n log n) may outperform O(n) for small n due to constants and cache effects.
  - Quicksort often beats merge sort in practice because its in-place access pattern is more cache-friendly despite the same asymptotic bound.
  - Profile before rewriting.

- **Practice the sliding window pattern**
  - Many subarray and substring problems reduce to a single template: expand the right pointer, contract the left pointer when the window violates a condition.
  - The key invariant is that the window is always valid after contraction.
  - Recognizing this eliminates the need for nested loops.

- **Test with edge cases before considering done**
  - Empty input, single element, all duplicates, negative numbers, integer overflow, null values.
  - Most DSA bugs live at boundaries, not in the core logic.
  - Write assertions for these cases during practice.

- **Use divide and conquer for problems that decompose cleanly**
  - When a problem on n elements can be split into two independent subproblems on n/2 elements and the combination step is O(n) or cheaper, the total complexity is O(n log n) by the Master Theorem.
  - Merge sort, quicksort, binary search, closest pair of points, and many tree algorithms follow this pattern.
