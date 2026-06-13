# Load Balancing

## Overview

- **Definition** — Distributing incoming network traffic across multiple backend servers to ensure no single server is overwhelmed
- **Why It Exists** — Single servers have capacity limits; load balancing provides horizontal scaling, fault tolerance, and high availability
- **Historical Context** — Early load balancing used round-robin DNS (1990s); hardware load balancers dominated the 2000s; software-defined and cloud-native LBs (HAProxy, Nginx, AWS ALB) emerged from 2010s onward
- **Key Concepts** — **L4** operates at transport layer (TCP/UDP); **L7** operates at application layer (HTTP/HTTPS); **Reverse proxy** adds features like TLS termination, caching, compression; **Health checks** detect backend failures; **Session affinity** pins a client to a backend server

## Core Concepts

- Round-robin distributes requests sequentially across the server list
  - Simple and stateless but ignores server load differences
- Least connections forwards requests to the server with the fewest active connections
  - Better for requests with varying processing times
  - Requires the load balancer to track connection counts per server
- Hash-based routing maps request attributes (client IP, URL, cookie) to a specific server via a hash function
  - Useful for session persistence without sticky cookies
  - When a server is added or removed, hash redistribution affects many clients
- Weighted distribution assigns proportionally more traffic to higher-capacity servers
  - Configure weights based on CPU, memory, or network capacity
- L4 load balancing forwards TCP/UDP packets without inspecting payload content
  - Lower latency, fewer resources, works for any protocol
  - Cannot make routing decisions based on HTTP headers or cookies
- L7 load balancing inspects application-layer data (HTTP headers, paths, cookies)
  - Enables content-based routing, rate limiting, authentication
  - Higher latency per request due to deeper packet inspection
- Passive health checks detect failures by observing real traffic patterns
  - Monitor for timeouts, 5xx status codes, connection resets
  - No extra probe traffic — relies on existing requests
- Active health checks periodically probe endpoints with configurable intervals
  - HTTP health endpoints returning 200, TCP port checks, or custom scripts
  - Detects failures before real traffic is affected
- Sticky sessions (session affinity) route a user to the same backend server using cookies
  - Breaks if the target server fails — the session is lost unless replicated
  - Alternative: store session state externally in Redis or a database for true statelessness
- DNS load balancing returns multiple A/AAAA records for a single domain
  - Clients pick an IP, usually the first or round-robin
  - Does not check server health — a down server still receives traffic until DNS TTL expires
- Geo-routing directs users to the nearest data center based on DNS resolver location
  - Reduces latency and can satisfy data sovereignty requirements
- Circuit breaker integration stops sending traffic to failing backends after a threshold of errors
  - States: closed (normal), open (failing), half-open (testing recovery)

## Common Mistakes

- **Health check misconfiguration causing cascading failures**
  - Setting health check intervals too aggressively floods backends with probes under load
  - **Why it looks correct:** Fast health checks seem to detect failures quickly
  - Set intervals to at least 5–10 seconds with a failure threshold of 2–3 consecutive failures; use separate health endpoints that do not require full application initialization
- **Sticky sessions as a default strategy**
  - Binding users to specific servers causes uneven load distribution and total session loss when a server fails
  - **Why it looks correct:** Sessions work without external storage; it is the simplest setup
  - Migrate session state to a distributed cache (Redis, Memcached) so any server can handle any request
- **Using L7 when L4 is sufficient**
  - Parsing HTTP headers for every request adds latency and CPU overhead when simple TCP forwarding is enough
  - **Why it looks correct:** L7 offers more features, so developers default to it
  - Use L4 for protocols that do not need content inspection (gRPC, WebSocket, database connections) and reserve L7 for HTTP-specific routing

## Real-World Scenarios

### HAProxy — High-Performance TCP/HTTP Proxy

- Used for Redis, MySQL, and HTTP load balancing
- Advanced health checks with customizable failure conditions
- ACL-based routing enables canary deployments and blue-green releases

### AWS ALB vs NLB

- ALB (Application Load Balancer): L7 with path-based routing, host-based routing, WebSocket support
- NLB (Network Load Balancer): L4 with ultra-low latency, static IP support, handles millions of requests per second
- Choose ALB for microservices with HTTP routing; choose NLB for performance-critical TCP workloads

### Envoy — Modern Service Mesh Proxy

- L7 proxy designed for microservices with built-in observability (metrics, tracing, logging)
- Supports advanced circuit breaking, retry policies, and outlier detection
- Used as the data plane in Istio service mesh

## Scenario-Based Questions

**Q: Your backend servers show uneven CPU usage despite round-robin load balancing. How do you diagnose and fix it?**

- Check if requests have varying processing times (heavy writes vs light reads) — round-robin does not account for this
- Switch to least-connections or weighted distribution based on CPU metrics
- Implement request-level metrics to identify slow endpoints causing head-of-line blocking
- **Interview follow-up:** How would you handle a "thundering herd" when a cold-cache server receives a burst from a newly activated load balancer?

**Q: A health check endpoint uses the full application stack and causes CPU spikes every 3 seconds across all backends. What is the fix?**

- Create a lightweight health endpoint that only checks essential dependencies (DB connection pool, disk space) without warming caches or running application logic
- Increase the health check interval to 15 seconds with a failure threshold of 3
- Use passive health checks to supplement or replace active probes
- **Interview follow-up:** How do you distinguish between a truly dead server and one that is simply slow due to a temporary GC pause?

## Interview Questions

- **What is the difference between L4 and L7 load balancing?**
  - L4 operates at the transport layer (TCP/UDP) forwarding packets blindly with low latency. L7 operates at the application layer (HTTP/HTTPS) inspecting content for smart routing but with higher overhead per request.
- **Explain active vs passive health checks and when to use each.**
  - Active health checks send periodic probes to configured endpoints, detecting failures proactively. Passive health checks monitor real traffic for error patterns. Use active for critical services and passive to supplement and reduce probe traffic.
- **How do you handle session persistence without sticky sessions?**
  - Store session state externally in Redis, Memcached, or a database with a session ID cookie. Every backend server reads session data from the shared store, making the system fully stateless.
- **What are the tradeoffs between DNS load balancing and a dedicated load balancer?**
  - DNS LB is simple, free, and works globally, but it cannot detect server health and relies on client DNS caching. A dedicated LB provides health checks, smart routing, TLS termination, and circuit breaking at the cost of infrastructure complexity.

## Developer Recommendations

- **Default to stateless design with external session storage**
  - Removes the dependency on sticky sessions and enables seamless horizontal scaling
  - Use Redis for session data with TTL-based expiration; make session ID the only cookie
  - **Production story:** A service with sticky sessions saw 30% of users lose their cart when a server failed — moving sessions to Redis eliminated the issue
- **Implement graceful health checks**
  - Separate liveness (is the process alive?) from readiness (can the server accept traffic?) probes
  - Health endpoint should check DB connectivity and queue depths but not force JIT compilation or cache warmup
  - **Production story:** A health check that validated payment provider connectivity caused all 50 backends to fail simultaneously when the payment API briefly timed out
- **Use connection pooling and circuit breakers on the LB side**
  - Prevents cascading failures by stopping traffic to degraded backends before they fully collapse
  - Integrate with Envoy or HAProxy circuit breaker settings: max connections, max pending requests, max retries
- **Match LB layer to protocol needs**
  - L4 for TCP-heavy workloads (gRPC streaming, database proxies, Redis)
  - L7 for HTTP API gateways with path/host routing, auth, and rate limiting
