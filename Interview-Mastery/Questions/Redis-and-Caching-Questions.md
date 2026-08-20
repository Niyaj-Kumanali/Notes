# Redis and Caching Questions

## Questions

1. What is Redis?
2. Why use Redis?
3. What data types does Redis support?
4. What is cache?
5. What is cache-aside pattern?
6. What is read-through cache?
7. What is write-through cache?
8. What is write-behind cache?
9. What is TTL?
10. What is cache eviction?
11. What is LRU?
12. What is cache stampede?
13. How do you prevent cache stampede?
14. What is cache penetration?
15. What is cache avalanche?
16. How do you invalidate cache?
17. What should not be cached?
18. How do you cache API responses?
19. How do you cache database queries?
20. How do you use Redis for session storage?
21. How do you use Redis for distributed locking?
22. What is Redis pub/sub?
23. What is Redis stream?
24. How do you secure Redis?
25. How did Redis fit in your project?

---

## Answers

1. What is Redis?
   - **Answer:**
      - Redis is an in-memory key-value store used for caching, session storage, and distributed locking
      - For example, Redis can cache API responses to reduce load on a time-series database and store session data so that stateless application instances can share user sessions
      - Redis uses a single-threaded event loop model, processing commands sequentially in memory, which eliminates lock contention and context-switching overhead — this is how it achieves sub-millisecond latency; optional persistence via RDB snapshots or AOF logs provides durability beyond the in-memory dataset
2. Why use Redis?
   - **Answer:**
      - Redis is fast (in-memory), supports multiple data structures (strings, hashes, lists, sets), and has built-in features like TTL, pub/sub, and distributed locking
      - For instance, Redis can be used to cache frequently accessed data, reducing database query load significantly
      - Compared with Memcached: Memcached is simpler (string-only key-value, multithreaded, good for flat caching), but Redis supports rich data structures (hashes, sorted sets, streams), built-in persistence (RDB/AOF), atomic operations, and pub/sub — these features make Redis more suitable for session storage (using hashes for per-field access) and distributed locking (using SETNX)
3. What data types does Redis support?
   - **Answer:**
      - Redis supports strings, hashes, lists, sets, sorted sets, bitmaps, hyperloglogs, and streams
      - Strings are used for cache entries, hashes for session storage (mapping a user ID to multiple session fields), and sorted sets for ranking items by a score such as temperature deviation
      - Data type selection is based on the access pattern — hashes are ideal for session data because individual fields can be read/written (e.g., updating a single session attribute) without serializing and deserializing the entire object, reducing network overhead and avoiding race conditions between concurrent field updates
4. What is cache?
   - **Answer:**
      - A cache stores frequently accessed data in a fast storage layer to reduce latency and backend load
      - For example, Redis can cache the latest sensor reading from each device so a dashboard can display it instantly without querying the time-series database every time a user loads the page
      - The cache-aside pattern: the application checks Redis first, and only queries the database on a cache miss, then stores the result in Redis with a TTL
5. What is cache-aside pattern?
   - **Answer:**
      - In cache-aside (lazy loading), the application checks the cache first; on a miss, it loads data from the database, stores it in the cache, and returns it
      - This pattern is commonly used for read-heavy APIs — a Redis miss triggers a database query, after which the result is cached with a short TTL (e.g., 30 seconds)
      - Cache-aside is chosen over read-through because it gives the application explicit control over what gets cached and when — for example, it's possible to skip caching for certain query results, apply custom serialization, or add business logic around the cache-miss path, and it's simpler to implement without requiring a cache provider abstraction layer like Spring's `CacheLoader`
6. What is read-through cache?
   - **Answer:**
      - In read-through, the cache itself loads data from the database on a miss, transparent to the application
      - Redis doesn't natively support read-through; it requires a cache provider like Spring's `@Cacheable` with `CacheLoader`
      - Spring's cache abstraction handles the miss transparently
      - Read-through is simpler for developers — the application just calls the cache API and never worries about database queries — but it's less flexible because fine-grained caching logic (e.g., which fields to cache, conditional caching, or custom serialization) can't be controlled without implementing a custom `CacheLoader`, and the cache provider owns the data-loading strategy
7. What is write-through cache?
   - **Answer:**
      - In write-through, data is written to both cache and database in the same transaction
      - This pattern can be used for updating critical configuration — when a status change is written to the database, the corresponding Redis cache entry is also updated immediately, preventing stale data
      - Write-through ensures cache consistency (the cache is always up-to-date after a write) but adds latency to write operations because both the cache and the database must be written before the operation returns — it should be reserved for critical data where staleness isn't acceptable, while using TTL-based expiry for less critical data
8. What is write-behind cache?
   - **Answer:**
      - In write-behind, data is written to cache immediately and asynchronously persisted to the database later
      - Write-behind can be considered for batch ingestion scenarios but is risky if losing buffered data during a Redis crash is unacceptable (e.g., compliance-critical data)
      - If Redis goes down before the async write to the database completes, data is lost — this makes write-behind risky for any compliance-critical data; it should be used only for non-critical data like heartbeat timestamps where occasional loss doesn't impact business correctness
