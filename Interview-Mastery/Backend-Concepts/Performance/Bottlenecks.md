# Bottlenecks

---

## Overview

- **Definition:** A bottleneck is a point in a system where limited capacity constrains overall performance — the slowest component determines maximum throughput.
- **Why It Exists:** Every system has a weakest link. Identifying and resolving bottlenecks is the core of performance engineering. Fixing one bottleneck reveals the next, following the bottleneck chain.
- **Key Concepts:** **Universal Scalability Law** (models throughput degradation from contention + coherency overhead), **Compute-bound** (CPU limited), **Memory-bound** (RAM limited), **I/O-bound** (disk/network), **Contention-bound** (locks, shared resources), **Queueing-bound** (request queue fills up).
- **Bottleneck Economics** — The cost of fixing a bottleneck increases exponentially as you move up the stack: optimizing a database query costs hours, adding a cache costs days, redesigning the architecture costs weeks. Always fix the cheapest bottleneck first and re-measure before moving to the next.
- **Amdahl's Law vs Universal Scalability Law** — Amdahl's Law predicts speedup as `1/((1-P)+P/N)` where P is the parallelizable portion. The USL adds coherency overhead: `C(N) = N / (1 + σ(N-1) + κN(N-1))`. The USL predicts that beyond a certain point, adding more nodes actually decreases throughput due to coordination overhead.

---

## Core Concepts

- **Types of Bottlenecks:** **CPU** — high utilization, context switching, inefficient algorithms. **Memory** — frequent GC, OOM, oversized caches, leaks. **Disk I/O** — missing indexes, full table scans, IOPS limit. **Network** — packet loss, small TCP buffers, chatty inter-service calls. **Database** — slow queries, lock contention, connection pool exhaustion. **Thread Pool** — queue building, rejections.
- **Bottleneck Chain:** System throughput is determined by its slowest component. Fixing one reveals the next: DB queries (100ms) → CPU (95% util) → Thread pool contention → Network bandwidth.
- **Bottleneck Detection Flow:** Define target → Measure current → Identify saturated resource → Hypothesize cause → Fix → Measure impact → Repeat.
- **Profiling Tools and Techniques** — Java: async-profiler for CPU and allocation profiling, JFR (Java Flight Recorder) for low-overhead runtime analysis, JMH for microbenchmarks. Database: `EXPLAIN ANALYZE`, `pg_stat_statements`, slow query logs. Network: `tcpdump`, Wireshark, mtr for path analysis.
- **Resource Saturation Indicators** — CPU: utilization > 70%, context switching rate > 10K/sec. Memory: GC frequency increasing, swap usage > 0. Disk: iowait > 10%, queue depth > 2x spindles. Network: dropped packets, retransmits > 1%. Database: connection pool exhaustion, lock wait time growing.

```java
// Thread pool monitoring with Micrometer
Gauge.builder("threadpool.active.count", executor, ThreadPoolExecutor::getActiveCount)
    .tag("pool", "biz-pool").register(registry);
Gauge.builder("threadpool.queue.size", executor, e -> e.getQueue().size())
    .tag("pool", "biz-pool").register(registry);
```

---

## Common Mistakes

