# Logging

---

## Overview

- **Definition:** Logging records application events, errors, and state changes for debugging, monitoring, auditing, and analysis.
- **Why It Exists:** Logs provide the detailed context needed to understand system behavior during incidents. In distributed systems, structured logging with correlation IDs enables searching across services.
- **Key Concepts:** **Log Levels** (TRACE, DEBUG, INFO, WARN, ERROR, FATAL), **Structured Logging** (JSON format for machine readability), **MDC** (Mapped Diagnostic Context for trace/user IDs), **Async Logging** (non-blocking I/O via queue), **Log Aggregation** (centralized collection via ELK/Loki).

---

## Core Concepts

- **Log Levels:** **ERROR** — failures needing investigation. **WARN** — should-watch conditions. **INFO** — significant business events (selective in production). **DEBUG/TRACE** — development only (enable temporarily per-package).
- **Structured vs Unstructured:** Structured (JSON) is machine-readable, filterable, searchable. Unstructured (plain text) is hard to parse programmatically. Always use structured logging in production.
- **Correlation ID (Trace ID):** Generated at entry point, propagated across all services via HTTP headers. Enables reconstructing a request's flow across service boundaries.
- **Async Logging:** Synchronous logging adds latency to each operation. Async logging queues messages and writes in a background thread, preventing I/O from blocking the request thread.

```java
// Structured logging with MDC
MDC.put("traceId", traceId);
MDC.put("userId", userId);
try {
    log.info("Creating order", StructuredArguments.keyValue("orderId", order.getId()));
    // ... business logic
} finally { MDC.clear(); }
```

---

## Common Mistakes

- **Logging Exceptions Without Context** — Always include identifiers (order ID, user ID) in error logs.
  - **Why it looks correct:** The stack trace shows the error and the line number — the developer assumes they can find the failing request by grepping timestamps, not realizing they need a searchable identifier.
- **Synchronous Logging in Hot Path** — Blocking file I/O under high load destroys throughput.
  - **Why it looks correct:** A single `log.info` takes microseconds — the hidden cost is that every request thread blocks on disk I/O, and under load the cumulative delay grows linearly with concurrency.
- **Logging Too Much (Info is the new Debug)** — Noisy logs hide real issues.
  - **Why it looks correct:** More data seems better for debugging — the signal-to-noise ratio degrades slowly, with no single log line announcing itself as noise.
- **No Correlation IDs** — Without trace IDs, reconstructing a request's flow across services is impossible.
  - **Why it looks correct:** Each service's logs are independently coherent — the gap only appears when you need to trace a single user's journey across 5 services.
- **Logging Sensitive Data** — Never log passwords, tokens, PII, or credit card numbers.
  - **Why it looks correct:** The data is in the log for legitimate debugging — the compliance violation is invisible until an audit or breach reveals the exposed PII.

---

## Key Design Considerations

- **Log Format Standards:** All services emit structured JSON with common fields: `@timestamp`, `level`, `service`, `traceId`, `spanId`, `message`, `userId`.
- **Dynamic Log Levels:** Use Spring Boot Actuator to change log levels at runtime without redeployment: `POST /actuator/loggers/{package}`.
- **Log Retention:** Hot storage (7d Elasticsearch), Warm (30d), Cold (1y S3/Glacier). Configure retention per environment.
- **Log-Based Alerting:** Stream logs to Kafka, detect patterns (error rate spikes, JNDI lookups), alert via PagerDuty. Use Elasticsearch Watcher or Loki rules.
- **Sensitive Data Redaction:** Use Logback message converters to mask credit cards, passwords. Never log full request/response bodies in production.
- **Async Appender Configuration:** `AsyncAppender` with `neverBlock=true` prevents logging from blocking application threads when queue is full.

---

## Real-World Scenarios

