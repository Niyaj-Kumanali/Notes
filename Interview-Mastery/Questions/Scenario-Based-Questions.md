# Scenario-Based Questions

## Questions

1. A stored procedure suddenly becomes slow in production. How do you debug it?
2. A query uses an index in testing but not in production. What could be the reason?
3. A database job takes 5 hours. How would you reduce it?
4. A report has incorrect data after ETL. How do you investigate?
5. Duplicate serial records are entering inventory. How do you prevent it?
6. Inventory data is delayed from some channels. How do you handle it?
7. Kafka consumer lag is increasing. How do you debug it?
8. Kafka messages are duplicated. How do you handle it?
9. Kafka messages are out of order. How do you handle it?
10. A Kafka consumer keeps failing on one message. What do you do?
11. AWS Lambda is timing out. How do you debug it?
12. EC2 deployment failed after Jenkins pipeline ran successfully. What do you check?
13. API response time increased suddenly. How do you debug it?
14. Database connections are exhausted. How do you debug it?
15. Redis cache has stale data. How do you fix it?
16. A scheduled job runs on all pods and sends duplicate emails. How do you solve it?
17. A scheduled job is missing executions. How do you monitor it?
18. JWT works locally but fails behind load balancer. What do you check?
19. Users get 403 after enabling CSRF. How do you fix it?
20. Admin role update is not reflected immediately. How do you design it?
21. Production API returns 500 but logs are unclear. How do you improve observability?
22. A service dependency is down. How should your service behave?
23. A dashboard becomes slow with live data. How do you optimize it?
24. IoT sensor sends invalid readings. How do you validate them?
25. IoT gateway sends data in bursts. How do you handle backpressure?
26. A database migration causes backlog. How do you recover safely?
27. A deployment needs rollback. What is your process?
28. A Docker container works locally but fails on EC2. What do you check?
29. A REST API has inconsistent error responses. How do you standardize it?
30. A frontend user sees pages they should not access. How do you fix RBAC?
31. A stored procedure returns correct results but is slow. What metrics do you check?
32. A query is fast for one parameter but slow for another. What could be wrong?
33. A microservice is receiving duplicate requests. How do you make it idempotent?
34. A payment-like operation succeeds but client times out. How do you handle retry?
35. A report must be generated exactly once across multiple servers. How do you design it?
36. An API must process 10,000 records. Do you process synchronously or asynchronously?
37. A customer asks for near real-time dashboard updates. What architecture would you choose?
38. A database table is growing very large. How do you manage performance?
39. A third-party API is slow. How do you protect your application?
40. A production issue happens at midnight. How do you approach debugging?

---

## Answers

1. A stored procedure suddenly becomes slow in production. How do you debug it?
   - **Answer:** I would check if the execution plan changed - parameter sniffing or stale statistics can cause the optimizer to choose a different plan. In CDMS, a procedure slowed because statistics were stale after a large data load; updating statistics restored performance.
   - **If asked more:** I would examine wait stats, compare actual vs estimated rows in the execution plan, test with OPTION (RECOMPILE) to rule out parameter sniffing, then decide between index fixes or plan guide.
2. A query uses an index in testing but not in production. What could be the reason?
   - **Answer:** Data distribution differences between environments - production has more data with different cardinality, making the optimizer choose a scan over a seek. In CDMS, a query used an index in dev but scanned in production because the indexed column had many NULL values in production.
   - **If asked more:** I would compare statistics between environments, check parameter sniffing, examine actual execution plans from production, and consider index maintenance or query hints after thorough testing.
3. A database job takes 5 hours. How would you reduce it?
   - **Answer:** I would analyze the execution plan to find the most expensive operations and target them with index improvements or query rewrites. In CDMS, I reduced a job from 5 hours to under 12 minutes by identifying missing indexes, rewriting correlated subqueries as joins, and breaking the procedure into batch operations.
   - **If asked more:** I would discuss batch processing (10K records at a time), partitioning large tables, updating statistics, and using temp tables for intermediate results instead of CTEs.
