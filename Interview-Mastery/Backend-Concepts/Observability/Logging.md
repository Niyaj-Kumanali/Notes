# Logging

## 1. Executive Summary

Logging is the practice of recording application events, errors, and state changes for debugging, monitoring, auditing, and analysis. In distributed systems, effective logging requires structured formats, centralized aggregation, correlation IDs, and appropriate log levels. Modern logging follows the "observability" paradigm: logs are part of a triad with metrics and traces, providing the context needed to understand system behavior.

## 2. Core Theory

### Log Levels (defined by severity)

| Level | Purpose | Example |
|-------|---------|---------|
| TRACE | Fine-grained debug details | Method entry/exit |
| DEBUG | Development debugging | SQL queries, variable values |
| INFO | Normal application events | Service start/stop, significant operations |
| WARN | Unexpected but handled | Retry attempts, deprecated API usage |
| ERROR | Errors that need attention | Failed operations, exceptions |
| FATAL | Application cannot continue | Out of memory, configuration error |

### Structured vs Unstructured Logging

**Unstructured (plain text):**
```
2024-01-01 12:00:00 [INFO] User 123 created order 456 for $50.00
```
Hard to parse, search, and analyze programmatically.

**Structured (JSON):**
```json
{
  "timestamp": "2024-01-01T12:00:00Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Order created successfully",
  "userId": 123,
  "orderId": 456,
  "amount": 50.00,
  "duration": 150,
  "traceId": "abc123def456"
}
```
Machine-readable, filterable, searchable.

## 3. Under-the-Hood Deep Dive

### Logging Pipeline

```
[Application] -> [Logging Framework] -> [Appender] -> [Aggregator] -> [Storage] -> [Analysis]
     |                    |                  |             |              |
  log.info()        Logback/Log4j2      File/Socket     Filebeat/     Elasticsearch/
                                        Appender        Fluentd        Loki/S3
```

### Asynchronous Logging

Synchronous logging adds latency to each operation. Async logging uses a separate thread:

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="FILE" />
        <queueSize>1024</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <neverBlock>true</neverBlock>
    </appender>
</configuration>
```

### Log Correlation in Distributed Systems

A correlation ID (trace ID) is generated at the entry point and propagated across all services:

```
[API Gateway] generates trace-id: abc123
  |
  v
[Order Service] logs with trace-id: abc123
  |
  v
[Payment Service] logs with trace-id: abc123
```

## 4. Production Code Examples

### Spring Boot Logback Configuration

```xml
<!-- src/main/resources/logback-spring.xml -->
<configuration>
    <springProperty scope="context" name="appName" source="spring.application.name"/>
    <springProperty scope="context" name="env" source="spring.profiles.active"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeContext>false</includeContext>
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <level>level</level>
                <logger>logger</logger>
                <message>message</message>
            </fieldNames>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/${appName}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/${appName}.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>500MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="FILE"/>
        <queueSize>4096</queueSize>
        <neverBlock>true</neverBlock>
        <includeCallerData>false</includeCallerData>
    </appender>

    <root level="INFO">
        <appender-ref ref="ASYNC"/>
        <appender-ref ref="CONSOLE"/>
    </root>

    <logger name="com.example" level="DEBUG"/>
    <logger name="org.springframework" level="WARN"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
</configuration>
```

### Structured Logging with MDC

```java
@Component
public class LoggingFilter implements Filter {

    private static final String TRACE_ID_KEY = "traceId";
    private static final String USER_ID_KEY = "userId";
    private static final String REQUEST_ID_KEY = "requestId";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;

        // Generate or propagate trace ID
        String traceId = httpRequest.getHeader("X-Trace-Id");
        if (traceId == null || traceId.isEmpty()) {
            traceId = UUID.randomUUID().toString().replace("-", "");
        }

        // Set MDC context
        MDC.put(TRACE_ID_KEY, traceId);
        MDC.put(REQUEST_ID_KEY, UUID.randomUUID().toString().substring(0, 8));

        String userId = httpRequest.getHeader("X-User-Id");
        if (userId != null) {
            MDC.put(USER_ID_KEY, userId);
        }

