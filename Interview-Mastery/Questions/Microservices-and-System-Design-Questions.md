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
      - In our cold-chain project, separate services for ingestion, alerting, and dashboard could each be deployed independently without affecting others
   - **If asked more:**
      - I would walk through how our cold-chain system could be refactored into microservices: sensor-ingestion service, alert-engine service, dashboard-api service, each with its own database (InfluxDB for time-series, PostgreSQL for config)
2. Difference between monolith and microservices.
   - **Answer:**
      - A monolith is a single codebase deployed as one unit, while microservices split into separate deployable services
      - Our CDMS was a monolith with all ETL and reporting in one codebase; microservices would let us scale ETL and reporting independently
   - **If asked more:**
      - I would discuss tradeoffs: monolith is simpler for small teams but limits independent scaling; microservices add communication and data consistency complexity
3. Advantages of microservices.
   - **Answer:**
      - Microservices enable independent deployment, scaling, and technology diversity
      - In our inventory system, if validation logic needed more resources, we could scale only that service instead of the entire application
   - **If asked more:**
      - I would add that microservices improve fault isolation - one service failing does not bring down the whole system, critical for cold-chain where alerting must stay up even if the dashboard has issues
4. Disadvantages of microservices.
   - **Answer:**
      - The main drawbacks are distributed complexity, network latency, data consistency challenges, and operational overhead
      - In the cold-chain project, debugging across Kafka, Spring Boot APIs, and InfluxDB was harder than a monolithic application would have been
   - **If asked more:**
      - I would explain that microservices need mature DevOps, monitoring, and team coordination
      - Without proper tooling, tracing a single failed data flow across Lambda, Kafka, and the API layer takes significantly longer than in a monolith
5. How do services communicate?
   - **Answer:**
      - Services communicate either synchronously via REST/gRPC or asynchronously via message queues like Kafka
      - In the cold-chain project, IoT data flowed asynchronously through Kafka from Lambda to Spring Boot, while dashboard queries were synchronous REST calls to the API layer
   - **If asked more:**
      - I would explain the choice criteria: REST for request-response with low latency needs, Kafka for high-throughput decoupled processing where the producer does not need an immediate answer
6. REST vs messaging.
   - **Answer:**
      - REST is synchronous, simpler, but creates tight coupling
      - Messaging with Kafka decouples producers from consumers and buffers data
      - In our cold-chain system, we used both: Kafka for sensor data ingestion and REST for dashboard API queries
   - **If asked more:**
      - I would compare based on use case - REST for CRUD where the client needs an immediate response, messaging for event-driven workflows like ETL pipelines or real-time data where throughput matters more than immediate reply
7. What is API gateway?
   - **Answer:**
      - An API gateway is a single entry point that routes requests to backend services, handling authentication, rate limiting, and aggregation
      - Our Spring Boot API layer acted as a mini-gateway for the React dashboard, consolidating sensor data from InfluxDB and alerts from the alert engine
   - **If asked more:**
      - I would discuss features like request transformation, circuit breaking, and how gateways simplify client code by hiding service decomposition from frontend consumers
8. What is service discovery?
   - **Answer:**
      - Service discovery allows services to find each other dynamically without hardcoded addresses
      - In our EC2-based projects, we used environment variables for service URLs, but in a true microservices setup, tools like Eureka or Kubernetes DNS handle this automatically
   - **If asked more:**
      - I would explain client-side vs server-side discovery and how load balancers like AWS ALB can act as a discovery mechanism by routing to healthy EC2 instances
9. What is circuit breaker?
   - **Answer:**
      - Circuit breaker prevents cascading failures by stopping calls to a failing service and failing fast
      - In our inventory system, if the validation database was slow, a circuit breaker would stop hitting it repeatedly and return a cached response until it recovered
   - **If asked more:**
      - I would explain the three states (closed, open, half-open) and how Resilience4j configures timeout thresholds and retry policies in Spring Boot
10. What is retry pattern?
   - **Answer:**
      - Retry pattern automatically re-attempts a failed operation, usually with exponential backoff
      - In our CDMS ETL pipeline, if a downstream API call failed temporarily, retry with backoff retried up to 3 times before logging a failure
   - **If asked more:**
      - I would distinguish retry from circuit breaker - retry works for transient failures, while circuit breaker protects against sustained failures
      - Combining both is recommended
