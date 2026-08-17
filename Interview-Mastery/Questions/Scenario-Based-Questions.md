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
   - **Answer:**
      - Check if the execution plan changed - parameter sniffing or stale statistics can cause the optimizer to choose a different plan
      - In CDMS, a procedure slowed because statistics were stale after a large data load; updating statistics restored performance
      - Examine wait stats from sys.dm_os_wait_stats to identify blocking, I/O, or memory pressure on the server
      - Compare actual vs estimated rows in the execution plan — large gaps indicate stale statistics or incorrect cardinality estimates
      - Test with OPTION (RECOMPILE) to rule out parameter sniffing; if performance changes, the issue is a sniffed plan
      - Based on findings, apply index fixes, update statistics, or use a plan guide to force the optimal plan
2. A query uses an index in testing but not in production. What could be the reason?
   - **Answer:**
      - Data distribution differences between environments - production has more data with different cardinality, making the optimizer choose a scan over a seek
      - In CDMS, a query used an index in dev but scanned in production because the indexed column had many NULL values in production
      - Compare statistics between environments using sys.dm_db_stats_properties to check if stats are outdated in production
      - Check for parameter sniffing where the production parameter value results in a plan that does not favor the index
      - Examine the actual execution plan from production to see what the optimizer chose and why it rejected the index
      - Consider index maintenance (rebuild/reorganize) or query hints only after thorough testing in a staging environment with production-like data
3. A database job takes 5 hours. How would you reduce it?
   - **Answer:**
      - Analyze the execution plan to find the most expensive operations and target them with index improvements or query rewrites
      - In CDMS, I reduced a job from 5 hours to under 12 minutes by identifying missing indexes, rewriting correlated subqueries as joins, and breaking the procedure into batch operations
      - Implement batch processing by processing 10,000 records at a time instead of loading the entire dataset into memory
      - Partition large tables by date or key ranges to reduce the amount of data scanned per operation
      - Update statistics so the optimizer can generate accurate plans for each partition or batch
      - Use temp tables for intermediate results instead of CTEs, as CTEs are re-evaluated on every reference within the query
4. A report has incorrect data after ETL. How do you investigate?
   - **Answer:**
      - Trace the data flow from source to report - check staging tables, transformation logic, and aggregation steps
      - In CDMS, an incorrect report was traced to a JOIN causing data duplication; adding DISTINCT and verifying row counts at each stage identified the issue
      - Compare row counts at each ETL stage — staging, transformation, and final table — to pinpoint where data is lost or duplicated
      - Check NULL handling in transformations, as NULLs in JOINs or WHERE clauses can silently filter out valid rows
      - Examine transformation SQL for logic errors such as incorrect JOIN conditions, missing WHERE filters, or unintended aggregations
      - Compare the report output against a manually calculated sample using a known subset of data to validate correctness
5. Duplicate serial records are entering inventory. How do you prevent it?
   - **Answer:**
      - Add a unique constraint on the serial number at the database level and a duplicate check in the application layer
      - In the inventory system, we prevented duplicates by checking the serial against the verified baseline and rejecting existing records
      - Handle race conditions with Serializable isolation level for check-then-insert patterns to prevent two concurrent requests from both inserting the same serial
      - Log rejected duplicates with timestamps and source information for auditing and to identify which upstream process is causing duplicates
6. Inventory data is delayed from some channels. How do you handle it?
   - **Answer:**
      - Investigate the data flow for each delayed channel - check submission timestamps, ETL processing, and error logs
      - In our inventory system, delays were often from partners sending data in non-standard formats that failed validation
      - Set up monitoring for each channel's submission time to track the average delay and identify trends over time
      - Create alerts for missed windows so the team is notified when a channel fails to submit data within the expected timeframe
      - Implement a grace period before marking data as stale, allowing for normal network or processing delays without triggering false alarms
7. Kafka consumer lag is increasing. How do you debug it?
   - **Answer:**
      - Check consumer lag metrics in Kafka monitoring
      - Identify which partition has the highest lag
      - Check consumer logs for slow processing
      - In the cold-chain project, lag increased when InfluxDB writes bottlenecked during high sensor traffic
      - Compare consumer processing rate vs producer rate to determine if the consumer simply cannot keep up with the message volume
      - Look for poison pill messages that cause the consumer to retry indefinitely or crash, stalling the entire partition
      - Consider increasing partitions or consumer instances to parallelize consumption across more threads
      - Optimize processing with batch writes to the database instead of single-message inserts to improve throughput
