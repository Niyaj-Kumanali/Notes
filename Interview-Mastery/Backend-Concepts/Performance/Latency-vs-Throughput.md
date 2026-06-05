# Latency vs Throughput

## 1. Executive Summary

Latency and throughput are two fundamental metrics for measuring system performance. Latency measures the time taken to complete a single operation, while throughput measures the number of operations completed per unit time. Understanding the relationship and trade-offs between these metrics is essential for designing scalable, responsive backend systems. Improving one often negatively impacts the other; optimal system design balances both based on business requirements.

## 2. Core Theory

### Definitions

**Latency:** The time between initiating an operation and receiving the result. Measured in milliseconds (ms) or microseconds (us).
- P50 (median): Half of requests are faster than this.
- P95: 95% of requests are faster than this (common SLA metric).
- P99: 99% of requests are faster than this (tail latency).
- P999: 99.9% (critical for high-reliability systems).

**Throughput:** The number of operations a system can process per unit time. Measured in requests per second (RPS), transactions per second (TPS), or operations per second (OPS).

### Little's Law

The fundamental relationship between latency, throughput, and concurrency:

```
L = lambda * W
```
Where:
- L = Average number of requests in the system (concurrency)
- lambda = Average throughput (arrival rate)
- W = Average latency (time in system)

**Implications:**
- To increase throughput without increasing latency, you must increase concurrency.
- If latency increases, either throughput decreases or concurrency increases.

### Latency Components

```
Total Latency = Processing Time + Queueing Time + Network Time + Contention Time
```

- **Processing Time**: Actual work time (CPU, I/O).
- **Queueing Time**: Time waiting for resources (CPU queue, thread pool, connection pool).
- **Network Time**: Data transfer over the network.
- **Contention Time**: Time waiting for locks or other synchronization.

## 3. Under-the-Hood Deep Dive

### Latency Distribution in Real Systems

```
Request Flow:
[Client] -> [Network] -> [Load Balancer] -> [App Server] -> [Database]
   |            |              |                 |              |
  5ms         10ms            5ms             30ms          20ms
  |                                                              |
  +------------------------ 70ms total -------------------------+
```

### Tail Latency (The Long Tail)

In distributed systems, latency follows a distribution where a small percentage of requests take significantly longer. Causes:
- Garbage collection pauses (Java)
- Network packet loss / retransmission
- Hot keys / uneven load distribution
- Resource contention (CPU throttling, disk I/O queues)
- Background tasks (compaction, replication, backups)

**The Jevons Paradox of Scale:** As you add more servers, tail latency gets worse because you're waiting for the slowest server in the group.

### Head-of-Line Blocking

A slow request blocks subsequent requests in a single-threaded or limited-concurrency system.

```
Thread: [Request A (slow)] [Request B (waiting)] [Request C (waiting)]
```

Mitigation: async I/O, non-blocking I/O, multiplexing.

## 4. Production Code Examples

### Measuring Latency in Spring Boot

```java
@Component
public class LatencyMetricsInterceptor implements HandlerInterceptor {

    private final MeterRegistry meterRegistry;

    public LatencyMetricsInterceptor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
            Object handler) {
        request.setAttribute("startTime", System.nanoTime());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
            Object handler, Exception ex) {
        long startTime = (Long) request.getAttribute("startTime");
        long duration = System.nanoTime() - startTime;

        String path = request.getRequestURI();
        int status = response.getStatus();

        // Record latency distribution
        Timer.Sample sample = Timer.start(meterRegistry);
        sample.stop(Timer.builder("http.server.requests")
            .tag("uri", path)
            .tag("status", String.valueOf(status))
            .tag("method", request.getMethod())
            .publishPercentiles(0.5, 0.95, 0.99, 0.999)
            .publishPercentileHistogram(true)
            .register(meterRegistry));
    }
}

// Configuration
@Configuration
public class LatencyMetricsConfig implements WebMvcConfigurer {

    @Autowired
    private MeterRegistry meterRegistry;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LatencyMetricsInterceptor(meterRegistry));
    }
}
```