4. A report has incorrect data after ETL. How do you investigate?
   - **Answer:** I would trace the data flow from source to report - check staging tables, transformation logic, and aggregation steps. In CDMS, an incorrect report was traced to a JOIN causing data duplication; adding DISTINCT and verifying row counts at each stage identified the issue.
   - **If asked more:** I would compare row counts at each ETL stage, check NULL handling, examine transformation SQL for logic errors, and compare against a manually calculated sample.
5. Duplicate serial records are entering inventory. How do you prevent it?
   - **Answer:** I would add a unique constraint on the serial number at the database level and a duplicate check in the application layer. In the inventory system, we prevented duplicates by checking the serial against the verified baseline and rejecting existing records.
   - **If asked more:** I would discuss handling race conditions with Serializable isolation level for check-then-insert, and logging rejected duplicates for auditing.
6. Inventory data is delayed from some channels. How do you handle it?
   - **Answer:** I would investigate the data flow for each delayed channel - check submission timestamps, ETL processing, and error logs. In our inventory system, delays were often from partners sending data in non-standard formats that failed validation.
   - **If asked more:** I would set up monitoring for each channel's submission time, create alerts for missed windows, and implement a grace period before marking data as stale.
7. Kafka consumer lag is increasing. How do you debug it?
   - **Answer:** I would check consumer lag metrics in Kafka monitoring, identify which partition has the highest lag, and check consumer logs for slow processing. In the cold-chain project, lag increased when InfluxDB writes bottlenecked during high sensor traffic.
   - **If asked more:** I would check consumer processing rate vs producer rate, look for poison pill messages, consider increasing partitions or consumer instances, and optimize processing with batch writes.
8. Kafka messages are duplicated. How do you handle it?
   - **Answer:** Duplicates are expected with at-least-once delivery. Our consumer was designed idempotently - processing the same message twice produced the same result. In the cold-chain project, we used deduplication IDs in InfluxDB to ignore duplicate sensor readings.
   - **If asked more:** I would discuss enabling idempotent producers, tracking processed message IDs in Redis, and designing business logic to tolerate duplicates.
9. Kafka messages are out of order. How do you handle it?
   - **Answer:** Out-of-order messages happen when producers retry or networks delay. Each sensor reading had a timestamp for sorting. For strict ordering, we used a partition key on sensor ID, ensuring messages for the same sensor stayed in order.
   - **If asked more:** I would discuss partition keys for per-entity ordering, handling late-arriving data with a tolerance window, and the tradeoff between ordering and parallelism.
10. A Kafka consumer keeps failing on one message. What do you do?
    - **Answer:** I would check consumer logs for the specific error and examine the problematic message. If corrupt, I would skip it using a dead-letter queue pattern. We configured a SeekToCurrentErrorHandler that retried a few times then sent to a DLQ topic.
    - **If asked more:** I would discuss implementing a dead-letter topic for failed messages, alerts for DLQ entries, and periodic review to fix upstream data issues.
11. AWS Lambda is timing out. How do you debug it?
    - **Answer:** I would check CloudWatch Logs, Lambda timeout configuration, and identify the slow operation. In the cold-chain project, a Lambda timed out because the downstream Kafka publish had network connectivity issues to the EC2 broker.
    - **If asked more:** I would increase timeout temporarily, check memory allocation (more memory = more CPU), optimize the code (connection reuse, reduce payload), and consider async invocation for long operations.
12. EC2 deployment failed after Jenkins pipeline ran successfully. What do you check?
    - **Answer:** I would SSH into EC2 and check Docker logs, disk space, and the application log. In our project, a deployment failed because the EC2 instance ran out of disk space from old Docker images not being cleaned up.
    - **If asked more:** I would check EC2 resource usage, verify the Docker daemon is running, check the compose file for correct image tags, and examine application startup logs for connection failures.
13. API response time increased suddenly. How do you debug it?
    - **Answer:** I would check recent deployments, database query performance, and external API dependencies. In the cold-chain project, response time increased when an InfluxDB query stopped using the time index after a frontend update changed the query filter, causing full scans.
    - **If asked more:** I would check APM tools or CloudWatch metrics, look for slow queries, thread pool exhaustion, Redis cache hit ratios, and identify slow endpoints via detailed request logging.
