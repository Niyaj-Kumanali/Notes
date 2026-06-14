# Spring Boot Caching

---

## Overview

- **Definition:** Spring Boot Caching provides a declarative caching abstraction that reduces repeated expensive operations by storing results and returning them on subsequent calls. It supports multiple cache providers: **Caffeine**, **Redis**, **EhCache**, **Hazelcast**, and a simple in-memory `ConcurrentHashMap`.
- **Why It Exists:** To improve application performance by avoiding redundant computation or database queries for frequently accessed data, reducing latency and backend load without changing business logic.
- **Key Concepts:**
  - **@Cacheable** — Caches the method's return value. On subsequent calls with the same arguments, the cached value is returned without executing the method body.
  - **@CachePut** — Always executes the method and updates the cache with the result. Useful for keeping the cache in sync after update operations.
  - **@CacheEvict** — Removes entries from the cache. Used when data is deleted or invalidated to prevent stale data from being served.
  - **@Caching** — Groups multiple cache annotations on a single method. Use when you need to combine put, evict, and cache operations together.
  - **@EnableCaching** — Enables the caching abstraction. Must be added to a `@Configuration` class for caching annotations to be processed.
  - **Annotation Usage:**

    ```java
    @Service
    public class ProductService {

        @Cacheable(value = "products", key = "#id")
        public Product findById(Long id) {
            // Expensive operation — result is cached
            slowMethod();
            return productRepository.findById(id).orElseThrow();
        }

        @Cacheable(value = "products", key = "#category",
                   condition = "#category != 'DISABLED'")
        public List<Product> findByCategory(String category) {
            return productRepository.findByCategory(category);
        }

        @CachePut(value = "products", key = "#product.id")
        public Product update(Product product) {
            // Always executes method AND updates cache
            return productRepository.save(product);
        }

        @CacheEvict(value = "products", key = "#id")
        public void delete(Long id) {
            // Removes entry from cache
            productRepository.deleteById(id);
        }

        @CacheEvict(value = "products", allEntries = true)
        public void clearCache() {
            // Removes ALL entries from products cache
        }

        @Caching(evict = {
            @CacheEvict(value = "products", key = "#product.id"),
            @CacheEvict(value = "productLists", allEntries = true)
        })
        public void complexUpdate(Product product) {
            productRepository.save(product);
        }
    }
    ```

  - **Key Attributes:**
    - **`key`** — SpEL expression for cache key. Default is derived from method parameters using `SimpleKeyGenerator`.
    - **`condition`** — SpEL condition that must be true for caching to occur. Evaluated before method execution.
    - **`unless`** — SpEL condition that prevents caching if true (evaluated after method execution). Useful for not caching null results.
    - **`sync`** — When `true`, only one thread executes the method (others wait for the cached result). Prevents cache stampede under high concurrency.

---

## Core Concepts

### Cache Manager Configuration (Caffeine)

- Caffeine is the recommended local cache provider:

  ```java
  @Configuration
  @EnableCaching
  public class CacheConfig {

      @Bean
      public CacheManager cacheManager() {
          CaffeineCacheManager manager = new CaffeineCacheManager();
          manager.setCaffeine(Caffeine.newBuilder()
              .initialCapacity(100)
              .maximumSize(10_000)
              .expireAfterWrite(5, TimeUnit.MINUTES)
              .recordStats()
          );
          manager.setCacheNames(Arrays.asList("products", "users", "orders"));
          return manager;
      }
  }
  ```

### Per-Cache Configuration

- Different caches with different TTLs and sizes:

  ```java
  @Configuration
  public class FineGrainedCacheConfig {

      @Bean
      public CacheManager cacheManager() {
          return new CaffeineCacheManager() {
              @Override
              protected Cache createCaffeineCache(String name) {
                  return switch (name) {
                      case "products" -> buildCache(name, 1000, 10, TimeUnit.MINUTES);
                      case "users" -> buildCache(name, 5000, 30, TimeUnit.MINUTES);
                      case "orders" -> buildCache(name, 500, 2, TimeUnit.MINUTES);
                      default -> buildCache(name, 100, 5, TimeUnit.MINUTES);
                  };
              }

              private Cache buildCache(String name, int maxSize,
                                       int duration, TimeUnit unit) {
                  return new CaffeineCache(name, Caffeine.newBuilder()
                      .maximumSize(maxSize)
                      .expireAfterWrite(duration, unit)
                      .recordStats()
                      .build());
              }
          };
      }
  }
  ```