9. What is TTL?
   - **Answer:**
      - TTL (Time To Live) is the duration after which a cached entry is automatically evicted
      - For example, a 30-second TTL can be set for frequently changing sensor-reading cache entries and a 24-hour TTL for configuration data that rarely changed
      - TTL values are chosen based on two factors: how frequently the underlying data changes (high-frequency updates -> short TTL; weekly config changes -> long TTL) and how stale the dashboard data can be before users notice or make incorrect decisions based on outdated information
10. What is cache eviction?
    - **Answer:**
       - Cache eviction removes entries when memory is full
       - Redis supports multiple eviction policies (LRU, LFU, TTL, etc.)
       - `allkeys-lru` eviction can be configured, which removes the least recently used keys when memory reaches the `maxmemory` limit, ensuring the most accessed data stays cached
       - LRU is chosen over other policies when there's a predictable access pattern — recently accessed items are more likely to be viewed again, while items that haven't been queried in a while are unlikely to be needed soon
11. What is LRU?
    - **Answer:**
       - LRU (Least Recently Used) evicts entries that haven't been accessed for the longest time
       - Redis's `allkeys-lru` policy approximates LRU
       - This means that items not being actively accessed are evicted first, while frequently viewed items stay cached
       - Redis's approximate LRU differs from exact LRU — instead of maintaining a full access-order linked list for every key (which would consume significant memory), Redis periodically samples a configurable number of keys (`maxmemory-samples`, default 5) and evicts the least recently used among the sample; increasing the sample size improves accuracy at the cost of more CPU per eviction, and even with a small sample size the approximation is accurate enough for caching workloads
12. What is cache stampede?
    - **Answer:**
       - A cache stampede occurs when many requests simultaneously miss the cache (e.g., after TTL expiry), all hitting the database at once
       - This can happen with a list API endpoint — every dashboard refresh can cause all users to query the database when the cache expires simultaneously
       - During stampede events, database CPU can spike significantly, which may threaten to delay write operations for other critical data — if writes are delayed, the system could miss real-time alerts, making stampede prevention critical beyond just performance
13. How do you prevent cache stampede?
    - **Answer:**
       - Stampades can be prevented by: (1) using mutex locks so only one request rebuilds the cache, (2) randomizing TTLs slightly to prevent simultaneous expiry, and (3) using early recomputation — refreshing the cache before it expires
       - A Redis lock can be used to serialize cache rebuilds for a list endpoint
       - The implementation uses `jedis.setnx` to acquire the lock — if `setnx` returns `true`, the current thread is responsible for rebuilding the cache; other threads wait and retry after a short delay. The lock key has a 5-second TTL to prevent deadlocks if the rebuilding thread crashes or hangs, ensuring the lock would automatically release
14. What is cache penetration?
    - **Answer:**
       - Cache penetration happens when requests come for data that doesn't exist in either cache or database, causing repeated database misses
       - For example, if a user requests a non-existent item ID, Redis would miss, the database would miss, and the bad request would bypass caching entirely
       - A common solution is to cache the null result (a special "not found" marker value) with a short TTL (e.g., 60 seconds) so repeated requests for the same invalid ID hit Redis instead of the database — the short TTL prevents the cache from being permanently polluted with invalid entries while still absorbing burst traffic from a misconfigured client or a stale link
15. What is cache avalanche?
    - **Answer:**
       - A cache avalanche occurs when many cached entries expire at the same time, causing a flood of requests to the database
       - If all cache entries for a resource share the same TTL and expire together, the database would get a sudden load spike
       - This is prevented by adding a random jitter of plus/minus 5 seconds to each TTL value — instead of a fixed 30-second TTL, each entry gets a TTL between 25 and 35 seconds, spreading cache rebuilds across a window and preventing simultaneous expiry of large batches of keys
16. How do you invalidate cache?
    - **Answer:**
       - Cache is invalidated by: (1) setting appropriate TTLs for automatic expiry, (2) explicitly deleting keys when data changes (write-through), and (3) using a version prefix in cache keys for bulk invalidation
       - For example, when a configuration is updated, the corresponding Redis key can be deleted so the next read would fetch fresh data
       - The version prefix pattern works by appending a version number to cache keys (e.g., `gateway:123:v2`) — when a config change requires invalidating all cached entries for an item, the version in a separate key is incremented; the next read uses the new version prefix, effectively invalidating all old cache entries without iterating through keys or issuing individual `DEL` commands, which would be expensive at scale
17. What should not be cached?
    - **Answer:**
       - Don't cache: (1) frequently changing data where staleness is unacceptable, (2) sensitive user data that has compliance requirements, (3) data used only once
       - For example, real-time alerts should not be cached because they need to be displayed immediately and the alert timing is unpredictable
       - Caching financial or health-critical data requires careful consideration of how stale values could impact decisions — real-time critical alerts (a delayed alert could mean undetected spoilage) carry risk when cached, so the cost of serving a slightly stale alert may far outweigh the performance savings
