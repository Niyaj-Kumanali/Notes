# Circuit Breaker Pattern

---

## Overview

- **Definition:** A resilience design pattern that detects failures and prevents cascading failures in distributed systems by failing fast when a downstream service is unhealthy.
- **Why It Exists:** Without circuit breakers, a failing downstream service causes upstream services to waste resources waiting for timeouts, leading to cascading failures across the system.
- **Key Concepts:** **CLOSED** (normal operation, failures counted), **OPEN** (requests fail immediately, timer starts), **HALF_OPEN** (limited test requests to check recovery), **Failure Threshold** (count or rate that triggers OPEN), **Sliding Window** (time or count window for failure calculation), **Fallback Method** (alternative behavior when circuit is open), **Resilience4j** (Java fault-tolerance library)
- **Circuit Breaker vs Retry** — Retry handles transient failures (network glitches, connection resets) by re-executing the same call. Circuit breaker handles persistent failures by stopping calls entirely. They work together: retry first (e.g., 3 attempts with exponential backoff), then circuit breaker for remaining failures. Never use retry without a circuit breaker — infinite retries against a dead service exhaust resources.
- **State Transition Events** — Each state transition (CLOSED→OPEN, OPEN→HALF_OPEN, HALF_OPEN→CLOSED) should emit an event for monitoring. Resilience4j exposes these via `CircuitBreakerRegistry` event listeners. Log and alert on every transition to OPEN — it indicates a downstream service is unhealthy and may require operational response.

---

## Core Concepts

- **Circuit Breaker States:** CLOSED — requests flow normally, failures are counted; OPEN — requests fail immediately without calling the service; HALF_OPEN — limited test requests pass through, success resets to CLOSED, failure returns to OPEN.
- **Key Parameters:** `failureRateThreshold` — % of failures to open circuit; `slidingWindowSize` — number of requests in the window; `minimumNumberOfCalls` — minimum calls before rate calculation; `waitDurationInOpenState` — time in OPEN before HALF_OPEN; `permittedNumberOfCallsInHalfOpenState` — test calls allowed.
- **Failure Counting Strategies:** Count-based (consecutive or total failures), Rate-based (percentage in sliding time window), Hybrid (both count and rate thresholds).
- **Thread Pool vs Semaphore Isolation:** Thread Pool — each call runs in a separate thread, complete isolation, higher overhead. Semaphore — calls run on the calling thread, lower overhead, less isolation.
- **Sliding Window Types** — Count-based: evaluates the last N calls (sliding window resets after N calls regardless of time). Time-based: evaluates calls within the last N seconds (better for variable traffic — ensures the window contains enough data). Choose count-based for consistent traffic patterns; time-based for bursty traffic where call volume fluctuates.
- **Slow Call Detection** — Circuit breakers catch failures (exceptions), but slow responses are equally damaging. Configure `slowCallDurationThreshold` (e.g., 500ms) and `slowCallRateThreshold` (e.g., 50%). When 50% of calls exceed 500ms, the circuit opens — even without exceptions. This catches performance degradation before hard failures occur.

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> customizer() {
    return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
        .circuitBreakerConfig(CircuitBreakerConfig.custom()
            .slidingWindowType(COUNT_BASED)
            .slidingWindowSize(10)
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(3)
            .build())
        .build());
}

@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResponse processPayment(PaymentRequest request) {
    return restTemplate.postForObject("http://payment-service/api/payments", request, PaymentResponse.class);
}

