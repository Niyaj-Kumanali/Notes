# Microservices

## 1. Executive Summary

Microservices is an architectural style that structures an application as a collection of small, loosely coupled, independently deployable services, each owning its own domain logic and data store. Each service runs in its own process and communicates via lightweight mechanisms (typically HTTP/REST or messaging). Microservices enable independent scaling, deployment, and team ownership, but introduce complexity in distributed systems concerns: service discovery, inter-service communication, data consistency, observability, and deployment automation.

## 2. Core Theory

### Principles
- **Single Responsibility:** Each service owns one business capability.
- **Bounded Context:** Service boundaries align with domain-driven design bounded contexts.
- **Autonomous:** Services can be developed, deployed, and scaled independently.
- **Decentralized Governance:** Teams choose their own tech stack for their service.
- **Decentralized Data Management:** Each service manages its own database.

### Service Communication Patterns
- **Synchronous:** REST, gRPC, GraphQL - client waits for response.
- **Asynchronous:** Messaging (Kafka, RabbitMQ) - event-driven, fire-and-forget.
- **Hybrid:** Both patterns used based on use case (commands sync, events async).

### Service Decomposition Strategies
1. **By Business Capability:** User service, order service, payment service.
2. **By Subdomain:** Following DDD bounded contexts.
3. **By Entity/Resource:** User service, product service.
4. **By Change Frequency:** Separate volatile from stable services.
5. **By Team Structure:** Conway's Law - services mirror team organization.

### Key Characteristics of a Good Microservice
- **Small:** Fits in a developer's head (usually <1000 lines of business logic).
- **Independent:** Can be deployed without affecting other services.
- **Resilient:** Failure of one service doesn't cascade.
- **Owned:** One team owns the service end-to-end.
- **Stateless (at service level):** State pushed to database/cache.

## 3. Under-the-Hood Deep Dive

### Service Discovery

Services need to find each other at runtime. Two main patterns:

**Client-Side Discovery:**
```
[Service A] -> [Service Registry (Eureka/Consul)]
    |                    |
    |-- Queries registry for Service B instances
    |-- Load balances across instances
    |-- Calls Service B directly
```

**Server-Side Discovery (via API Gateway/Load Balancer):**
```
[Service A] -> [API Gateway/Load Balancer] -> [Service B instances]
```

Spring Cloud Netflix Eureka Client-Side Discovery:
```yaml
# application.yml for Eureka Client
eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    preferIpAddress: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
```

### Inter-Service Communication

**REST (Synchronous):**
```java
// Service A calling Service B via REST
@Service
public class OrderServiceClient {

    @Autowired
    private RestTemplate restTemplate;

    private static final String USER_SERVICE_URL = "http://user-service/api/users";

    public User getUser(Long userId) {
        return restTemplate.getForObject(
            USER_SERVICE_URL + "/" + userId,
            User.class
        );
    }
}
```

**Feign Client (Declarative REST):**
```java
@FeignClient(name = "user-service", url = "${services.user-service.url}")
public interface UserServiceClient {

    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable("id") Long id);

    @PostMapping("/api/users")
    User createUser(@RequestBody User user);
}
```

### API Gateway Pattern

Single entry point for all client requests. Handles:
- Request routing to appropriate services
- Authentication/authorization
- Rate limiting
- Request/response transformation
- Aggregation (composing responses from multiple services)

```java
// Spring Cloud Gateway Configuration
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .circuitBreaker(config -> config
                        .setName("userServiceCB")
                        .setFallbackUri("forward:/fallback/users")))
                .uri("lb://user-service"))
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())))
                .uri("lb://order-service"))
            .build();
    }
}
```

### Database per Service Pattern

Each microservice has its own private database, accessed only through its API.

```
[Order Service] --ownes--> [Order DB (PostgreSQL)]
[User Service]  --ownes--> [User DB (MySQL)]
[Analytics]     --ownes--> [Analytics DB (Cassandra)]
```

**Data sharing between services:**
- Service API calls (synchronous)
- Event publishing (asynchronous)
- CQRS with read-only materialized views
- API composition in API Gateway or BFF

## 4. Production Code Examples

### Spring Boot Microservice Skeleton

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

### Service Configuration (application.yml)

