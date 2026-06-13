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

## Interview Questions

- **Explain cache-aside vs read-through vs write-through vs write-behind.**
  - Cache-aside: app manages cache (check, load on miss, populate). Read-through: cache auto-loads on miss. Write-through: synchronous dual write to cache and DB. Write-behind: async write to DB after cache write.
- **How does cache stampede happen and how do you prevent it?**
  - A cache stampede occurs when a popular key expires and many requests simultaneously miss the cache and hit the DB. Prevention: mutex locking on cache miss (only one reloads), early recomputation before TTL expiry, probabilistic early expiration, or serving stale data while asynchronously refreshing.
- **Compare LRU, LFU, and TTL eviction policies.**
  - LRU: removes least recently accessed — good for temporal locality. LFU: removes least frequently accessed — good for stable popularity. TTL: removes by time regardless of access — ensures freshness. Many systems combine TTL (for freshness) with LRU/LFU (for memory management).
- **What are the tradeoffs between Redis Cluster and Memcached?**
  - Redis Cluster offers sharding, replication, failover, data structures (lists, sets, sorted sets), and persistence. Memcached is simpler, multithreaded, with lower per-request overhead for simple key-value operations. Redis is better for feature-rich caching; Memcached is better for high-throughput, simple caching with minimal operational complexity.

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
