# Design Rate Limiter

## Overview

- **Definition** — A system that controls the rate of requests sent or received by throttling excess traffic above a configured threshold
- **Why It Exists** — Protects services from abuse, prevents resource exhaustion, ensures fair usage across tenants, and defends against DDoS attacks
- **Historical Context** — Early rate limiting was implemented at the network layer with firewall rules; modern APIs use token bucket and sliding window algorithms; cloud API gateways (AWS API Gateway, Kong) now provide built-in rate limiting
- **Key Concepts** — **Token bucket** allows bursts up to a capacity; **Sliding window** provides accurate counting without clock boundary spikes; **HTTP 429 + Retry-After** is the standard error response; **Distributed rate limiting** uses Redis with Lua for atomicity; **Placement** can be client-side, API gateway, or server-side

## Core Concepts

- Token bucket algorithm: tokens are added at a fixed rate (r tokens/second), each request consumes one token
  - Burst allowed up to bucket size (b tokens) — excess requests are dropped or queued
  - Parameters: refill rate (r), bucket capacity (b)
  - Use case: general API rate limiting, allows short bursts
- Leaky bucket algorithm: requests fill a bucket that leaks at a constant rate
  - Excess requests overflow and are discarded
  - Smooths traffic — no bursting allowed
  - Use case: shaping traffic for network bandwidth control
- Fixed window algorithm: count requests in discrete time buckets (e.g., per minute)
  - Reset counter at the boundary of each window
  - Problem: allows traffic spikes at window edges — 100 requests at 00:59 and 100 at 01:01 = 200 requests in 2 seconds
- Sliding window log: maintain a timestamp log per user, count requests within the sliding window
  - Accurate — no edge spikes
  - Memory intensive: stores timestamps for every request
- Sliding window counter: approximate sliding window by combining the current window partial count with the previous window weighted count
  - Memory efficient: only two counters per user (previous window, current window)
  - Redis sorted sets provide a hybrid approach with configurable precision
- Placement options
  - Client-side: easy to bypass, not sufficient alone
  - API gateway: centralized enforcement across services (Kong, AWS API Gateway, Envoy)
  - Server-side: sits closest to the resource being protected, most accurate
- HTTP 429 Too Many Requests with Retry-After header
  - Retry-After value in seconds tells the client when to retry
  - Standard error body should include the limit, remaining, and reset timestamp (X-RateLimit-* headers)
- Distributed rate limiting uses Redis sorted sets with Lua scripting for atomic window operations
  - ZREMRANGEBYSCORE removes expired entries, ZCARD counts active entries
  - Lua script ensures atomicity — no race conditions between check and increment
  - Redis Cluster shards rate limit state by key (user ID, IP)
- Rate shaping: instead of dropping excess requests, queue them and process at a controlled rate
  - More user-friendly than hard rejections
  - Requires a queue with bounded size to prevent unbounded backlog
- Per-user, per-IP, and global limits
  - Per-user: prevent a single user from overwhelming the system
  - Per-IP: protect against attacks from many accounts on one IP
  - Global: cap total throughput regardless of identity
  - Combine tiers for defense in depth

## Common Mistakes

- **Making the rate limiter itself a bottleneck**
  - A centralized rate limiter that every request must pass through becomes a single point of failure and a latency bottleneck
  - **Why it looks correct:** Centralized enforcement guarantees accuracy and consistency
  - Use local rate limiting (in-memory counters with periodic sync) for most decisions; use centralized Redis only for enforcement actions. Implement client-side caching of rate limit states with bounded staleness.
- **Using fixed window without handling edge spikes**
  - Burst traffic at window boundaries can double the effective rate, bypassing the intended limit
  - **Why it looks correct:** Fixed window is the simplest to implement with a Redis counter and TTL
  - Switch to sliding window counter or token bucket. If fixed window is unavoidable, reduce the window to a fraction of the desired interval (e.g., use 1-second windows for a per-minute limit).
- **Not returning clear rate limit headers**
  - Clients cannot adapt their behavior without knowing their limit status
  - **Why it looks correct:** The 429 status alone seems sufficient to signal throttling
  - Always return X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset headers. Include Retry-After in 429 responses.

## Real-World Scenarios

### Token Bucket in AWS API Gateway

- Each API key has a token bucket with configurable rate and burst
- Tokens refill per second; burst allows short traffic spikes
- Bucket state is stored per-region in a distributed cache

### Sliding Window Counter in GitHub API

- GitHub uses a sliding window counter for authenticated and unauthenticated requests
- 5000 requests per hour for authenticated, 60 per hour for unauthenticated
- X-RateLimit-* headers inform clients of remaining capacity

### Redis Sorted Sets in Kong API Gateway

