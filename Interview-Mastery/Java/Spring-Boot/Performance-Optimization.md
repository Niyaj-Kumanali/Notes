# Spring Boot Performance Optimization

---

## 1. Executive Summary

### Key Optimization Areas
1. **Startup time** — initialization, classpath scanning, bean creation
2. **Request latency** — slow endpoints, DB queries, external calls
3. **Memory usage** — heap, off-heap, GC pressure
4. **Throughput** — concurrent request handling
5. **Resource usage** — threads, connections, file handles

---

## 2. Core Strategies

### 2.1 Startup Optimization

```yaml
# application.yml
spring:
  autoconfigure:
    exclude:  # Exclude unused auto-configurations
      - org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration
      - org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration
  jpa:
    open-in-view: false  # Disable OSIV (significant performance cost)
```

```java
// Lazy initialization (Spring Boot 2.2+)
spring.main.lazy-initialization=true
```

```java
// Component scanning optimization
@SpringBootApplication
@ComponentScan(basePackages = {"com.company.order", "com.company.common"})
public class Application {}
```

### 2.2 Connection Pool Configuration (HikariCP)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10      # Rule: max(connections) = Tn × (Cm - 1) + 1
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1200000
      pool-name: OrderServicePool
```

**Formula:** `connections = ((core_count * 2) + effective_spindle_count)`

### 2.3 JPA Performance

```java
// BAD: N+1 queries
@OneToMany(fetch = FetchType.EAGER)  // Avoid — cartesian product
private List<OrderLine> lines;

// GOOD: Explicit fetch plans
@Query("SELECT o FROM Order o JOIN FETCH o.lines JOIN FETCH o.customer")
List<Order> findAllWithDetails();

// GOOD: Batch fetching
@BatchSize(size = 20)
@OneToMany
private List<OrderLine> lines;

// GOOD: Read-only queries
@Transactional(readOnly = true)
public List<Product> findAll() {
    // Hibernate skips dirty checking — faster
}

// GOOD: DTO Projections
@Query("SELECT new com.example.OrderSummary(o.id, o.total, o.status) FROM Order o")
List<OrderSummary> findAllSummaries();
```

### 2.4 Caching Strategy

```java
@Configuration
public class PerformanceCacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .recordStats()
        );
        return manager;
    }
}
```

### 2.5 Thread Pool Configuration

```java
@Bean(name = "taskExecutor")
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(25);
    executor.setQueueCapacity(200);
    executor.setThreadNamePrefix("worker-");
    executor.setRejectedExecutionHandler(new CallerRunsPolicy());
    executor.setWaitForTasksToCompleteOnShutdown(true);
    executor.initialize();
    return executor;
}

@Bean(name = "asyncExecutor")
public Executor asyncExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);
    executor.setMaxPoolSize(10);
    executor.setQueueCapacity(50);
    executor.setThreadNamePrefix("async-");
    executor.initialize();
    return executor;
}
```

### 2.6 Response Compression

```yaml
server:
  compression:
    enabled: true
    mime-types: text/html,text/xml,text/plain,application/json
    min-response-size: 1024
```

### 2.7 HTTP/2

```yaml
server:
  http2:
    enabled: true
```

### 2.8 Undertow (replace Tomcat for better throughput)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

### 2.9 JVM Tuning

```bash
# Production JVM options
java -Xms2g -Xmx2g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 \
     -XX:G1HeapRegionSize=16m \
     -XX:+ParallelRefProcEnabled \
     -XX:+UseStringDeduplication \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/app/heapdump.hprof \
     -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,pid:filecount=10,filesize=10m \
     -jar app.jar
