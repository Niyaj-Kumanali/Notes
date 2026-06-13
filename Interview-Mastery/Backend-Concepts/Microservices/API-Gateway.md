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

**Q: Your API Gateway is configured with per-route rate limiting, but during a promotion event, users across multiple routes are throttled even though individual route limits are not exceeded. Why?**

- Rate limits are applied per route, but a single user's requests span multiple routes. The user exceeds the global limit without exceeding any single route limit. The fix is to implement global rate limiting (per user across all routes) in addition to per-route limits, using a distributed counter (Redis) keyed by user ID or API key.
- **Interview follow-up:** How would you design a rate-limiting scheme that allows burst traffic for authenticated users while still protecting the system from abuse?

**Q: A team deploys a GraphQL endpoint behind the API Gateway. The gateway is configured for REST and fails to handle GraphQL requests properly. What needs to change?**

- GraphQL typically uses a single POST endpoint (`/graphql`) where all queries are sent. The API Gateway must route all GraphQL requests to the same backend service regardless of the query content. Caching at the gateway level is ineffective for GraphQL since queries vary. The gateway should skip request/response transformation for GraphQL traffic and pass the request body through unchanged. Consider deploying a separate GraphQL gateway or BFF for GraphQL traffic.
- **Interview follow-up:** How would you implement rate limiting for a GraphQL endpoint where a single request can trigger expensive or cheap queries?

**Q: During a regional cloud outage, your API Gateway's upstream services are unavailable, but the gateway keeps accepting requests and timing out, exhausting resources. How do you protect the gateway?**

- Implement a "circuit breaker per upstream service" that opens when the upstream is unhealthy and returns a cached or fallback response. Configure the gateway with a global "degraded mode" flag that, when activated, serves static responses for non-critical routes. Use a health check endpoint on each upstream and stop routing to unhealthy services at the gateway level.
- **Interview follow-up:** How would you design a fallback response strategy that provides meaningful data to users even when upstream services are down?

**Q: Your team uses an API Gateway for authentication, but the monolithic auth service is becoming a bottleneck — every request goes through it. How do you scale authentication without replacing the auth service?**

- Move token validation to the gateway by caching the auth service's public keys (JWKS) locally. The gateway validates JWT tokens without calling the auth service on every request. For token revocation, use a distributed revocation list (Redis) that the gateway checks with minimal overhead. This reduces auth service load from O(N) requests to O(1) per gateway instance.
- **Interview follow-up:** How do you handle token revocation when the user's permissions change while they hold a valid JWT that is cached in the gateway?

**Q: A mobile app connects to the API Gateway via WebSocket. After a gateway deployment, all WebSocket connections drop and the app must reconnect. How do you prevent this?**

- WebSocket connections are stateful and tied to a specific gateway instance. Use a gateway that supports WebSocket sticky sessions (session affinity via a cookie or source IP hash). For zero-downtime deployments, implement a graceful shutdown procedure where the gateway stops accepting new WebSocket connections, waits for existing connections to drain, and sends a close frame to remaining connections before shutting down.
- **Interview follow-up:** How would you implement WebSocket connection migration between gateway instances without requiring client reconnection?

**Q: Your API Gateway is deployed behind a CDN (CloudFront, Cloudflare). Some client requests are being cached at the CDN level, causing users to see stale data. What configuration is needed?**

- Configure the CDN to respect cache-control headers from the gateway or backend services. For dynamic APIs, set `Cache-Control: no-cache, no-store` or use short TTLs (e.g., 1-5 seconds). Use the CDN's cache key customization to include relevant headers (e.g., `Accept-Language`, `Authorization`) so different users get different cached responses. For authenticated APIs, consider bypassing CDN caching entirely.
- **Interview follow-up:** Some of your APIs are intentionally cacheable (product listings, static content) but others are not (user balances, order status). How do you design a caching strategy at the CDN and gateway layers that correctly handles both types?

**Q: A startup uses a single API Gateway instance with 2GB of memory. As traffic grows, the gateway's memory usage reaches 90% and requests are frequently dropped. How do you scale the gateway?**

- Horizontal scaling: deploy multiple gateway instances behind a load balancer. Make the gateway stateless (store rate limit counters, auth caches in Redis) so any instance can handle any request. Configure auto-scaling based on CPU and memory metrics. If the gateway is stateful (WebSocket connections), use sticky sessions to route clients to the same instance.
- **Interview follow-up:** How would you decide between scaling the gateway vertically (larger instance) versus horizontally (more instances) in terms of cost, complexity, and latency?

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

- **What is the role of the API Gateway in a microservices architecture?**
  - The API Gateway is the single entry point for all client requests. It handles routing, authentication, rate limiting, request/response transformation, API composition, and cross-cutting concerns like logging and circuit breaking.
  - It shields clients from the complexity of multiple service endpoints and centralizes edge concerns.

- **How does Spring Cloud Gateway handle requests asynchronously?**
  - Spring Cloud Gateway is built on Spring WebFlux and Project Reactor. It uses a non-blocking, event-driven model with a small number of threads handling many concurrent connections via Netty.
  - This contrasts with Zuul 1.x (Servlet-based, thread-per-request) and provides better resource utilization under high concurrency.

