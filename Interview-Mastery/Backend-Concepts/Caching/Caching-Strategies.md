# Caching Strategies

## 1. Executive Summary

Caching is a technique that stores frequently accessed data in a temporary storage layer (cache) to serve future requests faster, reducing latency and load on the primary data store. Caching strategies define how data is written to and read from the cache, governing consistency, performance, and reliability trade-offs. Choosing the right strategy is critical for building scalable, high-performance backend systems.

## 2. Core Theory

Caching operates on the principle of temporal and spatial locality: recently or frequently accessed data is likely to be accessed again. A cache sits between the application and the persistent storage, intercepting reads and optionally writes.

**Key Concepts:**
- Cache Hit: Requested data found in cache.
- Cache Miss: Requested data not found; must fetch from origin.
- Cache Invalidation: Removing or updating stale cache entries.
- Time-To-Live (TTL): Expiration duration for cache entries.
- Eviction Policy: Algorithm to decide which entries to remove when cache is full (LRU, LFU, FIFO, TTL).

**Common Caching Layers:**
- In-memory cache (Caffeine, Guava, Redis)
- Distributed cache (Redis Cluster, Hazelcast)
- HTTP cache (reverse proxy, CDN)
- Database query cache
- Client-side cache

## 3. Under-the-Hood Deep Dive

### Cache-Aside (Lazy Loading)
Application code explicitly loads data into cache on miss. The cache does not communicate with the database directly.

```
Read:  Check cache -> Miss -> Load from DB -> Store in cache -> Return
Write: Write to DB -> Invalidate cache entry
```

**Pros:** Simple, resilient to cache failures, data only cached when needed.
**Cons:** Cache miss penalty (read-through latency), stale data possible if invalidation fails.

### Read-Through
Cache layer automatically loads data from the database on miss. Application talks only to the cache.

```
Read:  Check cache -> Miss -> Cache loads from DB -> Return
Write: Write directly to DB (or cache + DB)
```

**Pros:** Simplifies application code, cache is authoritative.
**Cons:** Cache failure can cause downtime, higher complexity in cache layer.

### Write-Through
Data is written to cache and database simultaneously in the same transaction. Every write goes through the cache.

```
Write: Write to cache -> Cache writes to DB -> Confirm
Read:  Read from cache (always fresh)
```

**Pros:** Strong consistency between cache and DB, no stale reads.
**Cons:** Higher write latency, unnecessary writes for data not read frequently.

### Write-Behind (Write-Back)
Data is written to cache immediately, then asynchronously persisted to database.

```
Write: Write to cache -> Acknowledge -> Async write to DB later
Read:  Read from cache (may be dirty)
```

**Pros:** Very low write latency, write absorption (batching).
**Cons:** Risk of data loss if cache fails before persistence, complex recovery.

### Refresh-Ahead
Cache proactively refreshes entries before they expire, based on prediction.

```
Before TTL expires: Cache predicts access -> Refresh from DB -> Extend TTL
```

**Pros:** Reduces cache misses for popular entries.
**Cons:** Wastes resources on entries that may not be accessed, prediction overhead.

## 4. Production Code Examples

### Spring Boot Cache-Aside with @Cacheable

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Cacheable(value = "users", key = "#userId", unless = "#result == null")
    public User getUserById(Long userId) {
        // This method executes only on cache miss
        return userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
    }

    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        // @CachePut always executes and updates the cache
        return userRepository.save(user);
    }

    @CacheEvict(value = "users", key = "#userId")
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }
}
```

### Manual Cache-Aside with RedisTemplate

```java
@Service
public class ProductService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private ProductRepository productRepository;

    private static final String PRODUCT_CACHE_PREFIX = "product:";
    private static final long TTL_SECONDS = 3600;

    public Product getProduct(String productId) {
        String cacheKey = PRODUCT_CACHE_PREFIX + productId;

        // Check cache
        Product cached = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached; // Cache hit
        }

        // Cache miss - load from DB
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found"));

        // Store in cache with TTL
        redisTemplate.opsForValue().set(cacheKey, product, TTL_SECONDS, TimeUnit.SECONDS);
        return product;
    }

    public void updateProduct(Product product) {
        productRepository.save(product);
        // Invalidate cache
        redisTemplate.delete(PRODUCT_CACHE_PREFIX + product.getId());
    }
}
```

### Write-Through with Caffeine Cache

```java
@Configuration
public class CacheConfig {

    @Bean
    public Cache<String, Order> orderCache() {
        return Caffeine.newBuilder()
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .maximumSize(10_000)
            .recordStats()
            .build(key -> loadOrderFromDb(key)); // Read-through
    }

    private Order loadOrderFromDb(String orderId) {
        // Called automatically on cache miss
        return orderRepository.findById(orderId).orElse(null);
    }
}

@Service
public class OrderService {