### Scenario 1: Debugging a Production Incident Without Structured Logging
**Context:** A production outage causes 500 errors on the checkout endpoint. Developers SSH into servers, grep through 10GB of plain-text log files, find log lines like `2024-01-15 14:23:01 ERROR - Process failed`. No user ID, no order ID, no trace ID. Developers spend 4 hours manually correlating timestamps across services.

**Resolution (Before):** Unstructured logging, no correlation IDs, logs on local disk — 4 hours MTTR.

**Resolution (After):** Implement structured JSON logging with trace IDs. Each log entry includes `trace_id`, `service`, `user_id`, `order_id`, and `error_category`. Logs ship to Elasticsearch. Search for `level:ERROR AND service:checkout-service AND trace_id:*`. The trace ID reveals the failing request flow. Root cause identified in 15 minutes.

```java
// Structured logging with MDC
@Component
public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String traceId = request instanceof HttpServletRequest ?
            ((HttpServletRequest) request).getHeader("X-Trace-Id") : null;
        if (traceId == null) traceId = UUID.randomUUID().toString();

        MDC.put("trace_id", traceId);
        MDC.put("service", "checkout-service");
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}

// Log entry in JSON format
// {"@timestamp":"2024-01-15T14:23:01.123Z","level":"ERROR","service":"checkout-service",
//  "trace_id":"abc123","user_id":"user456","order_id":"order789",
//  "message":"Payment processing failed","error":"CARD_DECLINED",
//  "duration_ms":2345}
```

### Scenario 2: Async Logging Preventing Thread Starvation
**Context:** A high-throughput API (10K req/s) uses synchronous logging. Each `log.info()` blocks the request thread for 1-5ms for disk I/O. Under load, logging adds 20% to request latency, and threads pile up waiting for disk writes.

**Resolution:** Switch to async logging with Logback's `AsyncAppender`. The request thread enqueues the log message (sub-microsecond) and continues processing. A background thread drains the queue and writes to disk. Configure `neverBlock=true` to drop logs when the queue is full (rather than blocking request threads).

### Scenario 3: Log-Based Alerting for Security Incidents
**Context:** An attacker is probing the API for SQL injection vulnerabilities. The WAF blocks them, but the security team doesn't know until they review weekly reports.

**Resolution:** Stream logs to a real-time analysis pipeline. Logs → Filebeat → Kafka → Logstash → Elasticsearch. A Watcher rule detects patterns: `url:*union*select*` OR `url:*or*1=1*` with status `403`. Alert sent to PagerDuty within 30 seconds of detection. Similar patterns detect brute force attempts, JNDI lookups, and excessive 404s.

## Use Cases

- **Incident debugging and root cause analysis** — investigating production outages, errors, or performance regressions
  - Structured logs with trace IDs, service names, and error categories enable centralized search across distributed systems.
  - **Avoid when:** the service is stateless and the error is easily reproducible — local reproduction may be faster than log analysis.

- **Security auditing and compliance** — tracking access to sensitive data, authentication attempts, or admin actions
  - Immutable audit logs with user IDs, timestamps, and action details. Log retention policies satisfy regulatory requirements (SOC2, PCI-DSS, HIPAA).
  - **Avoid when:** logging sensitive data (PII, passwords, credit cards) — configure log masking or redaction rules.

- **Business analytics and user behavior** — feature adoption, funnel analysis, or usage patterns
  - INFO-level logs capture business events (user signed up, order placed). Aggregated in a data warehouse for product analytics.
  - **Avoid when:** high-cardinality user-level tracking is needed — use dedicated analytics tools (Amplitude, Mixpanel) instead.

- **Alerting and proactive monitoring** — detecting error spikes, slow responses, or suspicious patterns in real-time
  - Logs streamed through a real-time analysis pipeline (Filebeat → Kafka → Logstash → Elasticsearch). Watcher rules trigger alerts on patterns.
  - **Avoid when:** the alerting metric is rate-based (e.g., requests per second) — metrics systems (Prometheus) are more efficient for numeric thresholds.

