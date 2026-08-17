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
      - In the cold-chain project, Redis cached API responses to reduce load on InfluxDB and stored session data so that our stateless Spring Boot instances could share user sessions
      - Redis uses a single-threaded event loop model, processing commands sequentially in memory, which eliminates lock contention and context-switching overhead — this is how it achieves sub-millisecond latency; optional persistence via RDB snapshots or AOF logs provides durability beyond the in-memory dataset
2. Why use Redis?
   - **Answer:**
      - Redis is fast (in-memory), supports multiple data structures (strings, hashes, lists, sets), and has built-in features like TTL, pub/sub, and distributed locking
      - In cold-chain, I used Redis to cache frequently accessed temperature summaries, reducing InfluxDB query load by about 60%
      - Compared with Memcached: Memcached is simpler (string-only key-value, multithreaded, good for flat caching), but Redis supports rich data structures (hashes, sorted sets, streams), built-in persistence (RDB/AOF), atomic operations, and pub/sub — these features made Redis more suitable for session storage (using hashes for per-field access) and distributed locking (using SETNX) in the cold-chain project
3. What data types does Redis support?
   - **Answer:**
      - Redis supports strings, hashes, lists, sets, sorted sets, bitmaps, hyperloglogs, and streams
      - In cold-chain, I used strings for cache entries, hashes for session storage (mapping user ID to multiple session fields), and sorted sets for ranking gateways by temperature deviation
      - I chose the data type based on the access pattern — hashes were perfect for session data because I could read/write individual fields (e.g., update a single session attribute) without serializing and deserializing the entire object, reducing network overhead and avoiding race conditions between concurrent field updates
4. What is cache?
   - **Answer:**
      - A cache stores frequently accessed data in a fast storage layer to reduce latency and backend load
      - In cold-chain, Redis cached the latest temperature reading from each sensor so the dashboard could display it instantly without querying InfluxDB every time a user loaded the page
      - The cache-aside pattern: the application checked Redis first, and only queried InfluxDB on a cache miss, then stored the result in Redis with a TTL
5. What is cache-aside pattern?
   - **Answer:**
      - In cache-aside (lazy loading), the application checks the cache first; on a miss, it loads data from the database, stores it in the cache, and returns it
      - I used this for the latest-sensor-reading API — Redis miss meant a query to InfluxDB, after which the result was cached with a 30-second TTL
      - I chose cache-aside over read-through because it gives the application explicit control over what gets cached and when — for example, I could skip caching for certain query results, apply custom serialization, or add business logic around the cache-miss path, and it's simpler to implement without requiring a cache provider abstraction layer like Spring's `CacheLoader`
6. What is read-through cache?
   - **Answer:**
      - In read-through, the cache itself loads data from the database on a miss, transparent to the application
      - Redis doesn't natively support read-through; it requires a cache provider like Spring's `@Cacheable` with `CacheLoader`
      - Spring's cache abstraction handles the miss transparently
      - Read-through is simpler for developers — the application just calls the cache API and never worries about database queries — but it's less flexible because you can't control fine-grained caching logic (e.g., which fields to cache, conditional caching, or custom serialization) without implementing a custom `CacheLoader`, and the cache provider owns the data-loading strategy
7. What is write-through cache?
   - **Answer:**
      - In write-through, data is written to both cache and database in the same transaction
      - I used this pattern for updating gateway status in cold-chain — when a status change was written to MSSQL, the corresponding Redis cache entry was also updated immediately, preventing stale data
      - Write-through ensures cache consistency (the cache is always up-to-date after a write) but adds latency to write operations because both the cache and the database must be written before the operation returns — I reserved it for critical data where staleness wasn't acceptable, like gateway status, while using TTL-based expiry for less critical data
8. What is write-behind cache?
   - **Answer:**
      - In write-behind, data is written to cache immediately and asynchronously persisted to the database later
      - I considered this for sensor reading batching but didn't implement it because losing readings during a Redis crash was unacceptable for the cold-chain compliance requirements
      - If Redis goes down before the async write to the database completes, data is lost — this makes write-behind risky for any compliance-critical data; I used this pattern only for non-critical data like gateway heartbeat timestamps where occasional loss didn't impact business correctness
9. What is TTL?
   - **Answer:**
      - TTL (Time To Live) is the duration after which a cached entry is automatically evicted
      - In cold-chain, I set a 30-second TTL for sensor-reading cache entries and a 24-hour TTL for partner configuration data that rarely changed
      - I chose TTL values based on two factors: how frequently the underlying data changed (sensor readings every second → short TTL; partner config changes weekly → long TTL) and how stale the dashboard data could be before users noticed or made incorrect decisions based on outdated information
