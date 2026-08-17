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
   - **If asked more:**
      - I would explain Redis's single-threaded event loop model and how it achieves sub-millisecond latency by keeping all data in RAM with optional persistence to disk
2. Why use Redis?
   - **Answer:**
      - Redis is fast (in-memory), supports multiple data structures (strings, hashes, lists, sets), and has built-in features like TTL, pub/sub, and distributed locking
      - In cold-chain, I used Redis to cache frequently accessed temperature summaries, reducing InfluxDB query load by about 60%
   - **If asked more:**
      - I would compare Redis with Memcached — Redis's data structure support and persistence options made it more suitable for our session storage and locking needs
3. What data types does Redis support?
   - **Answer:**
      - Redis supports strings, hashes, lists, sets, sorted sets, bitmaps, hyperloglogs, and streams
      - In cold-chain, I used strings for cache entries, hashes for session storage (mapping user ID to multiple session fields), and sorted sets for ranking gateways by temperature deviation
   - **If asked more:**
      - I would explain how I chose the data type based on the access pattern — hashes were perfect for session data because I could read/write individual fields without serializing the entire object
4. What is cache?
   - **Answer:**
      - A cache stores frequently accessed data in a fast storage layer to reduce latency and backend load
      - In cold-chain, Redis cached the latest temperature reading from each sensor so the dashboard could display it instantly without querying InfluxDB every time a user loaded the page
   - **If asked more:**
      - I would explain the cache-aside pattern I used — the application checked Redis first, and only queried InfluxDB on a cache miss, then stored the result in Redis with a TTL
5. What is cache-aside pattern?
   - **Answer:**
      - In cache-aside (lazy loading), the application checks the cache first; on a miss, it loads data from the database, stores it in the cache, and returns it
      - I used this for the latest-sensor-reading API — Redis miss meant a query to InfluxDB, after which the result was cached with a 30-second TTL
   - **If asked more:**
      - I would explain why I chose cache-aside over read-through — it gives the application control over what gets cached and when, and it's simpler to implement without a cache provider abstraction
6. What is read-through cache?
   - **Answer:**
      - In read-through, the cache itself loads data from the database on a miss, transparent to the application
      - Redis doesn't natively support read-through; it requires a cache provider like Spring's `@Cacheable` with `CacheLoader`
      - Spring's cache abstraction handles the miss transparently
   - **If asked more:**
      - I would explain that read-through is simpler for developers but less flexible — you can't control caching logic (e.g., which fields to cache) without implementing a custom `CacheLoader`
7. What is write-through cache?
   - **Answer:**
      - In write-through, data is written to both cache and database in the same transaction
      - I used this pattern for updating gateway status in cold-chain — when a status change was written to MSSQL, the corresponding Redis cache entry was also updated immediately, preventing stale data
   - **If asked more:**
      - I would discuss the trade-off: write-through ensures cache consistency but adds latency to write operations, so I reserved it for critical data where staleness wasn't acceptable
8. What is write-behind cache?
   - **Answer:**
      - In write-behind, data is written to cache immediately and asynchronously persisted to the database later
      - I considered this for sensor reading batching but didn't implement it because losing readings during a Redis crash was unacceptable for the cold-chain compliance requirements
   - **If asked more:**
      - I would explain the risk — if Redis goes down before the async write completes, data is lost
      - I used this pattern only for non-critical data like gateway heartbeat timestamps
9. What is TTL?
   - **Answer:**
      - TTL (Time To Live) is the duration after which a cached entry is automatically evicted
      - In cold-chain, I set a 30-second TTL for sensor-reading cache entries and a 24-hour TTL for partner configuration data that rarely changed
   - **If asked more:**
      - I would explain how I chose TTL values — based on how frequently the data changed and how stale the dashboard could be before users noticed