    @Autowired
    private Cache<String, Order> orderCache;

    @Autowired
    private OrderRepository orderRepository;

    public Order getOrder(String orderId) {
        return orderCache.get(orderId, this::loadOrderFromDb);
    }

    public Order createOrder(Order order) {
        Order saved = orderRepository.save(order);
        orderCache.put(saved.getId(), saved); // Write-through
        return saved;
    }

    private Order loadOrderFromDb(String orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
    }
}
```

### Write-Behind with Spring @Async and Redis

```java
@Service
public class AnalyticsCacheService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private AnalyticsRepository analyticsRepository;

    private static final String ANALYTICS_QUEUE = "analytics:queue";
    private static final long FLUSH_INTERVAL_MS = 5000;

    @Scheduled(fixedDelay = FLUSH_INTERVAL_MS)
    @Async
    public void flushAnalytics() {
        // Batch write all pending analytics from cache to DB
        List<AnalyticsEvent> batch = new ArrayList<>();
        while (batch.size() < 100) {
            AnalyticsEvent event = (AnalyticsEvent) redisTemplate
                .opsForList().leftPop(ANALYTICS_QUEUE);
            if (event == null) break;
            batch.add(event);
        }
        if (!batch.isEmpty()) {
            analyticsRepository.saveAll(batch);
        }
    }

    public void trackEvent(AnalyticsEvent event) {
        // Immediate cache write, deferred DB write
        redisTemplate.opsForList().rightPush(ANALYTICS_QUEUE, event);
    }
}
```

## 5. Real-World Scenarios

### E-Commerce Product Catalog
**Strategy:** Cache-Aside with Redis cluster + local L1 cache (Caffeine)
- Product detail pages: Cache-Aside with 1-hour TTL
- Inventory counts: Write-Through for accuracy
- Search results: Cache-Aside with 5-minute TTL
- Cart data: Write-Behind for fast add/remove operations

### Social Media Feed
**Strategy:** Multi-tier caching with refresh-ahead
- L1: Local Caffeine cache (10ms, 10K entries)
- L2: Redis cluster (50ms, 1M entries)
- L3: Database (500ms)
- Popular feeds: Refresh-ahead 30 seconds before TTL expiry
- Cold start: Pre-warm cache with top 1000 most-followed users

### Payment Processing
**Strategy:** Write-Through with strong consistency
- Account balances: Write-Through to prevent double-spend
- Transaction idempotency: Cache-Aside with 24-hour TTL
- Rate limiting: Write-Behind with Redis sorted sets (sliding window)

## 6. Performance

### Cache Hit Ratio Optimization
- Monitor cache hit ratio; target >95% for read-heavy workloads.
- Right-size TTL based on data volatility.
- Use consistent hashing to minimize cache redistribution.

### Latency Impact
- In-memory cache: <1ms hit, 10-50ms miss (with DB fetch)
- Redis cache: 1-5ms hit (network round-trip), 50-100ms miss
- CDN cache: 1-10ms hit (edge), 200-500ms miss (origin fetch)

### Eviction Policy Selection
| Policy | Use Case | Overhead |
|--------|----------|----------|
| LRU    | General purpose | Low |
| LFU    | Hot data persists | Medium |
| FIFO   | Simple streaming | Low |
| TTL    | Time-bound data | None |
| TinyLFU| High hit ratio | Medium |

### Write-Behind Batch Sizing
- Smaller batches (10-50): Lower latency, higher DB write frequency.
- Larger batches (100-1000): Higher throughput, risk of data loss.
- Adaptive: Adjust batch size based on queue depth and DB load.

## 7. Security

### Cache Poisoning
Attackers inject malicious data into cache by exploiting validation gaps.
- Validate all data before caching.
- Sanitize cache keys to prevent key collision attacks.
- Use input hashing for cache keys derived from user input.

### Cache Key Separation
```java
public class SecureCacheKeyGenerator {

    public static String generateKey(String tenantId, String userId, String resourceId) {
        // Prefix with tenant to prevent cross-tenant cache leaks
        return String.format("tenant:%s:user:%s:res:%s",
            sanitize(tenantId), sanitize(userId), sanitize(resourceId));
    }

    private static String sanitize(String input) {
        // Remove characters that could cause key collision
        return input.replaceAll("[:\\s]", "_");
    }
}
```

### Sensitive Data in Cache
- Never cache PII, passwords, or payment data without encryption.
- Encrypt cache values for sensitive data using AES-256.
- Set aggressive TTL for sensitive cached data.
- Use separate Redis instances for sensitive vs. non-sensitive data.

### Cache Busting Attacks
Repeated cache misses on purpose (cache flooding) can overwhelm origin.
- Rate-limit cache misses per user/IP.
- Use Bloom filters to detect and block non-existent key lookups.
- Implement circuit breakers for cache miss fallbacks.

## 8. Common Mistakes

### Mistake 1: Using Cache as the Source of Truth
The database should always be the authoritative source. Cache is an optimization.
```java
// WRONG - data loss if cache fails
@CachePut("orders")
public Order saveOrder(Order order) {
    return order; // Not persisted to DB!
}