        try {
            chain.doFilter(request, response);
        } finally {
            // Clear MDC to prevent memory leaks
            MDC.clear();
        }
    }
}
```

### Structured Logging with Logstash Encoder

```java
@Service
@Slf4j
public class OrderService {

    public Order createOrder(CreateOrderRequest request) {
        // Structured logging with markers
        log.info("Creating order for user {}",
            request.getUserId(),
            StructuredArguments.keyValue("userId", request.getUserId()),
            StructuredArguments.keyValue("itemCount", request.getItems().size()));

        long start = System.nanoTime();
        try {
            Order order = new Order(request);
            orderRepository.save(order);

            log.info("Order created successfully",
                StructuredArguments.keyValue("orderId", order.getId()),
                StructuredArguments.keyValue("total", order.getTotalAmount()),
                StructuredArguments.keyValue("durationMs",
                    (System.nanoTime() - start) / 1_000_000));

            return order;
        } catch (Exception e) {
            log.error("Failed to create order",
                StructuredArguments.keyValue("userId", request.getUserId()),
                StructuredArguments.keyValue("error", e.getMessage()),
                e);
            throw e;
        }
    }
}
```

### Dynamic Log Level Change

```java
@RestController
@RequestMapping("/actuator")
public class LogLevelController {

    @Autowired
    private LoggingSystem loggingSystem;

    @PutMapping("/log-level")
    public ResponseEntity<Void> setLogLevel(
            @RequestParam String packageName,
            @RequestParam String level) {
        loggingSystem.setLogLevel(packageName, LogLevel.valueOf(level.toUpperCase()));
        return ResponseEntity.ok().build();
    }
}

// Usage:
// PUT /actuator/log-level?package=com.example.order&level=DEBUG
```

### Logging Aspect for Service Methods

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("@annotation(LoggedExecution)")
    public Object logExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().toShortString();
        Object[] args = joinPoint.getArgs();

        log.info("Method called: {} with args: {}",
            methodName, new ObjectMapper().writeValueAsString(args));

        long start = System.nanoTime();
        try {
            Object result = joinPoint.proceed();
            log.info("Method completed: {} in {}ms",
                methodName, (System.nanoTime() - start) / 1_000_000);
            return result;
        } catch (Exception e) {
            log.error("Method failed: {} after {}ms. Error: {}",
                methodName, (System.nanoTime() - start) / 1_000_000, e.getMessage());
            throw e;
        }
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LoggedExecution {}
```

### Log Aggregation with Filebeat

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
    json.keys_under_root: true
    json.add_error_key: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "app-logs-%{+yyyy.MM.dd}"

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - add_kubernetes_metadata:
      host: ${HOSTNAME}
      matchers:
        - logs_path:
            logs_path: "/var/log/app/"
```

## 5. Real-World Scenarios

### Incident Debugging Flow

```
1. Alert: Error rate > 5%
2. Grafana -> Logs view: Filter by error + traceId
3. Kibana: Search for error logs in affected time window
4. Find correlation ID from first error log
5. Follow traceId across all services in Kibana
6. Identify root cause: NullPointerException due to missing field
7. Fix: Add null check
```

### Audit Logging for Compliance

```java
@Component
public class AuditLogger {

    public void logEvent(String action, String resourceType,
            String resourceId, String userId, String details) {
        log.info("AUDIT: {} {} {} by user {}: {}",
            action, resourceType, resourceId, userId, details);
    }
}

// Usage in sensitive operations
@Service
public class PaymentService {
    public void processRefund(String orderId, String adminUserId) {
        // Process refund
        auditLogger.logEvent("REFUND", "ORDER", orderId, adminUserId,
            "Full refund processed");
    }
}
```

## 6. Performance

### Logging Overhead

| Operation | Overhead |
|-----------|----------|
| log.info("static string") | < 1 microsecond |
| log.info("with {} {}", arg1, arg2) | 1-5 microseconds |
| log.info(JSON string) | 10-50 microseconds |
| String concatenation in hot path | 1-100 microseconds |
| File appender (sync) | 10-100 microseconds |
| Async appender | 1-5 microseconds (just enqueue) |

### Best Practices for Performance
- Use parameterized logging, not string concatenation.
- Use async appenders in production.
- Avoid logging in tight loops.
- Use log level guards for expensive computations:
```java
// WRONG: evaluates toString() even at INFO level
log.debug("Heavy data: {}", computeExpensiveString());