- Kong's rate limiting plugin uses Redis sorted sets for distributed sliding window
- Each request's timestamp is added to a sorted set keyed by the limiting identifier
- Lua script atomically removes expired entries and checks the count

## Use Cases

- **API gateway rate limiting** — protecting public APIs from abuse by individual users or IPs
  - Token bucket or sliding window algorithm per user/IP. Returns `429 Too Many Requests` with `Retry-After` header when limit is exceeded.
  - **Avoid when:** the client is trusted (internal service-to-service) — internal traffic may bypass rate limiting to avoid latency overhead.

- **DDoS mitigation** — absorbing distributed denial-of-service attacks targeting application endpoints
  - Global rate limits at the edge (CDN/load balancer level) before traffic reaches application servers. Challenge-based protection (CAPTCHA) for suspicious sources.
  - **Avoid when:** the attack is a low-and-slow application-layer attack — rate limiting alone won't suffice; combine with WAF rules and anomaly detection.

- **Tiered pricing enforcement** — free tier limited to 1000 requests/hour while premium tier allows 100,000 requests/hour
  - Per-API-key rate limits configured based on subscription tier. Limits are checked on every request before processing.
  - **Avoid when:** pricing is based on usage volume rather than request count — consider metering and billing instead.

- **Login and registration throttling** — preventing brute-force password guessing or account enumeration
  - Aggressive rate limits on `/login` and `/register` endpoints per IP and per username/IP combination. Exponential backoff on repeated failures.
  - **Avoid when:** the application uses a dedicated identity provider (Auth0, Cognito) — delegate rate limiting to the IdP's configuration.

- **Resource fairness across tenants** — ensuring one noisy tenant doesn't degrade service for others in multi-tenant systems
  - Per-tenant rate limiting distributes resources proportionally. Burst capacity allows short spikes but sustained usage is capped.
  - **Avoid when:** tenants pay for dedicated capacity — rate limits should reflect purchased capacity, not enforce uniform fairness.

## Scenario-Based Questions

**Q: Your rate limiter uses a centralized Redis instance. During a traffic spike, Redis CPU hits 100% and the rate limiter starts failing open (allowing all requests). What happened and how do you fix it?**

- The rate limiter code is likely using synchronous Redis calls — every request blocks on Redis, creating a thundering herd
- Use local in-memory counters as the first line of defense; sync with Redis asynchronously every second
- Implement circuit breaker: degrade to local-only rate limiting when Redis latency exceeds a threshold
- **Interview follow-up:** How do you handle clock skew between application servers when using local rate limiting?

**Q: You need to rate limit API calls per user to 100 requests per minute. How do you ensure accuracy without slowing down every request?**

- Use a sliding window counter stored in local memory with periodic Redis sync
- Each server node maintains a local counter per user; when the local counter reaches 80% of the limit, check Redis for global count
- This reduces Redis calls by ~95% while maintaining reasonable accuracy
- **Interview follow-up:** What happens when a user switches servers mid-window and the new server has a stale local count?

**Q: Your rate limiter uses IP-based limiting. However, users behind a NAT all share the same IP and are incorrectly blocked. How do you fix this?**

- Combine IP-based limiting with user authentication — authenticated users are limited by user ID, not IP
- Use a higher rate limit for shared IPs to account for multiple users behind the same IP
- Implement device fingerprinting to distinguish users behind the same IP when authentication is not available
- **Interview follow-up:** How do you handle IPv6 addresses where each user may have a /64 subnet but the rate limiter sees individual IPs?

**Q: Your rate limiter blocks requests at the API gateway, but a downstream service needs to apply its own finer-grained limits. How do you design multi-layer rate limiting?**

- Implement rate limiting at both the gateway (coarse, per-client) and service (fine-grained, per-endpoint or per-resource)
- Pass rate limit headers from the gateway to downstream services so they know the client's current limit status
- Use a hierarchical token bucket: the gateway allocates tokens to services, and services allocate tokens to endpoints
- **Interview follow-up:** How do you prevent double-counting when both layers decrement the same token bucket?

**Q: You need to rate limit API calls per user to 1000 requests per hour. Which algorithm do you choose and why?**

- Use the sliding window counter algorithm: it provides good accuracy with low memory (two counters per user) and no edge spikes
- A fixed window would allow 1000 requests at 10:59 and another 1000 at 11:01, enabling 2000 requests in 2 minutes
- A token bucket allows bursts up to a capacity, which may not be desired if the limit must be strictly per-hour
- **Interview follow-up:** How would you handle the case where a customer needs burst capability beyond the steady-state rate?

**Q: Your rate limiter uses Redis for distributed state. During a Redis cluster partition, some rate limiter instances lose access to Redis and start allowing all requests. How do you prevent security issues?**