// RIGHT
public Order saveOrder(Order order) {
    Order saved = orderRepository.save(order);
    evictCache(saved.getId());
    return saved;
}
```

### Mistake 2: Cache Stampede (Thundering Herd)
Multiple concurrent requests for the same cache key all trigger DB queries on expiry.

Solution: Mutex-based cache reloading.
```java
public Product getProduct(String id) {
    String cacheKey = "product:" + id;
    Product product = (Product) redisTemplate.opsForValue().get(cacheKey);
    if (product != null) return product;

    // Acquire distributed lock
    String lockKey = "lock:" + cacheKey;
    Boolean locked = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "locked", Duration.ofSeconds(5));

    if (Boolean.TRUE.equals(locked)) {
        try {
            product = loadFromDb(id);
            redisTemplate.opsForValue().set(cacheKey, product, Duration.ofHours(1));
            return product;
        } finally {
            redisTemplate.delete(lockKey);
        }
    } else {
        // Wait and retry or degrade gracefully
        Thread.sleep(100);
        return getProduct(id); // Recursive retry
    }
}
```

### Mistake 3: Infinite TTL
Stale data lives forever. Always set TTL unless the data is truly immutable.

### Mistake 4: Caching Everything
Cache only what is read frequently and written infrequently. Profile access patterns first.

## 9. Senior Engineer Perspective

### Multi-Tier Caching Architecture
```
[Client] -> [CDN] -> [Load Balancer] -> [App Server L1 Cache] -> [Redis L2 Cache] -> [DB]
```

- L1 (local): Ultra-fast, per-node, small capacity. Use Caffeine.
- L2 (distributed): Shared across nodes, larger capacity. Use Redis.
- **Coordination**: L1 needs TTL shorter than L2. Use pub/sub to invalidate L1 across nodes.

### Cache Warm-Up Strategy
```java
@Component
public class CacheWarmer implements CommandLineRunner {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Override
    public void run(String... args) {
        // Pre-warm top 10,000 most-viewed products
        List<Product> hotProducts = productRepository
            .findTopViewed(10_000);

        hotProducts.parallelStream().forEach(product -> {
            redisTemplate.opsForValue().set(
                "product:" + product.getId(),
                product,
                Duration.ofHours(1)
            );
        });
    }
}
```

### Cache Invalidation Patterns
- **TTL-based**: Simplest, eventual consistency.
- **Event-driven**: Publish cache invalidation events (Kafka/RabbitMQ).
- **Database triggers**: Use change-data-capture (CDC) to invalidate cache on DB changes.
- **Version-based**: Increment a global version number; cache keys include version.

### Production Cache Metrics to Monitor
- Hit ratio per cache region
- Eviction count and rate
- Cache size vs. max capacity
- Average load time (miss penalty)
- Network latency to Redis cluster
- Cache stampede events

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What is caching and why is it used?
   **A:** Caching stores frequently accessed data in a temporary high-speed storage layer to reduce latency, decrease database load, and improve application throughput.

2. **Q:** What is the difference between cache hit and cache miss?
   **A:** Cache hit occurs when requested data is found in cache. Cache miss occurs when data is not found and must be fetched from the origin (database).

3. **Q:** Explain TTL in caching.
   **A:** TTL (Time-To-Live) specifies how long a cache entry is considered valid. After TTL expires, the entry is evicted or considered stale, forcing a fresh load from origin.

4. **Q:** What is cache eviction?
   **A:** Cache eviction is the process of removing entries from a full cache to make room for new entries, based on algorithms like LRU, LFU, or FIFO.

5. **Q:** What is the difference between local and distributed cache?
   **A:** Local cache resides in the application's memory (fast, but not shared across instances). Distributed cache is external (Redis, Memcached) and shared across all application instances.

6. **Q:** What annotation does Spring Boot provide for caching?
   **A:** `@Cacheable`, `@CachePut`, `@CacheEvict`, and `@Caching` from `org.springframework.cache.annotation`.

7. **Q:** What is the default caching provider in Spring Boot?
   **A:** Spring Boot auto-configures a `ConcurrentMapCacheManager` if no other cache provider (Redis, Caffeine, EhCache) is found on the classpath.

8. **Q:** What is a cache stampede?
   **A:** A cache stampede (thundering herd) occurs when many concurrent requests for the same key find it expired, all simultaneously hitting the database.

9. **Q:** What is the difference between @Cacheable and @CachePut?
   **A:** `@Cacheable` skips method execution on cache hit. `@CachePut` always executes the method and updates the cache with the result.

10. **Q:** What is cache-aside pattern?
    **A:** Application code explicitly manages cache: checks cache first, loads from DB on miss, stores result in cache. The cache is not responsible for loading data.

### Medium

11. **Q:** How would you handle cache consistency between cache and database?
    **A:** Use write-through or write-behind patterns. For eventual consistency, TTL-based invalidation. For strong consistency, use distributed transactions with cache-aside and explicit invalidation on writes.

12. **Q:** Explain how you would implement distributed cache locking.
    **A:** Use Redis SET NX with expiry for distributed mutex. The thread acquiring the lock loads data into cache, others wait and retry. Use Redisson for production-grade distributed locks.

13. **Q:** What caching strategy would you choose for a read-heavy social media feed?
    **A:** Multi-tier cache-aside with refresh-ahead. L1 (Caffeine) for hot data per node, L2 (Redis) for shared feed data. Use publish/subscribe for feed updates and proactive cache invalidation.

14. **Q:** How do you prevent cache stampede in a high-traffic system?
    **A:** Distributed locks (SET NX), early recalculation (refresh data before TTL expires), stale-while-revalidate (serve stale data while async refresh runs), or probabilistic early expiration.

15. **Q:** Design a caching layer for a product catalog with 10M SKUs.
    **A:** Use Redis Cluster with sharding. Cache-Aside pattern. Most-viewed SKUs cached with refresh-ahead. Long-tail products served from DB with small local cache. Bloom filter to prevent non-existent SKU lookups from hitting DB.

16. **Q:** What is the difference between write-through and write-behind caching?
    **A:** Write-through synchronously writes to both cache and database, ensuring consistency but higher write latency. Write-behind asynchronously persists to DB, offering lower latency but risk of data loss.

17. **Q:** How does Caffeine cache compare to Redis for caching?
    **A:** Caffeine is an in-memory, local cache with nanosecond read/write, limited to JVM heap. Redis is a distributed, network-based cache with millisecond latency, shared across instances, supports persistence and data structures.

18. **Q:** Explain the concept of "stale-while-revalidate" cache strategy.
    **A:** Serves stale cached data immediately while asynchronously fetching fresh data in the background. This eliminates cache miss latency for users while keeping cache eventually fresh.

19. **Q:** How would you implement cache warm-up for a new deployment?
    **A:** Use ApplicationRunner or CommandLineRunner to pre-load popular cache entries. Query the database for top-K frequently accessed items. For Redis, use pipelining for bulk loading. Consider using a cache warming service that runs after deployment.

20. **Q:** What metrics would you monitor for cache performance?
    **A:** Cache hit ratio, miss ratio, eviction count, average load time, cache size vs capacity, network latency to distributed cache, cache stampede frequency, and error rates.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Implement a thread-safe, non-blocking cache stampede prevention mechanism.
    **A:** Use Java's `CompletableFuture` with a loading cache pattern. The first thread triggers async loading and stores the Future in the cache. Subsequent threads attach to the same Future.

    ```java
    public class StampedeSafeCache<K, V> {
        private final ConcurrentHashMap<K, CompletableFuture<V>> cache = new ConcurrentHashMap<>();
        private final Function<K, V> loader;

        public V get(K key) {
            CompletableFuture<V> existingFuture = cache.get(key);
            if (existingFuture != null) {
                return existingFuture.join(); // Wait for ongoing load
            }

            CompletableFuture<V> newFuture = CompletableFuture.supplyAsync(() -> loader.apply(key));
            CompletableFuture<V> race = cache.putIfAbsent(key, newFuture);

            if (race != null) {
                return race.join(); // Another thread started first
            }

            try {
                V result = newFuture.get(5, TimeUnit.SECONDS);
                return result;
            } catch (Exception e) {
                cache.remove(key, newFuture);
                throw new CacheLoadException(e);
            }
        }
    }
    ```

2. **Q:** How do you handle cache invalidation across multiple data centers?
    **A:** Use global invalidation bus (Kafka across regions). Each data center has local Redis replicas. On write, publish invalidation event with global ID. Each DC consumes the event and invalidates local cache. Use vector clocks or timestamps to handle ordering. Alternatively, use Global Secondary Indexes with eventual consistency.

3. **Q:** Design a write-behind cache that guarantees at-most-once delivery to DB.
    **A:** Maintain a write-ahead log (WAL) in Redis (persistent) for pending writes. Track a monotonically increasing sequence number per partition. Store last committed sequence in DB. On recovery, replay from last committed sequence. Use idempotency keys on DB side to handle duplicates. Batch writes with configurable max batch size and flush interval.

4. **Q:** Explain how you would implement a multi-level cache with coherence protocol.
    **A:** L1 (per-node Caffeine), L2 (Redis Cluster), L3 (DB). Coherence via invalidation: writes invalidate L2, which broadcasts invalidation to all L1s via Redis pub/sub. L1 entries have shorter TTL than L2 (e.g., L1 TTL=60s, L2 TTL=300s). Use consistent hashing for Redis shard affinity. On L1 miss, check L2. On L2 miss, load from L3 and populate both L1 and L2.

5. **Q:** How do you detect and mitigate hot keys in a distributed cache?
    **A:** Detection: monitor request distribution per key. If one key exceeds N req/s (e.g., 1000), flag as hot. Mitigation: replicate hot key to multiple shards (read replicas), add local cache on each node for the hot key, split key with sub-keys (sharding within a key), or move to dedicated cache node.

6. **Q:** Implement a probabilistic early expiration algorithm.
    **A:** Instead of fixed TTL, add jitter. Compute expiry time as base TTL + random(-TTL/2, TTL/2). At read time, probabilistically check if we should early refresh: `refreshProbability = (currentTime - expiryTime) / (baseTTL / 2)`. If random() < probability, trigger async refresh. This distributes refreshes randomly, avoiding synchronized stampedes.

7. **Q:** Design a cache for a real-time leaderboard with millions of users.
    **A:** Use Redis Sorted Sets. Key: `leaderboard:{period}`. Member: userId. Score: total points. For real-time updates: ZINCRBY. For top-N: ZREVRANGE with scores. For user rank: ZREVRANK. Persist snapshot to DB every 5 minutes. Use read replicas for read scalability. For global + friends leaderboard, maintain per-user friend-group sorted sets.

8. **Q:** How do you handle cache poisoning in a multi-tenant SaaS application?
    **A:** Prefix all cache keys with tenant ID. Never allow tenant A to influence tenant B's cache. Validate all cached data schemas (deserialize with strict typing). Use separate Redis namespaces or databases per tenant tier. For shared-cache architectures, sign cache values with tenant-specific keys.

9. **Q:** Explain the write skew problem in caching and how to solve it.
    **A:** Write skew: Two concurrent transactions read overlapping data and write based on stale cache. Example: two doctors on call both see "no coverage" cache and both decline a shift. Solve with: pessimistic locking, validation based on DB fresh reads in transaction, or using compare-and-swap on cache with version counters.

10. **Q:** Design and implement a cache with adaptive TTL based on access frequency.
    **A:** Track access frequency per key using a sliding window (Redis sorted set per key with access timestamps). Compute optimal TTL as: `TTL = baseTTL / (1 + log(1 + freq))`. Hot keys get shorter TTL for fresher data, cold keys stay longer. Use a background task to recompute TTL for frequently accessed keys every N access.

### System Design

11. **Q:** Design a global caching layer for a video streaming platform (like Netflix).
    **A:** Multi-tier: CDN edge (popular content), regional Redis (metadata), local app cache (recommendations). Content: CDN with origin shielding. Metadata: Redis Cluster with read replicas per region. User progress: Write-Behind with async persistence. Use Open Connect (Netflix's own CDN) model for ISP-level caching. Pre-position popular content during off-peak.

12. **Q:** Design a caching strategy for an e-commerce flash sale event.
    **A:** Pre-warm cache with product data and inventory before sale. Use local L1 cache on each app node for inventory counts (eventually consistent). Write-through for inventory decrement to Redis. Batch DB writes with write-behind. Use Redis atomic DECR for inventory. Rate-limit cache miss refills. Serve stale inventory if Redis is down (allow limited overselling).

13. **Q:** Design a distributed rate limiter using caching.
    **A:** Redis sorted sets for sliding window: `ZREMRANGEBYSCORE key (now - window) now` + `ZCARD key`. Token bucket: Redis Lua script with `multi`/`exec`. For high throughput, use local token buckets sync to Redis periodically. For multi-region, divide rate limit by region count and allow bursting via central Redis.

14. **Q:** Design a cache layer for a real-time chat application.
    **A:** Cache recent messages per conversation (Redis sorted set by timestamp, TTL 24h). Cache user presence (Redis hash, TTL 5min). Cache unread counts (Redis counter with pub/sub for real-time update). Write-Behind for message persistence. L1 local cache for active conversations (LRU, 1000 entries per node).

15. **Q:** Design a caching solution for a weather data API.
    **A:** Weather data has natural TTL (forecasts). Cache at CDN (1-hour TTL) and Redis (30-min TTL). Cache key: `weather:{lat}:{lon}:{date}`. Use Geohash for spatial locality. Invalidate proactively when new forecast models run. Use stale-while-revalidate to serve slightly old data while fetching fresh forecast.

16. **Q:** Design a cache strategy for a social network's news feed.
    **A:** Fan-out-on-write: when user posts, pre-populate feed cache for all followers (Redis list per user, capped at 500). Fan-out-on-read for high-follower users (cache at read time). Write-Behind for post persistence. Multi-tier: local L1 for current session, Redis for active feeds, DB for history. Refresh-ahead for top 1% users.

17. **Q:** Design a caching layer for a stock trading application.
    **A:** Strong consistency required. Write-through cache with Redis + Redis Sentinel for HA. Real-time stock prices: Redis pub/sub + local L1 with 100ms TTL. Order book: Redis sorted sets. User positions: Write-through with optimistic locking (CAS). Audit trail: Write-Behind to separate DB. Circuit breaker: if Redis latency > 10ms, bypass cache and read directly from DB.

18. **Q:** Design a session caching system for a large web application.
    **A:** Redis as session store (Spring Session with Redis). Session serialization: JSON or Protocol Buffers. TTL based on session timeout (30 min default). Refresh TTL on each access. For high availability, Redis Sentinel or Cluster. Session replication across AZs using Redis cross-region replication. Sticky sessions optional (reduces Redis load).

19. **Q:** Design a caching strategy for a content management system (CMS).
    **A:** CDN for static assets. Edge caching (Varnish/CDN) for published pages with TTL. Redis for rendered page fragments (sidebar, header). Cache-Aside for content API. Invalidate by content ID on publish (Redis pub/sub + CDN purge). Hierarchical caching: page cache -> fragment cache -> data cache -> DB.

20. **Q:** Design a distributed job scheduler result cache.
    **A:** Cache job results in Redis with key `job:{jobId}:result`. TTL based on job result expiry (e.g., 7 days). On job completion, store result in cache and publish event. Polling clients check cache. For large results, store metadata in cache, data in object store (S3). Write-Behind to archive storage for long-term retention.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a write-back cache with ACID guarantees across a distributed system.
    **A:** Implement a distributed WAL (write-ahead log) using Kafka with exactly-once semantics. Cache acts as leader for writes; Kafka log is the source of truth. DB consumer reads from Kafka and persists. On cache failure, new cache replays from Kafka offset. Use idempotent DB writes with unique event IDs. For reads, serve from cache if offset matches latest committed, else read from DB.

2. **Q:** How would you design a cache invalidation protocol for a globally distributed application with 5 9s availability?
    **A:** Use a global invalidation bus (Kafka across regions) with CDC (Debezium). Each region has a full replica of cache. Invalidation events carry lamport timestamps for causal ordering. CRDT-based cache entries resolve concurrent updates. For availability, tolerate stale reads: serve cached data even if invalidation is pending. Use gossip protocol for region-to-region invalidation sync as fallback.

3. **Q:** Propose a caching architecture for a multi-tenant SaaS platform with 100K+ tenants and varying SLAs.
    **A:** Tiered: Free tier (shared Redis, LRU 1GB), Pro tier (dedicated Redis instance, 10GB), Enterprise tier (Redis Cluster with replication, 100GB). Tenant isolation via key prefix, namespacing, and optional DB index. Dynamic cache sizing: auto-scale Redis instance size based on tenant usage patterns. Cache-as-a-service: tenants can configure TTL, eviction policy, and cache warming via API.

4. **Q:** Design a consistency model for a hybrid cache (local + distributed) that guarantees monotonic reads.
    **A:** Assign monotonically increasing version numbers to each write. Distributed cache (Redis) stores the current global version. Local cache stores (value, version) pair. On read: get local version, compare with global version from Redis (lightweight). If local stale, fetch from Redis. Writes update global version + Redis. Use version leases: local cache holds a lease on a version range, avoiding version check on every read.

5. **Q:** Architect a self-tuning cache system that dynamically adjusts TTL, size, and eviction policy based on workload.
    **A:** Machine learning agent monitors access patterns (frequency, recency, inter-arrival time). Classifies entries into: hot (short TTL, replicate), warm (medium TTL), cold (long TTL, might evict), no-access (evict immediately). Dynamic policy: switch between LRU/LFU based on workload type. Auto-adjust max capacity: if eviction rate > 5%, increase capacity (if memory available). A/B test policy configurations across node clusters.

6. **Q:** How do you handle cache in a microservices architecture with hundreds of services sharing data?
    **A:** Data domains own their cache. Services never access another service's cache directly. Use a shared data platform (Redis Cluster) with domain-prefixed keys. Each domain's cache is an internal implementation detail. Cross-domain cache invalidation via event bus: Service A publishes `event.v1.OrderUpdated`, Service B consumes and invalidates its order-related cache entries. Consider GraphQL federation with cache per subgraph.

7. **Q:** Design a cache system that provides linearizable reads and writes across a distributed cluster.
    **A:** Use a consensus-based cache (like etcd or ZooKeeper for small data) or implement Raft consensus for cache operations. Cache nodes form a Raft cluster. Writes go through Raft leader, which replicates to followers. Reads also go through leader for linearizability, or use follower reads with bounded staleness. Combine with local caching where reads hold a read lease from the Raft cluster. This trades throughput for strong consistency.

8. **Q:** How would you architect a zero-downtime cache migration across data centers?
    **A:** Dual-cache pattern: run old and new cache simultaneously. Configure application to write to both, read from old (primary). Backfill new cache from old using scan + pipeline. Monitor new cache hit ratio. When ratio exceeds 99%, switch read to new (primary). Deprecate old cache. Use feature flags for gradual migration. For bidirectional replication during transition, use Kafka to capture writes and replay to both.

9. **Q:** Design a caching system for a financial trading platform that requires both low latency (<1ms) and auditability.
    **A:** L1: local Caffeine (nanosecond reads) for hot keys. L2: Redis on the same rack/zone (sub-millisecond). Audit trail: every cache mutation is logged to a write-ahead log (Kafka). WAL includes before/after values, timestamp, and request ID. L1 intercepts writes: writes go to L2, L2 publishes audit event, L2 responds, L1 caches. For full audit, replay WAL to reconstruct state at any point. Use epoch-based timing for clocks.

10. **Q:** Propose a multi-region active-active caching architecture with conflict-free replicated data types (CRDTs).
    **A:** Each region has a full Redis cluster. Cache entries are CRDTs: Last-Writer-Wins (LWW) for simple values, Observed-Remove Set (OR-Set) for collections, PN-Counter for counts. Writes are local (fast) and asynchronously replicated to other regions via Kafka. Merge conflicts automatically via CRDT semantics. Use hybrid logical clocks (HLC) for accurate timestamps across regions. Each key has a region-ID + timestamp. On conflict, region-ID breaks ties. Read-repair: on read in region A, merge concurrent updates from all regions.

## 13. Debugging & Troubleshooting

### Common Cache Issues

**Issue: Low cache hit ratio (<50%)**
- Check if TTL is too short (entries expire before reuse).
- Check if cache key composition is inconsistent.
- Verify that application code actually reuses the same keys.
- Use `redis-cli --bigkeys` to analyze key distribution.

**Issue: Redis memory full, evictions increasing**
```bash
redis-cli> INFO stats | grep evicted_keys
redis-cli> MEMORY DOCTOR
redis-cli> MEMORY USAGE product:123
```
- Increase maxmemory or adjust eviction policy.
- Identify largest keys with `redis-cli --bigkeys`.
- Reduce TTL or entry size.

**Issue: Inconsistent cache data**
- Check invalidation logic: are all write paths covered?
- Look for race conditions in async cache updates.
- Verify that serialization/deserialization is correct.
- Use A/B comparison: run parallel cache reads against DB.

**Issue: Cache stampede under load**
```java
// Detection: monitor load on DB during cache TTL boundaries
// Mitigation: implement distributed lock or jitter TTL
redisTemplate.opsForValue().set(cacheKey, value, 
    Duration.ofSeconds(baseTTL + ThreadLocalRandom.current().nextInt(-60, 60)));
