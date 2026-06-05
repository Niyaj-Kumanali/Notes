# Spring Boot Performance Optimization

---

## What is Performance Optimization?

**Spring Boot Performance Optimization** encompasses strategies for improving startup time, request latency, memory usage, throughput, and resource utilization. Optimization requires identifying actual bottlenecks through profiling before making changes.

### Key Concepts:

1. **Optimization Areas**:

   - **Startup Time** — How quickly the application becomes ready to serve requests.
   - **Request Latency** — How fast individual endpoints respond.
   - **Memory Usage** — Heap and off-heap memory consumption, GC pressure.
   - **Throughput** — How many concurrent requests the application can handle.
   - **Resource Usage** — Thread pool utilization, connection pool saturation, file handles.

2. **Startup Optimization**:

   ```yaml
   # application.yml
   spring:
     autoconfigure:
       exclude:  # Exclude unused auto-configurations
         - org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration
         - org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration
     jpa:
       open-in-view: false  # Disable OSIV (significant performance cost)

   # Lazy initialization (Spring Boot 2.2+)
   spring.main.lazy-initialization=true
   ```

   ```java
   // Limit component scanning
   @SpringBootApplication
   @ComponentScan(basePackages = {"com.company.order", "com.company.common"})
   public class Application {}
   ```

3. **Connection Pool Configuration (HikariCP)** :

   ```yaml
   spring:
     datasource:
       hikari:
         maximum-pool-size: 10
         minimum-idle: 5
         idle-timeout: 300000
         connection-timeout: 20000
         max-lifetime: 1200000
         pool-name: OrderServicePool
   ```

   A common formula for pool size: `connections = ((core_count * 2) + effective_spindle_count)`. Start with 10 and monitor.

4. **Caching Strategy**:

   Reduce repeated expensive operations:

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

---

## Core Concepts

### 1. JPA Performance

   The most common source of performance issues in Spring Boot applications:

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

   // GOOD: Read-only queries (Hibernate skips dirty checking)
   @Transactional(readOnly = true)
   public List<Product> findAll() { /* ... */ }

   // GOOD: DTO Projections (avoid full entity load)
   @Query("SELECT new com.example.OrderSummary(o.id, o.total, o.status) FROM Order o")
   List<OrderSummary> findAllSummaries();
   ```

### 2. Thread Pool Configuration

   Separate pools for different workloads:

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

### 3. Response Compression

   Reduce response size over the wire:

   ```yaml
   server:
     compression:
       enabled: true
       mime-types: text/html,text/xml,text/plain,application/json
       min-response-size: 1024
   ```

### 4. Undertow Instead of Tomcat

   Undertow offers better throughput under high concurrency:

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

### 5. JVM Tuning

   Production JVM options for G1GC:

   ```bash
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

## Common Bottlenecks

1. **N+1 queries** — Slow list endpoints. Fix with `JOIN FETCH` or `@EntityGraph`.

2. **Missing database indexes** — Full table scans on WHERE/JOIN/ORDER BY columns. Add composite indexes.

3. **No caching** — Same data fetched repeatedly. Add `@Cacheable` with appropriate TTL.

4. **Large JSON responses** — Full entities serialized as JSON. Use DTO projections and pagination.

5. **Synchronous blocking I/O** — Thread pool exhaustion under load. Use `@Async` or reactive programming.

6. **Open Session in View (OSIV)** — Database connection held for view rendering. Set `spring.jpa.open-in-view=false`.

7. **Full table scans** — Slow queries on large tables. Use `EXPLAIN ANALYZE` and add indexes.

8. **Serialization bottleneck** — Jackson serialization of complex object graphs. Use flat DTOs and optimize Jackson configuration.

9. **Memory leaks** — OOM after days of running. Perform heap dump analysis.

10. **Classpath scanning** — Slow startup with large codebases. Use explicit `@ComponentScan`.

---

## Real-World Scenarios

### Scenario 1: Startup Optimization for a Kubernetes-Deployed Microservice

A microservice takes 4 minutes to start. Kubernetes kills the pod after 60 seconds (liveness probe timeout). The team needs to reduce startup to under 30 seconds.

