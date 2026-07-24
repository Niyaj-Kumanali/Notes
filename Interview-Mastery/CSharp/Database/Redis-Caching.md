# Redis and Caching - Interview Mastery

---

## What is Redis

- **In-memory data store** that can optionally persist data to disk
  - Data lives in RAM, which is orders of magnitude faster than disk I/O
  - Persistence is achieved through RDB snapshots or AOF (Append-Only File) logs
  - You get the speed of memory with the safety net of disk persistence
- **Very fast** — sub-millisecond latency for read and write operations
  - Benchmarks typically show 100,000+ operations per second on modest hardware
  - This makes Redis ideal for high-throughput, low-latency scenarios
- **Single-threaded command processing**
  - Redis processes commands one at a time on a single thread
  - This eliminates the need for complex locking mechanisms
  - Multi-threading is used for I/O (network and disk), but command execution is single-threaded
  - This design simplifies data consistency but means CPU-bound operations can be a bottleneck
- **Network-based (client-server model)**
  - Redis runs as a server process; applications connect as clients
  - Communicates over TCP (default port 6379) or Unix sockets
  - Supports RESP (REdis Serialization Protocol) for communication
- **Primary use cases:**
  - Caching (most common — offload reads from database)
  - Session store (store user sessions for distributed web apps)
  - Message broker (Pub/Sub, Streams for inter-service communication)
  - Rate limiting (sliding window counters, token buckets)
  - Leaderboards and counters (Sorted Sets, INCR)
  - Real-time analytics (HyperLogLogs for unique counts)
  - Distributed locks (Redlock algorithm)

### Quick Redis Commands Reference

```bash
# Basic string operations
SET user:123:name "John Doe"
GET user:123:name
SETEX user:123:session 3600 "abc123"   # Set with expiry (seconds)
SET user:123:name "John Doe" EX 3600   # Same thing, different syntax
SETNX user:123:lock 1                  # Set if Not eXists (for locks)

# Increment/Decrement
INCR page:home:views
INCRBY user:123:balance 500

# Hash operations
HSET user:123 name "John" age 30 email "john@example.com"
HGET user:123 name
HGETALL user:123
HMGET user:123 name email

# List operations
LPUSH notifications:user:123 "You have a new message"
RPUSH queue:tasks '{"task":"process","id":1}'
LRANGE notifications:user:123 0 -1    # Get all items
LPOP queue:tasks                      # Pop from front

# Set operations
SADD tags:post:456 "csharp" "redis" "interview"
SMEMBERS tags:post:456
SISMEMBER tags:post:456 "redis"       # Check membership
SINTER tags:post:456 tags:post:789    # Intersection

# Sorted Set operations
ZADD leaderboard 1500 "player:1"
ZADD leaderboard 2300 "player:2"
ZREVRANGE leaderboard 0 9 WITHSCORES  # Top 10 with scores
ZRANK leaderboard "player:1"           # Get rank (0-based)

# TTL operations
EXPIRE user:123:session 3600           # Set TTL in seconds
TTL user:123:session                   # Check remaining TTL
PERSIST user:123:session               # Remove TTL (make permanent)

# Key management
KEYS user:*                            # Find keys by pattern (avoid in production!)
SCAN 0 MATCH user:* COUNT 100          # Safer iteration
DEL user:123:old_data                  # Delete a key
EXISTS user:123:name                   # Check if key exists
TYPE user:123:name                     # Check data type of key
```

---

## Redis Data Types

### Strings

- Most basic Redis data type
- Can store text, numbers, binary data, or serialized objects (JSON, XML)
- Maximum size: **512 MB** per string
- Atomic operations: INCR, DECR, INCRBY, DECRBY
- Supports bitwise operations: SETBIT, GETBIT, BITCOUNT

```csharp
// Storing a serialized object as a string
var user = new User { Id = 123, Name = "John", Email = "john@example.com" };
string json = JsonSerializer.Serialize(user);

await database.StringSetAsync("user:123", json, TimeSpan.FromMinutes(30));

string? cached = await database.StringGetAsync("user:123");
if (cached.HasValue)
{
    user = JsonSerializer.Deserialize<User>(cached!);
}

// Atomic counter
await database.StringIncrementAsync("page:home:views");
long views = await database.StringGetAsync("page:home:views");
```

### Hashes

- Field-value pairs within a single key
- Like a `Dictionary<string, string>` or a small database row
- Ideal for storing objects without serializing the entire thing
- Individual fields can be read/written without fetching the whole hash
- Maximum: 2^32 - 1 field-value pairs (over 4 billion)