```

**Issue: Serialization errors**
- Stack trace: `org.springframework.data.redis.serializer.SerializationException`
- Ensure cached objects implement `Serializable` or use JSON serialization.
- Version `serialVersionUID` when using Java serialization.
- For JSON, use Jackson with `@JsonTypeInfo` for polymorphic types.

### Debugging Commands
```bash
# Check cache hit ratio
redis-cli> INFO stats
# Output: keyspace_hits:15000, keyspace_misses:500, hit_ratio:96.7%

# Scan for large keys
redis-cli --bigkeys

# Monitor cache operations in real-time
redis-cli MONITOR | grep "product:"

# Check TTL of a specific key
redis-cli TTL product:123

# Check memory usage
redis-cli MEMORY STATS

# Trace cache flow in Spring Boot
# Set logging level in application.yml:
logging.level.org.springframework.cache=TRACE
logging.level.org.springframework.data.redis.cache=TRACE
```

## 14. Comparison Section

### Cache-Aside vs Read-Through vs Write-Through vs Write-Behind

| Aspect | Cache-Aside | Read-Through | Write-Through | Write-Behind |
|--------|-------------|--------------|---------------|--------------|
| Read Complexity | App manages cache | Cache manages itself | Simpler reads | Same as read-through |
| Write Complexity | App invalidates | App writes directly | App writes to cache | App writes to cache |
| Consistency | Eventual (if TTL) | Eventual | Strong | Eventual |
| Write Latency | Low (no cache update) | N/A | Higher (sync DB) | Very low (async) |
| Read Latency | Higher on miss | Higher on miss | Low | Low |
| Data Loss Risk | None | None | None | Yes (cache failure) |
| Use Case | Read-heavy, moderate writes | Read-heavy, simple reads | Consistency-critical | Write-heavy, throughput |

### Local Cache vs Distributed Cache

| Aspect | Local Cache (Caffeine) | Distributed Cache (Redis) |
|--------|----------------------|--------------------------|
| Latency | Nanosecond | Millisecond (network) |
| Capacity | JVM heap limited | GB/TB scale |
| Consistency | Per-node only | Shared across nodes |
| Complexity | Embedded | Requires deployment |
| Persistence | None (in-memory) | Optional (RDB/AOF) |
| Data Structures | Key-value | Rich (lists, sets, etc.) |

### LRU vs LFU vs FIFO vs TinyLFU

| Policy | Best For | Worst For | Overhead |
|--------|----------|-----------|----------|
| LRU | Recent access patterns | One-time large scans | Low |
| LFU | Stable popularity | Bursty traffic | Medium |
| FIFO | Streaming/cursors | Hot data | Minimal |
| TinyLFU | High hit ratio | Memory constrained | Medium |
| ARC (Adaptive) | Mixed workloads | Predictability | Higher |

## 15. Revision Notes

### Quick Recap
- **Cache-Aside**: Most common pattern. App checks cache, loads on miss, invalidates on write.
- **Read-Through**: Cache auto-loads on miss. Simpler app code.
- **Write-Through**: Sync write to cache + DB. Strong consistency.
- **Write-Behind**: Async DB write. High throughput, risk of loss.
- **Refresh-Ahead**: Proactive refresh before expiration.
- **Eviction Policies**: LRU (most used), LFU (frequency), FIFO (order), TinyLFU (efficient).
- **Cache Stampede Prevention**: Distributed locks, jitter, stale-while-revalidate.
- **Spring Boot Annotations**: `@Cacheable`, `@CacheEvict`, `@CachePut`, `@Caching`.

### Key Formulas
```
Hit Ratio = Cache Hits / (Cache Hits + Cache Misses)
Miss Penalty = Cache Load Time + DB Query Time
Effective Read Latency = Hit_Ratio * Cache_Time + (1 - Hit_Ratio) * Miss_Penalty
Optimal TTL = Data_Volatility_Window / 2
```

### Anti-Patterns to Avoid
- Caching mutable data without invalidation strategy.
- Same TTL for all entries regardless of access frequency.
- Ignoring cache stampede in high-concurrency systems.
- Using cache for authentication/authorization state.
- Mixing tenant data without key isolation.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                    CACHING STRATEGIES CHEAT SHEET                  |
+-------------------------------------------------------------------+
| PATTERN        | READ                   | WRITE                   |
|----------------+------------------------+-------------------------|
| Cache-Aside    | App: get->miss->DB->   | App: DB write->evict    |
|                |      set->return       |                         |
| Read-Through   | Cache: miss->DB->set   | App: DB write directly  |
|                |      ->return          |                         |
| Write-Through  | Same as read-through   | App: cache set->DB set->|
|                |                        |      confirm            |
| Write-Behind   | Same as read-through   | App: cache set->ok;     |
|                |                        |      async->DB write    |
| Refresh-Ahead  | Cache: predict refresh | App: direct DB write    |
|                |      ->DB->update TTL  |                         |
+-------------------------------------------------------------------+
| EVICTION POLICY | DESCRIPTION            | USE CASE                |
+-----------------+------------------------+-------------------------+
| LRU             | Evict least recently   | General purpose        |
|                 | used                   |                         |
| LFU             | Evict least frequently | Hot data with stable    |
|                 | used                   | popularity              |
| FIFO            | Evict oldest           | Streaming, cursors      |
| TTL             | Evict expired by time  | Time-bound data         |
| TinyLFU         | Frequency sketch +     | High hit ratio apps     |
|                 | LRU                    |                         |
+-------------------------------------------------------------------+
| SPRING BOOT ANNOTATIONS                                            |
+-------------------------------------------------------------------+
| @Cacheable(cacheNames, key, unless)   | Cache method result      |
| @CacheEvict(cacheNames, key, allEntries) | Remove from cache    |
| @CachePut(cacheNames, key)            | Always execute + update  |
| @Caching(evict={...}, put={...})      | Combine multiple ops     |
+-------------------------------------------------------------------+
| STAMPEDE PREVENTION                                                |
+-------------------------------------------------------------------+
| 1. Distributed Lock: SETNX lock_key TTL 5s                         |
| 2. TTL Jitter: baseTTL + random(-jitter, +jitter)                  |
| 3. Stale-while-revalidate: serve old + async refresh               |
| 4. Probabilistic early expiration                                  |
| 5. Cache warming: pre-load before traffic                          |
+-------------------------------------------------------------------+
| METRICS TO MONITOR                                                 |
+-------------------------------------------------------------------+
| hit_ratio = hits / (hits + misses)           target > 95%          |
| eviction_rate = evictions / second           should be low         |
| miss_latency = avg time to load + DB query   minimize              |
| cache_size / max_memory                       < 80% threshold       |
+-------------------------------------------------------------------+
```
