# Microservices and System Design Questions

## Questions

1. What is microservices architecture?
2. Difference between monolith and microservices.
3. Advantages of microservices.
4. Disadvantages of microservices.
5. How do services communicate?
6. REST vs messaging.
7. What is API gateway?
8. What is service discovery?
9. What is circuit breaker?
10. What is retry pattern?
11. What is timeout?
12. What is bulkhead pattern?
13. What is rate limiting?
14. What is load balancing?
15. What is distributed tracing?
16. What is centralized logging?
17. What is correlation id?
18. What is eventual consistency?
19. What is distributed transaction?
20. What is saga pattern?
21. What is CQRS?
22. What is event-driven architecture?
23. What is idempotency?
24. Why is idempotency important?
25. How do you design idempotent APIs?
26. How do you handle duplicate requests?
27. How do you handle API versioning?
28. How do you design pagination?
29. How do you design search APIs?
30. How do you design notification system?
31. How do you design rate limiter?
32. How do you design URL shortener?
33. How do you design distributed cache?
34. How do you design chat system?
35. How do you design inventory system?
36. How do you design IoT monitoring system?
37. How do you design real-time dashboard?
38. How do you handle high throughput ingestion?
39. How do you handle data consistency?
40. How do you handle failure in downstream services?
41. How do you choose between sync and async communication?
42. How do you secure microservices?
43. How do you monitor microservices?
44. What metrics would you track for backend services?
45. What is SLA?
46. What is SLO?
47. What is error budget?
48. How do you design for scalability?
49. How do you design for reliability?
50. How do you design for maintainability?

---

## Answers

1. What is microservices architecture?
   - **Answer:**
      - Microservices architecture breaks an application into small, independently deployable services that communicate over the network, each owning its own data and logic
      - Separate services for ingestion, alerting, and dashboard could each be deployed independently without affecting others
      - A system could be refactored into microservices: a sensor-ingestion service writing to InfluxDB for time-series data, an alert-engine service evaluating temperature thresholds, and a dashboard-api service querying PostgreSQL for configuration, each with its own database and deployable independently
2. Difference between monolith and microservices.
   - **Answer:**
      - A monolith is a single codebase deployed as one unit, while microservices split into separate deployable services
      - A monolithic ETL and reporting system in one codebase limits scaling flexibility; microservices would let ETL and reporting scale independently
      - Tradeoffs: monolith is simpler for small teams but limits independent scaling; microservices add communication complexity (service mesh, retries, circuit breakers) and data consistency challenges (eventual consistency, saga patterns) but enable scaling, deploying, and evolving services independently
3. Advantages of microservices.
   - **Answer:**
      - Microservices enable independent deployment, scaling, and technology diversity
      - If validation logic in an inventory system needed more resources, only that service could be scaled instead of the entire application
      - Microservices improve fault isolation - one service failing does not bring down the whole system, critical for IoT monitoring where alerting must stay up even if the dashboard has issues
4. Disadvantages of microservices.
   - **Answer:**
      - The main drawbacks are distributed complexity, network latency, data consistency challenges, and operational overhead
      - Debugging across Kafka, Spring Boot APIs, and InfluxDB is harder than in a monolithic application
      - Microservices need mature DevOps (CI/CD pipelines, container orchestration, infrastructure as code), monitoring (distributed tracing, centralized logging, alerting), and team coordination (API contracts, shared libraries, clear ownership boundaries) to be effective
      - Without proper tooling, tracing a single failed data flow across Lambda, Kafka, and the API layer takes significantly longer than in a monolith
5. How do services communicate?
   - **Answer:**
      - Services communicate either synchronously via REST/gRPC or asynchronously via message queues like Kafka
      - IoT data can flow asynchronously through Kafka from Lambda to Spring Boot, while dashboard queries are synchronous REST calls to the API layer
      - The choice depends on the use case: REST for request-response with low latency needs where the caller needs an immediate answer, Kafka for high-throughput decoupled processing where the producer does not need an immediate answer and can tolerate eventual delivery
