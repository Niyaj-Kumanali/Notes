# Design Distributed Cache

## Overview

- **Definition** — A cache system where data is distributed across multiple nodes to provide low-latency data access, reducing load on primary databases
- **Why It Exists** — Databases cannot serve every read at microsecond latency under high throughput; caching reduces database load, improves response times, and handles traffic spikes
- **Historical Context** — Memcached (2003) popularized distributed caching; Redis (2009) added data structures and persistence; Redis Cluster (2015) enabled native sharding and failover
- **Key Concepts** — **Cache-aside** is the most common pattern (app checks cache, loads on miss); **Eviction policies** (LRU, LFU, TTL) determine what to remove when memory fills; **Sharding** splits data across nodes; **Replication** copies data for redundancy; **Consistent hashing** distributes keys with minimal redistribution on scaling

## Core Concepts

- Cache-aside: application checks the cache first
  - On cache hit: return cached value
  - On cache miss: load from database, populate cache, return value
  - Cache population is the application's responsibility
- Read-through: cache automatically loads missing data from the database on cache miss
  - Cache is responsible for loading, not the application
  - Simplifies application code but requires the cache to have a DB connector
- Write-through: write to cache and database synchronously
  - Ensures cache and DB are always consistent
  - Higher write latency due to synchronous dual writes
- Write-behind: write to cache immediately, asynchronously write to database
  - Lower write latency
  - Risk of data loss if cache fails before the async write completes
  - Use for high-throughput write workloads where some data loss is acceptable
- Eviction policies
  - LRU (Least Recently Used): removes the least recently accessed items — good for general workloads with temporal locality
  - LFU (Least Frequently Used): removes items accessed least often — good for workloads with stable popularity distribution
  - TTL (Time-To-Live): removes items after a configurable time regardless of access pattern — ensures data freshness
  - FIFO (First In, First Out): removes items in the order they were added — simple but often suboptimal
- Sharding splits data across multiple cache nodes
  - Each node holds a subset of the data
  - Increases total memory capacity but adds complexity for multi-key operations
  - Consistent hashing distributes keys across shards with minimal redistribution on scaling
- Replication copies all data to every node
  - Provides read redundancy and failover capability
  - Writes must be propagated to all replicas — increases write overhead
- Redis Cluster supports sharding, replication, and automatic failover
  - 16384 hash slots distributed across nodes
  - Master-replica per shard for high availability
  - Automatic failover when a master is unreachable
- Memcached is simpler, multithreaded, but lacks replication and persistence
  - Pure key-value store with no data structures
  - Good for simple caching with high throughput
  - No built-in clustering — requires client-side consistent hashing
- Stale reads occur when the database is updated but the cache is not yet invalidated
  - Acceptable for read-heavy workloads where eventual consistency is tolerable
  - Mitigated by short TTLs or explicit cache invalidation on write
- Write conflicts happen in write-behind when multiple writes target the same key
  - Last-write-wins is the most common resolution strategy
  - Can cause data loss if intermediate values are important

## Common Mistakes

- **Cache stampede (thundering herd)**
  - When a popular cache key expires, multiple concurrent requests all miss the cache and simultaneously query the database, overwhelming it
  - **Why it looks correct:** The standard cache-aside pattern naturally reloads on cache miss
  - Use mutex locking for cache misses: only one request loads the data, others wait or serve stale data. Implement "early recompute" — refresh the cache before TTL expires when the key is heavily accessed. Use "probabilistic early expiration" — randomly expire keys before TTL to spread reloads.
- **Using cache as the primary data source**
  - Storing critical data only in cache and losing it on node failure
  - **Why it looks correct:** Cache reads are fast, and storing data there seems convenient
  - Always persist data to a database; cache is a performance layer, not a storage layer. Enable Redis persistence (RDB/AOF) if cache data must survive restarts.
- **Ignoring serialization overhead**
  - Using slow serialization (JSON for large objects, Java native serialization) that negates cache performance gains
  - **Why it looks correct:** JSON is human-readable and easy to debug
  - Use binary serialization (Protobuf, MessagePack, or language-specific optimized formats). Benchmark serialization time — it should be under 1ms for typical object sizes.

## Real-World Scenarios

### Redis Cluster for Session Store

- Session data stored as hashes with TTL
- Sharding across nodes for horizontal scaling
- Replication provides failover — if a master fails, a replica is promoted
- Consistent hashing via hash slots minimizes rehashing on cluster resizing

### Memcached for Database Query Cache

- Cache-aside pattern: cache query results with a TTL of a few minutes
- LRU eviction handles memory pressure
- Client-side consistent hashing distributes keys across Memcached nodes
- No persistence — a node restart means cold cache, mitigated by gradual warmup

### Write-Through Cache for Product Catalog

