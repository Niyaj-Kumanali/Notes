# Microservices

---

## Overview

- **Definition:** An architectural style that structures an application as a collection of small, loosely coupled, independently deployable services, each owning its own domain logic and data store.
- **Why It Exists:** Monoliths become difficult to scale, deploy, and maintain as they grow. Microservices enable independent scaling, deployment, and team ownership, allowing organizations to develop and deliver features faster and more reliably.
- **Key Concepts:** **API Gateway** (single entry point for routing, auth, rate limiting), **Service Discovery** (Eureka, Consul, Kubernetes DNS), **Database per Service** (each service owns its data), **Synchronous Communication** (REST, gRPC), **Asynchronous Communication** (Kafka, RabbitMQ), **Circuit Breaker** (resilience), **Saga** (distributed transactions), **CQRS** (separate read/write models), **Strangler Fig** (incremental migration), **BFF** (Backend for Frontend)
- **Modular Monolith as a Starting Point** — Before splitting into microservices, organize code as a modular monolith: clear package boundaries, separate database schemas per module, well-defined internal APIs, and independent testing per module. This provides development speed without distributed systems complexity. Extract to microservices only when you hit specific scaling or team autonomy bottlenecks.
- **BFF (Backend for Frontend) Pattern** — Create separate API surfaces for web, mobile, and third-party clients. Each BFF is owned by the corresponding frontend team and handles client-specific concerns: data aggregation, response shaping, and device-specific logic. Without BFF, the API Gateway becomes bloated with client-specific transformations.

---

## Core Concepts