public PaymentResponse paymentFallback(PaymentRequest request, Throwable t) {
    return PaymentResponse.builder().status("PENDING").message("Payment queued for later processing").build();
}
```

---

## Common Mistakes

- **No Fallback for OPEN State** — throwing an exception when the circuit is open instead of providing a fallback method that returns cached data or a sensible default.
  - **Why it looks correct:** throwing an exception makes the failure visible and avoids returning stale or incorrect data — the problem is that throwing an exception still fails the request, defeating the purpose of the circuit breaker's graceful degradation.
- **Too Sensitive Thresholds** — opening the circuit on a single failure causes unnecessary outages; set a minimum call count and rate threshold.
  - **Why it looks correct:** any failure is undesirable, so reacting immediately seems like the safest approach — the circuit tripping on a single transient blip causes more downtime than the failure itself.
- **Too Long Wait in OPEN State** — 30 seconds is usually sufficient; longer waits prolong degradation unnecessarily.
  - **Why it looks correct:** waiting longer gives the downstream more time to recover, which seems safer — the extended wait just means users experience degraded fallback behavior for longer than necessary.
- **Not Recording Relevant Exceptions** — recording only generic `Exception.class` instead of specific exceptions like `TimeoutException`, `ConnectException`, `HttpServerErrorException`.
  - **Why it looks correct:** catching all exceptions is the simplest configuration and ensures nothing is missed — recording too many exception types (including client errors) causes the circuit to open for the wrong reasons.
- **Circuit Breaker Without Monitoring** — if you can't see circuit breaker state changes, you can't respond to outages.
  - **Why it looks correct:** the circuit breaker handles failures silently and the system continues operating — without visibility, you only discover degraded service when users report that recommendations look stale or payments are pending.
- **Shared Circuit Breaker for Different APIs of the Same Service** — If a service has a health endpoint (always OK) and a payment endpoint (failing), a single circuit breaker for the whole service treats both the same. Use separate circuit breakers for different logical endpoints or operations within the same downstream service.
  - **Why it looks correct:** one circuit breaker per service is simpler to configure and manage — the shared circuit opens for the healthy endpoint too, blocking critical calls alongside the failing one.
- **Not Setting Minimum Number of Calls** — With default `minimumNumberOfCalls = 10`, the first failure on a quiet system can open the circuit. For low-traffic services, the circuit may never collect enough samples to make a statistically valid decision. Set `minimumNumberOfCalls` to 5-10 for high-traffic and 2-3 for low-traffic services.
  - **Why it looks correct:** the defaults seem reasonable and changing them requires understanding the traffic patterns — the default `minimumNumberOfCalls` of 10 means a low-traffic service with 3 requests per minute will never open its circuit breaker regardless of failure rate.

---

## Key Design Considerations

- **Fallback Strategies** — always provide a fallback: cache stale data, return default responses, queue requests for later retry, or degrade non-critical features before critical ones.
- **Auto-Recovery** — enable `automaticTransitionFromOpenToHalfOpenEnabled` for automatic recovery testing; consider gradual HALF_OPEN starting with 1% traffic.
- **Exception Classification** — record server errors (5xx, network timeouts, connection refused); ignore client errors (4xx) and authentication failures since they don't indicate downstream health.
- **Combined Resilience Patterns** — layer TimeLimiter (2s timeout), Retry (3 attempts, 500ms apart), CircuitBreaker (opens at 50% failure, 30s wait), and Bulkhead (max 10 concurrent calls) for complete protection.
- **Service Mesh Integration** — Istio and Linkerd provide circuit breaking at the network layer via sidecar proxies with centralized configuration and outlier detection.
- **Circuit Breaker Metrics and Alerts** — Export metrics via Micrometer: `circuit_breaker_state` (0=CLOSED, 1=OPEN, 2=HALF_OPEN), `circuit_breaker_failure_rate`, `circuit_breaker_call_count`, `circuit_breaker_slow_call_count`. Create Grafana dashboards showing circuit breaker states per service. Alert on any circuit remaining OPEN for more than 5 minutes — this indicates a persistent downstream issue requiring investigation.
- **Graceful Degradation with Stale Data** — When a circuit opens for a data service, the fallback should return the last known good data from a local cache rather than an error. For example, a product recommendation circuit breaker fallback returns cached recommendations that may be hours old. Users see slightly outdated recommendations instead of errors. The trade-off (stale data vs errors) almost always favors serving stale data.

---

## Real-World Scenarios

### Scenario 1: Payment Gateway Failure Cascade
**Context:** An e-commerce platform calls a payment gateway for every checkout. When the gateway slows down, all checkout threads block waiting for HTTP timeouts. Tomcat thread pools exhaust, and the entire site becomes unresponsive — including pages that don't use payments.

**Resolution:** Add a circuit breaker around the payment gateway call with a TimeLimiter (2s timeout). When failures hit 50% in a 10-request window, the circuit opens. The fallback returns a "Payment pending, you will receive a confirmation email" response. Other site features remain unaffected. After 30 seconds, a single test request probes the gateway; if it succeeds, the circuit closes.

```java
@CircuitBreaker(name = "paymentGateway", fallbackMethod = "paymentFallback")
@TimeLimiter(name = "paymentGateway")
@Bulkhead(name = "paymentGateway", type = THREADPOOL, maxWaitDuration = 5)
public CompletableFuture<PaymentResponse> processPayment(PaymentRequest request) {
    return CompletableFuture.supplyAsync(() ->
        restTemplate.postForObject("http://payment-gateway/api/charge", request, PaymentResponse.class));
}