### Cache Providers Comparison

| Provider | Use Case | Pros | Cons |
|----------|---------|------|------|
| Caffeine | Local caching | Fast, lightweight, feature-rich | Not distributed |
| Redis | Distributed caching | Shared across instances, TTL, persistence | Network overhead |
| Simple (ConcurrentHashMap) | Dev/test only | No setup needed | No TTL, no eviction, not for production |
| EhCache | Legacy | Feature-rich, disk overflow | Older API |
| Hazelcast | Distributed | In-memory data grid | Complex setup |

---

## Common Mistakes

- **Using `@Cacheable` on methods with side effects** — On a cache hit, the method does not execute, so side effects (event publishing, logging, counters) are silently skipped.
  - Why it looks correct: The annotation says "cacheable" which implies the method is expensive and repeating is wasteful — the side effects are invisible until an audit log is missing.
  - Fix: Use `@CachePut` for methods that must always execute and update the cache.

- **Self-invocation bypassing the cache** — Calling a `@Cacheable` method from within the same class does not trigger caching because the AOP proxy is bypassed.
  - Why it looks correct: The method compiles, runs, and returns the correct result — the cache is simply absent, with no error or warning.
  - Fix: Extract to a separate bean for the cache to work.

- **No TTL or eviction policy** — Cached data becomes stale over time and returns outdated information to users.
  - Why it looks correct: During development with low traffic and fresh data, stale entries never surface — the problem only becomes visible weeks later when a user sees data that should have been updated.
  - Fix: Always set `expireAfterWrite` or use `@CacheEvict` on update operations to keep data fresh.

- **Too large cache causing OOM** — Without `maximumSize`, the cache can grow unbounded and exhaust heap memory.
  - Why it looks correct: The cache works fine with small datasets during development — the OOM only occurs after sustained production traffic with high cardinality keys.
  - Fix: Always set a size limit appropriate for your data volume and available memory.

- **Caching mutable objects** — If the cached object is modified after retrieval, the cached copy changes too, corrupting the cache for all subsequent callers.
  - Why it looks correct: The first caller that modifies the object works correctly — the corruption only manifests when a second caller retrieves what appears to be the same object but sees the first caller's modifications.
  - Fix: Return immutable objects, records, or defensive copies.

- **`@Cacheable` on private methods** — Ignored because the proxy cannot intercept private methods.
  - Why it looks correct: The `@Cacheable` annotation is syntactically valid and the code compiles — Spring provides no warning that private cache annotations are silently ignored.
  - Fix: Use public methods only for caching annotations to be effective.

- **Forgetting `@EnableCaching`** — Without it, no caching annotations are processed and all methods execute unconditionally.
  - Why it looks correct: All methods still return the correct results — they just run the full expensive method every time, which is indistinguishable from correct behavior without profiling.
  - Fix: Always add `@EnableCaching` to a configuration class.

- **Not monitoring cache hit ratio** — Without monitoring, you cannot tune cache sizes and TTLs effectively.
  - Why it looks correct: The application works correctly regardless of hit ratio — the missing performance only becomes known when a load test shows poor throughput.
  - Fix: Enable `recordStats()` on Caffeine and expose cache metrics via Micrometer and Actuator.

---

## Real-World Scenarios

### Scenario 1: E-Commerce Product Catalog with Local Caching

- An e-commerce API serves product details. The same products are requested thousands of times per second. Without caching, each request hits the database. With Caffeine local caching, the database load drops from 10K QPS to 50 QPS.

  ```java
  @Service
  public class ProductService {
      private final ProductRepository productRepository;

      @Cacheable(value = "products", key = "#id", sync = true)
      public Product findById(Long id) {
          // Expensive DB query — called once per product, then cached
          return productRepository.findById(id).orElseThrow();
      }

      @CachePut(value = "products", key = "#product.id")
      public Product update(Product product) {
          return productRepository.save(product);
      }

      @CacheEvict(value = "products", key = "#id")
      public void delete(Long id) {
          productRepository.deleteById(id);
      }
  }
  ```

- With `sync = true`, cache stampede is prevented — when 100 concurrent requests ask for the same uncached product, only one executes the DB query; the other 99 wait for the cached result.