- **Service Decomposition Strategies:** By business capability (User, Order, Payment), by DDD bounded context, by change frequency (separate volatile from stable), by team structure (Conway's Law — services mirror team organization).
- **Service Discovery:** Client-side discovery — service queries a registry (Eureka) and load balances itself. Server-side discovery — API Gateway or load balancer handles routing. Kubernetes uses DNS-based service discovery.
- **API Gateway Pattern:** Single entry point for all client requests handling routing, authentication/authorization, rate limiting, request/response transformation, and aggregation of responses from multiple services.
- **Database per Service:** Each microservice owns its private database, accessed only through its API. Data sharing via service API calls (sync), event publishing (async), CQRS with materialized views, or API composition in the gateway.
- **Communication Patterns:** REST/gRPC for synchronous request-response where the caller needs an immediate answer. Messaging (Kafka, RabbitMQ) for asynchronous event-driven communication where eventual consistency is acceptable.
- **Service Mesh** — An infrastructure layer (Istio, Linkerd) that handles service-to-service communication via sidecar proxies: load balancing, service discovery, traffic management, mTLS, circuit breaking, and observability — without modifying application code. The control plane manages proxy configuration; the data plane (proxies) handles actual traffic.
- **Containerization and Orchestration** — Microservices are typically deployed in containers (Docker) and orchestrated by Kubernetes. Kubernetes provides service discovery (DNS), load balancing, auto-scaling, rolling updates, self-healing, and secret management. Each service is deployed as a Deployment with a ClusterIP Service for internal communication.

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
  - **Why it looks correct:** sharing a database is the simplest way to share data, and it avoids the latency and complexity of service-to-service calls — the coupling only becomes painful when one service's schema migration breaks another.
- **Chatty Inter-Service Communication** — N+1 calls between services (calling getProduct for each productId in a loop). Use batch endpoints instead.
  - **Why it looks correct:** each individual call works correctly and the pattern mirrors in-process database queries — the N+1 overhead only becomes visible when latency adds up across hundreds of calls.
- **Ignoring Distributed Transactions** — using distributed transactions (XA) across services hurts scalability. Use Sagas with compensating actions.
  - **Why it looks correct:** ACID transactions are the standard in monolithic databases, and extending that guarantee across services seems like natural evolution — the scalability ceiling of 2PC only becomes apparent under load.
- **Tight Coupling via Shared Libraries** — sharing domain objects between services creates coupling. Each service should have its own DTOs.
  - **Why it looks correct:** code reuse is a fundamental good practice, and sharing a common library reduces duplication — the coupling only manifests when a change in the shared library forces coordinated deployments across all consuming services.
- **Not Handling Partial Failures** — without circuit breakers, a failing downstream service cascades failure to the entire system.
  - **Why it looks correct:** in a monolith, one component failing just crashes that request — in a distributed system, blocked threads waiting for a timeout exhaust shared resources, taking down unrelated services too.
- **Synchronous Dependency Chains** — Service A calls B, B calls C, C calls D. If D slows down, all upstream services block. Replace synchronous chains with async messaging or at least implement timeouts and circuit breakers at each hop.
  - **Why it looks correct:** each service only calls the next one, and the chain seems like a natural flow — the cascading failure only surfaces when D's slowdown causes A, B, and C to exhaust their thread pools simultaneously.
- **Premature Extraction** — Extracting services before the boundaries are well-understood leads to frequent, costly rearchitecting. Start monolithic; extract when the module has stable interfaces and clear ownership.
  - **Why it looks correct:** microservices are the modern best practice, and extracting early seems proactive — the cost of rearchitecting poorly chosen boundaries only becomes clear when you're splitting services that should never have been separated.
- **Not Automating Deployment Pipeline** — Without CI/CD per service, microservices add deployment complexity without benefits. Each service must have: automated build, automated tests, container image, deployment pipeline, and rollback capability.
  - **Why it looks correct:** manually deploying a few services seems manageable for a small team, and setting up CI/CD per service is significant upfront work — the risk of human error multiplies with each service and each deployment.

---

## Key Design Considerations

- **Service Granularity** — too fine-grained causes excessive orchestration and network overhead; too coarse creates a distributed monolith. Right size: a team of 4-6 people can build and deploy in 2-week sprints.
- **Observability** — distributed tracing (OpenTelemetry + Zipkin/Jaeger) for request flow, structured JSON logging (ELK/Loki) for debugging, metrics (Micrometer + Prometheus + Grafana) for monitoring, health checks (liveness, readiness, deep) for Kubernetes.
- **Resilience Patterns** — Circuit Breaker (fail fast), Retry (transient recovery), Timeout (limit wait time), Bulkhead (resource isolation), Saga (distributed transactions), CQRS (read/write separation). Layer these patterns for comprehensive protection.
- **Deployment Strategies** — Blue-Green (full environment switch, instant rollback), Canary (gradual traffic shift, metrics-based), Rolling (sequential pod replacement), Feature Flags (toggle features without deployment).
- **API Versioning** — URL path versioning (`/v1/orders`, `/v2/orders`), header versioning (`Accept: application/vnd.company.v1+json`), or query parameter versioning. Header versioning follows REST principles best. Maintain multiple versions until migration completes.
- **Data Consistency Strategies** — Eventual consistency via events is the default. For critical read-your-write scenarios, the writing service can expose a read API that its own service reads from (bypassing eventual consistency). For transactions spanning services, use the Saga pattern with compensating actions.
- **Monitoring and Alerting Per Service** — Each service exports: health check endpoints (liveness + readiness), RED metrics (Rate, Errors, Duration), and business metrics. Centralized dashboards per service. Alerts on: error rate spikes, latency degradation, consumer lag, and health check failures.

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

- **Q: You're migrating a 1M-line monolith to microservices. The CEO wants it done in 3 months. How do you de-risk this?**
  - Use the strangler fig pattern — extract one service at a time, starting with the highest-change-frequency module (typically payments or orders).
  - Each extraction takes 2-4 weeks. Don't rewrite — extract with the same technology first, then optimize. The goal is independent deployability, not technology change.
  - Plan 12+ months for a full migration. The first extraction provides the most value (CI/CD independence) and the most learning.
  - **Interview follow-up:** You extract the payment module as the first service, but now every order creation needs to make a synchronous call to the new payment service — how do you handle the increased latency and failure risk compared to the in-process call it replaced?

- **Q: Your order service and payment service both need customer data. If each service has its own database, how does the order service get the customer's shipping address?**
  - During checkout, the order service synchronously calls the customer service to get the shipping address and caches it in its own database (for the order record). It stores only the fields needed for the order — not the full customer profile.
  - The customer service publishes `CustomerAddressUpdated` events. The order service consumes these to update shipping addresses for in-progress orders. The catalog of record for customer data remains the customer service.
  - **Interview follow-up:** The cached shipping address in the order service is stale — the customer updated their address after placing the order, but the order was already shipped to the old address. How would you design cache invalidation for transient order-time data that the business considers authoritative only at the moment of processing?

- **Q: Your team of 8 is transitioning to microservices. How do you organize the teams to prevent Conway's Law from creating coordination nightmares?**
  - Conway's Law states that system architecture mirrors communication structure. Start with 2-3 services aligned with your existing team boundaries.
  - Don't create 10 services for 8 developers — you'll spend all your time on cross-service coordination. Aim for one service per 2-4 developers.
  - Use shared ownership for common infrastructure (API Gateway, monitoring, CI/CD). Mature the team structure alongside the architecture.
  - **Interview follow-up:** Your 2-3 services each have different deployment cadences — one deploys 5 times a day, another deploys once a month. How do you handle API compatibility when the fast-moving service changes a contract the slow-moving service depends on?

- **Q: A critical bug is found in the shared library used by 5 microservices. Each service needs to be rebuilt and deployed. How do you handle this without coupling deployments?**
  - This is a sign that shared libraries create coupling. Refactor to reduce shared code — duplicate small utilities if necessary (cost of duplication < cost of coordinated deployments).
  - For truly shared logic, consider extracting it into a service (e.g., a shared authorization library becomes an auth service). Alternatively, use semantic versioning and let each service upgrade at its own pace with proper testing.
  - **Interview follow-up:** A critical security vulnerability is found in the shared authentication library — waiting for the slow-moving service's upgrade cycle leaves a security hole open for a month. How do you reconcile independent upgrade pace with security patching?

- **Q: Your microservices are deployed, but you can't debug a request that fails across 3 services. What observability infrastructure do you need?**
  - Three pillars: (1) Distributed tracing — propagate a trace ID across all services via HTTP headers (W3C `traceparent`). OpenTelemetry auto-instruments most frameworks.
  - (2) Centralized logging — structured JSON logs with `trace_id`, `service`, `span_id` shipped to ELK/Loki.
  - (3) Metrics — RED metrics (Rate, Errors, Duration) per service exported to Prometheus/Grafana. Dashboards showing service dependency health. Alerts on error rate spikes and latency degradation.
  - **Interview follow-up:** You have distributed tracing, but one service sends its traces to a different backend and the trace IDs don't propagate correctly — how do you enforce consistent observability across services owned by different teams?

- **Q: Your payment service depends on the order service. The payment team wants to deploy independently but tests break because they need the order service API. How do you enable independent deployments?**
  - Contract testing. The payment service defines its expectations of the order service API (request format, response shape, error codes) using Pact or Spring Cloud Contract.
  - The order service runs these contract tests in its CI pipeline. As long as contracts pass, both teams deploy independently.
  - Integration tests run against deployed environments, not during build. This decouples deployment schedules.
  - **Interview follow-up:** The contract tests pass, but integration tests fail in staging — the order service returns a response that matches the schema but with subtly different semantics, like a 204 No Content instead of a 200 OK with an empty body. How do you catch semantic mismatches that schema-based contract tests miss?

- **Q: Your database-per-service approach means each service has its own PostgreSQL instance. The ops cost is growing 10x. How do you manage this without reverting to a shared database?**
  - Use shared PostgreSQL clusters with separate schemas or databases per service. Each service owns its schema and connects only to its own database.
  - This reduces operational overhead (backup, monitoring, upgrades) while maintaining logical separation. Use connection pooling (PgBouncer) to manage connections.
  - For extreme scale services (orders), consider dedicated instances.

- **Q: A startup uses microservices from day one with 5 engineers. After 6 months, they're struggling with deployment complexity, debugging, and developer productivity. What went wrong?**
  - Premature microservices. Startups should start with a modular monolith — clear package boundaries, well-defined APIs, separate database schemas — but a single deployable unit.
  - This provides development speed without distributed systems complexity. Extract to microservices only when: the team can't deploy independently, scaling needs diverge, or the codebase exceeds Conway's Law boundaries.

- **Q: Your API Gateway is becoming a bottleneck — every request goes through it, and it's handling auth, rate limiting, routing, and request transformation. How do you scale it?**
  - Split the gateway into layers. A L7 reverse proxy (NGINX/Envoy) handles SSL termination, basic routing, and rate limiting at the network layer.
  - A lightweight gateway (Spring Cloud Gateway/Kong) handles auth token validation and routing. Heavy transformations move to BFF (Backend for Frontend) services specific to each client type (web, mobile, partner API).
  - This distributes the load and prevents a single gateway from being the bottleneck.

- **Q: Two microservices developed by different teams need to share a transaction — when an order is created, inventory must be reserved atomically. How do you handle this without distributed transactions?**
  - Use the Saga pattern. Orchestrator approach: an Order Saga sends "Reserve Inventory" command to Inventory service. If successful, the saga continues to "Process Payment".
  - If inventory reservation fails, the saga triggers "Cancel Order". Each step is a local ACID transaction within its service.
  - The saga ensures eventual consistency without distributed locks. Monitor saga failures and implement compensating actions for each step.

---

## Interview Questions

- **What is the difference between a monolith and microservices?**
  - Monolith is a single deployable unit with shared database and codebase. Microservices are independently deployable services with their own data stores.
  - Monoliths offer simplicity and fast development initially; microservices provide independent scaling, deployment autonomy, and organizational alignment at the cost of distributed systems complexity.

- **What is an API Gateway?**
  - A single entry point for all client requests that handles routing to appropriate services, authentication and authorization, rate limiting, request/response transformation, and API composition.
  - Examples: Spring Cloud Gateway, Kong, AWS API Gateway.

- **What is the difference between orchestration and choreography?**
  - Orchestration uses a central coordinator (orchestrator) that manages the workflow — sends commands to services, tracks state, handles failures.
  - Choreography is decentralized — services react to events published by other services without a central coordinator.

- **How do you handle distributed transactions in microservices?**
  - Use the Saga pattern — a sequence of local transactions with compensating actions on failure.
  - Orchestrated (central coordinator) for complex workflows. Choreographed (event-driven) for simpler flows. Never use distributed transactions (2PC/XA) across microservices.

- **What is the strangler fig pattern?**
  - A migration pattern that gradually replaces a monolith by routing functionality to new microservices incrementally.
  - The monolith is "strangled" over time until it can be decommissioned. Each step is independently deployable and revertable.

- **What is a service mesh?**
  - An infrastructure layer (Istio, Linkerd) that handles service-to-service communication via sidecar proxies: load balancing, service discovery, traffic management, mTLS, circuit breaking, and observability — without modifying application code.

- **How do you handle logging across microservices?**
  - Centralized logging with structured JSON format. Each log entry includes `service`, `traceId`, `spanId`, `level`, and `timestamp`.
  - All logs ship to a central platform (ELK, Loki, Datadog). Trace IDs propagate across service boundaries for correlation.

- **How do you ensure data consistency across microservices?**
  - Eventual consistency via event-driven communication. Each service owns its data and publishes events on state changes.
  - Other services consume events and update their own data. The Saga pattern handles multi-step workflows with compensating actions.

- **Explain blue-green deployment.**
  - Two identical environments (blue = live, green = staging). Deploy new version to green, run tests, then switch all traffic to green (router/load balancer update).
  - Instant rollback by switching back to blue. Zero-downtime deployments.

- **What is a distributed monolith?**
  - A system deployed as separate services that are so tightly coupled they must be deployed together, share a database, or have chatty synchronous dependencies.
  - It combines the worst of both worlds: complexity of distributed systems without the benefits of independent deployability.

- **What is the Circuit Breaker pattern?**
  - A resilience pattern that monitors for failures and prevents calls to a failing service. When failures exceed a threshold, the circuit "opens" and subsequent calls fail fast without attempting the call.
  - States: Closed (normal), Open (fail fast), Half-Open (test recovery). Implemented via libraries like Resilience4j, Hystrix.

- **How do you handle logging and monitoring in a microservices architecture?**
  - Centralized logging with structured JSON format (trace ID, service name, span ID) shipped to ELK/Loki. Metrics exported via Micrometer to Prometheus/Grafana.
  - Distributed tracing with OpenTelemetry propagates a trace context across service boundaries, enabling end-to-end request debugging.

- **Explain the Saga pattern.**
  - A sequence of local transactions where each step publishes an event or invokes the next step. If a step fails, the saga executes compensating transactions to undo the previous steps.
  - Two implementations: choreography (each service publishes events that trigger the next step) and orchestration (a central coordinator manages the workflow).

- **What is CQRS and when would you use it in microservices?**
  - Command Query Responsibility Segregation separates read and write models. Commands handle mutations; queries handle reads, potentially using a different data store or schema optimized for queries.
  - Use CQRS when read and write workloads have different performance requirements, or when you need to maintain separate read-optimized views (e.g., materialized views for reporting).

- **How do you manage configuration across multiple microservices?**
  - Externalized configuration using a centralized config server (Spring Cloud Config, Consul KV, etc.) or Kubernetes ConfigMaps/Secrets.
  - Configuration is versioned, environment-specific (dev/staging/prod), and can be refreshed at runtime without redeploying the service.

- **What are the challenges of testing microservices?**
  - Service dependencies require running multiple services for integration tests. Contract testing (Pact) and consumer-driven contracts help decouple test schedules.
  - Testing strategies: unit tests (fast, isolated), contract tests (API compatibility), integration tests (deployed environment), end-to-end tests (critical flows only, slow and brittle).

- **How do you handle service versioning in microservices?**
  - URL path versioning (`/v1/orders`, `/v2/orders`), request header versioning (`Accept: application/vnd.company.v1+json`), or query parameter versioning.
  - Maintain backward compatibility for a defined deprecation period. Use a gateway or routing layer to direct clients to the correct version.

- **What is the difference between REST and gRPC for inter-service communication?**
  - REST uses HTTP/1.1 with text-based JSON, easy to debug and broadly compatible. gRPC uses HTTP/2 with binary Protocol Buffers, offering lower latency, smaller payloads, and built-in streaming.
  - Choose REST for external APIs and polyglot clients; choose gRPC for high-throughput internal communication where performance matters.

- **What is a BFF (Backend for Frontend) pattern and why is it important?**
  - BFF creates separate API surfaces for each client type (web, mobile, IoT), owned by the corresponding frontend team. Each BFF handles client-specific data aggregation, response shaping, and device-specific logic.
  - Without BFF, the API Gateway becomes bloated with client-specific transformations, and changes for one client risk breaking others.

- **How do you ensure security in a microservices architecture?**
  - Implement OAuth2/OIDC with JWT tokens validated at the API Gateway. Use mTLS for inter-service communication. Apply the principle of least privilege — each service has its own service account and minimal permissions.
  - Additional measures: network policies (Kubernetes NetworkPolicies), secret management (HashiCorp Vault, AWS Secrets Manager), and regular dependency scanning for vulnerabilities.

---

## Developer Recommendations

- **Start with a modular monolith, extract services only when needed** — Premature microservices add immense complexity (network, data consistency, observability, deployment). Start with clear package boundaries, separate database schemas per module, and well-defined API contracts within the monolith. Extract a service when: (a) the team can't deploy independently, (b) a module needs different scaling, or (c) the codebase exceeds 300K lines. Most applications never need microservices.

- **Never share databases between services** — A shared database creates tight coupling — a schema change in one service can break another.
  - Each service must own its data and expose it only via its API. If another service needs that data, it calls the API or consumes events. The only exception is read replicas for reporting, which should be treated as an internal implementation detail of the owning service.
  - **Production story:** A ride-sharing company ignored this: the trip service and payment service shared a MySQL database; a `NOT NULL` column added to the trips table by the trip team caused all payment service queries to fail, blocking payment processing for 3 hours during peak evening hours.

- **Use the API Gateway pattern but keep it thin** — A gateway that handles routing, auth, rate limiting, protocol translation, request transformation, and API composition becomes a bottleneck.
  - Keep the gateway focused on cross-cutting concerns (auth, routing, rate limiting). Move heavy logic to BFF services per client type. Monitor gateway latency — it should add <5ms overhead.
  - **Production story:** A retail company's gateway handled routing, auth, request transformation, and API composition — during Black Friday, the gateway's CPU saturated at 90%, adding 800ms of latency per request and taking the entire site down for 2 hours before they split network-layer routing to NGINX.

- **Implement observability from day one** — Debugging a distributed system without tracing, centralized logging, and metrics is impossible.
  - Install OpenTelemetry auto-instrumentation in every service. Standardize on structured JSON logging with trace IDs. Export metrics to Prometheus/Grafana. Set up dashboards before the first production deployment, not after the first outage.
  - **Production story:** A travel booking platform skipped distributed tracing because staging tests passed — their first production incident took 14 hours to debug across 8 services, and the root cause (a missing header propagation) was found only after adding tracing post-outage.

- **Use circuit breakers, retries, and timeouts on every inter-service call** — Every synchronous call between services can fail.
  - Layer resilience: timeout (cap wait time), retry (3 attempts with backoff), circuit breaker (stop calling failing services), bulkhead (isolate resources per service). Without these, a single slow service cascades failures across the entire system.
  - **Production story:** A media streaming service learned this during a major live event: their recommendation service slowed down, and without circuit breakers, all upstream services waiting for recommendations exhausted their connections — the entire site went down for 22 minutes during peak viewership.

- **Design for failure, not for success** — Assume every service call will fail, every message will be delayed, and every database will go down. Design accordingly: graceful degradation (return cached data when a service is down), fallbacks (return defaults for non-critical data), async processing where possible (queue messages instead of blocking), and health check endpoints for every service.
- **Use contract testing to enable independent deployments** — Without contract testing, each service deployment requires integration tests against all dependencies. Use Pact or Spring Cloud Contract to define API contracts. The provider runs contract tests in its CI — as long as contracts pass, both sides deploy independently. This decouples deployment schedules across teams.
- **Align service boundaries with team boundaries (Conway's Law)** — Conway's Law states that system architecture mirrors communication structure. If two teams need to coordinate for every deployment, their services are too coupled. Organize services so each team owns end-to-end ownership of their services, with well-defined APIs for cross-team communication.
