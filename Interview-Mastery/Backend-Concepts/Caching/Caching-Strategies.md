# Caching Strategies

---

## Overview

- **Definition:** Caching stores frequently accessed data in a temporary storage layer to serve future requests faster, reducing latency and load on the primary data store.
- **Why It Exists:** Databases are slow relative to in-memory access; caching exploits temporal and spatial locality to avoid repeated expensive operations.
- **Key Concepts:** **Cache Hit** (data found in cache), **Cache Miss** (data not found; fetch from origin), **TTL** (time-to-live expiration), **Eviction Policy** (LRU, LFU, FIFO, TinyLFU), **Cache Invalidation** (removing stale entries), **Cache Stampede** (thundering herd on expiration).
- **Caching by Data Type** — Static data (CSS, JS, images) can be cached indefinitely with content hashing. Session data should use short TTLs (15-30 minutes) with sliding expiration. Database query results benefit from cache-aside with TTLs based on data volatility. API responses can be cached at the CDN level with stale-while-revalidate for dynamic content.
- **Cache Consistency Models** — Strong consistency (write-through: every write goes to cache and DB synchronously, higher latency), Eventual consistency (TTL-based: stale data served until expiry, simple), and Read-your-writes (session-level consistency: user always sees their own writes immediately). Choose based on business requirements.

---

## Core Strategies

- **Cache-Aside (Lazy Loading):** App checks cache → miss → loads from DB → stores in cache. Writes go to DB then invalidate cache. Simple and resilient; cache miss penalty on first read.
- **Read-Through:** Cache auto-loads from DB on miss. App talks only to cache. Simplifies app code but cache failure causes downtime.
- **Write-Through:** Data written to cache and DB synchronously. Strong consistency, higher write latency.
- **Write-Behind (Write-Back):** Data written to cache immediately, asynchronously persisted to DB. Very low write latency, risk of data loss on cache failure.
- **Refresh-Ahead:** Cache proactively refreshes entries before TTL expiry. Reduces misses for popular keys, wastes resources on unused entries.
- **Cache Penetration** — Requests for non-existent keys always miss the cache and hit the database. Mitigation: cache negative results (short TTL of 30-60s), use a Bloom filter to check key existence before cache lookup, or validate input parameters before cache access.
- **Cache Breakdown (Hot Key)** — A single key receives so many requests that the cache node handling it becomes saturated. Mitigation: local L1 cache on each application instance, replicate the hot key across multiple cache nodes (key:0, key:1), or use read replicas to distribute load.

```java
// Cache-Aside with RedisTemplate
public Product getProduct(String productId) {
    String key = "product:" + productId;
    Product cached = (Product) redisTemplate.opsForValue().get(key);
    if (cached != null) return cached;
    Product product = productRepository.findById(productId).orElseThrow();
    redisTemplate.opsForValue().set(key, product, 3600, TimeUnit.SECONDS);
    return product;
}
```

---

## Common Mistakes

- **Using Cache as Source of Truth** — DB must be authoritative; cache is an optimization.
  - **Why it looks correct:** The cache returns correct data during normal operation, so it feels like the cache can serve as the primary source — the divergence between cache and DB only surfaces when a cache node fails or an entry is evicted, causing silent data loss.
- **Cache Stampede (Thundering Herd)** — Multiple concurrent requests hit DB on expiry. Fix with distributed locks or stale-while-revalidate.
  - **Why it looks correct:** Setting a fixed TTL and letting each request fetch independently seems like the simplest approach — the stampede only becomes visible when a popular cache key's simultaneous expiry causes a 10x database CPU spike.
- **Infinite TTL** — Stale data lives forever. Always set TTL unless data is immutable.
  - **Why it looks correct:** The data rarely changes, so no expiry avoids the overhead of repopulating the cache — stale data accumulates silently until a user reports seeing outdated information that was changed days ago.
- **Caching Everything** — Cache only frequently-read, infrequently-written data.
  - **Why it looks correct:** Caching can only improve performance, so more caching seems strictly better — the memory and invalidation complexity of caching rarely-accessed data only becomes apparent in the Redis EC2 bill for data that is never read again.