- Implement a fallback mode: when Redis is unavailable, use local in-memory rate limiting with conservative limits (lower than the configured rate)
- Log all fallback events and alert the operations team immediately
- Consider using a "fail closed" (deny all) or "fail open" (allow all) strategy based on the criticality of the protected resource
- **Interview follow-up:** How do you design the rate limiter to gracefully re-sync with Redis when the partition heals?

**Q: Your rate limiter returns 429 Too Many Requests but clients ignore the Retry-After header and keep retrying immediately, making the situation worse. How do you handle non-compliant clients?**

- Implement a "deny period" on the server: once a client is rate limited, drop all requests from that client for the duration of the Retry-After period, regardless of what the client does
- Use exponential backoff hints in the response body with clear documentation
- As a last resort, temporarily blacklist clients that ignore Retry-After headers
- **Interview follow-up:** How do you distinguish between a misconfigured client and a malicious client that intentionally ignores rate limits?

**Q: Your API has per-endpoint rate limits (e.g., /login: 10/min, /search: 100/min, /data: 1000/min). How do you design the rate limiter configuration?**

- Store rate limit configurations per endpoint in a central configuration service (e.g., etcd, ZooKeeper) that all rate limiter nodes read
- Use a hierarchical key in Redis: rate_limit:{client_id}:{endpoint_group}:{window}
- Allow wildcard patterns: /api/v1/* could have a global limit while specific paths have overrides
- **Interview follow-up:** How do you add rate limiting to a new endpoint without disrupting existing traffic?

**Q: You need to rate limit concurrent connections (not request rate) for a WebSocket service. How is this different from request rate limiting?**

- Concurrent connection limiting tracks the number of active connections per user or per IP, not the request rate over time
- Use a counter (Redis INCR on connect, DECR on disconnect) with a maximum threshold
- The key challenge is handling disconnections gracefully — if a client disconnects without proper cleanup, the counter stays incremented
- **Interview follow-up:** How do you handle the case where a user opens many connections and then some drop due to network issues, leaving stale counters?

**Q:** Your team wants to rate limit outgoing API calls to a third-party service that has a limit of 10 requests per second. How do you implement a client-side rate limiter?**

- Use a token bucket in local memory: 10 tokens, refill rate 10 tokens/second
- Queue outbound requests if the bucket is empty and send them as tokens become available
- Monitor the third-party response headers for 429s and dynamically reduce the rate if the provider is throttling
- **Interview follow-up:** How do you handle clock drift affecting the token bucket refill rate across multiple client instances?

## Interview Questions

- **Compare token bucket and sliding window algorithms.**
  - Token bucket: tokens refill at a fixed rate, bucket capacity allows bursts. Simple, allows natural traffic patterns. Sliding window: counts requests within a moving time window, no edge spikes. More accurate but memory intensive in pure form. The sliding window counter hybrid provides good accuracy with low memory.
- **How do you implement distributed rate limiting?**
  - Use Redis with sorted sets (ZADD for timestamps, ZREMRANGEBYSCORE for cleanup, ZCARD for counting) in a Lua script for atomicity. Shard by user ID or IP across Redis Cluster. Fall back to local in-memory limiting if Redis is unavailable, and log the fallback for monitoring.
- **What headers should a rate-limited API return?**
  - X-RateLimit-Limit (maximum requests allowed), X-RateLimit-Remaining (remaining in current window), X-RateLimit-Reset (Unix timestamp when the window resets), and Retry-After in 429 responses.
- **How does rate limiting differ from rate shaping?**
  - Rate limiting rejects excess requests with 429. Rate shaping queues excess requests and processes them at a controlled rate, smoothing traffic. Shaping is more user-friendly but requires bounded queues to prevent unbounded backlog.
- **How do you implement rate limiting for unauthenticated users?**
  - Use IP address as the limiting key. Combine with device fingerprinting (user-agent, browser fingerprints) for more accuracy. Set stricter limits for unauthenticated users (e.g., 10 requests/minute vs 1000 for authenticated). Consider CAPTCHA challenges when unauthenticated limits are exceeded.
- **Explain the concept of "rate limit tiers" for a SaaS product.**
  - Different pricing tiers get different rate limits: free tier (100 requests/hour), pro tier (10,000/hour), enterprise (100,000/hour). The tier is looked up from the API key or user account and used to parameterize the token bucket or sliding window configuration.
- **How do you test a rate limiter under production traffic?**
  - Use chaos engineering: inject traffic at varying rates and verify that the limiter correctly allows/denies based on configured thresholds. Monitor for false positives (legitimate requests blocked) and false negatives (excess requests allowed). Test Redis failure scenarios.
- **What is the "burst vs sustained rate" distinction in rate limiting?**
  - Burst rate is the maximum number of requests allowed in a very short period (e.g., 100 requests in 1 second). Sustained rate is the average over a longer period (e.g., 1000 requests per hour). Token buckets handle this naturally: bucket size controls burst, refill rate controls sustained rate.
- **How do you implement rate limit headers efficiently?**
  - Compute and cache the remaining count and reset timestamp during the rate limit check. Return these as X-RateLimit-* headers. The header values are already computed by the rate limiter logic, so there is minimal additional overhead.
- **What is the difference between global rate limiting and per-instance rate limiting?**
  - Global: a single counter shared across all application instances, requiring coordination (Redis). Per-instance: each instance tracks its own counters independently. Global is accurate but adds latency; per-instance is fast but may allow more requests than intended (by the number of instances).
- **How do you prevent rate limiter bypass via IP rotation?**
  - IP-based rate limiting alone is insufficient against distributed bots. Combine with user-based limits, device fingerprinting, behavioral analysis (rate of account creation, pattern of requests), and CAPTCHA challenges. Use machine learning to detect coordinated attacks.
- **Explain the role of rate limiting in preventing DDoS attacks.**
  - Rate limiting absorbs low-and-slow DDoS attacks by capping requests per source IP or per user. For large volumetric DDoS, rate limiting alone is insufficient — use DDoS protection services (Cloudflare, AWS Shield) that filter at the network layer before traffic reaches the application.
- **How does the leaky bucket algorithm differ from token bucket in behavior?**
  - Leaky bucket enforces a constant output rate regardless of input bursts — excess is discarded. Token bucket allows bursts up to capacity. Leaky bucket smooths traffic completely; token bucket permits natural traffic patterns with occasional spikes.
- **What is "rate limit overage" and how do you handle billing for it?**
  - Overage occurs when a client exceeds their purchased rate limit. Options: reject with 429, or allow with a higher rate and charge for overage (bill-back model). If allowing overage, ensure the system can handle the extra load and clearly communicate costs to the client.
- **How do you implement rate limiting for serverless functions (AWS Lambda)?**
  - Lambda concurrency limits (reserved concurrency) control how many functions run simultaneously. Use API Gateway rate limiting and usage plans for per-client limits. For internal service-to-service calls, implement a token bucket in a shared layer (DynamoDB or ElastiCache).
- **How does rate limiting work for streaming APIs?**
  - Streaming APIs are often limited by connection count and message throughput. Use concurrent connection limits per user. For event streams, use a credit-based system: each client gets a budget of messages per second, and the server paces delivery accordingly.
- **What is the "soft limit" vs "hard limit" distinction?**
  - Soft limit: warn the client that they are approaching the limit but still allow the request (return 200 with a warning header). Hard limit: reject the request with 429. Soft limits help clients adjust behavior before being blocked. Use soft limits at 80% and 90% of the hard limit.
- **How do you implement hierarchical rate limiting (per-user, per-endpoint, global)?**
  - Layer multiple token buckets: check global bucket first, then per-endpoint bucket, then per-user bucket. The most restrictive limit wins. Implement as a chain of responsibility where each layer can reject or pass through. Use Redis multi-key Lua script for atomic multi-layer checks.
- **How do you handle rate limiter configuration changes without restarting services?**
  - Store rate limit configurations in a dynamic config store (etcd, Consul, or a database) that the rate limiter watches for changes. Hot-reload configurations by periodically polling or using a watch mechanism. Log configuration changes for auditability.
- **How do you implement rate limiting for GraphQL APIs where a single request can trigger multiple field resolvers?**
  - Count each resolver invocation individually against the rate limit, or assign a "cost" per query field and reject queries exceeding the cost budget. Use query complexity analysis to calculate cost before execution. Token bucket per user with cost-based consumption prevents deep nested queries from bypassing rate limits.

## Developer Recommendations

- **Use local rate limiting as the first line of defense**
  - In-memory token bucket per server node avoids Redis latency on every request
  - Sync aggregated counts to Redis periodically for cross-node coordination
  - **Production story:** A rate limiter that called Redis on every API request added 5ms latency p99 and became the bottleneck during a flash sale — local counters dropped Redis calls by 98% with no accuracy loss
- **Always return standard rate limit headers**
  - Clients, SDKs, and automated systems depend on headers to self-throttle
  - Include Retry-After in 429 responses with a reasonable delay (exponential backoff hint)
- **Implement graceful degradation for rate limiter failures**
  - If the backing store (Redis) is unreachable, fall back to local limiting or fail open with alerting
  - Never let the rate limiter crash the main service — use circuit breakers and timeouts
  - **Production story:** A Redis cluster outage caused the rate limiter to block all API traffic for 12 minutes — the fix was a fallback to local limiting with aggressive alerting
- **Set limits per tier (user/IP/global) for defense in depth**
  - A single user-based limit can be bypassed by rotating accounts on the same IP
  - A single IP-based limit can be bypassed by a distributed botnet
  - Combine all three tiers with the most restrictive winning
