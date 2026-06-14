# Redis

---

## Overview

- **Definition:** Redis (Remote Dictionary Server) is an open-source, in-memory data structure store used as a database, cache, message broker, and streaming engine.
- **Why It Exists:** Provides sub-millisecond latency with rich data structures (strings, hashes, lists, sets, sorted sets, streams, bitmaps, HyperLogLog, geospatial), replication, persistence, and clustering — making it the most widely deployed distributed cache.
- **Key Concepts:** **Single-Threaded Event Loop** (atomic commands, no race conditions), **Persistence** (RDB snapshots, AOF log, hybrid), **Eviction Policies** (allkeys-lru/lfu, volatile-ttl), **Replication** (async master-replica), **Redis Cluster** (16384 hash slots, CRC16 sharding), **Sentinel** (high availability failover).
- **Redis Use Cases Beyond Caching** — Real-time leaderboards (sorted sets), rate limiters (sorted sets + Lua), pub/sub notifications, job queues (lists or streams), distributed locks (SET NX EX + Redlock), session stores, geospatial queries (GEO API), and probabilistic data structures (Bloom filters via RedisBloom module).
- **Redis Memory Efficiency** — Use hashes for storing objects instead of individual string keys (reduces key overhead from ~80 bytes/key to ~20 bytes/field). Use integer encoding for small integers. Enable compression for large values. Use shorter key names (but keep them readable). Monitor `used_memory_overhead` to track key-level overhead.

---

## Core Concepts

- **Data Structures:** **Strings** for cache/counters, **Hashes** for objects, **Lists** for queues, **Sets** for tags/dedup, **Sorted Sets** for leaderboards, **Streams** for event sourcing, **Bitmaps** for analytics, **HyperLogLog** for cardinality estimation.
- **Persistence:** **RDB** — point-in-time snapshots, compact, fast recovery, data loss between snapshots. **AOF** — logs every write, durable (fsync everysec), larger file. **Hybrid** (Redis 4.0+) combines RDB base + AOF incremental.
- **Redis Cluster:** Automatic sharding across up to 1000 nodes. CRC16(key) % 16384 determines slot. Smart clients calculate node ownership. Nodes gossip for failure detection.
- **High Availability:** **Sentinel** provides monitoring, automatic failover, and service discovery. Minimum 3 Sentinels for quorum. **Replicas** provide read scaling and failover target.
- **Redis Transactions vs Lua Scripts** — Redis transactions (MULTI/EXEC) batch commands but don't provide rollback — if one command fails, the rest still execute. Lua scripts provide true atomicity: all commands execute or none do, and they can include conditional logic (if/else). Prefer Lua scripts for any operation that needs atomic read-modify-write.
- **Redis Stack (Redis 7+)** — Extends Redis with modules for search (full-text, vector search), JSON (native JSON document store), time series (ingestion, aggregation), and graph (property graph database). Redis Stack modules run within the same single-threaded event loop, maintaining atomicity guarantees.

```java
// Distributed rate limiter with Lua script
private static final String SCRIPT =
    "local key = KEYS[1]; local now = tonumber(ARGV[1]); " +
    "local window = tonumber(ARGV[2]); local limit = tonumber(ARGV[3]); " +
    "redis.call('ZREMRANGEBYSCORE', key, 0, now - window); " +
    "local count = redis.call('ZCARD', key); " +
    "if count < limit then redis.call('ZADD', key, now, now..':'..math.random()); " +
    "redis.call('EXPIRE', key, window); return 1 end; return 0";
```

---

## Common Mistakes

- **Not Setting TTL** — Creates memory leaks; entries live forever unless evicted.
  - **Why it looks correct:** The application writes data to Redis and it works perfectly, so forgetting a TTL feels harmless — the memory leak only becomes visible when the Redis `maxmemory` alarm fires at 3 AM.
- **Using KEYS in Production** — Blocks Redis for millions of keys. Use SCAN with cursor instead.
  - **Why it looks correct:** `KEYS *` returns all matching keys instantly in development with 1,000 keys, so it seems like a harmless debugging command — the 10-second block only happens when the production keyspace has 10M keys and every other Redis command queues up behind it.