18. How do you cache API responses?
    - **Answer:**
       - API responses can be cached using Spring Boot's `@Cacheable` annotation with Redis as the backing store
       - For example, a `getLatestTemperature` method can be annotated with `@Cacheable(value="sensors", key="#sensorId")`, which automatically caches the response and serves it on subsequent calls until the TTL expires
       - The cache configuration uses `spring.cache.redis.time-to-live=30s` for a 30-second default TTL, and a cache key prefix (`sensors:`) can be configured to namespace entries and avoid key collisions between different API caches (e.g., `sensors:` vs `gateways:` vs `sessions:`) — this makes cache debugging with `redis-cli` straightforward since entries can be scanned by prefix
19. How do you cache database queries?
    - **Answer:**
       - Database query results can be cached at the service layer using `@Cacheable` around the repository call
       - For example, a configuration query (which rarely changed) can be cached with a 1-hour TTL, reducing database query load for this frequently accessed but static data
       - The trade-off: if a config changes, the cached version could be up to 1 hour stale — to solve this, a manual cache eviction call (`@CacheEvict`) is added in the update endpoint so that any config change immediately invalidates the affected entry, ensuring the next read fetches fresh data while still benefiting from caching between updates
20. How do you use Redis for session storage?
    - **Answer:**
       - Spring Session can be configured with Redis as the session store, so user sessions are stored in Redis instead of the application's local memory
       - This allows any instance in the auto-scaling group to serve any user request without losing session state, enabling horizontal scaling
       - The setup requires adding `@EnableRedisHttpSession` to the Spring Boot application class and setting `spring.session.store-type=redis` in properties — by default, Spring Session serializes session attributes using Java serialization, but for cross-language compatibility (if another service in a different language needs to read sessions), Jackson-based JSON serialization can be configured via a custom `RedisSerializer` bean
21. How do you use Redis for distributed locking?
    - **Answer:**
       - Distributed locking ensures that only one instance of an application executes a critical section
       - Redis `SETNX` (set if not exists) with a TTL can be used to implement a distributed lock that prevents duplicate processing of the same event across multiple consumer instances
       - For stronger guarantees, the Redlock algorithm distributes the lock across multiple independent Redis nodes (majority must agree) — but for use cases where the processing logic is idempotent (processing an event twice is safe, just inefficient), a single Redis node with SETNX is sufficient and occasional lock failures during Redis failover are acceptable
22. What is Redis pub/sub?
    - **Answer:**
       - Redis pub/sub allows messages to be broadcast to multiple subscribers
       - Redis pub/sub can be considered for live dashboard updates, but alternatives like Kafka may be chosen instead because Kafka provides persistence and replay — if the dashboard client disconnects, it would miss messages sent via Redis pub/sub
       - Redis pub/sub is fire-and-forget: messages are not persisted to disk, there is no acknowledgment mechanism, and if no subscriber is connected at the time of publish, the message is lost; this makes it suitable only for ephemeral notifications (e.g., real-time UI alerts) but not for reliable data delivery where message loss would impact correctness
23. What is Redis stream?
    - **Answer:**
       - Redis Streams is an append-only log data structure similar to Kafka topics, with consumer groups and message acknowledgment
       - Redis Streams can be evaluated as an alternative to Kafka for event streaming, but Kafka's partition model provides better throughput and horizontal scaling at higher volumes
       - Redis Streams is a better choice than Kafka when: the setup needs to be simpler (single Redis instance, no separate broker cluster), message volume is lower (< 1000 msg/sec), latency matters more than throughput, and there's no need for the advanced consumer group features Kafka provides (multi-consumer fan-out, offset management across partitions) — for small-to-medium projects, Redis Streams avoids the operational overhead of managing a Kafka cluster
24. How do you secure Redis?
    - **Answer:**
       - Redis can be secured by: (1) not exposing it to the public internet (deployed in a private subnet), (2) using a strong password via the `requirepass` config, (3) disabling dangerous commands like `FLUSHALL` and `KEYS` via rename-command in redis.conf, and (4) using TLS encryption for data in transit
       - Redis's security model is simple — it trusts the network by default, meaning authentication is optional and connections are unencrypted, so network-level security is critical: Redis should be deployed in a private VPC subnet with security groups restricting access to only the application tier, port 6379 should never be exposed to the internet, and the default configuration (no password, bind to all interfaces) should always be changed before deploying to any environment
25. How did Redis fit in your project?
    - **Answer:**
       - Redis typically plays three roles in a real-time monitoring system: (1) cache for API responses (latest readings, device lists) to reduce database load, (2) session storage for application instances to enable horizontal scaling, and (3) distributed locking to prevent duplicate processing of events across multiple consumer instances
       - After adding Redis caching, the dashboard's latest-reading API latency can drop from 200ms (direct database query) to under 5ms (Redis lookup), and database query volume can reduce by 60% — the session store eliminates sticky sessions, allowing scaling from 2 to 4 application instances behind the load balancer without any session-related issues, and the distributed lock prevents a race condition where two consumer instances could process the same event simultaneously during a rebalance