14. Database connections are exhausted. How do you debug it?
    - **Answer:** I would check the database's active connection list and identify which application or query is holding connections without releasing. In our Spring Boot app, a missing connection pool configuration for a long-running report query consumed all connections.
    - **If asked more:** I would configure HikariCP properly (max pool size, timeout, leak detection), review code for missing connection closes, and add monitoring on connection pool metrics via Actuator.
15. Redis cache has stale data. How do you fix it?
    - **Answer:** I would check TTL configuration and cache invalidation strategy. Stale data means TTL is too long or cache is not invalidated when source data changes. In our cold-chain project, we set appropriate TTLs and invalidated cache entries when new sensor data arrived.
    - **If asked more:** I would discuss publishing invalidation events on data updates, using shorter TTLs as a quick fix, and write-through caching for data that changes frequently.
16. A scheduled job runs on all pods and sends duplicate emails. How do you solve it?
    - **Answer:** I would use a distributed lock with Redis via Redisson or a database lock table, so only one instance acquires the lock and executes the job. We used `@SchedulerLock` where the first Spring Boot instance to acquire the lock ran the job.
    - **If asked more:** I would discuss lock TTL (lease time) to prevent deadlocks, graceful handling of lock acquisition failures, and monitoring which instance executed each run.
17. A scheduled job is missing executions. How do you monitor it?
    - **Answer:** I would add logging at job start and end with timestamps and create a check verifying the last successful execution time is within expected intervals. In our inventory system, a health check endpoint reported the last reconciliation job run time, alerting if overdue.
    - **If asked more:** I would discuss exposing job execution metrics via Actuator, creating a Grafana dashboard for success/failure rates, and configuring alerts for missed schedules.
18. JWT works locally but fails behind load balancer. What do you check?
    - **Answer:** I would check if the load balancer strips or modifies headers, particularly the Authorization header. In our cold-chain project, JWT failed on EC2 because SSL termination at the load balancer changed the protocol, and the `secure` flag on JWT cookies did not match.
    - **If asked more:** I would also check X-Forwarded-Proto and X-Forwarded-For headers and verify Spring Security's requiresSecure() matches the actual protocol.
19. Users get 403 after enabling CSRF. How do you fix it?
    - **Answer:** For our REST APIs used by a SPA, we disabled CSRF protection because it is not needed for token-based authentication - CSRF is primarily for cookie-based auth. We called `http.csrf().disable()` in Spring Security.
    - **If asked more:** I would explain that SPAs using JWT in Authorization headers are not vulnerable to CSRF since browsers do not auto-include Auth headers for cross-origin requests.
20. Admin role update is not reflected immediately. How do you design it?
    - **Answer:** I would ensure role changes either force re-login (if stored in JWT) or check from the database on every request. In our system, roles in JWT required re-login. For immediate effect, database checks on every API call would be needed.
    - **If asked more:** I would discuss the tradeoff: JWT roles are fast but not immediately updatable; database-checked roles are slower but real-time. A hybrid uses short-lived JWTs with DB fallback for sensitive operations.
21. Production API returns 500 but logs are unclear. How do you improve observability?
    - **Answer:** I would add structured logging with correlation IDs, ensure exception stack traces are logged, and log request/response bodies at TRACE level for debugging. In the cold-chain project, unclear logs were fixed by adding method-level logging with input parameters and context.
    - **If asked more:** I would discuss implementing Actuator for health checks, distributed tracing, centralized log aggregation (CloudWatch or ELK), and alerting based on error rate thresholds.
22. A service dependency is down. How should your service behave?
    - **Answer:** The service should degrade gracefully - return cached data if available, return a meaningful error message (not 500), and not crash or hang. In the cold-chain project, when InfluxDB was unavailable, the API returned stale data from Redis with a `stale: true` flag.
    - **If asked more:** I would discuss circuit breaker to fail fast, reasonable timeouts (2-3 seconds), and fallback responses that let the frontend show appropriate UI like a "data may be delayed" banner.