```csharp
// Store user profile as a hash
await database.HashSetAsync("user:123", new HashEntry[]
{
    new HashEntry("name", "John Doe"),
    new HashEntry("age", "30"),
    new HashEntry("email", "john@example.com"),
    new HashEntry("lastLogin", DateTime.UtcNow.ToString("O"))
});

// Get individual fields
RedisValue name = await database.HashGetAsync("user:123", "name");
RedisValue email = await database.HashGetAsync("user:123", "email");

// Get all fields
HashEntry[] allFields = await database.HashGetAllAsync("user:123");

// Increment a field atomically
await database.HashIncrementAsync("user:123", "loginCount");
```

### Lists

- Ordered collection of strings (doubly linked list)
- Supports push/pop from both ends (LPUSH, RPUSH, LPOP, RPOP)
- Can be used as a queue (LPUSH + RPOP) or stack (LPUSH + LPOP)
- Supports blocking operations (BLPOP, BRPOP) for queue patterns
- Maximum length: 2^32 - 1 elements (over 4 billion)

```csharp
// Use as a queue
await database.ListLeftPushAsync("queue:tasks", taskJson1);
await database.ListLeftPushAsync("queue:tasks", taskJson2);

// Pop from the right (FIFO)
string? task = await database.ListRightPopAsync("queue:tasks");

// Use as a message timeline
await database.ListLeftPushAsync("timeline:user:123", notificationJson);
var recent = await database.ListRangeAsync("timeline:user:123", 0, 49); // Latest 50

// Blocking pop (waits for item to be available — useful for worker patterns)
var result = await database.ListLeftPopAsync("queue:tasks");
```

### Sets

- Unordered collection of unique strings
- Supports set operations: union, intersection, difference
- Perfect for tags, categories, or any "membership" scenario
- Maximum: 2^32 - 1 members

```csharp
// Store tags for a blog post
await database.SetAddAsync("post:456:tags", new RedisValue[] { "csharp", "redis", "interview" });

// Check if a tag exists
bool hasRedis = await database.SetContainsAsync("post:456:tags", "redis");

// Find posts that have both "csharp" AND "redis" tags
RedisValue[] taggedPosts = await database.SetIntersectAsync(
    new RedisKey[] { "tag:csharp:posts", "tag:redis:posts" });

// Get all members
RedisValue[] allTags = await database.SetMembersAsync("post:456:tags");
```

### Sorted Sets

- Each member has an associated **score** (double precision floating point)
- Members are ordered by score (ascending by default)
- Perfect for leaderboards, rankings, priority queues, time-series data
- Supports range queries by rank or by score

```csharp
// Leaderboard
await database.SortedSetAddAsync("leaderboard:game1", new SortedSetEntry[]
{
    new SortedSetEntry("player:alice", 1500),
    new SortedSetEntry("player:bob", 2300),
    new SortedSetEntry("player:charlie", 1800)
});

// Increment score atomically
await database.SortedSetIncrementAsync("leaderboard:game1", "player:alice", 100);

// Get top 10 players (highest score first)
var top10 = await database.SortedSetRangeByRankWithScoresAsync(
    "leaderboard:game1", 0, 9, Order.Descending);

// Get rank of a specific player
long? rank = await database.SortedSetRankAsync("leaderboard:game1", "player:alice", Order.Descending);
```

### Other Data Types (Brief Overview)

- **Bitmaps** — manipulate strings as arrays of bits; useful for feature flags, daily active users
  - `SETBIT user:123:login:2024-01 0 1` — mark day 0 as logged in
  - `BITCOUNT user:123:login:2024-01` — count total login days
- **HyperLogLogs** — probabilistic data structure for counting unique elements
  - Uses only 12 KB of memory regardless of cardinality
  - `PFADD unique:visitors "user1" "user2" "user1"` — deduplicates automatically
  - `PFCOUNT unique:visitors` — returns approximate unique count
- **Streams** — append-only log data structure (like Kafka topics)
  - Supports consumer groups for distributed message processing
  - `XADD mystream * field1 value1` — append to stream
  - `XREADGROUP GROUP consumers COUNT 10 STREAMS mystream >` — read as consumer

---

## Caching Patterns

### Cache-Aside (Lazy Loading)

- **Most common pattern** in web applications
- Application code is responsible for checking and updating the cache
- Flow:
  1. Application receives a read request
  2. Check the cache first
  3. If cache hit → return cached data (fast path)
  4. If cache miss → load from database, store in cache, return data
  5. On write → update database, then invalidate (delete) the cache entry