- **Premature Optimization** — Optimizing before measuring. "Make it work, make it right, make it fast — in that order." This *looks correct* because the optimization target seems obvious ("this loop must be slow") — the wasted effort is invisible until profiling reveals the bottleneck was elsewhere.
- **Ignoring the Obvious** — Checking JVM GC settings before looking for missing indexes. This *looks correct* because JVM tuning sounds like "real performance work" — the missing index that causes the actual 100x slowdown is found only after a day of GC tuning.
- **Local-Only Testing** — Bottlenecks often only appear under production load (concurrency, data volume). This *looks correct* because the API responds in 10ms on the developer's machine with one concurrent user — the 100x slowdown under 500 concurrent users is invisible in local testing.
- **Single Metric Focus** — High CPU doesn't mean CPU is the bottleneck; it could be I/O waiting. This *looks correct* because the CPU graph is high and red — the developer adds CPU capacity without checking that the real bottleneck is I/O wait masking as CPU usage.
- **Assuming Linearity** — Doubling servers does not double throughput (Amdahl's Law, USL). This *looks correct* because each server independently processes its share — the coordination overhead of distributed locking, cache coherency, and shared database connections only appears after the servers are deployed.
- **Optimizing the Wrong Tier** — Adding application servers when the database is the bottleneck. Always measure end-to-end before scaling any specific tier. This *looks correct* because the application servers are easy to scale — the database bottleneck becomes worse as more app servers compete for the same limited connection pool.
- **Not Measuring Baseline Performance** — Without baseline metrics, you can't tell if a change improved or degraded performance. Establish P50/P95/P99 latency baselines during normal load before making changes. This *looks correct* because the fix "feels faster" in ad-hoc testing — without a baseline, a change that actually degrades P99 by 200ms goes unnoticed until production.

---

## Key Design Considerations

- **Systematic Analysis Process:** Measure all resources (CPU, memory, disk, network, thread pools, DB). Identify which is saturated. Make one change at a time. Re-measure to verify improvement and find the next bottleneck.
- **Bottleneck Budget:** Set utilization targets per resource: CPU < 70%, memory < 80%, disk < 60% IOPS, connection pools < 70%. When exceeded, investigate.
- **Resolution Priority:** Low-hanging fruit (missing indexes, pool sizing) → Architecture changes (caching, async) → Code optimization (algorithms) → Hardware scaling → Architecture redesign (CQRS, event sourcing).
- **Predictive Detection:** Track growth rates of metrics (DB size, request rate). Model when each resource reaches capacity. Schedule upgrades before saturation.
- **Common Patterns:** High CPU + low throughput = algorithm inefficiency. Low CPU + low throughput = I/O/DB bottleneck. Increasing latency + stable CPU = queue buildup.
- **Load Testing for Bottleneck Discovery** — Gradually increase load while monitoring all resources. The first resource to saturate is the bottleneck. Use tools like Gatling, JMeter, or k6. Test with production-like data volumes and concurrency patterns. Monitor P50/P95/P99 latencies alongside resource utilization.
- **Distributed System Bottlenecks** — In microservices, common bottlenecks include: serialized service chains (cascading latency), shared databases (contention under parallel load), message broker partitions (insufficient partitions for consumer parallelism), and API Gateway (single entry point for all traffic).

---

## Real-World Scenarios

### Scenario 1: N+1 Query Problem in REST API
**Context:** A REST endpoint `GET /api/orders` returns 100 orders, each with the customer name. The code fetches orders with `SELECT * FROM orders`, then loops through each order calling `SELECT * FROM customers WHERE id = ?`. That's 1 + 100 = 101 queries. The endpoint takes 3 seconds.

**Resolution:** Fix with `JOIN FETCH` or `@EntityGraph`. A single query with `SELECT o FROM Order o JOIN FETCH o.customer` reduces to 1 query, taking 30ms. The bottleneck shifts from the database to the serialization layer.

```java
// Before — N+1 queries
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // Fetching orders doesn't fetch customers — each customer is lazy-loaded individually
}

// After — Single query
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o FROM Order o JOIN FETCH o.customer")
    List<Order> findAllWithCustomer();

    // Or use EntityGraph
    @EntityGraph(attributePaths = {"customer", "items"})
    @Query("SELECT o FROM Order o")
    List<Order> findAllEagerly();
}
```

### Scenario 2: Thread Pool Exhaustion from Slow Downstream Service
**Context:** A payment service calls an external gateway. The gateway slows down (100ms → 10s responses). All Tomcat threads (200) become blocked waiting for payment responses. New requests queue up, and the entire application becomes unresponsive — even pages that don't use payments.

**Resolution:** Add an async timeout (2s), circuit breaker, and separate thread pool for payment calls. The main request thread returns immediately; the payment call runs in a dedicated thread pool. When the circuit breaker opens, the fallback returns "payment pending." Non-payment features continue working.

### Scenario 3: Memory Leak from Unbounded Cache
**Context:** A team adds an in-memory cache to speed up product lookups. They forget to set a max size or TTL. Over 24 hours, the cache grows to 8GB, causing constant GC pauses (500ms+ every 30 seconds). Response times degrade from 50ms to 5 seconds.

**Resolution:** Switch to Caffeine cache with `maximumSize(10000)` and `expireAfterWrite(1, TimeUnit.HOURS)`. Enable `recordStats()` to monitor hit ratio. The cache stays at ~500MB. GC pauses drop to <50ms. The bottleneck shifts from memory/GC to the database (which is the right place).

---

## Scenario-Based Questions

1. **Q: Your API endpoint `GET /api/users` returns 200 users. Each user has a profile picture. The endpoint takes 8 seconds. Profiling shows 7.5 seconds are spent in the database. How do you diagnose and fix?**
   - A: Enable slow query logging and capture the actual queries. Most likely: N+1 problem where the main query fetches users, then a loop fetches each user's profile picture separately. Fix: use `JOIN FETCH` or batch fetching (`@BatchSize(size = 50)`) to load all profile pictures in one query. After fix: 200 queries → 2 queries, 8 seconds → 200ms. Profile again to find the next bottleneck.

2. **Q: Your application suddenly becomes unresponsive under load. CPU is at 20%, memory at 40%, but all threads are blocked. The health check endpoint also times out. What's the bottleneck?**
    - A: Thread pool exhaustion. CPU is low because threads aren't doing work — they're blocked waiting for something (database, external API, locks). Take a thread dump (`jstack`). Look for RUNNABLE threads that are actually BLOCKED or WAITING. Common causes: database connection pool exhaustion (all threads waiting for a connection), deadlock, or a downstream service that's not responding. Fix: add timeouts, increase pool sizes, or use async processing.
    > **Interview follow-up:** You increase the thread pool from 50 to 200 threads to fix the exhaustion, but now the database connection pool (20 connections) becomes the bottleneck — all 200 threads contend for 20 connections. How do you tune thread pool and connection pool sizes relative to each other?

3. **Q: Your database CPU is at 95%, queries are slow, and adding more application servers doesn't improve throughput. The database is the bottleneck. What are your options?**
   - A: Options in order of impact: (1) Optimize slow queries (add indexes, rewrite queries, use materialized views). (2) Add read replicas for read-heavy workloads. (3) Add caching (Redis) for frequently accessed data. (4) Scale up the database (more CPU/RAM). (5) Shard the database by tenant or entity. (6) Move to CQRS with separate read and write databases. Don't add application servers until the database bottleneck is resolved.

4. **Q: Your application processes messages from a queue. Each message takes 500ms to process. You add 10 more consumers, but throughput only increases by 20%. Why?**
    - A: The bottleneck is downstream of the consumer — likely a shared resource like the database or an external API. Adding consumers only increases contention on that shared resource. Profile the processing pipeline to find the actual bottleneck. If it's the database, add read replicas or optimize queries. If it's an external API with rate limits, you can't scale past its limit.
    > **Interview follow-up:** You identify the database as the downstream bottleneck and add read replicas — but the consumer workload is write-heavy (processing payments, updating inventory). Read replicas don't help. What alternatives do you have for scaling write-heavy consumers that compete for the same primary database?

5. **Q: Your server has 16 CPU cores. You configure a thread pool with 100 threads for I/O-bound work. The application is slower than with 16 threads. What's happening?**
   - A: Too many threads cause context switching overhead. With 100 threads on 16 cores, the OS spends significant time switching between threads instead of doing work. The "sweet spot" formula: `threads = cores * (1 + wait_time / service_time)`. If wait time = 100ms and service time = 10ms: `16 * (1 + 100/10) = 176`. But 100 threads on a 16-core machine for I/O might be fine; the real issue could be contention in shared data structures or database connections. Measure context switching rate with `vmstat`.

6. **Q: Your website loads in 3 seconds. The backend API responds in 200ms. The bottleneck is the frontend. How do you identify and fix?**
   - A: Use browser developer tools (Network tab, Performance tab, Lighthouse). Common frontend bottlenecks: (1) Large images not compressed/resized → use CDN with WebP. (2) Render-blocking JavaScript → defer or async scripts. (3) Too many HTTP requests → bundle and minify. (4) No caching → implement service workers and HTTP caching. (5) Slow JavaScript execution → optimize DOM manipulation, use virtual scrolling for long lists.

7. **Q: You add more application servers expecting linear throughput increase. Throughput only increases 60% when doubling from 10 → 20 servers. What law explains this?**
   - A: Amdahl's Law and the Universal Scalability Law. Amdahl: speedup limited by the serial portion of the workload. USL adds coherency overhead — as servers increase, coordination overhead grows quadratically (O(N²)). The formula: `C(N) = N / (1 + σ(N-1) + κN(N-1))`. At some point, adding servers actually decreases throughput. Identify serial contention points: shared database, distributed locks, cache coherency.

8. **Q: Your API has high latency but low CPU and low memory. Thread dumps show all threads in `TIMED_WAITING` on `SocketRead`. What's the bottleneck?**
   - A: Network I/O — threads are waiting for responses from downstream services. The bottleneck is either network latency or the downstream service itself. Use distributed tracing to find which external call is slow. Common causes: DNS resolution delays, TLS handshake overhead, or a slow third-party API. Fix: increase connection pool sizes, use connection keepalive, add HTTP/2 multiplexing, or implement circuit breakers.

9. **Q: Your application uses 8GB heap and has 5-second GC pauses every hour. What type of bottleneck is this and how do you diagnose?**
   - A: This is a memory management bottleneck. The GC pauses are caused by a full GC (major collection) triggered by heap exhaustion or fragmentation. Use JVM flags to diagnose: `-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log`. Analyze GC logs to determine which GC phase is slow. Solutions: (1) Switch to G1GC or ZGC for <10ms pause times. (2) Reduce object allocation rate. (3) Fix memory leaks (heap dump analysis with Eclipse MAT). (4) Increase heap size.

10. **Q: Your NoSQL database has high latency during peak hours. You add read replicas, but the primary node is still overloaded because writes go to the primary. What's the actual bottleneck?**
    - A: Write throughput of the primary node. Read replicas don't help with write load. Options: (1) Batch writes to reduce write operations. (2) Shard the database to distribute write load across primaries. (3) Use a write-behind cache to buffer writes. (4) Optimize write-heavy operations (remove secondary indexes, use lighter consistency levels). (5) Use asynchronous writes where eventual consistency is acceptable.

---

## Interview Questions

1. **What is a bottleneck in system performance?**
   - A: The slowest component that limits overall system throughput. Every system has a bottleneck; fixing one reveals the next. The goal of performance engineering is to systematically identify and resolve bottlenecks.

2. **Name five types of bottlenecks and their symptoms.**
   - A: CPU (high utilization, context switching), Memory (OOM errors, frequent GC, swapping), Disk I/O (high iowait, slow reads/writes), Network (packet loss, timeouts, high latency), Database (slow queries, lock contention, connection pool exhaustion).

3. **What is the N+1 query problem?**
   - A: One query fetches N parent entities, then N additional queries fetch related data for each. Result: 1 + N queries instead of 2. Fix with `JOIN FETCH`, `@EntityGraph`, or `@BatchSize`.

4. **How do you diagnose a CPU bottleneck?**
   - A: Use `top -H` for thread-level CPU, `async-profiler` for hot methods/flame graphs, `jstack` for thread dumps. Look for RUNNABLE threads consuming CPU. Profile CPU hotspots to identify inefficient algorithms.

5. **What is Amdahl's Law?**
   - A: Speedup = 1 / ((1-P) + P/N), where P is the parallelizable portion and N is the number of processors. Limits the benefit of parallelization — if 10% is serial, max speedup is 10x regardless of cores.

6. **How does thread pool sizing become a bottleneck?**
   - A: Too few threads underutilize CPU (requests queue). Too many threads increase context switching overhead and memory waste. Formula for I/O-bound: threads = cores × (1 + wait_time/service_time). CPU-bound: threads = cores or cores + 1.

7. **What is the Universal Scalability Law?**
   - A: Models throughput as C(N) = N / (1 + σ(N-1) + κ N (N-1)). σ = contention overhead, κ = coherency overhead. Predicts the optimal number of nodes — beyond which adding more nodes degrades throughput.

8. **How do you identify database query bottlenecks?**
   - A: Enable slow query log, use `EXPLAIN ANALYZE`, check for sequential scans (missing indexes), monitor lock contention, connection pool utilization, and replication lag. Use database-specific tools: `pg_stat_statements` for PostgreSQL, `performance_schema` for MySQL.

9. **What is connection pool exhaustion?**
   - A: All database connections are in use, new requests wait in the pool queue. Caused by slow queries holding connections too long or insufficient pool size. Symptoms: increased latency, thread pool exhaustion upstream. Fix: optimize queries, increase pool size, add read replicas.

10. **How do you handle cascading bottleneck propagation in microservices?**
    - A: Circuit breakers (stop calling slow services), bulkheads (isolate resources per service), load shedding (drop low-priority requests), backpressure (propagate saturation signals upstream), graceful degradation (disable non-critical features), async processing (decouple via messaging).

---

## Developer Recommendations

- **Measure before optimizing** — The #1 performance mistake is optimizing what you think is slow without measuring. Use profilers (async-profiler, JFR), metrics (Micrometer + Prometheus), and distributed tracing to identify the actual bottleneck. "Make it work, make it right, make it fast — in that order." Most "obvious" bottlenecks turn out not to be bottlenecks after measurement. A team once spent a week rewriting their serialization layer to use Protobuf, convinced JSON parsing was the bottleneck — profiling later showed the real culprit was an N+1 query adding 3 seconds per request.

- **Fix the N+1 query problem first** — N+1 is the most common and most impactful database bottleneck. It turns 2 queries into 1+N. Always check for it when an API endpoint is slow. Enable Hibernate SQL logging in development and look for repeated identical queries. Use `JOIN FETCH`, `@EntityGraph`, or `@BatchSize` to fix it. This single fix resolves 80% of "slow API" issues.

- **Monitor thread pool utilization, not just CPU** — CPU at 50% with 100% thread pool utilization means threads are waiting (blocked on I/O, locks, or network). CPU at 95% with low thread utilization means the bottleneck is CPU. Both scenarios require different optimizations. Always monitor both: thread pool active count, queue depth, and rejected count alongside CPU usage.

- **Understand the bottleneck chain** — Fixing the first bottleneck reveals the next. DB query slow (100ms) → optimized to 10ms → CPU bottleneck (95%) → optimize algorithm → network bandwidth (saturated) → load balancing → downstream service (slow). Don't stop after one fix — the system's new throughput will be constrained by the next bottleneck.

- **Use caching before scaling** — Adding application servers or database replicas is expensive. Caching often resolves the bottleneck with zero infrastructure changes. Add Redis for API response caching, Caffeine for hot objects, and CDN for static content. A 90% cache hit ratio effectively reduces load by 10x.

- **Set timeouts on all external calls** — Without timeouts, a slow downstream service blocks threads indefinitely. Set connect timeout (500ms), read timeout (P99 + buffer), and connection pool timeout. Use circuit breakers to fail fast when the downstream is unhealthy. Timeouts convert thread exhaustion (system down) into graceful degradation (slightly slower).
- **Establish baseline metrics before optimizing** — Record P50/P95/P99 latency, throughput, CPU, memory, and GC behavior during normal load before making any changes. Without baselines, you can't objectively measure improvement. Store historical metrics in Prometheus with at least 3 months of retention to compare performance across releases.
- **Use the bottleneck chain approach — fix one, find the next** — Performance optimization is iterative, not one-shot. After fixing a database query (100ms → 10ms), re-measure: the new bottleneck might be CPU (now at 95%) from serialization, or network bandwidth from increased throughput. Each fix shifts the constraint to the next resource. Plan for multiple optimization cycles.