public CompletableFuture<PaymentResponse> paymentFallback(PaymentRequest request, Throwable t) {
    return CompletableFuture.completedFuture(
        new PaymentResponse("PENDING", "Payment queued for later processing"));
}
```

### Scenario 2: Downstream Recommendation Service Degradation
**Context:** A product detail page calls a recommendation service that aggregates ML predictions. When the ML pipeline is being retrained, the service responds with 5-second latencies, dragging the page P99 from 200ms to 5.2s.

**Resolution:** Wrap the recommendation call with a circuit breaker configured for slow-call detection. Set `slowCallDurationThreshold = 500ms` and `slowCallRateThreshold = 50%`. When the circuit opens, the fallback serves cached or default recommendations. Users see slightly less relevant recommendations instead of a slow page.

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> recommendCB() {
    return factory -> factory.configure(builder -> builder
        .circuitBreakerConfig(CircuitBreakerConfig.custom()
            .slidingWindowType(COUNT_BASED).slidingWindowSize(20)
            .slowCallDurationThreshold(Duration.ofMillis(500))
            .slowCallRateThreshold(50)
            .failureRateThreshold(60)
            .waitDurationInOpenState(Duration.ofSeconds(15))
            .permittedNumberOfCallsInHalfOpenState(5)
            .build())
        .timeLimiterConfig(TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(2)).build())
        .build(), "recommendationService");
}
```

### Scenario 3: Database Connection Pool Exhaustion
**Context:** A burst of traffic causes slow database queries. Connection pool threads hold connections for 10+ seconds, exhausting the pool. New requests queue for pool access, and the queue grows unboundedly, eventually causing OOM.

**Resolution:** Layer a circuit breaker over the database access path. When the pool queue exceeds a threshold, open the circuit and serve stale cached data. This prevents queue growth from consuming memory and allows the DB to recover.

```java
@CircuitBreaker(name = "database", fallbackMethod = "dbFallback")
public List<Product> getProducts(String category) {
    return productRepository.findByCategory(category);
}

public List<Product> dbFallback(String category, Throwable t) {
    log.warn("DB circuit open, serving cached products", t);
    return cacheManager.getCache("products").get(category, List.class);
}
```

---

## Scenario-Based Questions

1. **Q: You are building a payment processing system that calls an external gateway. How do you prevent a gateway slowdown from taking down your entire service?**
   - A: Combine a CircuitBreaker with a TimeLimiter and Bulkhead. The TimeLimiter caps each call at 2s. The Bulkhead dedicates a separate thread pool for payment calls (max 10 threads). The CircuitBreaker opens at 50% failure rate over a 20-request window. The fallback returns a "processing" status and queues the payment for retry. This three-layer defense ensures a gateway issue affects only payments, not the entire application.

   - **Follow-up:** The fallback returns "processing" and queues the payment for retry, but 3 hours later the gateway is still down and the retry queue has 50,000 pending payments — how do you prevent the retry queue itself from becoming a failure vector?