```yaml
spring:
  application:
    name: order-service
  config:
    import: configserver:http://config-server:8888
  datasource:
    url: jdbc:postgresql://order-db:5432/orders
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        default_schema: order_service

server:
  port: 8082

eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka:8761/eureka/
  instance:
    leaseRenewalIntervalInSeconds: 10

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true

resilience4j:
  circuitbreaker:
    instances:
      userService:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10000
        permittedNumberOfCallsInHalfOpenState: 3
  retry:
    instances:
      userService:
        maxAttempts: 3
        waitDuration: 500ms
```

### Event-Driven Communication via Kafka

```java
@Service
public class OrderEventPublisher {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    private static final String TOPIC = "order.events";

    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .orderId(order.getId())
            .userId(order.getUserId())
            .totalAmount(order.getTotalAmount())
            .timestamp(Instant.now())
            .build();

        kafkaTemplate.send(TOPIC, order.getUserId().toString(), event);
    }
}

@Service
public class OrderEventConsumer {

    @KafkaListener(topics = "payment.events", groupId = "order-service")
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        // Update order status when payment completes
        orderRepository.findById(event.getOrderId()).ifPresent(order -> {
            order.setStatus(OrderStatus.PAID);
            order.setPaymentId(event.getPaymentId());
            orderRepository.save(order);
        });
    }
}
```

### Distributed Tracing Configuration

```java
@Configuration
public class TracingConfig {

    @Bean
    public Tracer tracer() {
        return BraveTracer.create(Tracing.newBuilder()
            .localServiceName("order-service")
            .spanReporter(spanReporter())
            .build());
    }

    @Bean
    public SpanReporter spanReporter() {
        return new ZipkinSpanReporter("http://zipkin:9411/api/v2/spans");
    }
}
```

### Health Check and Readiness

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(1000)) {
                return Health.up().build();
            }
            return Health.down().withDetail("database", "unreachable").build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

## 5. Real-World Scenarios

### E-Commerce Microservices Architecture

```
[Client] -> [API Gateway]
                |
    +-----------+-----------+-----------+
    |           |           |           |
[User Svc] [Catalog Svc] [Cart Svc] [Order Svc]
    |           |           |           |
[User DB]  [Catalog DB] [Cart DB]   [Order DB]
    |                               [Payment Svc]
    +----------->[Kafka]<-----------[Inventory Svc]
                      |
            [Analytics Svc] [Notification Svc]
```

### Migration from Monolith to Microservices

**Strangler Fig Pattern:**
1. Identify boundaries (bounded contexts).
2. Create new microservice for each boundary.
3. Route new functionality to microservice.
4. Gradually migrate existing functionality.
5. Remove old code from monolith when migration complete.

```java
// Routing servlet that gradually redirects to new services
@Component
public class StranglerFigFilter extends OncePerRequestFilter {

    private static final Set<String> MIGRATED_PATHS = Set.of(
        "/api/users", "/api/products"
    );

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        String path = request.getRequestURI();
        if (MIGRATED_PATHS.stream().anyMatch(path::startsWith)) {
            // Forward to new microservice gateway
            response.sendRedirect("https://api-gateway.internal" + path);
        } else {
            // Continue to monolith
            chain.doFilter(request, response);
        }
    }
}
```

## 6. Performance

### Latency Budget Distribution

```
Client -> API Gateway (5ms) -> User Service (20ms) -> DB (10ms)
                                   |
                            Payment Service (30ms) -> DB (15ms)
                                   |
                            Notification Service (10ms) -> Kafka (5ms)
Total: ~95ms average
```

### Performance Optimization Strategies

- **Caching:** Redis for frequently accessed data (product catalog, user profiles).
- **Connection Pooling:** HikariCP configured per service based on DB capacity.
- **Asynchronous Processing:** Move non-critical work to async event handlers.
- **Database Indexing:** Covering indexes for common query patterns.
- **Read Replicas:** Separate read/write data sources.

