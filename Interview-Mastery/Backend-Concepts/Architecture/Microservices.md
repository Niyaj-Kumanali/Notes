# Microservices

---

## Overview

- **Definition:** An architectural style that structures an application as a collection of small, loosely coupled, independently deployable services, each owning its own domain logic and data store.
- **Why It Exists:** Monoliths become difficult to scale, deploy, and maintain as they grow. Microservices enable independent scaling, deployment, and team ownership, allowing organizations to develop and deliver features faster and more reliably.
- **Key Concepts:** **API Gateway** (single entry point for routing, auth, rate limiting), **Service Discovery** (Eureka, Consul, Kubernetes DNS), **Database per Service** (each service owns its data), **Synchronous Communication** (REST, gRPC), **Asynchronous Communication** (Kafka, RabbitMQ), **Circuit Breaker** (resilience), **Saga** (distributed transactions), **CQRS** (separate read/write models), **Strangler Fig** (incremental migration), **BFF** (Backend for Frontend)

---

## Core Concepts

- **Service Decomposition Strategies:** By business capability (User, Order, Payment), by DDD bounded context, by change frequency (separate volatile from stable), by team structure (Conway's Law — services mirror team organization).
- **Service Discovery:** Client-side discovery — service queries a registry (Eureka) and load balances itself. Server-side discovery — API Gateway or load balancer handles routing. Kubernetes uses DNS-based service discovery.
- **API Gateway Pattern:** Single entry point for all client requests handling routing, authentication/authorization, rate limiting, request/response transformation, and aggregation of responses from multiple services.
- **Database per Service:** Each microservice owns its private database, accessed only through its API. Data sharing via service API calls (sync), event publishing (async), CQRS with materialized views, or API composition in the gateway.
- **Communication Patterns:** REST/gRPC for synchronous request-response where the caller needs an immediate answer. Messaging (Kafka, RabbitMQ) for asynchronous event-driven communication where eventual consistency is acceptable.

```java
// Feign Client — declarative REST
@FeignClient(name = "user-service")
public interface UserServiceClient {
    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable("id") Long id);
}

// Spring Cloud Gateway Configuration
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("user-service", r -> r.path("/api/users/**")
            .filters(f -> f.circuitBreaker(config -> config.setName("userServiceCB")))
            .uri("lb://user-service"))
        .route("order-service", r -> r.path("/api/orders/**")
            .filters(f -> f.requestRateLimiter(config -> config.setRateLimiter(redisRateLimiter())))
            .uri("lb://order-service"))
        .build();
}

// Distributed Tracing with OpenTelemetry
@Bean
public Tracer tracer() {
    return BraveTracer.create(Tracing.newBuilder()
        .localServiceName("order-service")
        .spanReporter(spanReporter())
        .build());
}
```

---

## Common Mistakes

- **Shared Database Across Services** — multiple services accessing the same database creates tight coupling. Services should access data only via service APIs.
- **Chatty Inter-Service Communication** — N+1 calls between services (calling getProduct for each productId in a loop). Use batch endpoints instead.
- **Ignoring Distributed Transactions** — using distributed transactions (XA) across services hurts scalability. Use Sagas with compensating actions.
- **Tight Coupling via Shared Libraries** — sharing domain objects between services creates coupling. Each service should have its own DTOs.
- **Not Handling Partial Failures** — without circuit breakers, a failing downstream service cascades failure to the entire system.

---

## Key Design Considerations

- **Service Granularity** — too fine-grained causes excessive orchestration and network overhead; too coarse creates a distributed monolith. Right size: a team of 4-6 people can build and deploy in 2-week sprints.
- **Observability** — distributed tracing (OpenTelemetry + Zipkin/Jaeger) for request flow, structured JSON logging (ELK/Loki) for debugging, metrics (Micrometer + Prometheus + Grafana) for monitoring, health checks (liveness, readiness, deep) for Kubernetes.
- **Resilience Patterns** — Circuit Breaker (fail fast), Retry (transient recovery), Timeout (limit wait time), Bulkhead (resource isolation), Saga (distributed transactions), CQRS (read/write separation). Layer these patterns for comprehensive protection.
- **Deployment Strategies** — Blue-Green (full environment switch, instant rollback), Canary (gradual traffic shift, metrics-based), Rolling (sequential pod replacement), Feature Flags (toggle features without deployment).
- **API Versioning** — URL path versioning (`/v1/orders`, `/v2/orders`), header versioning (`Accept: application/vnd.company.v1+json`), or query parameter versioning. Header versioning follows REST principles best. Maintain multiple versions until migration completes.

---

## Real-World Scenarios

### Scenario 1: Monolith to Microservices Migration
**Context:** A growing e-commerce company has a monolithic application handling catalog, orders, payments, shipping, and user management. Deployments take 4 hours, scaling requires the entire app, and a bug in the catalog search can crash the payment system.

**Resolution:** Apply the strangler fig pattern. Identify the order processing flow as the first independent service. Extract order management into a separate service with its own database. Route all `/api/orders/*` requests to the new service via an API Gateway. Keep the monolith serving other endpoints. Gradually extract payment, then shipping, then catalog search. Each extraction adds 20% deployment speed improvement. After 12 months, the monolith is reduced to a legacy read-only system.

```java
// API Gateway routing — routes to new services while strangling monolith
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("orders", r -> r.path("/api/v2/orders/**")
            .uri("lb://order-service"))
        .route("payments", r -> r.path("/api/v2/payments/**")
            .uri("lb://payment-service"))
        .route("monolith", r -> r.path("/api/**")
            .uri("lb://legacy-monolith"))
        .build();
}
```

### Scenario 2: Database per Service with Shared Data Problem
**Context:** An order service needs customer data (name, address) and product data (name, price) to process an order. But customer data is owned by the Customer service, and product data by the Catalog service.

**Resolution:** The order service stores only the data it needs right when processing — `customerId`, `productId`, `quantity`, `price` (price recorded at time of order). It queries the Customer service for shipping address synchronously during checkout, but doesn't store the full customer profile. It publishes `OrderPlaced` events for downstream analytics. This follows the database-per-service pattern while maintaining service autonomy.

### Scenario 3: Circuit Breaker Cascade Prevention
**Context:** During a flash sale, the payment service slows down (500ms → 10s response). The order service waits for payment responses, exhausting its thread pool. The API Gateway waiting for orders exhausts its connections. The entire site becomes unavailable.

**Resolution:** Implement resilience patterns at each layer. The order service wraps the payment call with a circuit breaker (50% failure rate, open after 10 failures, 30s wait). The API Gateway has per-service rate limits and circuit breakers. When the payment circuit opens, the order service returns "Payment pending" instead of blocking. Other services (catalog, search) remain fully functional.

---

## Scenario-Based Questions

1. **Q: You're migrating a 1M-line monolith to microservices. The CEO wants it done in 3 months. How do you de-risk this?**
   - A: Use the strangler fig pattern — extract one service at a time, starting with the highest-change-frequency module (typically payments or orders). Each extraction takes 2-4 weeks. Don't rewrite — extract with the same technology first, then optimize. The goal is independent deployability, not technology change. Plan 12+ months for a full migration. The first extraction provides the most value (CI/CD independence) and the most learning.

2. **Q: Your order service and payment service both need customer data. If each service has its own database, how does the order service get the customer's shipping address?**
   - A: During checkout, the order service synchronously calls the customer service to get the shipping address and caches it in its own database (for the order record). It stores only the fields needed for the order — not the full customer profile. The customer service publishes `CustomerAddressUpdated` events. The order service consumes these to update shipping addresses for in-progress orders. The catalog of record for customer data remains the customer service.

3. **Q: Your team of 8 is transitioning to microservices. How do you organize the teams to prevent Conway's Law from creating coordination nightmares?**
   - A: Conway's Law states that system architecture mirrors communication structure. Start with 2-3 services aligned with your existing team boundaries. Don't create 10 services for 8 developers — you'll spend all your time on cross-service coordination. Aim for one service per 2-4 developers. Use shared ownership for common infrastructure (API Gateway, monitoring, CI/CD). Mature the team structure alongside the architecture.

4. **Q: A critical bug is found in the shared library used by 5 microservices. Each service needs to be rebuilt and deployed. How do you handle this without coupling deployments?**
   - A: This is a sign that shared libraries create coupling. Refactor to reduce shared code — duplicate small utilities if necessary (cost of duplication < cost of coordinated deployments). For truly shared logic, consider extracting it into a service (e.g., a shared authorization library becomes an auth service). Alternatively, use semantic versioning and let each service upgrade at its own pace with proper testing.

5. **Q: Your microservices are deployed, but you can't debug a request that fails across 3 services. What observability infrastructure do you need?**
   - A: Three pillars: (1) Distributed tracing — propagate a trace ID across all services via HTTP headers (W3C `traceparent`). OpenTelemetry auto-instruments most frameworks. (2) Centralized logging — structured JSON logs with `trace_id`, `service`, `span_id` shipped to ELK/Loki. (3) Metrics — RED metrics (Rate, Errors, Duration) per service exported to Prometheus/Grafana. Dashboards showing service dependency health. Alerts on error rate spikes and latency degradation.

6. **Q: Your payment service depends on the order service. The payment team wants to deploy independently but tests break because they need the order service API. How do you enable independent deployments?**
   - A: Contract testing. The payment service defines its expectations of the order service API (request format, response shape, error codes) using Pact or Spring Cloud Contract. The order service runs these contract tests in its CI pipeline. As long as contracts pass, both teams deploy independently. Integration tests run against deployed environments, not during build. This decouples deployment schedules.

7. **Q: Your database-per-service approach means each service has its own PostgreSQL instance. The ops cost is growing 10x. How do you manage this without reverting to a shared database?**
   - A: Use shared PostgreSQL clusters with separate schemas or databases per service. Each service owns its schema and connects only to its own database. This reduces operational overhead (backup, monitoring, upgrades) while maintaining logical separation. Use connection pooling (PgBouncer) to manage connections. For extreme scale services (orders), consider dedicated instances.

8. **Q: A startup uses microservices from day one with 5 engineers. After 6 months, they're struggling with deployment complexity, debugging, and developer productivity. What went wrong?**
   - A: Premature microservices. Startups should start with a modular monolith — clear package boundaries, well-defined APIs, separate database schemas — but a single deployable unit. This provides development speed without distributed systems complexity. Extract to microservices only when: the team can't deploy independently, scaling needs diverge, or the codebase exceeds Conway's Law boundaries.

9. **Q: Your API Gateway is becoming a bottleneck — every request goes through it, and it's handling auth, rate limiting, routing, and request transformation. How do you scale it?**
   - A: Split the gateway into layers. A L7 reverse proxy (NGINX/Envoy) handles SSL termination, basic routing, and rate limiting at the network layer. A lightweight gateway (Spring Cloud Gateway/Kong) handles auth token validation and routing. Heavy transformations move to BFF (Backend for Frontend) services specific to each client type (web, mobile, partner API). This distributes the load and prevents a single gateway from being the bottleneck.

10. **Q: Two microservices developed by different teams need to share a transaction — when an order is created, inventory must be reserved atomically. How do you handle this without distributed transactions?**
    - A: Use the Saga pattern. Orchestrator approach: an Order Saga sends "Reserve Inventory" command to Inventory service. If successful, the saga continues to "Process Payment". If inventory reservation fails, the saga triggers "Cancel Order". Each step is a local ACID transaction within its service. The saga ensures eventual consistency without distributed locks. Monitor saga failures and implement compensating actions for each step.

---

## Interview Questions

1. **What is the difference between a monolith and microservices?**
   - A: Monolith is a single deployable unit with shared database and codebase. Microservices are independently deployable services with their own data stores. Monoliths offer simplicity and fast development initially; microservices provide independent scaling, deployment autonomy, and organizational alignment at the cost of distributed systems complexity.

2. **What is an API Gateway?**
   - A: A single entry point for all client requests that handles routing to appropriate services, authentication and authorization, rate limiting, request/response transformation, and API composition. Examples: Spring Cloud Gateway, Kong, AWS API Gateway.

3. **What is the difference between orchestration and choreography?**
   - A: Orchestration uses a central coordinator (orchestrator) that manages the workflow — sends commands to services, tracks state, handles failures. Choreography is decentralized — services react to events published by other services without a central coordinator.

4. **How do you handle distributed transactions in microservices?**
   - A: Use the Saga pattern — a sequence of local transactions with compensating actions on failure. Orchestrated (central coordinator) for complex workflows. Choreographed (event-driven) for simpler flows. Never use distributed transactions (2PC/XA) across microservices.

5. **What is the strangler fig pattern?**
   - A: A migration pattern that gradually replaces a monolith by routing functionality to new microservices incrementally. The monolith is "strangled" over time until it can be decommissioned. Each step is independently deployable and revertable.

6. **What is a service mesh?**
   - A: An infrastructure layer (Istio, Linkerd) that handles service-to-service communication via sidecar proxies: load balancing, service discovery, traffic management, mTLS, circuit breaking, and observability — without modifying application code.

7. **How do you handle logging across microservices?**
   - A: Centralized logging with structured JSON format. Each log entry includes `service`, `traceId`, `spanId`, `level`, and `timestamp`. All logs ship to a central platform (ELK, Loki, Datadog). Trace IDs propagate across service boundaries for correlation.

8. **How do you ensure data consistency across microservices?**
   - A: Eventual consistency via event-driven communication. Each service owns its data and publishes events on state changes. Other services consume events and update their own data. The Saga pattern handles multi-step workflows with compensating actions.

9. **Explain blue-green deployment.**
   - A: Two identical environments (blue = live, green = staging). Deploy new version to green, run tests, then switch all traffic to green (router/load balancer update). Instant rollback by switching back to blue. Zero-downtime deployments.

10. **What is a distributed monolith?**
    - A: A system deployed as separate services that are so tightly coupled they must be deployed together, share a database, or have chatty synchronous dependencies. It combines the worst of both worlds: complexity of distributed systems without the benefits of independent deployability.

---

## Developer Recommendations

- **Start with a modular monolith, extract services only when needed** — Premature microservices add immense complexity (network, data consistency, observability, deployment). Start with clear package boundaries, separate database schemas per module, and well-defined API contracts within the monolith. Extract a service when: (a) the team can't deploy independently, (b) a module needs different scaling, or (c) the codebase exceeds 300K lines. Most applications never need microservices.

- **Never share databases between services** — A shared database creates tight coupling — a schema change in one service can break another. Each service must own its data and expose it only via its API. If another service needs that data, it calls the API or consumes events. The only exception is read replicas for reporting, which should be treated as an internal implementation detail of the owning service.

- **Use the API Gateway pattern but keep it thin** — A gateway that handles routing, auth, rate limiting, protocol translation, request transformation, and API composition becomes a bottleneck. Keep the gateway focused on cross-cutting concerns (auth, routing, rate limiting). Move heavy logic to BFF services per client type. Monitor gateway latency — it should add <5ms overhead.

- **Implement observability from day one** — Debugging a distributed system without tracing, centralized logging, and metrics is impossible. Install OpenTelemetry auto-instrumentation in every service. Standardize on structured JSON logging with trace IDs. Export metrics to Prometheus/Grafana. Set up dashboards before the first production deployment, not after the first outage.

- **Use circuit breakers, retries, and timeouts on every inter-service call** — Every synchronous call between services can fail. Layer resilience: timeout (cap wait time), retry (3 attempts with backoff), circuit breaker (stop calling failing services), bulkhead (isolate resources per service). Without these, a single slow service cascades failures across the entire system.

- **Design for failure, not for success** — Assume every service call will fail, every message will be delayed, and every database will go down. Design accordingly: graceful degradation (return cached data when a service is down), fallbacks (return defaults for non-critical data), async processing where possible (queue messages instead of blocking), and health check endpoints for every service.
