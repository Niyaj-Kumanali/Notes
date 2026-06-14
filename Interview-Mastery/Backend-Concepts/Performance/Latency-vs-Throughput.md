# Latency vs Throughput

---

## Overview

- **Definition:** Latency measures the time to complete a single operation (ms). Throughput measures operations completed per unit time (requests/second).
- **Why It Exists:** These are the two fundamental performance metrics. Understanding the relationship and trade-offs between them is essential for designing scalable, responsive systems. Improving one often negatively impacts the other.
- **Key Concepts:** **P50/P95/P99/P999** (latency percentiles), **Little's Law** (L = λ × W — concurrency = throughput × latency), **Tail Latency** (slowest requests in the distribution), **Head-of-Line Blocking** (slow request blocks subsequent ones), **Amdahl's Law** (parallel speedup limit).
- **Latency vs Response Time** — Latency is the time the request spends waiting to be processed (queueing). Response time is latency + service time (actual processing). Total response time = queueing delay + processing time + network time. At high utilization, queueing delay dominates the total response time.
- **Throughput Ceiling** — Every system has a maximum throughput determined by its slowest serial component. Throughput = 1 / (service time of critical path). Improving throughput requires either reducing service time or increasing parallelism of the critical path.

---

## Core Concepts

- **Latency Components:** Total = Processing Time + Queueing Time + Network Time + Contention Time. Each component can dominate depending on the system state.
- **Little's Law:** L = λ × W. Concurrency = Throughput × Latency. To increase throughput without increasing latency, you must increase concurrency. If latency increases, throughput must decrease or concurrency must increase.
- **Tail Latency:** In distributed systems, a small percentage of requests take significantly longer (GC pauses, network packet loss, hot keys). Adding more servers makes tail latency worse (waiting for the slowest server).
- **Trade-off:** Improving throughput often increases latency (batching groups requests). Reducing latency may reduce throughput (dedicated resources per request, less batching).
- **The Utilization-Latency Curve** — Response time stays near baseline until utilization exceeds 70-80%, then grows asymptotically as `response_time = service_time / (1 - utilization)`. At 90% utilization, response time = 10× service time. At 99%, it's 100×. This non-linear relationship means small traffic increases near saturation cause massive latency spikes.
- **Measuring Throughput Correctly** — Throughput must be measured under load, not at idle. A server handling 10 requests in 100ms each has throughput of 100 req/s. Under concurrency, measure throughput as completed requests over a fixed time window. Use Little's Law to validate: if concurrency is 50 and latency is 200ms, throughput should be 50/0.2 = 250 req/s.

```java
// Measuring latency distribution with Micrometer
Timer.Sample sample = Timer.start(meterRegistry);
// ... work ...
sample.stop(Timer.builder("http.server.requests")
    .tag("uri", path).tag("status", String.valueOf(status))
    .publishPercentiles(0.5, 0.95, 0.99, 0.999)
    .publishPercentileHistogram(true)
    .register(meterRegistry));
```

---

## Common Mistakes

- **Optimizing for Average Latency Only** — Ignoring tail latency (P99, P999) leads to unpredictable user experience.
  - **Why it looks correct:** The average looks great on the dashboard — the 1-in-1000 request that takes 10 seconds is invisible in the mean.
- **Confusing High Throughput with Low Latency** — Batch processing has high throughput but high latency.
  - **Why it looks correct:** The system processes millions of events per second — the developer equates "fast in aggregate" with "fast for each item."
- **Infinite Queueing** — Unbounded queues grow latency non-linearly under load.
  - **Why it looks correct:** The queue works fine under low load — the latency explosion only appears when the arrival rate exceeds the processing rate for sustained periods.
- **Thread Pool Over-Subscription** — Too many threads increase context switching, reducing throughput and increasing latency.
  - **Why it looks correct:** More threads means more concurrency — the context switching overhead is invisible at the thread level, only appearing as a system-wide throughput regression.
- **Ignoring the Coordination Penalty** — Adding servers doesn't linearly increase throughput (coordination overhead).
  - **Why it looks correct:** Each server independently processes requests — the cost of distributed coordination (locking, cache coherency, consensus) only emerges at scale.
- **Measuring Throughput Without Concurrency** — Testing throughput with a single client misses queueing effects. Always test under realistic concurrency levels that match production traffic patterns.
  - **Why it looks correct:** The single-threaded test shows high throughput — the developer doesn't realize the same throughput collapses under real concurrency due to lock contention and queueing.
