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

## Interview Questions

- **Compare token bucket and sliding window algorithms.**
  - Token bucket: tokens refill at a fixed rate, bucket capacity allows bursts. Simple, allows natural traffic patterns. Sliding window: counts requests within a moving time window, no edge spikes. More accurate but memory intensive in pure form. The sliding window counter hybrid provides good accuracy with low memory.
- **How do you implement distributed rate limiting?**
  - Use Redis with sorted sets (ZADD for timestamps, ZREMRANGEBYSCORE for cleanup, ZCARD for counting) in a Lua script for atomicity. Shard by user ID or IP across Redis Cluster. Fall back to local in-memory limiting if Redis is unavailable, and log the fallback for monitoring.
- **What headers should a rate-limited API return?**
  - X-RateLimit-Limit (maximum requests allowed), X-RateLimit-Remaining (remaining in current window), X-RateLimit-Reset (Unix timestamp when the window resets), and Retry-After in 429 responses.
- **How does rate limiting differ from rate shaping?**
  - Rate limiting rejects excess requests with 429. Rate shaping queues excess requests and processes them at a controlled rate, smoothing traffic. Shaping is more user-friendly but requires bounded queues to prevent unbounded backlog.

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