- **What is the difference between a Gateway filter and a Global filter?**
  - Gateway filters are applied to specific routes (e.g., `AddRequestHeader`, `CircuitBreaker`). Global filters are applied to all routes automatically (e.g., `LoadBalancerClientFilter`, `NettyRoutingFilter`).
  - In Spring Cloud Gateway, global filters handle cross-cutting concerns for every request.

- **How do you implement rate limiting in Kong?**
  - Kong provides a built-in `rate-limiting` plugin that supports local (in-memory) and distributed (Redis) rate limiting. It can limit by consumer, credential, IP, or service.
  - Configuration includes `second`, `minute`, `hour`, `day`, `month`, `policy` (local/cluster/redis), `fault_tolerant`, and `hide_client_headers`.

- **What is the purpose of API Gateway caching?**
  - Caching at the gateway level reduces load on backend services and improves response latency for frequently accessed endpoints. AWS API Gateway supports caching with configurable TTL and cache key parameters.
  - Use caching for read-heavy, infrequently changing data (product listings, reference data). Disable caching for authenticated or dynamic responses.

- **How do you handle CORS at the API Gateway level?**
  - Configure CORS headers (Access-Control-Allow-Origin, Access-Control-Allow-Methods, Access-Control-Allow-Headers) at the gateway so that browser-based clients can make cross-origin requests.
  - In Spring Cloud Gateway, use a `CorsGlobalConfiguration` bean or per-route CORS configuration. In Kong, use the `cors` plugin.

- **What is the role of the API Gateway in authentication and authorization?**
  - The gateway authenticates requests by validating JWT tokens, API keys, or calling an external auth service. It extracts user identity and passes it to downstream services via headers (e.g., `X-User-ID`, `X-User-Roles`).
  - Authorization (checking permissions for specific resources) is typically delegated to the backend services, though coarse-grained authorization can be done at the gateway level.

- **How do you implement canary deployments with an API Gateway?**
  - Use a weight-based routing mechanism where a percentage of traffic is routed to the canary (new) version. Spring Cloud Gateway supports weight predicates (`Weight=canary-v2,10`). Kong supports canary via blue-green or weighted upstreams.
  - The gateway can also route based on headers (e.g., `X-Canary: true`) for internal testing before rolling out to production traffic.

- **What is the difference between synchronous and asynchronous API Gateways?**
  - Synchronous gateways (Spring Cloud Gateway, Kong) process requests in a request-response cycle, blocking the connection until the backend responds. Asynchronous gateways (message-based) receive requests, publish them to a queue, and return responses via a callback or polling mechanism.
  - Synchronous gateways are simpler for most use cases; asynchronous gateways are useful for long-running operations or when backends have variable latency.

- **How does the API Gateway handle gRPC traffic?**
  - gRPC uses HTTP/2 and binary Protocol Buffers. Some gateways support gRPC natively (Envoy, Kong with gRPC plugin, AWS API Gateway with gRPC proxy). Spring Cloud Gateway does not natively support gRPC — use a separate Envoy proxy for gRPC traffic.
  - For gRPC, the gateway typically performs TLS termination, routing based on the gRPC service/method, and load balancing across gRPC backends.

- **What is the difference between API Gateway composition and BFF?**
  - API Gateway composition aggregates responses from multiple services into a single client response. BFF (Backend for Frontend) is a dedicated backend per client type that handles composition, data shaping, and client-specific logic.
  - BFF is preferred when different clients need different data shapes; API Gateway composition is suitable when the same data shape is needed by multiple clients.

- **How do you secure the API Gateway itself?**
  - Deploy behind a WAF (AWS WAF, Cloudflare). Use TLS termination at the gateway. Implement IP allowlisting/blocklisting. Run the gateway in a private subnet with only necessary ports open. Use a CDN or DDoS protection service in front of the gateway.
  - Regularly update the gateway software, apply security patches, and audit access logs for suspicious activity.

- **What is the role of the API Gateway in observability?**
  - The gateway generates request-level metrics (latency, error rate, request count) per route and per client. It propagates distributed tracing headers (trace ID, span ID) to downstream services. It can generate access logs with detailed request/response metadata.
  - Centralized gateway observability provides a holistic view of all north-south traffic, making it easier to detect client-side issues.

- **How do you handle API Gateway failure?**
  - Deploy multiple gateway instances behind a load balancer in an active-active configuration. Use health checks at the load balancer level to detect and remove failed gateway instances. Implement a fallback mechanism: if all gateways are down, serve static error pages from the CDN or load balancer.
  - Design clients to handle gateway failures gracefully (retry, fallback to cached data).

- **What is the difference between a reverse proxy and an API Gateway?**
  - A reverse proxy (Nginx, HAProxy) operates at layers 4/7 and handles simple routing, SSL termination, and basic load balancing. An API Gateway builds on this with application-layer features: authentication, rate limiting, request/response transformation, API composition, and circuit breaking.
  - The API Gateway is the richer, more feature-complete evolution of a reverse proxy for microservices architectures.

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