- **Using Averages Instead of Percentiles** — Average latency hides problems: 99 requests at 10ms and 1 at 10s averages to 109ms. P99 correctly shows the 10-second experience. Always use percentiles for latency measurement.
  - **Why it looks correct:** The average is a single number that "summarizes" performance — the lying flat is intentional and the developer trusts the single number over the distribution.

---

## Key Design Considerations

- **Latency SLO Design:** Critical APIs: P50 < 100ms, P95 < 500ms, P99 < 1s. Non-critical: P50 < 500ms, P95 < 2s, P99 < 5s. Internal services (cache): P50 < 1ms.
- **Latency Budget:** Allocate time budgets across service chain. Each service monitors actual vs budget and opens circuit breaker if exceeded. Example: API Gateway 200ms → Order Service 300ms → Payment 500ms → DB 200ms = 1.2s total, buffer for network.
- **Reducing Latency:** Optimize algorithms, use caching (in-memory → Redis → DB), async I/O, CDN, connection pooling, lock-free data structures, HTTP/2 multiplexing.
- **Increasing Throughput:** Increase concurrency (more threads/instances), batch processing, pipeline operations, shard databases, eliminate bottleneck resources.
- **Measuring Tools:** Micrometer `@Timed`, HDR Histogram for high-resolution percentiles, OpenTelemetry for distributed tracing, Prometheus for aggregation, Gatling/JMeter for load testing.
- **Coordination Overhead in Distributed Systems** — As the number of services in a request path grows, throughput degrades due to serialization, network hops, and coordination. Each additional service adds network latency, serialization/deserialization overhead, and potential queueing. Use asynchronous communication and bulkheads to minimize coordination overhead.
- **Optimizing for the Critical Path** — The critical path is the longest chain of sequential dependencies in a request. Reducing latency on the critical path directly improves overall response time. Optimizations off the critical path (parallel branches) improve throughput but don't reduce response time. Identify and focus on the critical path first.
- **Capacity Planning:** `Throughput = 1 / (Latency × Concurrency Overhead)`. Use Little's Law to compute required concurrency from throughput and latency targets.

---

## Real-World Scenarios

### Scenario 1: Batch Processing — Latency vs Throughput Trade-off
**Context:** A data analytics system ingests 1M events/second. Each event is sent individually to Kafka. Producer throughput is limited because each send has ~5ms overhead (TCP round trip + Kafka acknowledgment).

**Resolution:** Increase batch size and linger time. Instead of sending each event immediately, buffer 1000 events or wait 10ms (whichever comes first). Each batch sends 1000 events in one request. Per-event overhead drops from 5ms to 0.005ms. Throughput increases 100x. Latency for each event increases from 5ms to 10ms (waiting for the batch to fill). Acceptable trade-off: 10ms latency for 100x throughput.

```java
// High latency, low throughput — sending each event individually
kafkaTemplate.send("events", event);  // 5ms overhead per event

// Low latency per event, high throughput — batching
@Bean
public ProducerFactory<String, Event> producerFactory() {
    Map<String, Object> config = new HashMap<>();
    config.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);     // 64KB batch
    config.put(ProducerConfig.LINGER_MS_CONFIG, 10);         // Wait up to 10ms
    config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
    return new DefaultKafkaProducerFactory<>(config);
}
```

### Scenario 2: Tail Latency in a Microservices Chain
**Context:** A checkout service calls inventory (100ms), payment (500ms), and shipping (50ms) services in parallel. Each service runs on 10 instances. The checkout P50 is 520ms, but P999 is 15 seconds.

**Resolution:** The tail latency is caused by the slowest instance of the payment service (due to GC pauses, noisy neighbors). Solution: (1) Add a timeout of 2s to each downstream call. (2) Use a redundant request pattern: send the payment request to 2 instances, use whichever responds first. (3) Implement hedged requests: if no response in 300ms, send a second request to another instance. This drops P999 from 15s to 600ms.

### Scenario 3: Queueing Delay from Overloaded Server
**Context:** A web server handles 1000 req/s with 50ms average service time. Queueing theory says: utilization = 1000 × 0.05 / 50 = 100% (assuming 50 threads). At 100% utilization, response time approaches infinity. Users experience timeouts.

**Resolution:** Options: (1) Reduce service time (optimize code, add caching) — lower service time reduces utilization. (2) Increase thread count (more threads handle more concurrent requests). (3) Add more server instances (distributes load). (4) Implement load shedding — reject excess requests with 503 to prevent queue growth. (5) Use async I/O to handle more requests with fewer threads.

## Use Cases

