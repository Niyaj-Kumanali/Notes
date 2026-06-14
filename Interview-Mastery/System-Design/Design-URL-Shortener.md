# Design URL Shortener

## Overview

- **Definition** — A service that takes a long URL and returns a short alphanumeric code that redirects to the original URL
- **Why It Exists** — Long URLs are hard to share, exceed character limits (SMS, Twitter), and are visually unappealing; short URLs also enable click tracking and analytics
- **Historical Context** — TinyURL launched in 2002; bit.ly became the dominant player post-2008; custom branded short URLs emerged with link management platforms
- **Key Concepts** — **Base62 encoding** generates compact keys from numeric IDs; **301 vs 302 redirects** determine browser caching behavior; **Analytics pipeline** captures click data asynchronously; **Custom aliases** allow user-defined short codes; **Rate limiting** prevents abuse

## Core Concepts

- Key generation via Base62 encoding (a-z, A-Z, 0-9, 62 characters)
  - A 7-character Base62 key supports 62^7 (~3.5 trillion) unique URLs
  - Convert a unique numeric ID (from a distributed ID generator) to Base62
  - Example: ID 123456789 encodes to "8m0Kx" in Base62
- Key generation via hashing
  - Hash the original URL with MD5 or SHA-256, take the first N characters
  - Collision handling: on collision, append a salt and re-hash, or use a lookup to return existing key
- Key generation via distributed counter
  - Use ZooKeeper, Snowflake, or a database sequence for unique IDs
  - Avoids collisions entirely but requires coordination
  - Snowflake ID: 41-bit timestamp + 10-bit worker ID + 12-bit sequence = 64-bit unique ID
- Redirect types: 301 vs 302
  - 301 (Moved Permanently): cached by browsers and search engines; reduces load on the shortener
  - 302 (Found): not cached; every request hits the shortener, enabling per-request analytics
  - Recommendation: use 301 for production links, 302 when analytics accuracy is critical
- Storage options
  - RDBMS (PostgreSQL, MySQL): transactions, unique constraints on short code, easy custom alias dedup
  - NoSQL (DynamoDB, Cassandra): horizontal scaling, high write throughput for analytics
  - Hybrid: RDBMS for mapping table, NoSQL for analytics events
- Scaling strategies
  - Read replicas for PostgreSQL/MySQL to handle high read volume
  - Redis cache for hot URLs with TTL — cache hit serves redirect without database read
  - Cache-aside: on redirect request, check Redis, on miss read DB and populate cache
- Analytics pipeline
  - Capture click events: timestamp, IP, user-agent, referrer, geolocation
  - Write to Kafka or Kinesis, batch-insert into a time-series database (ClickHouse, Druid)
  - Separate analytics from the redirect path to avoid latency impact
- Custom aliases
  - User provides a custom short key (e.g., /mybrand)
  - Check uniqueness in the mapping table — if taken, return error
- Rate limiting
  - Per-user or per-IP limits on short URL creation to prevent spam
  - Token bucket or sliding window algorithm, return 429 Too Many Requests

## Common Mistakes

- **Using synchronous analytics writes on the redirect path**
  - Writing click events to a database on every redirect increases latency and can bring down the service under load
  - **Why it looks correct:** Storing data immediately seems simpler and guarantees no events are lost
  - Offload analytics to a message queue (Kafka, SQS) and process asynchronously; use 301 redirects for production traffic to reduce the number of events hitting your shortener
- **Not handling key collisions in hash-based generation**
  - When two different URLs hash to the same short code, one URL overwrites the other silently
  - **Why it looks correct:** Hash collisions are rare, so developers skip collision handling
  - Always check for collision before inserting; on collision, append a salt (timestamp, counter) and re-hash up to a limit
- **Choosing Base62 without considering key length tradeoffs**
  - Too short (4 chars = 14M keys) exhausts quickly; too long (10 chars) defeats the purpose
  - **Why it looks correct:** A fixed length seems simpler to implement
  - Use a 7-character key as the default (62^7 = ~3.5T); support variable-length keys for high-volume services

## Real-World Scenarios

### bit.ly Architecture

- Uses a distributed ID generator for unique keys
- 301 redirects for most links, 302 for real-time analytics
- Redis cache with LRU eviction for frequently accessed URLs
- Analytics pipeline with Kafka + Hadoop for batch processing

### TinyURL Approach

- Base62 encoding from a database sequence
- RDBMS-backed for consistency and custom alias support
- Simpler architecture suitable for moderate scale

## Use Cases