6. REST vs messaging.
   - **Answer:**
      - REST is synchronous, simpler, but creates tight coupling
      - Messaging with Kafka decouples producers from consumers and buffers data
      - Both can be used together: Kafka for sensor data ingestion and REST for dashboard API queries
      - REST is suited for CRUD operations where the client needs an immediate response (querying sensor data for display), while messaging is suited for event-driven workflows like ETL pipelines or real-time data where throughput matters more than immediate reply (ingesting thousands of sensor readings per second)
7. What is API gateway?
   - **Answer:**
      - An API gateway is a single entry point that routes requests to backend services, handling authentication, rate limiting, and aggregation
      - A Spring Boot API layer can act as a mini-gateway for a React dashboard, consolidating sensor data from InfluxDB and alerts from the alert engine
      - Gateways also handle request transformation (format conversion between client and service), circuit breaking (protecting backend services from overload), and hiding service decomposition from frontend consumers so the client talks to one URL instead of discovering individual service endpoints
8. What is service discovery?
   - **Answer:**
      - Service discovery allows services to find each other dynamically without hardcoded addresses
      - In EC2-based deployments, environment variables for service URLs are common, but in a true microservices setup, tools like Eureka or Kubernetes DNS handle this automatically
      - Client-side discovery: the client queries a service registry (Eureka) and load-balances across available instances itself. Server-side discovery: a load balancer or API gateway maintains the registry and routes requests, like AWS ALB which routes to healthy EC2 instances in target groups based on health checks, acting as a discovery mechanism without the client needing to know individual IPs
9. What is circuit breaker?
   - **Answer:**
      - Circuit breaker prevents cascading failures by stopping calls to a failing service and failing fast
      - If a validation database is slow, a circuit breaker would stop hitting it repeatedly and return a cached response until it recovered
      - The three states: **closed** (normal operation, requests pass through, failures are counted), **open** (threshold breached, all requests fail fast immediately, no calls to the failing service), **half-open** (after a cooldown period, a limited number of test requests pass through to check if the service recovered - if they succeed, circuit closes; if they fail, it opens again). In Spring Boot, Resilience4j configures this with properties like `failureRateThreshold`, `waitDurationInOpenState`, and `slidingWindowSize`
10. What is retry pattern?
    - **Answer:**
      - Retry pattern automatically re-attempts a failed operation, usually with exponential backoff
      - If a downstream API call fails temporarily, retry with backoff can retry up to 3 times before logging a failure
      - Retry handles transient failures (network blips, temporary overload) while circuit breaker protects against sustained failures (service is down for minutes). They work best together: retry handles brief hiccups, and if retries keep failing, the circuit breaker trips and stops overwhelming the failing service entirely
11. What is timeout?
    - **Answer:**
      - Timeout sets a maximum wait time for an operation, preventing threads from hanging indefinitely
      - Connection and read timeouts can be configured on REST calls to ensure the dashboard never waits more than 5 seconds
      - Three types: **connection timeout** (max time to establish TCP connection, typically 3-5 seconds), **read timeout** (max time to wait for a response after connection is established, varies by operation), **write timeout** (max time to send request body). In Spring Boot, RestTemplate configures these via `setConnectTimeout` and `setReadTimeout` on the HttpComponentsClientHttpRequestFactory, while WebClient uses `HttpClient` with `responseTimeout(Duration.ofSeconds(5))`
12. What is bulkhead pattern?
    - **Answer:**
      - Bulkhead isolates resources into separate pools so failure in one does not deplete resources for others
      - Separate thread pools for real-time sensor processing and dashboard queries ensure a slow dashboard query never blocks sensor ingestion
      - Thread pool isolation: each dependency gets its own fixed-size thread pool (e.g., 20 threads for alerting, 10 for reporting), so a slow reporting query cannot exhaust alerting threads. Semaphore isolation: uses a count-based limiter without dedicated threads, lighter weight but does not provide timeout enforcement. In Resilience4j, `ThreadPoolBulkhead` provides thread pool isolation and `SemaphoreBulkhead` provides semaphore isolation. Critical paths like alerting should use dedicated pools isolated from less critical paths like reporting