8. Kafka messages are duplicated. How do you handle it?
   - **Answer:**
      - Duplicates are expected with at-least-once delivery
      - Our consumer was designed idempotently - processing the same message twice produced the same result
      - In the cold-chain project, we used deduplication IDs in InfluxDB to ignore duplicate sensor readings
      - Enable idempotent producers in Kafka to reduce duplicates at the broker level by setting the producer property `enable.idempotence=true`
      - Track processed message IDs in Redis with a TTL to quickly detect and skip duplicates without querying the database each time
      - Design business logic to tolerate duplicates so that even without infrastructure-level deduplication, processing the same event twice does not corrupt data
9. Kafka messages are out of order. How do you handle it?
   - **Answer:**
      - Out-of-order messages happen when producers retry or networks delay
      - Each sensor reading had a timestamp for sorting
      - For strict ordering, we used a partition key on sensor ID, ensuring messages for the same sensor stayed in order
      - Use partition keys consistently so all messages for a given entity go to the same partition, which guarantees ordering within that partition
      - Handle late-arriving data with a tolerance window by buffering messages for a short period and sorting by event timestamp before processing
      - Consider the tradeoff between ordering and parallelism — more partitions increase throughput but per-partition ordering means cross-partition ordering requires application-level sorting
10. A Kafka consumer keeps failing on one message. What do you do?
    - **Answer:**
       - Check consumer logs for the specific error and examine the problematic message
       - If corrupt, skip it using a dead-letter queue pattern
       - We configured a SeekToCurrentErrorHandler that retried a few times then sent to a DLQ topic
       - Implement a dead-letter topic (DLQ) for failed messages so they are not lost but also do not block the consumer from processing subsequent messages
       - Set alerts for DLQ entries using Kafka monitoring tools so the team is notified immediately when messages start failing
       - Conduct periodic review of DLQ messages to identify patterns and fix upstream data issues that are causing the failures
11. AWS Lambda is timing out. How do you debug it?
    - **Answer:**
       - Check CloudWatch Logs, Lambda timeout configuration, and identify the slow operation
       - In the cold-chain project, a Lambda timed out because the downstream Kafka publish had network connectivity issues to the EC2 broker
       - Increase timeout temporarily as a short-term fix while you investigate, but always aim to solve the root cause rather than just extending the timeout
       - Check memory allocation since Lambda allocates CPU proportionally to memory — increasing memory from 256 MB to 512 MB can significantly reduce execution time
       - Optimize the code by reusing database and HTTP connections across invocations and reducing payload sizes to cut down processing time
       - Consider async invocation with a destination for long-running operations so the Lambda returns immediately and the result is delivered later
12. EC2 deployment failed after Jenkins pipeline ran successfully. What do you check?
    - **Answer:**
       - SSH into EC2 and check Docker logs, disk space, and the application log
       - In our project, a deployment failed because the EC2 instance ran out of disk space from old Docker images not being cleaned up
       - Check EC2 resource usage including CPU, memory, and disk to determine if the instance has enough capacity to run the new container
       - Verify the Docker daemon is running and healthy, as it can occasionally crash or run out of disk overlay storage
       - Check the compose file for correct image tags to ensure the pipeline did not accidentally push a wrong or partial image
       - Examine application startup logs for connection failures to databases, caches, or other dependencies that may not be reachable from the EC2 network
13. API response time increased suddenly. How do you debug it?
    - **Answer:**
       - Check recent deployments, database query performance, and external API dependencies
       - In the cold-chain project, response time increased when an InfluxDB query stopped using the time index after a frontend update changed the query filter, causing full scans
       - Check APM tools or CloudWatch metrics to identify which endpoints are slow and whether the latency is in the application, database, or network layer
       - Look for slow queries by reviewing database query logs or enabling query logging for requests taking more than a threshold
       - Check for thread pool exhaustion where too many concurrent requests are waiting for limited threads, causing cascading delays
       - Check Redis cache hit ratios — a sudden drop in hits means more requests are hitting the database directly, increasing response time
       - Identify slow endpoints via detailed request logging with duration breakdowns to pinpoint exactly which internal call is the bottleneck
