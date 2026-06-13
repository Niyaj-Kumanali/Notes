# API Gateway

## Overview

- **Definition** — An API gateway is a single entry point for client requests that handles routing, composition, authentication, rate limiting, and cross-cutting concerns before forwarding to backend services.
- **Why It Exists** — In a microservices architecture, clients would otherwise need to track multiple service endpoints, handle authentication separately for each, and implement cross-cutting logic (rate limiting, logging, circuit breaking) redundantly; the gateway centralises these concerns.
- **Historical Context** — Early REST APIs used simple reverse proxies (Nginx, HAProxy); Netflix's Zuul (2012) popularised the "edge service" pattern for filtering and routing at scale; cloud providers later offered managed gateways (AWS API Gateway 2015); the pattern evolved to support WebSocket, gRPC, and GraphQL.
- **Key Concepts** — **Route** — mapping from an incoming request path/method to a target service; **Predicate** — condition that determines if a route matches (path, header, query param, time); **Filter** — intercepting logic executed before or after the request is forwarded; **Rate Limiter** — throttles requests based on client identity or IP; **Circuit Breaker** — stops forwarding to a failing upstream; **Aggregation** — combining responses from multiple services into a single client response.

## Core Concepts

- **Spring Cloud Gateway**
  - Built on Spring WebFlux (Project Reactor), providing non-blocking, reactive request handling.
  - Routes are defined in YAML/Java DSL with predicates (Path, Header, Method, Query, Cookie, Host, RemoteAddr, Weight) and filters (AddRequestHeader, CircuitBreaker, Retry, RateLimiter, RewritePath).
  - The `RouteDefinitionLocator` loads route definitions from configuration, a discovery service (Eureka/Consul), or a database.
  - Filters form a chain: pre-filters run before the downstream call; post-filters run after the response comes back.
  - Built-in integration with Spring Cloud Circuit Breaker (Resilience4j) and Spring Cloud LoadBalancer.
  - WebFlux-based: does not use Servlet API, blocking I/O, or Thread-per-Request model.

- **Zuul (Netflix)**
  - **Zuul 1.x** — Servlet-based, blocking I/O. Each request occupies a thread until the response is received. Simple and well-understood but suffers under high concurrency due to thread-per-request overhead.
  - **Zuul 2.x** — Non-blocking, reactive (Netty-based). Supports multiplexed connections, better resource utilisation, and async filters.
  - Both versions share the filter pipeline: pre-routing, routing, post-routing, and error filters.
  - Filters can dynamically change routing targets, inject headers, modify request bodies, or short-circuit responses.

- **Kong (API Gateway)**
  - Built on OpenResty (Nginx + LuaJIT), providing high-performance, low-latency request processing.
  - Plugin-based architecture: authentication (Key Auth, JWT, OAuth2, LDAP), security (CORS, IP restriction, WAF), traffic control (rate limiting, request size limiting), observability (logging, metrics, tracing).
  - Plugins are written in Lua, but Kong also supports Go, JavaScript (V8), and Python plugins via separate process execution.
  - Declarative configuration (decK) for CI/CD; DB-less mode for Kubernetes deployments.
  - Enterprise edition adds Dev Portal, Vault integrations, and advanced RBAC.

- **AWS API Gateway**
  - Managed service supporting REST APIs, HTTP APIs (lower latency, simpler), and WebSocket APIs.
  - REST APIs: full-featured with request/response transformation, API keys, usage plans, caching, and AWS WAF integration.
  - HTTP APIs: designed for lower latency and lower cost; supports JWT and Lambda authorisers; no API keys or usage plans.
  - WebSocket APIs: maintains persistent bidirectional connections; routes messages based on `$connect`, `$disconnect`, and custom route keys.
  - Usage plans and API keys allow throttling at the client level with burst and rate limits.
  - Throttling: account-level and method-level; exceeding limits returns `429 Too Many Requests`.

- **KrakenD**
  - Stateless, performance-focused gateway written in Go; no plugin system — behaviour is configured entirely via a JSON/YAML config file.
  - Designed for API aggregation: one client request becomes multiple backend calls merged into a single response.
  - Every endpoint definition includes `backend` objects specifying the upstream URL, method, and optional mapping/transformation.
  - Achieves very low p99 latency because there is no dynamic code execution per request.

## Common Mistakes

- **Gateway becoming a bottleneck**
  - The gateway becomes the single chokepoint in the system; every client request and every microservice response passes through it, causing latency spikes under load.
  - **Why it looks correct:** The gateway is supposed to centralise cross-cutting concerns; but teams often run a single gateway instance with no autoscaling and no circuit breakers to upstream services.
  - The fix: Deploy the gateway with horizontal autoscaling (CPU/memory/request rate), set aggressive timeouts per route, and implement circuit breakers so a slow upstream does not block gateway threads.