13. What is rate limiting?
    - **Answer:**
      - Rate limiting controls how many requests a client can make in a time window, protecting backend resources from overload
      - Token bucket and sliding window algorithms are useful for public APIs to prevent abuse
      - Implementation at the API gateway level uses Redis for distributed rate counting: each request increments a counter in Redis with a TTL, and different client tiers (free, standard, premium) get different limits. For example, free tier allows 100 requests/minute, premium allows 1000. Redis sorted sets enable sliding window by storing timestamps and counting entries within the window
14. What is load balancing?
    - **Answer:**
      - Load balancing distributes traffic across multiple servers to improve availability and throughput
      - AWS Application Load Balancer can distribute requests across EC2 instances for Spring Boot APIs serving a dashboard
      - Round-robin distributes requests sequentially to each server in turn, simplest approach when servers have similar capacity. Least-connections routes to the server with fewest active connections, better when request processing time varies. AWS ALB runs health checks (HTTP 200 on `/actuator/health`) at configurable intervals, and automatically removes unhealthy EC2 instances from the target group, routing traffic only to healthy ones
15. What is distributed tracing?
    - **Answer:**
      - Distributed tracing tracks a request as it flows through multiple services, helping debug latency and failures
      - A correlation ID can help trace a sensor reading across the entire flow from Lambda to Kafka to Spring Boot
      - Tools like Jaeger and Zipkin use **trace IDs** (unique per request, generated at the entry point) and **span IDs** (unique per service hop, representing one unit of work). A trace is composed of multiple spans forming a tree. Spring Cloud Sleuth automatically injects trace IDs into MDC logs and propagates them via HTTP headers (`X-B3-TraceId`), so searching one trace ID in Jaeger shows the full request flow with latency breakdown per service
16. What is centralized logging?
    - **Answer:**
      - Centralized logging aggregates logs from all services into a single searchable platform
      - CloudWatch Logs can collect and search logs from all Spring Boot EC2 instances without SSHing into each server individually
      - Log levels should follow a hierarchy (ERROR for failures, WARN for recoverable issues, INFO for key business events, DEBUG for development troubleshooting). Structured logging in JSON format (using Logback with `LogstashEncoder`) makes logs machine-parseable. Centralized logging correlates errors across services using the trace ID from distributed tracing, so filtering one trace ID shows every log entry across all services for that single request
17. What is correlation id?
    - **Answer:**
      - A correlation ID is a unique identifier attached to a request as it passes through multiple services, linking all related log entries
      - Each sensor reading can carry a UUID from ingestion to dashboard, making end-to-end tracing possible
      - The correlation ID is generated at the API gateway or entry point as a UUID, then propagated through Kafka message headers (`correlationId` header), HTTP request headers, and injected into MDC (Mapped Diagnostic Context) for all log statements. Every log line includes this ID, so searching for one correlation ID in CloudWatch or Elasticsearch shows every log entry across all services for that single request
18. What is eventual consistency?
    - **Answer:**
      - Eventual consistency means that after a write, all replicas will converge to the same state given enough time without new updates
      - Partner submissions can be eventually consistent across reporting views - not instant, but guaranteed within seconds
      - Strong consistency (linearizability) guarantees every read returns the most recent write, but requires coordination (locks, quorum reads) that adds latency and reduces availability. Eventual consistency improves availability and performance since writes can return immediately and replicas sync asynchronously. The tradeoff: application logic must handle stale reads gracefully - showing slightly outdated data is acceptable for dashboards, but not for financial reconciliation, which is why database transactions are used for inventory calculations
19. What is distributed transaction?
    - **Answer:**
      - A distributed transaction spans multiple services or databases, requiring coordination for all-or-nothing execution
      - Partner submissions can involve validating records, updating baseline, and calculating inventory - all coordinated in MSSQL transactions at the database level
      - Two-phase commit (2PC) coordinates distributed transactions: phase 1 asks all participants to prepare (vote yes/no), phase 2 commits or rolls back all. Drawbacks: the coordinator is a single point of failure, prepare phase locks resources reducing availability, and network partitions can block the entire system. Modern systems prefer eventual consistency with saga patterns over 2PC because sagas avoid global locks and work better across service boundaries