11. What is timeout?
   - **Answer:**
      - Timeout sets a maximum wait time for an operation, preventing threads from hanging indefinitely
      - In our cold-chain APIs, we configured connection and read timeouts on REST calls to InfluxDB to ensure the dashboard never waited more than 5 seconds
   - **If asked more:**
      - I would discuss connection timeout, read timeout, and write timeout, and how they are configured in Spring Boot's RestTemplate or WebClient
12. What is bulkhead pattern?
   - **Answer:**
      - Bulkhead isolates resources into separate pools so failure in one does not deplete resources for others
      - We could allocate separate thread pools for real-time sensor processing and dashboard queries, so a slow dashboard query never blocks sensor ingestion
   - **If asked more:**
      - I would explain thread pool isolation vs semaphore isolation in Resilience4j, and how bulkheads protect critical paths like alerting from less critical paths like reporting
13. What is rate limiting?
   - **Answer:**
      - Rate limiting controls how many requests a client can make in a time window, protecting backend resources from overload
      - I understand token bucket and sliding window algorithms - useful for public APIs to prevent abuse
   - **If asked more:**
      - I would discuss implementing rate limiting at API gateway level using Redis for distributed rate counting, with different limits for different client tiers
14. What is load balancing?
   - **Answer:**
      - Load balancing distributes traffic across multiple servers to improve availability and throughput
      - We used AWS Application Load Balancer to distribute requests across our EC2 instances for Spring Boot APIs serving the cold-chain dashboard
   - **If asked more:**
      - I would explain round-robin vs least-connections algorithms, and how health checks on the ALB automatically removed unhealthy EC2 instances from the target group
15. What is distributed tracing?
   - **Answer:**
      - Distributed tracing tracks a request as it flows through multiple services, helping debug latency and failures
      - In the cold-chain project, a correlation ID helped trace a sensor reading across the entire flow from Lambda to Kafka to Spring Boot
   - **If asked more:**
      - I would explain how tools like Jaeger or Zipkin work with trace IDs and span IDs, and how Spring Cloud Sleuth automatically injects trace IDs into logs and HTTP headers
16. What is centralized logging?
   - **Answer:**
      - Centralized logging aggregates logs from all services into a single searchable platform
      - We used CloudWatch Logs to collect and search logs from all Spring Boot EC2 instances without SSHing into each server individually
   - **If asked more:**
      - I would discuss log levels (ERROR, WARN, INFO, DEBUG), structured logging with JSON format, and how centralized logging correlates errors across services using the trace ID
17. What is correlation id?
   - **Answer:**
      - A correlation ID is a unique identifier attached to a request as it passes through multiple services, linking all related log entries
      - In the cold-chain system, each sensor reading carried a UUID from ingestion to dashboard, making end-to-end tracing possible
   - **If asked more:**
      - I would explain generating correlation IDs at the entry point and propagating them through Kafka message headers and log statements for full visibility
18. What is eventual consistency?
   - **Answer:**
      - Eventual consistency means that after a write, all replicas will converge to the same state given enough time without new updates
      - In our inventory system, partner submissions were eventually consistent across reporting views - not instant, but guaranteed within seconds
   - **If asked more:**
      - I would contrast with strong consistency and discuss tradeoffs: eventual consistency improves availability and performance but requires handling stale reads in application logic
19. What is distributed transaction?
   - **Answer:**
      - A distributed transaction spans multiple services or databases, requiring coordination for all-or-nothing execution
      - In our inventory system, partner submissions involved validating records, updating baseline, and calculating inventory - all in MSSQL transactions at the database level
   - **If asked more:**
      - I would discuss two-phase commit (2PC) drawbacks - latency and reduced availability - and why modern systems prefer eventual consistency with saga patterns
20. What is saga pattern?
   - **Answer:**
      - Saga is a sequence of local transactions where each step publishes an event to trigger the next, with compensating transactions for rollback
      - If our inventory had a distributed flow across services, a saga would ensure validation rollback if baseline update failed after validation succeeded
   - **If asked more:**
      - I would explain choreography vs orchestration sagas - choreography uses events, orchestration uses a coordinator - and when each is appropriate based on complexity needs
