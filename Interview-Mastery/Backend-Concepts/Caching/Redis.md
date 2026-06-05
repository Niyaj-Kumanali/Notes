# Redis

## 1. Executive Summary

Redis (Remote Dictionary Server) is an open-source, in-memory data structure store used as a database, cache, message broker, and streaming engine. It supports diverse data structures (strings, hashes, lists, sets, sorted sets, bitmaps, hyperloglogs, geospatial indexes, streams) with built-in replication, Lua scripting, LRU eviction, transactions, and disk persistence. Redis is the most widely deployed distributed cache and is a critical component in high-throughput, low-latency backend systems.

## 2. Core Theory

Redis operates as an in-memory key-value store where all data primarily resides in RAM for performance. It follows a single-threaded event loop architecture for command execution, ensuring atomicity for individual commands without locking overhead.

**Key Characteristics:**
- All data in memory (with optional persistence to disk)
- Single-threaded command processing (non-blocking I/O)
- Sub-millisecond latency for most operations
- Rich data structures beyond simple key-value
- Master-slave replication for read scalability
- Built-in clustering for horizontal sharding
- Pub/sub messaging and stream processing

**Data Structures:**
| Structure | Description | Use Case |
|-----------|-------------|----------|
| String | Binary safe string (max 512MB) | Cache values, counters, sessions |
| Hash | Field-value pairs | Object representation, user profiles |
| List | Ordered collection of strings | Message queues, timelines |
| Set | Unordered unique strings | Tags, relationships, deduplication |
| Sorted Set | Ordered by score | Leaderboards, rate limits |
| Stream | Append-only log | Event sourcing, messaging |
| Bitmap | Bit-level operations | Feature flags, analytics |
| HyperLogLog | Probabilistic cardinality | Unique count estimates |
| Geospatial | Location-based operations | Nearby queries |

## 3. Under-the-Hood Deep Dive

### Single-Threaded Event Loop

Redis uses a single-threaded reactor pattern (using epoll/kqueue on Unix, IOCP on Windows via RedisLabs). All commands are executed sequentially, providing:

- Atomic execution of single commands
- No race conditions within a command
- No context switching overhead
- Simplified codebase and predictable performance

**Caveats:** Slow commands (KEYS, SMEMBERS on large sets) block all other operations.

### Persistence Mechanisms

**RDB (Redis Database File):** Point-in-time snapshots at configurable intervals.
- Triggered by `SAVE` (sync), `BGSAVE` (fork + async write)
- Pros: Compact, fast recovery, good for disaster recovery
- Cons: Data loss between snapshots, fork can cause latency spikes

**AOF (Append-Only File):** Logs every write operation.
- fsync policies: always (1 op), everysec (default), no (OS-managed)
- Pros: Durability (minimal data loss), human-readable
- Cons: Larger file size, slower recovery than RDB

**Mixed:** Redis 4.0+ supports RDB + AOF hybrid (RDB base + AOF incremental).

### Memory Management

- `maxmemory` limits total memory usage.
- Eviction policies when limit is reached:
  - noeviction: Return errors on writes.
  - allkeys-lru: Evict least recently used keys.
  - allkeys-lfu: Evict least frequently used keys.
  - volatile-lru: Evict LRU among keys with TTL.
  - volatile-lfu: Evict LFU among keys with TTL.
  - allkeys-random: Random eviction.
  - volatile-random: Random eviction among keys with TTL.
  - volatile-ttl: Evict keys with shortest TTL.

### Replication

Redis uses asynchronous master-replica replication:
1. Replica connects to master and issues PSYNC.
2. Master forks, creates RDB snapshot, sends to replica.
3. Master buffers all writes during snapshot transfer.
4. Replica loads RDB, requests buffered writes.
5. After sync, master streams write commands to replica.
6. Replica acknowledges applied offsets.

### Redis Cluster

Automatic sharding across up to 1000 nodes:
- Uses 16384 hash slots; keys are hashed via CRC16.
- Each node owns a subset of slots.
- Smart clients calculate which node owns a key.
- Nodes gossip to detect failures and propagate configuration.
- Replicas provide failover.

## 4. Production Code Examples

### Spring Boot Redis Configuration

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.port}")
    private int redisPort;

    @Value("${spring.redis.password}")
    private String redisPassword;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration(redisHost, redisPort);
        config.setPassword(redisPassword);
        config.setDatabase(0);

        LettuceConnectionFactory factory = new LettuceConnectionFactory(config);
        factory.setValidateConnection(true);
        return factory;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(Map.of(
                "users", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(60)),
                "sessions", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(15)),
                "products", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofHours(1))
            ))
            .transactionAware()
            .build();
    }
}
```

### Redis Cluster Configuration

```java
@Configuration
public class RedisClusterConfig {

    @Bean
    public RedisConnectionFactory redisClusterConnectionFactory() {
        List<String> clusterNodes = List.of(
            "redis-node1:6379", "redis-node2:6379", "redis-node3:6379",
            "redis-node4:6379", "redis-node5:6379", "redis-node6:6379"
        );

        RedisClusterConfiguration clusterConfig = new RedisClusterConfiguration(clusterNodes);
        clusterConfig.setMaxRedirects(3);
        clusterConfig.setPassword("cluster-password");

        return new LettuceConnectionFactory(clusterConfig);
    }
}
```

### Redis Sentinel Configuration for High Availability

```java
@Configuration
public class RedisSentinelConfig {