14. Database connections are exhausted. How do you debug it?
    - **Answer:**
       - Check the database's active connection list and identify which application or query is holding connections without releasing
       - In our Spring Boot app, a missing connection pool configuration for a long-running report query consumed all connections
       - Configure HikariCP properly with an appropriate max pool size, connection timeout to fail fast when the pool is full, and leak detection to identify code that does not close connections
       - Review code for missing connection closes, especially in error paths where exceptions may bypass the finally block or try-with-resources
       - Add monitoring on connection pool metrics via Spring Boot Actuator to track active connections, pending threads, and connection creation rates in real time
15. Redis cache has stale data. How do you fix it?
    - **Answer:**
       - Check TTL configuration and cache invalidation strategy
       - Stale data means TTL is too long or cache is not invalidated when source data changes
       - In our cold-chain project, we set appropriate TTLs and invalidated cache entries when new sensor data arrived
       - Publish invalidation events on data updates so that when source data changes, the corresponding cache entry is evicted immediately rather than waiting for TTL expiry
       - Use shorter TTLs as a quick fix to reduce the staleness window while you implement proper event-driven invalidation
       - Use write-through caching for data that changes frequently so that every write updates both the cache and the database simultaneously
16. A scheduled job runs on all pods and sends duplicate emails. How do you solve it?
    - **Answer:**
       - Use a distributed lock with Redis via Redisson or a database lock table, so only one instance acquires the lock and executes the job
       - We used `@SchedulerLock` where the first Spring Boot instance to acquire the lock ran the job
       - Set a lock TTL (lease time) that is slightly longer than the maximum expected job duration to prevent deadlocks if the holding instance crashes
       - Handle lock acquisition failures gracefully by logging a warning and skipping the execution rather than throwing an exception that could restart the pod
       - Monitor which instance executed each run by logging the instance ID with the lock owner, making it easy to trace which pod actually processed the job
17. A scheduled job is missing executions. How do you monitor it?
    - **Answer:**
       - Add logging at job start and end with timestamps and create a check verifying the last successful execution time is within expected intervals
       - In our inventory system, a health check endpoint reported the last reconciliation job run time, alerting if overdue
       - Expose job execution metrics via Spring Boot Actuator as custom metrics so they are visible in monitoring dashboards
       - Create a Grafana dashboard for success/failure rates with panels for job duration trends and failure counts over time
       - Configure alerts for missed schedules using Prometheus alerting rules or CloudWatch alarms that trigger when a job has not run within the expected window
18. JWT works locally but fails behind load balancer. What do you check?
    - **Answer:**
       - Check if the load balancer strips or modifies headers, particularly the Authorization header
       - In our cold-chain project, JWT failed on EC2 because SSL termination at the load balancer changed the protocol, and the `secure` flag on JWT cookies did not match
       - Also check X-Forwarded-Proto and X-Forwarded-For headers, as the application needs to trust these when behind a proxy to correctly determine the original request protocol and client IP
       - Verify Spring Security's requiresSecure() matches the actual protocol by configuring it to trust the X-Forwarded-Proto header from the load balancer
19. Users get 403 after enabling CSRF. How do you fix it?
    - **Answer:**
       - For our REST APIs used by a SPA, we disabled CSRF protection because it is not needed for token-based authentication - CSRF is primarily for cookie-based auth
       - We called `http.csrf().disable()` in Spring Security
       - SPAs using JWT in Authorization headers are not vulnerable to CSRF since browsers do not auto-include custom Authorization headers for cross-origin requests, making the CSRF token exchange unnecessary
20. Admin role update is not reflected immediately. How do you design it?
    - **Answer:**
       - Ensure role changes either force re-login (if stored in JWT) or check from the database on every request
       - In our system, roles in JWT required re-login
       - For immediate effect, database checks on every API call would be needed
       - The tradeoff is clear: JWT roles are fast (no DB call) but not immediately updatable, while database-checked roles are slower (extra query) but reflect real-time changes
       - A hybrid approach uses short-lived JWTs (5-15 minutes) with database fallback for sensitive operations, so role changes take effect within minutes without requiring re-login