### Async Processing for Throughput

```java
@Service
public class HighThroughputService {

    private final ExecutorService executor = Executors.newFixedThreadPool(50);

    @Autowired
    private OrderRepository orderRepository;

    public CompletableFuture<Order> processOrderAsync(CreateOrderRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            long start = System.nanoTime();
            try {
                // Process order
                Order order = new Order(request);
                order = orderRepository.save(order);

                long duration = (System.nanoTime() - start) / 1_000_000;
                if (duration > 100) {
                    log.warn("Slow order processing: {}ms", duration);
                }

                return order;
            } catch (Exception e) {
                log.error("Order processing failed", e);
                throw new CompletionException(e);
            }
        }, executor);
    }
}

// Spring async configuration
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

### Latency Budget Configuration

```java
// HTTP client with fine-grained timeouts
@Configuration
public class LatencyAwareHttpClientConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
            .setConnectTimeout(Duration.ofMillis(500))
            .setReadTimeout(Duration.ofMillis(2000))
            .additionalInterceptors((request, body, execution) -> {
                long start = System.nanoTime();
                try {
                    return execution.execute(request, body);
                } finally {
                    long elapsed = (System.nanoTime() - start) / 1_000_000;
                    if (elapsed > 500) {
                        log.warn("Slow HTTP call: {} {} took {}ms",
                            request.getMethod(), request.getURI(), elapsed);
                    }
                }
            })
            .build();
    }
}

// Database connection timeout
spring:
  datasource:
    hikari:
      connection-timeout: 2000      # Max time to get connection
      idle-timeout: 300000          # 5 minutes idle
      max-lifetime: 1200000         # 20 minutes max
      maximum-pool-size: 20
      minimum-idle: 5
      leak-detection-threshold: 60000 # 1 minute leak detection
```

### Throughput-Latency Trade-off Demonstration

```java
@Service
public class BenchmarkService {

    private final MeterRegistry meterRegistry;

    // Simulate latency vs throughput trade-off
    public void demonstrateTradeoff(int concurrencyLevel) {
        ExecutorService executor = Executors.newFixedThreadPool(concurrencyLevel);
        List<CompletableFuture<Long>> futures = new ArrayList<>();

        for (int i = 0; i < 1000; i++) {
            futures.add(CompletableFuture.supplyAsync(() -> {
                long start = System.nanoTime();
                // Simulate work
                sleep(50);
                return System.nanoTime() - start;
            }, executor));
        }

        List<Long> latencies = futures.stream()
            .map(CompletableFuture::join)
            .sorted()
            .collect(Collectors.toList());

        long p50 = latencies.get(500);
        long p99 = latencies.get(990);
        long totalTime = latencies.stream().mapToLong(Long::longValue).sum();
        double throughput = 1000.0 / (totalTime / 1_000_000_000.0);

        log.info("Concurrency {}: P50={}ms, P99={}ms, Throughput={} req/s",
            concurrencyLevel, p50 / 1_000_000, p99 / 1_000_000, throughput);
    }
}
```

### Circuit Breaker with Latency-Based Threshold

```java
@Configuration
public class LatencyAwareCircuitBreakerConfig {

    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> latencyAwareConfig() {
        return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .circuitBreakerConfig(CircuitBreakerConfig.custom()
                .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.TIME_BASED)
                .slidingWindowSize(60) // 60 seconds
                .minimumNumberOfCalls(10)
                .failureRateThreshold(50)
                // Open circuit if >50% of calls slower than threshold
                .slowCallRateThreshold(50)
                .slowCallDurationThreshold(Duration.ofMillis(2000))
                .waitDurationInOpenState(Duration.ofSeconds(30))
                .build())
            .build());
    }
}
```

## 5. Real-World Scenarios

### E-Commerce Checkout API

**Latency Budget:**
```
Client -> API Gateway (10ms) -> Order Service (50ms) -> Payment Gateway (500ms)
                                       -> Inventory Check (100ms)
                                       -> Notification (200ms async)