20. What is saga pattern?
    - **Answer:**
      - Saga is a sequence of local transactions where each step publishes an event to trigger the next, with compensating transactions for rollback
      - If a distributed inventory flow spans services, a saga would ensure validation rollback if baseline update failed after validation succeeded
      - **Choreography sagas**: each service listens for events and decides what to do next (validation service publishes `ValidationComplete`, baseline service listens and publishes `BaselineUpdated`, inventory service listens and updates). Simple, no central coordinator, but hard to track overall flow as complexity grows. **Orchestration sagas**: a central coordinator tells each service what to do and handles compensating actions. Easier to understand and debug complex flows, but adds a single coordination point. Choose choreography for 2-3 step flows with simple logic, orchestration for complex multi-step workflows requiring visibility
21. What is CQRS?
    - **Answer:**
      - CQRS separates read and write operations into different models, optimizing each for its purpose
      - Writes can go to Kafka/InfluxDB for high-throughput ingestion, while reads use optimized InfluxDB queries and Redis caching for fast dashboard rendering
      - CQRS is useful when read and write workloads differ significantly - IoT writes are high-volume append-only (thousands of sensor readings per second), while reads are aggregation queries with time range filters (average temperature over last hour). The write model optimizes for throughput (Kafka batching, InfluxDB append), while the read model optimizes for query speed (pre-aggregated views, Redis caching, denormalized read tables). This separation allows independent scaling of read and write paths
22. What is event-driven architecture?
    - **Answer:**
      - Event-driven architecture uses events to trigger and communicate between decoupled services
      - An IoT monitoring system can be fully event-driven: IoT data triggers Lambda, which publishes to Kafka, which triggers Spring Boot processing, which stores to InfluxDB and triggers dashboard updates
      - **Event sourcing** stores every state change as an immutable event in a log (Kafka topics), making the system fully replayable and auditable - state can be rebuilt by replaying events from the beginning. **Event notification** is simpler: services publish events to signal something happened, but the event is not the source of truth. Kafka's log-based approach provides event sourcing benefits naturally - sensor data can be replayed to reprocess historical periods or debug data pipeline issues