- **No Cache Key Namespacing** — Without prefixes like `product:v2:` or `user:{tenantId}:`, cache keys collide across environments or tenants. Use consistent key naming with version and tenant prefixes.
- **Ignoring Serialization Overhead** — Storing complex objects in cache requires serialization/deserialization. Java serialization is slow; use JSON, Protocol Buffers, or Kryo for faster serialization. Measure the serialization cost as part of cache miss latency.
  - **Why it looks correct:** The cache stores and retrieves objects correctly regardless of serialization method, so the mechanism seems like an implementation detail — the CPU cost of Java serialization only becomes visible when profiling shows 40% of cache miss time is spent in ObjectInputStream.
- **Cache Warming on Every Deployment** — After a deployment, all L1 caches are cold, causing a thundering herd on Redis. Warm caches before accepting traffic using startup probes and pre-load scripts.
  - **Why it looks correct:** Application restarts are normal operations and caches should warm up naturally through regular traffic — the thundering herd of cold starts only becomes a problem when Kubernetes rolling restarts all pods simultaneously during a deployment.

---

## Key Design Considerations

- **Multi-Tier Caching** — L1 (local Caffeine, sub-ms) → L2 (Redis cluster, 1-5ms) → L3 (DB). L1 TTL shorter than L2.
- **Cache Hit Ratio** — Target >95% for read-heavy workloads. Monitor per region.
- **Eviction Policy Selection** — LRU for general purpose, LFU for stable popularity, TinyLFU for high hit ratio.
- **Cache Warm-Up** — Pre-load popular entries on startup via `CommandLineRunner` with Redis pipelining.
- **Cache Invalidation Patterns** — TTL-based (simplest), event-driven (Kafka/RabbitMQ), CDC (change-data-capture), version-based.
- **Cache Poisoning** — Validate data before caching, sanitize keys, use tenant prefixes in multi-tenant systems.
- **Geographic Cache Distribution** — For global applications, use a multi-region cache topology: write to local region's cache, replicate or invalidate across regions via a global event bus. Accept cross-region replication latency (typically 1-5 seconds). Use a global Redis Cluster or DynamoDB Accelerator (DAX) for active-active multi-region caching.
- **Cache Monitoring and Observability** — Track cache hit ratio (per key pattern), eviction rate (should be near zero), average load time (miss penalty), cache size vs capacity, and network latency to cache servers. Alert on hit ratio drops below 85% and eviction rate spikes. Use Redis `INFO stats` and Caffeine `.recordStats()` for detailed metrics.

---

## Real-World Scenarios

### Scenario 1: E-Commerce Product Catalog Caching
**Context:** An e-commerce site with 10M SKUs experiences 50ms database latency per product page. During flash sales, the database reaches 95% CPU, causing timeout errors. The most popular 1,000 SKUs receive 80% of traffic.

**Resolution:** Implement a multi-tier caching strategy. L1: Caffeine in-app cache (local, sub-ms, 10,000 entries, 5-minute TTL) for hot products. L2: Redis cluster (distributed, 1-5ms, all SKUs, 1-hour TTL). Cache-aside pattern: check L1 → miss → check L2 → miss → load from DB → write to L2 → write to L1. Write-through for inventory updates: update DB → invalidate L2 → broadcast invalidation to all instances for L1.

```java
@Component
public class ProductCacheService {
    private final Cache<String, Product> localCache;
    private final RedisTemplate<String, Object> redis;

    public ProductCacheService() {
        this.localCache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .recordStats()
            .build();
    }

    public Product getProduct(String sku) {
        Product cached = localCache.getIfPresent(sku);
        if (cached != null) return cached;

        String redisKey = "product:" + sku;
        cached = (Product) redis.opsForValue().get(redisKey);
        if (cached != null) {
            localCache.put(sku, cached);
            return cached;
        }

        Product product = productRepository.findBySku(sku);
        redis.opsForValue().set(redisKey, product, 1, TimeUnit.HOURS);
        localCache.put(sku, product);
        return product;
    }

    public void invalidateProduct(String sku) {
        localCache.invalidate(sku);
        redis.delete("product:" + sku);
    }
}
```

### Scenario 2: Social Media Feed with Cache Stampede
**Context:** A social media app caches user feeds for 60 seconds. When a popular user posts, the feed cache expires simultaneously for 100K followers, all hitting the database at once — a cache stampede that takes the DB down.

**Resolution:** Apply stale-while-revalidate and TTL jitter. Instead of a fixed 60s TTL, randomize between 45-75s. Use a distributed lock for cache regeneration: the first request to miss acquires a Redis lock (`SET product:feed:user123 NX EX 10`), fetches from DB, and repopulates the cache. Other requests either wait briefly or get the stale entry with a `warning: stale` header.

