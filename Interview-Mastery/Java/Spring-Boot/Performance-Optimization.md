# Spring Boot Performance Optimization

---

## Overview

- **Definition:** Spring Boot Performance Optimization encompasses strategies for improving startup time, request latency, memory usage, throughput, and resource utilization. Optimization requires identifying actual bottlenecks through profiling before making changes.
- **Why It Exists:** Spring Boot applications in production must handle increasing load, scale efficiently, and provide fast responses. Without deliberate optimization, even well-architected applications degrade under load due to N+1 queries, connection pool exhaustion, serialization bottlenecks, and other common issues.
- **Key Concepts:**
  - **Optimization Areas:** Performance optimization spans five dimensions: 
    1. startup time (how quickly the application serves requests), 
    2. request latency (individual endpoint response speed), 
    3. memory usage (heap and off-heap consumption with GC pressure), 
    4. throughput (concurrent request capacity), and
    5. resource usage (thread pool and connection pool saturation). 
    Identify the specific area causing pain before choosing a strategy.
  - **Startup Optimization:** Reduce startup time by excluding unused auto-configurations, limiting `@ComponentScan` to specific packages, disabling OSIV (`spring.jpa.open-in-view=false`), and enabling lazy initialization (`spring.main.lazy-initialization=true`). For Spring Boot 3.x, AOT compilation can dramatically reduce startup time for Kubernetes deployments.
    ```yaml
    # application.yml
    spring:
      autoconfigure:
        exclude:
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
  - **Connection Pool Configuration (HikariCP):** Configure HikariCP with `maximum-pool-size` based on the formula `connections = ((core_count * 2) + effective_spindle_count)`, starting with 10 and monitoring. Set `minimum-idle` to handle traffic spikes, `connection-timeout` to fail fast when the pool is exhausted, and `max-lifetime` to recycle connections before database-enforced timeouts.
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
  - **Caching Strategy:** Reduce repeated expensive operations by caching computationally heavy or I/O-bound results. Use Caffeine for single-instance caching with bounded size and TTL, and Redis for distributed caching. Always configure `maximumSize`, `expireAfterWrite`, and `recordStats` to monitor cache effectiveness.
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

### JPA Performance

- The most common source of performance issues in Spring Boot applications:
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

### Thread Pool Configuration

- Separate pools for different workloads:
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

### Response Compression

- Reduce response size over the wire:
  ```yaml
  server:
    compression:
      enabled: true
      mime-types: text/html,text/xml,text/plain,application/json
      min-response-size: 1024
  ```

### Undertow Instead of Tomcat

- Undertow offers better throughput under high concurrency:
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

### JVM Tuning

- Production JVM options for G1GC:
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

## Common Mistakes

- **N+1 queries**
  - **Why it looks correct:** Each individual query succeeds and returns data — the performance problem is invisible without SQL logging or profiling.
  - **Fix:** Use `JOIN FETCH` or `@EntityGraph` to load all required data in a single query. Slow list endpoints caused by fetching entities and accessing lazy associations in a loop.

- **Missing database indexes**
  - **Why it looks correct:** Queries return correct results and are fast in development with small datasets — the scans only become slow as the table grows to millions of rows.
  - **Fix:** Add composite indexes based on query patterns and use `EXPLAIN ANALYZE` to verify index usage. Full table scans on WHERE, JOIN, or ORDER BY columns cause performance degradation as tables grow.

- **No caching**
  - **Why it looks correct:** The application works correctly and returns fresh data every time — the performance loss is invisible without monitoring query volume.
  - **Fix:** Add `@Cacheable` with appropriate TTL and monitor cache hit ratio to ensure effectiveness. Same data fetched repeatedly from the database without a cache layer.

- **Large JSON responses**
  - **Why it looks correct:** The endpoint returns valid JSON and the data is correct — the excessive payload size and serialization time only become apparent under load or on slow networks.
  - **Fix:** Use DTO projections and pagination to reduce payload size and serialization overhead. Full entities serialized as JSON include unnecessary columns and trigger lazy associations.


- **Synchronous blocking I/O**
  - **Why it looks correct:** With low concurrency, each request completes within the expected time — the thread exhaustion only occurs when concurrent requests exceed the pool size.
  - **Fix:** Use `@Async` for truly parallelizable tasks or reactive programming for I/O-bound services. REST calls, file I/O, or database operations block Tomcat threads, causing thread pool exhaustion under load.

- **Open Session in View (OSIV)**
  - **Why it looks correct:** Lazy associations work without `LazyInitializationException` and developers see it as a convenient feature — the hidden cost is holding a database connection for the entire response serialization.
  - **Fix:** Set `spring.jpa.open-in-view=false` to force explicit fetch planning and release connections earlier. Database connection held for the entire HTTP request, including view rendering and serialization.

- **Full table scans**
  - **Why it looks correct:** The query returns the right data and modern hardware makes small scans fast — the scan only becomes a problem when the table outgrows the buffer pool.
  - **Fix:** Use `EXPLAIN ANALYZE` to identify scans and add appropriate indexes on filtered and joined columns. Slow queries on large tables that scan every row instead of using indexes.

- **Serialization bottleneck**
  - **Why it looks correct:** Serialization succeeds and the response is valid JSON — the 400ms spent serializing is invisible when the total response time is under a second.
  - **Fix:** Use flat DTOs, optimize Jackson configuration with `@JsonView`, and consider Protocol Buffers for latency-critical APIs. Jackson serialization of complex object graphs with circular references and lazy associations.

- **Memory leaks**
  - **Why it looks correct:** The application starts fine and passes all tests — the gradual memory growth is only detectable through monitoring over hours or days.
  - **Fix:** Perform heap dump analysis with Eclipse MAT to identify leak suspects. Application crashes with OOM after days of running due to unbounded caches, thread-local accumulation, or classloader leaks.

- **Classpath scanning**
  - **Why it looks correct:** The application starts successfully and serves requests — the 4-minute startup time is only a problem when Kubernetes kills the pod before it becomes ready.
  - **Fix:** Use explicit `@ComponentScan` with specific base packages and exclude unused auto-configurations. Slow startup with large codebases that scan many packages.

---

## Real-World Scenarios

### Scenario 1: Startup Optimization for a Kubernetes-Deployed Microservice

- A microservice takes 4 minutes to start. Kubernetes kills the pod after 60 seconds (liveness probe timeout). The team needs to reduce startup to under 30 seconds.
  ```yaml
  spring:
    autoconfigure:
      exclude:
        - org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration
        - org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration
    main:
      lazy-initialization: true
    jpa:
      open-in-view: false

  @SpringBootApplication
  @ComponentScan(basePackages = {"com.company.orderservice"})
  public class Application {}
  ```
- Plus JVM tuning: Use `-XX:+UseContainerSupport`, `-Xms512m -Xmx512m` (smaller initial heap), and AOT compilation with Spring Boot 3.x AOT engine. Result: startup drops from 240s to 18s.

### Scenario 2: N+1 Query Fix in a Reporting Endpoint

- A reporting dashboard shows 500 orders with customer names and product details. The endpoint takes 12 seconds and generates 1001 SQL queries. The fix: eliminate N+1 and add DTO projection.
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
- Result: 1 query, 200ms response, 5MB heap instead of 200MB.

### Scenario 3: Connection Pool Tuning for a Payment Processing Service

- A payment service processes 500 transactions/second. During peak traffic (Black Friday), transactions start failing with "Connection is not available" after 30 seconds. The HikariCP pool is configured with default values.
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
        connection-timeout: 5000
        idle-timeout: 600000
        max-lifetime: 1800000
        pool-name: PaymentPool
        leak-detection-threshold: 60000
  ```