Total: ~660ms synchronous, ~200ms async
P99 Target: <2s
```

**Throughput Requirements:**
- Black Friday: 10,000 checkouts/minute -> 167 TPS
- Normal day: 500 checkouts/minute -> 8.3 TPS

### Video Streaming Service

**Latency:**
- First frame: <2 seconds (buffering)
- Seek: <500ms
- Live stream delay: <5 seconds

**Throughput:**
- Concurrent viewers: 1 million
- Bandwidth: 5 Mbps per stream -> 5000 Gbps total
- Transcoding: 100 frames/second per stream

### Database Performance

**Latency:**
- Cache (Redis): <1ms
- SQL query (indexed): <10ms
- SQL query (full scan): >100ms

**Throughput:**
- Redis: 100,000 ops/s per node
- PostgreSQL: 10,000 TPS (tuned)
- Cassandra: 100,000 writes/s per node

## 6. Performance

### Optimizing Latency

1. **Reduce processing time**: Optimize algorithms, use caching, compile queries.
2. **Reduce queueing time**: Increase thread pool, async processing, faster hardware.
3. **Reduce network time**: CDN, edge computing, connection pooling, keep-alive.
4. **Reduce contention**: Lock-free data structures, sharding, CQRS.
5. **Eliminate head-of-line blocking**: Async I/O, HTTP/2 multiplexing.

### Optimizing Throughput

1. **Increase concurrency**: More threads, async processing, horizontal scaling.
2. **Reduce per-request overhead**: Batch processing, pipelining, connection reuse.
3. **Parallelize**: Split work across multiple workers, shard databases.
4. **Eliminate bottlenecks**: Profile to find the limiting resource (CPU, I/O, network, memory).

### The Performance Equation

```
Throughput = 1 / (Latency per request * Concurrency overhead)

Example:
- Single-threaded: 1 / (50ms * 1.0) = 20 req/s
- 10 threads: 1 / (50ms * 1.1) * 10 = 182 req/s (90% efficiency)
- 100 threads: 1 / (50ms * 2.0) * 100 = 1000 req/s (50% efficiency due to contention)
```

### Amdahl's Law for Latency

```
Speedup = 1 / ((1 - P) + P/N)
```
Where P = parallelizable portion, N = number of processors.

## 7. Security

### Latency vs Security Trade-offs

| Security Measure | Latency Impact | Throughput Impact |
|-----------------|----------------|-------------------|
| TLS handshake | +10-50ms (first request) | -10% (connection reuse) |
| JWT validation | +1-5ms | - |
| SQL encryption | +5-20ms per query | -20-50% |
| Input sanitization | +0.1-1ms | - |
| Rate limiting | +0.1ms | - |
| WAF | +1-10ms | - |

### Mitigation
- TLS session resumption reduces handshake latency.
- Cache JWT validation results.
- Use hardware acceleration (AES-NI) for encryption.
- Offload security processing to API gateway / edge.

## 8. Common Mistakes

### Mistake 1: Optimizing for Average Latency Only
Ignoring tail latency (P99, P999) leads to unpredictable user experience.

### Mistake 2: Confusing High Throughput with Low Latency
High throughput can coexist with high latency (batch processing).

### Mistake 3: Infinite Queueing
Unbounded queues increase latency as they grow.

### Mistake 4: Thread Pool Over-Subscription
Too many threads increase context switching overhead, reducing throughput and increasing latency.

### Mistake 5: Ignoring the Coordination Penalty
Adding more servers doesn't linearly increase throughput due to coordination overhead (Amdahl's Law).

## 9. Senior Engineer Perspective

### Latency SLO/SLA Design

```
Critical API (e.g., Payment):
  P50 < 100ms
  P95 < 500ms
  P99 < 1s
  P999 < 5s