### Inter-Service Call Optimization

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
            .setConnectTimeout(Duration.ofMillis(500))
            .setReadTimeout(Duration.ofMillis(2000))
            .requestFactory(() -> {
                HttpComponentsClientHttpRequestFactory factory =
                    new HttpComponentsClientHttpRequestFactory();
                factory.setMaxConnTotal(200);
                factory.setMaxConnPerRoute(50);
                factory.setConnectionRequestTimeout(500);
                return factory;
            })
            .build();
    }
}
```

## 7. Security

### Authentication and Authorization

**JWT-based authentication with API Gateway:**
```
[Client] -> [Login] -> [Auth Service] -> JWT
[Client] -> [Request + JWT] -> [API Gateway validates JWT]
    -> [Forward JWT to Service] -> [Service validates scopes]
```

```java
// JWT validation filter in API Gateway
@Component
public class JwtAuthenticationFilter implements GlobalFilter {

    @Autowired
    private ReactiveJwtDecoder jwtDecoder;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String authHeader = exchange.getRequest().getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing or invalid token");
        }

        String token = authHeader.substring(7);
        return jwtDecoder.decode(token)
            .flatMap(jwt -> {
                ServerWebExchange mutatedExchange = exchange.mutate()
                    .request(r -> r.header("X-User-Id", jwt.getSubject())
                                  .header("X-User-Roles", jwt.getClaimAsString("roles")))
                    .build();
                return chain.filter(mutatedExchange);
            })
            .onErrorResume(e -> unauthorized(exchange, "Token validation failed"));
    }
}
```

### Service-to-Service Authentication (mTLS)

```yaml
# application.yml for service-to-service mTLS
server:
  ssl:
    enabled: true
    client-auth: need
    key-store: classpath:service-keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    trust-store: classpath:truststore.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
```

### Secret Management
```yaml
spring:
  cloud:
    config:
      server:
        vault:
          host: vault.internal
          port: 8200
          scheme: https
          backend: secret/microservices/order-service
```

## 8. Common Mistakes

### Mistake 1: Shared Database Across Services
```java
// WRONG - multiple services access the same database
// Order service queries user database directly
@Query(value = "SELECT * FROM users.users WHERE id = ?", nativeQuery = true)

// RIGHT - services access data only via service API
User user = userServiceClient.getUser(userId);
```

### Mistake 2: Chatty Inter-Service Communication
```java
// WRONG - N+1 calls between services
for (Long productId : productIds) {
    Product p = productServiceClient.getProduct(productId); // N calls
}