- Also added monitoring: HikariCP metrics exposed via Actuator for Grafana dashboards. After tuning, the service handles 1500 TPS without connection exhaustion.

## Use Cases

- **Startup time reduction** — speeding up Spring Boot application initialization in production
  - Exclude unused auto-configurations, enable lazy initialization, use `@ConditionalOn*` annotations judiciously, and consider AOT compilation (Spring Native / GraalVM).
  - **Avoid when:** the application starts once and runs for weeks — startup time matters less for long-lived processes.

- **Database connection pool tuning** — preventing connection exhaustion under high load
  - Size the pool based on concurrent requests and query duration. HikariCP's `maximumPoolSize` should account for database max connections across all application instances.
  - **Avoid when:** the application rarely touches the database — a small pool (5–10 connections) is sufficient.

- **HTTP client optimization** — reducing latency for outbound API calls from Spring services
  - Use connection pooling (HttpClient, RestTemplate with PoolingHttpClientConnectionManager). Enable keep-alive. Set socket and connection timeouts. Use async non-blocking I/O with WebClient.
  - **Avoid when:** the service calls only one external API with low volume — a simple RestTemplate with default settings is adequate.

- **Caching with Spring Cache abstraction** — reducing repeated expensive computations or database calls
  - `@Cacheable` with TTL and eviction policies. Multi-tier caching (Caffeine L1 + Redis L2). Cache-aside pattern. Avoid cache stampede with mutex-based reload.
  - **Avoid when:** the data changes on every request — caching adds overhead without benefit.

