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

## Use Cases

- **Traffic distribution across servers** — scaling a web application across multiple instances for high availability
  - ALB or NLB distributes incoming requests to healthy backend instances. Health checks automatically remove failing instances from the pool.
  - **Avoid when:** only one instance exists — a load balancer doesn't help availability without redundancy.

- **TLS termination** — offloading SSL/TLS decryption from application servers
  - Load balancer handles certificate management and decryption, passing plain HTTP to backend. Reduces CPU load and centralizes certificate management.
  - **Avoid when:** end-to-end encryption is mandatory — configure passthrough mode or re-encrypt between LB and backend.

- **Session persistence (sticky sessions)** — ensuring a user's requests always go to the same backend
  - Cookie-based or source-IP-based affinity. Essential for stateful applications that store session data in local memory.
  - **Avoid when:** the application is stateless or uses a distributed session store (Redis) — stickiness reduces load balancing effectiveness.

- **Path-based routing** — routing `api.example.com/users` to the User Service and `api.example.com/orders` to the Order Service
  - ALB or API Gateway routes based on URL path, host header, or query parameters. Enables microservice architectures behind a single endpoint.
  - **Avoid when:** the routing rules are complex and conditional — a service mesh (Envoy, Istio) provides more sophisticated routing.

- **Weighted target groups for canary deployments** — gradually shifting traffic from v1 to v2
  - Route 5% of traffic to the new version, monitor metrics, then increase to 25%, 50%, 100%. Rollback by resetting weights to 0%.
  - **Avoid when:** traffic patterns are identical across versions — blue-green deployment (instant switch) is simpler for verified releases.

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

**Q: A new backend server added to the pool instantly gets overloaded while existing servers are idle. What is the cause and how do you fix it?**

- The load balancer may be using weighted distribution where the new server has a disproportionately high weight
- DNS caching may be directing old clients only to the new server if DNS-based load balancing is also in use
- Check if health checks are failing on existing servers, causing the load balancer to route all traffic to the new server as the only healthy target
- **Interview follow-up:** How would you implement a "slow start" mode where a new server gradually receives traffic to warm its caches?

**Q: During a traffic spike, your load balancer's connection pool fills up and new connections are refused. How do you handle this?**

- Increase the maximum connection limit on the load balancer and set per-backend connection limits to avoid overwhelming downstream servers
- Implement connection queuing with a bounded queue — excess connections wait briefly rather than being immediately rejected
- Use auto-scaling on backend servers triggered by load balancer connection depth
- **Interview follow-up:** How do you distinguish between a legitimate traffic spike and a DDoS attack at the load balancer level?

**Q: Your WebSocket connections are dropped intermittently when traffic shifts between backend servers. How do you fix this?**

- Ensure session affinity (sticky sessions) for WebSocket routes so all messages in a session go to the same backend
- Use a dedicated WebSocket load balancer (HAProxy in TCP mode) that forwards the raw TCP stream without HTTP inspection
- Implement WebSocket reconnection on the client side with exponential backoff and session ID restoration
- **Interview follow-up:** How does L4 load balancing differ from L7 load balancing for WebSocket traffic?

**Q: One downstream microservice becomes slow, causing all requests to that service through the API gateway to time out. How do you prevent this from affecting other services?**

- Implement circuit breakers per service — if error rate exceeds a threshold, stop sending requests and return a fallback response
- Set per-service connection and read timeouts individually so a slow service does not hold gateway threads
- Use bulkhead isolation: dedicate a separate thread pool per downstream service
- **Interview follow-up:** How do you determine the optimal timeout values for each service?

**Q: You use round-robin DNS with multiple IPs for a web service. Some users report intermittent failures. What is the cause and fix?**

- DNS load balancing does not perform health checks — a dead server still receives traffic until DNS TTL expires
- Client DNS caching varies — some clients cache stale records beyond TTL, hitting removed servers
- Fix: use a dedicated load balancer with health checks as the sole DNS target
- **Interview follow-up:** How do you minimize downtime during DNS TTL propagation when switching load balancer IPs?

**Q: Write-heavy requests through your load balancer are starving read requests. How do you separate them?**

- Use L7 routing to split read and write traffic to separate backend pools that scale independently
- Implement weighted fair queuing so read requests always get a minimum bandwidth share
- Use separate load balancers or ports for read and write traffic with dedicated backend instances
- **Interview follow-up:** How would you handle read replicas that lag — should reads go to the master if freshness is required?

**Q: During a blue-green deployment with sticky sessions, users are unexpectedly logged out. What happened?**

- The sticky session cookie points to blue servers, but those servers are now draining — the session was stored locally and is unavailable on green servers
- Fix: store sessions externally (Redis, database) so any server in any environment can serve any user
- Use gradual traffic shifting (canary deployment) rather than instant cutover
- **Interview follow-up:** How do you drain existing connections on the blue environment without dropping active users?

**Q: Your load balancer's TLS termination is causing high CPU usage. How do you reduce overhead without compromising security?**

- Offload TLS to dedicated hardware or use cloud-managed TLS termination (AWS ALB, CloudFront)
- Use ECDHE instead of DHE for forward secrecy — ECDHE is ~10x faster on modern CPUs
- Enable TLS session resumption (session IDs or tickets) so repeated connections skip the full handshake
- **Interview follow-up:** What security risks are introduced by terminating TLS at the load balancer instead of the application server?