2. **Q: A recommendation service occasionally has 5-second GC pauses. How do you configure a circuit breaker to handle slow responses differently from errors?**
   - A: Use `slowCallDurationThreshold` and `slowCallRateThreshold` alongside `failureRateThreshold`. Set the threshold at 500ms — any call exceeding this is counted as slow. When 50% of calls are slow in a 20-request window, the circuit opens. This catches degraded performance before hard errors occur. Add a separate TimeLimiter at 2s to cut off truly hung requests.

3. **Q: Your circuit breaker opens, but the downstream service recovers quickly. How do you minimize downtime while avoiding flapping?**
   - A: Use a short `waitDurationInOpenState` (5-15s) combined with gradual HALF_OPEN recovery. In HALF_OPEN, send only 1-3 test requests. If they succeed, transition to CLOSED. If even one fails, go back to OPEN. For critical services, use a gradual recovery — start with 1% of traffic, then 10%, then 100% — monitoring failure rates at each step.
   - **Follow-up:** With gradual recovery at 1% traffic, how do you ensure that the probe requests are representative of real traffic patterns and not just hitting a warmed-up cache or a healthy instance while other instances are still failing?

4. **Q: A third-party API charges per call. How do you balance circuit breaker protection with cost when they have occasional blips?**
   - A: Use a higher `minimumNumberOfCalls` (e.g., 50) and a longer sliding window before opening. This prevents brief blips from triggering protection and eating into your API budget. Set `failureRateThreshold` to 60-70% to tolerate minor issues. Consider a separate cost-aware fallback that degrades to cached responses for non-critical calls.
   - **Follow-up:** Your higher `minimumNumberOfCalls` of 50 means the circuit breaker won't open until 50 calls have been made — during a brief outage that affects only 10 requests, the circuit stays closed and all 10 fail. How do you balance protecting against costly API calls with providing fast failure detection?

5. **Q: You have multiple downstream services. How do you prevent one service's circuit breaker from starving another?**
   - A: Use the Bulkhead pattern alongside circuit breakers. Each downstream service gets its own thread pool with a fixed max (e.g., 10 threads for payment, 20 for recommendations, 50 for product catalog). This ensures one service's circuit breaker doesn't consume all available threads in the shared pool. Monitor each bulkhead's queue depth to adjust sizing.

6. **Q: How do you handle authentication failures in a circuit breaker — should 401 responses open the circuit?**
   - A: No. Authentication failures (4xx) indicate client issues, not downstream health. Configure the circuit breaker to record only 5xx errors, network timeouts, and connection refused exceptions. Use `recordExceptions` to specify exactly which exceptions count as failures: `ConnectException`, `TimeoutException`, `HttpServerErrorException`. Ignore `HttpClientErrorException`.
   - **Follow-up:** A downstream service changes its API contract and starts returning 400 Bad Request for all requests — the circuit breaker ignores 4xx and stays CLOSED, so every call still goes through and fails. How do you detect contract drift that manifests as a non-recorded exception type?

7. **Q: Your service calls a gRPC endpoint that streams results. How does circuit breaking work with streaming?**
   - A: Circuit breakers wrap the initial gRPC call establishment, not individual stream messages. If the stream setup fails or the initial connection times out, the circuit counts a failure. Once the stream is established, message-level failures are handled by the stream's own error handling. Use separate per-method circuit breakers for different RPCs on the same channel.

8. **Q: You deploy a new version of a downstream service that has a bug causing intermittent null pointer exceptions. How do your circuit breakers respond?**
   - A: If the NPEs propagate as 500 responses, they'll be recorded as failures. The circuit opens when the failure threshold is exceeded. To speed up detection, reduce `minimumNumberOfCalls` from 10 to 5 and `failureRateThreshold` from 50 to 40 for newly deployed services. Once the bug is fixed and the circuit closes, restore normal thresholds.
   - **Follow-up:** You tightened thresholds for the new deployment, but the old healthy version is still running alongside it in a canary — the circuit breaker is shared across all instances of the downstream service, so the healthy instances are now penalized for the buggy ones. How do you isolate circuit breaker state per instance or per deployment version?