- **Storing Large Values (>10MB)** — Degrades performance and network throughput. Use object store for blobs.
  - **Why it looks correct:** Redis stores whatever you put in it, so storing a 15MB serialized object seems fine — the performance degradation only becomes visible when Redis throughput drops by 50% because the single-threaded event loop is spending 100ms serializing a single large value to the network.
- **Ignoring Connection Management** — Connection leaks from unclosed connections. Use `RedisTemplate` or try-with-resources.
  - **Why it looks correct:** Creating a `Jedis` connection per request works fine in a single-threaded test — the connection leak only surfaces after the application runs for hours and hits the OS file descriptor limit.
- **No maxmemory Configuration** — Without it, Redis can exhaust server RAM and get OOM-killed.
  - **Why it looks correct:** The server has 64GB RAM and Redis only uses 10GB, so setting `maxmemory` seems unnecessary — the OOM killer only triggers when another process on the same server uses memory during a traffic spike, and Redis gets killed because it has no bounds.
- **Using Redis as a Primary Database Without Persistence** — Without RDB or AOF, all data is lost on restart. Redis is primarily a cache, not a primary store. For critical data, configure both RDB (periodic snapshots) and AOF (everysec fsync) persistence.
  - **Why it looks correct:** The application reads and writes to Redis correctly in development, and Redis never restarts — the data loss only becomes apparent when the Redis node crashes during a deployment and all user sessions are wiped out.
- **Not Configuring Connection Pooling** — Creating a new Redis connection per request exhausts file descriptors and increases latency. Use connection pooling (Lettuce or Jedis Pool) with a pool size of 10-50 connections per application instance.
  - **Why it looks correct:** A single request response is fast, so creating a new connection per request seems like a minor overhead — the file descriptor exhaustion only shows up when 100 concurrent requests each open a new connection, hitting the OS limit of 1024 FDs.
- **Running Redis on the Same Server as the Application** — Redis competes for CPU, memory, and network bandwidth. Dedicate separate instances for Redis, ideally with high-clock-speed CPUs for the single-threaded event loop.
  - **Why it looks correct:** It reduces infrastructure complexity and network latency when Redis is on localhost — the CPU contention only becomes visible when the application's garbage collection pauses cause Redis latency to spike from 1ms to 50ms.

---

## Key Design Considerations

- **Hybrid Caching (L1 + L2)** — Local Caffeine (nanosecond) + Redis (millisecond) reduces Redis load by 80% for hot keys. Coordinate via pub/sub for L1 invalidation.
- **Pipelining** — Batch multiple commands to reduce round-trip latency. Use `executePipelined` for bulk operations.
- **Memory Optimization** — Use hashes for objects (vs many strings), enable compression (Snappy/Zstd), use shorter key names.
- **Hot Key Mitigation** — Replicate hot key to multiple shards, add local cache on each node, split key into sub-keys.
- **Security** — Use ACLs (Redis 6+), TLS, bind to private interfaces, rename dangerous commands (`FLUSHALL`), deploy in VPC.
- **Monitoring** — `INFO stats` for hit ratio, `SLOWLOG` for slow queries, `MEMORY DOCTOR` for fragmentation, `--hotkeys` and `--bigkeys` analysis.
- **Redis Persistence Configuration Trade-offs** — RDB with 5-minute save intervals: best for cache (acceptable data loss, fast startup). AOF everysec: good for session store (lose 1 second of data, slightly slower startup). AOF always: strongest durability, but 10x slower writes. Hybrid (RDB base + AOF incremental): best of both — fast startup with minimal data loss.
- **Cluster Resharding and Slot Migration** — When adding or removing nodes in Redis Cluster, slots must be migrated. Use `redis-cli --cluster reshard` to move slots from source to target nodes. Resharding is online and non-blocking. Monitor cluster state during resharding — the cluster remains available but per-node CPU increases. Plan resharding during low traffic windows.

---

## Real-World Scenarios

### Scenario 1: Real-Time Leaderboard for a Gaming Platform
**Context:** A gaming platform needs a real-time leaderboard showing top 100 players, any player's rank, and percentile distribution. The leaderboard is updated every time a player scores points. Latency must be <10ms.