```yaml
# Step 1: Exclude unused auto-configurations
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration
      - org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration
  main:
    lazy-initialization: true  # Step 2: Defer bean creation
  jpa:
    open-in-view: false        # Step 3: Disable OSIV

# Step 4: Limit component scan
@SpringBootApplication
@ComponentScan(basePackages = {"com.company.orderservice"})
public class Application {}
```

Plus JVM tuning: Use `-XX:+UseContainerSupport`, `-Xms512m -Xmx512m` (smaller initial heap), and AOT compilation with Spring Boot 3.x AOT engine. Result: startup drops from 240s to 18s.

### Scenario 2: N+1 Query Fix in a Reporting Endpoint

A reporting dashboard shows 500 orders with customer names and product details. The endpoint takes 12 seconds and generates 1001 SQL queries. The fix: eliminate N+1 and add DTO projection.

```java
// Before: N+1 — 1 query for orders + 500 for customers + 500 for products
@GetMapping("/reports/orders")
public List<Order> getOrders() {
    return orderRepository.findAll(); // 1001 queries
}

// After: Single optimized query with DTO projection
@Query("""
    SELECT new com.app.dto.OrderReport(
        o.id, o.orderDate, c.name, p.name, o.total
    ) FROM Order o
    JOIN o.customer c
    JOIN o.product p
    ORDER BY o.orderDate DESC
    """)
Page<OrderReport> findOrderReports(Pageable pageable);

@GetMapping("/reports/orders")
public ResponseEntity<Page<OrderReport>> getOrders(Pageable pageable) {
    return ResponseEntity.ok(orderRepository.findOrderReports(pageable));
}
```

Result: 1 query, 200ms response, 5MB heap instead of 200MB.

### Scenario 3: Connection Pool Tuning for a Payment Processing Service

A payment service processes 500 transactions/second. During peak traffic (Black Friday), transactions start failing with "Connection is not available" after 30 seconds. The HikariCP pool is configured with default values.

```yaml
# Before: default HikariCP (10 connections)
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      connection-timeout: 30000

# After: optimized for throughput
spring:
  datasource:
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 5000       # Fail fast if no connection
      idle-timeout: 600000           # 10 minutes
      max-lifetime: 1800000          # 30 minutes
      pool-name: PaymentPool
      leak-detection-threshold: 60000 # Log if connection held > 60s
```

Also added monitoring: HikariCP metrics exposed via Actuator for Grafana dashboards. After tuning, the service handles 1500 TPS without connection exhaustion.

---

## Scenario-Based Questions

1. **Q: Your Spring Boot application starts in 4 minutes in production but 30 seconds locally. Both use the same codebase. What's different and how do you diagnose?**
   A: Production likely has: (a) Slower disks — classpath scanning is I/O-bound. (b) More auto-configuration classes matched (different classpath). (c) Hibernate schema validation on a large database. (d) Network-attached config files. Diagnose by: adding `-Dspring.autoconfigure.logging=true` to see which auto-configurations match, enabling `logging.level.org.springframework.boot=DEBUG`, and using `-XX:+PrintClassHistogram` at startup. Fix: exclude unused auto-configurations, use explicit `@ComponentScan`, set `spring.jpa.hibernate.ddl-auto=none` if schema is managed externally, and enable AOT compilation.

2. **Q: Your API response time P95 is 2 seconds. The P50 is 200ms. What's causing the tail latency and how do you fix it?**
   A: A large gap between P50 and P95 indicates occasional slow requests. Common causes: (a) GC pauses — check GC logs for stop-the-world pauses. (b) Cache misses — first request after TTL expiry is slow (cold start). (c) Thread contention — requests queuing behind a slow one. (d) Database query plan changes — some queries use different plans. Fix: (a) Tune GC (use G1GC, set `MaxGCPauseMillis=50`). (b) Add cache warming (`@EventListener(ContextRefreshedEvent.class)`). (c) Use async processing for slow paths. (d) Pin query plans with `pg_hint_plan` or SQL Server plan guides. Profile with JFR/Async Profiler to identify the actual cause.