- Writes go to cache and database synchronously
- Ensures the cache is always fresh
- Acceptable because the catalog write rate is low compared to reads

## Use Cases

- **Database query result cache** — caching expensive SQL queries (user feeds, aggregated reports, product searches)
  - Cache-aside pattern: check cache, miss, query DB, populate cache. TTL based on data freshness requirements. Reduces database load by 80–95%.
  - **Avoid when:** query results change on every execution (e.g., `NOW()`, random values) — caching is ineffective for non-deterministic queries.

- **Session store** — sharing user session state across multiple application server instances
  - Distributed cache holds session data with TTL-based expiration. Any server can serve any user. No sticky session requirement.
  - **Avoid when:** session data is large (>1 MB) — consider storing session data in a database and using the cache only for active session indexes.

- **Rate limiter state** — tracking request counts per user/IP across multiple application instances
  - Atomic increment operations in Redis provide consistent distributed rate limiting without a single point of contention.
  - **Avoid when:** rate limit granularity is coarse (e.g., hourly limits) — local in-memory counters with periodic sync reduce Redis load.

- **Real-time leaderboards and counters** — gaming scores, trending hashtags, or live poll results
  - Sorted sets provide O(log N) ranked queries. Atomic updates ensure consistent counting. Single-digit millisecond read performance.
  - **Avoid when:** data must survive cache node failures — persist to a database and use cache only for hot data.

- **Distributed locking** — coordinating access to shared resources (file writes, job scheduling, inventory deduction)
  - Redlock algorithm provides distributed mutexes. Lock acquisition with TTL prevents deadlocks from crashed holders.
  - **Avoid when:** locks are held for more than a few seconds — lease-based mechanisms with heartbeats (like ZooKeeper) are more suitable for long-held locks.

## Scenario-Based Questions

**Q: A popular product page goes viral. The cache key expires and 10,000 concurrent requests all hit the database simultaneously, causing a 5-minute outage. How do you prevent this?**

- Implement cache stampede protection: use a mutex lock on cache miss — only one thread reloads from DB, others wait briefly or get a stale value
- Use early recompute: when TTL is nearly expired and the key is hot, refresh proactively
- Set a longer TTL with background refresh to keep popular data always cached
- **Interview follow-up:** How do you detect which keys are "hot" without exhaustively scanning all cache entries?

**Q: Your cache cluster runs out of memory every evening when traffic spikes. How do you handle this?**

- Review eviction policy: is LRU configured? LFU might be better if a subset of keys accounts for most traffic
- Add more cache nodes to increase total memory (sharding)
- Reduce TTLs so that stale data is evicted faster
- Implement data compression for large values (Snappy, LZ4)
- **Interview follow-up:** How do you distinguish between a cache that needs more memory and one that has a memory leak from unscoped caching?

**Q: Your cache-aside implementation causes high latency on cache misses because every miss queries the database synchronously. How do you improve this?**

- Use a read-through cache that automatically loads data from the database on miss, reducing application complexity
- Pre-warm the cache with popular data before traffic arrives, so misses are rare during peak hours
- Implement batch loading: when a miss occurs, check if other nearby keys are also missing and load them together
- **Interview follow-up:** How do you determine which keys to pre-warm without prior knowledge of access patterns?

**Q: Your write-through cache adds 10ms latency to every write because it synchronously writes to both cache and database. How do you reduce write latency?**

- Switch to write-behind: write to cache immediately, and asynchronously flush to the database in batches
- If immediate consistency is required, use a distributed transaction with a commit log that replicates asynchronously
- Consider whether write-through is actually needed — if reads tolerate staleness, use cache-aside with TTL instead
- **Interview follow-up:** How do you handle data loss risk with write-behind if the cache node fails before the async write completes?

**Q: Your Redis cache stores user sessions. When a cache node fails, all users assigned to that node are logged out. How do you improve resilience?**

- Enable Redis replication: each master has one or more replicas. If the master fails, a replica is promoted automatically
- Use Redis Cluster with replication factor 2 — each hash slot is replicated to a replica node in a different availability zone
- Implement client-side session fallback: if Redis is unavailable, fall back to a database-backed session store with a warning
- **Interview follow-up:** How do you handle the case where the promoted replica has stale data because replication lag was high?

**Q: Your cache eviction policy is LRU, but some keys that are accessed rarely but are expensive to compute keep getting evicted. How do you fix this?**

- Use LFU instead of LRU — LFU tracks access frequency, so expensive-but-rarely-accessed keys are not evicted quickly
- Assign a higher "cost" value to expensive keys and use a custom eviction policy that considers both access recency and recomputation cost
- Add a "protected" segment to the cache — a small portion of memory where keys are never evicted (pinned cache)
- **Interview follow-up:** How do you identify which keys are expensive to recompute without manual annotation?