### Scenario 3: Session Cache Consistency
**Context:** A microservices application stores user sessions in Redis. When a user's role changes (e.g., upgraded to premium), the existing cached session still shows the old role. The user sees the wrong features.

**Resolution:** Use write-through cache invalidation. When the user service updates a user's role, it publishes a `UserRoleChanged` event. A cache service consumes the event and invalidates the session cache for that user. The next request loads the fresh session from the database. For critical consistency requirements, use a CQRS approach where writes go to both the database and cache synchronously.

---

## Scenario-Based Questions

1. **Q: You're building a product catalog with 10M SKUs. The database is struggling with read traffic during peak hours. How do you design a caching strategy that handles both hot products and long-tail items?**
   - A: Multi-tier caching. L1 (in-app Caffeine, 10K entries, 5 min TTL) for hot products — identified via access frequency tracking. L2 (Redis Cluster, 1 hour TTL) for all products. Cache-aside pattern. Use a Bloom filter for SKU existence checks to prevent cache penetration (queries for non-existent SKUs hitting the DB). Cache warm-up on deployment: pre-load top 10K SKUs from DB to Redis using pipelining. Monitor hit ratio per tier — target >95% combined.

  - **Interview follow-up:** Your Bloom filter for product SKU existence has a 1% false positive rate — 1% of phantom queries still reach the database. How do you handle the 1% false positive case without caching every SKU?

2. **Q: Your cache for user sessions expires, and 50K users simultaneously hit the login service. The database can't handle 50K concurrent reads. How do you prevent this cache stampede?**
   - A: Distributed lock on cache miss. When a cache miss occurs, the first request acquires a Redis lock (`SET session:lock:{userId} NX EX 5`). Only this request loads from DB and repopulates the cache. Other requests either wait briefly (spin with 10ms sleep) or get the stale session data (stale-while-revalidate). Add TTL jitter (±20% of the configured TTL) so sessions don't expire simultaneously.

3. **Q: You're using write-behind caching to reduce write latency. The cache node crashes before the batch is persisted. You lose 5 seconds of data. How do you mitigate this?**
   - A: Add a write-ahead log (WAL) before the cache write. The WAL is persisted to disk (or a sentinel database) before the cache accepts the write. On cache recovery, replay the WAL to the database. Alternatively, add redundancy — write to two cache nodes, and if one fails, the other has the data. Accept data loss only for non-critical data (analytics, counters). For critical data (orders, payments), use write-through or at least synchronous replication.

4. **Q: Your application uses cache-aside, but a deployment restarts all instances, clearing the L1 cache. The L2 Redis cache is fine, but traffic to Redis spikes 10x. How do you handle this?**
   - A: Cache warm-up on startup. Use a `CommandLineRunner` that loads the top 1,000 entries from Redis to the local Caffeine cache. Use Redis pipelining for efficient bulk loading. For stateless deployments, use a pre-warm script that loads cache before the instance accepts traffic. During warm-up, serve requests at slightly higher latency (Redis-only, no L1) until the L1 cache stabilizes.

  - **Interview follow-up:** Cache warm-up on startup loads the top 1,000 entries from Redis to L1, but the hot set changes over time — how do you detect when the warm-up set no longer matches current traffic patterns and refresh it?

5. **Q: You're caching API responses in Redis with a 5-minute TTL. The data changes every 30 seconds. Users see stale data. How do you balance consistency with caching?**
   - A: Reduce TTL to 15-30 seconds. Use write-through invalidation: on data update, publish a cache invalidation event via Redis pub/sub or Kafka. All instances subscribe and invalidate the relevant key. For critical data, skip caching entirely or use a very short TTL (5s). For user-facing dashboards, consider WebSocket push for real-time updates alongside a stale cache for initial load.

6. **Q: Your Redis cache has a 90% hit ratio but uses 12GB of memory. Instance restarts cause a 10-minute warm-up period with 50% hit ratio. How do you reduce the warm-up time?**
   - A: Enable Redis RDB persistence with frequent snapshots (every 5 minutes). On restart, Redis loads the RDB file, so the cache is mostly warm. For L1 Caffeine cache, use a persistent backup file. Additionally, use Redis replication: promote a replica first, so it already has the cached data. For zero-downtime cache recovery, use a blue-green Redis deployment.