**Resolution:** Use Redis Sorted Sets. Each game mode has a sorted set key (`leaderboard:classic`, `leaderboard:battle-royale`). `ZINCRBY leaderboard:classic player123 150` updates the score atomically. `ZREVRANGE leaderboard:classic 0 99 WITHSCORES` returns the top 100. `ZREVRANK leaderboard:classic player123` returns any player's rank. The sorted set handles millions of players with O(log(N)) operations. Periodic snapshots to the database persist the leaderboard.

```java
@Service
public class LeaderboardService {
    private final RedisTemplate<String, Object> redis;

    public void addScore(String gameMode, String playerId, double score) {
        redis.opsForZSet().incrementScore("leaderboard:" + gameMode, playerId, score);
    }

    public List<LeaderboardEntry> getTop100(String gameMode) {
        Set<ZSetOperations.TypedTuple<Object>> top = redis.opsForZSet()
            .reverseRangeWithScores("leaderboard:" + gameMode, 0, 99);
        return top.stream()
            .map(t -> new LeaderboardEntry((String) t.getValue(), t.getScore()))
            .toList();
    }

    public Long getRank(String gameMode, String playerId) {
        Long rank = redis.opsForZSet().reverseRank("leaderboard:" + gameMode, playerId);
        return rank != null ? rank + 1 : null;  // ZREVRANK is 0-based
    }
}
```

### Scenario 2: Distributed Rate Limiting with Lua Scripts
**Context:** An API gateway needs to rate limit requests per user to 100 requests per minute. With multiple gateway instances, in-memory rate limiting doesn't work because counts aren't shared. Database-based rate limiting adds too much latency.

**Resolution:** Use Redis sorted sets with Lua scripting for atomic sliding window rate limiting. Each user has a sorted set key (`ratelimit:user:123`) with timestamps as scores. The Lua script removes entries outside the window, counts remaining entries, and adds the new entry if under the limit — all atomically in a single Redis operation.

### Scenario 3: Session Cache with High Availability
**Context:** A microservices application stores user sessions in Redis. If Redis goes down, all users are logged out. The application needs automatic failover without data loss.

**Resolution:** Deploy Redis Sentinel with 3 Sentinel nodes and a master-replica setup. When the master fails, Sentinel promotes a replica to master within seconds. The application uses Lettuce (which supports Sentinel) and reconnects automatically. Session data on the promoted replica is preserved (asynchronous replication may lose the last few writes, but sessions are short-lived).

## Use Cases

- **Real-time leaderboards and counters** — gaming scores, social media likes, or trending hashtags
  - Sorted Sets (`ZINCRBY`, `ZREVRANK`) provide O(log N) ranked queries. Single-digit millisecond latency for millions of entries.
  - **Avoid when:** strong durability is required — Redis is primarily in-memory; use a database for authoritative storage.

- **Session store for distributed apps** — user sessions across microservices or multiple server instances
  - Hash or String with TTL. Stateless servers share session state via Redis. Sliding expiration auto-cleans stale sessions.
  - **Avoid when:** sessions are extremely large (>1 MB) — consider external blob storage with Redis as a cache index.

- **Distributed locking** — coordinating access to shared resources across service instances
  - Use Redlock algorithm (`SET key NX EX 10`) for distributed mutexes. Lua scripting ensures atomic acquire-and-set-TTL.
  - **Avoid when:** lock duration is unpredictable — prefer a lease-based mechanism with heartbeats.

- **Rate limiting** — API throttling per user, IP, or API key
  - Sliding window via Sorted Sets or `INCR` with TTL. Lua scripts ensure atomic window checks under high concurrency.
  - **Avoid when:** rate limits are coarse — local in-memory counters with async Redis sync are more performant.

- **Pub/Sub for real-time messaging** — live chat, notifications, or event broadcasting within a service
  - Lightweight publish/subscribe with `PUBLISH`/`SUBSCRIBE`. Sub-second delivery to all subscribers.
  - **Avoid when:** delivery guarantees matter — Redis Pub/Sub is fire-and-forget; use Redis Streams or a message queue for at-least-once delivery.

---

## Scenario-Based Questions