**Q:** Your cache stores JSON objects that are 100KB each. Serialization and deserialization overhead adds 50ms to each cache operation. How do you optimize this?**

- Switch to a binary serialization format like Protobuf or MessagePack — these are faster and produce smaller payloads
- Compress large values using Snappy or LZ4 before storing in Redis — decompression is fast and reduces memory usage
- Consider splitting large objects into smaller, frequently accessed fields and cache them separately
- **Interview follow-up:** How does compression affect Redis memory fragmentation and what metrics should you monitor?

**Q: Your distributed cache cluster has 10 nodes. When you add an 11th node, you expect 10% fewer keys per node, but instead see 30% of keys redistributed. Why?**

- You are likely using a simple mod-based sharding (hash(key) % 10 → hash(key) % 11), which changes the mapping for almost all keys
- Fix: use consistent hashing so that adding a node redistributes only K/N keys (K = total keys, N = number of nodes)
- If consistent hashing is already in use, check the number of virtual nodes — too few vnodes can cause uneven redistribution
- **Interview follow-up:** How does client-side consistent hashing handle the transitional period when some clients still use the old node list?

**Q: Your cache hit rate dropped from 90% to 60% after a deployment. What could have caused this and how do you investigate?**

- The deployment may have changed the cache key format — if keys are constructed differently, existing cache entries are not found
- Application code changes may have altered query patterns, accessing different data than before
- Check if the deployment restarted cache nodes (e.g., if cache is embedded in the application process rather than external Redis)
- **Interview follow-up:** How do you implement a cache key schema versioning strategy to prevent key changes from invalidating the entire cache?

**Q: Your application uses Redis for distributed locking with SETNX. During a network partition, two nodes acquire the same lock simultaneously. How do you fix split-brain locks?**

- Use Redlock algorithm: acquire the lock from a majority of Redis nodes (N/2 + 1) rather than a single node
- Add a unique token to each lock acquisition attempt so the resource owner can verify the lock is valid
- Use a fencing token: a monotonically increasing number that allows the resource to reject stale lock holders
- **Interview follow-up:** How does Redlock handle the case where a client acquires a lock but pauses (GC pause) long enough for the lock to expire?

## Interview Questions

- **Explain cache-aside vs read-through vs write-through vs write-behind.**
  - Cache-aside: app manages cache (check, load on miss, populate). Read-through: cache auto-loads on miss. Write-through: synchronous dual write to cache and DB. Write-behind: async write to DB after cache write.
- **How does cache stampede happen and how do you prevent it?**
  - A cache stampede occurs when a popular key expires and many requests simultaneously miss the cache and hit the DB. Prevention: mutex locking on cache miss (only one reloads), early recomputation before TTL expiry, probabilistic early expiration, or serving stale data while asynchronously refreshing.
- **Compare LRU, LFU, and TTL eviction policies.**
  - LRU: removes least recently accessed — good for temporal locality. LFU: removes least frequently accessed — good for stable popularity. TTL: removes by time regardless of access — ensures freshness. Many systems combine TTL (for freshness) with LRU/LFU (for memory management).
- **What are the tradeoffs between Redis Cluster and Memcached?**
  - Redis Cluster offers sharding, replication, failover, data structures (lists, sets, sorted sets), and persistence. Memcached is simpler, multithreaded, with lower per-request overhead for simple key-value operations. Redis is better for feature-rich caching; Memcached is better for high-throughput, simple caching with minimal operational complexity.
- **Explain the difference between "cache hit" and "cache miss" and why each matters.**
  - Cache hit: requested data is found in cache — fast response (sub-millisecond). Cache miss: data must be fetched from origin — slow response (10–100ms). The hit rate (ratio of hits to total requests) is the primary metric for cache effectiveness. High hit rates reduce database load and improve response times.
- **How do you handle cache invalidation when the underlying data changes?**
  - Strategies: TTL-based (data expires after a fixed time), event-driven (publish invalidation event when data changes), write-through (update cache on every write). For relational data, use a change data capture (CDC) pipeline to invalidate affected cache keys when the database changes.
- **What is "thundering herd" and how does it differ from "cache stampede"?**
  - Thundering herd: many clients simultaneously detect a cache miss and attempt to reload from the database. Cache stampede: a broader system failure caused by the thundering herd overwhelming the database. Thundering herd is the cause; cache stampede is the effect. Both terms are often used interchangeably.
- **How does Redis handle memory pressure when all memory is used?**
  - Redis uses eviction policies configured by maxmemory-policy: noeviction (returns errors on writes), allkeys-lru (evicts least recently used keys), volatile-lru (evicts LRU keys with TTL), allkeys-lfu (least frequently used), volatile-ttl (shortest TTL first). Choose based on whether you want Redis to manage memory or error.