```csharp
public async Task<User?> GetUserAsync(int userId)
{
    string cacheKey = $"user:{userId}";

    // Step 1: Check cache
    string? cached = await _cache.GetStringAsync(cacheKey);
    if (cached != null)
    {
        _metrics.Increment("cache.hit");
        return JsonSerializer.Deserialize<User>(cached);
    }

    _metrics.Increment("cache.miss");

    // Step 2: Load from database
    var user = await _dbContext.Users.FindAsync(userId);
    if (user != null)
    {
        // Step 3: Store in cache with TTL
        string json = JsonSerializer.Serialize(user);
        await _cache.SetStringAsync(cacheKey, json, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
        });
    }

    return user;
}

public async Task UpdateUserAsync(User user)
{
    // Update database
    _dbContext.Users.Update(user);
    await _dbContext.SaveChangesAsync();

    // Invalidate cache (delete the old entry)
    await _cache.RemoveAsync($"user:{user.Id}");
}
```

### Read-Through

- Cache itself loads data from the database when a miss occurs
- Application only talks to the cache — never directly to the database for reads
- Typically implemented with a cache-aside library or cache server feature
- Less common in practice with Redis (usually implemented via custom code)

```csharp
// Conceptual read-through (using a library or custom abstraction)
public async Task<User?> GetUserAsync(int userId)
{
    // The cache abstraction handles DB loading transparently
    return await _readThroughCache.GetOrLoadAsync(
        key: $"user:{userId}",
        loadFromDb: async () => await _dbContext.Users.FindAsync(userId),
        expiration: TimeSpan.FromMinutes(30)
    );
}
```

### Write-Through

- Writes go to both the cache and the database simultaneously
- Cache and database are always in sync
- Adds latency to every write (both operations must complete)
- Good when reads vastly outnumber writes

```csharp
public async Task UpdateUserAsync(User user)
{
    string cacheKey = $"user:{user.Id}";
    string json = JsonSerializer.Serialize(user);

    // Write to cache and database at the same time
    var cacheTask = _cache.SetStringAsync(cacheKey, json, new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
    });

    var dbTask = _dbContext.Users.UpdateAsync(user);

    await Task.WhenAll(cacheTask, dbTask);
    await _dbContext.SaveChangesAsync();
}
```

### Write-Behind (Write-Back)

- Writes go to the cache immediately, database is updated asynchronously
- Very fast writes from the application's perspective
- Risk: if the cache crashes before the async DB write, data is lost
- Good for write-heavy workloads where eventual consistency is acceptable

```csharp
// Conceptual write-behind pattern
private readonly Channel<CacheWrite> _writeBehindQueue = Channel.CreateUnbounded<CacheWrite>();

public async Task UpdateUserAsync(User user)
{
    string cacheKey = $"user:{user.Id}";
    string json = JsonSerializer.Serialize(user);

    // Write to cache immediately (fast)
    await _cache.SetStringAsync(cacheKey, json);

    // Queue database write for background processing
    await _writeBehindQueue.Writer.WriteAsync(new CacheWrite
    {
        Key = cacheKey,
        Data = user,
        Operation = WriteOperation.Update
    });
}

// Background worker processes queue
private async Task ProcessWriteBehindQueue(CancellationToken ct)
{
    await foreach (var write in _writeBehindQueue.Reader.ReadAllAsync(ct))
    {
        try
        {
            _dbContext.Users.Update(write.Data);
            await _dbContext.SaveChangesAsync();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to persist write for key {Key}", write.Key);
            // Retry logic or dead-letter queue
        }
    }
}
```

---

## Cache Invalidation

### What is Cache Invalidation

- The process of **removing or updating stale data** in the cache
- One of the two hardest problems in computer science (along with naming things and off-by-one errors)
- If done wrong: users see stale data, inconsistencies, bugs
- If done too aggressively: cache hit ratio drops, more database load

### TTL-Based Expiry

- Set a Time-To-Live on every cache entry
- Redis automatically deletes the key when TTL expires
- Simple and reliable — no manual invalidation needed
- Trade-off: data may be stale for up to the TTL duration

```csharp
// Set TTL when caching
await _cache.SetStringAsync(cacheKey, json, new DistributedCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15)
});

// Or use sliding expiration (resets TTL on each access)
await _cache.SetStringAsync(cacheKey, json, new DistributedCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(15)
});
```

### Explicit Invalidation

- Delete the cache key immediately when the source data changes
- Guarantees freshness but adds complexity
- Must handle edge cases: what if the delete fails? What about race conditions?