10. What is cache eviction?
    - **Answer:**
       - Cache eviction removes entries when memory is full
       - Redis supports multiple eviction policies (LRU, LFU, TTL, etc.)
       - I configured Redis with `allkeys-lru` eviction, which removed the least recently used keys when memory reached the `maxmemory` limit, ensuring the most accessed data stayed cached
    - **If asked more:**
       - I would explain why I chose LRU over other policies — cold-chain had a predictable access pattern where recently accessed sensors were more likely to be viewed again
11. What is LRU?
    - **Answer:**
       - LRU (Least Recently Used) evicts entries that haven't been accessed for the longest time
       - Redis's `allkeys-lru` policy approximates LRU
       - In cold-chain, this meant that sensors not being actively monitored were evicted first, while the dashboard's frequently viewed sensors stayed cached
    - **If asked more:**
       - I would explain how Redis's approximate LRU differs from exact LRU — it samples a subset of keys (configurable via maxmemory-samples) rather than tracking all keys, reducing memory overhead
12. What is cache stampede?
    - **Answer:**
       - A cache stampede occurs when many requests simultaneously miss the cache (e.g., after TTL expiry), all hitting the database at once
       - In cold-chain, this happened with the gateways list API — every dashboard refresh caused all users to query InfluxDB when the cache expired
    - **If asked more:**
       - I would explain the impact — InfluxDB CPU spiked to 90% during stampede events, which could delay write operations for new sensor readings
13. How do you prevent cache stampede?
    - **Answer:**
       - I prevent stampedes by: (1) using mutex locks so only one request rebuilds the cache, (2) randomizing TTLs slightly to prevent simultaneous expiry, and (3) using early recomputation — refreshing the cache before it expires
       - In cold-chain, I used a Redis lock to serialize cache rebuilds for the gateways list
    - **If asked more:**
       - I would explain the specific implementation — `jedis.setnx` to acquire the lock, with a 5-second TTL on the lock key to prevent deadlocks if the rebuilding thread crashed
14. What is cache penetration?
    - **Answer:**
       - Cache penetration happens when requests come for data that doesn't exist in either cache or database, causing repeated database misses
       - In cold-chain, if a user requested a non-existent sensor ID, Redis would miss, InfluxDB would miss, and the bad request would bypass caching entirely
    - **If asked more:**
       - I would explain my solution — I cached the null result (or a special "not found" marker) with a short TTL so repeated requests for invalid data didn't hit InfluxDB repeatedly
15. What is cache avalanche?
    - **Answer:**
       - A cache avalanche occurs when many cached entries expire at the same time, causing a flood of requests to the database
       - In cold-chain, if all sensor-reading cache entries had the same 30-second TTL and expired together, InfluxDB would get a sudden load spike
    - **If asked more:**
       - I would explain how I prevented this by adding a random jitter of ±5 seconds to each TTL value, spreading the expiry across a window and preventing simultaneous cache rebuilds
16. How do you invalidate cache?
    - **Answer:**
       - I invalidate cache by: (1) setting appropriate TTLs for automatic expiry, (2) explicitly deleting keys when data changes (write-through), and (3) using a version prefix in cache keys for bulk invalidation
       - In cold-chain, when a partner updated gateway configuration, I deleted the corresponding Redis key so the next read would fetch fresh data
    - **If asked more:**
       - I would explain the version prefix pattern — appending a version number to cache keys (`gateway:123:v2`) so incrementing the version effectively invalidated all old cache entries without iterating through keys
17. What should not be cached?
    - **Answer:**
       - Don't cache: (1) frequently changing data where staleness is unacceptable, (2) sensitive user data that has compliance requirements, (3) data used only once
       - In cold-chain, I didn't cache temperature excursion alerts because they needed to be displayed immediately and the alert TTL was unpredictable
    - **If asked more:**
       - I would explain that caching financial or health-critical data requires careful consideration of how stale values could impact decisions — temperature excursion alerts were real-time critical, so caching added risk without benefit
