# Circuit Breaker Pattern

## 1. Executive Summary

The Circuit Breaker pattern is a resilience design pattern that detects failures and prevents cascading failures in distributed systems. Modeled after electrical circuit breakers, it monitors calls to external services. When failures exceed a threshold, the circuit "opens" and subsequent calls fail immediately (fast-fail) instead of waiting for the service to time out. After a recovery period, the circuit "half-opens" to test if the service has recovered, and either resets to "closed" or reopens.

## 2. Core Theory

### Circuit Breaker States

```
CLOSED (normal operation)
  |-- Failure threshold exceeded --> OPEN
  |                                     |
  |                              (timeout elapses)
  |                                     |
  |                                     v
  |                               HALF_OPEN
  |                                  /    \
  |                          Success      Failure
  |                             |             |
  |                             v             v
  |                          CLOSED        OPEN
```

- **CLOSED**: Requests flow normally. Failures are counted.
- **OPEN**: Requests fail immediately without calling the service. A timer starts.
- **HALF_OPEN**: Limited test requests pass through. If successful, reset to CLOSED; if failure, back to OPEN.

### Key Parameters
- **failureThreshold**: Number of failures before opening circuit (e.g., 5).
- **successThreshold**: Number of successes before closing in HALF_OPEN (e.g., 3).
- **timeout**: Duration to wait before transitioning from OPEN to HALF_OPEN (e.g., 30s).
- **slidingWindowSize**: Number of requests to consider for failure rate (e.g., 10).
- **minimumNumberOfCalls**: Minimum calls before calculating failure rate (e.g., 5).

## 3. Under-the-Hood Deep Dive

### Failure Counting Strategies
- **Count-based**: Count consecutive or total failures.
- **Rate-based**: Percentage of failures in a sliding time window.
- **Hybrid**: Both count and rate thresholds must be exceeded.

### Request Monitoring
The circuit breaker tracks:
- Total call count
- Success count
- Failure count (split by exception type)
- Call duration
- Slow call count (above configurable threshold)

### Thread Management

**Thread Pool Isolation vs Semaphore Isolation:**
- **Thread Pool**: Each downstream call runs in a separate thread. Complete isolation but higher overhead.
- **Semaphore**: Calls run on the calling thread. Lower overhead but less isolation.

```java
// Resilience4j thread pool isolation
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> customizer() {
    return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
        .circuitBreakerConfig(CircuitBreakerConfig.custom()
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(10)
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(3)
            .build())
        .timeLimiterConfig(TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(2))
            .build())
        .build());
}
```

## 4. Production Code Examples

### Resilience4j Circuit Breaker with Spring Boot

```java
@Service
public class PaymentServiceClient {

    @Autowired
    private RestTemplate restTemplate;

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @TimeLimiter(name = "paymentService")
    @Retry(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        return restTemplate.postForObject(
            "http://payment-service/api/payments",
            request,
            PaymentResponse.class
        );
    }

    // Fallback method must match return type
    public PaymentResponse paymentFallback(PaymentRequest request, Throwable t) {
        log.warn("Payment service unavailable, using fallback: {}", t.getMessage());
        return PaymentResponse.builder()
            .status("PENDING")
            .message("Payment queued for later processing")
            .build();
    }
}
```

### Configuration in application.yml

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 30s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10
        recordExceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignoreExceptions:
          - com.example.BusinessException
  timelimiter:
    instances:
      paymentService:
        timeoutDuration: 2s
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 500ms
        retryExceptions:
          - org.springframework.web.client.HttpServerErrorException
```

### Programmatic Circuit Breaker

```java
@Component
public class DatabaseCircuitBreaker {

    private final CircuitBreaker circuitBreaker;
    private final DataSource dataSource;

    public DatabaseCircuitBreaker(DataSource dataSource) {
        this.dataSource = dataSource;

        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.TIME_BASED)
            .slidingWindowSize(60) // 60 seconds window
            .minimumNumberOfCalls(10)
            .failureRateThreshold(30) // Open at 30% failure
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .permittedNumberOfCallsInHalfOpenState(5)
            .recordExceptions(SQLException.class, DataAccessException.class)
            .build();

