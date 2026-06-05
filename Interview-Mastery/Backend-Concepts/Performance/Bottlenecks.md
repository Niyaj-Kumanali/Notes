# Bottlenecks

## 1. Executive Summary

A bottleneck is a point in a system where limited capacity constrains overall performance. In backend systems, bottlenecks can occur at any layer: network, CPU, memory, disk I/O, database, thread pools, or external services. Identifying and resolving bottlenecks is the core of performance engineering. Bottleneck analysis follows a systematic approach: measure, identify, prioritize, resolve, and verify.

## 2. Core Theory

### Universal Scalability Law (USL)

The USL models performance under load:

```
C(N) = N / (1 + sigma * (N - 1) + kappa * N * (N - 1))
```
Where:
- C(N) = Relative capacity with N processors
- sigma = Contention (serialization overhead)
- kappa = Coherency (cross-processor communication overhead)

### Types of Bottlenecks

1. **Compute-bound (CPU)**: Processing power is the limiting factor.
2. **Memory-bound**: RAM capacity or bandwidth is insufficient.
3. **I/O-bound**: Disk or network I/O limits throughput.
4. **Contention-bound**: Locks, mutexes, or shared resources cause serialization.
5. **Queueing-bound**: Request queue fills up, increasing latency.

### The Bottleneck Chain

A system's throughput is determined by its slowest component (the bottleneck). Fixing one bottleneck reveals the next one.

```
Original bottleneck: Database queries (100ms)
After optimization: CPU (80% -> 95% utilization)
Next bottleneck: Thread pool contention
After optimization: Network bandwidth
```

## 3. Under-the-Hood Deep Dive

### CPU Bottlenecks

**Symptoms:**
- High CPU utilization (>90%)
- Context switching rate > 10,000/s
- Runnable threads queueing (high "load average")

**Common Causes:**
- Inefficient algorithms (N^2 instead of N log N)
- Excessive serialization/deserialization
- Regex operations on large strings
- Tight loops without optimization
- Garbage collection

**Diagnosis:**
```bash
# Linux
top -H -p <pid>          # Thread-level CPU
perf top -p <pid>        # CPU sampling
pidstat -p <pid> 1       # CPU stats per second

# Java
jstack <pid>             # Thread dump (check RUNNABLE threads)
jcmd <pid> Thread.print  # Alternative thread dump
```

### Memory Bottlenecks

**Symptoms:**
- High memory usage, frequent GC
- OutOfMemoryError
- Swap usage (paging to disk)
- Increasing GC time

**Common Causes:**
- Memory leaks (unclosed resources, static collections)
- Oversized caches
- Large object allocations
- Fragmentation

**Diagnosis:**
```bash
# Memory monitoring
jmap -heap <pid>                 # Heap summary
jmap -histo:live <pid>           # Class instances histogram
jcmd <pid> GC.heap_info          # GC details

# GC logging
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log
```

### Disk I/O Bottlenecks

**Symptoms:**
- High disk utilization (>90%)
- High I/O wait time
- Long database query times
- Slow file operations

**Common Causes:**
- Missing indexes (full table scans)
- Large sequential scans
- Inefficient queries (N+1, Cartesian products)
- Undersized storage (IOPS limit)

### Network Bottlenecks

**Symptoms:**
- High network utilization
- Packet loss / retransmissions
- Socket timeouts
- Connection pool exhaustion

**Common Causes:**
- Small TCP buffer sizes
- TLS overhead
- Chatty inter-service communication
- Insufficient bandwidth

### Database Bottlenecks

**Symptoms:**
- Slow queries (long execution time)
- Lock contention (deadlocks, lock waits)
- Connection pool exhaustion
- Replication lag

## 4. Production Code Examples

### Profiling with Spring Boot Actuator

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,threaddump,heapdump,flyway
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
```

### Thread Pool Monitoring

```java
@Configuration
public class ThreadPoolMonitorConfig {

    @Bean
    public ThreadPoolTaskExecutor taskExecutor(MeterRegistry registry) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("biz-");
        executor.setRejectedExecutionHandler(new CallerRunsPolicy());
        executor.initialize();

