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

## Interview Questions

- **Explain the difference between 301 and 302 redirects and when to use each.**
  - 301 is permanent, cached by browsers and search engines, reducing load on the shortener. 302 is temporary, every request reaches the shortener. Use 301 for production links to minimize latency; use 302 when you need per-request analytics or the target URL changes frequently.
- **How do you handle key collisions in a hash-based URL shortener?**
  - On hash generation, check if the key already exists. If it exists and maps to the same URL, return the existing key. If it maps to a different URL, append a salt (timestamp or counter) and re-hash. Repeat up to a configurable limit, then fall back to a longer hash or distributed ID approach.
- **Design the analytics pipeline for a URL shortener handling 1M clicks/second.**
  - Use a fire-and-forget approach: write click events to Kafka asynchronously from the redirect handler. Kafka consumers batch-write to a columnar store (ClickHouse, BigQuery) for analytics queries. Aggregate counters in Redis (INCR per URL per minute) for real-time dashboards, with periodic flushes to persistent storage.
- **How would you scale the database for a global URL shortener?**
  - Use read replicas in each region for the mapping table. Shard the mapping table by short code's hash modulo N. Cache hot entries in regional Redis clusters with TTL. Use a distributed database (Spanner, CockroachDB) for multi-region writes if custom aliases need global consistency.

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