        this.circuitBreaker = CircuitBreaker.of("database", config);
    }

    public <T> T executeWithBreaker(Supplier<T> databaseCall) {
        return circuitBreaker.executeSupplier(databaseCall);
    }

    public Connection getConnection() throws SQLException {
        return executeWithBreaker(() -> {
            try {
                return dataSource.getConnection();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        });
    }
}
```

### Circuit Breaker Event Listener

```java
@Component
public class CircuitBreakerEventHandler {

    @EventListener
    public void handleCircuitBreakerEvent(CircuitBreakerEvent event) {
        switch (event.getType()) {
            case ERROR -> log.error("Circuit breaker error on {}: {}",
                event.getCircuitBreakerName(), event.getThrowable().getMessage());
            case SUCCESS -> log.debug("Circuit breaker success on {}", event.getCircuitBreakerName());
            case STATE_TRANSITION -> {
                CircuitBreaker.StateTransition transition =
                    (CircuitBreaker.StateTransition) event.getAdditionalInformation();
                log.warn("Circuit breaker '{}' transitioned from {} to {}",
                    event.getCircuitBreakerName(),
                    transition.getFromState(),
                    transition.getToState());
                // Alert operations team on OPEN state
                if (transition.getToState() == CircuitBreaker.State.OPEN) {
                    alertOperations(event.getCircuitBreakerName());
                }
            }
            case RESET -> log.info("Circuit breaker '{}' reset to CLOSED", event.getCircuitBreakerName());
            case NOT_PERMITTED -> log.warn("Circuit breaker '{}' rejected call (OPEN state)",
                event.getCircuitBreakerName());
        }
    }

    @EventListener
    public void handleStateTransition(CircuitBreakerStateTransition transition) {
        if (transition.getToState() == CircuitBreaker.State.OPEN) {
            // Publish metric for monitoring
            metricsPublisher.publishEvent("circuitbreaker.opened", transition);
        }
    }
}
```

### Fallback with Custom Logic

```java
@Service
public class ProductCatalogService {

    @Autowired
    private ProductServiceClient productClient;

    @Autowired
    private LocalCacheService localCache;

    @CircuitBreaker(name = "productService", fallbackMethod = "getProductFromCache")
    public Product getProduct(String productId) {
        return productClient.fetchProduct(productId);
    }

    // Fallback: serve from local cache
    public Product getProductFromCache(String productId, Throwable t) {
        Product cached = localCache.getProduct(productId);
        if (cached != null) {
            log.warn("Serving product {} from cache due to: {}", productId, t.getMessage());
            return cached;
        }
        // Last resort: return basic product stub
        return Product.builder()
            .id(productId)
            .name("Product Unavailable")
            .price(BigDecimal.ZERO)
            .available(false)
            .build();
    }
}
```

### Bulkhead Pattern (Resource Isolation)

```java
@Service
public class BulkheadedService {

    @Bulkhead(name = "paymentService", type = Bulkhead.Type.THREADPOOL)
    @CircuitBreaker(name = "paymentService")
    public CompletableFuture<PaymentResponse> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() ->
            restTemplate.postForObject("http://payment-service/api/payments",
                request, PaymentResponse.class));
    }
}

// Configuration
resilience4j:
  bulkhead:
    instances:
      paymentService:
        maxConcurrentCalls: 10
        maxWaitDuration: 500ms
  thread-pool-bulkhead:
    instances:
      paymentService:
        maxThreadPoolSize: 5
        coreThreadPoolSize: 2
        queueCapacity: 10
```

## 5. Real-World Scenarios

### API Gateway Circuit Breakers
```
[Client] -> [API Gateway]
                |
    [User Service CB] [Order Service CB] [Payment Service CB]
                |
    [Fallback: cache/error page]
```

### Microservice Inter-Dependencies
```
[Order Service] -> [Payment Service] (CB here)
                -> [Inventory Service] (CB here)
                -> [Notification Service] (CB here, graceful degradation)