```csharp
public async Task UpdateProductAsync(Product product)
{
    // 1. Update database
    await _dbContext.Products.UpdateAsync(product);
    await _dbContext.SaveChangesAsync();

    // 2. Invalidate cache
    await _cache.RemoveAsync($"product:{product.Id}");

    // Also invalidate related caches (e.g., product list)
    await _cache.RemoveAsync("products:all");
    await _cache.RemoveAsync($"category:{product.CategoryId}:products");
}
```

### Tag-Based Invalidation

- Group related cache entries under a "tag"
- When invalidating, delete the tag and all associated entries
- Useful when one data change affects multiple cache entries

```csharp
// Tag-based invalidation concept
public async Task InvalidateByTagAsync(string tag)
{
    // Get all keys associated with this tag
    var keys = await _database.SetMembersAsync($"tag:{tag}");

    // Delete all associated keys
    if (keys.Length > 0)
    {
        var redisKeys = keys.Select(k => (RedisKey)k.ToString()).ToArray();
        await _database.KeyDeleteAsync(redisKeys);
    }

    // Delete the tag itself
    await _database.KeyDeleteAsync($"tag:{tag}");
}
```

---

## TTL and Expiration

### How TTL Works in Redis

```bash
# Set key with TTL in seconds
SET user:123:session "abc123" EX 3600

# Set key with TTL in milliseconds
SET user:123:session "abc123" PX 3600000

# Set TTL on existing key
EXPIRE user:123:session 3600

# Set TTL in milliseconds on existing key
PEXPIRE user:123:session 3600000

# Set key that expires at a specific Unix timestamp
EXPIREAT user:123:session 1706227200

# Check remaining TTL (in seconds)
TTL user:123:session
# Returns: 3599 (remaining seconds), -1 (no TTL set), -2 (key doesn't exist)

# Check remaining TTL (in milliseconds)
PTTL user:123:session

# Remove TTL (make key permanent again)
PERSIST user:123:session
```

### TTL for Different Data Types

- TTL works the same for all data types (Strings, Hashes, Lists, Sets, Sorted Sets)
- The TTL applies to the **entire key**, not individual fields within a hash
- For fine-grained expiry (e.g., per-field), you need application-level logic or separate keys

### Automatic Cleanup

- Redis uses **lazy expiration** (check on access) and **active expiration** (background sampling)
- Lazy: when a key is accessed, Redis checks if it's expired and deletes it if so
- Active: Redis periodically samples keys with TTL set and deletes expired ones
- This means expired keys may linger briefly before being cleaned up

### Cache Stampede (Thundering Herd Problem)

- **What happens:** A popular cache entry expires. Suddenly hundreds of requests all see a cache miss and simultaneously try to rebuild the cache from the database.
- **Impact:** Database gets hit with a spike of identical queries, potentially causing slowdowns or outages
- **This is one of the most critical caching problems to solve**

### Preventing Cache Stampede

```csharp
// Approach 1: Distributed lock — only one request rebuilds the cache
public async Task<User?> GetUserAsync(int userId)
{
    string cacheKey = $"user:{userId}";
    string lockKey = $"lock:{cacheKey}";

    string? cached = await _cache.GetStringAsync(cacheKey);
    if (cached != null)
    {
        return JsonSerializer.Deserialize<User>(cached);
    }

    // Try to acquire lock
    bool acquired = await _database.LockTakeAsync(lockKey, "rebuild", TimeSpan.FromSeconds(10));
    if (acquired)
    {
        try
        {
            // Double-check after acquiring lock
            cached = await _cache.GetStringAsync(cacheKey);
            if (cached != null)
            {
                return JsonSerializer.Deserialize<User>(cached);
            }

            // Rebuild cache
            var user = await _dbContext.Users.FindAsync(userId);
            if (user != null)
            {
                await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(user),
                    new DistributedCacheEntryOptions
                    {
                        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
                    });
            }
            return user;
        }
        finally
        {
            await _database.LockReleaseAsync(lockKey, "rebuild");
        }
    }
    else
    {
        // Another request is rebuilding; wait and retry
        await Task.Delay(100);
        return await GetUserAsync(userId); // Retry
    }
}

// Approach 2: Early/Probabilistic expiry — refresh cache before it expires
public async Task<User?> GetUserWithEarlyExpiryAsync(int userId)
{
    string cacheKey = $"user:{userId}";

    string? cached = await _cache.GetStringAsync(cacheKey);
    long? ttl = await _database.KeyTimeToLiveAsync(cacheKey);

    if (cached != null && ttl.HasValue && ttl.Value > 0)
    {
        // If TTL is less than 20% of original, refresh in background
        if (ttl.Value < TimeSpan.FromMinutes(6).Ticks / TimeSpan.TicksPerSecond) // 20% of 30min
        {
            // Fire and forget — don't await
            _ = RefreshCacheInBackgroundAsync(userId, cacheKey);
        }
        return JsonSerializer.Deserialize<User>(cached);
    }

    return await RefreshCacheInBackgroundAsync(userId, cacheKey);
}

// Approach 3: Mutex / Request coalescing
// Use a ConcurrentDictionary to deduplicate in-flight requests
private readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();

public async Task<User?> GetUserCoalescedAsync(int userId)
{
    string cacheKey = $"user:{userId}";

    string? cached = await _cache.GetStringAsync(cacheKey);
    if (cached != null)
    {
        return JsonSerializer.Deserialize<User>(cached);
    }

    var semaphore = _locks.GetOrAdd(cacheKey, _ => new SemaphoreSlim(1, 1));
    await semaphore.WaitAsync();
    try
    {
        // Double-check after acquiring semaphore
        cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            return JsonSerializer.Deserialize<User>(cached);
        }

        var user = await _dbContext.Users.FindAsync(userId);
        if (user != null)
        {
            await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(user),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
                });
        }
        return user;
    }
    finally
    {
        semaphore.Release();
        _locks.TryRemove(cacheKey, out _);
    }
}
```