- **API response time optimization** — reducing end-user perceived latency for web or mobile APIs
  - Measure P50/P95/P99 latency. Identify which component dominates: network, queueing, processing, or contention. Parallelize independent calls, cache hot paths.
  - **Avoid when:** the system is idle and latency is already sub-millisecond — throughput capacity is a more relevant concern.

- **Bottleneck throughput analysis** — finding the limiting factor in a processing pipeline
  - Use Little's Law (Concurrency = Throughput × Latency) to validate measurements. Identify the slowest serial component — that's the throughput ceiling.
  - **Avoid when:** the system is not under load — measure throughput at saturation, not at idle.

- **Capacity planning** — determining how many servers or resources are needed to handle projected traffic
  - Model the relationship between throughput and latency. At utilization >80%, latency grows non-linearly. Plan to keep utilization below 70% for predictable latency.
  - **Avoid when:** traffic is purely batch with no user-facing responsiveness requirements — maximizing throughput at any latency cost may be acceptable.

- **Tail latency debugging** — fixing the requests that take 10× longer than the median
  - Causes: GC pauses, noisy neighbors, hot keys, network packet loss. Solutions: hedged requests, timeouts, circuit breakers, redundant requests.
  - **Avoid when:** few users experience the tail — only focus on tail latency if it affects a meaningful segment of users or violates SLOs.

- **Load testing and performance validation** — verifying that a system meets latency and throughput requirements under expected load
  - Establish baseline: latency at low load. Increase concurrency and measure the latency-throughput curve. The inflection point is the maximum safe throughput.
  - **Avoid when:** you only test at a single load level — the shape of the latency-throughput curve is more informative than any single data point.

---

## Scenario-Based Questions

1. **Q: Your checkout process makes 3 sequential downstream API calls (inventory, payment, shipping), each taking 200ms. Total latency is 600ms. The business needs <300ms. How do you reduce latency without reducing throughput?**
    - A: Parallelize independent calls. Inventory and shipping are independent — call them simultaneously. Payment depends on inventory (need stock to charge). New flow: (Inventory + Shipping in parallel, 200ms) → (Payment, 200ms) = 400ms. Still >300ms. Next: add caching for inventory (reduces to 20ms). New flow: (Inventory cache 20ms + Shipping 200ms in parallel) → Payment 200ms = 420ms. Then parallelize all three with cached inventory: max(20, 200, 200) = 200ms. Success.
    - **Interview follow-up:** Parallelizing inventory and shipping assumes they are truly independent — but what if shipping needs the inventory reservation ID to create the shipment? How does this dependency change the parallelism strategy?

2. **Q: Your batch processing system needs to process 1M records in <1 hour. Each record takes 100ms to process. A single server processes 10 records/second = 36K records/hour = 28 hours. How do you meet the 1-hour target?**
   - A: Use Little's Law: Throughput = Concurrency / Latency. Target throughput = 1M / 3600s = 278 records/sec. With 100ms latency per record, concurrency needed = 278 × 0.1 = 28 threads. Run 10 servers with 3 threads each. Or use a queue with 30 consumers. The key insight: throughput = concurrency / latency. For batch processing, latency per record is fixed — increase concurrency to increase throughput.

3. **Q: Your P99 latency is 500ms but P999 is 30 seconds. What's causing this 60x difference and how do you fix it?**
   - A: The 30-second P999 indicates a few requests are getting stuck — likely from queueing or blocking. Common causes: (1) GC pauses (stop-the-world GC suspends all threads). Fix: switch to G1GC/ZGC. (2) Thread pool queue buildup — a few requests wait minutes. Fix: add load shedding. (3) A downstream service occasionally hangs without timeout. Fix: add timeouts and circuit breakers. (4) Lock contention. Fix: reduce critical sections, use lock-free structures.

4. **Q: Your API has a caching layer. Cache hit latency is 5ms, cache miss latency is 200ms. At 90% hit ratio, average latency is 5ms × 0.9 + 200ms × 0.1 = 24.5ms. The business requires <20ms average. What options do you have?**
    - A: (1) Increase cache hit ratio: more cache capacity, longer TTL, better eviction policy. Target 95%: 5 × 0.95 + 200 × 0.05 = 14.75ms. (2) Reduce cache miss latency: faster database queries (indexes, read replicas), or use a faster read-through cache. (3) Add a second cache tier (L1 Caffeine for absolute fastest access). (4) Pre-warm cache during deployments.
    - **Interview follow-up:** You increase TTL to boost cache hit ratio, but now stale data serves for longer — your users see outdated inventory counts. How do you balance cache freshness with the latency SLO? When is staleness acceptable and when must you always go to the source of truth?