### Scenario 2: Distributed Cache Invalidation Across Microservices

- A product catalog service updates product prices. The search service caches product data independently. After a price update, the search service must be notified to invalidate its cache.

  ```java
  @Service
  public class ProductService {
      private final RedisTemplate<String, Product> redisTemplate;
      private final RabbitTemplate rabbitTemplate;

      @CachePut(value = "products", key = "#product.id")
      public Product updatePrice(Long productId, BigDecimal newPrice) {
          Product updated = productRepository.save(
              productRepository.findById(productId).orElseThrow()
                  .withPrice(newPrice));

          // Invalidate Redis cache for other services
          redisTemplate.delete("product::" + productId);
          // Broadcast invalidation event
          rabbitTemplate.convertAndSend("cache.invalidation", 
              new CacheInvalidationEvent("products", productId));

          return updated;
      }
  }
  ```

### Scenario 3: Multi-Level Caching with Redis + Caffeine

- A high-traffic API uses both local Caffeine (L1) and distributed Redis (L2) caching. This provides sub-millisecond L1 hits for hot data and fallback to L2 for warmer data, reducing Redis load by 80%.

  ```java
  @Configuration
  public class MultiLevelCacheConfig {
      @Bean
      public CacheManager cacheManager(RedisConnectionFactory redisFactory) {
          return new MultiLevelCacheManager(
              caffeineCache(),    // L1: local, super fast
              redisCache(redisFactory) // L2: distributed, shared
          );
      }
  }
  // Custom CacheManager delegates: check L1 first, then L2, then DB
  // On cache put, write to both L1 and L2
  // On cache evict, evict from both
  ```

---

## Use Cases

- Spring's caching abstraction reduces latency and backend load with minimal code changes. These use cases cover cache placement, invalidation strategies, and multi-level caching patterns.

- **Read-heavy data with `@Cacheable`** — A product catalog is queried thousands of times per second and data changes infrequently (price updates a few times daily).
  - Add `@Cacheable(value = "products", key = "#id")` on the lookup method. Use `sync = true` to prevent cache stampede on concurrent misses.
  - **Avoid when:** Data changes every few seconds — the invalidation overhead may exceed the caching benefit.

- **Write-through with `@CachePut`** — A product price update must immediately refresh the cache so subsequent reads see the new value.
  - Use `@CachePut(value = "products", key = "#product.id")` on the update method. The method always executes and the return value replaces the cached entry.
  - **Avoid when:** The update rarely happens and stale data is acceptable — periodic `@CacheEvict` with a scheduler is simpler.

- **Cache eviction on delete** — A `ProductService.delete(id)` must remove the product from cache so deleted products are not returned on subsequent lookups.
  - Use `@CacheEvict(value = "products", key = "#id")`. For clearing entire cache groups, use `@CacheEvict(value = "products", allEntries = true)`.
  - **Avoid when:** The cache has a short TTL — letting the entry expire naturally may be good enough and avoids an explicit eviction code path.

- **Distributed cache invalidation across services** — A product service updates a price, and a search service that caches product data must learn about the change.
  - After updating the local cache with `@CachePut`, publish an invalidation event to a message queue. The search service listens and evicts its own cache entry.
  - **Avoid when:** All services share the same cache store (e.g., Redis) — evicting from the shared store invalidates it for all consumers automatically.

- **Multi-level caching (L1 local + L2 distributed)** — A high-traffic API needs sub-millisecond reads for hot data and fallback to a shared Redis cache for colder data.
  - Configure a custom `CacheManager` that checks Caffeine (L1) first, then Redis (L2), then the database. Writes populate both levels.
  - **Avoid when:** The application is single-instance — local Caffeine alone is simpler and avoids serialization overhead.

- **Distributed caching with Redis** — sharing cache state across multiple application instances in a cluster
  - Spring Cache with Redis as the backing store. Cache entries stored in Redis hashes or JSON serialized. TTL, eviction policies, and cluster configuration at the Redis level.
  - **Avoid when:** cache size is small (<10K entries) and instances are few (<3) — local cache with cache-aside pattern is simpler.

- **Conditional caching for selective optimization** — caching only expensive operations or frequently accessed keys
  - `@Cacheable(condition = "#result.expensive")` caches only if the condition is true. Combine with `unless` to skip caching error results.
  - **Avoid when:** the condition evaluation itself is expensive — cache miss detection overhead may negate benefits.