- **Distributed tracing correlation** — connecting logs across service boundaries during a single request flow
  - MDC propagates `trace_id` and `span_id` through all services. Logs from each service are correlated in the log aggregator by trace ID.
  - **Avoid when:** your system is a monolith — a single trace_id is still useful but easier to implement without distributed context propagation.

---

## Scenario-Based Questions

1. **Q: A production outage just occurred. The checkout service is returning 500 errors. Your logs are plain text files on each server. How do you find the root cause?**
    - A: You can't efficiently — this is why structured logging with centralized aggregation is essential. To fix the immediate problem, you'd grep all servers for recent ERROR logs, manually correlate with timestamps, and hope to find a pattern. For the future: implement structured JSON logging with `trace_id`, `service`, `user_id`, and `error_category`. Ship to Elasticsearch. Create a dashboard showing error rates by service and trace ID. Next outage: search `level:ERROR AND service:checkout` and trace the failing request across services.
    - **Interview follow-up:** You implement structured logging, but the `user_id` field is sometimes missing because a developer forgot to populate it in a specific code path. How do you detect and alert on missing mandatory fields in structured logs without adding runtime overhead?

2. **Q: Your application logs 100GB/day. Searching logs for a specific user's actions takes 30+ seconds. Developers complain that logging is useless for debugging. How do you improve this?**
   - A: Implement structured logging with indexed fields. In Elasticsearch, index `trace_id`, `user_id`, `order_id`, and `error_code` as keyword fields (not full-text). When a developer reports an issue, get the user's trace ID from their support ticket, then search `trace_id:abc123` — results in <1 second. Also: add index lifecycle management (hot 7d → warm 30d → cold → delete) and use data streams for efficient storage.

3. **Q: During a traffic spike, your synchronous logging adds 500ms to every request because the disk is saturated. The application becomes unresponsive. How do you decouple logging from request processing?**
    - A: Switch to async logging with Logback's `AsyncAppender`. The appender enqueues log events in a bounded queue (default 256). The request thread returns immediately. A background thread drains the queue. Configure `neverBlock=true` — if the queue is full, log events are dropped (better to lose logs than block requests). For critical ERROR logs that must not be dropped, use a separate higher-priority queue or direct synchronous write.
    - **Interview follow-up:** With `neverBlock=true`, critical ERROR logs are dropped when the queue fills up — the very logs you need during an outage. If you switch to a separate non-blocking queue for ERROR logs, what prevents that queue from filling up too? How do you guarantee ERROR delivery without blocking the application?

4. **Q: A developer accidentally logs all request bodies including credit card numbers. PII is now in your log aggregation system. How do you handle this breach and prevent recurrence?**
   - A: Immediately: rotate the log index/stream to prevent further access. Use Logback's message converter to mask sensitive fields. Create a custom converter that detects common patterns (credit card regex, `password` field, `token` field) and replaces them with `[REDACTED]`. Implement automated scanning of log entries for PII patterns. Add a code review checklist item: "No credentials, PII, or sensitive data in logs."