Non-critical API (e.g., History):
  P50 < 500ms
  P95 < 2s
  P99 < 5s

Internal Service (e.g., Cache):
  P50 < 1ms
  P95 < 5ms
  P99 < 10ms
```

### The Multi-Level Latency Strategy

```
Level 1 (In-memory):       0.1ms - 1ms   (Caffeine cache)
Level 2 (Distributed):     1ms - 5ms     (Redis)
Level 3 (Database):        5ms - 50ms    (PostgreSQL indexed query)
Level 4 (Slow query):      50ms - 500ms  (Complex aggregation)
Level 5 (External):        500ms - 5s    (Third-party API)
```

### Capacity Planning Formula

```java
public class CapacityPlanner {

    public static int calculateNodes(
            double expectedThroughput,     // requests/second
            double avgLatencySeconds,       // average response time
            double targetCpuUtilization,    // 0.0 - 1.0
            double coresPerNode) {

        // Using Little's Law and utilization formula
        double requestsPerCorePerSec = 1.0 / avgLatencySeconds;
        double maxThroughputPerNode = requestsPerCorePerSec * coresPerNode;
        double targetThroughputPerNode = maxThroughputPerNode * targetCpuUtilization;
        int nodes = (int) Math.ceil(expectedThroughput / targetThroughputPerNode);

        return Math.max(nodes, 2); // Minimum 2 for redundancy
    }
}
```

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What is latency?
   **A:** The time taken for a single operation to complete, measured from initiation to response.

2. **Q:** What is throughput?
   **A:** The number of operations a system can process per unit time (requests/second, transactions/second).

3. **Q:** What is the difference between P50, P95, and P99 latency?
   **A:** P50 is median latency (50% faster). P95 is 95th percentile (95% faster). P99 is 99th percentile. P99 shows tail latency.

4. **Q:** What is Little's Law?
   **A:** L = lambda * W, where L is average concurrency, lambda is throughput, W is average latency.

5. **Q:** How do you measure latency in Spring Boot?
   **A:** Using Micrometer Timer with @Timed annotation or programmatically with Timer.Sample.

6. **Q:** What is the difference between network latency and processing latency?
   **A:** Network latency is time spent transferring data over the network. Processing latency is time spent doing work (CPU, I/O, database).

7. **Q:** What is a timeout?
   **A:** A limit on how long to wait for an operation before declaring it failed. Prevents unbounded latency.

8. **Q:** What is the relationship between latency and user experience?
   **A:** Higher latency degrades user experience. <100ms feels instant, <1s feels natural, >1s feels slow, >10s users abandon.

9. **Q:** What is bandwidth in relation to throughput?
   **A:** Bandwidth is the maximum data transfer rate of a network. Throughput may be lower due to protocol overhead, congestion.

10. **Q:** What is a latency SLA?
    **A:** A service level agreement specifying latency targets (e.g., P95 < 200ms, P99 < 500ms).

### Medium

11. **Q:** Explain the trade-off between latency and throughput.
    **A:** Improving throughput often increases latency (batching, more concurrency). Reducing latency may reduce throughput (dedicated resources, less batching).

12. **Q:** What is tail latency and why does it matter?
    **A:** Tail latency (P99/P999) measures the slowest requests. In distributed systems, waiting for the slowest server determines overall response time.

13. **Q:** How does queueing affect latency?
    **A:** As queue depth increases, waiting time increases. Little's Law: queue depth = arrival_rate * service_time.

14. **Q:** What is the throughput limit of a single CPU core?
    **A:** Roughly 1/latency of the critical path. If latency is 10ms, maximum throughput is ~100 ops/s per core (for that operation).

15. **Q:** How do you reduce latency in a database query?
    **A:** Indexes, query optimization, connection pooling, read replicas, caching, denormalization.

16. **Q:** What is head-of-line blocking?
    **A:** A slow request at the front of a queue blocks all subsequent requests from being processed.

17. **Q:** How does HTTP/2 improve latency over HTTP/1.1?
    **A:** Multiplexing (multiple requests on one connection), header compression, server push. Eliminates head-of-line blocking at HTTP level.

18. **Q:** What is request batching and how does it affect latency/throughput?
    **A:** Batching groups multiple operations into one. Increases throughput (less overhead per op) but increases latency (wait for batch).

19. **Q:** How do you set timeouts correctly?
    **A:** Based on latency SLOs. Connect timeout < 500ms. Read timeout < P99 + buffer. Use cascading timeouts: client timeout > server timeout.

20. **Q:** What is coordinated omission?
    **A:** A measurement error where slow requests are excluded from latency statistics because they fall outside the measurement window. Results in overly optimistic latency numbers.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** How do you accurately measure P99 latency in a distributed system?
    **A:** Use HDR Histogram or T-Digest for efficient percentile calculation. Consistent sampling across services. Trace-level timing with OpenTelemetry. Avoid coordinated omission by measuring all requests including those that time out.

2. **Q:** Design a latency budget allocation for a 2-second end-to-end SLA across 5 microservices.
    **A:** API Gateway: 200ms, Service 1: 300ms, Service 2: 500ms, Service 3: 400ms, Database: 200ms = 1600ms (buffer 400ms for network, queueing). Each service monitors actual vs budget and breaks circuit if exceeded.

3. **Q:** How do you handle the thundering herd problem and its effect on latency?
    **A:** Cache stampede: multiple requests for expired cache key simultaneously hit DB. Solutions: request coalescing, probabilistic early expiration, distributed locks, stale-while-revalidate.

4. **Q:** Explain the relationship between GC pauses and tail latency in Java applications.
    **A:** GC pauses (especially Full GC) cause all threads to stop (STW), drastically increasing P99/P999 latency. Mitigations: GC tuning (G1, ZGC, Shenandoah), allocate less, pool objects.

5. **Q:** How do you design a system that maintains low latency under 10x traffic spikes?
    **A:** Auto-scaling based on latency (not CPU). Over-provision by 2x for headroom. Load shedding (reject low-priority requests). Degrade gracefully (disable expensive features). Cache aggressively.

6. **Q:** What is the relationship between service mesh sidecars and latency?
    **A:** Each sidecar adds 1-5ms of latency per hop (mTLS, routing, telemetry). For a chain of 5 services, that's 5-25ms additional. Mitigation: sidecar per node vs per pod, eBPF-based mesh (Cilium).

7. **Q:** How do you implement latency-based routing in a multi-region deployment?
    **A:** Measure latency from each region to each client. Route requests to lowest latency region. Use anycast DNS, latency-based DNS routing (Route53), or client-side latency measurement.

8. **Q:** Design a system that achieves both high throughput (1M req/s) and low latency (<10ms P50).
    **A:** Must be in-memory, no database in critical path. Event-driven architecture with non-blocking I/O (Netty, Vert.x). Zero-copy serialization. CPU pinning. Kernel bypass (DPDK). Tiered: in-memory cache (sub-ms) -> DB (async persistence).

9. **Q:** How does connection pooling affect latency and throughput?
    **A:** Pooling reduces connection setup latency (TCP handshake, TLS). But pool exhaustion increases latency (queueing). Optimal pool size: calculated from latency and throughput (Little's Law).

10. **Q:** What is the latency impact of distributed transactions (2PC)?
    **A:** 2PC adds prepare + commit round trips. Each participant holds locks during transaction, increasing contention and blocking. Typical overhead: 2-5x latency increase vs single-node transactions.

### System Design

11. **Q:** Design a real-time chat system with <100ms latency for 10M users.
    **A:** WebSocket connections to chat servers. In-memory message routing. Kafka for persistence (async). Redis pub/sub for cross-server fanout. Geo-distributed: users connect to nearest PoP.

12. **Q:** Design a stock trading platform with microsecond latency.
    **A:** Co-location: servers at exchange data center. Kernel bypass (DPACK, Solarflare). In-memory order book. FPGA for matching engine. No GC languages (C++, Rust, Java with Azul Zing).

13. **Q:** Design a CDN that serves content with <10ms edge latency globally.
    **A:** 500+ PoPs globally. Anycast routing. In-memory cache (RAM + SSD). Pre-fetch popular content. TCP optimizations (fast open, BBR congestion control). HTTP/3 (QUIC) for faster connection setup.

14. **Q:** Design a caching system that balances latency and throughput for a social media feed.
    **A:** Two-tier: L1 (in-memory, sub-ms, 10K entries) + L2 (Redis, 1-3ms, 10M entries). Cache-aside for reads. Write-behind for writes. Predictive pre-fetching for hot users.

15. **Q:** Design a database replication system with <10ms replication lag.
    **A:** Synchronous replication (semi-sync). Parallel apply on replicas. Optimized network (dedicated links, no congestion). Monitor lag with heartbeat. Automatic failover if lag exceeds threshold.

16. **Q:** Design a latency monitoring system for 1000+ microservices.
    **A:** OpenTelemetry for distributed tracing. Prometheus for metrics aggregation. HDR Histogram per endpoint per service. Centralized dashboard showing P50/P95/P99 per service, latency heatmap, service dependency graph.

17. **Q:** Design an API gateway that maintains low latency under high throughput.
    **A:** Netty-based (non-blocking). Connection pooling to upstream services. Request collapsing for cacheable requests. Async response aggregation. Circuit breakers per upstream. Rate limiting in memory (no Redis call per request).

18. **Q:** Design a system for processing credit card transactions with <500ms end-to-end latency.
    **A:** In-memory fraud checks (ML model in Redis). Async logging (write-behind). Synchronous gateway call. Circuit breaker for 3rd-party gateways. Result cached for idempotency checks.

19. **Q:** Design a video conferencing system with <150ms real-time delay.
    **A:** WebRTC for peer-to-peer media. Selective Forwarding Unit (SFU) for group calls. UDP (not TCP) for media. Simulcast for adaptive quality. Echo cancellation, jitter buffer. Geo-distributed SFUs.

20. **Q:** Design a latency budget management system for microservices.
    **A:** Each service registers its expected P50/P95 latency. Service mesh measures actual per-hop latency. Dashboard: budget actual vs allocated. Alerts when exceeded. Automatic circuit breaking when budget exhausted.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a latency-aware load balancing algorithm.
    **A:** Weighted round-robin based on each node's current P50 latency (measured via moving window). Send more traffic to faster nodes. Exponential weighted moving average for smooth adjustment. Circuit breakers for nodes exceeding latency threshold.

2. **Q:** How do you design a system that provides latency SLOs in the presence of noisy neighbors?
    **A:** Resource isolation (cgroups, Kubernetes QoS classes). CPU pinning for critical services. Latency SLOs enforced via admission control. Rate limit noisy tenants. Separate critical and batch workloads. Use preemptible instances for batch.

3. **Q:** Design a global multi-region active-active system with consistent sub-10ms latency for reads.
    **A:** CRDT-based data replication. Local reads from in-region data store. Writes propagate asynchronously. Conflict resolution via LWW or application semantics. Region-local caches with TTL. Global DNS routing to nearest region.

4. **Q:** How do you implement request cancellation to avoid wasting resources on timed-out requests?
    **A:** Propagate cancellation context across service boundaries. gRPC has built-in cancellation propagation. For HTTP: use tracing context with deadline. When client disconnects, cancel downstream calls. Thread pool interruption.

5. **Q:** Design a system that achieves both high throughput and low latency through workload isolation.
    **A:** Separate thread pools for fast and slow operations. Fast path: optimized, in-memory, no DB queries. Slow path: queued, async, can take longer. Priority queueing: high-priority requests processed first. Admission control.

6. **Q:** How do you build a latency prediction model for auto-scaling?
    **A:** Features: request rate, queue depth, CPU/memory/IO utilization, GC pause duration, external service latency. Model: gradient boosting or LSTM. Predict P99 latency given current conditions. Scale out if predicted latency > threshold. Scale in if predicted latency < threshold for N minutes.

7. **Q:** Design a system to detect and mitigate latency anomalies automatically.
    **A:** Baseline modeling: moving window of P50/P95/P99. Anomaly detection: if current latency > baseline * threshold (e.g., 2x). Automated mitigation: restart slow instance, remove from load balancer, enable circuit breaker, route around region. Root cause analysis: correlate with deployments, traffic patterns.

8. **Q:** How do you design a system where latency guarantees improve as you add more resources?
    **A:** Partitioned data: more partitions = less contention per partition. Queue depth: more workers = shorter queues. Caching: more memory = higher cache hit ratio. The system must be horizontally scalable (no central bottleneck).

9. **Q:** Design a queuing system that minimizes tail latency for urgent requests.
    **A:** Multi-level priority queues. Urgent: processed immediately (dedicated resources). Normal: standard queue. Batch: processed in background. Admission control: reject low-priority if system is overloaded. Expedite aging: boost priority of long-waiting requests.

10. **Q:** How do you model the latency-throughput curve for capacity planning?
    **A:** Load test at increasing concurrency levels. Measure throughput and latency at each level. Fit curve: latency = a + b * throughput + c * throughput^2. Identify inflection point where latency increases non-linearly (knee). Set operating point at 70-80% of maximum throughput before knee.

## 13. Debugging & Troubleshooting

### Common Latency Issues

**Issue: Sporadic latency spikes**
- Check GC logs (Full GC events).
- Check for background jobs (compaction, backups, batch processing).
- Check thread dumps for deadlocks or contention.
- Check network for packet loss (TCP retransmissions).

**Issue: Increasing latency over time**
- Memory leak causing more GC.
- Connection pool exhaustion.
- Database query plan changes.
- Disk fragmentation or compaction.

**Issue: High latency for specific users/requests**
- Hot keys (single partition/cache key receiving disproportionate traffic).
- Data skew in database sharding.
- Slow query for specific parameter values.

### Debugging Tools
```bash
# Measure latency to a service
curl -w "Connect: %{time_connect}s, TTFB: %{time_starttransfer}s, Total: %{time_total}s\n" \
  -o /dev/null -s https://api.example.com/endpoint