---

## Scenario-Based Questions

**Q: Your `@Cacheable` method returns a `List<Product>`. You update one product in the database. The cached list still contains the old data. How do you ensure the list cache is invalidated when any product in that list is updated?**
- The granularity problem — caching a list of entities requires invalidation when any entity in the set changes. Options: (a) Use `@CacheEvict(value = "productLists", allEntries = true)` on every create/update/delete — simple but may evict too much. (b) Use a cache key that includes the filter criteria and evict specifically. (c) Use a Redis set for the list and add/remove individual items. Best approach: Separate entity-level caches from list-level caches, and use `@Caching` to evict both:
  ```java
  @Caching(
      put = @CachePut(value = "products", key = "#result.id"),
      evict = @CacheEvict(value = "productLists", allEntries = true)
  )
  public Product save(Product product) { ... }
  ```
- **Interview follow-up:** The candidate suggested `@CacheEvict(value = "productLists", allEntries = true)` on every write. If there are 50 different list caches (by category, by search query, by price range, etc.) and the application performs 1000 writes/second, all 50 list caches are evicted 1000 times/second. Every subsequent read request misses every list cache, causing 50,000 cache misses per second that all hit the database. How would you design a cache invalidation strategy that avoids this mass-eviction cascade?

**Q: You cache the result of a method that returns a mutable object. A caller modifies the returned object. The next caller gets the modified (corrupted) data from the cache. How do you prevent this?**
- Never cache mutable objects directly. Either: (a) Return a defensive copy from the cached method, (b) Make the object immutable (use records), or (c) Clone the object on cache retrieval:
  ```java
  @Cacheable(value = "products", key = "#id")
  public Product findById(Long id) {
      return productRepository.findById(id).orElseThrow();
  }
  // Callers must not modify the returned object
  // Better: return a DTO/record instead of the entity
  @Cacheable(value = "productDTOs", key = "#id")
  public ProductDTO findByIdAsDto(Long id) {
      return ProductDTO.from(productRepository.findById(id).orElseThrow());
  }
  ```

**Q: Your Spring Boot application uses `@Cacheable` but the cache is never populated. Returning values are always null. The repository method works when called without caching. What's wrong?**
- Most likely a self-invocation issue — calling a `@Cacheable` method from within the same class bypasses the proxy. Extract the `@Cacheable` method to a separate bean. Or, the cache manager might not be configured correctly — ensure `@EnableCaching` is on a `@Configuration` class and the `CacheManager` bean is created. Also check that the method is `public` — `@Cacheable` on private methods is silently ignored.

**Q: Your Redis cache performance degrades over time. The cache hit rate drops from 95% to 60%. You suspect memory pressure causes Redis to evict keys. How do you diagnose and fix this?**
- Check Redis eviction stats: `INFO stats` shows `evicted_keys`. Common causes: (a) TTL is too long — stale keys accumulate. (b) No TTL set — keys live forever. (c) Too many unique keys — high cardinality in key generation (e.g., including timestamps in keys). Fix: Set appropriate TTLs, review key generation to exclude high-cardinality values, and configure Redis maxmemory-policy to `allkeys-lru` (least recently used) for a sane eviction strategy.
- **Interview follow-up:** The candidate suggested switching from `noeviction` to `allkeys-lru`. After the change, the hit ratio recovers to 90% but some customers report seeing stale data for 15+ minutes. Redis evicted their product cache keys because they were least recently used, and the next request had to re-fetch from the database — but the database has a write-heavy replica lag of 2 seconds, and the re-fetched data is sometimes not yet visible. The stale data persists because the eviction happened minutes ago and the cache simply has no entry for that key now. What combination of TTL, eviction policy, and write strategy would you use to prevent this scenario?

**Q: You have a service that calls an external weather API. The API rate-limits to 10 calls/minute. Your application serves 100 requests/second. How do you cache the weather data so you never exceed the rate limit?**
- Use `@Cacheable` with a long TTL and `sync = true`:
  ```java
  @Cacheable(value = "weather", key = "#city", sync = true)
  public Weather getWeather(String city) {
      return weatherApiClient.fetch(city); // 1 API call per city per TTL
  }
  ```