9. **Q: How do you test that your circuit breakers actually work in production without causing real outages?**
   - A: Use chaos engineering: inject faults into specific service instances (e.g., using Chaos Monkey or Toxiproxy). Introduce 2-second delays on 50% of requests to a single instance. Verify that circuit breakers open, fallbacks execute, and the system degrades gracefully. Run these tests in staging first, then in production during low traffic with proper monitoring.

10. **Q: You have a circuit breaker around a Redis cache call. If Redis fails, should the circuit breaker prevent reads or fall through to the database?**
    A: Configure the circuit breaker to record only connection-level failures, not cache misses. The fallback should go directly to the database (cache-aside pattern). Set a short `waitDurationInOpenState` (5s) since Redis typically recovers quickly. Once the circuit closes, the cache warms up naturally as subsequent reads populate it. Never let a cache circuit breaker become a single point of failure.
    - **Follow-up:** All requests now fall through to the database when the cache circuit is open, causing the database connection pool to saturate under the extra load — how do you protect the database from the cache miss storm without adding another circuit breaker around the database?

---

## Interview Questions

1. **What is the Circuit Breaker pattern?**
   - A: A resilience pattern that detects failures and prevents cascading failures in distributed systems by failing fast when a downstream service is unhealthy. It transitions through three states: CLOSED (normal), OPEN (fail fast), and HALF_OPEN (testing recovery).

2. **What are the three states of a Circuit Breaker and what triggers each transition?**
   - A: CLOSED → OPEN (failure threshold exceeded), OPEN → HALF_OPEN (wait duration expires), HALF_OPEN → CLOSED (test requests succeed) or back to OPEN (test requests fail).

3. **What is the difference between circuit breaker and retry patterns?**
   - A: Circuit breaker provides long-term protection by stopping calls to persistently failing services. Retry handles short-term transient failures (network glitches, connection resets). Use both: retry first (e.g., 3 attempts with 500ms backoff), then circuit breaker for persistent failures.

4. **What is the Bulkhead pattern and how does it relate to circuit breakers?**
   - A: Bulkhead isolates resources (thread pools, connections) per downstream service — like compartments in a ship. Circuit breakers handle failure detection and fast-failing. Used together: Bulkhead limits resource usage per service; circuit breaker stops calling failing services entirely.

5. **How do you choose between count-based and time-based sliding windows?**
   - A: Count-based uses the last N calls regardless of timing — good for consistent traffic. Time-based uses calls within the last N seconds — better for variable traffic patterns where request volume fluctuates.

6. **What is the difference between thread pool and semaphore isolation in Resilience4j?**
   - A: Thread pool isolation runs each call in a separate thread — complete resource isolation, supports TimeLimiter, but higher overhead. Semaphore isolation runs on the calling thread — lower overhead, no TimeLimiter support, less isolation.

7. **How does a circuit breaker handle slow responses as failures?**
   - A: Via `slowCallDurationThreshold`. Calls exceeding the configured duration are counted as slow calls. When `slowCallRateThreshold` is exceeded, the circuit opens — even if no exceptions were thrown.

8. **What exceptions should be recorded vs ignored in circuit breakers?**
   - A: Record: ConnectException, TimeoutException, HttpServerErrorException (5xx), SocketException. Ignore: HttpClientErrorException (4xx), AuthenticationException, ValidationException — these indicate client issues, not downstream health.

9. **How do you test circuit breakers?**
   - A: Unit tests with mocked services throwing exceptions. Integration tests with togglable fault injection (e.g., WireMock). Chaos testing with network delays (Toxiproxy). Production verification via chaos engineering with proper monitoring and rollback plans.

10. **How does the HALF_OPEN state prevent flapping?**
    - A: HALF_OPEN allows a limited number of probe requests. If all succeed, the circuit closes. If any fail, it reopens. Gradual recovery (1% → 10% → 100% traffic) prevents a single success from immediately sending full traffic to a still-unstable service.

---

## Developer Recommendations