// RIGHT: guard with level check
if (log.isDebugEnabled()) {
    log.debug("Heavy data: {}", computeExpensiveString());
}
```

## 7. Security

### Sensitive Data Redaction

```java
@Component
public class SensitiveDataFilter {

    private static final Set<String> SENSITIVE_FIELDS = Set.of(
        "password", "secret", "token", "authorization",
        "creditCard", "ssn", "cvv"
    );

    public String sanitize(String message) {
        // Use Logstash encoder with custom filters
        return message;
    }
}

// Logback filter for sensitive data
public class SensitiveDataConverter extends MessageConverter {
    @Override
    public String convert(ILoggingEvent event) {
        String message = event.getFormattedMessage();
        // Mask credit card numbers
        message = message.replaceAll("\\b(\\d{4})[- ]?(\\d{4})[- ]?(\\d{4})[- ]?(\\d{4})\\b",
            "$1-****-****-$4");
        // Mask passwords
        message = message.replaceAll("password[=:]['\"]?[^\\s'\"]+['\"]?", "password=****");
        return message;
    }
}
```

### Security Best Practices
- Never log PII, passwords, tokens, or credit card numbers.
- Use structured logging to selectively include/exclude fields.
- Encrypt log files at rest.
- Control access to log aggregation systems.
- Set retention policies for log data.
- Audit log access.

## 8. Common Mistakes

### Mistake 1: Logging Exceptions Without Context
```java
// WRONG - no context to reproduce the issue
try {
    processOrder(orderId);
} catch (Exception e) {
    log.error("Error processing order", e);
}