---

## Cache Strategies

### Full Cache

- All data is cached; database is rarely read
- Best for: small datasets, reference data, configuration
- Risk: high memory usage, complex synchronization
- Example: caching a list of countries, currencies, product categories

### Partial Cache (Hot Data)

- Only frequently accessed ("hot") data is cached
- Cold data is read directly from the database
- Best for: large datasets where only a subset is frequently accessed
- Most real-world applications use this strategy

### Read-Through Cache

- Application code is transparent — it only talks to the cache
- Cache handles all database interactions internally
- Simplifies application code but requires a caching library that supports it

### Cache Warming

- Pre-populate the cache at application startup or during off-peak hours
- Prevents cold-start cache misses
- Common techniques:
  - Background service that loads popular data into cache
  - Scheduled job that refreshes cache before peak traffic
  - On deployment, pre-warm critical cache entries

```csharp
// Cache warming at startup
public class CacheWarmingService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Load top products into cache
        var topProducts = await _dbContext.Products
            .OrderByDescending(p => p.PopularityScore)
            .Take(1000)
            .ToListAsync(stoppingToken);

        foreach (var product in topProducts)
        {
            string json = JsonSerializer.Serialize(product);
            await _cache.SetStringAsync($"product:{product.Id}", json,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
                }, stoppingToken);
        }

        _logger.LogInformation("Cache warmed with {Count} products", topProducts.Count);
    }
}
```

### Cache Monitoring

- **Hit ratio** = cache hits / (cache hits + cache misses)
  - Good hit ratio: > 80%
  - Poor hit ratio: < 50% (cache may not be effective)
- **Memory usage** — monitor Redis memory consumption
- **Eviction rate** — how often Redis evicts keys to make room
- **Key expiration rate** — how many keys expire per second

```bash
# Monitor cache stats in Redis
INFO stats          # Keyspace hits/misses, operations per second
INFO memory         # Memory usage and fragmentation
INFO keyspace       # Number of keys per database
DBSIZE              # Total number of keys
```

---

## Redis in ASP.NET Core

### IDistributedCache Interface

- Built-in .NET abstraction for distributed caching
- Implementations: Redis, SQL Server, NCache, In-Memory
- Methods: `GetStringAsync`, `SetStringAsync`, `RemoveAsync`, `GetAsync`, `SetAsync`

```csharp
// Register in Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyApp_";
});

// Inject into services
public class ProductService
{
    private readonly IDistributedCache _cache;

    public ProductService(IDistributedCache cache)
    {
        _cache = cache;
    }
}
```

### ConnectionMultiplexer

- **Shared connection** to Redis (not one connection per request)
- Thread-safe — can be shared across the entire application
- Handles connection pooling, reconnection, and failover automatically
- **Never create a new ConnectionMultiplexer per request** — this is a critical performance mistake