// RIGHT - batch endpoint
List<Product> products = productServiceClient.getProducts(productIds); // 1 call
```

### Mistake 3: Ignoring Distributed Transactions
Don't use distributed transactions (XA) across services. Use Sagas instead.

### Mistake 4: Tight Coupling via Shared Libraries
Shared domain objects between services creates coupling. Each service should have its own DTOs.

### Mistake 5: Not Handling Partial Failures
Without circuit breakers, a failing downstream service can cascade failure to the entire system.

## 9. Senior Engineer Perspective

### Team Topology and Service Boundaries

**Conway's Law:** Organizations design systems that mirror their communication structure.

- **Stream-aligned teams:** Own a service or group of services along a business flow.
- **Enablement teams:** Help stream-aligned teams with tools, practices, infrastructure.
- **Complicated-subsystem teams:** Own complex services (search, recommendation).
- **Platform teams:** Build internal platforms (CI/CD, observability, service mesh).

### Service Granularity Guidelines

- Too fine-grained = excessive orchestration, network overhead, operational complexity.
- Too coarse = becomes a distributed monolith without benefits.
- Right size: service can be built and deployed by a team of 4-6 people in 2-week sprints.

### Operational Maturity Model

| Level | Characteristics |
|-------|----------------|
| 1. Initial | Manual deployment, no monitoring, heroic debugging |
| 2. Managed | Basic CI/CD, health checks, centralized logging |
| 3. Defined | Standardized deployment, metrics, alerting |
| 4. Measured | SLA tracking, capacity planning, chaos engineering |
| 5. Optimized | Auto-scaling, self-healing, predictive analytics |

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What are microservices?
   **A:** An architectural style where an application is built as a collection of small, independently deployable services, each responsible for a specific business capability.

2. **Q:** What is the difference between monolith and microservices?
   **A:** Monolith is a single deployable unit; microservices are multiple independent services. Monolith has simpler development but harder scaling; microservices offer independent scaling but added complexity.

3. **Q:** What is an API Gateway?
   **A:** A single entry point for client requests that routes to appropriate services, handles authentication, rate limiting, and request transformation.

4. **Q:** What is service discovery?
   **A:** The mechanism by which services find each other's network locations at runtime. Examples: Eureka, Consul, Kubernetes DNS.

5. **Q:** What is the difference between REST and gRPC in microservices?
   **A:** REST uses HTTP/1.1 + JSON (text, human-readable). gRPC uses HTTP/2 + Protobuf (binary, typed, streaming-capable, more performant).

6. **Q:** What is a distributed monolith?
   **A:** A microservices-style deployment running a monolithic codebase, or services so tightly coupled they must be deployed together, negating microservices benefits.

7. **Q:** What is the difference between client-side and server-side service discovery?
   **A:** Client-side: service queries registry and load balances itself. Server-side: load balancer or gateway handles discovery and routing.

8. **Q:** What is a bounded context in DDD?
   **A:** A logical boundary within which a particular domain model applies. Each bounded context maps naturally to a microservice boundary.

9. **Q:** What is the strangler fig pattern?
   **A:** A migration pattern that gradually replaces parts of a monolith with microservices by incrementally routing functionality to new services.

10. **Q:** What is the difference between orchestration and choreography in microservices?
    **A:** Orchestration has a central coordinator controlling the flow. Choreography has services reacting to events without a central coordinator.

### Medium

11. **Q:** How do you handle distributed transactions in microservices?
    **A:** Use the Saga pattern: a sequence of local transactions with compensating actions on failure. Can be orchestrated (central coordinator) or choreographed (event-driven).

12. **Q:** Explain the circuit breaker pattern and how it helps microservices resilience.
    **A:** Circuit breaker monitors failures to a downstream service. After threshold failures, it "opens" and fails fast without calling the service. After cooldown, it "half-opens" to test recovery.

13. **Q:** How do you ensure data consistency across microservices?
    **A:** Use eventual consistency with event-driven communication. Each service publishes events on state change; other services consume events and update their own data. Sagas for multi-step processes.

14. **Q:** What is the difference between CQRS and Event Sourcing in microservices?
    **A:** CQRS separates read and write models. Event Sourcing stores all state changes as events. They are often used together but are independent patterns.

15. **Q:** How do you handle schema changes in microservices?
    **A:** Use backward-compatible changes (add fields, don't remove), version APIs (v1, v2), parallel run old + new schemas, use event versioning for async messages.

16. **Q:** Explain blue-green deployment for microservices.
    **A:** Two identical environments (blue = live, green = staging). Deploy new version to green, test, switch traffic to green. Allows instant rollback by switching back to blue.

17. **Q:** How do you test microservices?
    **A:** Unit tests (per service), integration tests (with dependencies), contract tests (consumer-driven contracts), end-to-end tests (across services), chaos testing.

18. **Q:** What is the role of a service mesh (e.g., Istio, Linkerd)?
    **A:** A service mesh provides infrastructure layer for service-to-service communication: load balancing, service discovery, traffic management, mTLS, observability - via sidecar proxies.

19. **Q:** How do you handle logging across microservices?
    **A:** Centralized logging (ELK stack). Structured logging (JSON) with correlation IDs propagated across service boundaries for tracing.

20. **Q:** What is the difference between metrics and monitoring in microservices?
    **A:** Metrics are quantitative measurements (latency, error rate, throughput). Monitoring is collecting, visualizing, and alerting on metrics to ensure system health.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** How do you implement distributed tracing across microservices without modifying application code?
    **A:** Use auto-instrumentation via OpenTelemetry agent (java -javaagent:opentelemetry-javaagent.jar). Bytecode weaving intercepts HTTP calls, database queries, messaging. Propagates trace context via W3C TraceContext headers. No code changes needed.

2. **Q:** Design a deployment strategy for 200 microservices with zero downtime.
    **A:** Use Kubernetes with rolling updates. Canary deployments: route 1% traffic to new version, gradually increase to 100% while monitoring error rates. Blue-green for major changes. Feature flags for toggling functionality at runtime without deployment. GitOps for declarative deployments.

3. **Q:** How do you handle API versioning in microservices?
    **A:** Strategies: URL path versioning (/v1/orders, /v2/orders), header versioning (Accept: application/vnd.company.v1+json), query parameter versioning (?version=1). Best: header versioning as it follows REST principles. Maintain multiple versions until migration complete. Deprecation headers to notify clients.

4. **Q:** Explain how you would design a BFF (Backend For Frontend) pattern.
    **A:** Separate API gateway per client type (mobile BFF, web BFF, IoT BFF). Each BFF optimizes responses for its client (mobile: minimized payload, web: rich data). BFF aggregates calls to backend services. Reduces chatty frontend-to-backend communication. Client-specific authentication and rate limiting.

5. **Q:** How do you ensure exactly-once message processing in event-driven microservices?
    **A:** Idempotent consumers: store processed message IDs in DB (unique constraint) or use idempotency keys. Deduplication window in memory (Guava cache with TTL) or Redis. Kafka's exactly-once semantics with idempotent producer + transactional API.

6. **Q:** Design a service registry and discovery system that works across multiple data centers.
    **A:** Multi-region service registry: each region has a Eureka/Consul cluster. Services register with local registry. Cross-region discovery via DNS (service.internal.region1.example.com) or global registry with replication (gossip protocol). Latency-based routing to nearest region. Failure domains: registry failure in one region doesn't affect others.

7. **Q:** How do you implement global rate limiting across distributed services?
    **A:** Centralized rate limiter using Redis sorted sets (sliding window) or Redis counters. Each service instance checks with Redis before processing request. For high throughput, use local token buckets synchronized to Redis every 100ms. Hierarchical: per-user, per-service, per-API, global limits.

8. **Q:** Explain the outbox pattern for reliable event publishing.
    **A:** When a service updates its DB and needs to publish an event, it writes the event to an "outbox" table within the same DB transaction. A separate process (CDC or scheduled poll) reads the outbox and publishes events to the message broker. This ensures exactly-once event publishing without distributed transactions.

9. **Q:** How do you handle schema registry evolution in event-driven systems?
    **A:** Use Schema Registry (Confluent Schema Registry, Apicurio) with Apache Avro or Protobuf. Backward-compatible changes: add optional fields, never remove. Forward-compatible: old readers can read new data. Evolution rules: set default values, use union types. Schema versioning: each event includes schema version ID.

10. **Q:** Design a health check and self-healing system for microservices.
    **A:** Health checks: liveness (is app running), readiness (can accept traffic), and deep (can it serve requests). Self-healing: Kubernetes restarts unhealthy pods, service mesh removes from load balancing, circuit breaker opens on failure. Autoscaling based on metrics. Chaos engineering to test healing mechanisms.

### System Design

11. **Q:** Design an e-commerce platform using microservices.
    **A:** Services: User, Product, Cart, Order, Payment, Inventory, Shipping, Notification, Analytics. API Gateway + BFF for mobile/web. Event bus (Kafka) for async flows. Sagas for order processing. CQRS for read-heavy catalog. Redis for session/cart/cache. CDN for product images. Kubernetes for orchestration.

12. **Q:** Design a real-time ride-sharing application (Uber-like) with microservices.
    **A:** Services: User, Driver, Ride, Payment, Location, Matching, Pricing, Notification. WebSocket gateway for real-time location updates. Geohash-based location service (Redis GEO). Kafka Streams for ride matching. CQRS for pricing. SAGA for ride lifecycle. Cassandra for location history (time-series).

13. **Q:** Design a social media platform backend with microservices.
    **A:** Services: User, Post, Feed, Notification, Search, Analytics, Media. Fan-out-on-write for feed generation (Kafka). Redis lists per user for feed cache. Elasticsearch for search. CDN for media. GraphQL BFF for flexible client queries. Eventual consistency with event sourcing.

14. **Q:** Design a banking system with microservices (high consistency requirements).
    **A:** Services: Account, Transaction, Ledger, Fraud, Notification. Strong consistency via distributed transactions (2PC) or Saga with compensating transactions. Idempotency keys for all writes. Event sourcing for audit trail. CQRS with read replicas. Circuit breakers for fraud/notification services.

15. **Q:** Design a SaaS platform with multi-tenant microservices.
    **A:** Per-service tenant isolation: database-per-tenant (enterprise) or schema-per-tenant (standard) or shared database with tenant_id (free). Tenant context propagated via JWT claims. API Gateway routes based on tenant. Rate limiting per tenant. Tenant-specific feature flags. Separate deployment for dedicated tenants.

16. **Q:** Design a video streaming platform (YouTube-like) with microservices.
    **A:** Services: Upload, Transcode, Content, Recommendation, Search, Comment, Analytics. Async upload pipeline: upload -> transcode (FFmpeg) -> CDN distribution. Recommendation via ML service. Elasticsearch for search. Cassandra for metadata (write-heavy). Microservices for comment moderation pipeline.

17. **Q:** Design a payment processing system with microservices (high availability).
    **A:** Services: Payment, Wallet, Ledger, Fraud, Gateway, Notification. Idempotent payment processing: idempotency key prevents duplicate charges. Multi-step saga: authorize -> capture -> settle. Read replicas for balance queries (eventually consistent). Circuit breakers per payment gateway. Dead letter queue for failed payments.

18. **Q:** Design an IoT platform with microservices for millions of devices.
    **A:** Services: Device Auth, Device Profile, Telemetry, Command, Alert, Rule. MQTT gateway for device connectivity. Kafka for telemetry ingestion (high throughput). Time-series DB (InfluxDB/TimescaleDB) for sensor data. Rule engine: complex event processing. Downstream commands via Kafka + MQTT.

19. **Q:** Design an online food delivery platform with microservices.
    **A:** Services: Restaurant, Menu, Order, Cart, Payment, Delivery, Notification, Rating. Location-based restaurant search (Elasticsearch). Dynamic pricing (demand-based). Real-time order tracking via WebSocket. Delivery matching: nearest driver algorithm. Saga for order lifecycle. Rating/Review service with sentiment analysis.

20. **Q:** Design a cloud-native CI/CD platform with microservices.
    **A:** Services: Repository, Pipeline, Build, Artifact, Deploy, Environment, Config, Secret. Event-driven: git push event -> pipeline trigger -> build -> test -> deploy. Kubernetes-native: each pipeline step as a pod. Artifact registry: S3/Nexus. Blue-green/canary deployments config per service. Observability: logs, metrics, traces, all correlated by pipeline run ID.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a microservices platform for a global financial exchange requiring millisecond latency and 99.999% uptime.
    **A:** Services split by performance tier: low-latency path (C++/Java NIO, co-located on same rack via RDMA) for order matching; medium-latency (order management, risk checks); high-latency (reporting, compliance). Deterministic replay for exactly-once ordering. In-memory data grid (Aeron) for inter-service communication. Dual-site active-active with synchronous replication. Full chaos engineering program.

2. **Q:** How would you design a microservices architecture that supports 10,000 engineers working on 1,000 services?
    **A:** Standardized service template (cookie-cutter) with built-in observability, health checks, CI/CD. Internal platform team provides service mesh, API gateway, schema registry. Backstage developer portal for service catalog. Semantic API versioning with deprecation policies. Automated contract testing (PACT) for inter-service contracts. Feature flags platform. Ownership documented per service (CODEOWNERS).

3. **Q:** Design a migration strategy from a 15-year-old monolith to microservices for a SaaS platform with 99.99% uptime SLA.
    **A:** Multi-year phased approach: Phase 1 (3 months): extract auth service (minimal risk). Phase 2 (6 months): extract reporting (read-only, low risk). Phase 3 (6 months): extract billing (critical, with strangler fig). Phase 4 (ongoing): extract core domains incrementally. Parallel run: monolith and new services operate together. Feature parity validated with traffic mirroring. Data sync via CDC.

4. **Q:** How do you implement end-to-end encryption in a microservices architecture?
    **A:** mTLS between all services (service mesh). Application-level encryption for sensitive fields: encrypt at service boundary, decrypt only at destination service. Key management via HashiCorp Vault with automatic rotation. Never log encrypted data. Use envelope encryption: data key encrypts fields, master key encrypts data key. HSM for master key storage.

5. **Q:** Design a microservices debugging system for production incidents requiring root cause analysis across 100+ services.
    **A:** Unified observability: traces (OpenTelemetry), logs (structured JSON -> Loki), metrics (Prometheus). Trace-based: automatic span correlation. Logs enriched with trace ID, service name, version. Metrics with service context labels. Incident analysis: timeline view of trace spans, log events, and metric anomalies. Service dependency graph automatically generated from trace data. AI-based anomaly detection.

6. **Q:** How would you design a standards governance system for a large-scale microservices adoption?
    **A:** Architecture Decision Records (ADRs) for significant decisions. Automated compliance checks in CI/CD: REST API style, header format, error response format, logging format, metrics naming. Service templates enforcing standards. Review board for service creation. Periodic architecture reviews. Linter for API specs (OpenAPI/Swagger). Deprecation policy and migration guides.

7. **Q:** Design a multi-cloud microservices deployment strategy that avoids vendor lock-in.
    **A:** Kubernetes on both AWS and GCP (abstraction layer). Services as containers: portable across clouds. Cloud-agnostic messaging (Kafka), database (PostgreSQL via Patroni), object storage (MinIO or S3-compatible). Multi-cloud service mesh (Istio across clusters). Abstraction layers for cloud services: Terraform modules for portable infra. Chaos testing for cloud-failover scenarios.

8. **Q:** How do you design a strategy for splitting a monolith's database as part of microservices migration?
    **A:** Step 1: Identify bounded contexts in schema (ownership boundaries). Step 2: Create views for cross-context queries (no schema changes). Step 3: Extract service with write access to its tables, other services read via views. Step 4: Move extracted tables to new database. Step 5: Sync data between old and new during transition. Step 6: Remove old tables. Use Schema-on-Read pattern during migration.

9. **Q:** Design a global microservices platform that supports data sovereignty compliance (GDPR, CCPA).
    **A:** Region-locked data: user data stored and processed in region of origin. Services deployed per region with regional databases. Global services (stateless routing, frontend) route requests based on user's region. Data classification: PII tagged with access controls. Data deletion API: cascading delete across all services. Audit trail for all data access. DSR (Data Subject Request) automation service.

10. **Q:** How would you design a decision framework for when to use microservices vs monolith?
    **A:** Monolith recommended: small team (<10), early-stage product, simple domain, low scale, fast time-to-market. Microservices: large team (>50), complex domain, high scale, need for independent deployability, multiple data stores needed. Conway's Law: team structure should drive architecture choice. Start monolith, extract to microservices when boundaries become clear. Never microservices-first for startups.

## 13. Debugging & Troubleshooting

### Common Microservices Issues

**Issue: Cascading failures**
- Check circuit breaker status, bulkhead usage, timeouts.
- Verify downstream service health.
- Check for resource leaks (thread pools, connections).

**Issue: Data inconsistency between services**
- Check event ordering (Kafka partition ordering).
- Verify saga compensating transactions executed.
- Check for lost events (consumer not committing offset).
- Verify idempotency handling on consumers.

**Issue: Slow inter-service calls**
```bash
# Trace request flow
curl -H "traceparent: ..." http://api-gateway/orders/123