7. **Q: A malicious client is sending requests for random non-existent product IDs (e.g., `/product/abc123`, `/product/xyz789`). Each miss hits the database. How do you prevent this cache penetration?**
   - A: Bloom filter. Maintain a Bloom filter containing all valid product IDs. Before checking Redis, check the Bloom filter. If the ID doesn't exist in the filter, return 404 immediately without hitting Redis or the DB. For IDs that pass the Bloom filter but don't exist in the DB (false positives), cache the "not found" result with a short TTL (60 seconds) to prevent repeated DB hits.

8. **Q: Your multi-region application has a cache in us-east-1 and eu-west-1. A product price change in us-east-1 creates stale cache in eu-west-1 for up to 1 hour. How do you handle cross-region cache invalidation?**
   - A: Use a global invalidation bus. Publish cache invalidation events to a cross-region Kafka topic. Each region subscribes and invalidates the affected keys. For latency-critical invalidation, use Redis' pub/sub across regions (less reliable but faster). Accept a small invalidation delay (1-5 seconds) for cross-region propagation.

  - **Interview follow-up:** Your cross-region invalidation bus uses Kafka with 2-second replication latency between regions — during a flash sale, a price update in us-east-1 takes 2+ seconds to reach eu-west-1, and users in Europe buy at the old price. How do you handle the accounting reconciliation for these cross-region pricing inconsistencies? For very strong consistency, use a global Redis cluster with active-active replication (Redis Enterprise or similar).

9. **Q: Your product catalog cache returns data including price. A flash sale changes thousands of prices simultaneously. How do you invalidate all affected cache entries without a stampede?**
   - A: Tag-based invalidation. Store cache entries with a tag (e.g., `category:electronics`, `sale:flash`). On price update, publish invalidation events with the tag. All entries with that tag are invalidated. To avoid stampede during re-caching, use a gradual re-cache strategy: invalidate 10% of entries per second rather than all at once. Combined with stale-while-revalidate, clients see stale prices for at most a few seconds.

10. **Q: You have a social media feed that aggregates posts from followed users. Caching the entire feed for each user is memory-prohibitive (10M users × 100KB each = 1TB). How do you cache efficiently?**
    - A: Cache individual posts, not aggregated feeds. Each post is cached in Redis with a 1-hour TTL. For feed rendering, fetch the user's follow list, batch-load post IDs, then batch-load posts from cache (pipeline Redis MGET). For the heavy users (verified accounts), pre-compute and cache their feeds with a 30-second TTL. For everyone else, compute on-the-fly from cached posts. LRU eviction in Redis naturally keeps hot posts available.

---

## Interview Questions

1. **What is caching and why is it used?**
   - A: Caching stores frequently accessed data in a high-speed storage layer to reduce latency (sub-millisecond vs 10-50ms DB), decrease database load, and improve throughput. It exploits temporal locality — recently accessed data is likely to be accessed again.

2. **Explain the difference between cache-aside and read-through.**
   - A: Cache-aside: application checks cache, on miss loads from DB and populates cache. Application manages both cache and DB. Read-through: cache auto-loads from DB on miss — application talks only to the cache. Read-through is simpler for the application but couples it to the cache provider.

3. **What is a cache stampede and how do you prevent it?**
   - A: A stampede occurs when many requests for the same key hit the DB simultaneously after cache expiry. Prevention: distributed locks (one request regenerates), stale-while-revalidate (serve stale data while refreshing), probabilistic early expiration, TTL jitter (±random% to desynchronize expirations).

4. **Compare write-through and write-behind caching.**
   - A: Write-through: synchronously writes to cache + DB — strong consistency, higher write latency. Write-behind: writes to cache immediately, asynchronously persists to DB — very low write latency, risk of data loss on cache failure. Choose write-through for critical data (orders), write-behind for analytics.

5. **How would you design caching for an e-commerce catalog with 10M SKUs?**
   - A: Redis Cluster with cache-aside. L1 Caffeine (10K hot entries). L2 Redis (all entries, 1-hour TTL). Bloom filter for non-existent keys. Cache warm-up on deploy. Eviction: LFU for Redis, LRU for Caffeine. Hit ratio target: >95%.

6. **What metrics do you monitor for cache performance?**
   - A: Cache hit ratio, eviction count/rate, cache size vs max capacity, average load time (miss penalty), network latency to Redis, stampede events, cache memory usage, and per-key access frequency.