```csharp
// Register ConnectionMultiplexer as a singleton
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
{
    return ConnectionMultiplexer.Connect(
        builder.Configuration.GetConnectionString("Redis")!);
});

// Inject and use IDatabase for advanced operations
public class AdvancedCacheService
{
    private readonly IConnectionMultiplexer _redis;

    public AdvancedCacheService(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }

    public async Task<HashEntry[]> GetUserHashAsync(int userId)
    {
        var db = _redis.GetDatabase();
        return await db.HashGetAllAsync($"user:{userId}");
    }

    public async Task<long> IncrementCounterAsync(string counterName)
    {
        var db = _redis.GetDatabase();
        return await db.StringIncrementAsync($"counter:{counterName}");
    }
}
```

### Redis Transactions

```csharp
// Redis transactions (MULTI/EXEC)
public async Task TransferPointsAsync(string fromUser, string toUser, int points)
{
    var db = _redis.GetDatabase();
    var transaction = db.CreateTransaction();

    transaction.KeyDeleteAsync($"user:{fromUser}:points");
    transaction.StringIncrementAsync($"user:{toUser}:points", points);

    // Execute atomically
    await transaction.ExecuteAsync();
}
```

---

## Common Mistakes

### Cache Stampede

- **Problem:** Popular key expires, many simultaneous requests all miss cache and hit database
- **Solution:** Use distributed locks, early expiry, or request coalescing (see TTL section)
- **Always plan for this** — it's the most common production caching issue

### Stale Data Served from Cache

- **Problem:** Data in the database has changed, but the cache still holds old values
- **Solution:** Use appropriate TTLs, explicit invalidation on writes, event-driven cache invalidation
- **Balance:** too short TTL = low hit ratio; too long TTL = stale data risk

### Not Handling Redis Downtime Gracefully

- **Problem:** Application crashes or throws exceptions when Redis is unavailable
- **Solution:** Always implement fallback logic — fall back to database if cache is unavailable

```csharp
public async Task<User?> GetUserResilientAsync(int userId)
{
    try
    {
        string? cached = await _cache.GetStringAsync($"user:{userId}");
        if (cached != null)
        {
            return JsonSerializer.Deserialize<User>(cached);
        }
    }
    catch (Exception ex)
    {
        _logger.LogWarning(ex, "Redis unavailable, falling back to database");
    }

    // Fallback to database
    return await _dbContext.Users.FindAsync(userId);
}
```

### Caching Frequently Changing Data

- **Problem:** Data changes every few seconds, but you cache it for 30 minutes
- **Result:** Almost every cache read returns stale data
- **Solution:** Don't cache data that changes too frequently, or use very short TTLs with event-driven invalidation

### Not Setting TTL on Cache Entries

- **Problem:** Keys accumulate forever, eventually exhausting Redis memory
- **Solution:** **Always** set a TTL on every cache entry — no exceptions

### Large Objects in Cache

- **Problem:** Caching a 50 MB serialized object causes memory pressure and slow serialization
- **Solution:** Cache smaller, granular pieces of data; use compression; set memory limits

### Not Monitoring Cache Hit Ratio

- **Problem:** Cache is deployed but nobody checks if it's actually effective
- **Solution:** Track hit ratio, log cache hits/misses, alert on degradation

---

## Real-World Scenarios

### Caching User Session Data

- Store session data in Redis with a TTL matching session duration
- Fast access for every HTTP request (sub-millisecond)
- Automatic expiry when session is inactive

```csharp
public class SessionService
{
    private readonly IDistributedCache _cache;

    public async Task SetSessionAsync(string sessionId, SessionData data)
    {
        string json = JsonSerializer.Serialize(data);
        await _cache.SetStringAsync($"session:{sessionId}", json,
            new DistributedCacheEntryOptions
            {
                SlidingExpiration = TimeSpan.FromMinutes(30)
            });
    }

    public async Task<SessionData?> GetSessionAsync(string sessionId)
    {
        string? json = await _cache.GetStringAsync($"session:{sessionId}");
        return json != null ? JsonSerializer.Deserialize<SessionData>(json) : null;
    }
}
```

### Caching API Responses

- Cache responses from expensive or slow external APIs
- Reduces latency and external API costs

```csharp
public class CachedExternalApiService
{
    public async Task<WeatherData?> GetWeatherAsync(string city)
    {
        string cacheKey = $"weather:{city.ToLower()}";
        string? cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            return JsonSerializer.Deserialize<WeatherData>(cached);
        }

        var data = await _externalApiClient.GetWeatherAsync(city);
        if (data != null)
        {
            await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(data),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
                });
        }
        return data;
    }
}
```

### Caching Security Tokens

- Cache OAuth tokens, JWT signing keys, or API keys to avoid repeated authentication calls
- Set TTL slightly shorter than the token's actual expiry