        // Monitor pool metrics
        MonitoredThreadPoolExecutor.monitor(executor.getThreadPoolExecutor(), registry);

        return executor;
    }
}

@Component
public class MonitoredThreadPoolExecutor {

    public static void monitor(ThreadPoolExecutor executor, MeterRegistry registry) {
        Gauge.builder("threadpool.active.count", executor, ThreadPoolExecutor::getActiveCount)
            .tag("pool", "biz-pool")
            .register(registry);

        Gauge.builder("threadpool.queue.size", executor, e -> e.getQueue().size())
            .tag("pool", "biz-pool")
            .register(registry);

        Gauge.builder("threadpool.pool.size", executor, ThreadPoolExecutor::getPoolSize)
            .tag("pool", "biz-pool")
            .register(registry);

        Gauge.builder("threadpool.rejected.count", executor,
            e -> (long) e.getRejectedExecutionHandler().getClass().getSimpleName().hashCode())
            .tag("pool", "biz-pool")
            .register(registry);
    }
}
```

### Database Query Bottleneck Detection

```java
@Component
public class SlowQueryLogger {

    @Autowired
    private DataSource dataSource;

    @EventListener(ApplicationReadyEvent.class)
    public void enableSlowQueryLogging() {
        if (dataSource instanceof HikariDataSource hikari) {
            // Log queries slower than 100ms
            hikari.setConnectionInitSql("SET log_min_duration_statement = 100");
        }
    }
}

// JPA query logging for debugging
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        session:
          events:
            log: true

// Using Hibernate Query Plan Cache
@Bean
public HibernatePropertiesCustomizer queryPlanCacheCustomizer() {
    return props -> {
        props.put("hibernate.query.plan_cache_max_size", 2048);
        props.put("hibernate.query.plan_parameter_metadata_max_size", 64);
    };
}
```

### CPU Bottleneck Detection Example

```java
@Service
public class CpuIntensiveOperation {

    private static final int PARALLELISM = Runtime.getRuntime().availableProcessors();

    public List<Report> generateReports(List<Data> dataList) {
        // Parallel processing to maximize CPU utilization
        return dataList.parallelStream()
            .map(this::processData) // CPU-bound operation
            .collect(Collectors.toList());
    }

    @Async("cpuIntensiveExecutor")
    public CompletableFuture<Report> processData(Data data) {
        long start = System.nanoTime();
        Report report = performHeavyComputation(data);
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        if (elapsed > 1000) {
            log.warn("Slow CPU computation: {}ms for data id={}", elapsed, data.getId());
        }
        return CompletableFuture.completedFuture(report);
    }

    @Bean("cpuIntensiveExecutor")
    public Executor cpuIntensiveExecutor() {
        // Use parallelism equal to CPU cores for CPU-bound tasks
        return Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors());
    }
}
```

### Memory Leak Prevention

```java
@Component
public class SafeCachingService {

    // WRONG - memory leak
    // private static final Map<String, Product> productCache = new HashMap<>();