21. What is CQRS?
   - **Answer:**
      - CQRS separates read and write operations into different models, optimizing each for its purpose
      - In our cold-chain project, writes went to Kafka/InfluxDB for high-throughput ingestion, while reads used optimized InfluxDB queries and Redis caching for fast dashboard rendering
   - **If asked more:**
      - I would explain that CQRS is useful when read and write workloads differ - our IoT writes were high-volume append-only, while reads were aggregation queries with time range filters
22. What is event-driven architecture?
   - **Answer:**
      - Event-driven architecture uses events to trigger and communicate between decoupled services
      - The cold-chain system was fully event-driven: IoT data triggered Lambda, which published to Kafka, which triggered Spring Boot processing, which stored to InfluxDB and triggered dashboard updates
   - **If asked more:**
      - I would explain event sourcing vs event notification, and how Kafka's log-based approach made our system replayable and auditable
23. What is idempotency?
   - **Answer:**
      - Idempotency means performing the same operation multiple times produces the same result as doing it once
      - In our inventory system, if a partner submitted the same serial record twice, the validation engine identified it as duplicate using a unique constraint on the serial number, preventing double-counting
   - **If asked more:**
      - I would explain idempotency keys - clients send a unique key, the server deduplicates based on that key, storing the result for subsequent identical requests
24. Why is idempotency important?
   - **Answer:**
      - Idempotency prevents data corruption from duplicate requests caused by network retries or client errors
      - In the inventory system, without idempotency, partners could accidentally inflate inventory counts, causing reconciliation issues and financial discrepancies
   - **If asked more:**
      - I would mention that idempotency is critical for payment-like operations and inventory updates where even a single duplicate can have significant business impact
25. How do you design idempotent APIs?
   - **Answer:**
      - I would use an idempotency key pattern: the client sends a unique key, the server checks if it has already processed that key and returns the cached response
      - In our inventory API, the batch submission ID served as the idempotency key
   - **If asked more:**
      - I would discuss storage options for idempotency keys (Redis with TTL or database table), expiration policy, and handling concurrent requests with the same key
26. How do you handle duplicate requests?
   - **Answer:**
      - For naturally idempotent operations like GET or PUT, no extra handling is needed
      - For non-idempotent operations like inventory submission, I would add a unique constraint on batch reference and reject duplicates at the database level with appropriate error messaging
   - **If asked more:**
      - I would add that duplicate detection should have a time window - a duplicate with the same data arriving after a week might be a legitimate new request, so TTL-based deduplication is preferred
27. How do you handle API versioning?
   - **Answer:**
      - I prefer URL-based versioning (`/api/v1/resource`) because it is explicit and easy to route at the load balancer level
      - In our internal projects, we did not need aggressive versioning since API consumers were controlled, but for public APIs I would use this approach
   - **If asked more:**
      - I would discuss backward compatibility: never remove fields clients depend on, add fields as optional, deprecate with a migration timeline documented in Swagger
28. How do you design pagination?
   - **Answer:**
      - I use cursor-based pagination with a last-seen ID or timestamp for large or real-time datasets because offset-based pagination can be inconsistent when data changes
      - In the inventory dashboard, offset-based was fine for static partner lists, but for sensor readings, cursor-based prevents duplicates during live streaming
   - **If asked more:**
      - I would discuss page size limits, total count performance, and caching total counts in Redis instead of counting on every request
29. How do you design search APIs?
   - **Answer:**
      - I keep search APIs simple with query parameters for filtering, sorting, and pagination
      - In CDMS, search endpoints accepted date range, partner ID, and product category as query parameters, with WHERE clauses built dynamically using Spring Data JPA Specifications
   - **If asked more:**
      - I would discuss full-text search with Elasticsearch, indexed columns for filter fields, and preventing SQL injection by using parameterized queries
30. How do you design notification system?
   - **Answer:**
      - I would design it as an event-driven system: notification events published to a Kafka topic, consumers process them and send via email/SMS/push, with a database storing delivery status
      - In the cold-chain project, excursion alerts were notifications triggered when temperature exceeded thresholds
   - **If asked more:**
      - I would discuss delivery guarantees (at-least-once with deduplication), retry mechanisms for failed deliveries, and rate limiting to avoid flooding recipients during multi-sensor excursions