// RIGHT - include identifiers
try {
    processOrder(orderId);
} catch (Exception e) {
    log.error("Error processing order {} for user {}",
        orderId, userId, e);
}
```

### Mistake 2: Synchronous Logging in Hot Path
Blocking file I/O on every log statement under high load.

### Mistake 3: Logging Too Much (Info is the new Debug)
Noisy logs hide real issues in the noise.

### Mistake 4: Logging Too Little
No logs for production incidents -> blind debugging.

### Mistake 5: No Correlation IDs
Without trace IDs, reconstructing a request's flow across services is impossible.

## 9. Senior Engineer Perspective

### Logging Strategy

1. **What to log:**
   - Service entry/exit with key parameters.
   - Business events (order created, payment processed).
   - Errors with full stack trace and context.
   - External service calls (URL, response time, status).
   - Performance metrics (timings for key operations).

2. **What NOT to log:**
   - Sensitive data (passwords, tokens, PII).
   - Full request/response bodies in production.
   - Debug-level logs in production (unless investigating).
   - In tight loops (log outside the loop).

3. **Log format standards:**
   - All services use structured JSON.
   - Common fields: @timestamp, level, logger, message, service, traceId, spanId.
   - Namespace conventions: `method=createOrder, status=success, duration=150`.

### Log Levels in Production

| Level | Production | Explanation |
|-------|------------|-------------|
| ERROR | Yes | Incidents that need investigation |
| WARN | Yes | Issues that should be watched |
| INFO | Selective | Significant business events |
| DEBUG | No (temporarily per service) | On-demand debugging |
| TRACE | No | Development only |

### Log Retention Policy

| Environment | Retention | Storage |
|-------------|-----------|---------|
| Development | 7 days | Local files |
| Staging | 30 days | Centralized |
| Production - Hot | 7 days | Elasticsearch |
| Production - Warm | 30 days | Elasticsearch |
| Production - Cold | 1 year | S3/Glacier |

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What are log levels?
   **A:** TRACE, DEBUG, INFO, WARN, ERROR, FATAL - categorize log message severity.

2. **Q:** What is structured logging?
   **A:** Logging in a machine-readable format (JSON) instead of plain text, making logs searchable and analyzable.

3. **Q:** What is a correlation ID?
   **A:** A unique identifier propagated across service calls to correlate all logs related to a single request.

4. **Q:** What is the difference between synchronous and asynchronous logging?
   **A:** Sync: log statements block until written. Async: log statements are queued and written by a background thread.

5. **Q:** What is log aggregation?
   **A:** Collecting logs from multiple sources into a centralized platform (ELK, Loki) for search and analysis.

6. **Q:** What is MDC in logging?
   **A:** Mapped Diagnostic Context: a map of key-value pairs attached to each log message (trace ID, user ID).

7. **Q:** What information should always be in a log?
   **A:** Timestamp, level, logger name, message, trace ID, service name.

8. **Q:** What is the ELK stack?
   **A:** Elasticsearch (storage/search), Logstash (processing), Kibana (visualization).

9. **Q:** How do you change log level at runtime in Spring Boot?
   **A:** Using Actuator endpoint: POST /actuator/loggers/{packageName} with body {"configuredLevel": "DEBUG"}.

10. **Q:** What is log rotation?
    **A:** Archiving old log files based on time or size to prevent disk exhaustion.

### Medium

11. **Q:** How do you implement distributed logging across microservices?
    **A:** Propagate trace ID via HTTP headers (X-Trace-Id). All services log with this trace ID. Centralized aggregation (ELK/Loki) allows searching across services.

12. **Q:** What is the difference between Logback and Log4j2?
    **A:** Logback is the default Spring Boot logger. Log4j2 offers async loggers with disruptor (higher throughput), plugin architecture, and lambda support.

13. **Q:** How do you handle sensitive data in logs?
    **A:** Use custom filters/message converters to mask/redact sensitive fields. Never log full request bodies. Use structured logging to selectively include fields.

14. **Q:** Explain the concept of "structured logging as data".
    **A:** Each log is a structured data point (JSON) with typed fields. This enables querying, aggregation, and alerting on specific fields.

15. **Q:** How do you prevent logging from becoming a performance bottleneck?
    **A:** Use async appenders, log level guards for expensive computations, parameterized logging (no concatenation), and avoid logging in hot paths.

16. **Q:** What is the FATAL log level?
    **A:** Indicates the application cannot continue (OOM, configuration error). After logging FATAL, the application usually exits.

17. **Q:** How do you correlate logs from a batch job?
    **A:** Generate a job execution ID, log it in all batch-related operations, use as correlation ID.

18. **Q:** What is the difference between Logstash and Fluentd?
    **A:** Both are log aggregators. Logstash (ELK) has more plugins, heavier. Fluentd is lighter, more cloud-native, preferred in Kubernetes.

19. **Q:** How do you handle high-volume logging (100K+ logs/second)?
    **A:** Async logging, avoid blocking appender. Use batching (Logstash, Fluentd). Sample or filter debug/trace in production.

20. **Q:** What is a logging pattern and why use one?
    **A:** A format template. Logback pattern: %d{ISO8601} [%level] %logger{36} - %msg%n. Using JSON encoder is better for structured logging.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Design a logging system that handles 1M log events per second.
    **A:** Async logging with disruptor-based appender. Buffer writes (batch). Write to local files, ship via Fluentd with compression. Use Kafka as intermediate buffer before Elasticsearch. Hot tier (7d) in ES, warm (30d) in S3-backed ES.

2. **Q:** How do you implement a logging framework that respects data privacy regulations (GDPR)?
    **A:** Dynamic field redaction based on data classification. Schema with PII tags. Automated PII detection in log pipeline. Encryption at rest. Deletion API for user data. Configurable retention.

3. **Q:** Design a log-based alerting system.
    **A:** Elasticsearch Watcher, Promtail + Loki rules, or custom: stream logs to Kafka, process with Kafka Streams, detect patterns (error rate > threshold), send alert via PagerDuty. Real-time: <1 minute from log to alert.

4. **Q:** How do you implement a centralized logging system for 500+ microservices?
    **A:** Standardized logging library (JSON structure, mandatory fields). Filebeat/Fluentd DaemonSet per Kubernetes node. Kafka as buffering layer. Logstash for enrichment/transformation. Elasticsearch cluster (hot-warm-cold). Kibana with pre-configured dashboards.

5. **Q:** Design a log sampling strategy for high-volume services.
    **A:** Head-based: sample first N of each request type. Tail-based: keep all ERROR, sample 10% of INFO, 1% of DEBUG. Adaptive: reduce sampling rate during normal operation, increase during incidents.

6. **Q:** How do you implement structured logging across different programming languages?
    **A:** Standard JSON schema shared across all services. Common fields: @timestamp, level, service, traceId, spanId, message. Per-language libraries implement the same schema.

7. **Q:** Design a multi-tenant logging system with data isolation.
    **A:** Tenant ID in each log entry. Elasticsearch index per tenant (for enterprise). Shared index with tenant-level RBAC (for standard). Retention policy per tenant tier.

8. **Q:** How do you handle log format migrations without breaking existing queries?
    **A:** Add new fields, don't remove old ones. Schema version in log metadata. Elasticsearch mapping with "dynamic: false". Query old fields by version filter. Deprecate old fields after migration period.

9. **Q:** Design a system to detect log-based anomalies (log4shell, attacks).
    **A:** Real-time log processing with anomaly detection: sudden increase in ERROR level, unusual patterns (JNDI lookups), blocked IPs. ML model trained on normal patterns. Alert on deviations.

10. **Q:** How do you implement a cost-effective logging system for 100TB/day?
    **A:** Filter and drop low-value logs at source. Sample DEBUG/TRACE. Compress (gzip/zstd). Tiered storage: hot (SSD, 3d), warm (HDD, 30d), cold (S3 Glacier, 1y). Use cost-effective storage like Loki + S3.

### System Design

11. **Q:** Design a logging pipeline for a Kubernetes-based microservices platform.
    **A:** DaemonSet collector (Fluent Bit) per node reads container stdout/stderr. Enriches with Kubernetes metadata (pod, namespace, container). Output to Kafka. Logstash transforms. Elasticsearch stores. Kibana for visualization.

12. **Q:** Design an audit logging system for a financial platform.
    **A:** Every state change event logged with: user, action, resource, before/after values, timestamp, IP. Immutable audit store (append-only, write-protected). Checksum chain for integrity verification. Retention: 7+ years.

13. **Q:** Design a log shipping system for an on-premise to cloud migration.
    **A:** On-premise agents send to on-prem Kafka. Kafka MirrorMaker replicates to cloud Kafka. Cloud Logstash processes and sends to Elasticsearch Cloud. Dual-search during migration.

14. **Q:** Design a real-time user activity logging system for a SaaS platform.
    **A:** User action -> API/Kafka -> Logstash -> Elasticsearch. Real-time dashboard with user activity, feature usage, error funnel.

15. **Q:** Design a log-based cost attribution system (per team/service/feature).
    **A:** Each log has cost center tag. Log volume tracked per tag. Measure: per-service log volume (GB/day), per-team, per-feature. Dashboard for cost allocation.

16. **Q:** Design a log retention and archival system.
    **A:** Hot (Elasticsearch, 7d), Warm (Elasticsearch with S3 snapshot, 30d), Cold (S3 Glacier, 1y), Deep Archive (S3 Glacier Deep Archive, 7y). Automated lifecycle transitions.

17. **Q:** Design a federated logging system across multiple data centers.
    **A:** Local Logstash/ES per datacenter. Cross-DC queries via Kibana cross-cluster search. Global trace ID enables correlation across DCs.

18. **Q:** Design a system to log and analyze third-party API calls.
    **A:** Interceptor wraps all HTTP client calls. Logs: URL, method, request/response summary, status, duration, error. Structured logs in central ES. Dashboard: SLAs, error rates, latency per API.

19. **Q:** Design a logging system for IoT device messages.
    **A:** Devices send logs to MQTT broker -> Kafka -> Logstash -> ES. High write throughput (100K msg/s). Time-based indices for efficient query. Retention based on device type.

20. **Q:** Design a system to automatically generate runbooks from logs.
    **A:** Cluster error logs by stack trace + context. For each cluster, extract: service, error type, frequency, affected resources. Generate runbook: error pattern, impact, resolution steps (from git history), alert threshold.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a unified observability data model that combines logs, metrics, and traces.
    **A:** All share common fields: service, timestamp, traceId, spanId. Logs have message + context. Metrics have name + value + dimensions. Traces have spans with timing. Query across all three: "find logs for traces with error count > 10".

2. **Q:** How do you implement a chaos engineering informed logging strategy?
    **A:** Inject log anomalies during chaos experiments. Validate that monitoring/alerting fires correctly on those patterns. Use canary log analysis to detect silent failures.

3. **Q:** Design a logging system that supports replay of past state for debugging.
    **A:** Event sourcing: all state changes logged. Replay: reconstruct state at any point in time. Requires complete event log with ordered, immutable entries.

4. **Q:** How do you design a zero-trust logging architecture?
    **A:** All logs signed with service identity (mTLS). Immutable storage (WORM). Access control: encrypt logs, decrypt at query time. Audit: every log read is logged. Integrity: Merkle tree of log entries.

5. **Q:** Design a logging system that costs < $0.01 per GB.
    **A:** Loki + S3/Object storage (no indexing overhead). Fluent Bit (lightweight collector). Filter 90% of debug/info at source. Use log streams directly to S3 for long-term.

6. **Q:** How do you implement automatic PII detection and redaction in logs?
    **A:** ML-based or regex-based detector in log pipeline. Classify fields: PII, non-PII. Hash/PII in logs (reversible for authorized queries). Quarantine logs with undetected PII for manual review.

7. **Q:** Design a multi-cloud log management system.
    **A:** Cloud-agnostic collector (Fluentd/Fluent Bit). Write to S3-compatible storage across clouds. Query engine (Trino/Presto) on top of multi-cloud storage. Unified query interface.

8. **Q:** How do you design a logging strategy for serverless/FaaS applications?
    **A:** CloudWatch Logs (AWS Lambda) to Lambda Subscription Filter -> Elasticsearch. Or use OpenTelemetry collector as sidecar. Structured JSON stdout. External logging library that flushes context.

9. **Q:** Design a system for log-based distributed debugging (interactive query across services).
    **A:** Trace-driven log correlation: enter trace ID, get all logs across all services for that trace. Real-time streaming of logs for live debugging. Pause/resume log collection for active traces.

10. **Q:** How do you measure the business value of logging investment?
    **A:** MTTR (Mean Time to Resolve) before vs after: if MTTR drops from 4h to 30min, value = (savings in engineer hours + reduced downtime cost). Also: compliance cost avoidance, customer retention from faster issue resolution.

## 13. Debugging & Troubleshooting

### Common Logging Issues

**Issue: Logs not appearing**
- Check log level: is the package configured correctly?
- Check appender configuration: file path, permissions.
- Check async appender: was the queue full and discarded?
- Check log rotation: disks full?

**Issue: Too many logs, can't find relevant ones**
- Increase log level temporarily for noisy packages.
- Use structured logging with filterable fields.
- Create saved searches/filters in Kibana.

**Issue: Logs don't have enough context**
- Ensure MDC is populated with trace ID, user ID, request ID.
- Include key business identifiers in each log.
- Log entry and exit of important operations.

**Issue: Log aggregation pipeline broken**
- Check Filebeat/Fluentd status.
- Check Kafka consumers (lag).
- Check Elasticsearch disk space, cluster health.

## 14. Comparison Section

### Logging Frameworks

| Framework | Async | Performance | Configuration | Spring Boot Default |
|-----------|-------|-------------|---------------|---------------------|
| Logback | Yes | Good | XML | Yes (default) |
| Log4j2 | Yes (Disruptor) | Excellent | XML/JSON | Available |
| java.util.logging | No | Poor | Properties | No |
| SLF4J | Facade only | N/A | N/A | Facade |

### Centralized Logging Solutions

| Solution | Storage | Query | Cost | Best For |
|----------|---------|-------|------|----------|
| ELK (Elasticsearch + Kibana) | Elasticsearch | Powerful DSL | Medium | Full-featured |
| Loki + Grafana | Object store | LogQL (labels) | Low | Kubernetes, light |
| Splunk | Proprietary | SPL | High | Enterprise, compliance |
| Datadog | Cloud | Custom | Per GB | SaaS, integrated |
| CloudWatch | AWS | CloudWatch Insights | Per GB | AWS-native |

### Structured vs Unstructured Logging

| Aspect | Structured (JSON) | Unstructured (Text) |
|--------|------------------|---------------------|
| Searchability | High (field-level) | Low (text search only) |
| Machine parsing | Easy | Difficult |
| Human readability | Lower | Higher |
| Schema evolution | Flexible | Not applicable |
| Tooling support | Excellent | Limited |

## 15. Revision Notes

### Quick Recap
- **Log Levels**: TRACE < DEBUG < INFO < WARN < ERROR < FATAL.
- **Structured Logging**: JSON format, machine-readable.
- **MDC**: Context propagation (traceId, userId).
- **Async Logging**: Non-blocking, queue-based.
- **Log Aggregation**: Centralized collection (ELK, Loki).
- **Correlation ID**: Trace requests across services.
- **Log Rotation**: Prevent disk exhaustion.
- **Sensitive Data**: Never log PII/passwords.

### Best Practices
1. Use structured (JSON) logging.
2. Use async appenders in production.
3. Include trace ID and service name in every log.
4. Set appropriate log levels per environment.
5. Never log sensitive data.
6. Log entry, exit, and errors of key operations.
7. Use parameterized logging (avoid string concatenation).
8. Enable dynamic log level changes.
9. Define and enforce retention policies.
10. Monitor logging pipeline health.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                     LOGGING CHEAT SHEET                            |
+-------------------------------------------------------------------+
| LEVEL     | PRODUCTION | PURPOSE                                   |
+-----------+------------+-------------------------------------------+
| ERROR     | Always     | Failures needing investigation             |
| WARN      | Always     | Should-watch conditions                   |
| INFO      | Selective  | Business events, state changes            |
| DEBUG     | Per-package| Troubleshooting (temporary)               |
| TRACE     | Never      | Development only                          |
+-----------+------------+-------------------------------------------+
| SPRING BOOT LOGGING CONFIG                                         |
+-------------------------------------------------------------------+
| application.yml:                                                    |
| logging.level.com.example=DEBUG                                    |
| logging.level.org.springframework=WARN                             |
| logging.level.org.hibernate.SQL=DEBUG                             |
|                                                                     |
| logback-spring.xml:                                                 |
| <appender class="AsyncAppender"> (performance)                     |
| <encoder class="LogstashEncoder"> (structured JSON)               |
+-------------------------------------------------------------------+
| MDC FIELDS TO INCLUDE                                              |
+-------------------------------------------------------------------+
| traceId      | Request tracing across services                     |
| spanId       | Current span in trace                               |
| userId       | Authenticated user                                  |
| requestId    | Unique request identifier                           |
| service      | Service name (spring.application.name)              |
| environment  | dev/staging/prod                                    |
| instance     | Pod/host identifier                                 |
+-------------------------------------------------------------------+
| COMMON LOGGING ANTI-PATTERNS                                       |
+-------------------------------------------------------------------+
| [ ] Logging exceptions without context                             |
| [ ] String concatenation in log messages                           |
| [ ] Synchronous logging on hot path                                |
| [ ] Logging passwords/tokens/PII                                   |
| [ ] No trace ID in distributed system logs                         |
| [ ] Logging in tight loops                                         |
| [ ] Different log formats across services                          |
| [ ] No log rotation                                                |
+-------------------------------------------------------------------+
| STRUCTURED LOG JSON EXAMPLE                                        |
+-------------------------------------------------------------------+
| {                                                                   |
|   "@timestamp": "2024-01-01T12:00:00.000Z",                        |
|   "level": "INFO",                                                  |
|   "service": "order-service",                                       |
|   "traceId": "abc123def456",                                        |
|   "userId": "user-789",                                             |
|   "message": "Order created",                                       |
|   "orderId": "order-456",                                           |
|   "amount": 50.00,                                                  |
|   "duration": 150                                                   |
| }                                                                   |
+-------------------------------------------------------------------+
```