    // RIGHT - bounded cache with eviction
    private final Cache<String, Product> productCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofHours(1))
        .recordStats()
        .build();

    // RIGHT - weak references for caches
    private final Map<String, Product> weakCache = new WeakHashMap<>();

    public Product getProduct(String id) {
        return productCache.get(id, this::loadFromDb);
    }

    private Product loadFromDb(String id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found"));
    }
}
```

## 5. Real-World Scenarios

### E-Commerce Black Friday Bottleneck

**Initial Bottleneck:** Database CPU at 100%
- Root cause: N+1 queries for order items.
- Fix: Batch fetching, JOINs.

**Secondary Bottleneck:** Connection pool exhausted
- Root cause: Queries took too long holding connections.
- Fix: Query optimization + connection pool sizing.

**Tertiary Bottleneck:** Redis hot key
- Root cause: Single product page (iPhone) getting 80% of traffic.
- Fix: Local cache for hot keys, read replicas.

### Social Media Feed Bottleneck

**Bottleneck:** Feed generation too slow
- Root cause: On-read fan-out: generating feed per request.
- Fix: Switch to fan-out-on-write: pre-compute feeds on post.

**Secondary:** Write throughput limited
- Root cause: Redis list push per follower.
- Fix: Async write-behind, batch updates.

## 6. Performance

### Bottleneck Detection Tools

| Tool | Type | What it Detects |
|------|------|-----------------|
| top/htop | CPU/Memory | High CPU, memory, swap |
| iostat | Disk | I/O wait, IOPS, throughput |
| netstat/ss | Network | Connections, retransmits |
| jstack | Java | Thread states, deadlocks |
| jmap | Java | Memory, instances |
| async-profiler | Java | CPU sampling, wall clock |
| Prometheus | Metrics | All metrics |
| Grafana | Dashboard | Visualization |
| Jaeger/Zipkin | Tracing | Latency per span |
| Database EXPLAIN | SQL | Query execution plan |

### Bottleneck Resolution Priority

1. **Low-hanging fruit**: Missing indexes, connection pools, obvious N+1 queries.
2. **Architecture changes**: Caching, async processing, read replicas.
3. **Code optimization**: Algorithm improvement, better data structures.
4. **Hardware scaling**: More CPU, memory, faster disks.
5. **Architecture redesign**: CQRS, event sourcing, service decomposition.

## 7. Security

### Security Bottleneck Considerations

**TLS/SSL**: Handshake overhead adds latency. Use TLS session resumption, terminate at edge.

**Authentication**: JWT validation per request. Cache public keys for signature verification.

**Rate Limiting**: Can become bottleneck if Redis calls per request. Use local rate limiters synced periodically.

**Input Validation**: Heavy regex validation on large payloads. Set limits on request size.

## 8. Common Mistakes

### Mistake 1: Premature Optimization
Optimizing code before measuring. "Make it work, make it right, make it fast - in that order."

### Mistake 2: Ignoring the Obvious
Checking for missing indexes before tuning JVM GC settings.

### Mistake 3: Local-Only Testing
Bottlenecks often only appear under real production load (concurrency, data volume).

### Mistake 4: Single Metric Focus
CPU is high -> add more CPU. But the actual bottleneck may be I/O waiting.

### Mistake 5: Assuming Linearity
Doubling servers does not double throughput (Amdahl's Law/USL).

## 9. Senior Engineer Perspective

### Systematic Bottleneck Analysis Process

1. **Define the goal**: What throughput/latency do we need?
2. **Measure current state**: Baseline metrics.
3. **Identify bottleneck**: Which resource is saturated?
4. **Hypothesize cause**: Why is it saturated?
5. **Fix**: Make one change at a time.
6. **Measure impact**: Did it improve? Did it shift bottleneck?
7. **Repeat**: Find the next bottleneck.

### Bottleneck Budget

```java
public class BottleneckBudget {

    // Define per-resource budgets
    public static final double CPU_TARGET = 0.70;      // 70% utilization target
    public static final double MEMORY_TARGET = 0.80;   // 80%
    public static final double DISK_TARGET = 0.60;     // 60% (IOPS)
    public static final double NETWORK_TARGET = 0.50;  // 50% bandwidth
    public static final double POOL_TARGET = 0.70;     // 70% pool usage