21. Production API returns 500 but logs are unclear. How do you improve observability?
    - **Answer:**
       - Add structured logging with correlation IDs
       - Ensure exception stack traces are logged
       - Log request/response bodies at TRACE level for debugging
       - In the cold-chain project, unclear logs were fixed by adding method-level logging with input parameters and context
       - Implement Spring Boot Actuator for health checks to give you an at-a-glance view of all dependency health status without digging through logs
       - Implement distributed tracing with tools like Micrometer + Zipkin or AWS X-Ray so you can follow a single request across multiple services and identify exactly where it failed
       - Use centralized log aggregation like CloudWatch Logs or ELK so logs from all instances are searchable in one place instead of SSH-ing into individual pods
       - Set alerting based on error rate thresholds so you are notified proactively when the 500 error rate spikes rather than waiting for user reports
22. A service dependency is down. How should your service behave?
    - **Answer:**
       - The service should degrade gracefully - return cached data if available, return a meaningful error message (not 500), and not crash or hang
       - In the cold-chain project, when InfluxDB was unavailable, the API returned stale data from Redis with a `stale: true` flag
       - Implement a circuit breaker to fail fast after consecutive failures instead of letting every request wait for the full timeout, which would pile up threads
       - Use reasonable timeouts of 2-3 seconds for dependency calls so a slow dependency does not consume threads indefinitely
       - Provide fallback responses that let the frontend show appropriate UI like a "data may be delayed" banner instead of an error page
23. A dashboard becomes slow with live data. How do you optimize it?
    - **Answer:**
       - Optimize the backend (query optimization, pagination, caching) and the frontend (reduce re-renders, virtual lists, debounce updates)
       - In our cold-chain dashboard, we updated data every 30 seconds instead of on every reading and pre-aggregated time-series data in InfluxDB
       - Use WebSocket or Server-Sent Events instead of polling to push updates only when data changes, reducing unnecessary network traffic
       - Use React.memo to prevent unnecessary re-renders of components whose props have not changed
       - Use server-side rendering for initial page load to reduce the time to first meaningful paint when the dashboard has complex aggregations
24. IoT sensor sends invalid readings. How do you validate them?
    - **Answer:**
       - Apply range checks (temperature between -40 and 100C), format checks (valid JSON, required fields), and rate-of-change checks (flag if temp jumps 20 degrees in one minute)
       - In the cold-chain project, Lambda validated incoming MQTT messages before publishing to Kafka
       - Handle different invalid data types with specific strategies: reject sensor malfunction readings that return constant values, flag communication errors like gaps or checksum failures for manual review, and interpolate out-of-range values using surrounding valid readings to avoid gaps in the time series
25. IoT gateway sends data in bursts. How do you handle backpressure?
    - **Answer:**
       - Kafka acts as a buffer that absorbs bursts - producers publish at high rates while consumers process at their own pace
       - In the cold-chain project, when gateways reconnected and sent bursts, Kafka queued messages and consumers caught up gradually without data loss
       - Monitor consumer lag continuously to detect backpressure early before it becomes critical and starts impacting data freshness SLAs
       - Scale consumers by increasing partitions so that burst data can be processed in parallel across more consumer instances
       - Rate limit at ingestion if bursts exceed Kafka capacity by throttling the gateway or buffering at the edge before publishing to Kafka
26. A database migration causes backlog. How do you recover safely?
    - **Answer:**
       - Stop writes to the affected table, assess backlog size, and run batch processing to catch up
       - For CDMS, if a migration caused failures, we would restore from backup with point-in-time recovery, fix the script, and re-run during low-traffic hours
       - Test migrations on staging with production data volume and patterns before applying to production to catch performance regressions
       - Use backward-compatible migrations such as adding nullable columns instead of modifying existing ones, so old and new code can run simultaneously during deployment
       - Have a rollback script ready before running any migration, and verify it works in staging so you can revert quickly if the migration causes unexpected issues
27. A deployment needs rollback. What is your process?
    - **Answer:**
       - Stop the current deployment, restore the previous Docker image tag from Jenkins artifacts, and redeploy
       - We kept the last three successful image tags, so rollback was `docker pull app:v1.2.3-previous` and restarting containers, followed by smoke tests
       - Differentiate code rollback from database rollback — DB schema changes must be backward-compatible so that rolling back code does not break against the new schema
       - For irreversible database changes, deploy a fix-forward migration instead of rolling back the database, since data may have already been written in the new format