- **Rate limiting misconfiguration**
  - Rate limits are applied too broadly (global instead of per-client) or too narrowly (per-route but clients call multiple routes), causing legitimate traffic to be throttled or malicious traffic to slip through.
  - **Why it looks correct:** Setting a single rate limit is simple and seems sufficient, but it treats all clients equally and does not account for different usage tiers or API endpoints.
  - The fix: Use tiered rate limiting (per client ID, per route, per IP) with burst allowances; monitor rate limit hit rates to adjust thresholds; use distributed rate limit stores (Redis) across gateway instances.

- **Authentication overhead on every request**
  - Every request performs a full authentication flow (e.g., JWT validation with a remote introspection endpoint, or database lookup), adding 50–200ms of latency per request.
  - **Why it looks correct:** The gateway is the "security boundary" so validating auth on every request seems necessary, but this ignores caching and the trade-off between security and performance.
  - The fix: Cache JWT validation results (short TTL), use local JWKS for token signature verification (no remote call), and pass validated claims via headers to downstream services to avoid re-validation.

- **Single gateway for internal and external traffic**
  - The same gateway handles both public-facing APIs and inter-service communication, making it a high-value attack target and a single point of failure for internal traffic.
  - **Why it looks correct:** Using one gateway is simpler operationally, but it introduces unnecessary latency for internal calls and exposes internal service boundaries to the outside.
  - The fix: Split into an external gateway (handles auth, rate limiting, WAF) and an internal gateway (handles routing, aggregation, circuit breaking) — the internal gateway can skip expensive auth checks.

## Real-World Scenarios

### Netflix's Zuul 1.x to Zuul 2.x Migration

- Zuul 1.x's thread-per-request model became unsustainable as Netflix scaled to billions of requests per day; thread pools were exhausted during traffic spikes.
- Zuul 2.x was rewritten on Netty with a fully async filter pipeline, reducing the thread footprint by 80% for the same throughput.
- The migration required rewriting every custom filter (400+ filters) from synchronous Servlet-based to async Netty-based, taking over a year.

### AWS API Gateway Throttling During a Flash Sale

- An e-commerce site used API Gateway with a usage plan of 10,000 requests per second (RPS). During a flash sale, traffic spiked to 30,000 RPS and API Gateway returned 429 to 20% of users.
- The team had not configured burst limits appropriately and had no Redis-based rate limiting on the application side as a safety net.
- They mitigated by implementing client-side retry with exponential backoff and adding a request queue in front of API Gateway for the next sale.

### KrakenD as BFF (Backend for Frontend)

- A media company replaced a monolithic Rails API with KrakenD as a BFF for their mobile app.
- KrakenD aggregated 8–12 separate microservice calls per mobile screen into a single JSON response, reducing mobile client latency from 800ms to 120ms.
- The absence of a plugin system was a benefit: teams could not add complex logic to the gateway, forcing them to push business logic into backend services.

## Scenario-Based Questions

**Q: A team deploys a new version of a microservice that has a bug causing 5-second response times on 10% of requests. The API Gateway's thread pool is soon exhausted and all routes become slow. What went wrong?**

- The gateway had no per-route circuit breaker or timeout. A slow upstream service consumed all available threads/p connections, starving other routes. The fix is to configure per-route timeouts and circuit breakers (e.g., Spring Cloud Gateway + Resilience4j) that open after a configurable failure threshold, isolating the failing service.
- **Interview follow-up:** How would you choose the circuit breaker threshold so that transient failures do not open the circuit unnecessarily while still protecting the gateway?

**Q: A client app sends the same JWT token with every request. The gateway validates the token by calling an external OAuth introspection endpoint each time, adding 150ms of latency. How can you reduce this latency?**

- Cache the JWT validation result in-memory with a TTL based on the token's expiry. Better yet, validate the JWT locally using the JWKS endpoint (fetch the signing keys once and cache them) to avoid any remote call during validation. Pass the validated claims (user ID, roles) in request headers to downstream services.
- **Interview follow-up:** How would you handle JWT revocation when a user is logged out or their permissions change, if you are using local JWKS validation with cached keys?

**Q: During a DDoS attack, the API Gateway CPU reaches 100% and legitimate users cannot access the system. The rate limiter is configured but did not trigger. Why?**