```

### Database Access
Circuit breaker on database connection pool: if DB is slow or failing, circuit opens and requests fail fast instead of queuing up in the connection pool.

### External API Integration
Circuit breaker for third-party APIs (payment gateways, SMS providers, email services). Fallback to queue for later retry.

## 6. Performance

### Overhead Measurement
- **CLOSED state**: Negligible overhead (counter increment per request).
- **OPEN state**: Minimal (state check + fast fail).
- **HALF_OPEN state**: Same as CLOSED but limited to permitted calls.

### Latency Impact
- Normal operation: +0.01ms overhead (state tracking).
- During OPEN state: -2000ms saving (avoiding timeout waiting).
- Overall: Circuit breaker reduces latency during failures by failing fast.

### Thread Pool vs Semaphore
| Aspect | Thread Pool | Semaphore |
|--------|------------|-----------|
| Overhead | Higher (context switching) | Lower |
| Isolation | Full (separate thread) | Partial (caller's thread) |
| Timeout support | Yes | No |
| Best for | Long-running calls | Fast calls (<10ms) |

## 7. Security

### Security Considerations
- Circuit breaker should NOT break for security failures (auth failures).
- Use `ignoreExceptions` for security-related exceptions.
- Fallback responses should not leak internal system state.

```java
@CircuitBreaker(name = "authService", ignoreExceptions = {
    AuthenticationException.class,
    AuthorizationException.class
})
public AuthResponse authenticate(String token) {
    return authClient.validateToken(token);
}
```

### Rate Limiting and Circuit Breaker
Circuit breaker protects the caller. Rate limiter protects the callee. Use both together.

## 8. Common Mistakes

### Mistake 1: No Fallback for OPEN State
```java
// WRONG - throws exception when circuit is open
@CircuitBreaker(name = "svc")
public Data fetchData() {
    return client.getData();
}

// RIGHT - provide fallback
@CircuitBreaker(name = "svc", fallbackMethod = "fallback")
public Data fetchData() {
    return client.getData();
}

public Data fallback(Throwable t) {
    return Data.empty();
}
```

### Mistake 2: Too Sensitive Thresholds
Opening circuit on single failure causes unnecessary outages. Set minimum call count and rate threshold.

### Mistake 3: Too Long Wait in OPEN State
30 seconds is usually sufficient. Longer waits lead to prolonged degradation.

### Mistake 4: Not Recording Relevant Exceptions
```java
// WRONG - only records generic Exception
@CircuitBreaker(name = "svc", recordExceptions = {Exception.class})

// RIGHT - record relevant ones
@CircuitBreaker(name = "svc", recordExceptions = {
    TimeoutException.class, ConnectException.class, HttpServerErrorException.class
})
```

### Mistake 5: Circuit Breaker Without Monitoring
If you can't see circuit breaker state changes, you can't respond to outages.

## 9. Senior Engineer Perspective

### Circuit Breaker Hierarchy
```
[Client]
    |
[Global CB] - opens if too many downstream failures
    |
[Service A CB] [Service B CB] [Service C CB]
```

### Graceful Degradation Strategies
1. **Cache fallback**: Serve stale data from cache.
2. **Default response**: Return sensible default.
3. **Queue for retry**: Store request for later processing.
4. **Prioritize**: Degrade non-critical features before critical ones.
5. **User feedback**: Inform user of degraded service.

### Auto-Recovery Considerations
- `automaticTransitionFromOpenToHalfOpenEnabled: true` enables automatic recovery testing.
- Consider doing gradual HALF_OPEN: start with 1% traffic, increase if successful.
- Combine with health checks for faster recovery detection.

### Integration with Service Mesh
Service mesh (Istio, Linkerd) provides circuit breaking at the network layer:
```yaml
# Istio DestinationRule for circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What is the Circuit Breaker pattern?
   **A:** A resilience pattern that detects failures and prevents cascading failures by failing fast when a downstream service is unhealthy.

2. **Q:** What are the three states of a Circuit Breaker?
   **A:** CLOSED (normal), OPEN (failing fast), HALF_OPEN (testing recovery).

3. **Q:** What happens when the circuit is OPEN?
   **A:** Requests fail immediately without calling the downstream service.

4. **Q:** What triggers transition from CLOSED to OPEN?
   **A:** Exceeding the failure threshold (count or rate) within the sliding window.

5. **Q:** What is the HALF_OPEN state?
   **A:** A testing state where limited requests are allowed through to check if the service has recovered.

6. **Q:** What is a fallback method in circuit breaker?
   **A:** A method that provides alternative behavior when the circuit is open or the call fails.

7. **Q:** What is the difference between circuit breaker and retry?
   **A:** Circuit breaker prevents calls to failing services (long-term protection). Retry retries failed calls (short-term recovery from transient failures).