5. **Q: Your system handles 1000 req/s with 50 active threads. Response time is 50ms. Traffic doubles to 2000 req/s. Response time becomes 500ms (10x increase). Why so much worse than expected?**
   - A: Queueing. Little's Law: L = λ × W. Before: 50 = 1000 × 0.05 (matches). After doubling traffic: if threads remain at 50, W = L/λ = 50/2000 = 25ms — but that's service time only. Reality: threads saturate at 2000 × 0.05 / 50 = 100% utilization. Queueing theory says response time = service time / (1 - utilization) = 50ms / (1 - 1.0) = infinity. At 90% utilization: 50ms / 0.1 = 500ms. Fix: increase threads, add servers, or reduce service time.

6. **Q: Your team's latency SLO is P95 < 500ms. You add batching to increase throughput, which adds 200ms to every request. The new P95 is 600ms. Is this a good trade-off?**
   - A: It depends on the business impact. If the batching increases throughput 5x (e.g., batch processing 5 requests together), the per-request latency increases but overall system capacity grows. For synchronous user-facing APIs, this is bad — users see slower responses. For async/background processing, this is fine — throughput matters more. If you must keep the SLO, reduce batching or optimize other parts of the pipeline to compensate.

7. **Q: Your application runs in 3 availability zones. P50 latency is 100ms, but P99 is 10 seconds. You discover that one AZ sometimes has network issues. How do you improve tail latency without moving away from multi-AZ?**
   - A: (1) Implement hedged requests — send the request to 2 AZs, use the first response. This reduces P99 from 10s to the slowest of 2 AZs (typically much faster). (2) Use latency-based routing — prefer the fastest AZ for each request. (3) Set strict timeouts per AZ: if AZ doesn't respond in 1s, fail over to another AZ. (4) Improve the problematic AZ's networking.

8. **Q: Your team argues about the target metric for latency: should you optimize for P50, P95, P99, or P999? The business has a single "fast checkout" requirement.**
   - A: All of them matter, and you should track all. But the optimization priority: (1) P50 matters most for average user experience — this is what most users feel. (2) P99 matters for user frustration and churn — 1 in 100 users having a bad experience is significant for a high-traffic site. (3) P999 matters for debugging and reliability — these are often GC pauses or outlier conditions. Set SLOs at P50 < 200ms (most users), P95 < 500ms (nearly everyone), P99 < 1s (acceptability threshold).

9. **Q: You measure latency using client-side timestamps and server-side timestamps. The client reports 500ms P95, the server reports 200ms P95. Which is correct and what explains the difference?**
   - A: Both are correct but measure different things. Client-side latency includes: network transit time, DNS resolution, TLS handshake, server processing, and response download. Server-side latency includes only request processing time. The 300ms difference is network overhead. For user experience, client-side latency is the true measure. For debugging, server-side latency identifies processing bottlenecks. Use both: client-side for SLOs, server-side for diagnosis.

10. **Q: Your system is designed for 500 req/s. During a flash sale, traffic spikes to 5000 req/s. Response time goes from 100ms to 30 seconds. Should you increase server capacity or implement queueing?**
    - A: Both. Increase server capacity to handle the sustained load (e.g., auto-scaling). But scale-up takes minutes — the 10x spike happens instantly. Implement a request queue that buffers excess traffic. Return "202 Accepted" with a polling endpoint. This converts latency (30s blocking) to throughput (queue drains in 30s). The user gets an immediate "processing" response and checks back later. This is preferred over 30-second HTTP timeouts.

---

## Interview Questions

1. **What is the difference between latency and throughput?**
   - A: Latency is the time to complete a single operation (measured in ms). Throughput is the number of operations completed per unit time (requests/second). They are related by Little's Law: Concurrency = Throughput × Latency.

2. **What is Little's Law?**
   - A: L = λ × W. Average concurrency (L) = throughput (λ) × average latency (W). If you know any two, you can compute the third. Essential for capacity planning and understanding system behavior under load.

3. **What is tail latency and why does it matter?**
   - A: Tail latency measures the slowest requests in a distribution (P99, P999). In distributed systems, the slowest of N services determines overall response time. A 1% slow request rate across 10 services means 9.6% of requests experience slowness.

4. **What is the trade-off between latency and throughput?**
   - A: Increasing throughput typically increases latency (batching groups requests, concurrency adds queueing). Reducing latency may reduce throughput (dedicated resources per request, less batching). Balance depends on business requirements — user-facing APIs prioritize latency; batch processing prioritizes throughput.