- With `sync = true`, only one thread fetches the weather for a given city within the TTL window. All other threads wait and get the cached result. Set TTL to 5-10 minutes. Also add a `@Scheduled` task to refresh popular cities proactively.

**Q: You deploy a new version that changes the structure of a cached object. The old cache entries cause deserialization errors because the class changed. How do you handle cache migration?**
- Strategies: (a) Use a cache prefix with the application version: `@Cacheable(value = "products:v2")` — this creates a fresh cache namespace for each deploy. (b) Use a TTL short enough that old entries expire quickly after deploy. (c) Invalidate all caches on startup via `@EventListener(ContextRefreshedEvent.class)` with `cacheManager.getCache("products").clear()`. (d) Use a serializer that handles version differences (e.g., Jackson with `@JsonIgnoreProperties(ignoreUnknown = true)` for forward compatibility).

**Q: Your `@Cacheable` method uses `condition = "#id > 100"`. When the condition is false (id <= 100), the method executes but the result is not cached. The next call with the same id <= 100 executes the method again. You want to cache the "no-cache" decision too. How?**
- The `condition` attribute controls whether the result IS cached, not whether the method is called. If you want to cache "do not cache" decisions, you cannot use `condition` alone. Instead, always cache but use `unless` to skip caching for specific values:
  ```java
  @Cacheable(value = "products", key = "#id", unless = "#id <= 100")
  public Product findById(Long id) {
      return productRepository.findById(id).orElseThrow();
  }
  ```
- Wait — this still doesn't cache for id <= 100. To cache the "not in cache" result, cache all results but include the condition in the key:
  ```java
  @Cacheable(value = "products", key = "{#id, #root.methodName}")
  ```

**Q: Your application's cache hit ratio is 99% but the database is still under heavy load. How is this possible?**
- The 99% hit ratio means 1% of requests miss the cache and hit the database. If your traffic is 100K QPS, 1% is 1000 DB queries per second, which can still saturate the database. Additionally: (a) The 1% of unique keys might be the most expensive queries. (b) Cache writes (`@CachePut`) also execute the database query. (c) Cache evictions may be too aggressive. Fix: Profile the DB queries that miss the cache and optimize them. Consider a higher-level cache (CDN) for static data. Increase cache size or TTL. Check if `@CachePut` methods are being called unnecessarily.

**Q: You use `@CacheEvict(allEntries = true)` on a method that runs every 5 minutes via `@Scheduled`. User traffic peaks every 4 minutes. The cache is evicted during peak traffic, causing a temporary load spike on the database. How do you smooth this?**
- Don't evict all entries at once. Instead: (a) Use TTL-based expiration instead of explicit eviction — entries expire gradually. (b) Use `@CacheEvict` with a staggered schedule — evict subsets of the cache at different times. (c) Use a write-behind or refresh-ahead pattern — proactively refresh entries before they expire. (d) Increase the eviction interval to align with low-traffic periods. (e) Set a short TTL instead of explicit eviction — entries expire naturally and gradually.

**Q: You migrate from Caffeine to Redis as the cache provider. After the switch, the application is slower than before. What could explain this?**
- Redis adds network latency for every cache get/put — typically 1-5ms per operation, compared to sub-millisecond Caffeine reads. Use a multi-level cache: Caffeine as L1 (local, fast) and Redis as L2 (distributed, slower). Cache hits go to L1 in microseconds; misses fall through to L2. Also ensure Redis is on the same network/region as your application to minimize latency. Consider connection pooling to avoid connection overhead.

---

## Interview Questions

- **What is the difference between `@Cacheable`, `@CachePut`, and `@CacheEvict`?**
  - `@Cacheable` checks the cache first; if a hit exists, returns the cached value without executing the method. `@CachePut` always executes the method and updates the cache with the result. `@CacheEvict` removes entries from the cache. Use `@CachePut` for writes and `@CacheEvict` for deletes.

- **What is cache stampede and how do you prevent it?**
  - Cache stampede occurs when many concurrent requests miss the cache simultaneously and all execute the expensive method. Prevent with `sync = true` on `@Cacheable` — only one thread executes the method; others wait and get the cached result.

- **What is the difference between `condition` and `unless` in `@Cacheable`?**
  - `condition` is evaluated before method execution — if false, the result is not cached (and the method always executes). `unless` is evaluated after method execution — if true, the result is not cached. Use `condition` for argument-based decisions and `unless` for result-based decisions (e.g., `unless = "#result == null"`).