    public static boolean isConstrained(String resource, double utilization) {
        return switch (resource) {
            case "cpu" -> utilization > CPU_TARGET;
            case "memory" -> utilization > MEMORY_TARGET;
            case "disk" -> utilization > DISK_TARGET;
            case "network" -> utilization > NETWORK_TARGET;
            case "pool" -> utilization > POOL_TARGET;
            default -> false;
        };
    }
}
```

### Predictive Bottleneck Detection
- Track growth rate of metrics (database size, request rate).
- Model when each resource will reach capacity.
- Schedule upgrades before saturation.

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What is a bottleneck in system performance?
   **A:** A point in the system where limited capacity constrains overall throughput or latency.

2. **Q:** Name five types of bottlenecks.
   **A:** CPU, memory, disk I/O, network, database.

3. **Q:** How do you identify a CPU bottleneck?
   **A:** High CPU utilization (>90%), high context switching, runnable threads queueing.

4. **Q:** What is Amdahl's Law?
   **A:** Speedup = 1 / ((1-P) + P/N). Limits how much parallel execution can improve performance.

5. **Q:** What tool shows Java thread states?
   **A:** jstack or jcmd Thread.print.

6. **Q:** What is a memory leak?
   **A:** Objects that are no longer needed but are still referenced, preventing garbage collection.

7. **Q:** How does a missing database index cause a bottleneck?
   **A:** It forces full table scans instead of index lookups, increasing I/O and CPU.

8. **Q:** What is connection pool exhaustion?
   **A:** All connections in a pool are in use, causing new requests to wait.

9. **Q:** What is the N+1 query problem?
   **A:** One query to fetch N entities, then N additional queries for each entity's related data.

10. **Q:** What is the difference between latency and throughput bottleneck?
    **A:** Latency bottleneck: single operation is too slow. Throughput bottleneck: system cannot handle enough concurrent operations.

### Medium

11. **Q:** How do you find the root cause of a CPU bottleneck?
    **A:** Use CPU profiling (async-profiler, perf), capture thread dumps, sort by CPU time, identify hot methods.

12. **Q:** What is GC overhead and how does it become a bottleneck?
    **A:** GC overhead is time spent garbage collecting. When >10% of CPU, it's a bottleneck. Caused by excessive allocation, large heap, memory leaks.

13. **Q:** How do you diagnose database query bottlenecks?
    **A:** Enable slow query log, use EXPLAIN ANALYZE, check for missing indexes, monitor lock contention.

14. **Q:** Explain how to resolve the N+1 query problem in JPA.
    **A:** Use JOIN FETCH, EntityGraph, or batch fetching (@BatchSize).

15. **Q:** What is the USL and how does it relate to bottlenecks?
    **A:** Universal Scalability Law models throughput degradation due to contention and coherency overhead. Predicts the bottleneck from scaling.

16. **Q:** How does thread pool sizing become a bottleneck?
    **A:** Too few threads: CPU underutilized. Too many: context switching overhead, memory waste. Optimal: CPU-bound = core count, IO-bound = higher.

17. **Q:** What is I/O wait and how do you diagnose it?
    **A:** Time CPU spends waiting for I/O operations. Diagnose with iostat, check disk utilization, IOPS, await time. Look for slow storage, missing indexes.

18. **Q:** How do network bottlenecks manifest?
    **A:** High latency, packet loss, socket timeouts, connection resets, low throughput despite available CPU.

19. **Q:** What is the bottleneck in a synchronous REST call chain?
    **A:** The slowest service in the chain. Threads block waiting for response, causing thread pool exhaustion.

20. **Q:** How do you handle cascading bottlenecks?
    **A:** Identify the primary bottleneck, fix it, measure. The next bottleneck will appear. Repeat.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Design a bottleneck detection system that automatically identifies the limiting resource.
    **A:** Agent collects per-node metrics (CPU, memory, disk, network, thread pools, DB). Correlation engine matches patterns: high CPU + low I/O = CPU bound. High I/O wait + queue depth = I/O bound. System generates bottleneck report with confidence score.

2. **Q:** How do you identify a bottleneck caused by JVM GC in production?
    **A:** Monitor: GC pause duration, frequency, CPU overhead. GC logs: check for Full GC events, promotion failures, concurrent mode failures. JFR events for detailed GC analysis.

3. **Q:** Design a thread pool configuration that auto-tunes for bottleneck avoidance.
    **A:** Monitor throughput, latency, pool utilization. If queue builds = too few threads (or upstream too slow). If threads constantly active + CPU high = CPU bound, decrease threads. If threads constantly active + I/O wait = I/O bound, increase threads.

4. **Q:** How do you detect hot key bottlenecks in a distributed cache?
    **A:** Monitor per-key access frequency. If single key receives >N req/s, it's hot. Detection: Redis --hotkeys, custom key-level metrics in application. Mitigation: local cache, replicate key, split key.

5. **Q:** Explain how coordinated omission distorts bottleneck analysis.
    **A:** Coordinated omission: when measurement system stops measuring during overload, slow requests are excluded. This makes performance look better than reality. Fix: use HDR Histogram, measure all requests including failures and timeouts.

6. **Q:** Design a system that predicts bottlenecks before they occur.
    **A:** Time-series forecasting on metrics. Machine learning model trained on historical data (request rate, resource utilization). Predict when resource will reach capacity. Proactive scaling before saturation.

7. **Q:** How do you identify a bottleneck in a message queue (Kafka)?
    **A:** Monitor consumer lag (increasing = consumer bottleneck). Monitor producer request latency. Check partition count vs consumer count. Check disk utilization on brokers. Check network throughput.

8. **Q:** What are the bottlenecks specific to microservices vs monoliths?
    **A:** Microservices: network latency, serialization overhead, distributed transaction complexity, service discovery latency, inter-service auth overhead. Monolith: single process scaling limits, shared resource contention.

9. **Q:** How do you diagnose a bottleneck that only happens under peak load?
    **A:** Production load testing with traffic mirroring. Chaos engineering: gradually increase load while monitoring. JFR continuous profiling with low overhead.

10. **Q:** Design a system that automatically resolves common bottlenecks without human intervention.
    **A:** Auto-scaler: increases instances when CPU/throughput threshold exceeded. Auto-indexer: detects slow queries, recommends/create indexes. Auto-cache: identifies repeated queries, adds caching. Auto-thread-pool: adjusts pool based on utilization.

### System Design

11. **Q:** Design a system to detect and resolve database bottleneck in real-time.
    **A:** Monitor: query latency, lock waits, connection pool usage, replication lag. Auto-resolution: add read replicas for read bottleneck, kill long-running queries, promote replica for write bottleneck.

12. **Q:** Design a load testing framework to identify bottlenecks.
    **A:** Distributed load generators (Gatling/Locust). Start at low concurrency, gradually increase. Monitor all resources. Identify first resource to saturate. Report bottleneck latency/throughput curve.

13. **Q:** Design a system for bottleneck visualization across 100+ microservices.
    **A:** Service graph from distributed tracing. Color-code services by bottleneck type (red=CPU, blue=memory, yellow=DB). Show throughput and latency on edges. Aggregate by deployment, region. Root cause drill-down.

14. **Q:** Design a database sharding strategy that avoids hot partition bottlenecks.
    **A:** Hash-based sharding with consistent hashing. Virtual nodes for even distribution. Monitor partition size/load. Rebalance hot partitions by splitting or moving. Application-level routing.

15. **Q:** Design a CDN strategy that prevents origin server bottlenecks during traffic spikes.
    **A:** Origin shield to aggregate cache misses. Pre-warm CDN before expected spikes. Auto-scale origin behind CDN. Rate limit origin access per CDN edge. Circuit breaker when origin saturated.

16. **Q:** Design an anti-entropy system for cache bottlenecks.
    **A:** Cache with local (L1) + distributed (L2). L1: per-node Caffeine, TTL 60s. L2: Redis cluster. Cache-aside pattern. Stampede prevention: distributed locks for cache misses.

17. **Q:** Design a system that eliminates database as bottleneck for read-heavy workloads.
    **A:** Cache at multiple levels: CDN, Redis, in-memory. Read replicas with load balancing. CQRS: separate read/write models. Materialized views. Eventual consistency for non-critical reads.

18. **Q:** Design a queue-based throttling system to prevent upstream bottlenecks.
    **A:** Bounded work queue with rejection policy. Thread pool executor with caller-runs policy for backpressure. Circuit breakers to fail-fast when upstream saturated.

19. **Q:** Design a multi-region deployment that avoids cross-region latency bottlenecks.
    **A:** Regional deployments with local databases. CRDT-based or event-driven cross-region sync. DNS routing to nearest region. Global read replicas.

20. **Q:** Design a bottleneck detection system using eBPF.
    **A:** eBPF programs attached to kernel probes: trace TCP retransmits, file I/O, page cache, run queue, syscall latency. Export metrics to Prometheus. Dashboard showing resource-level bottleneck indicators.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design an automated bottleneck analysis system that provides root cause in natural language.
    **A:** AI-based RCA: collect metrics, traces, logs. Correlation engine identifies strongest indicator. Decision tree classifies bottleneck type. LLM generates explanation: "Throughput is limited by database CPU (97%) due to missing index on orders.user_id causing 5s sequential scan."

2. **Q:** How do you eliminate all synchronous bottlenecks in a system?
    **A:** Async everywhere: event-driven architecture, non-blocking I/O (Netty/WebFlux), reactive streams, CQRS with eventual consistency. Request hedging. No blocking in request threads.

3. **Q:** Design a system that guarantees no single point of bottleneck.
    **A:** Fully distributed: no single database, no single service, no single queue. Sharded, replicated, partitioned. Any node can fail or be saturated without impacting overall throughput.

4. **Q:** How do you design for zero-bottleneck auto-scaling?
    **A:** Scale based on the bottleneck resource: CPU, memory, queue depth, connection pool utilization. Predictive scaling. Canary deployments with bottleneck detection.

5. **Q:** Design a system architecture that degrades gracefully when every component is bottlenecked.
    **A:** Degradation modes: read-only mode (no writes), cache-only (no DB), last-known-good data, static content mode, queue mode (accept but delay), reject mode (503). Automatic transitions based on bottleneck severity.

6. **Q:** How do you handle cascading bottleneck propagation in a microservices chain?
    **A:** Circuit breakers, bulkheads, load shedding. Semantic degradation (disable non-critical features). Backpressure: propagate saturation signals upstream.

7. **Q:** Design a financial trading system with zero allocation (to avoid GC bottleneck).
    **A:** Object pooling for all mutable objects. Pre-allocate data structures. DirectByteBuffer for off-heap storage. Sun.misc.Unsafe for low-level memory operations. No GC in critical path.

8. **Q:** How do you design a bottleneck-free database for 1M writes/second?
    **A:** Sharded distributed database (Cassandra, CockroachDB). Time-series partitioning. In-memory buffer (Redis) for ingestion, batch write to DB. Append-only tables.

9. **Q:** Design a self-healing system that automatically reconfigures around bottlenecks.
    **A:** Monitor all resources. When bottleneck detected: auto-scale, add cache, throttle clients, reroute traffic. Post-bottleneck analysis: was it temporary (auto) or permanent (alert ops)?

10. **Q:** How do you design an organization and process that minimizes human-induced bottlenecks?
    **A:** DevOps culture: developers own performance. Performance testing in CI/CD. Automated canary analysis with performance gates. On-call runbooks for common bottlenecks. Architecture reviews focused on scalability.

## 13. Debugging & Troubleshooting

### Bottleneck Investigation Workflow

1. **Check overall health**: CPU, memory, disk, network, connection pools.
2. **Check application metrics**: Request rate, latency, error rate.
3. **Check dependent services**: Database, cache, external APIs.
4. **Isolate the layer**: Is it the network, application, or data store?
5. **Drill into specifics**: Slow query? Hot method? Lock contention?

### Common Patterns

**Pattern: High CPU, low throughput**
- Algorithm inefficiency (N+1, heavy loops, regex).
- Excessive serialization (JSON parsing large payloads).
- GC overhead.

**Pattern: Low CPU, low throughput**
- I/O bottleneck (database, disk, network).
- Thread pool starvation.
- External service latency.

**Pattern: Increasing latency, stable CPU**
- Queue building up (queue depth increasing).
- Connection pool saturation.
- Lock contention.

**Pattern: Memory growing, then sudden drop (GC)**
- Memory leak or allocation pressure.
- Full GC events.
- Check GC logs.

## 14. Comparison Section

### Types of Bottlenecks

| Type | Primary Metric | Diagnosis Tool | Typical Fix |
|------|---------------|---------------|-------------|
| CPU | % CPU, load avg | top, perf | Algorithm opt, add CPU |
| Memory | RSS, GC time | jmap, heap dump | Cache sizing, fix leaks |
| Disk I/O | iowait, await | iostat, sar | Faster storage, indexes |
| Network | latency, loss | netstat, traceroute | Bandwidth, compression |
| Database | query time, locks | EXPLAIN, slow log | Indexes, connection pool, replica |
| Thread Pool | active threads, queue | jstack, metrics | Pool sizing, async |
| External API | response time | tracing | Cache, circuit breaker |

### Synchronous vs Asynchronous Bottlenecks

| Aspect | Synchronous | Asynchronous |
|--------|-------------|--------------|
| Bottleneck | Thread/connection pool | Queue depth/consumer lag |
| Diagnosis | Thread dumps, pool metrics | Queue metrics, consumer offset |
| Resolution | Pool sizing, faster execution | Scale consumers, optimize processing |

## 15. Revision Notes

### Quick Recap
- **Bottleneck**: The slowest component limiting system performance.
- **USL**: Models throughput degradation with scaling.
- **CPU bottleneck**: Algorithm inefficiency, high concurrency.
- **I/O bottleneck**: Database, disk, or network limited.
- **Memory bottleneck**: Leaks, oversized caches, GC pressure.
- **Contention bottleneck**: Locks, shared resources, pools.
- **Diagnosis process**: Measure -> Identify -> Fix -> Verify.

### Bottleneck Detection Checklist
1. [ ] Check CPU utilization (not too high, not too low).
2. [ ] Check memory (RSS, GC frequency, heap usage).
3. [ ] Check disk I/O (utilization, IOPS, latency).
4. [ ] Check network (bandwidth, latency, errors).
5. [ ] Check thread pools (active, queued, rejected).
6. [ ] Check database (slow queries, connections, locks).
7. [ ] Check external dependencies.
8. [ ] Check queue depths (message brokers, buffers).
9. [ ] Review recent deployments/changes.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                    BOTTLENECKS CHEAT SHEET                         |
+-------------------------------------------------------------------+
| RESOURCE   | SYMPTOM          | DIAGNOSE                | FIX      |
+------------+------------------+-------------------------+----------+
| CPU        | >90% util,       | top -H, perf,           | Optimize |
|            | high load avg    | async-profiler          | algo     |
| Memory     | OOM, frequent GC | jmap -histo,            | Tune     |
|            | heap dump        | GC logs, heap dump      | cache    |
| Disk I/O   | iowait >30%,     | iostat -x 1,            | Indexes, |
|            | high await       | iotop, slow query log   | SSD      |
| Network    | packet loss,     | netstat -s,             | Compress,|
|            | high latency     | mtr, traceroute         | CDN      |
| Database   | slow queries,    | EXPLAIN ANALYZE,        | Indexes, |
|            | lock waits       | pg_stat_activity,       | replicas |
|            |                   | SHOW PROCESSLIST        |          |
| Thread Pool| queue building,   | jstack (WAITING/        | Size     |
|            | rejections       | BLOCKED threads)        | pool     |
| External   | high response    | tracing,                | Cache,   |
| API        | time, timeouts   | circuit breaker stats   | timeout  |
+------------+------------------+-------------------------+----------+
| CAPACITY PLANNING FORMULAS                                        |
+-------------------------------------------------------------------+
| Utilization = Arrival_Rate * Service_Time / Servers               |
| Queue_Depth = Utilization / (1 - Utilization) (M/M/1)            |
| Response_Time = Service_Time / (1 - Utilization)                  |
| Optimal_Threads = (1 - Blocking_Coeff) * Cores                   |
+-------------------------------------------------------------------+
| DIAGNOSIS WORKFLOW                                                |
+-------------------------------------------------------------------+
| 1. Define target: "P95 < 200ms, 1000 req/s"                      |
| 2. Measure current: "P95 = 500ms, 500 req/s"                     |
| 3. Check CPU: 95% -> CPU bottleneck                               |
| 4. Profile: async-profiler shows 60% time in serialize(Object)   |
| 5. Fix: Replace Java serialization with JSON                     |
| 6. Re-measure: P95 = 250ms, 800 req/s                            |
| 7. Find next bottleneck: DB I/O wait 40%                         |
+-------------------------------------------------------------------+
```