1. **Q: You're building a real-time leaderboard for a mobile game with 10M players. Scores update every time a player completes a match. How do you design for sub-10ms rank queries?**
   - A: Use Redis Sorted Sets per game mode (`leaderboard:{gameMode}`). `ZINCRBY` atomically updates scores. `ZREVRANK` returns a player's rank in O(log(N)). Partition by game mode and time period (daily, weekly, all-time). Use Redis Cluster for horizontal scaling — hash by game mode so each leaderboard fits on one node. Snapshot to PostgreSQL every 5 minutes for persistence.

  - **Interview follow-up:** Your leaderboard sorted set stores 10M player scores per game mode. A single `ZREVRANGE 0 99` is fast, but someone asks for the 999,900th to 1,000,000th rank — how does the sorted set handle a deep-range query on a 10M-member set?

2. **Q: Your Redis instance runs out of memory and starts evicting keys. Critical session data is evicted, causing users to be logged out. How do you prevent this?**
   - A: Configure `maxmemory` and select the right eviction policy. For session data, use `allkeys-lru` (most recently used sessions stay) or `volatile-ttl` (evict sessions closest to expiry). Set TTL on all session keys. Monitor `evicted_keys` in `INFO stats`. Add memory alarms at 70%, 80%, 90% usage. Use Redis Cluster to distribute memory across nodes.

3. **Q: During a flash sale, a single product key in Redis gets millions of requests per second (hot key). The Redis node handling that key becomes CPU-saturated. How do you mitigate?**
   - A: Multiple strategies: (1) Add a local L1 cache (Caffeine) on each application instance for the hot key. (2) Replicate the hot key across multiple Redis nodes by adding a suffix (`product:123:replica-0`, `product:123:replica-1`). The application picks a random replica. (3) Use read replicas to distribute read load. (4) Use Redis Cluster and ensure the hot key is in its own slot that can be moved to a larger node.

  - **Interview follow-up:** You replicated the hot product key across 3 Redis nodes with `product:123:replica-0`, `product:123:replica-1`, `product:123:replica-2`. When the product price is updated, how do you invalidate all 3 replicas atomically so no application instance reads a stale price?

4. **Q: Your application uses Redis pub/sub for real-time notifications. When a subscriber disconnects briefly, it misses all messages published during that time. Users don't receive critical notifications. How do you fix this?**
   - A: Migrate from pub/sub to Redis Streams. Streams persist messages, support consumer groups, and allow replay from any point. Configure `MAXLEN ~ 10000` to cap stream size while keeping recent messages. Consumers acknowledge messages after processing. If a consumer reconnects, it reads from the last acknowledged message. This provides reliable delivery that pub/sub lacks.

  - **Interview follow-up:** You migrate to Redis Streams with a consumer group for notifications. After a deployment, the consumer group's last-delivered ID is lost because it was stored in the consumer's local state — how do you ensure the consumer group position survives application restarts?

5. **Q: You need to implement a distributed lock to prevent duplicate order processing. Redis provides `SET key NX EX 10`. How do you handle cases where the lock holder takes longer than 10 seconds?**
   - A: Use Redisson which implements automatic lock extension (watchdog). The lock TTL refreshes automatically as long as the holder is still processing. If the holder crashes, the lock expires after the original TTL. For safety-critical locks, use the Redlock algorithm (acquire lock on N/2+1 Redis nodes) to prevent issues if a single Redis node fails while holding a lock.

6. **Q: Your Redis cache hit ratio is 85% and you want to improve it. The cache contains 5GB of data in a 10GB maxmemory configuration. `evicted_keys` is 0. What should you investigate?**
   - A: Check the eviction policy — `noeviction` returns errors on memory full but doesn't evict. Switch to `allkeys-lru` or `allkeys-lfu` for better cache utilization. Investigate TTL settings — entries with very short TTL expire before being reused. Check for cache key pattern issues — high-cardinality keys (user IDs, session IDs in product cache) reduce reuse. Monitor access frequency — perhaps 20% of your 5GB serves 80% of requests, and optimizing the hot data could improve the ratio.