18. How do you cache API responses?
    - **Answer:**
       - I cache API responses using Spring Boot's `@Cacheable` annotation with Redis as the backing store
       - In cold-chain, I annotated the `getLatestTemperature` method with `@Cacheable(value="sensors", key="#sensorId")`, which automatically cached the response and served it on subsequent calls until the TTL expired
    - **If asked more:**
       - I would explain the cache configuration — I set `spring.cache.redis.time-to-live=30s` and configured the cache key prefix to avoid collisions between different API caches
19. How do you cache database queries?
    - **Answer:**
       - I cache database query results at the service layer using `@Cacheable` around the repository call
       - In cold-chain, the gateway configuration query (which rarely changed) was cached with a 1-hour TTL, reducing MSSQL query load for this frequently accessed but static data
    - **If asked more:**
       - I would explain the risk — if a gateway config changed, the cached version was up to 1 hour stale, so I added a manual cache eviction call in the update endpoint to immediately invalidate the affected entry
20. How do you use Redis for session storage?
    - **Answer:**
       - I configured Spring Session with Redis as the session store, so user sessions were stored in Redis instead of the application's local memory
       - This allowed any instance in the auto-scaling group to serve any user request without losing session state, enabling horizontal scaling
    - **If asked more:**
       - I would explain the configuration — adding `@EnableRedisHttpSession` and the `spring.session.store-type=redis` property, and how session serialization works (Java serialization by default, JSON for cross-language compatibility)
21. How do you use Redis for distributed locking?
    - **Answer:**
       - Distributed locking ensures that only one instance of an application executes a critical section
       - In the cold-chain project, I used Redis `SETNX` (set if not exists) with a TTL to implement a distributed lock that prevented duplicate processing of the same sensor reading across multiple consumer instances
    - **If asked more:**
       - I would discuss the Redlock algorithm for stronger guarantees, but explain that for cold-chain's use case, a single Redis node with SETNX was sufficient since occasional lock failures were acceptable due to the idempotent processing logic
22. What is Redis pub/sub?
    - **Answer:**
       - Redis pub/sub allows messages to be broadcast to multiple subscribers
       - In cold-chain, I considered using Redis pub/sub for live dashboard updates but chose Kafka instead because Kafka provides persistence and replay — if the dashboard client disconnected, it would miss messages sent via Redis pub/sub
    - **If asked more:**
       - I would explain the limitation of pub/sub — messages are fire-and-forget, no persistence, no acknowledgment
       - It's suitable for ephemeral notifications but not for reliable data delivery
23. What is Redis stream?
    - **Answer:**
       - Redis Streams is an append-only log data structure similar to Kafka topics, with consumer groups and message acknowledgment
       - I evaluated Redis Streams for cold-chain but chose Kafka because Kafka's partition model provides better throughput and horizontal scaling for our volume
    - **If asked more:**
       - I would explain when Redis Streams is better than Kafka — simpler setup, lower latency for small volumes (< 1000 msg/sec), and no need for separate broker management
24. How do you secure Redis?
    - **Answer:**
       - I secure Redis by: (1) not exposing it to the public internet (deployed in private subnet), (2) using a strong password via the `requirepass` config, (3) disabling dangerous commands like `FLUSHALL` and `KEYS` via rename-command in redis.conf, and (4) using TLS encryption for data in transit
    - **If asked more:**
       - I would explain that Redis's security model is simple — it trusts the network, so network-level security (VPC, security groups) is critical
       - I never deployed Redis without first changing the default configuration
25. How did Redis fit in your project?
    - **Answer:**
       - Redis played three roles in the cold-chain project: (1) cache for API responses (latest temperature readings, gateway lists) to reduce InfluxDB load, (2) session storage for Spring Boot instances to enable horizontal scaling, and (3) distributed locking to prevent duplicate processing of sensor events across multiple consumer instances
    - **If asked more:**
       - I would give the concrete improvement — after adding Redis caching, the dashboard's latest-temperature API latency dropped from 200ms (InfluxDB query) to under 5ms (Redis lookup), and InfluxDB query volume reduced by 60%