    @Bean
    public RedisConnectionFactory sentinelConnectionFactory() {
        RedisSentinelConfiguration sentinelConfig = new RedisSentinelConfiguration()
            .master("mymaster")
            .sentinel("sentinel1", 26379)
            .sentinel("sentinel2", 26379)
            .sentinel("sentinel3", 26379);

        return new LettuceConnectionFactory(sentinelConfig);
    }
}
```

### Using Redis Data Structures

```java
@Service
public class RedisDataStructureService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // String operations
    public void setValue(String key, String value) {
        redisTemplate.opsForValue().set(key, value);
    }

    public String getValue(String key) {
        return (String) redisTemplate.opsForValue().get(key);
    }

    public Long increment(String key) {
        return redisTemplate.opsForValue().increment(key);
    }

    // Hash operations - represent objects
    public void setUserSession(String sessionId, Map<String, Object> sessionData) {
        redisTemplate.opsForHash().putAll("session:" + sessionId, sessionData);
        redisTemplate.expire("session:" + sessionId, Duration.ofMinutes(30));
    }

    public Map<Object, Object> getUserSession(String sessionId) {
        return redisTemplate.opsForHash().entries("session:" + sessionId);
    }

    // List operations - message queue
    public void pushToQueue(String queueName, Object message) {
        redisTemplate.opsForList().rightPush("queue:" + queueName, message);
    }

    public Object popFromQueue(String queueName) {
        return redisTemplate.opsForList().leftPop("queue:" + queueName);
    }

    public List<Object> batchPop(String queueName, int count) {
        List<Object> batch = new ArrayList<>();
        for (int i = 0; i < count; i++) {
            Object item = redisTemplate.opsForList().leftPop("queue:" + queueName);
            if (item == null) break;
            batch.add(item);
        }
        return batch;
    }

    // Set operations - unique tags
    public void addTags(String articleId, String... tags) {
        redisTemplate.opsForSet().add("article:tags:" + articleId, (Object[]) tags);
    }

    public Set<Object> getTags(String articleId) {
        return redisTemplate.opsForSet().members("article:tags:" + articleId);
    }

    public Set<Object> intersectTags(String... articleIds) {
        String[] keys = Arrays.stream(articleIds)
            .map(id -> "article:tags:" + id)
            .toArray(String[]::new);
        return redisTemplate.opsForSet().intersect(Arrays.asList(keys));
    }

    // Sorted Set operations - leaderboard
    public void updateScore(String leaderboard, String playerId, double score) {
        redisTemplate.opsForZSet().add("leaderboard:" + leaderboard, playerId, score);
    }

    public Set<Object> getTopPlayers(String leaderboard, int count) {
        return redisTemplate.opsForZSet()
            .reverseRange("leaderboard:" + leaderboard, 0, count - 1);
    }

    public Long getRank(String leaderboard, String playerId) {
        return redisTemplate.opsForZSet()
            .reverseRank("leaderboard:" + leaderboard, playerId);
    }
}
```

### Distributed Lock with Redisson

```java
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
            .setAddress("redis://localhost:6379")
            .setPassword("password")
            .setConnectionPoolSize(10)
            .setConnectionMinimumIdleSize(5);

        return Redisson.create(config);
    }
}

@Service
public class DistributedLockService {

    @Autowired
    private RedissonClient redissonClient;