## Interview Questions

- **What is the difference between L4 and L7 load balancing?**
  - L4 operates at the transport layer (TCP/UDP) forwarding packets blindly with low latency. L7 operates at the application layer (HTTP/HTTPS) inspecting content for smart routing but with higher overhead per request.
- **Explain active vs passive health checks and when to use each.**
  - Active health checks send periodic probes to configured endpoints, detecting failures proactively. Passive health checks monitor real traffic for error patterns. Use active for critical services and passive to supplement and reduce probe traffic.
- **How do you handle session persistence without sticky sessions?**
  - Store session state externally in Redis, Memcached, or a database with a session ID cookie. Every backend server reads session data from the shared store, making the system fully stateless.
- **What are the tradeoffs between DNS load balancing and a dedicated load balancer?**
  - DNS LB is simple, free, and works globally, but it cannot detect server health and relies on client DNS caching. A dedicated LB provides health checks, smart routing, TLS termination, and circuit breaking at the cost of infrastructure complexity.
- **How do you implement canary deployments using a load balancer?**
  - Route a small percentage of traffic (e.g., 5%) to the new version using weighted distribution. Monitor error rates, latency, and business metrics. Gradually increase the weight until 100% goes to the new version. If metrics degrade, route all traffic back to the old version instantly.
- **Explain the role of a health check endpoint design.**
  - A health endpoint should be lightweight, checking DB connectivity and essential dependencies without warming caches or running complex queries. Liveness probes check if the process is alive; readiness probes check if the server can accept traffic.
- **What is the difference between a reverse proxy and a load balancer?**
  - A reverse proxy forwards client requests to backends, providing caching, TLS termination, and compression. A load balancer distributes traffic across servers for scalability and fault tolerance. Many tools (Nginx, HAProxy, Envoy) serve as both.
- **How does the least-connections algorithm work and when would you use it?**
  - The load balancer tracks active connections per backend and forwards each request to the server with the fewest active connections. This is ideal when request processing times vary significantly — faster servers naturally receive more requests.
- **What is connection draining and why is it important?**
  - Connection draining allows existing in-flight requests to complete on a server being taken out of rotation. The LB stops sending new requests but keeps existing connections open until they finish or a timeout expires. This prevents abrupt termination during deployments or scale-in events.
- **Explain how a circuit breaker pattern integrates with load balancing.**
  - The circuit breaker monitors error rate from a backend. When errors exceed a threshold, the breaker opens and traffic stops. After a cooldown, it half-opens to test a few requests. If they succeed, the breaker closes; if not, it stays open.
- **How do you handle load balancing for database connections?**
  - Use a protocol-aware LB (HAProxy, ProxySQL) to distribute read queries across replicas via least-connections. Route all writes to the primary. Health checks should verify DB connectivity and that replication lag is within bounds.
- **What is geo-routing and how does it differ from latency-based routing?**
  - Geo-routing directs traffic based on the client's geographic location. Latency-based routing directs traffic to the data center with the lowest measured latency for that client. Geo-routing is simpler; latency-based routing requires continuous measurements.
- **How does a load balancer handle WebSocket upgrades?**
  - L7 LBs detect the HTTP Upgrade header and either forward in TCP mode or proxy WebSocket natively. Once upgraded, the LB maintains the persistent connection to the same backend (session affinity). L4 LBs handle WebSocket transparently.
- **What are the tradeoffs of software vs hardware load balancers?**
  - Software LBs (HAProxy, Nginx, Envoy) are cost-effective, dynamically configurable, and integrate with container orchestration. Hardware LBs (F5, Citrix) offer dedicated ASIC processing and higher raw throughput but are expensive and less flexible.
- **How do you implement rate limiting at the load balancer level?**
  - Configure per-IP or per-connection rate limits. L7 LBs can apply finer limits per URL path or API key. Excess requests get 429. For distributed limiting, the LB coordinates via a shared Redis instance.
- **What is the difference between a VIP and a real server?**
  - The VIP is the public-facing IP clients connect to. The LB forwards traffic to real servers. The VIP provides a single entry point, abstracting backend topology — if a real server fails, traffic shifts to healthy ones transparently.
- **How do you test if a load balancer is working correctly?**
  - Verify traffic distribution matches the configured algorithm. Test failover by stopping a backend and confirming traffic shifts. Test health check accuracy by introducing dependency failures. Measure latency added by the LB vs baseline.
- **Explain exponential backoff for client retries in a load-balanced environment.**
  - The client waits 1s before retrying, then 2s, 4s, 8s up to a maximum. This prevents retry storms. Jitter (random variation) prevents synchronized retries from multiple clients.
- **What is head-of-line blocking in load balancing and how do you prevent it?**
  - One slow request blocks subsequent requests on the same connection. In HTTP/1.1, requests on one connection are serialized. Prevention: use HTTP/2 multiplexing, enable connection pooling with parallel connections, or use separate backend connections per request.
- **How does a load balancer handle SSL/TLS termination?**
  - The load balancer decrypts incoming HTTPS requests and forwards plain HTTP to backend servers. This offloads the CPU-intensive encryption work from backends. The LB manages TLS certificates and can support multiple domains via SNI. Backend communication should still be encrypted if traversing untrusted networks.

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