7. **Q: Your Redis Cluster has 6 nodes (3 master, 3 replica). During a network partition, quorum is lost and the cluster goes down. How do you design for higher availability?**
   - A: Use 5+ nodes for better partition tolerance (3 masters, 2+ replicas). Configure `cluster-node-timeout` to balance between quick failover and avoiding false positives. Use Sentinel alongside Cluster for automatic failover. For multi-datacenter, use Redis Enterprise with active-active replication or deploy independent clusters per DC with application-level routing.

8. **Q: You need to perform a full cache flush (FLUSHALL) after a data migration. During the flush, all cached data is lost and the DB is overwhelmed by cache misses. How do you do this safely?**
   - A: Never use `FLUSHALL` in production. Instead, use key prefix pattern deletion with `SCAN` or use a lazy migration strategy: deploy the new application that uses a different cache key prefix (`app:v2:product:123`). The old keys expire naturally via TTL. If you must flush, do it gradually — `SCAN 0 MATCH app:v1:*` in batches of 1000 keys, delete each batch, and monitor DB load.

9. **Q: Your application stores user sessions in Redis with TTL. Users report being randomly logged out before their session expires. The TTL is set to 24 hours. What's happening?**
   - A: Check `maxmemory` and `evicted_keys` — Redis is evicting session keys to stay within memory limits. Solution: increase `maxmemory` on the Redis instance, use `volatile-ttl` eviction policy so sessions with the longest remaining TTL are evicted last (instead of random LRU eviction), or move sessions to a dedicated Redis instance with enough memory for all active sessions.

10. **Q: Your application needs to cache large JSON blobs (2MB each) for product details. Redis performance degrades because of large values. How do you optimize?**
    - A: Compress the JSON before storing (Snappy/Zstd compression reduces 2MB → ~400KB). Split large blobs into smaller hashes (each field is a section of the product). For very large payloads (>10MB), store them in S3 and cache only the URL in Redis. Increase `maxmemory` appropriately. Use L1 Caffeine cache for the most popular products to reduce Redis load.

---

## Interview Questions

1. **Is Redis single-threaded or multi-threaded?**
   - A: Command execution is single-threaded (atomic operations, no race conditions). I/O uses multiple threads (Redis 6+ for networking). Background tasks (BGSAVE, AOF rewrite) use child processes/fork.

2. **What is the difference between RDB and AOF persistence?**
   - A: RDB is a point-in-time snapshot (compact, fast recovery, data loss between snapshots). AOF logs every write operation (durable with `appendfsync everysec`, larger file, slower recovery). Redis 4+ hybrid: RDB base + AOF incremental for best of both.

3. **How does Redis Cluster shard data?**
   - A: 16384 hash slots. `CRC16(key) % 16384` determines the slot. Each node owns a subset of slots. Smart clients (lettuce, jedis) compute the node from the key. Nodes gossip for failure detection. Slots can be migrated between nodes for resharding.

4. **What is Redis Sentinel?**
   - A: A high-availability system: monitors master/replicas, performs automatic failover (promotes replica on master failure in 10-30s), and provides service discovery. Minimum 3 Sentinel nodes for quorum-based decisions.

5. **How do you implement a distributed lock with Redis?**
   - A: `SET key value NX EX 10` (set if not exists, 10s expiry). For production: use Redisson which handles auto-extension (watchdog), reentrancy, and the Redlock algorithm for safety across multiple Redis nodes.

6. **What is the difference between Redis pub/sub and Streams?**
   - A: Pub/sub is fire-and-forget — messages are lost if no subscriber is listening, no persistence, no replay. Streams persist messages (configurable retention), support consumer groups with load balancing, acknowledge individual messages, and allow replay from any point.

7. **How do you handle hot keys in Redis?**
   - A: Identify via `redis-cli --hotkeys`. Mitigate: local L1 cache (Caffeine), replicate hot key across shards with key suffixes (`key:0`, `key:1`), use read replicas for read-heavy hot keys, or split the hot key into sub-keys across multiple nodes.

8. **Explain Redis memory eviction policies.**
   - A: `allkeys-lru` (evict least recently used), `allkeys-lfu` (least frequently used), `volatile-lru` (LRU among keys with TTL), `volatile-ttl` (shortest TTL first), `noeviction` (return OOM errors). Default: `noeviction`. Choose based on access pattern and data importance.