- **Link shortening for social media** — Twitter/X character limits, SMS marketing, or printed QR codes
  - Short URL reduces character count. Redirect via 301 (permanent) for caching efficiency or 302 (temporary) for analytics tracking.
  - **Avoid when:** the original URL is already short — shortening adds a redirect hop with no benefit.

- **Custom branded short links** — marketing campaigns where the short domain matches the brand (e.g., `go.company.com/sale`)
  - Custom aliases (e.g., `/blackfriday`) are readably memorable. The system reserves a namespace of human-readable short codes.
  - **Avoid when:** scale is massive (billions of URLs) — custom aliases cause hash collisions and require lookup tables.

- **Analytics and tracking** — measuring click-through rates, geographic distribution, and referrer data
  - Each redirect records IP, user-agent, referrer, and timestamp. Analytics pipeline (Kafka + batch processing) aggregates for dashboards.
  - **Avoid when:** privacy regulations (GDPR, CCPA) require minimal data collection — consider anonymized tracking or opt-out.

- **Rate-limited temporary links** — password reset links, email verification, or document sharing with expiration
  - Short URL configured with TTL and single-use semantics. The redirect logic checks expiration and usage count before serving the target URL.
  - **Avoid when:** the content is highly sensitive — the short URL's target is visible in the HTTP redirect; use signed URLs instead.

- **API response pagination cursors** — encoding opaque pagination tokens for API page navigation
  - Short hash encodes the last item ID and page size. The API decodes the cursor and queries the next page efficiently.
  - **Avoid when:** the API uses offset-based pagination — offset parameters are simpler and don't need encoding.

## Scenario-Based Questions

**Q: A viral short URL causes millions of requests per second. Your database is overwhelmed. How do you handle it?**

- Cache the mapping in Redis with a long TTL; a cache hit serves the redirect without touching the DB
- Use 301 redirects so browsers and CDNs cache the response, reducing requests to the shortener
- Add CDN caching (CloudFront, Cloudflare) at the DNS level for known hot URLs
- **Interview follow-up:** How do you handle the first request for a URL that is not yet cached when the DB is already saturated?

**Q: You need to generate 10,000 unique short codes per second. How do you design the key generation service?**

- Use Snowflake-style ID generator: each worker generates unique IDs without coordination (timestamp + worker ID + sequence)
- Pre-generate batches of IDs in memory to reduce database round trips
- Alternatively, use a Redis INCR counter to produce sequential IDs, then encode to Base62
- **Interview follow-up:** What happens if your distributed ID generator runs out of sequence numbers in a millisecond? How do you handle clock skew?

**Q: Your URL shortener uses a database sequence for key generation. At peak load, the sequence becomes a bottleneck and key generation latency spikes. How do you fix this?**

- Pre-allocate batches of IDs in each application instance — instead of calling the sequence per request, fetch a range of IDs (e.g., 10,000) and generate short codes locally
- Use a distributed ID generator like Snowflake that does not require database coordination per request
- Implement a key generation service that pre-generates batches of short codes and stores them in Redis for fast allocation
- **Interview follow-up:** How do you handle scenario where an application instance pre-allocates a range but crashes before using all the IDs — are those IDs wasted?

**Q: Your URL shortener needs to support custom aliases. A user wants "mybrand" but it is already taken by another user. How do you handle custom alias conflicts?**

- Check uniqueness in the mapping table with a unique constraint on the short code column
- If taken, return a meaningful error suggesting alternatives or additional characters
- Allow users to "reserve" aliases with a payment tier to prevent squatting
- **Interview follow-up:** How do you prevent alias squatting where users register many aliases but never use them?

**Q: Your URL shortener analytics pipeline shows that 30% of click events are lost during traffic spikes. What is the likely cause and how do you fix it?**

- Click events are likely written synchronously or through a queue that drops messages when full
- Switch to a fire-and-forget approach: write events to a highly available message queue (Kafka, Kinesis) with at-least-once delivery
- Buffer events in memory on the application server and flush in batches to the queue every second
- **Interview follow-up:** How do you handle duplicate analytics events from retries when using at-least-once delivery?

**Q: Your URL shortener uses 7-character Base62 keys. You are running out of available keys due to high creation volume. What is the migration strategy?**

- Extend the key length incrementally: support variable-length keys so new keys use 8 characters while existing 7-char keys continue working
- Implement a two-phase migration: first, update the database and application to support 8-char keys, then start generating new keys at the longer length
- Consider using a larger alphabet (Base64) or a hash-based approach to maximize the key space
- **Interview follow-up:** How do you handle users who have bookmarked old 7-character URLs after the migration?