3. **Q: You deploy a Spring Boot 3.x application and notice memory usage is 30% higher than Spring Boot 2.x for the same code. What changed?**
   A: Spring Boot 3.x (based on Spring 6 / Java 17+) uses virtual threads (Project Loom) by default with Tomcat. Virtual threads have smaller stacks but the virtual thread scheduler and carrier thread management add overhead. Also, Spring 6 uses more records and sealed classes internally which may increase object allocation. Check: (a) Is `spring.threads.virtual.enabled=true` set? Virtual threads use less memory for idle threads but can increase allocation rate. (b) Is Micrometer's observation API enabled (adds per-request overhead). Profile with JFR to compare allocation rates between versions. Consider disabling virtual threads if memory is constrained.

4. **Q: Your database queries are fast (5ms each), but your API endpoint takes 500ms. You discover that Jackson serialization of 50 JPA entities takes 400ms. How do you optimize?**
   A: Jackson serialization time is proportional to object graph complexity. Every lazy association, every `@JsonBackReference`, and every field adds overhead. Fix: (a) Use DTO projections — serialize flat objects with only the needed fields. (b) Use `@JsonView` to define serialization views. (c) Pre-serialize with Jackson's `ObjectMapper` and cache the JSON string. (d) Use Jackson's `afterburner` module (optimizes getter/setter reflection). (e) If response is large, enable compression:
   ```yaml
   server.compression.enabled=true
   server.compression.min-response-size=1024
   ```
   Tests show DTOs serialize 5-10x faster than entities due to smaller object graphs.

5. **Q: Your application has 20 `@OneToMany` associations with `FetchType.EAGER`. Every entity load fetches huge Cartesian products. The team is afraid to change to LAZY because it might break code that accesses these associations outside transactions. How do you migrate safely?**
   A: Gradual migration: (1) Add `spring.jpa.open-in-view=true` temporarily (keeps sessions open for the request). (2) Change ONE association from EAGER to LAZY and run all tests. (3) Add `@EntityGraph` on queries that need the association. (4) Remove OSIV (`spring.jpa.open-in-view=false`). (5) Fix any `LazyInitializationException` by adding explicit fetch plans. This approach lets you migrate incrementally without breaking existing code. Use ArchUnit to ban `FetchType.EAGER` on any new code.

6. **Q: Your application's memory usage grows by 100MB/hour in production. Heap dumps show the largest object is a `HashMap` inside a library you cannot modify. You suspect a cache without TTL. How do you confirm and mitigate?**
   A: Take two heap dumps 1 hour apart and compare with Eclipse MAT's "Leak Identification" report. If the library's internal cache is the growth source: (a) Use `-XX:+AlwaysPreTouch` and larger initial heap to give the cache room without triggering GC. (b) Use `@Bean` to wrap the library and periodically call its cleanup method via `@Scheduled`. (c) Use a `MeterBinder` to expose the cache size as a metric and set up a Grafana alert. (d) If the library supports configuration, pass a bounded cache via its API. As a last resort, use reflection to clear the cache periodically (not ideal but functional).

7. **Q: You replace Tomcat with Undertow expecting better throughput. Instead, throughput drops by 20%. Your application is CPU-bound, not I/O-bound. Why did Undertow underperform?**
   A: Undertow is optimized for I/O-bound workloads with its non-blocking IO. For CPU-bound applications, Tomcat's thread-per-request model often outperforms because it doesn't pay the overhead of non-blocking IO management. Undertow's event-driven model adds CPU overhead for managing I/O channels that provide no benefit when the bottleneck is CPU. Switch back to Tomcat. The embedded server choice matters less for CPU-bound apps — focus on optimizing the actual computation (caching, algorithm optimization, parallelization).