23. A dashboard becomes slow with live data. How do you optimize it?
    - **Answer:** I would optimize the backend (query optimization, pagination, caching) and the frontend (reduce re-renders, virtual lists, debounce updates). In our cold-chain dashboard, we updated data every 30 seconds instead of on every reading and pre-aggregated time-series data in InfluxDB.
    - **If asked more:** I would discuss WebSocket updates instead of polling, React.memo to prevent unnecessary re-renders, and server-side rendering for initial page load.
24. IoT sensor sends invalid readings. How do you validate them?
    - **Answer:** I would apply range checks (temperature between -40 and 100C), format checks (valid JSON, required fields), and rate-of-change checks (flag if temp jumps 20 degrees in one minute). In the cold-chain project, Lambda validated incoming MQTT messages before publishing to Kafka.
    - **If asked more:** I would discuss handling different invalid data types - sensor malfunction (constant values), communication errors (gaps, checksum failures), and out-of-range values - each with different strategy (reject, flag, or interpolate).
25. IoT gateway sends data in bursts. How do you handle backpressure?
    - **Answer:** Kafka acts as a buffer that absorbs bursts - producers publish at high rates while consumers process at their own pace. In the cold-chain project, when gateways reconnected and sent bursts, Kafka queued messages and consumers caught up gradually without data loss.
    - **If asked more:** I would discuss monitoring consumer lag to detect backpressure, scaling consumers by increasing partitions, and rate limiting at ingestion if bursts exceed Kafka capacity.
26. A database migration causes backlog. How do you recover safely?
    - **Answer:** I would stop writes to the affected table, assess backlog size, and run batch processing to catch up. For CDMS, if a migration caused failures, we would restore from backup with point-in-time recovery, fix the script, and re-run during low-traffic hours.
    - **If asked more:** I would discuss testing migrations on staging with production data, using backward-compatible migrations, and having a rollback script ready before running any migration.
27. A deployment needs rollback. What is your process?
    - **Answer:** I would stop the current deployment, restore the previous Docker image tag from Jenkins artifacts, and redeploy. We kept the last three successful image tags, so rollback was `docker pull app:v1.2.3-previous` and restarting containers, followed by smoke tests.
    - **If asked more:** I would differentiate code rollback vs database rollback - DB changes must be backward-compatible so code rollback is safe. For irreversible changes, deploy a fix forward instead.
28. A Docker container works locally but fails on EC2. What do you check?
    - **Answer:** I would compare environment variables, Docker image versions, and resource limits. In our project, a Spring Boot container failed on EC2 because the instance had less memory, causing the JVM to hit its limit and get OOM-killed by Docker.
    - **If asked more:** I would check EC2 docker logs, verify environment variables, check file permissions for mounted volumes, compare OS architecture, and ensure ports are open in the security group.
29. A REST API has inconsistent error responses. How do you standardize it?
    - **Answer:** I would implement global exception handling with `@ControllerAdvice` that catches all exceptions and returns a consistent JSON format: `{status, error, message, timestamp}`. In our projects, a custom ErrorResponse DTO ensured every error had the same structure.
    - **If asked more:** I would discuss defining error codes, field-level validation errors in standard format, and documenting error schemas in OpenAPI so the frontend handles errors generically.
30. A frontend user sees pages they should not access. How do you fix RBAC?
    - **Answer:** I would check both frontend route protection and backend authorization. The frontend might hide a button but not protect the route. In our dashboard, route guards checked user roles from JWT, and backend always enforced `@PreAuthorize` on APIs.
    - **If asked more:** I would discuss the principle: frontend RBAC is UX only, backend must always validate permissions. The fix adds server-side authorization checks matching frontend route guards.
31. A stored procedure returns correct results but is slow. What metrics do you check?
    - **Answer:** I would check the actual execution plan for index scans, key lookups, and sort operations, plus logical reads (high reads = inefficiency), wait stats (blocking, I/O), and estimated vs actual row counts.
    - **If asked more:** I would use SET STATISTICS TIME and IO for detailed metrics, compare with OPTION (RECOMPILE) to rule out parameter sniffing, and check if statistics need updating.