- **JVM tuning and GC optimization** — reducing pause times and memory churn in latency-sensitive services
  - Choose the right GC (G1GC for predictable pause times, Shenandoah/ZGC for sub-10ms pauses). Tune heap size, young generation sizing, and GC threads. Use JMH for microbenchmarks.
  - **Avoid when:** the service has no latency SLO — default JVM settings may be adequate for background processing services.

---

## Scenario-Based Questions

1. **Q: Your Spring Boot application starts in 4 minutes in production but 30 seconds locally. Both use the same codebase. What's different and how do you diagnose?**
   - Production likely has slower disks (classpath scanning is I/O-bound), more auto-configuration classes matched, Hibernate schema validation on a large database, or network-attached config files. Diagnose by adding `-Dspring.autoconfigure.logging=true` to see which auto-configurations match and enabling `logging.level.org.springframework.boot=DEBUG`. Fix by excluding unused auto-configurations, using explicit `@ComponentScan`, setting `spring.jpa.hibernate.ddl-auto=none` if schema is managed externally, and enabling AOT compilation.

2. **Q: Your API response time P95 is 2 seconds. The P50 is 200ms. What's causing the tail latency and how do you fix it?**
   - A large gap between P50 and P95 indicates occasional slow requests from GC pauses, cache misses after TTL expiry, thread contention behind a slow request, or database query plan changes. Tune GC with G1GC and `MaxGCPauseMillis=50`, add cache warming via `@EventListener(ContextRefreshedEvent.class)`, use async processing for slow paths, and pin query plans. Profile with JFR or Async Profiler to identify the actual cause.
   - **Interview follow-up:** The candidate suggested profiling with JFR to find the root cause. After profiling, the P95 spike is traced to a specific external API call that times out 5% of the time (30s timeout). The timeout is unavoidable — the external API occasionally has 30s pauses. The client library has no built-in circuit breaker. One slow external API call blocks a Tomcat thread for 30 seconds, which then blocks other requests that need the same thread pool. The P95 covers all endpoints, not just the one calling the external API. How would you isolate the slow external API so it doesn't affect the P95 of other, unrelated endpoints?

3. **Q: You deploy a Spring Boot 3.x application and notice memory usage is 30% higher than Spring Boot 2.x for the same code. What changed?**
   - Spring Boot 3.x uses virtual threads by default with Tomcat, which adds virtual thread scheduler and carrier thread management overhead. Spring 6 also uses more records and sealed classes internally, increasing object allocation. Check if `spring.threads.virtual.enabled=true` is set and profile allocation rates with JFR between versions. Consider disabling virtual threads if memory is constrained.

4. **Q: Your database queries are fast (5ms each), but your API endpoint takes 500ms. You discover that Jackson serialization of 50 JPA entities takes 400ms. How do you optimize?**
   - Jackson serialization time is proportional to object graph complexity — every lazy association and field adds overhead. Fix by using DTO projections with only the needed fields, applying `@JsonView` to limit serialization per endpoint, pre-serializing and caching JSON strings for static data, and using Jackson's `afterburner` module to optimize reflection. Enable compression for large responses. DTOs serialize 5-10x faster than entities due to smaller object graphs.

5. **Q: Your application has 20 `@OneToMany` associations with `FetchType.EAGER`. Every entity load fetches huge Cartesian products. The team is afraid to change to LAZY because it might break code that accesses these associations outside transactions. How do you migrate safely?**
   - Migrate gradually by first enabling OSIV temporarily (`spring.jpa.open-in-view=true`), changing one association at a time from EAGER to LAZY, adding `@EntityGraph` on queries that need the association, then finally disabling OSIV. Fix any `LazyInitializationException` by adding explicit fetch plans. Use ArchUnit to ban `FetchType.EAGER` on any new code to prevent regression.