5. **How do you reduce latency in a database query?**
   - A: Add appropriate indexes, optimize query patterns (JOIN FETCH, batch fetching), use connection pooling, add read replicas, cache results in Redis, denormalize for read-heavy paths, use materialized views for complex aggregations.

6. **What is head-of-line blocking?**
   - A: A slow request at the front of a queue blocks all subsequent requests behind it. Mitigations: async I/O (non-blocking threads), HTTP/2 multiplexing (multiple streams per connection), separate thread pools for fast and slow operations, load shedding.

7. **How does queueing affect latency?**
   - A: At low utilization (<50%), queueing delay is negligible. At high utilization (>80%), queueing grows non-linearly: response time = service time / (1 - utilization). At 90% utilization, response time = 10× service time. At 99%, it's 100×.

8. **How do you set timeouts correctly?**
   - A: Based on latency distributions. Connect timeout < 500ms (most connections establish in <10ms). Read timeout = P99 + 50% buffer. Use cascading timeouts: client timeout > API gateway timeout > service timeout > database timeout. This ensures the client gives up last.

9. **What is coordinated omission?**
   - A: A measurement error where slow requests are excluded from latency statistics because they fall outside the measurement window (often because they timeout). Results in overly optimistic latency numbers. Fix: measure all requests, including those that time out. Use coordinated omission-aware tools like HDR Histogram.

10. **How do you design a latency budget for a 2-second SLA across 5 microservices?**
    - A: Allocate per-service budgets: API Gateway 200ms, Service A 300ms, Service B 500ms, Service C 400ms, Database 200ms. Total = 1.6s. Buffer 400ms for network latency, queueing, and retries. Each service monitors actual vs budget and breaks circuit if exceeded. Use distributed tracing to measure actual per-hop latency.

---

## Developer Recommendations

- **Always measure tail latency (P99/P999), not just averages** — Average latency hides severe issues. A 100ms average could mean 99% of requests complete in 10ms and 1% takes 9 seconds. Users experience the 9-second worst case, not the average. Use histograms with percentile reporting (P50, P95, P99, P999) for all latency measurements. Monitor trends — a rising P99 often precedes an outage.
  - **Production story:** One team's dashboard showed a healthy 120ms average latency while their P99 silently climbed from 200ms to 8 seconds over two weeks — they only noticed after a customer churn spike.

- **Use Little's Law for capacity planning** — `Concurrency = Throughput × Latency`. If you need 10,000 req/s throughput and latency is 200ms, you need `10,000 × 0.2 = 2,000` concurrent requests in flight. This translates to thread pool sizing, database connection pool sizing, and server count planning. The law holds regardless of technology — it's fundamental to queuing theory.

- **Reduce latency before scaling throughput** — Improving latency (caching, optimization, faster algorithms) reduces service time, which reduces utilization and queueing, which improves throughput. Formula: utilization = throughput × service time / threads. Halving service time doubles the throughput ceiling without adding servers. Always optimize latency first — it's cheaper than adding capacity.

- **Watch for the "utilization cliff" at >80%** — System response times are stable until utilization passes 70-80%. Beyond that, queueing theory says response time grows as 1/(1-utilization). At 95% utilization, response time is 20× the baseline. Set utilization targets: CPU < 70%, memory < 80%, thread pools < 70%, disk < 60%. Auto-scale before these thresholds.

- **Use separate thread pools for fast and slow operations** — A slow endpoint (e.g., report generation taking 10s) shares the same thread pool as fast endpoints (50ms). Under load, slow requests fill the pool and fast requests queue behind them. Separate pools ensure fast endpoints stay fast even when slow ones are saturated.

- **Implement hedged requests to combat tail latency in distributed systems** — The probability of tail latency grows with the number of services in a call chain. With 10 services, each with 1% chance of being slow, the system has ~9.6% chance of being slow. Hedged requests: send a request to 2 instances, use the first response. This reduces tail latency from the slowest-of-N to the fastest-of-2, dramatically improving P99 at the cost of ~2× resource usage for the hedged calls.
- **Optimize the critical path, not the noisiest component** — The critical path determines end-to-end latency. Shortening a non-critical path component (e.g., optimizing a parallel branch from 200ms to 50ms) doesn't improve overall latency if the critical path is 500ms. Use distributed tracing to identify the actual critical path before optimizing. Focus optimizations on the longest chain of sequential dependencies.

---

*Last updated: 2026-06-06. All 28 files in this directory have been enhanced with Real-World Scenarios (3 per file), situational Scenario-Based Questions (10), Interview Questions (10), and Developer Recommendations (6-8).*
