# Distributed Logging

## Overview

- **Definition** — Centralized logging strategy for microservices where logs from all services are aggregated into a single searchable platform

- **Why It Exists** — In microservices, each service produces its own logs on its own host. Without aggregation, debugging a request that spans 5 services requires SSH-ing into 5 different machines and correlating timestamps manually — impossible at scale.

- **Historical Context** — Monolithic applications wrote logs to a single file on one server. As systems moved to SOA and later microservices, the number of log sources grew exponentially, giving rise to centralized logging platforms like ELK and cloud-native solutions like Loki.

- **Key Concepts** — **Correlation IDs** link related log entries across services; **Structured logging** uses JSON for machine-parseable output; **Log aggregation** collects all logs in one searchable store; **Log levels** control verbosity in production; **Sampling** reduces volume for high-throughput services

## Core Concepts

- **Structured Logging**
  - Logs are emitted in a structured format like JSON with consistent keys (timestamp, level, service, message, correlation_id). This enables automated parsing, filtering, and querying across services.

  ```json
  {"timestamp":"2026-06-13T10:00:00Z","level":"ERROR","service":"payment-service","correlation_id":"abc-123","message":"Payment declined","amount":49.99}
  ```

  - Without structured logging, teams waste time writing custom parsers for each service's ad-hoc format.

- **Correlation IDs**
  - A unique identifier generated at the edge (API gateway or ingress) and propagated through every subsequent service call via headers (e.g., X-Correlation-ID). All logs from that request across services share the same ID, enabling full request tracing.
  - Services must forward the correlation ID to downstream services and include it in all log statements.
  - Correlation IDs also appear in error responses so clients can report them in support tickets.

- **Log Aggregation Platforms**
  - ELK (Elasticsearch, Logstash, Kibana) is the most common open-source stack. Loki + Grafana is a lighter alternative optimized for logs. Cloud offerings include AWS CloudWatch Logs, Azure Log Analytics, and GCP Cloud Logging.
  - Elasticsearch indexes log content for full-text search; Kibana provides dashboards and visualizations.
  - Loki stores logs compressed and indexes only metadata (labels), making it cheaper for high-volume scenarios.

- **Log Levels**
  - Standard levels are ERROR, WARN, INFO, DEBUG, TRACE.
  - In production, only ERROR, WARN, and INFO should be enabled by default. DEBUG is enabled temporarily for specific services during incident investigation.
  - Too many INFO logs in a hot path can generate terabytes per day — use sparingly based on business criticality.

- **Log Sampling**
  - For services handling millions of requests per minute, logging every request is infeasible. Sampling strategies include:
    - **Rate-limited sampling** — log at most N events per second
    - **Probabilistic sampling** — log a random percentage of requests
    - **Head-based vs. tail-based sampling** — head preserves entire traces for a sample; tail preserves traces that contain errors.

- **Centralized Log Shipping**
  - Agents like Filebeat, Fluentd, or Promtail run as sidecars or daemonsets on each node, tailing log files and shipping them to the aggregation platform.
  - These agents handle backpressure, retries, and buffering so the application never blocks on log I/O.

- **Indexing Strategies**
  - Time-based indices (e.g., logs-2026.06.13) allow efficient retention policies. Older indices are closed or deleted after a retention period (e.g., 30 days for INFO, 7 days for DEBUG, 1 year for ERROR).
  - Hot-warm-cold architectures tier index storage by age and query frequency.

- **Multi-Tenancy in Logs**
  - Separate environments (dev, staging, prod) should be logically or physically separated in the log store. Use separate indices or label-based namespaces to prevent cross-environment contamination.

- **GDPR/PII Redaction**
  - Logs must never contain passwords, credit card numbers, or personal data. Redaction happens at the application layer (before emitting the log) or at the shipping layer (via Logstash filters or Fluentd plugins).
  - Common patterns: masking fields like "password": "***", truncating email addresses, stripping query parameters that contain tokens.

## Common Mistakes

- **Logging Sensitive Data**
  - Developers log request/response bodies verbatim, including passwords, credit card numbers, and session tokens. This creates a compliance and security risk — anyone with Kibana access can read user secrets.
  - **Why it looks correct:** Logging the full payload helps debug API integration issues. The developer is trying to be thorough.
  - The fix: Use a structured logger with a blacklist. Before emitting the log, the logger scrubs known sensitive keys. Also implement log sanitization at the shipping layer as a defense-in-depth measure.

- **Debug Logging in Production**
  - Leaving DEBUG-level logging enabled on high-throughput services creates a firehose that overwhelms the aggregation platform, increases storage costs by orders of magnitude, and buries real warnings and errors in noise.
  - **Why it looks correct:** "More data is always better" — developers want full visibility into production behaviour.
  - The fix: Configure the logging framework to default to INFO in production. Use dynamic log-level changes (e.g., via an admin endpoint or Kubernetes ConfigMap) to enable DEBUG on a single pod or service during an incident, then revert.