8. **Q:** What is Resilience4j?
   **A:** A lightweight, easy-to-use fault tolerance library for Java/Spring Boot, providing circuit breaker, retry, bulkhead, rate limiter, and time limiter.

9. **Q:** What is the sliding window in circuit breaker?
   **A:** The time window or call count window used to calculate failure rate. Older calls outside the window are discarded.

10. **Q:** What is a slow call threshold?
    **A:** Calls taking longer than a configured duration are counted as slow calls, contributing to the failure rate.

### Medium

11. **Q:** Explain the difference between count-based and time-based sliding windows.
    **A:** Count-based: last N calls regardless of time. Time-based: calls within the last N seconds. Time-based is better for variable traffic.

12. **Q:** How do you choose the wait duration in OPEN state?
    **A:** Based on typical service recovery time: 10-60 seconds for most services. Longer for external APIs (minutes).

13. **Q:** What is the bulkhead pattern and how does it relate to circuit breaker?
    **A:** Bulkhead isolates resources (thread pools, connections) per downstream service. Circuit breaker prevents cascading failures. Used together for complete resilience.

14. **Q:** How do you handle circuit breaker state changes in monitoring?
    **A:** Expose circuit breaker metrics (state, call count, failure rate) via Micrometer/Prometheus. Alert on OPEN state transitions.

15. **Q:** What exceptions should be recorded vs ignored in circuit breaker?
    **A:** Record: network errors, timeouts, 5xx server errors. Ignore: 4xx client errors, authentication failures (not downstream service health issues).

16. **Q:** How does circuit breaker work with asynchronous calls?
    **A:** Resilience4j supports `CompletableFuture` with thread pool bulkhead. Circuit breaker wraps async call, tracks success/failure.

17. **Q:** What is the difference between Resilience4j and Hystrix?
    **A:** Hystrix is deprecated with Netflix dependency. Resilience4j is lighter, modular, supports Java 8+ functional programming, Micrometer integration.

18. **Q:** How do you test circuit breakers?
    **A:** Unit test with mocked service (throw exceptions), integration test with togglable fault injection, chaos testing with network delays/failures.

19. **Q:** What is the effect of circuit breaker on user experience?
    **A:** Without fallback: errors. With fallback: degraded but functional (cached data, default values, queued requests).

20. **Q:** Can circuit breaker detect slow responses as failures?
    **A:** Yes, via TimeLimiter. Calls exceeding timeout are recorded as failures.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Design a circuit breaker that dynamically adjusts its threshold based on historical patterns.
    **A:** Adaptive circuit breaker: ML model learns normal failure rate patterns (time-of-day, day-of-week). Threshold adjusts dynamically: if normal failure rate is 2%, threshold set at 5%. Detects anomalies: 10% failure rate triggers open.

2. **Q:** How do you implement a circuit breaker that handles cascading failures across service chains?
    **A:** Propagate circuit breaker state via context headers. Service A's circuit breaker opens; it returns CircuitBreaker-Open: true header. Service B propagates: if upstream CB is open, increase sensitivity. Shared circuit breaker state via distributed registry.

3. **Q:** Design a circuit breaker strategy for a multi-region deployment.
    **A:** Per-region circuit breakers. Redundant circuit: if primary region circuit opens, route to secondary region. Region-level circuit: if entire region's services are failing, route all traffic to other region.

4. **Q:** How do you implement a circuit breaker for a database connection pool?
    **A:** Wrap connection pool with circuit breaker parameters: waitDuration = 30s, failureThreshold = 50% of connection acquisition failures. Actuator health check: show circuit state. Fallback: read-only replica or cache.

5. **Q:** Design a circuit breaker that works with long-polling and streaming.
    **A:** Stream circuit breaker: track stream interruption rate. If streams fail more than threshold, open circuit and switch to polling mode. For long-polling: timeout-based failure detection.

6. **Q:** How do you differentiate between circuit breaker tripping for latency vs errors?
    **A:** Separate thresholds for slow calls and error calls. Example: circuit opens if >50% errors OR >70% slow calls. Different fallback strategies for each.

7. **Q:** Design a circuit breaker with gradual recovery (not full HALF_OPEN).
    **A:** Gradual recovery: start with 1% traffic, if successful increase to 5%, 10%, 25%, 50%, 100%. Each stage lasts 30 seconds. Rollback to OPEN if any failures in current stage.