8. **Q: Your service calls 3 external APIs during request processing. The APIs take 200ms, 300ms, and 500ms respectively. The total response time is 1000ms (sequential calls). How do you reduce this to 500ms?**
   A: Parallelize the external calls using `CompletableFuture`:
   ```java
   public ServiceResponse handle(Request request) {
       CompletableFuture<ApiAResult> futureA = CompletableFuture
           .supplyAsync(() -> apiA.call(request), apiExecutor);
       CompletableFuture<ApiBResult> futureB = CompletableFuture
           .supplyAsync(() -> apiB.call(request), apiExecutor);
       CompletableFuture<ApiCResult> futureC = CompletableFuture
           .supplyAsync(() -> apiC.call(request), apiExecutor);
       return CompletableFuture.allOf(futureA, futureB, futureC)
           .thenApply(v -> combine(futureA.join(), futureB.join(), futureC.join()))
           .join(); // Total time = max(200, 300, 500) = 500ms
   }
   ```
   Use a separate thread pool (`apiExecutor`) sized for I/O-bound tasks. With 3 parallel calls, the response time drops from 1000ms to 500ms — limited by the slowest API.

9. **Q: Your application has frequent GC pauses (2 per second, 100ms each). Users experience response time spikes during pauses. You cannot increase heap size. What do you do?**
   A: Reduce object allocation rate — this is the root cause of frequent GC. Techniques: (a) Avoid creating objects in hot paths — reuse buffers, use `StringBuilder` instead of `+`, use `LongAdder` instead of `AtomicLong`. (b) Use primitive collections (Eclipse Collections, FastUtil) instead of boxing. (c) Pool expensive objects (byte arrays, JSON parsers). (d) Use `@JsonView` to reduce serialization objects. (e) Tune G1GC: `-XX:G1NewSizePercent=5 -XX:G1MaxNewSizePercent=40` to control young generation sizing. (f) Use ZGC (Java 17+) which has sub-millisecond pause times regardless of heap size — trade CPU for lower latency. Profile allocation with `-XX:+PrintStringTableStatistics` and JFR.

10. **Q: You optimize a critical endpoint from 2 seconds to 200ms. Two weeks later, it's back to 1 second. You check git history — no code changes to that endpoint. What happened?**
    A: Performance regression without code changes is usually data growth or configuration drift. Check: (a) Database table size — has the table grown 10x? Queries that were fast on 100K rows may be slow on 10M rows. (b) Index fragmentation — rebuild indexes. (c) Cache efficiency — has the cache hit ratio dropped? Check `cache.hit.ratio` metric. (d) Connection pool — is pool contention growing due to more concurrent users? (e) External API — has the upstream service degraded? Add performance regression tests to CI/CD that alert if P95 response time exceeds a threshold.

---

## Interview Questions

1. **What are the most common causes of slow Spring Boot applications?** 
   A: N+1 queries, missing database indexes, no caching, large JSON serialization (returning entities instead of DTOs), synchronous blocking I/O inside transactions, OSIV holding connections, unbounded thread pools, and excessive classpath scanning at startup.

2. **How do you identify performance bottlenecks in a Spring Boot application?** 
   A: Use a systematic approach: (1) Actuator metrics (`/actuator/metrics`) for system-level health. (2) Database slow query log. (3) JFR (JDK Flight Recorder) or Async Profiler for CPU/memory profiling. (4) Thread dumps for contention. (5) GC logs for pause analysis. (6) APM tools (Datadog, New Relic, Grafana).

3. **What is the OSIV (Open Session in View) anti-pattern?** 
   A: OSIV keeps the Hibernate session open for the entire HTTP request, including view rendering and JSON serialization. This holds a database connection longer than necessary and triggers N+1 queries silently in templates/serialization. Disable it with `spring.jpa.open-in-view=false` and load all required data explicitly in the service layer.

4. **How do you optimize startup time?** 
   A: Exclude unused auto-configurations, limit `@ComponentScan` to specific packages, use `@Lazy` for expensive beans, set `spring.main.lazy-initialization=true`, disable OSIV, use Spring Boot 3.x AOT engine, and configure JVM container support (`-XX:+UseContainerSupport`).

5. **What is the N+1 query problem and how do you fix it?** 
   A: The N+1 problem occurs when you fetch N entities and then access their lazy associations in a loop, generating N extra queries. Fix with `JOIN FETCH`, `@EntityGraph`, DTO projections (best), or `@BatchSize`. Enable Hibernate statistics to detect N+1: `spring.jpa.properties.hibernate.generate_statistics=true`.