    public void executeWithLock(String lockKey, Runnable action) {
        RLock lock = redissonClient.getLock(lockKey);
        try {
            // Wait up to 10s, auto-release after 30s
            if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
                action.run();
            } else {
                throw new LockAcquisitionException("Could not acquire lock: " + lockKey);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockAcquisitionException("Lock acquisition interrupted", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### Rate Limiter using Redis + Lua Script

```java
@Service
public class RateLimiterService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // Lua script for sliding window rate limiting
    private static final String SLIDING_WINDOW_SCRIPT =
        "local key = KEYS[1]\n" +
        "local now = tonumber(ARGV[1])\n" +
        "local window = tonumber(ARGV[2])\n" +
        "local limit = tonumber(ARGV[3])\n" +
        "redis.call('ZREMRANGEBYSCORE', key, 0, now - window)\n" +
        "local count = redis.call('ZCARD', key)\n" +
        "if count < limit then\n" +
        "   redis.call('ZADD', key, now, now .. ':' .. math.random())\n" +
        "   redis.call('EXPIRE', key, window)\n" +
        "   return 1\n" +
        "end\n" +
        "return 0";

    public boolean isAllowed(String userId, String action, int limit, int windowSeconds) {
        String key = "ratelimit:" + userId + ":" + action;
        long now = System.currentTimeMillis();

        DefaultRedisScript<Long> script = new DefaultRedisScript<>(SLIDING_WINDOW_SCRIPT, Long.class);
        Long result = redisTemplate.execute(script, List.of(key), now, windowSeconds, limit);

        return result != null && result == 1;
    }
}
```

### Redis Stream for Event Processing

```java
@Service
public class EventStreamService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    private static final String STREAM_KEY = "events:order";

    public String publishEvent(OrderEvent event) {
        Map<String, Object> body = new HashMap<>();
        body.put("orderId", event.getOrderId());
        body.put("type", event.getType());
        body.put("timestamp", event.getTimestamp().toString());
        body.put("data", event.getData());

        ObjectRecord record = StreamRecords.newRecord()
            .in(STREAM_KEY)
            .ofObject(body)
            .withId(RecordId.autoGenerate());

        RecordId messageId = redisTemplate.opsForStream().add(record);
        return messageId.getValue();
    }

    public List<MapRecord<String, Object, Object>> readEvents(String consumerGroup, 
            String consumerName, int count) {
        StreamOffset<String> offset = StreamOffset.create(STREAM_KEY, ReadOffset.lastConsumed());

        List<MapRecord<String, Object, Object>> messages = redisTemplate
            .opsForStream()
            .read(Consumer.from(consumerGroup, consumerName), offset);

        return messages;
    }

    public void acknowledge(String consumerGroup, String messageId) {
        redisTemplate.opsForStream()
            .acknowledge(STREAM_KEY, consumerGroup, messageId);
    }
}
```

## 5. Real-World Scenarios

### Session Store for Spring Boot Application
- Use Spring Session backed by Redis for distributed session management.
- Serialize session attributes as JSON.
- Session TTL: 30 minutes, refreshed on each request.
- Redis Cluster for HA across availability zones.

### Real-Time Leaderboard
- Gaming platform with 10M+ users.
- Redis Sorted Sets: `leaderboard:daily`, `leaderboard:weekly`, `leaderboard:alltime`.
- Update: `ZINCRBY leaderboard:daily userId 1` per action.
- Read top 100: `ZREVRANGE leaderboard:daily 0 99 WITHSCORES`.
- Periodic snapshot to MySQL for historical data.

### Message Queue
- Use Redis List (LPUSH/BRPOP) for simple task queues.
- Use Redis Stream for consumer groups with acknowledgment.
- Priority queues via multiple lists (high/medium/low) + BRPOP with timeout.

### Distributed Rate Limiting
- Per-user API rate limiting using sliding window (sorted sets).
- Per-IP rate limiting for DDoS protection.
- Token bucket algorithm using Lua scripts for atomicity.

### Geospatial Queries
- Find nearby restaurants: `GEOADD restaurants 13.361389 38.115556 "Palermo"`.
- Search 5km radius: `GEORADIUS restaurants 13.36 38.11 5 km`.
- Used in food delivery apps, ride sharing.

### Real-Time Analytics
- HyperLogLog for daily unique visitor counts: `PFADD daily:visitors:2024-01-01 userId1`.
- Bitmaps for tracking user daily activity (365-bit per user).
- Counters via INCR for page views, likes, shares.

## 6. Performance

### Benchmarking

| Operation | Single Instance (1ms latency) | Cluster (3 nodes) | Pipeline (100 cmds) |
|-----------|------------------------------|-------------------|---------------------|
| SET | ~50,000 ops/s | ~150,000 ops/s | ~1,000,000 ops/s |
| GET | ~50,000 ops/s | ~150,000 ops/s | ~1,000,000 ops/s |
| INCR | ~48,000 ops/s | ~144,000 ops/s | ~950,000 ops/s |
| LPUSH | ~45,000 ops/s | ~135,000 ops/s | ~900,000 ops/s |
| LRANGE (100) | ~30,000 ops/s | ~90,000 ops/s | ~600,000 ops/s |

### Optimization Techniques

**Pipelining:** Batch multiple commands to reduce round-trips.
```java
List<Object> results = redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
    connection.stringCommands().set("key1".getBytes(), "value1".getBytes());
    connection.stringCommands().set("key2".getBytes(), "value2".getBytes());
    connection.stringCommands().get("key1".getBytes());
    return null;
});
```

**Connection Pooling:**
```yaml
spring:
  redis:
    lettuce:
      pool:
        max-active: 16
        max-idle: 8
        min-idle: 4
        max-wait: 200ms
```

**Avoid Expensive Commands:**
- KEYS (scan all keys) -> use SCAN with cursor.
- SMEMBERS (large sets) -> use SSCAN.
- HGETALL (large hashes) -> use HSCAN.
- SORT with GET/BY -> use sorted sets.

**Memory Optimization:**
- Use Redis Hash for objects instead of many String keys.
- Enable compression (Snappy/Zstd) for large values.
- Use shorter key names but maintain readability.
- Redis 7.4+ uses less memory with better hash encoding.

## 7. Security

### Authentication
```bash
# In redis.conf
requirepass your-strong-password

# Or via ACL (Redis 6+)
ACL SETUSER alice on >strong-password ~cached:* +@read -@write
ACL SETUSER bob on >another-password ~* +@all -@dangerous
```

### Network Security
- Bind to specific interfaces: `bind 127.0.0.1 10.0.0.1`
- Disable dangerous commands: `rename-command FLUSHALL ""`
- Use TLS: `tls-port 6380`, `tls-cert-file`, `tls-key-file`
- Deploy in private VPC, never expose to public internet.
- Use network policies (firewall, security groups).

### Spring Boot Redis Security Configuration
```java
@Bean
public RedisConnectionFactory secureRedisConnectionFactory() {
    RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
    config.setHostName("redis.internal");
    config.setPort(6380);
    config.setPassword("encrypted-password");
    config.setUsername("app-user"); // Redis ACL

    // SSL/TLS configuration
    LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
        .useSsl()
        .sslVerifyMode(SslVerifyMode.VERIFY)
        .build();

    return new LettuceConnectionFactory(config, clientConfig);
}
```

### Best Practices
- Never store plaintext secrets in Redis.
- Use ACLs to restrict user permissions.
- Enable TLS for data in transit.
- Rotate passwords regularly.
- Audit logs for admin commands.
- Run Redis as non-root user.

## 8. Common Mistakes

### Mistake 1: Not Setting TTL
```java
// WRONG - memory leak
redisTemplate.opsForValue().set("key", value);

// RIGHT
redisTemplate.opsForValue().set("key", value, Duration.ofHours(1));
```

### Mistake 2: Using KEYS in Production
```java
// WRONG - blocks Redis for millions of keys
Set<String> keys = redisTemplate.keys("user:*");

// RIGHT - use SCAN
Set<String> keys = new HashSet<>();
ScanOptions options = ScanOptions.scanOptions().match("user:*").count(100).build();
try (Cursor<String> cursor = redisTemplate.scan(options)) {
    while (cursor.hasNext()) {
        keys.add(cursor.next());
    }
}
```

### Mistake 3: Large Values (>10MB)
Storing large values degrades Redis performance and network throughput.
- Consider object store (S3) for large blobs.
- Compress values before storing.
- Split large data into smaller chunks.

### Mistake 4: Ignoring Connection Management
```java
// WRONG - connection leak
RedisConnection conn = redisTemplate.getConnectionFactory().getConnection();
// ... forget to close

// RIGHT - use try-with-resources or template
redisTemplate.execute((RedisCallback<String>) connection -> {
    // connection is managed by Lettuce
    return new String(connection.stringCommands().get("key".getBytes()));
});
```

### Mistake 5: Blocking Operations in Transaction
Avoid Redis transactions (MULTI/EXEC) wrapping blocking operations (BLPOP, BRPOP).

## 9. Senior Engineer Perspective

### High Availability Architecture

```
[Client Apps]
     |
[Load Balancer (HAProxy)]
     |
[Redis Sentinel Cluster]  <----> [Redis Master]
     |                              |
     |                    [Replica 1] [Replica 2]
     |
   [Sentinel 1] [Sentinel 2] [Sentinel 3]
```

- Minimum 3 Sentinel instances for quorum.
- Replicas for read scaling and failover.
- Client-side: Lettuce supports automatic discovery via Sentinels.

### Redis Cluster Design

```
[App] -> [Redis Cluster]
            |
     [Node 1: Slots 0-5460]
     [Node 2: Slots 5461-10922]
     [Node 3: Slots 10923-16383]
            |
     [Replica A] [Replica B] [Replica C]
```

- Always use replicas for availability (not just performance).
- Deploy nodes across availability zones.
- Hash tags `{user:123}:profile` to force key affinity to same slot.

### Performance Tuning for Redis
```bash
# redis.conf tuning
maxmemory 4gb
maxmemory-policy allkeys-lfu
save 900 1          # RDB: 15 min if 1 change
save 300 10         # RDB: 5 min if 10 changes
appendonly yes
appendfsync everysec
hz 10               # Adjust for more frequent cleanup
lfu-log-factor 10
lfu-decay-time 1
```

### Cache-Aside with Redis + Local Cache
```java
@Service
public class HybridCacheService<K, V> {

    private final Cache<K, V> localCache; // Caffeine
    private final RedisTemplate<String, Object> redisTemplate;
    private final Class<V> type;
    private final String keyPrefix;

    public V get(K key) {
        String redisKey = keyPrefix + key;

        // Check local cache
        V value = localCache.getIfPresent(key);
        if (value != null) return value;

        // Check Redis
        value = (V) redisTemplate.opsForValue().get(redisKey);
        if (value != null) {
            localCache.put(key, value);
            return value;
        }

        return null;
    }

    public void put(K key, V value, Duration ttl) {
        String redisKey = keyPrefix + key;
        redisTemplate.opsForValue().set(redisKey, value, ttl);
        localCache.put(key, value);
    }

    public void evict(K key) {
        String redisKey = keyPrefix + key;
        redisTemplate.delete(redisKey);
        localCache.invalidate(key);
    }
}
```

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What does Redis stand for?
   **A:** Remote Dictionary Server.

2. **Q:** Is Redis single-threaded or multi-threaded?
   **A:** Command execution is single-threaded. I/O operations use multiple threads (Redis 6+). Background tasks (BGSAVE, replication) use child processes.

3. **Q:** What data structures does Redis support?
   **A:** Strings, Hashes, Lists, Sets, Sorted Sets, Streams, Bitmaps, HyperLogLogs, Geospatial indexes.

4. **Q:** What is Redis persistence?
   **A:** Redis offers RDB (point-in-time snapshots) and AOF (append-only command log) for persisting data to disk.

5. **Q:** What is the difference between Redis and Memcached?
   **A:** Redis supports richer data structures, persistence, replication, clustering, pub/sub, and Lua scripting. Memcached is simpler, multi-threaded, and only supports key-value strings.

6. **Q:** What is TTL in Redis?
   **A:** Time-To-Live, the expiration time of a key. After TTL expires, the key is automatically deleted.

7. **Q:** How does Redis handle key expiration?
   **A:** Redis uses passive expiration (key accessed when expired) and active expiration (background cron samples keys every 100ms).

8. **Q:** What is Redis pub/sub?
   **A:** A messaging pattern where publishers send messages to channels, and subscribers receive messages from channels they subscribe to. Messages are not persisted.

9. **Q:** How do you connect Spring Boot to Redis?
   **A:** Add `spring-boot-starter-data-redis` dependency, configure connection properties in `application.yml`. Spring Boot auto-configures `RedisTemplate` and `RedisConnectionFactory`.

10. **Q:** What is the default Redis port?
    **A:** 6379.

### Medium

11. **Q:** Explain Redis sentinel and its role.
    **A:** Redis Sentinel provides high availability: monitoring master/replicas, automatic failover (promoting replica when master fails), notification, and service discovery for clients.

12. **Q:** What is Redis Cluster and how does it shard data?
    **A:** Redis Cluster automatically shards data across multiple nodes using 16384 hash slots. CRC16(key) % 16384 determines the slot. Each node owns a subset of slots.

13. **Q:** What is a Redis lock and how do you implement it properly?
    **A:** Use `SET key value NX EX 10` (set if not exists, expire 10s). Redlock algorithm for distributed environments. Redisson provides robust distributed lock implementation.

14. **Q:** Explain the difference between Redis Transactions and Lua scripting.
    **A:** Transactions (MULTI/EXEC) batch commands and guarantee isolation but no rollback. Lua scripts execute atomically and can include logic (if/else loops). Prefer Lua for complex atomic operations.

15. **Q:** What are Redis Streams and how are they different from pub/sub?
    **A:** Streams persist messages, support consumer groups, allow replay of messages, and track acknowledgment. Pub/sub is fire-and-forget with no persistence.

16. **Q:** How does Redis memory eviction work?
    **A:** When `maxmemory` is reached, Redis applies the configured eviction policy (e.g., allkeys-lru) to remove keys and free memory.

17. **Q:** What is the WRITE-BEHIND pattern with Redis?
    **A:** Data is written to Redis immediately and asynchronously persisted to the database. Redis buffers writes and a background process flushes them to DB.

18. **Q:** Explain the difference between Lettuce and Jedis.
    **A:** Jedis is a blocking Redis client (one connection per thread). Lettuce is asynchronous/non-blocking (netty-based, connection shared across threads). Spring Boot defaults to Lettuce.

19. **Q:** How do you monitor Redis performance?
    **A:** Use `INFO` command, `redis-cli --stat`, `MONITOR` (debug only), `SLOWLOG GET 100`, RedisInsight, or Prometheus with redis_exporter.

20. **Q:** What is cache warmup and how would you do it with Redis?
    **A:** Pre-loading frequently accessed data into Redis before production traffic hits. Use Spring's `CommandLineRunner` to query DB and populate Redis with pipelining.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Implement a distributed rate limiter using Redis that supports burst traffic.
    **A:** Use token bucket algorithm with Lua script:
    ```lua
    local key = KEYS[1]
    local rate = tonumber(ARGV[1])
    local capacity = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])
    local tokens = redis.call('HGETALL', key)
    local last_tokens = capacity
    local last_refreshed = now
    if #tokens > 0 then
        last_tokens = tonumber(tokens[2])
        last_refreshed = tonumber(tokens[4])
    end
    local delta = math.max(0, now - last_refreshed)
    local new_tokens = math.min(capacity, last_tokens + delta * rate / 1000)
    local allowed = 0
    if new_tokens >= 1 then
        new_tokens = new_tokens - 1
        allowed = 1
    end
    redis.call('HMSET', key, 'tokens', new_tokens, 'last_refreshed', now)
    redis.call('EXPIRE', key, 10)
    return allowed
    ```

2. **Q:** How do you handle Redis hot keys in production?
    **A:** Identify via `redis-cli --hotkeys` or custom monitoring. Mitigations: local cache on app side, replicate hot key to multiple shards with key-suffix (e.g., `key:0` to `key:N`), use Redis read replicas, or split into sub-keys.

3. **Q:** Explain the Redlock algorithm and its trade-offs.
    **A:** Redlock acquires locks from N independent Redis nodes (typically 5). Lock is acquired if majority (>N/2) respond within timeout. Trade-offs: relies on synchronized clocks, vulnerable to long GC pauses, complexity. For most systems, single-instance Redis lock with sentinel is sufficient.

4. **Q:** How do you implement a Redis-based priority queue?
    **A:** Use Sorted Sets with priority as score. Enqueue: `ZADD queue:priority priorityValue taskId`. Dequeue: `ZPOPMIN queue:priority 1` (lowest score first). For multiple priority levels, use multiple sorted sets and dequeue from highest priority first.

5. **Q:** Design a Redis-based session store that survives regional failover.
    **A:** Use Redis Cluster with cross-region replication (Active-Passive). Writes go to primary region Redis. Replicate asynchronously to secondary region via Redis replication or Kafka-based replication. On failover, DNS changes point to secondary Redis. For active-active, use CRDTs or application-level conflict resolution.

6. **Q:** How do you migrate 100GB of Redis data without downtime?
    **A:** Options: 1) Set up replication from old to new instance, wait for sync, then switch clients. 2) Use RedisShake or similar for incremental sync. 3) Dual-write to both old and new, gradually migrate reads. 4) Dump RDB, transfer, load on new instance while handling incremental writes via replication buffer.

