# Spring Boot Caching

---

## 1. Executive Summary

Spring Boot provides a caching abstraction via `@Cacheable`, `@CacheEvict`, `@CachePut`, and `@Caching`. The abstraction supports multiple cache providers: Caffeine, Redis, EhCache, Hazelcast, and Simple (concurrent HashMap).

---

## 2. Core Theory

### Annotations

```java
@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        // Expensive operation — result is cached
        // Subsequent calls with same id return cached value
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

### Cache Manager Configuration (Caffeine)

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

// Custom per-cache configuration
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
            
            private Cache buildCache(String name, int maxSize, int duration, TimeUnit unit) {
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

---

## 3. Common Mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | @Cacheable on methods with side effects | Side effects lost on cache hit | @CachePut for writes |
| 2 | Self-invocation (calling cached method from same class) | Cache bypassed | Extract to separate bean |
| 3 | No TTL / eviction | Stale data | Set expireAfterWrite |
| 4 | Too large cache | OOM | Set maximumSize |
| 5 | Caching mutable objects | Cache poisoning | Return immutable or defensive copy |
| 6 | @Cacheable on private methods | Ignored | Use public |
| 7 | Not enabling @EnableCaching | Nothing cached | Add to @Configuration |
| 8 | No cache monitoring | Can't tune | recordStats() + Actuator |

---

## 4. Cheat Sheet

```
═══ SPRING BOOT CACHING ═══════════════════════════════════════

┌─ ANNOTATIONS ──────────────────────────────────────────────┐
│ @Cacheable("cacheName")       — cache method result         │
│ @CachePut("cacheName")        — update cache                │
│ @CacheEvict("cacheName")      — remove from cache           │
│ @CacheEvict(allEntries=true)  — clear entire cache          │
│ @Caching(...)                 — combine multiple            │
│ @EnableCaching                — enable (on @Configuration)  │
└─────────────────────────────────────────────────────────────┘

┌─ KEY ATTRIBUTES ───────────────────────────────────────────┐
│ key = "#id"                 — SpEL key expression           │
│ key = "#root.args[0]"       — first argument               │
│ condition = "#id > 100"     — conditionally cache          │
│ unless = "#result == null"  — don't cache if result null   │
│ sync = true                 — synchronized cache load       │
└─────────────────────────────────────────────────────────────┘

┌─ CACHE PROVIDERS ──────────────────────────────────────────┐
│ Caffeine     — Best for local caching (in-memory)          │
│ Redis        — Distributed caching                         │
│ Simple       — ConcurrentHashMap (default, dev only)       │
│ EhCache      — Legacy, feature-rich                         │
│ Hazelcast    — Distributed, with other features            │
└─────────────────────────────────────────────────────────────┘

┌─ BEST PRACTICES ───────────────────────────────────────────┐
│ • Cache read-heavy, write-rare data                        │
│ • Always set TTL (expireAfterWrite)                        │
│ • Monitor cache hit ratio (Actuator)                       │
│ • Use sync=true for concurrent cache load prevention       │
│ • Cache immutable or defensive copies                      │
│ • Evict related caches on updates                          │
│ • Distributed caches (Redis) for multi-instance apps       │
└─────────────────────────────────────────────────────────────┘
```