```csharp
public class TokenCacheService
{
    public async Task<string?> GetAccessTokenAsync()
    {
        return await _cache.GetStringAsync("auth:access_token");
    }

    public async Task SetAccessTokenAsync(string token, TimeSpan expiresAt)
    {
        // Cache for slightly less than actual expiry
        var ttl = expiresAt - TimeSpan.FromMinutes(5);
        if (ttl > TimeSpan.Zero)
        {
            await _cache.SetStringAsync("auth:access_token", token,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = ttl
                });
        }
    }
}
```

### Rate Limiting with Redis

```csharp
public class RedisRateLimiter
{
    private readonly IConnectionMultiplexer _redis;

    public async Task<bool> IsAllowedAsync(string clientId, int maxRequests, int windowSeconds)
    {
        var db = _redis.GetDatabase();
        string key = $"ratelimit:{clientId}";

        long current = await db.StringIncrementAsync(key);

        if (current == 1)
        {
            // First request — set the TTL
            await db.KeyExpireAsync(key, TimeSpan.FromSeconds(windowSeconds));
        }

        return current <= maxRequests;
    }
}
```

### Distributed Locks with Redis

```csharp
public class DistributedLockService
{
    private readonly IConnectionMultiplexer _redis;

    public async Task<IDisposable?> AcquireLockAsync(string lockName, TimeSpan timeout)
    {
        var db = _redis.GetDatabase();
        string lockKey = $"lock:{lockName}";
        string lockValue = Guid.NewGuid().ToString();

        bool acquired = await db.LockTakeAsync(lockKey, lockValue, timeout);
        if (!acquired) return null;

        return new RedisLock(db, lockKey, lockValue);
    }

    private class RedisLock : IDisposable
    {
        private readonly IDatabase _db;
        private readonly string _key;
        private readonly string _value;

        public RedisLock(IDatabase db, string key, string value)
        {
            _db = db;
            _key = key;
            _value = value;
        }

        public void Dispose()
        {
            _db.LockRelease(_key, _value);
        }
    }
}
```

---

## Interview Questions

### Q1: Explain the Cache-Aside pattern. When would you use it?

- **Cache-Aside (Lazy Loading):** Application checks cache first on read. On miss, loads from DB, stores in cache. On write, updates DB then invalidates cache.
- **When to use:** Most web applications, read-heavy workloads, when you want full control over caching logic
- **Pros:** Simple to implement, cache only contains data that's actually requested
- **Cons:** First request for each key always hits the DB (cold start), cache invalidation logic lives in application code

### Q2: What is cache invalidation and why is it considered hard?

- Cache invalidation is the process of removing or updating stale data in the cache
- It's hard because:
  - Race conditions between reads and writes
  - Distributed systems make consistent invalidation difficult
  - Choosing between consistency (always fresh) and performance (long TTLs)
  - Handling failures during invalidation (what if the delete fails?)
  - Multi-layer caches (CDN + application cache + database)

### Q3: What is cache stampede and how do you prevent it?

- **Cache stampede:** When a popular cache entry expires, many simultaneous requests all miss the cache and hit the database at the same time
- **Prevention strategies:**
  - Distributed locks (only one request rebuilds the cache)
  - Early/probabilistic expiry (refresh cache before it expires)
  - Request coalescing (deduplicate in-flight requests)
  - Background refresh (proactive TTL-based refresh)

### Q4: When should you NOT cache data?

- Data that changes very frequently (every few seconds)
- Data that is rarely accessed (caching adds complexity for no benefit)
- Data where consistency is critical (financial transactions, medical records)
- Very large objects that would cause memory pressure
- Data with complex interdependencies where invalidation is impractical
- When the database is fast enough and caching adds unnecessary complexity

### Q5: Compare Redis vs Memcached.

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data types | Strings, Hashes, Lists, Sets, Sorted Sets, Streams | Strings only |
| Persistence | Yes (RDB, AOF) | No |
| Replication | Yes (master-slave) | No |
| Clustering | Yes (built-in) | Client-side sharding |
| Pub/Sub | Yes | No |
| Lua scripting | Yes | No |
| Memory efficiency | Moderate | Very efficient for simple key-value |
| Use case | Feature-rich caching, data structures, message broker | Simple caching of flat data |

### Q6: Explain TTL-based vs explicit cache invalidation.

- **TTL-based:** Set an expiration time on cache entries; Redis auto-deletes them
  - Pros: Simple, no coordination needed
  - Cons: Data can be stale up to TTL duration
- **Explicit:** Delete cache entry when source data changes
  - Pros: Always fresh data
  - Cons: Complex, requires handling failure cases, race conditions