28. A Docker container works locally but fails on EC2. What do you check?
    - **Answer:**
       - Compare environment variables, Docker image versions, and resource limits
       - In our project, a Spring Boot container failed on EC2 because the instance had less memory, causing the JVM to hit its limit and get OOM-killed by Docker
       - Check EC2 Docker logs with `docker logs <container>` to see the exact error and exit code that caused the failure
       - Verify environment variables match between local and EC2, as missing database URLs or API keys are a common cause of startup failures
       - Check file permissions for mounted volumes, since Linux EC2 instances may run the container as a different user than your local machine
       - Compare OS architecture between local (often ARM on Mac) and EC2 (x86_64) to catch binary compatibility issues
       - Ensure ports are open in the EC2 security group so that the container can reach databases, caches, and external APIs
29. A REST API has inconsistent error responses. How do you standardize it?
    - **Answer:**
       - Implement global exception handling with `@ControllerAdvice` that catches all exceptions and returns a consistent JSON format: `{status, error, message, timestamp}`
       - In our projects, a custom ErrorResponse DTO ensured every error had the same structure
       - Define standard error codes like `VALIDATION_ERROR`, `NOT_FOUND`, `UNAUTHORIZED` so the frontend can map errors to user-friendly messages generically
       - Use field-level validation errors in a standard format such as `{field: "email", message: "must be valid email"}` inside the error response so the frontend can highlight specific form fields
       - Document error schemas in OpenAPI so the frontend team knows exactly what error shapes to expect and can handle them uniformly across the application
30. A frontend user sees pages they should not access. How do you fix RBAC?
    - **Answer:**
       - Check both frontend route protection and backend authorization
       - The frontend might hide a button but not protect the route
       - In our dashboard, route guards checked user roles from JWT, and backend always enforced `@PreAuthorize` on APIs
       - Frontend RBAC is UX only — it hides menus and buttons but does not prevent a user from navigating directly to the URL, so backend must always validate permissions
       - The fix adds server-side authorization checks matching frontend route guards, ensuring every API endpoint verifies the caller has the required role before returning data
31. A stored procedure returns correct results but is slow. What metrics do you check?
    - **Answer:**
       - Check the actual execution plan for index scans, key lookups, and sort operations
       - Check logical reads (high reads = inefficiency)
       - Check wait stats (blocking, I/O)
       - Check estimated vs actual row counts
       - Use SET STATISTICS TIME and IO to get detailed CPU time and logical read counts that help quantify the exact cost of each operation
       - Compare with OPTION (RECOMPILE) to rule out parameter sniffing as a cause of the slow plan
       - Check if statistics need updating by examining the last_updated column in sys.dm_db_stats_properties, as outdated stats lead to poor cardinality estimates
32. A query is fast for one parameter but slow for another. What could be wrong?
    - **Answer:**
       - This is classic parameter sniffing - the optimizer creates a plan based on the first parameter value, which may be inefficient for others
       - In CDMS, a date-filtered procedure was fast for 2024 (small range) but slow for 2023 (large range)
       - Use OPTION (RECOMPILE) to test — if the slow parameter becomes fast, parameter sniffing is confirmed
       - Check parameter data types match column types to avoid implicit conversions that prevent index usage
       - Look for skewed data distribution where one parameter value matches a few rows (index seek) while another matches millions (scan is actually faster)
       - Consider OPTION (OPTIMIZE FOR UNKNOWN) to let the optimizer use average density statistics instead of being sniffed to a specific value
33. A microservice is receiving duplicate requests. How do you make it idempotent?
    - **Answer:**
       - Check a unique request ID from the client before processing
       - Store processed IDs in a database with a unique constraint
       - Return the existing result for duplicates
       - In our inventory system, the batch reference ID was the idempotency key
       - Generate idempotency keys using UUID v4 for globally unique identifiers that clients can include in every request
       - Store idempotency keys in Redis with TTL for fast lookups, or in a database table if you need to return cached responses for duplicate requests that arrive hours later
       - Use response caching so that when a duplicate request arrives, the stored response is returned immediately without re-executing the business logic
       - Set TTL-based expiry for keys so that old idempotency records do not accumulate indefinitely and waste storage