- **Layer circuit breaker with TimeLimiter and Bulkhead** — A circuit breaker alone doesn't prevent slow responses from blocking threads; it only stops calls after failures accumulate. TimeLimiter caps individual call duration (e.g., 2s). Bulkhead dedicates separate thread pools per downstream service. Together, they provide defense in depth: Bulkhead isolates, TimeLimiter cuts off slow calls, CircuitBreaker stops calling dead services.
  - **Production story:** A major airline had a circuit breaker on their booking API but no TimeLimiter — when the upstream seat-mapping service hung, the circuit breaker took 30 seconds and 10 failures to open, by which time all Tomcat threads were exhausted and the entire booking site was down for 12 minutes.

- **Use recordExceptions/ignoreExceptions explicitly** — The default records all exceptions as failures, including 4xx client errors. Explicitly configure `recordExceptions` to include only `ConnectException`, `TimeoutException`, `HttpServerErrorException` and `ignoreExceptions` for `HttpClientErrorException`. This ensures the circuit opens only for genuine downstream health issues, not bad client requests.
  - **Production story:** A fintech startup used the default exception recording and their circuit breaker opened every time a customer entered an invalid card number (400 Bad Request) — the circuit stayed open for 30 seconds, blocking all payment attempts despite the downstream service being perfectly healthy.

- **Use slow-call detection alongside failure-rate detection** — Failures (exceptions) alone miss the case where a service responds but takes 10 seconds. Enable `slowCallDurationThreshold` and `slowCallRateThreshold` to catch degraded performance. Set the slow-call threshold based on your P99 latency — typically 2-3x the normal P99.
  - **Production story:** A video streaming service set their slow-call threshold to 2x P99 but didn't account for a daily cache refresh that caused a 3-second P99 spike every morning — the circuit breaker opened daily at 9 AM, serving stale recommendations for 15 minutes before the team added a separate maintenance window exclusion.

- **Set minimumNumberOfCalls to avoid premature opening** — With `minimumNumberOfCalls = 10`, the circuit breaker waits for at least 10 calls before calculating the failure rate. Without this, a single failure on a quiet system would open the circuit. For low-traffic services, reduce this to 3-5; for high-traffic, keep it at 20+.
  - **Production story:** A background job that processed 2 orders per minute had `minimumNumberOfCalls = 10` — its circuit breaker never opened because it never collected 10 calls within the sliding window, silently allowing all failures to pass through until an alert on error rate finally caught the issue 6 hours later.

- **Provide meaningful fallbacks for every circuit breaker** — A fallback that throws an exception defeats the purpose. Good fallbacks: return cached data (even if stale), return a default response, queue the request for later processing, or degrade non-critical features. The fallback should let the system continue operating at reduced capacity.

- **Monitor circuit breaker state changes with metrics and alerts** — Export circuit breaker metrics (state, failure rate, call count) via Micrometer to Prometheus/Grafana. Create alerts for state transitions: "paymentService circuit breaker OPEN" should page the on-call engineer immediately. Without monitoring, circuit breakers silently degrade the user experience.

- **Use gradual HALF_OPEN recovery instead of a hard transition** — The default HALF_OPEN sends all permitted test calls at once. For critical services, implement a phased recovery: start with 1% of traffic, observe for 30 seconds, then 10%, then 100%. This prevents a recovered-but-wobbly service from taking down the entire system again.
- **Layer circuit breaker with TimeLimiter and Bulkhead for defense in depth** — A circuit breaker alone doesn't prevent slow responses from blocking threads. TimeLimiter cuts off slow calls at a configured duration (e.g., 2 seconds). Bulkhead dedicates a separate thread pool per downstream service (e.g., 10 threads for payments, 20 for catalog). Together: Bulkhead isolates, TimeLimiter cuts off, CircuitBreaker stops calling dead services. This three-layer defense prevents any single downstream failure from cascading.
- **Always provide a meaningful fallback that lets the system continue** — A fallback that throws an exception back to the user defeats circuit breaking. Good fallbacks: return stale cached data (even if outdated), return a default response (empty list instead of recommendations), queue the request for later processing, or degrade non-critical features. The fallback should let the system operate at reduced capacity rather than failing entirely.