- **How do you monitor cache health in production?**
  - Key metrics: hit rate, miss rate, eviction count, memory usage, latency (p50/p99/p999), number of connected clients, CPU usage. Alert on sudden drops in hit rate or spikes in evictions. Use Redis INFO, MONITOR, and SLOWLOG commands for deep diagnostics.
- **What is the "cache-aside" pattern and when would you choose it over "read-through"?**
  - Cache-aside: application code checks cache, loads from DB on miss, and populates cache. Read-through: the cache library handles miss loading automatically. Cache-aside gives the application more control (custom serialization, business logic on miss). Read-through simplifies code and centralizes loading logic.
- **Explain the concept of "write-behind" caching and its risk.**
  - Write-behind: application writes to cache immediately; the cache asynchronously writes to the database. Risk: if the cache node fails before the async write, the data is lost. Mitigations: replicate cache writes, enable Redis persistence (AOF), or accept the risk for non-critical data only.
- **How do you implement a cache warming strategy?**
  - Before going live with a new deployment, replay recent traffic logs against the cache to populate hot keys. Alternatively, seed the cache from the database with the most frequently accessed records (based on access logs). Warm gradually to avoid overwhelming the database.
- **What is the role of "TTL jittering" in distributed caching?**
  - If many keys expire at the same time, the reload traffic creates a thundering herd. TTL jittering adds random variation (e.g., TTL = 300s + random(0, 60)s) so expires are spread over time. This prevents synchronized cache misses and reduces database load spikes.
- **How does Redis handle key expiration?**
  - Redis uses two mechanisms: passive (key is checked on access — if expired, it is removed) and active (Redis periodically samples a subset of keys with TTL and removes expired ones). The active expiration runs every 100ms and removes up to 20 keys per sampling cycle.
- **What is the difference between cache "invalidation" and cache "eviction"?**
  - Invalidation: proactively removing or updating cached data because the source data changed (application-initiated). Eviction: automatically removing data when the cache runs out of memory (system-initiated). Invalidation ensures freshness; eviction manages memory.
- **How do you handle caching of partial data (e.g., user profile without email field)?**
  - Cache the full object and mask sensitive fields at the application layer, or cache only the non-sensitive subset. For GDPR compliance, ensure that cached user data can be purged on request. Consider using separate caches for public and private data with different TTLs.
- **Explain the "client-side caching" pattern and when to use it.**
  - Store frequently accessed, rarely changed data in the application's local memory (e.g., configuration, feature flags). The client polls or subscribes to change notifications. Benefits: zero network latency for reads. Risk: stale data if notifications are missed or delayed.
- **How does Redis Cluster handle resharding without downtime?**
  - Redis Cluster supports online resharding: it moves hash slots from source nodes to target nodes while the cluster serves traffic. During migration, a slot's keys exist on both old and new nodes. The cluster tracks the migration state and redirects clients as needed (ASK redirect).
- **What is "cache concurrency" and how do you prevent race conditions on cache updates?**
  - When two requests simultaneously miss the cache and both try to populate it, they may write different values. Use a mutex: only one request loads from DB and writes to cache; others wait briefly or serve stale data. Use Redis SET NX with a lock key to coordinate concurrent cache population.
- **How does Redis persistence (RDB vs AOF) affect cache performance?**
  - RDB snapshots periodically dump the entire dataset to disk, causing potential latency spikes during snapshotting. AOF logs every write operation, providing better durability but higher disk I/O. For cache-only workloads, disable persistence entirely — rely on replication for failover. For mixed use, AOF with every-second fsync balances performance and durability.

## Developer Recommendations

- **Always protect against cache stampedes**
  - Use mutex locking or early recomputation for any key that can receive high traffic
  - Monitor cache miss rates per key — spikes indicate potential stampedes
  - **Production story:** A flash sale caused a 10x DB traffic spike when the product inventory cache expired — implementing a mutex lock on the cache miss path eliminated the stampede entirely
- **Set appropriate TTLs and monitor hit rates**
  - TTL too short: low hit rate, increased DB load. TTL too long: stale data, storage waste.
  - Start with TTL = 5 minutes and adjust based on data freshness requirements and hit-rate monitoring
- **Choose serialization format carefully**
  - Avoid JSON for large objects; prefer Protobuf, MessagePack, or Kryo
  - Profile serialization time and compressed size — the goal is <1ms serialization for 99th percentile objects
  - **Production story:** Switching from JSON to Protobuf reduced cache read latency by 40% and memory usage by 30% for a product catalog cache
- **Use consistent hashing for client-side sharding**
  - When using Memcached or client-side Redis sharding, consistent hashing ensures minimal key redistribution on node changes
  - Number of virtual nodes should be high enough to handle node weight differences
- **Implement cache warming for new deployments**
  - A cold cache after deployment or restart causing high DB load
  - Pre-warm critical keys (popular products, configs, user sessions) by replaying recent traffic or seeding from DB