7. **Q:** Explain the concept of Redis Cluster hash tags and their use in multi-key operations.
    **A:** Hash tags (`{...}`) ensure keys with same tag map to same hash slot. Example: `user:{123}:profile` and `user:{123}:settings` both go to same node, enabling multi-key operations like SUNION or transaction.

8. **Q:** How do you implement a distributed counter that can handle 1M+ updates per second?
    **A:** Use local counters per application instance that batch-flush to Redis. Local INCR on JVM LongAdder. Send aggregated delta to Redis every 100ms via `INCRBY`. Redis cluster with multiple nodes. For real-time accuracy, use Redis atomic INCR with pipelining.

9. **Q:** What are the pitfalls of using Redis pub/sub for critical message delivery?
    **A:** Pub/sub is fire-and-forget: messages lost if subscriber is offline, no backpressure, no acknowledgment. For reliable messaging, use Redis Streams (persistent, consumer groups, ACK).

10. **Q:** How does Redis handle concurrent writes from multiple clients?
    **A:** Redis commands are serialized by single-threaded event loop, so individual commands are atomic. For multi-command atomicity, use transactions (MULTI/EXEC), Lua scripts, or optimistic locking via WATCH.

### System Design

11. **Q:** Design a real-time leaderboard for a game with 100M users.
    **A:** Redis Sorted Set per leaderboard period (daily/weekly/all-time). Update score with ZINCRBY. For top 100, use ZREVRANGE with caching (refresh every 10s). For user rank, use ZREVRANK. Shard by region if needed. Snapshot to relational DB every hour for history. Use Redis read replicas for read-heavy load (99% reads).