8. **Q:** How do you handle idempotency across circuit breaker retries?
    **A:** Use idempotency key header. The downstream service uses this key to deduplicate requests. Even if circuit breaker allows retry on half-open, downstream ensures exactly-once processing.

9. **Q:** What is the relationship between circuit breaker and graceful degradation in microservices?
    **A:** Circuit breaker detects failure; graceful degradation decides what to do about it. CB is the mechanism; degradation is the strategy.

10. **Q:** How do you implement a distributed circuit breaker that prevents cascading across services?
    **A:** Distributed circuit breaker registry (Redis/Etcd). All service instances of the same service share circuit breaker state. If one instance opens circuit, other instances are notified via pub/sub.

### System Design

11. **Q:** Design a circuit breaker system for an API gateway handling 50+ downstream services.
    **A:** Per-service circuit breaker. Aggregated circuit breaker: if >5 services' circuits are open, open global circuit (fail fast for everything). Dashboard showing all circuit states. Auto-recovery with staggered half-open.

12. **Q:** Design a circuit breaker strategy for a payment processing pipeline.
    **A:** Circuit breakers per stage: payment gateway CB, fraud check CB, ledger CB. If payment gateway CB opens, queue payments for later processing. If fraud CB opens, allow payments but flag for manual review.

13. **Q:** Design a resilience strategy combining circuit breaker, retry, and time limiter.
    **A:** Order: TimeLimiter wraps call (2s timeout). Retry (3 attempts, 500ms apart). CircuitBreaker (opens after 5 failures, 30s wait). Bulkhead (max 10 concurrent calls). Each handles different failure modes.

14. **Q:** Design a circuit breaker system for a multi-tenant SaaS platform.
    **A:** Per-tenant circuit breakers: one noisy tenant shouldn't affect others. Tenant tiers: enterprise tenants have longer timeouts, higher thresholds. Circuit breaker state visible to tenant dashboard.

15. **Q:** Design a circuit breaker strategy for a real-time bidding system.
    **A:** Millisecond-latency circuit breaker: use semaphore isolation (no thread overhead). Threshold: if bid response time > 100ms, open circuit. Fallback: use last known price or reject bid.

16. **Q:** Design circuit breaker integration with Kubernetes health probes.
    **A:** Expose circuit breaker state in /actuator/health. If critical downstream circuit is open, pod reports itself as not healthy. Kubernetes restarts the pod or removes from service.

17. **Q:** Design a circuit breaker system for a social media platform's feed service.
    **A:** Multiple circuit breakers: Feed generation CB (open = show cached feed), Recommendation CB (open = hide recommended posts), Notification CB (open = skip non-critical notifications).

18. **Q:** Design a circuit breaker for a websocket/streaming connection.
    **A:** Track connection failures and reconnection rate. If reconnection attempts exceed threshold in window, open circuit and switch to polling. Periodic half-open to attempt WebSocket reconnection.

19. **Q:** Design circuit breaker fallback for a search service degradation.
    **A:** Circuit open = serve basic search (DB query instead of Elasticsearch). Degraded results but functional. Inform user: "Search is using basic mode". Cache popular search results for fallback.

20. **Q:** Design a circuit breaker monitoring and alerting system.
    **A:** Metrics: circuit state (1=closed, 2=half_open, 3=open), call count, failure rate, slow call rate. Alerts: OPEN state > 5 minutes, frequent state transitions, failure rate > 30%. Dashboard: circuit heatmap.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a self-tuning circuit breaker that adapts to changing traffic patterns.
    **A:** Reinforcement learning agent adjusts: failure threshold, wait duration, sliding window size. Inputs: time of day, traffic volume, downstream health metrics. Reward function: minimize downtime while maximizing availability.

2. **Q:** How do you implement circuit breaker propagation across a service mesh?
    **A:** Envoy/ Istio: circuit breaker at sidecar level. If sidecar detects upstream failures, it opens circuit. Downstream sidecars detect this via outlier detection headers and adjust their own circuit breaker sensitivity. Propagation metadata in Istio attributes.