- The rate limiter was likely configured at the application level (after request parsing) rather than at the network level. A DDoS attack can overwhelm the gateway before it reaches the rate-limiting logic. The fix involves adding network-level rate limiting (AWS WAF, Cloudflare, Nginx limit_req) in front of the gateway, and ensuring the gateway's rate limiter uses a fast, distributed store (e.g., Redis) with minimal overhead.
- **Interview follow-up:** How would you distinguish between a legitimate traffic surge from a flash sale and a DDoS attack?

## Interview Questions

- **Compare Spring Cloud Gateway and Zuul 2.x. When would you choose one over the other?**
  - Both are reactive (Netty-based) gateways. Spring Cloud Gateway is tightly integrated with the Spring ecosystem (Eureka, Config, Security) and uses a declarative route DSL with WebFlux predicates and filters. Zuul 2.x is more flexible with its filter pipeline (filter chaining, dynamic filter loading from a store) but requires more JVM tuning. Choose Spring Cloud Gateway for Spring Boot microservices; choose Zuul for environments requiring dynamic, hot-reloaded filter logic.

- **What is the difference between a Kong plugin and a Spring Cloud Gateway filter?**
  - Kong plugins are written in Lua (or Go/JS) and run at the Nginx worker level via OpenResty; they can modify requests at a very low level (TCP, TLS, HTTP/2). Spring Cloud Gateway filters are Java-based and run at the application layer (WebFlux exchange). Kong plugins are loaded dynamically from a plugin server; Gateway filters are compiled and deployed with the application.

- **How would you design a gateway to handle 100,000 requests per second?**
  - Use a stateless, reactive gateway (Kong, KrakenD, Spring Cloud Gateway with WebFlux). Deploy behind a global load balancer (AWS Global Accelerator, Cloudflare) with multiple active instances in different regions. Keep all state (rate limit counters, auth tokens) in a fast external store (Redis Cluster) with local caching. Use connection pooling and HTTP/2 multiplexing to reduce per-connection overhead.

- **What is the purpose of usage plans in AWS API Gateway?**
  - Usage plans define throttling limits (rate, burst) and quota (daily/monthly) per API key. They allow API providers to offer tiered access (free tier: 1000 req/day; paid tier: 100,000 req/day) and prevent a single consumer from overwhelming the API. Exceeding limits returns `429 Too Many Requests`.

- **How does an API gateway handle WebSocket connections differently from REST?**
  - WebSocket connections are long-lived and bidirectional; the gateway must maintain a persistent connection to both the client and the backend (or route messages via a pub/sub channel). AWS API Gateway uses a `$connect` route to authenticate the WebSocket upgrade and a `$disconnect` route for cleanup; messages are routed to registered integrations. Kong and Spring Cloud Gateway support WebSocket proxying but do not offer the same managed connection state as AWS.

## Developer Recommendations

- **Deploy separate gateways for external and internal traffic**
  - External gateways handle authentication, WAF, rate limiting, and DDoS protection. Internal gateways focus on routing, aggregation, circuit breaking, and observability. This isolates failure domains and reduces latency for inter-service calls.
  - Implementation: Deploy an external Kong cluster behind a CDN; deploy an internal Spring Cloud Gateway cluster with mTLS to services. The internal gateway skips auth filters but adds distributed tracing headers.
  - **Production story:** A logistics company split their gateway and reduced p99 inter-service latency from 45ms to 12ms because the internal gateway avoided auth token parsing on every hop.

- **Always configure per-route timeouts, retries, and circuit breakers**
  - Without these, a single slow or failing upstream can saturate gateway resources and degrade all routes.
  - Implementation: In Spring Cloud Gateway, use `Resilience4jCircuitBreakerFilter` with a sliding window of 20 requests, failure threshold of 50%, and a timeout of 2 seconds. Configure retry with exponential backoff (max 3 retries, 100ms initial delay).

- **Use distributed rate limiting with Redis**
  - In-memory rate limiting is inconsistent across gateway instances; a client can exceed the intended limit by hitting multiple instances.
  - Implementation: Use the Sliding Window Log algorithm with Redis sorted sets. Kong has a built-in Redis rate-limiting plugin; for Spring Cloud Gateway, use Spring Cloud Gateway Redis Rate Limiter (`RequestRateLimiter` filter with a Redis-based `KeyResolver`).

- **Monitor gateway metrics aggressively**
  - The gateway is a critical infrastructure component; its performance and error rates must be monitored with low-latency alerting.
  - Implementation: Expose metrics (request rate, latency p50/p95/p99, error rate by route, thread pool utilisation, connection pool depth) to Prometheus. Set alerts for p99 latency > 500ms, error rate > 1%, and thread pool utilisation > 80%.