6. **How does connection pool size affect performance?** 
   A: Too small → requests queue waiting for connections. Too large → database overhead from managing many connections, thread contention. Formula: `connections = ((core_count * 2) + effective_spindle_count)`. Start with 10-20 and monitor. PostgreSQL can handle ~100 connections; beyond that, use PgBouncer for connection pooling.

7. **What is the difference between Tomcat and Undertow?** 
   A: Tomcat uses a thread-per-request model (each request gets a dedicated thread). Undertow uses non-blocking IO with fewer threads. Tomcat is better for CPU-bound applications; Undertow is better for I/O-bound applications with many concurrent connections. Spring Boot default is Tomcat.

8. **How do you reduce JSON serialization time?** 
   A: (1) Use DTOs instead of entities (smaller object graph). (2) Use `@JsonView` to limit fields per endpoint. (3) Enable compression. (4) Use Jackson's `afterburner` module. (5) Cache pre-serialized JSON for static data. (6) Use Protocol Buffers or Avro for extremely latency-sensitive APIs.

9. **How do you tune JVM garbage collection for a Spring Boot application?** 
   A: Use G1GC (`-XX:+UseG1GC`) as default. Tune `-XX:MaxGCPauseMillis=100` for latency. Adjust heap size based on live data. For low-latency apps (p99 < 10ms), use ZGC (`-XX:+UseZGC`) which has sub-millisecond pauses. Enable GC logging: `-Xlog:gc*:file=gc.log`. Analyze with GCeasy.

10. **How do you implement caching for performance?** 
    A: Use `@Cacheable` on service methods. Choose Caffeine for single-instance (fast, local), Redis for distributed. Set TTL, max size, and `sync = true` for cache stampede protection. Monitor hit ratio — target > 90%. Warm the cache after deployment. Use multi-level caching for critical paths (Caffeine L1 + Redis L2).

---

## Developer Recommendations

- **Always use DTO projections for read operations** — Returning JPA entities serializes all columns and triggers lazy associations. DTOs select only the needed fields, reducing memory, network, and serialization overhead. Use constructor expressions in JPQL or interface-based projections.
- **Disable OSIV (`spring.jpa.open-in-view=false`) in production** — OSIV holds database connections through JSON serialization and masks N+1 problems. Disabling it forces explicit fetch planning, making performance behavior predictable. The default is `true` in Spring Boot 2.x — opt out explicitly.
- **Profile before optimizing** — The most common performance mistake is optimizing the wrong thing. Use JFR + Async Profiler to find actual bottlenecks. A 10% improvement to a method that accounts for 1% of total time is wasted effort. Measure, identify, then optimize.
- **Use `@Cacheable` with `sync = true` for high-traffic endpoints** — Without sync, 100 concurrent misses all hit the database. With sync, only one thread executes the method; the rest wait for the cached result. This is critical for cache stampede prevention.
- **Keep transactions short** — Long transactions hold database connections and locks, increasing contention. Never perform blocking I/O (REST calls, file I/O) inside a transaction. The transaction should only cover the database operations — external calls go outside.
- **Use pagination for list endpoints** — Returning unbounded lists causes OOM under load, slow serialization, and poor user experience. Always paginate with `Pageable` and return `Page<T>`. Set sensible defaults: `page=0, size=20` with a max limit of 1000.
- **Use explicit `@ComponentScan` and exclude unused auto-configurations** — Classpath scanning and auto-configuration matching are startup bottlenecks. Limit scanning to packages your code actually uses. Exclude auto-configurations that don't apply to your application (MongoDB if you use JPA, Flyway if you use Liquibase).
- **Monitor performance metrics in production** — Without metrics, performance optimization is guesswork. Track P50/P95/P99 response times, GC pause frequency/duration, connection pool utilization, thread pool saturation, and cache hit ratio. Set up alerts for regressions. Use Micrometer + Prometheus + Grafana.