6. **Q: Your application's memory usage grows by 100MB/hour in production. Heap dumps show the largest object is a `HashMap` inside a library you cannot modify. You suspect a cache without TTL. How do you confirm and mitigate?**
   - Take two heap dumps 1 hour apart and compare with Eclipse MAT's Leak Identification report. If the library's internal cache is the growth source, use a larger initial heap to give the cache room, wrap the library with a `@Bean` that periodically calls its cleanup method via `@Scheduled`, or expose the cache size as a metric via `MeterBinder` with a Grafana alert. As a last resort, use reflection to clear the cache periodically.

7. **Q: You replace Tomcat with Undertow expecting better throughput. Instead, throughput drops by 20%. Your application is CPU-bound, not I/O-bound. Why did Undertow underperform?**
   - Undertow is optimized for I/O-bound workloads with non-blocking IO, but for CPU-bound applications, Tomcat's thread-per-request model outperforms because it avoids non-blocking IO management overhead. Undertow's event-driven model adds CPU overhead for managing I/O channels that provide no benefit when the bottleneck is CPU. Switch back to Tomcat and focus on optimizing actual computation through caching, algorithm optimization, or parallelization.

8. **Q: Your service calls 3 external APIs during request processing. The APIs take 200ms, 300ms, and 500ms respectively. The total response time is 1000ms (sequential calls). How do you reduce this to 500ms?**
   - Parallelize the external calls using `CompletableFuture` with a separate thread pool sized for I/O-bound tasks. With all three calls executing concurrently, the response time drops from 1000ms to 500ms — limited by the slowest API. Use a dedicated executor for external calls to avoid starving the main request handling threads.

9. **Q: Your application has frequent GC pauses (2 per second, 100ms each). Users experience response time spikes during pauses. You cannot increase heap size. What do you do?**
   - Reduce object allocation rate by avoiding object creation in hot paths — reuse buffers, use `StringBuilder` instead of concatenation, and use `LongAdder` instead of `AtomicLong`. Use primitive collections (Eclipse Collections, FastUtil) to avoid boxing, pool expensive objects like byte arrays and JSON parsers, and tune G1GC with `-XX:G1NewSizePercent=5 -XX:G1MaxNewSizePercent=40`. For sub-millisecond pause times, use ZGC (Java 17+) at the cost of higher CPU usage.

10. **Q: You optimize a critical endpoint from 2 seconds to 200ms. Two weeks later, it's back to 1 second. You check git history — no code changes to that endpoint. What happened?**
    - Performance regression without code changes is usually data growth or configuration drift. Check if database table size has grown significantly (queries fast on 100K rows may be slow on 10M rows), index fragmentation, cache hit ratio drops, connection pool contention from more concurrent users, or upstream service degradation. Add performance regression tests to CI/CD that alert if P95 response time exceeds a threshold.
    - **Interview follow-up:** The candidate suggested performance regression tests in CI/CD. The test passes because it runs against a fixed dataset of 1000 rows in the CI environment, but the production database has 10M rows. The test assertion monitors P95 response time at 200ms +/- 50ms and always passes because the CI dataset is small. How would you design performance tests that detect data-volume regressions when the CI dataset cannot match production scale?

---

## Interview Questions

- **What are the most common causes of slow Spring Boot applications?**
  - A: N+1 queries, missing database indexes, no caching, large JSON serialization (returning entities instead of DTOs), synchronous blocking I/O inside transactions, OSIV holding connections, unbounded thread pools, and excessive classpath scanning at startup.

- **How do you identify performance bottlenecks in a Spring Boot application?**
  - A: Use a systematic approach: Actuator metrics for system-level health, database slow query log, JFR or Async Profiler for CPU/memory profiling, thread dumps for contention, GC logs for pause analysis, and APM tools (Datadog, New Relic, Grafana) for production monitoring.

- **What is the OSIV (Open Session in View) anti-pattern?**
  - A: OSIV keeps the Hibernate session open for the entire HTTP request, including view rendering and JSON serialization, which holds a database connection longer than necessary and silently triggers N+1 queries in templates or serialization. Disable it with `spring.jpa.open-in-view=false` and load all required data explicitly in the service layer.

- **How do you optimize startup time?**
  - A: Exclude unused auto-configurations, limit `@ComponentScan` to specific packages, use `@Lazy` for expensive beans, set `spring.main.lazy-initialization=true`, disable OSIV, use Spring Boot 3.x AOT engine, and configure JVM container support with `-XX:+UseContainerSupport`.