- **Missing Correlation IDs**
  - Services log events without any common identifier. When an error occurs, operators have no way to connect log entries across services for the same request. Each log is an isolated island.
  - **Why it looks correct:** The service works fine in isolation. Unit tests and integration tests verify single-service behaviour.
  - The fix: Mandate correlation ID propagation at the architecture level. The API gateway generates the ID. Every HTTP client library and message queue adapter must forward it automatically. Add middleware that extracts the ID from inbound requests and injects it into the downstream context.

- **Synchronous Log Shipping**
  - The application writes logs via a blocking network call to the aggregation platform (e.g., directly calling Elasticsearch HTTP API on the critical path). If the aggregator is slow or down, the application thread blocks or fails.
  - **Why it looks correct:** The developer wants logs to appear in Kibana immediately, so they avoid async buffering.
  - The fix: Always write logs to stdout/stderr or a local file, and let a sidecar agent (Filebeat, Fluentd) handle shipping asynchronously. The application must never depend on the availability of the log aggregator.

- **Missing Log Rotation**
  - Logs are written to a single file that grows without bound. Eventually the disk fills up, causing the service to crash or the node to become unhealthy.
  - **Why it looks correct:** In development, machines are cleaned frequently. In containerized environments, developers assume logs are ephemeral.
  - The fix: Configure log rotation (size-based or time-based) in the logging framework and the container runtime. In Kubernetes, enable stdout logging with Docker's log rotation or configure the sidecar agent to manage file rotation.

- **Inconsistent Log Format**
  - Each service team chooses a different log format (some use JSON, some use plain text, some use key=value pairs). The aggregation platform cannot parse them uniformly, so searches miss events and dashboards break.
  - **Why it looks correct:** Each team optimizes for their own convenience without considering the cross-team debugging use case.
  - The fix: Define a company-wide logging standard as a shared library (e.g., a base JSON schema with required fields). Enforce it in CI/CD with linters that reject non-conforming log output.

## Real-World Scenarios

### Incident Response for Payment Failure

- A customer reports a failed payment. The request flows: API Gateway → Auth Service → Payment Service → Ledger Service → Notification Service.
- Without distributed logging, the on-call engineer must check each service's logs independently, guessing timestamps and hoping to find the error.
- With correlation IDs, the engineer searches Kibana for the trace ID from the customer's support ticket. All five services' log entries appear in a single view, sorted by timestamp.
- The logs show the Payment Service received a timeout from the Ledger Service. The Ledger Service logs show a database connection pool exhaustion at that moment.
- **Outcome:** Root cause identified in 5 minutes instead of 2 hours. The correlation ID turned a multi-service firefight into a single Kibana search.

### High-Throughput Service at 100GB/Day

- A recommendation service handles 50,000 requests per second. Each request generates 3 log lines (incoming, processing, outgoing) at INFO level. This produces 100GB of logs per day.
- The storage cost alone exceeds the budget. Kibana queries become slow because the index has billions of documents.
- **Solution applied:**
  - Reduce INFO logging: only log incoming requests with key parameters, remove verbose processing logs.
  - Enable probabilistic sampling: log only 10% of requests at INFO, but always log ERROR and WARN regardless of sampling.
  - Add log-level endpoint: operations can enable DEBUG on a single pod for targeted investigation.
  - Switch from ELK to Loki + Grafana for the hot path, reducing storage costs by 80% while keeping the same query capability for structured labels.
- **Outcome:** Daily log volume drops to 8GB. All ERROR and WARN logs are preserved. Debuggability is maintained via dynamic log levels and sampling.

### Distributed Tracing vs. Logging

- Logging captures event details — what happened, when, and with what data. Distributed tracing captures request flow — how long each service hop took and the dependency graph.
- They complement each other: traces point you to the service and span where an error occurred, and logs provide the detailed context for that specific event.
- Correlation IDs bridge the two: the trace ID is used as the correlation ID in logs, allowing seamless navigation between tracing UI (Jaeger, Zipkin) and logging UI (Kibana, Grafana).

## Scenario-Based Questions

- **Q: A critical production incident occurs overnight. The logs are so noisy with DEBUG messages that the relevant ERROR entries are buried. What went wrong and how do you prevent it?**
  - Answer: DEBUG logging was left enabled in production. The fix is to ensure the production logging configuration defaults to INFO and to implement a dynamic log-level mechanism that allows operators to enable DEBUG on-demand for specific services or pods. Additionally, log sampling can be configured to cap the rate of DEBUG messages.
  - **Interview follow-up:** How would you design a dynamic log-level system that works across 100 microservices without requiring a deployment per change?

- **Q: Your team is migrating from monolith to microservices. The current logging writes to a single shared file on a network drive. What is the first thing you change?**
  - Answer: Move from shared-file logging to structured JSON logging to stdout. Each container or process writes structured logs to stdout. A log shipper (Filebeat, Fluentd) collects them and forwards to a central aggregation platform (ELK or Loki). This decouples services from each other and from shared infrastructure.
  - **Interview follow-up:** How would you handle the transition period where some services are still in the monolith and some are already microservices?