31. How do you design rate limiter?
   - **Answer:**
      - I would use a sliding window counter stored in Redis, tracking request counts per client per time window using sorted sets
      - The API gateway would check eligibility before forwarding each request, returning 429 Too Many Requests when the limit is exceeded
   - **If asked more:**
      - I would explain the token bucket algorithm as an alternative and discuss distributed rate limiting challenges - clock skew and Redis being a single point of failure
32. How do you design URL shortener?
   - **Answer:**
      - I would use a hash-based approach: generate a unique short key from a hash of the original URL, store the mapping in a database with Redis caching, and return a 302 redirect on lookup
   - **If asked more:**
      - I would discuss collision handling (if two URLs produce the same hash, append a salt), key length optimization, and analytics tracking for click counts
33. How do you design distributed cache?
   - **Answer:**
      - I would use Redis with a cache-aside pattern: check Redis first, if not found, fetch from the database and store in Redis with a TTL
      - In our cold-chain project, we cached frequently accessed sensor metadata and dashboard configurations in Redis to reduce database load
   - **If asked more:**
      - I would discuss consistent hashing for Redis cluster sharding, handling cache stampede with early expiration and locking, and monitoring cache hit/miss ratios
34. How do you design chat system?
   - **Answer:**
      - I would design it with WebSocket connections for real-time messaging, Kafka for message persistence and ordering, and a database for message history partitioned by conversation ID
   - **If asked more:**
      - I would discuss presence detection (heartbeat mechanism), message delivery guarantees, and scaling WebSocket servers using sticky session load balancers
35. How do you design inventory system?
   - **Answer:**
      - I would design it with a centralized validation engine that receives serial records from partners, validates against a verified baseline, and calculates real-time inventory
      - This is exactly what we built: MSSQL stored procedures validated 10,000+ records from 200+ partners in under 15 seconds
   - **If asked more:**
      - I would walk through the exact flow - partner submits batch, validation checks duplicates and invalid serials against baseline, accepted records update inventory, rejected records are flagged for reconciliation
36. How do you design IoT monitoring system?
   - **Answer:**
      - I would design it with a scalable ingestion pipeline: IoT gateways send MQTT to AWS IoT Core, which triggers Lambda to publish to Kafka, Spring Boot consumers process and store time-series data in InfluxDB, and Grafana/React dashboards query the APIs
      - This is our cold-chain architecture
   - **If asked more:**
      - I would explain the reasoning behind each choice - MQTT for lightweight IoT protocol, Kafka for buffering bursts, InfluxDB for time-series optimization
37. How do you design real-time dashboard?
   - **Answer:**
      - I would design it with polling using React setInterval, backed by optimized InfluxDB queries and Redis caching
      - In our cold-chain dashboard, we polled the Spring Boot API every 30 seconds for sensor readings, with Redis caching frequently accessed data points
   - **If asked more:**
      - I would discuss WebSocket vs polling tradeoffs, data aggregation strategies (pre-aggregating at 1-minute intervals), and lazy loading for historical data
38. How do you handle high throughput ingestion?
   - **Answer:**
      - I would use a message queue like Kafka as a buffer between producers and consumers
      - In the cold-chain system, hundreds of IoT gateways sent data simultaneously; Kafka absorbed the burst, and consumers processed at their own pace without data loss
   - **If asked more:**
      - I would discuss partition count planning, consumer group scaling, and monitoring consumer lag to detect processing bottlenecks early
39. How do you handle data consistency?
   - **Answer:**
      - For same-database consistency, I rely on ACID transactions
      - For cross-service consistency, I use eventual consistency with idempotent operations
      - In CDMS, the ETL pipeline used database transactions to ensure partial failures did not leave reporting data inconsistent
   - **If asked more:**
      - I would discuss CAP theorem tradeoffs - in distributed systems, we often choose availability over strong consistency and design APIs to handle eventual consistency gracefully
40. How do you handle failure in downstream services?
   - **Answer:**
      - I use timeouts, retries with exponential backoff, circuit breakers, and fallback mechanisms
      - In the cold-chain system, if InfluxDB was slow, we returned stale data from Redis cache as a fallback instead of showing an error
   - **If asked more:**
      - I would discuss graceful degradation - the system should partially work even when dependencies fail, and monitoring should alert when fallback paths are activated