12. **Q:** Design a Redis-backed session management system for a large e-commerce platform.
    **A:** Redis Cluster with 6 nodes (3 master + 3 replica). Session key: `session:{sessionId}`. Session data as JSON hash. TTL 30 min, auto-refresh on touch. Serialization: Protobuf for efficiency. For high availability, deploy replicas per AZ. For disaster recovery, cross-region AOF replication.

13. **Q:** Design a caching layer for a social media platform using Redis.
    **A:** L1: local Caffeine (hot data). L2: Redis Cluster (shared). Data types: Strings for user profiles (TTL 1h), Sorted Sets for feed timelines (TTL 30 min), Sets for followers/following, Streams for notification pipelines. Write-Behind for post persistence. Pub/sub for real-time updates to connected clients.

14. **Q:** Design a distributed job scheduler using Redis.
    **A:** Sorted Set for scheduled tasks with score=nextRunTimestamp. Workers poll `ZRANGEBYSCORE key -inf now WITHSCORES LIMIT 0 1`, atomically move to processing set via Lua script. After completion, update score to next run time. Handle failures by rescheduling. Streams for immediate job dispatch.

15. **Q:** Design a Redis-based message queue for an order processing system.
    **A:** Redis Streams: order events as stream entries with consumer groups. Shard by order ID hash tag. Consumer groups for order-processing, notification, analytics. Use PEL (Pending Entry List) for retry. Dead letter stream for failed messages. Acknowledge after successful processing.