7. **How do you handle cache consistency in a distributed system?**
   - A: Write-through for strong consistency (every write updates DB and cache). TTL-based invalidation for eventual consistency. Event-driven invalidation (Kafka/Redis pub/sub) for cross-instance coordination. For critical data, use a version field in the cache entry and compare with DB version.

8. **What is the difference between local cache and distributed cache?**
   - A: Local cache (Caffeine/Guava) in application memory — fastest (nanosecond), not shared across instances, lost on restart. Distributed cache (Redis) — shared across instances, millisecond latency, persists across restarts, larger capacity. Use both (L1 + L2) for optimal performance.

9. **How do you implement cache warm-up for a new deployment?**
   - A: Use `CommandLineRunner` to load top-K entries from DB to Redis via pipelining. For L1 cache, load from Redis on instance startup. Pre-warm before accepting traffic in Kubernetes by using startup probes. Gradual warm-up to avoid DB overload.

10. **What eviction policy would you choose for a read-heavy social media feed?**
    - A: TinyLFU or LFU — hot posts (viral content, trending topics) stay in cache. Combined with TTL-based expiry for freshness. For multi-tier: L1 Caffeine (LRU) + L2 Redis (LFU/LRU). The eviction policy should match the access pattern — LFU for stable popularity, LRU for recency-based access.

---

## Developer Recommendations

- **Use multi-tier caching (L1 + L2) for high-traffic systems** — A single Redis cache adds network latency (1-5ms) per request. Adding a local Caffeine cache (nanosecond access) reduces Redis load by 80% for hot keys. Configure L1 with shorter TTL than L2 and invalidate L1 entries via Redis pub/sub. Monitor both tiers' hit ratios separately — L1 target >50%, L2 target >95%.
  - **Production story:** A gaming platform used a single Redis cache without L1 and saw 5ms average latency — after adding Caffeine L1, hot key latency dropped to 0.1ms and Redis CPU dropped from 70% to 20%.

- **Always set TTL on cached entries** — Without TTL, cached data lives forever, causing stale data and memory leaks. The only exception is truly immutable data (country codes, historical reference data). For mutable data, TTL should be based on how stale data is acceptable — user profiles (5 min), product prices (30s), session data (TTL = session expiry).

- **Use Bloom filters to prevent cache penetration** — Cache penetration (queries for non-existent keys hitting the DB) is a common attack vector and performance issue. A Bloom filter with 1% false positive rate uses ~10 bits per entry. For 10M valid products, a 12MB Bloom filter prevents 99% of phantom queries from reaching the DB. Cache "not found" results with short TTL for false positives.

- **Handle cache stampede with distributed locks or stale-while-revalidate** — A stampede can take down your database when a popular cache key expires. Stale-while-revalidate is the simplest: serve the stale entry while asynchronously refreshing. For stronger consistency, use a distributed lock — only one request regenerates the cache; others wait or get stale data.

- **Monitor cache hit ratio as a critical SLO** — Cache hit ratio tells you if your caching strategy is working. Alert if the ratio drops below 90% (or your target). A sudden drop indicates a configuration issue, a deployment cleared the cache, or a code change altered the cache key pattern. Track per-cache-region ratios to identify specific problem areas.
  - **Production story:** A news site's cache hit ratio dropped from 97% to 60% overnight — it took 3 days to discover that a new deployment had changed the cache key prefix from `article:v1:` to `article:v2:` without migrating the old cached entries, effectively creating a cold cache for all article traffic.

- **Use consistent cache key naming with version prefixes** — A cache key like `product:v2:{sku}` allows safe invalidation of all v2 keys when the data format changes. Include relevant dimensions in the key: tenant ID for multi-tenant systems, locale for internationalized content. Avoid excessively long keys (waste memory) — use hashed keys for long composite keys.
- **Cache negative results to prevent cache penetration** — When a query returns no data (e.g., non-existent product ID), cache the "not found" result with a short TTL (30-60 seconds). This prevents repeated database lookups for the same invalid key. Without negative caching, an attack cycling through random IDs would bypass your cache entirely and overload the database.
- **Use local (L1) caches to protect Redis from hot keys** — A single hot key in Redis can saturate a CPU core on the Redis node, degrading all traffic to that node. Adding a local Caffeine cache in each application instance absorbs 90%+ of reads for the hot key. The local cache has a shorter TTL than Redis to stay fresh. This pattern is essential for viral content, trending products, and flash sale items.