**Q: Malicious users are creating short URLs that redirect to phishing sites. How do you detect and prevent this?**

- Scan target URLs at creation time using a URL reputation service (Google Safe Browsing, PhishTank)
- Implement a reporting mechanism for users to flag malicious short links
- Add a moderation queue: suspicious URLs (flagged by automated scanning) go to manual review before activation
- **Interview follow-up:** How do you handle the case where a legitimate URL is mistakenly flagged as malicious?

**Q: Your URL shortener has users in the US and Europe. Users in Europe experience higher latency for redirects. How do you optimize this?**

- Deploy the redirect service in multiple regions (US and EU) with a global load balancer (DNS-based or anycast)
- Use geo-routing to direct users to the nearest region
- Replicate the URL mapping database across regions using cross-region replication or a global database (Spanner, CockroachDB)
- **Interview follow-up:** How do you handle consistency for custom alias creation when users can create aliases from any region?

**Q: A customer wants to track click-through rates for their marketing campaigns using your URL shortener. How do you provide real-time analytics?**

- Aggregate click counters in Redis using INCR per short URL per minute for real-time dashboards
- Flush aggregated counters to persistent storage (ClickHouse, BigQuery) every few minutes for historical queries
- Use approximate data structures (HyperLogLog) for unique click counts and counters for total clicks
- **Interview follow-up:** How do you prevent real-time analytics from impacting the redirect latency?

**Q: Your URL shortener API allows programmatic creation of short URLs. A user floods the API with 1 million requests per minute, creating spam links. How do you protect the system?**

- Implement rate limiting per user and per IP: token bucket algorithm with limits suited for URL creation (e.g., 1000 URLs per hour per user)
- Require API keys for programmatic access and enforce per-key limits
- Add CAPTCHA for anonymous creation and require email verification for new accounts
- **Interview follow-up:** How do you distinguish between a legitimate power user and an abusive user for rate limiting purposes?

## Interview Questions

- **Explain the difference between 301 and 302 redirects and when to use each.**
  - 301 is permanent, cached by browsers and search engines, reducing load on the shortener. 302 is temporary, every request reaches the shortener. Use 301 for production links to minimize latency; use 302 when you need per-request analytics or the target URL changes frequently.
- **How do you handle key collisions in a hash-based URL shortener?**
  - On hash generation, check if the key already exists. If it exists and maps to the same URL, return the existing key. If it maps to a different URL, append a salt (timestamp or counter) and re-hash. Repeat up to a configurable limit, then fall back to a longer hash or distributed ID approach.
- **Design the analytics pipeline for a URL shortener handling 1M clicks/second.**
  - Use a fire-and-forget approach: write click events to Kafka asynchronously from the redirect handler. Kafka consumers batch-write to a columnar store (ClickHouse, BigQuery) for analytics queries. Aggregate counters in Redis (INCR per URL per minute) for real-time dashboards, with periodic flushes to persistent storage.
- **How would you scale the database for a global URL shortener?**
  - Use read replicas in each region for the mapping table. Shard the mapping table by short code's hash modulo N. Cache hot entries in regional Redis clusters with TTL. Use a distributed database (Spanner, CockroachDB) for multi-region writes if custom aliases need global consistency.
- **What is the advantage of Base62 encoding over Base64 for URL shorteners?**
  - Base62 uses only alphanumeric characters (a-z, A-Z, 0-9), avoiding URL-unsafe characters like + and / that appear in Base64. Base62 keys are safe to use directly in URLs without percent-encoding, making them shorter and cleaner.
- **How do you handle expired or unused short URLs?**
  - Implement a TTL policy: short URLs that have not been accessed in N months can be marked as expired. Return a 410 Gone for expired URLs. Recycle expired keys after a grace period, or archive them with a redirection to a notice page.
- **Explain the tradeoff between hash-based and counter-based key generation.**
  - Hash-based: deterministic from the URL, no coordination needed, but collisions possible and the same URL always gets the same short code. Counter-based: sequential IDs encoded to Base62, no collisions, but requires a coordination service for distributed generation and different URLs always get different codes.
- **How do you implement a custom domain for a URL shortener (branded short domain)?**
  - Allow users to configure a custom domain (e.g., go.acme.com) that CNAMEs to the shortener's infrastructure. Store the domain-to-account mapping in a DNS-level config. The shortener routes requests based on the Host header to the correct account's URL mappings.
- **What storage considerations matter for a URL shortener?**
  - The mapping table is read-heavy (redirects far outnumber creations). Use a fast read-optimized store like Redis cache with a database backing store. The analytics table is write-heavy with append-only patterns — use a time-series database or columnar store.