16. **Q:** Design a real-time analytics system using Redis HyperLogLog and Bitmaps.
    **A:** HyperLogLog for unique visitors per page (daily, hourly). PFADD + PFCOUNT. Bitmaps for daily active users: `SETBIT active:2024-01-01 userId 1`. Traffic sources: INCR counters per referrer. Top pages: Sorted Set with score=view count. Aggregate with PFMERGE and BITOP for cross-period analysis.

17. **Q:** Design a globally distributed Redis caching system for a low-latency application.
    **A:** Active-Passive with local read replicas per region. Write to primary region (e.g., us-east-1). Replicate to eu-west-1, ap-southeast-1 via Redis replication or Kafka MirrorMaker. Reads served from nearest region replica. Consistency: monotonic reads via version vector. Failover: promote local replica to master.

18. **Q:** Design a Redis-based feature flag system for a SaaS product.
    **A:** Redis Hashes for feature flags per tenant/organization. Key: `feature:{orgId}`. Fields: feature names, values: JSON with rollout percentage. Pub/sub for real-time flag updates. Local cache with 10s TTL on app side to reduce Redis calls. SDK with polling or WebSocket for flag updates.

19. **Q:** Design a distributed cache with Redis and local caching for a high-traffic API.
    **A:** Two-level: Caffeine L1 (100MB per node, LRU, TTL 60s) + Redis L2 (10GB, LFU). Cache-Aside pattern. On L1 miss, check L2. On L2 miss, load from DB. Writes invalidate both L1 (local) and L2 (Redis pub/sub to all nodes). This hybrid approach reduces Redis load by 80% for hot keys.

20. **Q:** Design a Redis-backed shopping cart service.
    **A:** Redis Hash per user: `cart:{userId}`. Fields: `productId:quantity`. Cart merge on login (persistent cart -> Redis cart). TTL 7 days. Write-Behind to DB on checkout. Pipeline for batch operations. Redis Streams for cart abandonment detection (expiry event via keyspace notification).

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a multi-region active-active Redis architecture with conflict resolution.
    **A:** Use Redis CRDT (Conflict-free Replicated Data Types) or application-level CRDTs on top of Redis. Each region has full read/write Redis. Asynchronous replication between regions. Conflicts resolved via: last-writer-wins (timestamps), observed-remove sets (for collections), or application-defined merge functions. Use hybrid logical clocks (HLC) for causality. Monitor divergence with Merkle trees for periodic reconciliation.

2. **Q:** How would you design a Redis-based data store with linearizable consistency?
    **A:** Implement Raft consensus on top of Redis: one Redis node as leader, others as followers. All writes go through leader, replicated to majority before acknowledging. Reads from leader for strong consistency, or from followers with bounded staleness. This sacrifices throughput for guarantees. For practical alternatives, use Redis with WATCH + MULTI/EXEC for optimistic locking (CAS semantics).

3. **Q:** Design a Redis cluster auto-scaler that handles rebalancing with zero downtime.
    **A:** Monitor CPU/memory/network per node. When threshold exceeded, trigger scaling: 1) Add new nodes to cluster. 2) Move hash slots from hot nodes to new nodes using MIGRATE. 3) Use resharding in small batches (100 slots at a time). 4) Rate-limit migrations to avoid impact. 5) Rollback if migration fails. Use consistent hashing with virtual nodes for minimal key redistribution.