- **What cache providers does Spring Boot support?**
  - Caffeine (recommended for local), Redis (distributed), Simple (ConcurrentHashMap, dev only), EhCache, Hazelcast, JCache (JSR-107). Spring Boot auto-detects the provider from the classpath. Caffeine is the best choice for single-instance apps; Redis for multi-instance.

- **What is the default cache key in `@Cacheable`?**
  - The default key is derived from the method parameters using `SimpleKeyGenerator`. For a method with one parameter, the key is that parameter. For multiple parameters, it's a `SimpleKey` of all parameters. For no parameters, it's `SimpleKey.EMPTY`. Override with `key` attribute using SpEL.

- **What is the self-invocation problem with caching?**
  - Calling a `@Cacheable` method from within the same class bypasses the AOP proxy, so the cache is not checked. The method always executes. Fix by extracting the cached method to a separate bean or using a self-injection pattern.

- **How do you configure TTL per cache?**
  - With Caffeine: `Caffeine.newBuilder().expireAfterWrite(5, TimeUnit.MINUTES)`. Configure per-cache by creating separate `CaffeineCache` instances in the `CacheManager` with different TTLs. With Redis: `@Cacheable(value = "products", cacheManager = "shortTtlCacheManager")` or configure TTL in Redis config per cache name.

- **What is the purpose of `@Caching` annotation?**
  - `@Caching` groups multiple cache annotations on a single method. Use it when a method needs to combine `@Cacheable`, `@CachePut`, and/or `@CacheEvict` — for example, evicting multiple caches when an entity is updated.

- **How do you monitor cache performance?**
  - Enable Micrometer metrics with `Caffeine.recordStats()`. Monitor `cache.gets` (hit/miss ratio), `cache.evictions`, `cache.loads`. Expose via Actuator/Prometheus. A hit ratio below 80% indicates poor cache utilization — increase size or TTL, or review the caching strategy.

- **How do you implement distributed caching across multiple application instances?**
  - Use Redis as a shared cache provider. All application instances point to the same Redis instance/cluster. For cache invalidation, Redis handles it automatically — an update on one instance immediately affects all others. For multi-level caching (Caffeine + Redis), implement a custom `CacheManager` that checks local cache first, then Redis.

---

## Developer Recommendations

- **Use `sync = true` on `@Cacheable` for high-traffic endpoints** — Without sync, 100 concurrent misses all execute the method simultaneously (cache stampede). With sync, only one thread executes; others wait for the cached result. This reduces database load dramatically during cache warming.
- **Always set TTL for cached data** — Without TTL, entries live forever and become stale. Set a reasonable TTL based on data volatility: product prices (5 min), user profiles (30 min), configuration (1 hour). For data that never changes, TTL is still useful as an eviction mechanism.
- **Never cache mutable objects** — If a caller modifies the returned object, the cache is corrupted. Return immutable objects (records), DTOs, or defensive copies. This is especially important when caching entities that may be modified after retrieval.
- **Use `@CacheEvict(allEntries = true)` to invalidate list caches** — When a single entity is updated, all cached lists that might contain it must be evicted. `allEntries = true` clears the entire cache namespace, which is safer than trying to figure out which lists were affected.
- **Avoid caching method results that include the current time or random values** — Keys that include timestamps, UUIDs, or random numbers will never hit the cache because every call generates a unique key. Review your SpEL key expressions for high-cardinality components.
- **Use multi-level caching (Caffeine + Redis) for performance-critical paths** — Caffeine provides sub-millisecond reads (L1), Redis provides shared state across instances (L2). A custom `CacheManager` checks L1 first, then L2, then the database. This reduces Redis load by 80%+.
- **Warm the cache after deployment** — The first request after a deploy always misses the cache. Use `@EventListener(ContextRefreshedEvent.class)` or a startup runner to pre-populate the cache with frequently accessed data. This prevents the "cold start" performance degradation.
- **Monitor cache hit ratio and set up alerts** — A sudden drop in hit ratio indicates a problem: code change that invalidates keys, TTL that is too short, or a cache eviction policy that is too aggressive. Set up Prometheus alerts for `cache.hit.ratio < 0.8`.