9. **How do you connect Spring Boot to Redis?**
   - A: Add `spring-boot-starter-data-redis`. Configure host/port in `application.yml`. Spring Boot auto-configures `RedisTemplate<String, Object>` (Lettuce client) and `CacheManager` for `@Cacheable` annotations. Use `@EnableRedisRepositories` for Redis repositories.

10. **How would you design a real-time leaderboard with Redis?**
    - A: Sorted Set per period: `leaderboard:daily`, `leaderboard:weekly`, `leaderboard:alltime`. `ZINCRBY` updates scores atomically. `ZREVRANGE 0 99 WITHSCORES` for top 100. `ZREVRANK` for individual rank. `ZCOUNT` for percentile. Snapshot to DB periodically for persistence.

---

## Developer Recommendations

- **Always set TTL on every key** — Without TTL, keys live forever, consuming memory until eviction kicks in. Even for "permanent" data, set a long TTL (e.g., 7 days) and refresh it on access. This prevents memory leaks and ensures stale data is eventually cleaned up. Use `EXPIRE` or set TTL on `SET`/`SETEX`.
  - **Production story:** An e-commerce platform stored product data in Redis without TTL, expecting eviction to handle cleanup — after 6 months, `used_memory` hit 48GB on a 32GB instance, Redis started evicting all keys indiscriminately, and the checkout flow broke because active session data was evicted alongside stale product cache entries.

- **Use Redis Streams instead of pub/sub for reliable messaging** — Pub/sub loses messages when subscribers are offline. Streams persist messages, support consumer groups with acknowledgment, and allow replay from any point. The only scenario where pub/sub is correct is ephemeral notifications where message loss is acceptable (e.g., live user count updates).

- **Never use `KEYS` in production** — `KEYS *` blocks Redis for millions of keys, blocking all other operations. Use `SCAN 0 MATCH pattern:* COUNT 100` for safe iteration over keys. It returns results in batches without blocking. The same applies to `FLUSHALL` and `FLUSHDB` — never run them in production without careful planning.

- **Use pipelining and batching for bulk operations** — Sending 1000 individual `SET` commands involves 1000 round trips (each ~1ms = 1 second total). Pipelining sends all commands in one batch — 1000 commands in one round trip (~5ms total). Spring's `RedisTemplate.executePipelined()` batches commands automatically. Use pipelining for cache warm-up, bulk updates, and data migration.

- **Monitor memory, hit ratio, and evictions** — Three critical Redis metrics: `used_memory` vs `maxmemory` (track growth trends), `keyspace_hits / (keyspace_hits + keyspace_misses)` for hit ratio (target >95%), and `evicted_keys` (should be near zero — evictions mean cache is undersized). Set up alerts for each. Use `MEMORY DOCTOR` for fragmentation issues.
  - **Production story:** A fintech company's Redis hit ratio silently dropped from 94% to 72% over 3 months — the team hadn't set up hit ratio monitoring, so they only noticed when the production database CPU hit 100%. The root cause was a code change that added a random query parameter (`?_t=timestamp`) to every cache key, making every request a cache miss.

- **Use Lua scripts for atomic multi-key operations** — Redis single-threaded execution makes Lua scripts atomic. Instead of GET → check → SET (3 round trips, race condition possible), write a Lua script that does all operations atomically in one round trip. Redis's built-in replication ensures the script runs the same way on replicas.
- **Use the right eviction policy for your use case** — `allkeys-lru` for general-purpose caching (hot data stays, cold data evicted). `allkeys-lfu` for workloads with stable popularity (viral content, trending items). `volatile-ttl` for session stores where each key has a TTL and you want to evict soonest-expiring first. Never use `noeviction` in a cache deployment — it causes write failures when memory fills up.
- **Use Redis Cluster for datasets exceeding a single node's memory** — A single Redis instance is limited by available RAM (typically 16-64GB in production). Redis Cluster automatically shards data across nodes using 16384 hash slots. Each node handles a subset of slots. For 100GB dataset, use 3-5 nodes with replicas. Cluster mode requires smart clients (Lettuce, Jedis) that understand slot routing. Operations spanning multiple keys (MGET, transactions) only work within the same slot.