4. **Q:** How would you implement a Redis-based global rate limiter for a multi-region API gateway?
    **A:** Hierarchical: per-region local rate limiter (token bucket in memory, sync to Redis every 100ms) + global rate limiter in Redis. Global allows "borrowing" from regional quotas. Redis CRDT counter for eventual consistency of global count. Algorithm: leaky bucket per user/IP across all regions. TTL-based reset for window counters.

5. **Q:** Design a Redis caching layer for a financial risk calculation engine requiring sub-millisecond reads and absolute consistency.
    **A:** Redis on same host (Unix socket, no network latency) for L1. Persistent Redis Cluster with strong consistency for L2 via WAIT command (replica acknowledgment). Write-through pattern. Every write waits for N replica acknowledgments before responding. Use monotonic clocks for versioning. Circuit breaker: if Redis latency > 1ms degrade to direct DB read. All cache mutations are logged to audit trail.

6. **Q:** Propose a Redis deployment strategy for a Kubernetes-based microservices platform with 200+ services.
    **A:** Operator pattern: Redis Operator (e.g., KubeDB, Redis Operator) manages Redis Cluster CRDs. Per-domain clusters: `user-redis`, `order-redis`, `analytics-redis`. Shared infrastructure cluster for cross-domain data. Sidecar pattern: Redis sidecar per pod for local caching (L1), talking to central Redis for L2. Service mesh for Redis traffic (mTLS). GitOps for Redis config management.

7. **Q:** How do you implement ACID transactions across Redis and a relational database?
    **A:** Two-phase commit (2PC) with Redis as coordinator: 1) Prepare phase: Redis writes to local store + DB executes and holds. 2) Commit phase: if both prepared, commit both. Rollback if either fails. Practical alternative: Saga pattern with compensating transactions. For less strict, use CDC (Debezium) to capture DB changes and propagate to Redis.

8. **Q:** Design a Redis-based event sourcing system that can replay years of events within minutes.
    **A:** Redis Streams for recent events (7 days). Object store (S3) for archived events by year/month. Partition streams by aggregate type and ID. Store snapshots in Redis Hash every N events (e.g., 100). Replay: load latest snapshot, then replay subsequent events from stream. For full history replay, load archived events from S3 sequentially. Use parallel stream processing per aggregate ID.

9. **Q:** How do you design and implement a Redis proxy that provides consistent hashing, circuit breaking, and query routing?
    **A:** Proxy layer (Twemproxy, RedisLabs proxy, or custom Envoy filter): 1) Consistent hashing on key for node selection. 2) Circuit breaker: track error rate per backend node, trip when >50% errors. 3) Query routing: parse command, route based on key hash slot. 4) Connection pooling to backend nodes. 5) Health checks and automatic failover. 6) Observability: metrics per command type, latency percentiles.

10. **Q:** Design a Redis-based cache for an AI/ML feature store requiring both low latency (<5ms) and high throughput (1M req/s) with 100GB of features.
    **A:** Sharded Redis Cluster with 20 nodes (10 master, 10 replica). Key: `feature:{namespace}:{entityId}`. Value: Protocol Buffers encoded feature vector. Pipelining for batch feature retrieval. Local L1 cache on ML inference servers (Caffeine, 10K entries, 30s TTL) for hot entities. Feature pre-computation writes to Redis via streaming pipeline (Flink -> Redis sink). TTL-based eviction for feature staleness. Redis on NVMe instance types for larger capacity.

## 13. Debugging & Troubleshooting

### Common Redis Issues

**Issue: Redis latency spikes**
```bash
redis-cli --latency -h host -p 6379
redis-cli SLOWLOG GET 50
```
Check: BGSAVE causing fork, high network latency, large key operations, swap usage, or CPU contention.

**Issue: Connection refused**
```bash
redis-cli -h host -p 6379 ping
# PONG = OK, connection refused = port blocked or Redis not running
```
Check: firewall rules, Redis bind address, Redis process alive, systemd status.

**Issue: OOM command not allowed**
- Redis `maxmemory` reached with `noeviction` policy.
- Increase maxmemory or change eviction policy.

**Issue: Master link constantly going down**
```bash
redis-cli INFO replication
# Check master_link_status, slave_repl_offset
```
Check: network stability, replication buffer size, master/slave timeout.

**Issue: Key disappears unexpectedly**
- Check TTL: `redis-cli TTL key`
- Check eviction: `redis-cli INFO stats | grep evicted_keys`
- Check if another process deletes keys.

**Issue: High memory fragmentation**
```bash
redis-cli INFO memory | grep mem_fragmentation_ratio
# ratio > 1.5 indicates fragmentation. Restart or use MEMORY PURGE.
```

### Spring Boot Redis Debugging
```yaml
logging:
  level:
    org.springframework.data.redis: DEBUG
    io.lettuce.core: DEBUG
    org.springframework.cache: TRACE
```

### Redis Command Monitoring
```bash
# Show slow queries (slower than 5ms)
redis-cli CONFIG SET slowlog-log-slower-than 5000
redis-cli SLOWLOG GET 100

# Monitor all commands (DEBUG ONLY - performance killer)
redis-cli MONITOR

# Track latency
redis-cli --latency -h myredis.internal -p 6379
redis-cli --latency-dist

# Big key analysis
redis-cli --bigkeys
```

## 14. Comparison Section

### Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data Types | Strings, Lists, Sets, Sorted Sets, Hashes, Streams, etc. | Simple strings only |
| Persistence | RDB, AOF | None |
| Replication | Master-slave, Cluster | No native replication |
| Transactions | MULTI/EXEC, Lua scripts | No |
| Pub/Sub | Yes | No |
| L1 Cache | No (but used as such) | No |
| Memory Efficiency | Higher (compression) | Lower (slab allocation) |
| Threading | Single-threaded commands | Multi-threaded |
| Use Case | Caching + data structures + messaging | Simple caching only |