32. A query is fast for one parameter but slow for another. What could be wrong?
    - **Answer:** This is classic parameter sniffing - the optimizer creates a plan based on the first parameter value, which may be inefficient for others. In CDMS, a date-filtered procedure was fast for 2024 (small range) but slow for 2023 (large range).
    - **If asked more:** I would use OPTION (RECOMPILE) to test, check parameter data types match column types, look for skewed data distribution, and consider OPTION (OPTIMIZE FOR UNKNOWN).
33. A microservice is receiving duplicate requests. How do you make it idempotent?
    - **Answer:** I would check a unique request ID from the client before processing, store processed IDs in a database with a unique constraint, and return the existing result for duplicates. In our inventory system, the batch reference ID was the idempotency key.
    - **If asked more:** I would discuss idempotency key generation (UUID v4), storage (Redis with TTL or DB table), response caching, and TTL-based expiry for the key.
34. A payment-like operation succeeds but client times out. How do you handle retry?
    - **Answer:** I would design the operation to be idempotent with a unique transaction ID. On retry with the same ID, the server detects it is a duplicate and returns the saved result instead of processing again.
    - **If asked more:** I would discuss implementing a status check endpoint where the client polls for completion using the transaction ID, and webhooks for async notification.
35. A report must be generated exactly once across multiple servers. How do you design it?
    - **Answer:** I would use a distributed lock (Redis or database-based) that the first server acquires before generating. The lock prevents other servers from starting the same report. If the generating server crashes, the lock expires and another server retries.
    - **If asked more:** I would also use a database table tracking report status (submitted, in-progress, completed) with optimistic locking and scheduled cleanup for abandoned "in-progress" entries.
36. An API must process 10,000 records. Do you process synchronously or asynchronously?
    - **Answer:** Asynchronously. Synchronous processing would hold the HTTP connection for minutes, causing timeouts. I would return 202 Accepted with a job ID, process asynchronously via Kafka or a thread pool, and let the client poll a status endpoint.
    - **If asked more:** I would discuss the tradeoffs: async adds complexity but prevents timeouts and enables scaling. For smaller batches (<100 records), synchronous is simpler.
37. A customer asks for near real-time dashboard updates. What architecture would you choose?
    - **Answer:** I would use WebSocket or Server-Sent Events for push-based updates. In our cold-chain project, 30-second polling was sufficient, but for sub-second real-time, WebSocket is better - the backend pushes sensor updates to clients as they arrive from Kafka.
    - **If asked more:** I would discuss the pipeline: sensor -> Kafka -> Spring Boot (WebSocket broadcast) -> React. For scaling, Redis Pub/Sub coordinates broadcasts across multiple server instances.
38. A database table is growing very large. How do you manage performance?
    - **Answer:** I would implement table partitioning (by date for time-series), archive old data to cold storage, and ensure queries use partition elimination. InfluxDB handled this naturally with time-based shards; for MSSQL, I would partition by month with a data retention policy.
    - **If asked more:** I would discuss narrow indexes on queried columns, maintaining statistics, page compression for storage/I/O savings, and read replicas for reporting queries.
39. A third-party API is slow. How do you protect your application?
    - **Answer:** I would implement circuit breaker with timeout to fail fast, return cached data as fallback, and process third-party calls asynchronously with retry. In the cold-chain project, if the weather API dependency was slow, the dashboard returned last known weather with a "stale" indicator.
    - **If asked more:** I would discuss appropriate timeouts (connection + read), bulkhead isolation to prevent slow API calls from consuming all threads, and monitoring circuit breaker state.
40. A production issue happens at midnight. How do you approach debugging?
    - **Answer:** I would first check scheduled jobs and batch processes running at midnight - they often cause resource contention. In CDMS, midnight was when the ETL pipeline ran, so a midnight issue was likely ETL-related. I would check job logs, database waits, and system resources during that window.
    - **If asked more:** I would review recent deployments before midnight, check monitoring dashboards for the specific time, look for concurrent operations that might conflict, and reproduce by running jobs in staging with similar load.