- **What is the N+1 query problem and how do you fix it?**
  - A: The N+1 problem occurs when you fetch N entities and then access their lazy associations in a loop, generating N extra queries. Fix with `JOIN FETCH`, `@EntityGraph`, DTO projections (best), or `@BatchSize`. Enable Hibernate statistics with `spring.jpa.properties.hibernate.generate_statistics=true` to detect N+1.

- **How does connection pool size affect performance?**
  - A: A pool that is too small causes requests to queue waiting for connections, while a pool that is too large creates database overhead from managing many connections and thread contention. Use the formula `connections = ((core_count * 2) + effective_spindle_count)`, starting with 10-20 and monitoring utilization. PostgreSQL can handle approximately 100 connections; beyond that, use PgBouncer.

- **What is the difference between Tomcat and Undertow?**
  - A: Tomcat uses a thread-per-request model where each request gets a dedicated thread, making it better for CPU-bound applications. Undertow uses non-blocking IO with fewer threads, making it better for I/O-bound applications with many concurrent connections. Spring Boot defaults to Tomcat.

- **How do you reduce JSON serialization time?**
  - A: Use DTOs instead of entities for a smaller object graph, apply `@JsonView` to limit fields per endpoint, enable compression, use Jackson's `afterburner` module, cache pre-serialized JSON for static data, and consider Protocol Buffers or Avro for extremely latency-sensitive APIs.

- **How do you tune JVM garbage collection for a Spring Boot application?**
  - A: Use G1GC (`-XX:+UseG1GC`) as the default choice and tune `-XX:MaxGCPauseMillis=100` for latency. Adjust heap size based on live data set size. For low-latency applications (p99 < 10ms), use ZGC (`-XX:+UseZGC`) which has sub-millisecond pause times. Enable GC logging with `-Xlog:gc*:file=gc.log` and analyze with GCeasy.

- **How do you implement caching for performance?**
  - A: Use `@Cacheable` on service methods with Caffeine for single-instance caching (fast, local) or Redis for distributed caching. Configure TTL, max size, and `sync = true` for cache stampede protection. Monitor hit ratio targeting over 90% and warm the cache after deployment. Use multi-level caching for critical paths with Caffeine as L1 and Redis as L2.

---

## Developer Recommendations

- **Always use DTO projections for read operations**
  - Returning JPA entities serializes all columns and triggers lazy associations. DTOs select only the needed fields, reducing memory, network, and serialization overhead. Use constructor expressions in JPQL or interface-based projections for compile-time safety.

- **Disable OSIV (`spring.jpa.open-in-view=false`) in production**
  - OSIV holds database connections through JSON serialization and masks N+1 problems. Disabling it forces explicit fetch planning, making performance behavior predictable. The default is `true` in Spring Boot 2.x, so opt out explicitly.

- **Profile before optimizing**
  - The most common performance mistake is optimizing the wrong thing. Use JFR and Async Profiler to find actual bottlenecks — a 10% improvement to a method that accounts for 1% of total time is wasted effort. Measure, identify, then optimize.

- **Use `@Cacheable` with `sync = true` for high-traffic endpoints**
  - Without sync, 100 concurrent cache misses all hit the database simultaneously. With sync, only one thread executes the method while the rest wait for the cached result, preventing cache stampede in high-concurrency scenarios.

- **Keep transactions short**
  - Long transactions hold database connections and locks, increasing contention and reducing throughput. Never perform blocking I/O (REST calls, file I/O) inside a transaction — the transaction should only cover database operations with external calls placed outside.

- **Use pagination for list endpoints**
  - Returning unbounded lists causes OOM under load, slow serialization, and poor user experience. Always paginate with `Pageable` and return `Page<T>` with sensible defaults like `page=0, size=20` and a maximum limit of 1000.

- **Use explicit `@ComponentScan` and exclude unused auto-configurations**
  - Classpath scanning and auto-configuration matching are significant startup bottlenecks. Limit scanning to packages your code actually uses and exclude auto-configurations that don't apply, such as MongoDB when you use JPA or Flyway when you use Liquibase.

- **Monitor performance metrics in production**
  - Without metrics, performance optimization is guesswork. Track P50/P95/P99 response times, GC pause frequency and duration, connection pool utilization, thread pool saturation, and cache hit ratio. Use Micrometer with Prometheus and Grafana to set up alerts for regressions.