# Check Zipkin/Jaeger for span timings
# Check service logs for slow queries
# Check database query performance
```

**Issue: Deployment conflicts**
- Verify API version compatibility.
- Check consumer-driven contract tests.
- Verify database migrations are backward-compatible.
- Check feature flag configuration.

### Debugging Commands
```bash
# Check service health
curl http://gateway/actuator/health

# Check service registry
curl http://eureka:8761/eureka/apps

# Check circuit breaker state
curl http://service/actuator/health | jq .components.circuitBreakers

# Check distributed tracing
curl http://zipkin:9411/api/v2/traces?serviceName=order-service&limit=10

# Check Kafka consumer lag
kafka-consumer-groups --bootstrap-server kafka:9092 --group order-service --describe
```

## 14. Comparison Section

### Microservices vs Monolith

| Aspect | Microservices | Monolith |
|--------|---------------|----------|
| Deployment | Independent per service | Single deployable unit |
| Scaling | Per service | Entire application |
| Team autonomy | High | Low |
| Performance | Network overhead | In-process calls |
| Complexity | High (distributed systems) | Lower |
| Testing | Complex (contract, integration) | Simpler |
| Debugging | Distributed tracing needed | Single process |
| Data management | DB per service | Shared DB |
| Startup time | Low per service | High (one large app) |
| Rewrite risk | Low (incremental) | High (big bang) |

### Orchestration vs Choreography

| Aspect | Orchestration | Choreography |
|--------|---------------|--------------|
| Control | Central coordinator (orchestrator) | Distributed (each service reacts) |
| Coupling | Tighter (services know coordinator) | Loose (services only know events) |
| Visibility | Easy to see flow | Harder to trace |
| Failure handling | Centralized | Distributed (compensating events) |
| Complexity | Higher in coordinator | Higher in each service |
| Best for | Complex business processes | Simple event propagation |

### API Gateway vs Service Mesh

| Aspect | API Gateway | Service Mesh |
|--------|-------------|--------------|
| Layer | L7 (application) | L3/L4/L7 (network) |
| Responsibility | Auth, routing, rate limiting | Traffic, security, observability |
| Scope | East-West (external) | North-South (inter-service) |
| Deployment | Centralized | Sidecar per service pod |
| Configuration | Routes, filters | Traffic policies, mTLS |

## 15. Revision Notes

### Quick Recap
- **Single Responsibility**: One business capability per service.
- **Decentralized Data**: Each service owns its DB.
- **API Gateway**: Single entry point, handles cross-cutting concerns.
- **Service Discovery**: Eureka, Consul, or Kubernetes DNS.
- **Inter-service Communication**: REST sync, async via events/messaging.
- **Resilience**: Circuit breakers, retries, timeouts, bulkheads.
- **Observability**: Distributed tracing (OpenTelemetry), centralized logging, metrics.
- **Saga**: Local transactions + compensating actions for multi-service workflows.
- **CQRS**: Separate read/write models for scalability.
- **Strangler Fig**: Incremental migration from monolith.

### Common Pitfalls
- Distributed monolith (tight coupling across services)
- Chatty communication (N+1 calls)
- Shared database between services
- Ignoring partial failures
- No monitoring/tracing from day one
- Over-engineering (microservices for simple applications)

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                    MICROSERVICES CHEAT SHEET                       |
+-------------------------------------------------------------------+
| PATTERN             | PURPOSE                           |
+---------------------+-----------------------------------+
| API Gateway         | Single entry, auth, routing,      |
|                     | rate limiting, aggregation         |
| Service Discovery   | Find service instances at runtime |
| Circuit Breaker     | Fail fast, prevent cascade        |
| Saga                | Distributed transaction via       |
|                     | local transactions + compensation  |
| CQRS                | Separate read/write models        |
| Event Sourcing      | Store events, not state           |
| Strangler Fig       | Incremental monolith migration    |
| Outbox              | Reliable event publishing         |
| BFF                 | Optimized backend per client type |
+---------------------+-----------------------------------+
| SPRING BOOT / CLOUD ANNOTATIONS                                |
+----------------------------------------------------------------+
| @SpringBootApplication      | Main entry point                 |
| @EnableDiscoveryClient      | Service registry integration     |
| @EnableFeignClients         | Declarative REST clients         |
| @FeignClient(name="svc")    | REST client definition           |
| @EnableCircuitBreaker       | Circuit breaker support          |
| @EnableConfigServer         | External configuration           |
| @EnableEurekaServer         | Service registry server          |
| @EnableZuulProxy / Gateway  | API Gateway                      |
+----------------------------------------------------------------+
| OBSERVABILITY                                                   |
+----------------------------------------------------------------+
| Tracing: OpenTelemetry + Zipkin/Jaeger                          |
| Logging: Structured JSON + ELK/Loki                             |
| Metrics: Micrometer + Prometheus + Grafana                      |
| Health: /actuator/health (liveness, readiness, deep)            |
+----------------------------------------------------------------+
| DEPLOYMENT PATTERNS                                             |
+----------------------------------------------------------------+
| Blue-Green   | Full env switch, instant rollback                |
| Canary       | Gradual % traffic shift, metrics-based           |
| Rolling      | Sequential pod replacement                       |
| Feature Flag | Toggle features without deploy                   |
+----------------------------------------------------------------+
```