- **Q: A junior developer proposes logging the full HTTP request body including passwords "for debugging". How do you respond?**
  - Answer: Logging passwords is a security and compliance violation (GDPR, PCI-DSS). Instead, define a structured log schema with a blacklist of sensitive fields. Use a logging middleware that automatically redacts known sensitive keys before emitting the log entry. Also implement a secondary redaction layer in the log shipper for defense in depth.
  - **Interview follow-up:** How would you handle the case where a sensitive field is nested three levels deep in a JSON payload?

## Interview Questions

- **Q: Explain the ELK stack architecture for distributed logging. What role does each component play?**
  - **A:** Elasticsearch is the search and storage engine that indexes log documents for full-text queries. Logstash is the ingestion pipeline that parses, transforms, and enriches logs before indexing. Kibana provides the visualization and dashboard layer for querying logs. Filebeat (or Beats family) ships logs from nodes to Logstash or directly to Elasticsearch. In modern deployments, Logstash may be replaced by Elasticsearch ingest pipelines for simplicity.

- **Q: How would you ensure a correlation ID propagates through a chain of microservices that communicate via both HTTP and message queues (Kafka/RabbitMQ)?**
  - **A:** For HTTP, use middleware that extracts the X-Correlation-ID header from inbound requests and injects it into outbound requests. For message queues, include the correlation ID in the message headers (e.g., Kafka record headers or RabbitMQ message properties). The consuming service reads the header and sets it in its logging context. This requires every service to use a shared client library that handles propagation automatically, avoiding reliance on individual developers remembering to pass the ID.

- **Q: Design a log schema for a payment microservice. What fields are mandatory and why?**
  - **A:** Mandatory fields: timestamp (ISO 8601), level (ERROR/WARN/INFO), service_name (payment-service), correlation_id, message, environment (prod/staging/dev). Optional but recommended: duration_ms, amount, payment_method (redacted), user_id (hashed, not raw). The schema must be versioned to handle field additions without breaking downstream dashboards. Sensitive fields such as credit_card_number must never appear in the schema — only a boolean `is_redacted` flag or a masked token.

- **Q: Compare and contrast Elasticsearch + Kibana vs. Loki + Grafana for log aggregation. When would you choose one over the other?**
  - **A:** ELK indexes the full log content, enabling arbitrary full-text search, regex queries, and complex aggregations. It is powerful but expensive in storage and compute. Loki indexes only metadata labels (service, level, environment) and stores the log content compressed. It is cheaper and faster for label-based queries but limited for full-text search. Choose ELK when you need deep search capabilities and have budget for the infrastructure. Choose Loki for high-volume, cost-sensitive environments where most queries filter by service and level.

## Developer Recommendations

- **Adopt a structured logging library early**
  - Reasoning: Structured logging is the foundation of every other distributed logging practice. Without it, aggregation, searching, and alerting are all significantly harder.
  - Implementation: Choose a logging library that natively outputs JSON (e.g., Serilog for .NET, Winston for Node.js, structlog for Python, slog for Go). Define a company-wide JSON schema in a shared package with required fields: timestamp, level, service, correlation_id, message. All services must use this schema. Block non-compliant services at the CI/CD level with automated log format checks.

- **Implement mandatory correlation ID propagation in shared infrastructure**
  - Reasoning: Correlation IDs are the single most impactful debugging tool for microservices. Their absence makes every incident response dramatically slower.
  - Implementation: Add correlation ID middleware to the API gateway and every service's HTTP client. For message queues, add a serializer/deserializer that copies the correlation ID to message headers. Use OpenTelemetry's trace propagation as the underlying mechanism — it provides correlation IDs as a byproduct of distributed tracing and is language-agnostic.

- **Configure log-level management with dynamic overrides**
  - Reasoning: Production should run at INFO or WARN by default. But when an incident occurs, operators need the ability to increase verbosity for specific services without redeploying or restarting.
  - Implementation: Expose a `/loglevel` admin endpoint (secured behind mTLS or a service mesh) on each service that allows changing the log level at runtime. For Kubernetes, use a ConfigMap that the sidecar log agent watches and propagates to the application. In cloud environments, use a feature flag or a dedicated logging configuration service.

- **Use a sidecar log shipper with backpressure handling**
  - Reasoning: The application must never block on log shipping. Synchronous shipping couples application availability to the log aggregator's availability.
  - Implementation: Run Filebeat (or Fluentd, Promtail) as a sidecar container in each pod. The application writes structured logs to stdout. The sidecar tails stdout, buffers logs to disk if the aggregator is unavailable, and ships with exponential backoff. Monitor the shipper's buffer size and queue depth to detect aggregator issues early.

- **Enforce PII redaction at two layers**
  - Reasoning: A single layer of redaction can fail due to a bug or misconfiguration. Defense in depth ensures that even if the application layer misses something, the shipping layer catches it.
  - Implementation: Layer 1 — Application-level: Configure the logging library with a list of blacklisted field patterns (password, ssn, credit_card, token). The logger scrubs these fields before serializing. Layer 2 — Shipping-level: In Logstash or Fluentd, add a filter that scans for regex patterns matching PII (email, SSN, credit card numbers) and redacts or rejects matching log entries. Audit both layers weekly with sample log exports.