- **How do you handle URL redirection for mobile deep links?**
  - Detect the user-agent and device type on the redirect endpoint. If the URL corresponds to a mobile app deep link, return a 302 redirect to the app's custom scheme (myapp://path) or use a universal link (iOS) / app link (Android) with the appropriate HTTP headers.
- **What is the role of CDN in a URL shortener architecture?**
  - CDN can cache the 301/302 redirect responses for popular URLs, absorbing traffic at the edge and reducing load on the origin shortener. CDN also accelerates static assets (analytics tracking pixel, landing pages) and provides DDoS protection.
- **How do you ensure the URL shortener is highly available?**
  - Deploy across multiple availability zones with a load balancer. Use a multi-region active-active setup with DNS failover. The mapping database should have read replicas and automated failover. Cache aggressively to survive database outages.
- **Explain the concept of "key collision" in a distributed URL shortener.**
  - Two different applications or processes may generate the same short code simultaneously. Prevention: use atomic ID generation (database sequence, Redis INCR) or distributed ID generators (Snowflake). Detection: always check for existing key before inserting (unique constraint + retry on conflict).
- **How do you perform A/B testing with a URL shortener?**
  - Support multiple target URLs per short code with traffic distribution weights. When a redirect request arrives, the shortener selects a target based on the configured split (50/50, 90/10). Track which variant each user sees using a cookie or redirect parameter.
- **What is the difference between a 301 and 302 redirect for SEO?**
  - 301 passes link equity (PageRank) to the target URL and is cached by search engines permanently. 302 does not pass link equity and is not cached. For permanent URL shorteners, use 301. For temporary campaigns or A/B testing, use 302 to avoid permanently assigning ranking to one target.
- **How do you handle special characters in long URLs?**
  - The long URL should be normalized before storage: decode percent-encoded characters, lowercase the domain, remove default ports, and sort query parameters. Always validate the URL format and reject malformed URLs. Store the normalized form to ensure consistent short code generation for the same logical URL.
- **What is a "preview page" and why might you implement one?**
  - A preview page shows the target URL and asks the user to confirm before redirecting. This protects users from malicious or unexpected destinations. It is commonly used for link safety, but adds friction. Implement it as optional per-user or per-link configuration.
- **How do you estimate storage requirements for a URL shortener handling 1B URLs?**
  - Each mapping record: short code (7 bytes) + target URL (avg 200 bytes) + metadata (50 bytes) ≈ 260 bytes. For 1B URLs: 260 GB. Add indexes (short code unique index, created_at index for purging) and analytics data. Total storage ≈ 500 GB–1 TB, which is manageable on a single large database but should be sharded for performance.
- **How do you implement click fraud detection for analytics?**
  - Detect rapid repeated clicks from the same IP or user-agent within a short window — these are likely bots or automated scripts. Use per-IP rate limiting for analytics events. Track unusual patterns (1000 clicks in 1 second from a single referrer) and exclude them from analytics reports.
- **How do you handle URL shortener redirection for QR codes printed on physical materials?**
  - QR codes are permanent once printed, so the short URLs they encode must never break. Use 301 redirects (permanent) for QR code targets. Never change or delete these mappings. Consider a dedicated prefix (qr.yourdomain.com) with a promise of permanent resolution and an SLA for uptime.

## Developer Recommendations

- **Decouple the redirect path from analytics**
  - Analytics writes should never block the redirect response; use async queues and batch processing
  - Use 301 redirects for the hot path, with a small percentage of 302 redirects for sampling analytics
  - **Production story:** A URL shortener crashed during a Super Bowl ad because synchronous analytics writes overwhelmed the DB — switching to Kafka + async writes fixed it without data loss
- **Cache aggressively on the redirect path**
  - Redis cluster with at least 24-hour TTL for URL mappings; CDN for the most popular URLs
  - Cache-aside pattern: serve from Redis, populate on miss from DB, set TTL
  - **Production story:** Caching reduced DB reads by 95% for a shortener handling 500M redirects/day
- **Pre-allocate key batches for high-throughput creation**
  - Reduce ID generation latency by pre-fetching ranges of IDs into application memory
  - Each application instance claims a range (e.g., 10,000–20,000) and generates short codes locally
  - **Production story:** Generating IDs one-at-a-time via DB sequence became a bottleneck at 2K writes/second — batch allocation eliminated the contention
- **Always validate short URLs for safety**
  - Scan target URLs for phishing, malware, or prohibited content before creating the mapping
  - Implement a reporting mechanism for users to flag malicious short links