# Check GC pauses
jstat -gcutil <pid> 1000

# Network latency
ping -c 10 <host>
mtr <host>

# Thread dump for contention
jstack <pid> | grep -A 20 "BLOCKED"

# Database query latency
EXPLAIN ANALYZE SELECT ...;
```

### Spring Boot Latency Debugging
```yaml
logging:
  level:
    org.springframework.jdbc: DEBUG     # Query execution times
    org.springframework.web: DEBUG      # Request handling times
    org.hibernate.SQL: DEBUG            # Hibernate SQL
    org.hibernate.stat: DEBUG           # Hibernate statistics

spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

## 14. Comparison Section

### Latency vs Throughput Trade-offs

| Strategy | Effect on Latency | Effect on Throughput | Use Case |
|----------|------------------|---------------------|----------|
| Batching | Increases | Increases | Analytics, bulk writes |
| Caching | Decreases | Increases (cache hit) | Read-heavy workloads |
| Compression | Increases (CPU) | Increases (network) | Large payloads |
| Async processing | Decreases (perceived) | Increases | Non-critical operations |
| Connection pooling | Decreases | Increases | Any network call |
| Thread pool increase | May increase (contention) | Increases (more parallelism) | CPU-bound |
| Queueing | Increases | Protects from overload | Traffic spikes |