34. A payment-like operation succeeds but client times out. How do you handle retry?
    - **Answer:**
       - Design the operation to be idempotent with a unique transaction ID
       - On retry with the same ID, the server detects it is a duplicate and returns the saved result instead of processing again
       - Implement a status check endpoint where the client polls for completion using the transaction ID, so it can verify the operation completed even after a timeout
       - Use webhooks for async notification so the server can push the result to the client when processing finishes, eliminating the need for repeated polling
35. A report must be generated exactly once across multiple servers. How do you design it?
    - **Answer:**
       - Use a distributed lock (Redis or database-based) that the first server acquires before generating
       - The lock prevents other servers from starting the same report
       - If the generating server crashes, the lock expires and another server retries
       - Also use a database table tracking report status (submitted, in-progress, completed) with optimistic locking via a version column, so concurrent updates are detected and rejected
       - Schedule cleanup for abandoned "in-progress" entries that exceed a timeout threshold, releasing the claim so another server can retry the generation
36. An API must process 10,000 records. Do you process synchronously or asynchronously?
    - **Answer:**
       - Asynchronously - synchronous processing would hold the HTTP connection for minutes, causing timeouts
       - Return 202 Accepted with a job ID
       - Process asynchronously via Kafka or a thread pool
       - Let the client poll a status endpoint
       - Async adds complexity with job tracking, status updates, and error handling, but it prevents timeouts and enables independent scaling of the processing layer
       - For smaller batches under 100 records, synchronous processing is simpler and avoids the overhead of job management
37. A customer asks for near real-time dashboard updates. What architecture would you choose?
    - **Answer:**
       - Use WebSocket or Server-Sent Events for push-based updates
       - In our cold-chain project, 30-second polling was sufficient, but for sub-second real-time, WebSocket is better - the backend pushes sensor updates to clients as they arrive from Kafka
       - The pipeline flows: sensor -> Kafka -> Spring Boot (WebSocket broadcast) -> React
       - For scaling across multiple server instances, use Redis Pub/Sub to coordinate broadcasts so that a message received by one server is relayed to all connected clients regardless of which server they are connected to
38. A database table is growing very large. How do you manage performance?
    - **Answer:**
       - Implement table partitioning (by date for time-series), archive old data to cold storage, and ensure queries use partition elimination
       - InfluxDB handled this naturally with time-based shards; for MSSQL, I would partition by month with a data retention policy
       - Use narrow indexes on only the columns actually used in WHERE and JOIN clauses to minimize index size and maintenance overhead
       - Maintain statistics so the optimizer can generate efficient plans even as data distribution changes over time
       - Use page compression for storage and I/O savings, which can reduce table size by 50-70% for text-heavy data
       - Use read replicas for reporting queries so that heavy analytical workloads do not compete with transactional queries for resources on the primary
39. A third-party API is slow. How do you protect your application?
    - **Answer:**
       - Implement circuit breaker with timeout to fail fast, return cached data as fallback, and process third-party calls asynchronously with retry
       - In the cold-chain project, if the weather API dependency was slow, the dashboard returned last known weather with a "stale" indicator
       - Set appropriate timeouts for both connection and read separately, for example 2 seconds for connection and 5 seconds for read, to fail fast when the external service is unresponsive
       - Use bulkhead isolation to prevent slow API calls from consuming all threads in the thread pool, keeping other endpoints responsive even when the third-party dependency is degraded
       - Monitor circuit breaker state transitions (closed -> open -> half-open) using metrics to track how often each dependency fails and plan capacity or find alternatives
40. A production issue happens at midnight. How do you approach debugging?
    - **Answer:**
       - First check scheduled jobs and batch processes running at midnight - they often cause resource contention
       - In CDMS, midnight was when the ETL pipeline ran, so a midnight issue was likely ETL-related
       - Check job logs, database waits, and system resources during that window
       - Review recent deployments before midnight, as a release deployed in the evening could be the cause if the issue started shortly after
       - Check monitoring dashboards for the specific time to see CPU, memory, disk, and network patterns that correlate with the issue start time
       - Look for concurrent operations that might conflict, such as a database maintenance job running alongside the ETL pipeline competing for the same resources
       - Reproduce by running jobs in staging with similar load to validate that the issue is reproducible and confirm the root cause before applying a fix