10. What is cache eviction?
    - **Answer:**
       - Cache eviction removes entries when memory is full
       - Redis supports multiple eviction policies (LRU, LFU, TTL, etc.)
       - I configured Redis with `allkeys-lru` eviction, which removed the least recently used keys when memory reached the `maxmemory` limit, ensuring the most accessed data stayed cached
       - I chose LRU over other policies because cold-chain had a predictable access pattern — recently accessed sensors (those actively reporting temperature) were more likely to be viewed again on the dashboard, while sensors that hadn't been queried in a while were unlikely to be needed soon
11. What is LRU?
    - **Answer:**
       - LRU (Least Recently Used) evicts entries that haven't been accessed for the longest time
       - Redis's `allkeys-lru` policy approximates LRU
       - In cold-chain, this meant that sensors not being actively monitored were evicted first, while the dashboard's frequently viewed sensors stayed cached
       - Redis's approximate LRU differs from exact LRU — instead of maintaining a full access-order linked list for every key (which would consume significant memory), Redis periodically samples a configurable number of keys (`maxmemory-samples`, default 5) and evicts the least recently used among the sample; increasing the sample size improves accuracy at the cost of more CPU per eviction, and even with a small sample size the approximation is accurate enough for caching workloads
12. What is cache stampede?
    - **Answer:**
       - A cache stampede occurs when many requests simultaneously miss the cache (e.g., after TTL expiry), all hitting the database at once
       - In cold-chain, this happened with the gateways list API — every dashboard refresh caused all users to query InfluxDB when the cache expired
       - During stampede events, InfluxDB CPU spiked to 90%, which threatened to delay write operations for new sensor readings — if sensor writes were delayed, the cold-chain compliance system could miss temperature excursion alerts, making stampede prevention critical beyond just performance
13. How do you prevent cache stampede?
    - **Answer:**
       - I prevent stampedes by: (1) using mutex locks so only one request rebuilds the cache, (2) randomizing TTLs slightly to prevent simultaneous expiry, and (3) using early recomputation — refreshing the cache before it expires
       - In cold-chain, I used a Redis lock to serialize cache rebuilds for the gateways list
       - The implementation used `jedis.setnx` to acquire the lock — if `setnx` returned `true`, the current thread was responsible for rebuilding the cache; other threads waited and retried after a short delay. The lock key had a 5-second TTL to prevent deadlocks if the rebuilding thread crashed or hung, ensuring the lock would automatically release
14. What is cache penetration?
    - **Answer:**
       - Cache penetration happens when requests come for data that doesn't exist in either cache or database, causing repeated database misses
       - In cold-chain, if a user requested a non-existent sensor ID, Redis would miss, InfluxDB would miss, and the bad request would bypass caching entirely
       - My solution was to cache the null result (a special "not found" marker value) with a short TTL (e.g., 60 seconds) so repeated requests for the same invalid sensor ID hit Redis instead of InfluxDB — the short TTL prevented the cache from being permanently polluted with invalid entries while still absorbing burst traffic from a misconfigured client or a stale dashboard link
15. What is cache avalanche?
    - **Answer:**
       - A cache avalanche occurs when many cached entries expire at the same time, causing a flood of requests to the database
       - In cold-chain, if all sensor-reading cache entries had the same 30-second TTL and expired together, InfluxDB would get a sudden load spike
       - I prevented this by adding a random jitter of ±5 seconds to each TTL value — instead of a fixed 30-second TTL, each entry got a TTL between 25 and 35 seconds, spreading cache rebuilds across a window and preventing simultaneous expiry of large batches of keys
16. How do you invalidate cache?
    - **Answer:**
       - I invalidate cache by: (1) setting appropriate TTLs for automatic expiry, (2) explicitly deleting keys when data changes (write-through), and (3) using a version prefix in cache keys for bulk invalidation
       - In cold-chain, when a partner updated gateway configuration, I deleted the corresponding Redis key so the next read would fetch fresh data
       - The version prefix pattern works by appending a version number to cache keys (e.g., `gateway:123:v2`) — when a config change requires invalidating all cached entries for a gateway, I increment the version in a separate key; the next read uses the new version prefix, effectively invalidating all old cache entries without iterating through keys or issuing individual `DEL` commands, which would be expensive at scale
17. What should not be cached?
    - **Answer:**
       - Don't cache: (1) frequently changing data where staleness is unacceptable, (2) sensitive user data that has compliance requirements, (3) data used only once
       - In cold-chain, I didn't cache temperature excursion alerts because they needed to be displayed immediately and the alert TTL was unpredictable
       - Caching financial or health-critical data requires careful consideration of how stale values could impact decisions — temperature excursion alerts were real-time critical (a delayed alert could mean undetected spoilage), so caching added risk without benefit; the cost of serving a slightly stale alert far outweighed the performance savings