### Synchronous vs Asynchronous

| Aspect | Synchronous | Asynchronous |
|--------|-------------|--------------|
| Latency (user-perceived) | Higher (wait for result) | Lower (immediate ack) |
| Throughput | Lower (blocking) | Higher |
| Complexity | Lower | Higher |
| Error handling | Easier | Harder (callbacks, futures) |

### P50 vs P95 vs P99 Latency

| Metric | What it Means | Optimization Focus |
|--------|---------------|-------------------|
| P50 | Typical user experience | Average performance |
| P95 | Poor experience (1 in 20) | Common issues |
| P99 | Very poor experience (1 in 100) | Edge cases, GC, contention |
| P999 | Almost unacceptable (1 in 1000) | Extreme outliers, failures |

## 15. Revision Notes

### Quick Recap
- **Latency**: Time to complete a single request.
- **Throughput**: Requests completed per unit time.
- **Little's Law**: L = lambda * W (concurrency = throughput * latency).
- **P50/P95/P99/P999**: Latency percentiles for understanding distribution.
- **Tail latency**: The high-percentile latencies that determine worst-case user experience.
- **Queueing**: Adding load increases latency (non-linear at saturation).
- **Trade-off**: Increasing throughput typically increases latency.
- **Measurement**: Use HDR Histogram, Micrometer Timer, distributed tracing.