41. How do you choose between sync and async communication?
   - **Answer:**
      - I choose synchronous (REST) when the client needs an immediate response and the operation is quick
      - I choose asynchronous (Kafka) when high throughput, decoupling, or background processing is needed
      - In our cold-chain system, sensor ingestion was async, dashboard queries were sync
   - **If asked more:**
      - I would add that async communication introduces complexity in error handling and tracing, so the decision should consider whether the client can tolerate delayed responses
42. How do you secure microservices?
   - **Answer:**
      - I secure microservices with JWT-based authentication, role-based access control, HTTPS, API gateway as a security barrier, and input validation at every service boundary
      - We used Spring Security with JWT tokens for API authentication and configured CORS for the React frontend
   - **If asked more:**
      - I would discuss OAuth2 for delegated authorization, service-to-service authentication using mutual TLS, and secrets management using environment variables
43. How do you monitor microservices?
   - **Answer:**
      - I monitor with health check endpoints, metrics (request rate, latency, error rate, resource usage), centralized logging, and distributed tracing
      - We used Spring Boot Actuator, CloudWatch for metrics and logs, and Grafana dashboards for application-level monitoring
   - **If asked more:**
      - I would discuss the three pillars of observability - logging, metrics, and tracing - and how each serves a different purpose
44. What metrics would you track for backend services?
   - **Answer:**
      - I track request rate (throughput), latency percentiles (p50, p95, p99), error rate (4xx/5xx), resource usage (CPU, memory, connections), and business metrics (records processed)
      - In CDMS, we tracked stored procedure execution time as a key metric
   - **If asked more:**
      - I would explain RED method (Rate, Errors, Duration) for microservices and USE method (Utilization, Saturation, Errors) for resource monitoring
45. What is SLA?
   - **Answer:**
      - SLA (Service Level Agreement) is a commitment about expected service level, like 99.5% uptime
      - In our projects, we had informal SLAs - the dashboard had to load within 2 seconds, and sensor data had to be available within 30 seconds of ingestion
   - **If asked more:**
      - I would discuss how SLAs drive architectural decisions - a 99.9% SLA requires redundancy across AZs, while 99% allows simpler single-region deployment
46. What is SLO?
   - **Answer:**
      - SLO (Service Level Objective) is an internal target, usually stricter than the SLA
      - For the cold-chain system, our SLO was 99% of dashboard queries complete within 1 second, giving a buffer before breaching the 2-second SLA
   - **If asked more:**
      - I would explain how SLOs trigger alerts - if error rate exceeds the SLO for 5 minutes, on-call is paged, catching issues before the SLA is breached
47. What is error budget?
   - **Answer:**
      - Error budget is the acceptable amount of downtime based on the SLO
      - If our SLO is 99.9% uptime, the error budget allows 0.1% downtime (about 8.7 hours per year)
      - Teams can deploy confidently within the budget but must prioritize stability when it is depleted
   - **If asked more:**
      - I would discuss how error budget drives the balance between velocity and reliability - more deployments when the budget is healthy, focus on stability when it is exhausted
48. How do you design for scalability?
   - **Answer:**
      - I design for scalability by identifying bottlenecks - usually the database or a single-threaded component
      - For inventory, we optimized queries and indexing rather than adding servers
      - For cold-chain, Kafka allowed adding more consumers as data volume grew
   - **If asked more:**
      - I would discuss horizontal vs vertical scaling, stateless API design (store session data in Redis), and database scaling strategies like read replicas and sharding
49. How do you design for reliability?
   - **Answer:**
      - I design for reliability by eliminating single points of failure, implementing retries and circuit breakers, and monitoring health checks
      - In the cold-chain project, we ran two Spring Boot instances behind a load balancer so if one failed, the other continued serving
   - **If asked more:**
      - I would discuss redundancy at every level (multiple AZs, multiple instances, database replication), graceful degradation, and chaos engineering principles
50. How do you design for maintainability?
   - **Answer:**
      - I design for maintainability with clean, modular code, clear separation of concerns, consistent error handling, and comprehensive logging
      - In CDMS, I documented optimized stored procedures with comments explaining execution plan changes and why each index was created
   - **If asked more:**
      - I would discuss coding standards, code reviews, automated tests, and the importance of reducing the time a new team member takes to understand and modify code safely