23. What is idempotency?
    - **Answer:**
      - Idempotency means performing the same operation multiple times produces the same result as doing it once
      - If a partner submits the same serial record twice, the validation engine can identify it as duplicate using a unique constraint on the serial number, preventing double-counting
      - Idempotency keys work by having the client generate a unique key (UUID) for each logical operation and send it in the request header. The server stores the key along with the response in a cache (Redis with TTL matching the operation's relevance window). On subsequent requests with the same key, the server returns the cached response without re-executing the operation, ensuring network retries or client mistakes do not cause duplicate processing
24. Why is idempotency important?
    - **Answer:**
      - Idempotency prevents data corruption from duplicate requests caused by network retries or client errors
      - Without idempotency, accidental duplicate submissions could inflate inventory counts, causing reconciliation issues and financial discrepancies
      - Idempotency is critical for payment-like operations (charging a card twice), inventory updates (double-counting serial records), and any state-changing operation where even a single duplicate has significant business impact. Network-level retries (at TCP, HTTP, or load balancer layers) can silently duplicate requests, making idempotency a necessity rather than a nice-to-have for production systems
25. How do you design idempotent APIs?
    - **Answer:**
      - Use an idempotency key pattern: the client sends a unique key, the server checks if it has already processed that key and returns the cached response
      - A batch submission ID can serve as the idempotency key
      - Storage options: Redis with TTL (simple, fast, automatic expiration of old keys) or a database table (durable, queryable for auditing). TTL should match the operation's relevance window - 24 hours for most operations. For concurrent requests with the same key: use Redis `SETNX` (set-if-not-exists) with a lock value to ensure only one request processes, while others wait or receive the in-progress response. Return the same HTTP status code and response body for duplicate requests
26. How do you handle duplicate requests?
    - **Answer:**
      - For naturally idempotent operations like GET or PUT, no extra handling is needed
      - For non-idempotent operations like inventory submission, a unique constraint on batch reference can reject duplicates at the database level with appropriate error messaging
      - Duplicate detection should use a time window - a duplicate with the same data arriving after a week might be a legitimate new request, so TTL-based deduplication is preferred. Implement with a deduplication table storing (request_key, timestamp, response) with a TTL index. The time window depends on business logic: 24 hours for API submissions, 5 minutes for payment webhooks, or configurable per endpoint
27. How do you handle API versioning?
    - **Answer:**
      - URL-based versioning (`/api/v1/resource`) is explicit and easy to route at the load balancer level
      - For internal projects with controlled API consumers, aggressive versioning may not be needed, but for public APIs this approach is recommended
      - Backward compatibility principles: never remove or rename fields clients depend on, add new fields as optional (do not make them required), deprecate old versions with a documented migration timeline in Swagger/OpenAPI specs. Use `Sunset` HTTP header to signal deprecation dates. Support at most 2 concurrent versions, and communicate breaking changes to consumers well in advance
28. How do you design pagination?
    - **Answer:**
      - Cursor-based pagination with a last-seen ID or timestamp works best for large or real-time datasets because offset-based pagination can be inconsistent when data changes
      - Offset-based is fine for static lists, but for sensor readings, cursor-based prevents duplicates during live streaming
      - Page size limits: enforce a max (e.g., 100 records) to prevent oversized responses, with a default of 20. Total count is expensive for large datasets - cache it in Redis with a TTL instead of counting on every request. For cursor-based, return `hasNext` boolean and the next cursor value, skip the total count unless the client specifically requests it. Offset-based still works for admin dashboards with static, small datasets where simplicity matters more
29. How do you design search APIs?
    - **Answer:**
      - Keep search APIs simple with query parameters for filtering, sorting, and pagination
      - Search endpoints can accept date range, partner ID, and product category as query parameters, with WHERE clauses built dynamically using Spring Data JPA Specifications
      - For full-text search (searching across unstructured text fields), use Elasticsearch with inverted indexes for fast text matching. Index frequently filtered columns in the relational database (partner ID, product category, date range) to avoid full table scans. Prevent SQL injection by using parameterized queries exclusively - JPA Specifications with `CriteriaBuilder` or MyBatis with `#{param}` bind variables, never string concatenation
30. How do you design notification system?
    - **Answer:**
      - Design it as an event-driven system: notification events published to a Kafka topic, consumers process them and send via email/SMS/push, with a database storing delivery status
      - Excursion alerts are notifications triggered when temperature exceeds thresholds
      - Delivery guarantees: at-least-once delivery with deduplication (idempotency keys on notification ID) ensures no notifications are lost. Retry mechanisms with exponential backoff handle transient failures in email/SMS providers (3 retries over 15 minutes). Rate limiting prevents flooding recipients during multi-sensor excursions - if 50 sensors breach simultaneously, batch alerts per recipient into a single digest instead of sending 50 separate notifications
31. How do you design rate limiter?
    - **Answer:**
      - A sliding window counter stored in Redis tracks request counts per client per time window using sorted sets
      - The API gateway would check eligibility before forwarding each request, returning 429 Too Many Requests when the limit is exceeded
      - Token bucket algorithm is an alternative: tokens refill at a fixed rate, each request consumes one token, allowing bursts up to bucket size while maintaining average rate. Distributed rate limiting challenges: clock skew between Redis nodes can cause inconsistent window boundaries (mitigated by using Redis server time), and Redis being a single point of failure (mitigated by Redis Sentinel or Cluster for HA). Use `XADD` with `MAXLEN` for Redis streams-based sliding window to avoid unbounded memory growth
32. How do you design URL shortener?
    - **Answer:**
      - Use a hash-based approach: generate a unique short key from a hash of the original URL, store the mapping in a database with Redis caching, and return a 302 redirect on lookup
      - Collision handling: if two URLs produce the same hash, append a salt (counter or random string) and re-hash until unique, or use a collision counter suffix (e.g., `abc123-2`). Key length optimization: Base62 encoding of an auto-incrementing ID produces 7-character keys supporting billions of URLs (62^7 = 3.5 trillion). Analytics: increment a Redis counter on each redirect and periodically flush counts to the database for click tracking, avoiding write contention on the main redirect table
33. How do you design distributed cache?
    - **Answer:**
      - Use Redis with a cache-aside pattern: check Redis first, if not found, fetch from the database and store in Redis with a TTL
      - Frequently accessed sensor metadata and dashboard configurations can be cached in Redis to reduce database load
      - Consistent hashing for Redis cluster sharding: hash both keys and nodes onto a ring, so adding/removing a node only remaps a fraction of keys (not all). Cache stampede: when a popular key expires, multiple requests simultaneously hit the database. Prevent with early expiration (recompute at 80% TTL in background) or distributed locking (only one request repopulates, others wait). Monitor cache hit/miss ratio - below 80% suggests TTLs are too short or cache size is insufficient
34. How do you design chat system?
    - **Answer:**
      - Design with WebSocket connections for real-time messaging, Kafka for message persistence and ordering, and a database for message history partitioned by conversation ID
      - Presence detection: client sends heartbeat pings every 30 seconds, server marks user offline after missing 2 consecutive heartbeats, other clients see status change via WebSocket push. Message delivery guarantees: persist to Kafka before acknowledging to sender, consumers track read offsets to ensure at-least-once delivery with deduplication on message ID. Scaling WebSocket servers: use sticky session load balancers (ALB with stickiness enabled) so a client's connection stays on the same server, enabling in-memory pub/sub between connections on the same instance, with Redis Pub/Sub for cross-instance message broadcast
35. How do you design inventory system?
    - **Answer:**
      - Design with a centralized validation engine that receives serial records from partners, validates against a verified baseline, and calculates real-time inventory
      - MSSQL stored procedures can validate 10,000+ records from 200+ partners in under 15 seconds
      - Exact flow: partner submits a batch via REST API with a batch reference ID (used as idempotency key). Validation engine checks each record for duplicates within the batch, valid serial format, and existence against the verified baseline. Accepted records update the inventory count in a transaction, rejected records are flagged with rejection reason (duplicate, invalid format, not in baseline) and stored for reconciliation. The partner receives a response with accepted/rejected counts, and can query rejected records to fix and resubmit
36. How do you design IoT monitoring system?
    - **Answer:**
      - Design with a scalable ingestion pipeline: IoT gateways send MQTT to AWS IoT Core, which triggers Lambda to publish to Kafka, Spring Boot consumers process and store time-series data in InfluxDB, and Grafana/React dashboards query the APIs
      - This is a common architecture for IoT monitoring scenarios
      - Technology rationale: MQTT is lightweight and designed for constrained IoT devices with small packet sizes and QoS levels. Kafka buffers bursts from hundreds of gateways simultaneously sending data, preventing consumer overload. InfluxDB is purpose-built for time-series data with native support for downsampling, retention policies, and efficient range queries on timestamp-indexed data. This combination handles high throughput requirements while keeping each layer independently scalable
37. How do you design real-time dashboard?
    - **Answer:**
      - Design with polling using React setInterval, backed by optimized InfluxDB queries and Redis caching
      - Polling the Spring Boot API every 30 seconds for sensor readings, with Redis caching frequently accessed data points, is a practical approach
      - WebSocket vs polling tradeoffs: WebSocket provides true real-time push but adds connection management complexity (reconnection logic, scaling challenges, sticky sessions). Polling is simpler, works through proxies/firewalls, and 30-second intervals are sufficient for temperature monitoring use cases. Data aggregation: pre-aggregate raw readings into 1-minute averages at the consumer level to reduce dashboard query payload size. Lazy loading: load last 24 hours by default, fetch historical data on-demand when user selects a custom date range
38. How do you handle high throughput ingestion?
    - **Answer:**
      - Use a message queue like Kafka as a buffer between producers and consumers
      - When hundreds of IoT gateways send data simultaneously, Kafka absorbs the burst, and consumers process at their own pace without data loss
      - Partition count planning: start with partitions equal to the max expected consumer parallelism (e.g., 12 partitions for 12 consumer instances), since partitions define the maximum concurrency. Consumer group scaling: add consumers up to the partition count, each consumer handles a subset of partitions. Monitoring consumer lag (Kafka `__consumer_offsets` topic) detects processing bottlenecks early - if lag grows consistently, consumers need more resources or partitions need to increase. Kafka's log compaction and retention policies manage storage without losing recent data
39. How do you handle data consistency?
    - **Answer:**
      - For same-database consistency, ACID transactions are the standard approach
      - For cross-service consistency, eventual consistency with idempotent operations is used
      - An ETL pipeline can use database transactions to ensure partial failures do not leave reporting data inconsistent
      - CAP theorem tradeoffs: in distributed systems, during a network partition, you choose between consistency (CP - reject writes until partition heals) and availability (AP - accept writes and resolve conflicts later). For IoT monitoring systems where availability is paramount - sensor data must always be ingested, even if some services are temporarily unreachable, AP with eventual consistency is the right choice. Design APIs to handle this: return stale data with a timestamp, use version vectors for conflict detection, and implement compensating actions for business-critical operations
40. How do you handle failure in downstream services?
    - **Answer:**
      - Use timeouts, retries with exponential backoff, circuit breakers, and fallback mechanisms
      - If InfluxDB is slow, returning stale data from Redis cache as a fallback instead of showing an error provides a better user experience
      - Graceful degradation means the system partially works even when dependencies fail - a dashboard showing cached sensor data with a "last updated" timestamp is better than a blank error page. Set up alerting on fallback path activation (CloudWatch alarm when Redis fallback is used more than 10% of requests) to detect degraded state before users complain. Design fallbacks in priority order: cached data > default values > partial results > clear error message
41. How do you choose between sync and async communication?
    - **Answer:**
      - Choose synchronous (REST) when the client needs an immediate response and the operation is quick
      - Choose asynchronous (Kafka) when high throughput, decoupling, or background processing is needed
      - In IoT systems, sensor ingestion is typically async while dashboard queries are sync
      - Async communication introduces complexity: error handling requires dead-letter queues and compensating logic, tracing requires correlation ID propagation through message headers, and debugging requires tools that understand message flow (not just request-response). The decision should consider whether the client can tolerate delayed responses - if the user clicks "submit" and needs confirmation, sync is appropriate; if the operation can complete in the background with a status check later, async reduces coupling and improves resilience
42. How do you secure microservices?
    - **Answer:**
      - Secure microservices with JWT-based authentication, role-based access control, HTTPS, API gateway as a security barrier, and input validation at every service boundary
      - Spring Security with JWT tokens can be used for API authentication and CORS can be configured for the React frontend
      - OAuth2 for delegated authorization: third-party clients get limited-access tokens without handling user credentials (e.g., a partner system accesses only their inventory data via OAuth2 client credentials flow). Service-to-service authentication uses mutual TLS (mTLS): each service presents a certificate, both client and server verify each other, preventing unauthorized services from calling internal APIs. Secrets management: store API keys, database passwords, and certificates in environment variables or AWS Secrets Manager, never in code or configuration files committed to Git
43. How do you monitor microservices?
    - **Answer:**
      - Monitor with health check endpoints, metrics (request rate, latency, error rate, resource usage), centralized logging, and distributed tracing
      - Spring Boot Actuator, CloudWatch for metrics and logs, and Grafana dashboards for application-level monitoring are common choices
      - The three pillars of observability: **logging** answers "what happened" (structured JSON logs with trace IDs for each request), **metrics** answers "how is the system performing" (time-series numbers like request rate, latency percentiles, error rate), **tracing** answers "where is the problem" (following a single request across services to identify which hop is slow or failing). Each serves a different purpose - metrics detect that something is wrong, logging explains why, tracing pinpoints where in the distributed flow the failure occurred
44. What metrics would you track for backend services?
    - **Answer:**
      - Track request rate (throughput), latency percentiles (p50, p95, p99), error rate (4xx/5xx), resource usage (CPU, memory, connections), and business metrics (records processed)
      - Stored procedure execution time can be a key metric for database-heavy operations
      - **RED method** for microservices: Rate (requests per second), Errors (error rate as percentage), Duration (latency percentiles) - focused on the request lifecycle and user experience. **USE method** for resource monitoring: Utilization (percentage of CPU/memory/disk in use), Saturation (queue length, thread pool usage), Errors (hardware errors, OOM kills) - focused on infrastructure health. Use RED to detect user-facing issues, USE to detect capacity issues before they impact users. Alert on p99 latency exceeding SLA threshold and error rate exceeding 1%
45. What is SLA?
    - **Answer:**
      - SLA (Service Level Agreement) is a commitment about expected service level, like 99.5% uptime
      - Informal SLAs might require the dashboard to load within 2 seconds, and sensor data to be available within 30 seconds of ingestion
      - SLAs drive architectural decisions: a 99.9% uptime SLA (8.7 hours downtime/year) requires redundancy across multiple availability zones, automated failover, and database replication, significantly increasing infrastructure cost. A 99% SLA (3.6 days downtime/year) allows simpler single-region deployment with manual failover. Higher SLA = higher cost and complexity, so SLAs should reflect actual business requirements, not arbitrary numbers. Breaching an SLA may have contractual penalties (SLAs with customers) or reputation impact (internal SLAs with stakeholders)
46. What is SLO?
    - **Answer:**
      - SLO (Service Level Objective) is an internal target, usually stricter than the SLA
      - An SLO of 99% of dashboard queries completing within 1 second provides a buffer before breaching a 2-second SLA
      - SLOs trigger proactive alerts: if the error rate exceeds the SLO threshold for a defined window (e.g., 5 minutes), on-call engineers are paged, catching issues before the SLA is breached. The buffer between SLO and SLA (e.g., SLO 99% within 1s, SLA 99% within 2s) provides margin for investigation and remediation. Set SLOs at the 50th percentile of your SLA - if SLA says 2 seconds, SLO should target 1 second
47. What is error budget?
    - **Answer:**
      - Error budget is the acceptable amount of downtime based on the SLO
      - If the SLO is 99.9% uptime, the error budget allows 0.1% downtime (about 8.7 hours per year)
      - Teams can deploy confidently within the budget but must prioritize stability when it is depleted
      - Error budget drives the balance between velocity and reliability: when the budget is healthy (e.g., 80% remaining), teams can ship features aggressively and take risks with new deployments. When the budget is exhausted (0% remaining), all engineering effort shifts to reliability work - fixing bugs, improving monitoring, adding redundancy - until the budget recovers. This creates a data-driven conversation between product and engineering about release cadence, replacing subjective debates with measurable thresholds
48. How do you design for scalability?
    - **Answer:**
      - Design for scalability by identifying bottlenecks - usually the database or a single-threaded component
      - For inventory systems, optimizing queries and indexing is often more effective than adding servers
      - For IoT systems, Kafka allows adding more consumers as data volume grows
      - Horizontal scaling: add more instances behind a load balancer (stateless APIs scale horizontally trivially). Vertical scaling: increase instance size (CPU, memory) for stateful components like databases. Stateless API design: store session data in Redis instead of in-memory, so any instance can handle any request. Database scaling: read replicas for read-heavy workloads (dashboard queries), sharding for write-heavy workloads (partitioning sensor data by device ID or time range), and connection pooling (HikariCP) to manage database connections efficiently
49. How do you design for reliability?
    - **Answer:**
      - Design for reliability by eliminating single points of failure, implementing retries and circuit breakers, and monitoring health checks
      - Running two Spring Boot instances behind a load balancer ensures that if one fails, the other continues serving
      - Redundancy at every level: multiple availability zones for compute (if AZ-A goes down, AZ-B serves traffic), database replication (primary-replica with automatic failover), and message broker clustering (Kafka with replication factor 3). Graceful degradation means the system continues operating at reduced capacity - dashboard shows cached data if the database is slow, alerting continues even if the reporting service is down. Chaos engineering: regularly inject failures (terminate instances, add network latency, fill disk) to verify resilience before real failures occur
50. How do you design for maintainability?
    - **Answer:**
      - Design for maintainability with clean, modular code, clear separation of concerns, consistent error handling, and comprehensive logging
      - Documenting optimized stored procedures with comments explaining execution plan changes and why each index was created improves long-term maintainability
      - Coding standards: consistent naming conventions, formatting rules (Checkstyle/Prettier), and architectural guidelines documented in team wikis. Code reviews: every PR reviewed by at least one other developer, enforcing standards and sharing knowledge. Automated tests: unit tests for business logic, integration tests for API contracts, and regression tests for critical flows - aiming for confidence to refactor without breaking things. Reducing onboarding time: clear README with setup instructions, well-organized code with obvious directory structure, and domain-specific comments explaining "why" decisions were made, not just "what" the code does