5. **Q: Your team uses different log formats across 15 microservices (some JSON, some plain text, different field names). Correlating a request across services is nearly impossible. How do you standardize?**
   - A: Create a shared logging library (or use Spring's `LogstashEncoder`) that all services adopt. Define a standard log schema: `@timestamp`, `level`, `service`, `trace_id`, `span_id`, `message`, `user_id`, `error_code`. Use a centralized logging configuration via Spring Cloud Config. Add a CI check that validates log format compliance. All services must emit the same fields with the same names.

6. **Q: Your application logs sensitive debug information at the INFO level. In production, this information fills the logs with noise and exposes internal details. How do you manage log levels effectively?**
   - A: Use appropriate log levels: ERROR (failures requiring immediate action), WARN (unexpected but handled), INFO (significant business events only — 1-10 per request), DEBUG (details for debugging — enable per package via Actuator). In production, INFO level should be clean enough to read. Use Spring Boot Actuator to dynamically enable DEBUG for specific packages when investigating issues: `POST /actuator/loggers/com.example.paymentservice` with `"configuredLevel": "DEBUG"`. Disable when done.

7. **Q: You need to audit all access to sensitive customer data for compliance (GDPR/SOX). Each read of a user's personal data must be logged with who, what, when, and why. How do you implement this without impacting performance?**
    - A: Use AOP with `@Auditable` annotation on data access methods. Log audit events asynchronously to a separate audit log index (not mixed with application logs). Include: `user_id` (who accessed), `resource_type` and `resource_id` (what), `timestamp` (when), `reason` (why — e.g., `CUSTOMER_SUPPORT`, `ORDER_PROCESSING`). The audit log has its own retention policy (typically 1-7 years). Async logging ensures audit doesn't impact request latency.
    - **Interview follow-up:** The `@Auditable` annotation relies on AOP, which doesn't capture reads performed directly via JPA repository methods or native queries. How do you ensure that every data access path — including batch jobs and admin scripts — goes through the same audit mechanism?

8. **Q: Your logs are shipped to Elasticsearch, but Filebeat can't keep up during peak traffic. Logs are lost because the queue overflows. How do you ensure reliable log delivery?**
   - A: Use a buffering layer between the application and Elasticsearch. Configure Logback to write to local files (rolling files with retention). Filebeat reads from these files — if Elasticsearch is down, Filebeat tracks its position and resumes when ES is back. For higher reliability, use Kafka as the transport layer: Logback → local file → Filebeat → Kafka → Logstash → Elasticsearch. Kafka provides durable storage, replay, and backpressure.

9. **Q: Your onboarding checklist for a new microservice includes "add logging." Developers add `System.out.println()` for debugging. How do you enforce proper logging practices?**
   - A: Add ArchUnit tests that fail if `System.out` or `System.err` is used. Create a checkstyle/PMD rule against `System.out.println`. Provide a logging starter library that all services must use. The starter configures structured JSON logging, MDC filters, async appenders, and the standard log schema. Add a README section: "Always use the logging starter; never use System.out."

10. **Q: Your log aggregation system shows ERROR entries, but they're from third-party library internals with no business context. Developers ignore them because they can't tell if they're important. How do you add context to errors?**
    - A: Always log errors with context using structured arguments. Instead of `log.error("Payment failed")`, use `log.error("Payment failed for order {}", orderId)`. Add `StructuredArguments` from Logstash: `log.error("Payment failed", keyValue("orderId", order.getId()), keyValue("errorCode", exception.getCode()))`. The MDC should already contain `trace_id`, `user_id`, and `service`. Every ERROR log should have enough context to understand the failure without reading surrounding logs.

---

## Interview Questions

1. **What are the standard log levels and when should each be used?**
   - A: TRACE (finest detail, rarely used), DEBUG (development troubleshooting), INFO (significant business events), WARN (unexpected but handled), ERROR (failures needing investigation), FATAL (application cannot continue). In production, typically INFO or WARN level.

2. **What is structured logging?**
   - A: Logging in a machine-readable format (JSON) instead of plain text. Each log entry is a structured object with fields (`@timestamp`, `level`, `service`, `trace_id`, `message`). Enables search, filtering, and analysis by field.

3. **What is MDC and why is it useful?**
   - A: Mapped Diagnostic Context — a map of key-value pairs attached to each log message per thread. Used to include trace ID, user ID, and request ID without passing them as parameters. Automatically includes context in every log line.

4. **How do you implement distributed logging across microservices?**
   - A: Propagate a trace ID via HTTP headers (`X-Trace-Id` or W3C `traceparent`). All services log with this trace ID in MDC. Centralized aggregation (ELK, Loki, Datadog) enables searching across services by trace ID.

5. **What is the difference between synchronous and async logging?**
   - A: Sync logging blocks the request thread until the log is written to disk/network (adds latency). Async logging enqueues the log message and writes in a background thread — request thread returns immediately. Use async in production for performance.

6. **How do you change log levels at runtime in Spring Boot?**
   - A: Use Actuator endpoint: `POST /actuator/loggers/{packageName}` with body `{"configuredLevel": "DEBUG"}`. Enables debugging specific packages in production without redeployment.

7. **What is log aggregation and why is it needed?**
   - A: Collecting logs from multiple sources (servers, services, containers) into a centralized platform for search, correlation, analysis, and alerting. Essential in distributed systems where logs are scattered across many instances.

8. **How do you handle sensitive data in logs?**
   - A: Use Logback message converters to detect and mask patterns (credit cards, passwords). Use structured logging to selectively include/exclude fields. Never log credentials, tokens, or PII in plaintext. Audit log configuration for compliance.

9. **What is the ELK stack?**
   - A: Elasticsearch (distributed search and storage), Logstash (log processing and transformation), Kibana (visualization and dashboards). Alternative: Loki (log aggregation inspired by Prometheus) + Grafana.

10. **How do you prevent logging from becoming a performance bottleneck?**
    - A: Async appenders with `neverBlock=true`, parameterized logging (no string concatenation — `log.info("user {}", id)` not `log.info("user " + id)`), log level guards for expensive computations (`if (log.isDebugEnabled())`), and avoid logging in hot paths (tight loops, critical sections).

---

## Developer Recommendations

- **Always use structured logging (JSON) in production** — Plain text logs are impossible to parse reliably at scale. Structured JSON logs with consistent field names (`@timestamp`, `level`, `service`, `trace_id`, `message`) enable automated analysis, alerting, and debugging. Configure Logback with `LogstashEncoder` for JSON output. In development, use a human-readable console appender for readability.
  - **Production story:** One team spent 8 hours debugging a production incident because their plain-text logs had no consistent delimiter — they couldn't reliably extract timestamps or error codes across 15 services.

- **Use MDC to automatically include trace_id, user_id, and service in every log entry** — Without MDC, every log method call needs to pass context manually, which developers forget. Configure a servlet filter that puts `trace_id` (from request header), `user_id` (from authentication), and `service` (from configuration) into MDC. These fields are automatically included in every log line via the encoder configuration. Zero additional code per log statement.

- **Use async logging with neverBlock=true in production** — Synchronous logging turns every `log.info()` into a blocking I/O operation. Under high load, this adds significant latency and can exhaust thread pools. Async logging enqueues the event (microseconds) and writes in a background thread. `neverBlock=true` prevents the queue from blocking the application — logs are dropped (better than blocking). Monitor the async appender's queue depth and drop rate.

- **Use parameterized logging, never string concatenation** — `log.info("User {} placed order {}", userId, orderId)` is faster than `log.info("User " + userId + " placed order " + orderId)`. With parameterized logging, string construction only happens if the log level is enabled. With concatenation, strings are always built — even for DEBUG statements that are filtered out. This is a significant performance difference at scale.

- **Log errors with full context, not just the message** — `log.error("Payment failed for order {}: {}", orderId, exception.getMessage())` is unhelpful. Include: `StructuredArguments.keyValue("orderId", order.getId())`, the full exception (stack trace), and ensure MDC has `trace_id` and `user_id`. A proper error log should contain enough information to understand the failure without reading other logs.

- **Use log aggregation and alerting, not grep on servers** — Logs on local disks are useless during incidents because developers can't access them quickly. Ship all logs to a centralized platform. Create dashboards for error rates by service. Set up alerts for ERROR rate spikes, specific error patterns, and audit violations. The goal: detect and diagnose issues from your monitoring dashboard, not by SSH'ing into servers.