18. How do you cache API responses?
    - **Answer:**
       - I cache API responses using Spring Boot's `@Cacheable` annotation with Redis as the backing store
       - In cold-chain, I annotated the `getLatestTemperature` method with `@Cacheable(value="sensors", key="#sensorId")`, which automatically cached the response and served it on subsequent calls until the TTL expired
       - The cache configuration used `spring.cache.redis.time-to-live=30s` for a 30-second default TTL, and I configured the cache key prefix (`sensors:`) to namespace entries and avoid key collisions between different API caches (e.g., `sensors:` vs `gateways:` vs `sessions:`) — this made cache debugging with `redis-cli` straightforward since I could scan by prefix
19. How do you cache database queries?
    - **Answer:**
       - I cache database query results at the service layer using `@Cacheable` around the repository call
       - In cold-chain, the gateway configuration query (which rarely changed) was cached with a 1-hour TTL, reducing MSSQL query load for this frequently accessed but static data
       - The trade-off: if a gateway config changed, the cached version could be up to 1 hour stale — to solve this, I added a manual cache eviction call (`@CacheEvict`) in the gateway update endpoint so that any config change immediately invalidated the affected entry, ensuring the next read fetched fresh data from MSSQL while still benefiting from caching between updates
20. How do you use Redis for session storage?
    - **Answer:**
       - I configured Spring Session with Redis as the session store, so user sessions were stored in Redis instead of the application's local memory
       - This allowed any instance in the auto-scaling group to serve any user request without losing session state, enabling horizontal scaling
       - The setup required adding `@EnableRedisHttpSession` to the Spring Boot application class and setting `spring.session.store-type=redis` in properties — by default, Spring Session serializes session attributes using Java serialization, but for cross-language compatibility (if a Python service needed to read sessions), I configured Jackson-based JSON serialization via a custom `RedisSerializer` bean
21. How do you use Redis for distributed locking?
    - **Answer:**
       - Distributed locking ensures that only one instance of an application executes a critical section
       - In the cold-chain project, I used Redis `SETNX` (set if not exists) with a TTL to implement a distributed lock that prevented duplicate processing of the same sensor reading across multiple consumer instances
       - For stronger guarantees, the Redlock algorithm distributes the lock across multiple independent Redis nodes (majority must agree) — but for cold-chain's use case, a single Redis node with SETNX was sufficient because the processing logic was idempotent (processing a sensor reading twice was safe, just inefficient), so occasional lock failures during Redis failover were acceptable
22. What is Redis pub/sub?
    - **Answer:**
       - Redis pub/sub allows messages to be broadcast to multiple subscribers
       - In cold-chain, I considered using Redis pub/sub for live dashboard updates but chose Kafka instead because Kafka provides persistence and replay — if the dashboard client disconnected, it would miss messages sent via Redis pub/sub
       - Redis pub/sub is fire-and-forget: messages are not persisted to disk, there is no acknowledgment mechanism, and if no subscriber is connected at the time of publish, the message is lost; this makes it suitable only for ephemeral notifications (e.g., real-time UI alerts) but not for reliable data delivery where message loss would impact correctness
23. What is Redis stream?
    - **Answer:**
       - Redis Streams is an append-only log data structure similar to Kafka topics, with consumer groups and message acknowledgment
       - I evaluated Redis Streams for cold-chain but chose Kafka because Kafka's partition model provides better throughput and horizontal scaling for our volume
       - Redis Streams is a better choice than Kafka when: the setup needs to be simpler (single Redis instance, no separate broker cluster), message volume is lower (< 1000 msg/sec), latency matters more than throughput, and there's no need for the advanced consumer group features Kafka provides (multi-consumer fan-out, offset management across partitions) — for small-to-medium projects, Redis Streams avoids the operational overhead of managing a Kafka cluster
24. How do you secure Redis?
    - **Answer:**
       - I secure Redis by: (1) not exposing it to the public internet (deployed in private subnet), (2) using a strong password via the `requirepass` config, (3) disabling dangerous commands like `FLUSHALL` and `KEYS` via rename-command in redis.conf, and (4) using TLS encryption for data in transit
       - Redis's security model is simple — it trusts the network by default, meaning authentication is optional and connections are unencrypted, so network-level security is critical: I deployed Redis in a private VPC subnet with security groups restricting access to only the application tier, never exposed port 6379 to the internet, and always changed the default configuration (no password, bind to all interfaces) before deploying to any environment
25. How did Redis fit in your project?
    - **Answer:**
       - Redis played three roles in the cold-chain project: (1) cache for API responses (latest temperature readings, gateway lists) to reduce InfluxDB load, (2) session storage for Spring Boot instances to enable horizontal scaling, and (3) distributed locking to prevent duplicate processing of sensor events across multiple consumer instances
       - After adding Redis caching, the dashboard's latest-temperature API latency dropped from 200ms (direct InfluxDB query) to under 5ms (Redis lookup), and InfluxDB query volume reduced by 60% — the session store eliminated sticky sessions, allowing us to scale from 2 to 4 application instances behind the load balancer without any session-related issues, and the distributed lock prevented a race condition where two consumer instances could process the same sensor event simultaneously during a rebalance