### Key Formulas
```
Throughput = 1 / (Latency per request)
Throughput = Concurrency / Latency (Little's Law)
Effective Latency = P50 + (P99 - P50) * cost_of_tail
Utilization = Arrival_Rate * Service_Time / Servers
```

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|               LATENCY vs THROUGHPUT CHEAT SHEET                    |
+-------------------------------------------------------------------+
| METRIC      | DEFINITION                         | EXAMPLE         |
+-------------+------------------------------------+-----------------+
| Latency     | Time per operation                 | 50ms (P50)      |
| Throughput  | Operations per time                | 1000 req/s      |
| P50         | Median latency                     | 50ms            |
| P95         | 95th percentile latency            | 200ms           |
| P99         | 99th percentile (tail)             | 500ms           |
| Concurrency | Requests in-flight                 | Little's Law     |
| Bandwidth   | Max data transfer rate             | 1 Gbps          |
+-------------+------------------------------------+-----------------+
| LATENCY BUDGET DISTRIBUTION                                       |
+-------------------------------------------------------------------+
| Component              | Budget  | Actual   | Status              |
+------------------------+---------+----------+---------------------+
| Client -> API Gateway  | 50ms    | 30ms     | OK                  |
| API Gateway            | 100ms   | 80ms     | OK                  |
| Order Service          | 200ms   | 150ms    | OK                  |
| Payment Gateway        | 500ms   | 600ms    | EXCEEDED (CB opens) |
| Database               | 100ms   | 50ms     | OK                  |
| Total                  | 950ms   | 910ms    | OK                  |
+------------------------+---------+----------+---------------------+
| OPTIMIZATION STRATEGIES                                           |
+-------------------------------------------------------------------+
| Goal           | Strategy                                           |
+----------------+----------------------------------------------------+
| Lower latency  | Caching, faster hardware, async I/O,              |
|                | connection pooling, CDN, edge compute              |
| Higher         | Horizontal scaling, batching, async processing,    |
| throughput     | connection reuse, pipelining, sharding             |
| Lower tail     | GC tuning, request hedging, circuit breakers,      |
| latency        | latency-aware load balancing, priority queueing    |
+----------------+----------------------------------------------------+
| TOOLS                                                              |
+-------------------------------------------------------------------+
| Micrometer @Timed   | Method-level latency metrics                 |
| HDR Histogram       | High-resolution percentile tracking          |
| OpenTelemetry       | Distributed tracing with span timing         |
| Prometheus          | Latency/throughput metric aggregation        |
| Grafana             | Dashboards for P50/P95/P99 latency           |
| JMeter / Gatling    | Load testing for latency/throughput curves   |
+-------------------------------------------------------------------+
```