3. **Q:** Design a circuit breaker for a financial exchange with microsecond latency requirements.
    **A:** Hardware-level circuit breaker: FPGA-based monitoring of market data feeds. If feed latency > 1 microsecond for 100 consecutive ticks, hardware circuit opens and switches to backup feed. Software layer: zero-allocation Java, pre-warmed threads, lock-free data structures.

4. **Q:** How do you implement circuit breaking for read replicas in a database cluster?
    **A:** Per-replica circuit breaker. If replica health check fails or query latency exceeds threshold, remove from read pool. Circuit opens for that replica. HALF_OPEN: periodically check replica health before adding back.

5. **Q:** Design a multi-layer circuit breaker strategy for a cloud-native application across availability zones.
    **A:** AZ-level circuit: if all services in us-east-1a are failing, open AZ circuit and route to us-east-1b. Service-level: within AZ, per-service circuit breaker. Global-level: if multiple AZs fail, open global circuit and serve static site.

6. **Q:** How do you ensure circuit breaker state consistency in an active-active multi-region deployment?
    **A:** Distributed circuit breaker state via CRDTs or last-writer-wins. Each region has local circuit breaker that asynchronously syncs state to global store. Region-local decisions are immediate; cross-region sync is informational.

7. **Q:** Design a circuit breaker that intelligently selects between multiple fallback strategies.
    **A:** Strategy selector: if primary CB open, try secondary service (if count < threshold), then cache, then default value, then queue. Each fallback has its own circuit breaker. Strategy selection based on call context (user tier, request type).

8. **Q:** How do you implement a circuit breaker for a function-as-a-service (FaaS/serverless) architecture?
    **A:** External circuit breaker service (Redis/API). Each function checks circuit state before calling downstream. Circuit open: function returns early or calls alternative function. State stored in distributed cache for cross-function coordination.

9. **Q:** Design a circuit breaker that provides probabilistic HALF_OPEN traffic (not fixed count).
    **A:** Instead of fixed 3 calls in HALF_OPEN, use probability: 5% of traffic in first 30s, gradually increase to 100% over 5 minutes, as long as success rate > 95%. If success rate drops, recalculate allowed traffic percentage.

10. **Q:** How do you design circuit breakers for a legacy monolith being decomposed to microservices?
    **A:** During strangler fig migration: circuit breakers on calls between monolith modules (emulating future service boundaries). As services are extracted, circuit breakers become service-level. Gradual introduction: start with monitoring-only mode (half-open), then enable breaking.

## 13. Debugging & Troubleshooting

### Common Issues

**Issue: Circuit opens too frequently**
- Check failure threshold: too sensitive? Increase threshold.
- Check sliding window: too small? Increase window size.
- Check upstream service: is it actually unhealthy?
- Check timeouts: too short for normal latency?

**Issue: Circuit never opens despite failures**
- Check `recordExceptions` configuration.
- Check if exceptions are being caught and swallowed.
- Check `ignoreExceptions` configuration.
- Check `minimumNumberOfCalls` (not enough calls to trigger).

**Issue: Fallback not working**
- Verify fallback method signature matches original method.
- Check if fallback method is in the same class.
- Spring AOP requires the call to go through the proxy.

**Issue: Circuit breaker state not resetting**
- Check `waitDurationInOpenState`: too long?
- Check `automaticTransitionFromOpenToHalfOpenEnabled`: enabled?
- Check downstream: still failing during HALF_OPEN?

### Debugging
```bash
# Check circuit breaker health
curl http://service:8080/actuator/health | jq .components.circuitBreakers

# Check circuit breaker metrics
curl http://service:8080/actuator/metrics/resilience4j.circuitbreaker.state
curl http://service:8080/actuator/metrics/resilience4j.circuitbreaker.calls

# Enable debug logging
logging:
  level:
    io.github.resilience4j: DEBUG
```

## 14. Comparison Section

### Circuit Breaker vs Retry vs Timeout

| Pattern | Purpose | When to use |
|---------|---------|-------------|
| Circuit Breaker | Prevent calls to failing service | Ongoing failures, service down |
| Retry | Recover from transient failure | Sporadic failures, network glitches |
| Timeout/Limiter | Limit wait time | Slow service responses |
| Bulkhead | Isolate resources | Protect from resource exhaustion |

### Resilience4j vs Hystrix vs Spring Cloud Circuit Breaker