### Redis vs Hazelcast

| Feature | Redis | Hazelcast |
|---------|-------|-----------|
| Architecture | Client-server | Peer-to-peer (embedded) |
| Data Structures | 10+ | Similar |
| Compute | Lua scripting | Java execution (distributed executor) |
| Persistence | RDB/AOF | MapStore/MapLoader |
| Query | Secondary index (RediSearch) | SQL-like queries |
| Deployment | External cluster | Embedded in JVM |

### RDB vs AOF

| Aspect | RDB | AOF |
|--------|-----|-----|
| Format | Binary snapshot | Command log (text/redis protocol) |
| Recovery Speed | Fast | Slow |
| Data Loss | Between snapshots | 1 second (everysec) or 0 (always) |
| File Size | Compact | Larger |
| Impact on Performance | Fork overhead | Depends on fsync frequency |

## 15. Revision Notes

### Quick Recap
- **Protocol**: TCP request-response, RESP (Redis Serialization Protocol).
- **Single-threaded**: Commands execute sequentially, atomic.
- **Eviction**: allkeys-lru, allkeys-lfu, volatile-ttl, etc.
- **Persistence**: RDB (snapshots), AOF (log), or hybrid.
- **High Availability**: Sentinel (failover) + Replication.
- **Sharding**: Redis Cluster (16384 hash slots, CRC16).
- **Spring Boot**: `@Cacheable`, `RedisTemplate`, `RedisRepository`.
- **Key operations**: `SET`, `GET`, `DEL`, `EXPIRE`, `TTL`.
- **Rich types**: Lists (LPUSH/LPOP), Sets (SADD/SMEMBERS), ZSets (ZADD/ZRANGE), Hashes (HSET/HGET), Streams (XADD/XREAD).

### Key Formulas
```
Hash Slot = CRC16(key) % 16384
Max Throughput ≈ Node Throughput * Cluster Nodes (linear scaling for reads)
Effective Cache Hit Ratio = L1_Hits + (L1_Miss * L2_Hit_Ratio)
Memory Fragmentation = Used_RSS / Used_Memory
```

### Anti-Patterns to Avoid
- Using KEYS on production instances.
- Storing large blobs (>10MB) directly.
- No TTL on cache keys.
- Using `FLUSHALL` without confirmation.
- Sharing Redis instances between environments.
- Ignoring `maxmemory` configuration.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                       REDIS CHEAT SHEET                            |
+-------------------------------------------------------------------+
| DATA TYPE    | COMMANDS                              | USE CASE    |
+--------------+---------------------------------------+-------------+
| STRING       | SET, GET, INCR, DECR, APPEND, MGET    | Cache,       |
|              |                                       | counters     |
| HASH         | HSET, HGET, HGETALL, HINCRBY, HDEL   | Objects      |
| LIST         | LPUSH, RPUSH, LPOP, BRPOP, LLEN,      | Queues       |
|              | LRANGE                                |              |
| SET          | SADD, SMEMBERS, SINTER, SUNION        | Tags, dedup  |
| SORTED SET   | ZADD, ZRANGE, ZREVRANGE, ZSCORE,     | Leaderboards |
|              | ZRANK, ZINCRBY                        |              |
| STREAM       | XADD, XREAD, XREADGROUP, XACK,        | Event stream |
|              | XRANGE, XDEL                          |              |
+-------------------------------------------------------------------+
| PERSISTENCE                 | EVICTION POLICIES                     |
+-----------------------------+--------------------------------------+
| RDB: SAVE/BGSAVE           | noeviction, allkeys-lru, allkeys-lfu  |
| AOF: appendonly yes        | volatile-lru, volatile-lfu,           |
|      appendfsync everysec  | volatile-ttl, allkeys-random          |
| HYBRID: aof-use-rdb-preamble|                                      |
+-----------------------------+--------------------------------------+
| HIGH AVAILABILITY          | CLUSTER                              |
+-----------------------------+--------------------------------------+
| Sentinel: monitor/failover | 16384 hash slots                      |
| Replication: async master  | CRC16(key) % 16384                    |
| -> replica                 | Cluster nodes gossip on 16379         |
| Automatic failover         | MOVED/ASK redirection                 |
+-----------------------------+--------------------------------------+
| REDIS-CLI COMMANDS                                                |
+-------------------------------------------------------------------+
| INFO [section]   | Server, Clients, Memory, Persistence, Stats     |
| CONFIG GET/SET   | Runtime configuration                          |
| SLOWLOG GET N    | Last N slow queries                            |
| MONITOR          | Real-time command stream (debug only!)        |
| SCAN cursor      | Non-blocking key scan                          |
| --bigkeys        | Analyze big key distribution                   |
| --hotkeys        | Analyze hot key distribution (Redis 4.0+)      |
| LATENCY DOCTOR   | Latency analysis report                        |
+-------------------------------------------------------------------+
| SPRING BOOT                                                       |
+-------------------------------------------------------------------+
| @Cacheable(cacheNames, key)     | Cache method result               |
| RedisTemplate opsForValue/List/ | Access data structures            |
| Set/ZSet/Hash/Stream            |                                   |
| LettuceConnectionFactory        | Non-blocking Redis client         |
| RedissonClient                  | Distributed locks, data structures|
+-------------------------------------------------------------------+
```