```

---

## 3. Profiling & Monitoring

### JVM Metrics to Watch

| Metric | Alert Threshold | What It Means |
|--------|----------------|---------------|
| GC pause time | > 200ms | G1GC tuning needed |
| GC frequency | > 5/min | Heap too small or memory leak |
| Heap usage | > 80% after GC | Increase heap or reduce leak |
| Thread count | > baseline * 2 | Thread leak or pool too large |
| Connection pool usage | > 80% | Pool too small or slow queries |
| 99th percentile latency | > 500ms | Bottleneck in request processing |

### Spring Boot Actuator + Micrometer for Prometheus

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true
```

### Custom Performance Metrics

```java
@Component
public class PerformanceMetrics {
    private final MeterRegistry registry;
    private final Map<String, Timer> timers = new ConcurrentHashMap<>();
    
    public PerformanceMetrics(MeterRegistry registry) {
        this.registry = registry;
    }
    
    public <T> T time(String operation, Supplier<T> action) {
        Timer timer = timers.computeIfAbsent(operation, 
            name -> registry.timer("app.operation.duration", "operation", name));
        return timer.record(action);
    }
}
```

---

## 4. Common Bottlenecks

| # | Issue | Symptom | Solution |
|---|-------|---------|----------|
| 1 | N+1 queries | Slow list endpoints | JOIN FETCH / EntityGraph |
| 2 | Missing indexes | Slow DB queries | Add composite indexes |
| 3 | No caching | Same data fetched repeatedly | @Cacheable |
| 4 | Large JSON responses | High latency | DTO projections, pagination |
| 5 | Synchronous blocking I/O | Thread pool exhaustion | Async, reactive |
| 6 | OSIV (open-in-view) | Connection held for view rendering | spring.jpa.open-in-view=false |
| 7 | Full table scans | Slow queries | EXPLAIN + index |
| 8 | Serialization bottleneck | High CPU | Flat DTOs, Jackson optimization |
| 9 | Memory leak | OOM after days | Heap dump analysis |
| 10 | Classpath scanning | Slow startup | Explicit @ComponentScan |

---

## 5. Cheat Sheet

```
═══ PERFORMANCE OPTIMIZATION ═══════════════════════════════════

┌─ QUICK WINS ───────────────────────────────────────────────┐
│ ❌ spring.jpa.open-in-view=true    → false                  │
│ ❌ FetchType.EAGER                 → LAZY + JOIN FETCH     │
│ ❌ Entity → DTO in service         → JPQL constructor      │
│ ❌ No cache                        → @Cacheable            │
│ ❌ Tomcat                          → Undertow              │
│ ❌ No compression                  → server.compression    │
└─────────────────────────────────────────────────────────────┘

┌─ JVM OPTIONS ──────────────────────────────────────────────┐
│ -Xms2g -Xmx2g          — Heap size (min=max for G1GC)      │
│ -XX:+UseG1GC           — Garbage-First collector           │
│ -XX:MaxGCPauseMillis=100 — Target max pause                │
│ -XX:+UseStringDeduplication — Dedup duplicate strings      │
│ -Xlog:gc*:file=gc.log  — GC logging                       │
└─────────────────────────────────────────────────────────────┘

┌─ DATABASE ─────────────────────────────────────────────────┐
│ • Add indexes on WHERE, JOIN, ORDER BY columns             │
│ • Composite indexes for multi-column queries                │
│ • Avoid SELECT * (use DTO projections)                     │
│ • Paginate list endpoints                                   │
│ • Connection pool: HikariCP (max 10-20 per instance)       │
│ • Read replicas for read-heavy workloads                    │
└─────────────────────────────────────────────────────────────┘

┌─ PROFILE BEFORE OPTIMIZING ────────────────────────────────┐
│ 1. Measure: Actuator / Prometheus / Grafana                │
│ 2. Profile: Async Profiler, JFR, YourKit                   │
│ 3. Database: EXPLAIN ANALYZE, slow query log               │
│ 4. GC: GC logs → GCeasy / GCViewer                        │
│ 5. Fix the bottleneck, NOT the symptom                     │
└─────────────────────────────────────────────────────────────┘
```