| Feature | Resilience4j | Hystrix | Spring Cloud CB |
|---------|-------------|---------|-----------------|
| Active | Yes | Deprecated | Yes (wraps Resilience4j) |
| Thread Isolation | Yes | Yes | Via Resilience4j |
| Semaphore Isolation | Yes | Yes | Via Resilience4j |
| Micrometer | Native | Requires adapter | Native |
| Functional Style | Java 8+ | Annotation-only | Both |
| Modular | Yes (separate modules) | No | Via Resilience4j |

### Client-Side vs Server-Side Circuit Breaking

| Aspect | Client-Side (Resilience4j) | Server-Side (Service Mesh) |
|--------|--------------------------|---------------------------|
| Location | Application code | Sidecar proxy |
| Configuration | Per instance | Centralized |
| Fallback logic | Application level | Limited |
| Visibility | Application metrics | Infrastructure metrics |
| Complexity | Developer responsibility | Ops responsibility |

## 15. Revision Notes

### Quick Recap
- **States**: CLOSED (normal) -> OPEN (failing fast) -> HALF_OPEN (testing) -> CLOSED or OPEN.
- **Parameters**: failure threshold, wait duration, sliding window, minimum calls.
- **Resilience4j**: Annotations `@CircuitBreaker`, `@Retry`, `@TimeLimiter`, `@Bulkhead`.
- **Fallback**: Must have same return type, same parameter types + Throwable.
- **Record vs Ignore**: Record server errors (5xx), ignore client errors (4xx).
- **Thread Pool**: Isolated thread per call. Higher overhead, complete isolation.
- **Semaphore**: Caller's thread. Lower overhead, less isolation.

### Best Practices
1. Always provide a fallback method.
2. Set appropriate failure thresholds (not too sensitive, not too tolerant).
3. Monitor circuit breaker state changes with alerts.
4. Use TimeLimiter with CircuitBreaker for slow call detection.
5. Test compensation flows as well as forward flows.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                  CIRCUIT BREAKER CHEAT SHEET                       |
+-------------------------------------------------------------------+
| STATE         | BEHAVIOR                                           |
+---------------+---------------+-----------------------------------+
| CLOSED        | Requests pass normally, failures counted           |
| OPEN          | Requests fail fast, timer starts                   |
| HALF_OPEN     | Limited test requests pass through                 |
+---------------+---------------+-----------------------------------+
| TRANSITIONS                                                        |
+-------------------------------------------------------------------+
| CLOSED -> OPEN   | Failure threshold exceeded                      |
| OPEN -> HALF_OPEN | Wait duration elapsed                          |
| HALF_OPEN -> OPEN | Test request fails                             |
| HALF_OPEN -> CLOSED | Success threshold reached (test passes)     |
+-------------------------------------------------------------------+
| RESILIENCE4J PARAMETERS                                            |
+-------------------------------------------------------------------+
| failureRateThreshold          | % of failures to open circuit       |
| slidingWindowSize             | Calls/time in window (e.g., 10)    |
| minimumNumberOfCalls          | Min calls before rate calc (5)     |
| waitDurationInOpenState       | Time in OPEN before HALF_OPEN (30s)|
| permittedNumberOfCallsInHalfOpenState | Test calls in HALF_OPEN (3)|
| automaticTransitionFromOpenToHalfOpenEnabled | Auto-recovery test   |
| recordExceptions              | Failures that count                |
| ignoreExceptions              | Failures that don't count         |
+-------------------------------------------------------------------+
| RESILIENCE4J ANNOTATIONS                                           |
+-------------------------------------------------------------------+
| @CircuitBreaker(name="svc", fallbackMethod="fallback")             |
| @Retry(name="svc", fallbackMethod="fallback")                      |
| @TimeLimiter(name="svc")                                           |
| @Bulkhead(name="svc", type=THREADPOOL)                             |
| @RateLimiter(name="svc")                                           |
+-------------------------------------------------------------------+
| COMBINED RESILIENCE PATTERN                                        |
+-------------------------------------------------------------------+
| 1. TimeLimiter (2s timeout)                                        |
| 2. Retry (3 attempts, 500ms apart)                                 |
| 3. CircuitBreaker (opens at 50% failure, 30s wait)                 |
| 4. Bulkhead (max 10 concurrent calls)                              |
| 5. Fallback (cache/default)                                        |
+-------------------------------------------------------------------+
```