- **Best practice:** Use both — TTL as a safety net, explicit invalidation for freshness

### Q7: How does Redis handle persistence?

- **RDB (Redis Database):** Periodic snapshots of the dataset to disk
  - Fast recovery, compact files
  - Risk: may lose data between snapshots
- **AOF (Append-Only File):** Logs every write operation to disk
  - More durable (configurable: every write, every second, or OS-managed)
  - Larger files, slower recovery
- **Best practice:** Use both RDB + AOF for maximum durability

### Q8: What is ConnectionMultiplexer and why is it important?

- ConnectionMultiplexer is StackExchange.Redis's connection manager
- Maintains a shared, persistent connection to Redis
- Handles connection pooling, reconnection, and failover
- **Never create a new connection per request** — Redis connections are expensive to establish
- It's thread-safe and can be shared as a singleton across the entire application

### Q9: How would you implement rate limiting with Redis?

- **Fixed window:** INCR key with TTL (simplest approach)
- **Sliding window:** Use sorted sets with timestamps, count entries within time window
- **Token bucket:** Use Lua scripts for atomic decrement-and-check logic
- Redis is ideal for rate limiting because operations are atomic and fast

### Q10: What is the difference between absolute and sliding expiration?

- **Absolute:** Cache entry expires at a fixed time after creation (e.g., created at 2pm, expires at 2:30pm)
- **Sliding:** TTL resets on each access (e.g., if accessed at 2:25pm, expires at 2:55pm)
- Sliding is better for session data (keeps active sessions alive)
- Absolute is better for data with hard freshness requirements

### Q11: How do you handle cache warming?

- Pre-populate cache at application startup or during off-peak hours
- Use background services to load popular/critical data
- Consider lazy warming: when a key is first accessed, load it eagerly
- Schedule warming jobs before peak traffic periods
- Monitor which keys are frequently missed and prioritize warming those

### Q12: What happens when Redis runs out of memory?

- Redis has a configurable `maxmemory` setting
- When limit is reached, Redis uses an eviction policy:
  - `noeviction`: Returns errors on write operations
  - `allkeys-lru`: Evict least recently used keys across all keys
  - `volatile-lru`: Evict LRU keys that have an expire set
  - `allkeys-random`: Evict random keys
  - `volatile-ttl`: Evict keys with shortest TTL first
- Choose policy based on your use case

### Q13: Explain the Write-Behind pattern and its risks.

- Writes go to cache immediately, database is updated asynchronously in the background
- **Pros:** Very fast writes, reduced database load
- **Cons:** Risk of data loss if cache crashes before DB write completes; eventual consistency only
- **Use when:** Write-heavy workloads where slight data loss is acceptable (e.g., analytics counters)

### Q14: How do you debug cache-related issues in production?

- Monitor cache hit ratio (should be > 80% for most workloads)
- Use Redis `INFO` command to check memory, keyspace hits/misses
- Use `SLOWLOG` to identify slow commands
- Log cache hits/misses in application code with correlation IDs
- Monitor Redis latency with `redis-cli --latency`
- Use Redis `MONITOR` command sparingly for real-time debugging

### Q15: When would you choose distributed cache over in-memory cache?

- **Distributed cache (Redis)** when:
  - Running multiple application instances (shared state)
  - Cache is larger than single server's memory
  - Need persistence or replication
  - Need advanced data structures or pub/sub
  - Need cache to survive application restarts
- **In-memory cache (MemoryCache)** when:
  - Single server deployment
  - Cache is small and application-specific
  - Ultra-low latency is critical (no network hop)
  - No need for shared state between instances

### Q16: How do you ensure cache consistency in a microservices architecture?

- Use event-driven cache invalidation (message bus notifications when data changes)
- Implement cache versioning (include version in cache key)
- Use read-your-writes consistency (after a write, invalidate and force cache miss)
- Consider CQRS pattern (separate read and write models with different cache strategies)
- Accept eventual consistency and design for it (show "last updated" timestamps to users)

### Q17: What are Redis Cluster and Redis Sentinel?

- **Redis Sentinel:** High availability solution
  - Monitors Redis master and replicas
  - Automatically promotes a replica to master if the current master fails
  - Provides service discovery for clients
- **Redis Cluster:** Horizontal scaling solution
  - Data is sharded across multiple Redis nodes (16,384 hash slots)
  - Each node holds a subset of the data
  - Supports automatic failover and resharding
  - Best for datasets larger than single machine's memory

---

*Last Updated: July 2026*
