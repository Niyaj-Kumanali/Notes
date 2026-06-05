# Distributed Tracing

## 1. Executive Summary

Distributed tracing is an observability technique that tracks requests as they flow through multiple services in a distributed system. Each request is assigned a unique trace ID that is propagated across service boundaries. Individual units of work (spans) are collected from each service and assembled into a complete trace, providing end-to-end visibility into request flow, latency breakdown, and error propagation. Distributed tracing is essential for debugging performance issues and understanding system behavior in microservices architectures.

## 2. Core Theory

### Key Concepts

**Trace:** The complete path of a single request as it travels through the distributed system. A trace is a tree of spans.

**Span:** A named, timed operation representing a unit of work. Contains:
- Span ID (unique within trace)
- Parent span ID (for tree structure)
- Trace ID (shared across all spans)
- Operation name
- Start and end timestamps
- Tags/attributes (key-value metadata)
- Status (OK/ERROR)
- Events (log statements with timestamps)

**Context Propagation:** The mechanism by which trace context (trace ID, span ID) is passed from one service to another, typically via HTTP headers or message metadata.

### Trace Structure

```
Trace: "Order Request"
  |
  +-- Span: "API Gateway" (duration: 50ms)
       |
       +-- Span: "Authenticate" (duration: 10ms, parent: API Gateway)
       |
       +-- Span: "Create Order" (duration: 35ms, parent: API Gateway)
            |
            +-- Span: "Reserve Inventory" (duration: 15ms, parent: Create Order)
            |
            +-- Span: "Process Payment" (duration: 20ms, parent: Create Order)
```

## 3. Under-the-Hood Deep Dive

### OpenTelemetry Architecture

OpenTelemetry is the industry standard for distributed tracing, combining OpenTracing and OpenCensus.

```
[Application] -> [OpenTelemetry SDK] -> [Exporter] -> [Backend (Jaeger/Zipkin)]
     |                    |                  |               |
 Instrumentation     Context       BatchSpanProcessor     Storage + UI
  via API or      Propagation      (async, batched)
  auto-inject
```

### Context Propagation Headers

**W3C TraceContext (standard):**
- `traceparent`: Format: `00-traceId-spanId-traceFlags`
- `tracestate`: Vendor-specific trace data

**Example:**
```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-00f067aa0ba902b7-01
             |  |______________________________| |________________| |
             |                   |                       |         |
           Version           Trace Id               Span Id     Flags
```

### Sampling Strategies

1. **Head-based**: Decision at the start of the trace. Simple, but may miss interesting events.
2. **Tail-based**: Keep all traces, decide later. More resource intensive.
3. **Probabilistic**: Random sampling (e.g., 1% of traces).
4. **Rate-limiting**: Fixed number of traces per second.
5. **Adaptive**: Adjust sampling rate based on system state (increase during errors).

## 4. Production Code Examples

### OpenTelemetry with Spring Boot

```java
@Configuration
public class TracingConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        // Configure OTLP exporter to send to Jaeger/collector
        OtlpGrpcSpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://jaeger:4317")
            .setCompression("gzip")
            .setTimeout(Duration.ofSeconds(30))
            .build();

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .setResource(Resource.getDefault()
                .toBuilder()
                .put(ResourceAttributes.SERVICE_NAME, "order-service")
                .put(ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "production")
                .build())
            .addSpanProcessor(BatchSpanProcessor.builder(spanExporter)
                .setMaxQueueSize(2048)
                .setMaxExportBatchSize(512)
                .setScheduleDelay(Duration.ofMillis(5000))
                .build())
            .setSampler(Sampler.traceIdRatioBased(0.1)) // 10% sampling
            .build();

        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(ContextPropagators.create(
                W3CTraceContextPropagator.getInstance()))
            .build();
    }

    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("com.example.order");
    }
}
```

### Manual Instrumentation with OpenTelemetry

```java
@Service
public class OrderService {

    private final Tracer tracer;

    public OrderService(Tracer tracer) {
        this.tracer = tracer;
    }

    public Order createOrder(CreateOrderRequest request) {
        // Create a span for the order creation operation
        Span span = tracer.spanBuilder("createOrder")
            .setSpanKind(SpanKind.SERVER)
            .setAttribute("user.id", request.getUserId())
            .setAttribute("order.items", request.getItems().size())
            .startSpan();

        // Put span in context
        try (Scope scope = span.makeCurrent()) {
            // Business logic
            Order order = new Order(request);

            // Call inventory service with propagated context
            reserveInventory(order, request.getItems());

            // Call payment service with propagated context
            processPayment(order);

            span.setStatus(StatusCode.OK);
            span.setAttribute("order.id", order.getId());
            span.setAttribute("order.total", order.getTotalAmount().doubleValue());

            return order;
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end(); // Always end the span
        }
    }

    private void reserveInventory(Order order, List<OrderItem> items) {
        Span span = tracer.spanBuilder("reserveInventory")
            .setSpanKind(SpanKind.CLIENT)
            .setAttribute("items.count", items.size())
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // HTTP call to inventory service
            // OpenTelemetry propagates context via HTTP headers automatically
            inventoryClient.reserve(order.getId(), items);
            span.setStatus(StatusCode.OK);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### Auto-Instrumentation with OpenTelemetry Agent

```bash
# No code changes needed! Add Java agent to JVM startup
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=order-service \
     -Dotel.traces.exporter=otlp \
     -Dotel.exporter.otlp.endpoint=http://jaeger:4317 \
     -Dotel.traces.sampler=parentbased_traceidratio \
     -Dotel.traces.sampler.arg=0.1 \
     -jar order-service.jar
```

### Custom Span Attributes and Events

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(@PathVariable String id) {
        // Get current span from context
        Span currentSpan = Span.current();

        // Add attributes to current span
        currentSpan.setAttribute("order.id", id);

        // Add events (timed log entries within span)
        currentSpan.addEvent("Fetching order from database",
            Attributes.of(
                AttributeKey.stringKey("db.query"), "SELECT * FROM orders WHERE id = ?",
                AttributeKey.stringKey("db.system"), "postgresql"
            ));

        Order order = orderService.findById(id);

        if (order == null) {
            currentSpan.setStatus(StatusCode.ERROR, "Order not found");
            return ResponseEntity.notFound().build();
        }

        currentSpan.setAttribute("order.status", order.getStatus().name());
        return ResponseEntity.ok(order);
    }
}
```

### Spring Cloud Sleuth / Micrometer Tracing

```java
// With Micrometer Tracing (Spring Boot 3.x)
@Service
public class PaymentService {

    private final Tracer tracer;
    private final RestTemplate restTemplate;

    public PaymentService(Tracer tracer, RestTemplate restTemplate) {
        this.tracer = tracer;
        this.restTemplate = restTemplate;
    }

    @Observed(name = "payment.process",
        contextualName = "process-payment",
        lowCardinalityKeyValues = {"paymentType", "credit_card"})
    public PaymentResponse processPayment(PaymentRequest request) {
        // Observation creates spans automatically
        return restTemplate.postForObject(
            "http://payment-service/api/payments",
            request,
            PaymentResponse.class
        );
    }
}

// application.yml for tracing
spring:
  application:
    name: order-service
  sleuth:
    sampler:
      probability: 0.1  # 10% sampling (Spring Boot 2.x)
  tracing:
    sampling:
      probability: 0.1  # Spring Boot 3.x with Micrometer Tracing
```

### Trace ID in Logs

```java
// OpenTelemetry auto-instruments logging framework
// trace_id and span_id are automatically added to MDC

// logback-spring.xml - include trace/span ID in logs
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <includeMdcKeyName>trace_id</includeMdcKeyName>
    <includeMdcKeyName>span_id</includeMdcKeyName>
    <includeMdcKeyName>trace_flags</includeMdcKeyName>
</encoder>

// Resulting log entry
{
  "@timestamp": "2024-01-01T12:00:00Z",
  "level": "INFO",
  "message": "Order created",
  "trace_id": "0af7651916cd43dd8448eb211c80319c",
  "span_id": "00f067aa0ba902b7",
  "service": "order-service"
}
```

## 5. Real-World Scenarios

### Diagnosing Latency Issues

A user reports slow checkout. Tracing shows:
```
Total: 3500ms
  - API Gateway: 100ms
  - Order Service: 3200ms
      - Inventory Check: 50ms
      - Payment Processing: 3000ms (slowest span!)
      - Confirmation: 50ms
  - Notification: 100ms (async, not in critical path)
```

Root cause: Payment gateway call is slow. Add circuit breaker and timeout.

### Error Detection with Traces

Alert: error rate > 5%.
- Open Jaeger, search for traces with errors in the last 15 minutes.
- Filter by service: payment-service.
- See error spans with stack traces.
- Identify: "Connection refused" to payment gateway.
- Fix: Restart payment gateway connection pool.

## 6. Performance

### Tracing Overhead

| Component | Overhead |
|-----------|----------|
| Auto-instrumentation agent | ~5-10% CPU (max) |
| Span creation | ~1 microsecond |
| Attribute addition | < 1 microsecond per attribute |
| Batch export | Background thread, batching |
| Export to backend | Per batch, background |

### Optimization
- Reduce sampling rate (1-10% is typical).
- Batch export (not per-span export).
- Limit span attributes (avoid high cardinality).
- Use async exporter (non-blocking).

## 7. Security

### Trace Security Considerations

- Traces may contain sensitive data (user IDs, request payloads).
- Configure span attribute filtering to exclude sensitive data.
- Use encrypted connections between agent and collector/backend.
- Access control on tracing backend.
- Retention policy for trace data.

```java
@Bean
public SpanProcessor spanProcessor() {
    return new BatchSpanProcessor(spanExporter) {
        @Override
        public void onStart(Context context, ReadWriteSpan span) {
            // Filter out sensitive attributes
            // Implementation depends on OpenTelemetry version
        }
    };
}
```

## 8. Common Mistakes

### Mistake 1: No Sampling
Tracing every request creates massive data volumes and costs.

### Mistake 2: Too Many Spans
Every method call as a span creates noise. Focus on service boundaries and significant operations.

### Mistake 3: No Context Propagation
Tracing breaks when context isn't propagated to async operations or message queues.

### Mistake 4: Missing Error Attributes
Errors without recording exception details in spans are hard to diagnose.

### Mistake 5: Not Using Auto-Instrumentation
Manual instrumentation is error-prone. Auto-instrumentation covers most common libraries.

## 9. Senior Engineer Perspective

### Trace-Driven Development

- Use traces to validate performance during development.
- Add custom spans for business transactions (checkout, signup).
- Set trace-based SLOs: "95% of checkout traces complete within 2s."
- Trace analysis in CI/CD: compare trace distributions before/after deployment.

### Correlation with Other Signals

```
Trace ID: abc123
  |
  +-- Logs filtered by traceId: "abc123"
  +-- Metrics filtered by traceId: latency, error rate
  +-- All three correlated in a single dashboard
```

### Distributed Tracing Maturity

| Level | Capability |
|-------|------------|
| 1. Basic | Single-service spans |
| 2. Connected | Context propagation across services |
| 3. Instrumented | All services traced, custom spans |
| 4. Analyzed | Trace-based alerting, SLO tracking |
| 5. Automated | Trace-driven auto-scaling, AI-based root cause |

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What is distributed tracing?
   **A:** A technique to track requests as they flow through multiple services in a distributed system.

2. **Q:** What is a trace?
   **A:** The complete path of a single request through the system, composed of multiple spans.

3. **Q:** What is a span?
   **A:** A named, timed unit of work within a trace, representing a single operation.

4. **Q:** What is context propagation?
   **A:** Passing trace context (trace ID, span ID) across service boundaries via headers or message metadata.

5. **Q:** What is OpenTelemetry?
   **A:** The industry standard for distributed tracing, providing API, SDK, and auto-instrumentation.

6. **Q:** What is Jaeger?
   **A:** An open-source distributed tracing backend for visualizing and analyzing traces.

7. **Q:** What is the W3C traceparent header?
   **A:** A standardized HTTP header for trace context propagation: `00-traceId-spanId-flags`.

8. **Q:** What is sampling and why is it needed?
   **A:** Sampling decides which traces to capture (e.g., 1%). Needed because capturing every trace is expensive.

9. **Q:** What is the difference between a trace ID and span ID?
   **A:** Trace ID identifies the entire request. Span ID identifies a single operation within the trace.

10. **Q:** What libraries does OpenTelemetry auto-instrument?
    **A:** HTTP clients, JDBC, Kafka, gRPC, Servlet, Redis, and many more.

### Medium

11. **Q:** How do you propagate trace context across async boundaries?
    **A:** Store current context before async operation, restore context in the async thread. OpenTelemetry's ContextStorage handles this via `Context#wrap` or agent auto-wrapping.

12. **Q:** What is the difference between head-based and tail-based sampling?
    **A:** Head-based: decide at trace start. Tail-based: keep all traces, decide later which to store. Tail-based can selectively keep error traces.

13. **Q:** How do you trace messaging systems (Kafka)?
    **A:** OpenTelemetry auto-instruments Kafka producer/consumer. Trace context is propagated in Kafka record headers.

14. **Q:** What is the OpenTelemetry Collector?
    **A:** A vendor-agnostic agent that receives traces, processes them (filtering, sampling, enrichment), and exports to backends.

15. **Q:** How do you correlate traces with logs?
    **A:** Include trace ID in log MDC. Configure logging framework to emit trace_id and span_id. Query logs by trace ID.

16. **Q:** What is a span attribute?
    **A:** Key-value metadata added to a span (e.g., HTTP method, status code, user ID, database query).

17. **Q:** What is SpanKind?
    **A:** Classification of span: CLIENT (outgoing call), SERVER (incoming request), INTERNAL (internal operation), PRODUCER (message send), CONSUMER (message receive).

18. **Q:** How does tracing help with performance debugging?
    **A:** Shows which service/operation is slowest in the request chain. Provides timing breakdown per span.

19. **Q:** What is the OTLP protocol?
    **A:** OpenTelemetry Protocol, the standard for exporting telemetry data to backends (gRPC or HTTP).

20. **Q:** How do you handle trace context in gRPC?
    **A:** OpenTelemetry auto-instruments gRPC. Context propagated via gRPC metadata (binary traceparent).

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Design a tail-based sampling system for traces.
    **A:** Collect all spans in a buffer. Define rules: keep all ERROR traces, sample 10% of slow traces (P99+), sample 1% of normal traces. Make decisions after trace completes. Implement with OpenTelemetry Collector or streaming processor.

2. **Q:** How do you implement trace context propagation across message queues (Kafka)?
    **A:** OpenTelemetry auto-instruments Kafka: inject trace context into Kafka record headers on produce, extract on consume. For manual: extract context from current span, put in headers, send.

3. **Q:** Design a system to detect trace anomalies in real-time.
    **A:** Stream traces through Kafka. ML model detects: unusual span duration, missing spans, new service in trace path, cycle detection. Alert on anomalies. Use Flink/KSQL for stream processing.

4. **Q:** How do you handle trace context in event-driven systems with eventual consistency?
    **A:** Trace ID is generated at the entry point (API). Propagated to all events. Each event handler adds its own spans under the same trace. Different event handlers may run at different times; backend assembles the full trace.

5. **Q:** Design a distributed tracing system that handles 100K spans/second.
    **A:** OpenTelemetry Collector with multiple pipelines. Load balance spans across collector instances. Use Kafka as buffer between collector and backend. Backend: Jaeger with Cassandra/Elasticsearch. Downsample older traces.

6. **Q:** How do you trace database queries?
    **A:** OpenTelemetry auto-instruments JDBC: each query is a child span under the caller. Captures: query text (sanitized), bind parameters, duration, rows returned.

7. **Q:** Design a system for cross-service dependency discovery using traces.
    **A:** Process trace data to build service dependency graph. Nodes = services. Edges = calls between services. Weight by call count. Track changes: new dependencies (unexpected), deprecated dependencies.

8. **Q:** How do you implement trace-based SLO monitoring?
    **A:** Define SLO: "95% of checkout traces complete within 2s." Track traces by type. Compute SLO compliance over rolling window. Alert when error budget depleted.

9. **Q:** Design a trace storage strategy that balances query performance and cost.
    **A:** Hot storage (Cassandra/Elasticsearch): 7 days, full detail. Warm: 30 days, downsampled (remove high-cardinality attributes). Cold: S3/Parquet, query via Presto/Athena. Ingest via OpenTelemetry Collector.

10. **Q:** How do you debug missing spans in a trace?
    **A:** Check context propagation (missing headers). Check sampling decisions (was trace sampled?). Check span exporter errors. Check if async boundaries preserve context.

### System Design

11. **Q:** Design a distributed tracing system for 500+ microservices.
    **A:** Auto-instrumentation for all services. OpenTelemetry Collector per cluster. Kafka for buffering. Jaeger backend with Cassandra storage. Service mesh (Istio) provides trace context propagation by default. Centralized sampling configuration.

12. **Q:** Design a tracing system that correlates frontend (browser) and backend traces.
    **A:** Frontend: OpenTelemetry JS SDK generates trace. Propagate trace ID via response header. Backend: extract and continue the same trace. Combined view in Jaeger.

13. **Q:** Design a tracing system for a serverless architecture (AWS Lambda).
    **A:** AWS X-Ray as backend. OpenTelemetry Lambda layers for auto-instrumentation. X-Ray SDK for custom spans. Propagate trace context via HTTP headers / SQS message attributes.

14. **Q:** Design a multi-cluster tracing system for a global platform.
    **A:** Each region has its own trace backend (Jaeger). Global trace ID enables cross-region correlation. Propagate trace context across regions via gRPC/HTTP headers.

15. **Q:** Design a cost-effective tracing solution for a startup.
    **A:** Jaeger all-in-one (single binary). 1% sampling rate. Auto-instrumentation only (no manual spans). Retention: 7 days. Upgrade to Jaeger production stack when scale demands.

16. **Q:** Design a trace-based debugging tool for production.
    **A:** Search traces by: trace ID, service, operation, tags, status (error/latency). View waterfall diagram. Compare traces side by side. Slow-motion replay of trace events.

17. **Q:** Design a system for continuous trace comparison (canary analysis).
    **A:** Compare traces from canary vs stable deployment. Metrics: P50/P95/P99 latency difference, error rate difference, new span operations. Auto-rollback if significant degradation detected.

18. **Q:** Design a tracing system for a financial platform requiring full audit traceability.
    **A:** 100% sampling (no probabilistic). Long trace retention (1 year). Immutable trace storage. Trace integrity verification (Merkle chain). Access control on tracing backend.

19. **Q:** Design a tracing system that respects data privacy (GDPR).
    **A:** Attribute filtering: remove PII before export. Retention: configurable per data class (business traces: 30d, technical: 7d). Deletion API: delete traces by user ID. Encryption: TLS + storage encryption.

20. **Q:** Design a unified observability query across traces, logs, and metrics.
    **A:** Common labels: trace_id, service, environment. Query: "find traces with error, then get logs for those trace_ids, overlay on metric dashboard." Implemented via Loki (logs) + Tempo (traces) + Mimir (metrics) with Grafana.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a distributed tracing system that provides root cause analysis automatically.
    **A:** ML model trained on historical traces: learns normal patterns. For each new trace, flag deviations: unexpected latency in specific span, new error pattern, missing completion. Rank by impact. Suggest root cause: "Service X had 500ms latency increase due to database query Y."

2. **Q:** How do you design a trace context propagation system that works across protocols (HTTP, gRPC, Kafka, WebSocket, RabbitMQ)?
    **A:** W3C TraceContext for HTTP/gRPC. Inject traceparent into message headers for messaging systems. OpenTelemetry SDK auto-instruments all protocols. Custom instrumentation for non-standard protocols.

3. **Q:** Design a zero-overhead tracing system for latency-sensitive applications.
    **A:** eBPF-based tracing: kernel-level instrumentation with zero application code changes. Trace syscalls, network calls, not application code. Higher-level traces sampled at very low rate (0.1%).

4. **Q:** How do you implement distributed tracing for a multi-tenant SaaS platform?
    **A:** Each trace tagged with tenant ID. Tenant-isolated trace storage. Rate limit traces per tenant. Tenant-specific sampling rules. Dashboard: tenant-level trace analysis.

5. **Q:** Design a system that uses traces for capacity planning.
    **A:** Analyze trace patterns: which services are called for each operation, what resources they use. Model: trace volume + per-trace resource usage = total load. Forecast capacity based on trace growth.

6. **Q:** How do you design trace sampling for a platform where every error must be captured?
    **A:** Tail-based sampling: keep all traces, decide at end. Rules: keep 100% of ERROR traces, keep 100% of traces from priority users, keep 10% of slow traces (>P50), drop remaining if over budget.

7. **Q:** Design a system that detects unknown/unexpected service dependencies using traces.
    **A:** Build dependency graph from trace data. Compare to declared dependencies (service mesh config, API registry). Alert on undeclared dependencies. Track new dependencies over time.

8. **Q:** How do you implement distributed tracing in a high-throughput (1M req/s) system?
    **A:** Low overhead: 0.01% sampling rate. Auto-instrumentation only. Minimal attributes. eBPF-level tracing for common cases. Prioritize error traces.

9. **Q:** Design a system that replays production traces in a staging environment.
    **A:** Capture trace attributes: request URL, headers, parameters, response. Anonymize sensitive data. Replay: send traced requests to staging. Compare: staging trace vs production trace (latency, behavior differences).

10. **Q:** How do you design a federated tracing system across organizational boundaries?
    **A:** Shared trace ID: generated by entry point, propagated across org boundaries. Each org runs its own trace backend. Global trace endpoint: queries each org's backend and assembles. Auth: mTLS between backends.

## 13. Debugging & Troubleshooting

### Common Tracing Issues

**Issue: Missing spans in trace**
- Context propagation failure (missing header).
- Async boundary not preserving context.
- Sampling: different decisions at different services.

**Issue: Trace not visible in Jaeger**
- Check span exporter: any errors?
- Check sampling: was trace sampled?
- Check Jaeger connection: port, network.
- Check indexing delay: may take a few seconds.

**Issue: High tracing overhead**
- Reduce sampling rate.
- Reduce span attributes.
- Check for synchronous exporter (should be async).

**Issue: Context not propagated to async threads**
```java
// WRONG - context lost in async
CompletableFuture.supplyAsync(() -> {
    Span span = tracer.spanBuilder("async-work").startSpan(); // No parent!
});

// RIGHT - propagate context
Context context = Context.current();
CompletableFuture.supplyAsync(() -> {
    try (Scope scope = context.makeCurrent()) {
        Span span = tracer.spanBuilder("async-work").startSpan();
        // span has correct parent
    }
});
```

## 14. Comparison Section

### Distributed Tracing vs Logging vs Metrics

| Aspect | Tracing | Logging | Metrics |
|--------|---------|---------|---------|
| Granularity | Per-request path | Per-event | Aggregated |
| Data | Span tree | Text/JSON events | Numeric time-series |
| Volume | Medium | High | Low |
| Query | By trace/service | By text/filters | By metric/labels |
| Use case | Performance, flow | Debugging, audit | Alerts, dashboards |

### OpenTelemetry vs Jaeger vs Zipkin

| Feature | OpenTelemetry | Jaeger | Zipkin |
|---------|--------------|--------|--------|
| Role | API/SDK/Collector | Backend (storage + UI) | Backend (storage + UI) |
| Protocol | OTLP | Jaeger thrift | Zipkin JSON/Thrift |
| Storage | N/A | Cassandra, ES, Badger | Cassandra, ES, MySQL |
| Sampling | Configurable | Head-based | Configurable |
| Adoption | Industry standard | Widely used | Legacy |

### Head-based vs Tail-based Sampling

| Aspect | Head-based | Tail-based |
|--------|------------|------------|
| Decision timing | At trace start | At trace end |
| Resource usage | Lower | Higher (buffer all) |
| Error capture | Probabilistic | Guaranteed |
| Slow trace capture | Probabilistic | Guaranteed |
| Implementation | Simple | Complex |
| Best for | Most systems | Error-sensitive systems |

## 15. Revision Notes

### Quick Recap
- **Trace**: End-to-end request path across services.
- **Span**: Single timed operation within a trace.
- **Context Propagation**: Passing trace context across boundaries.
- **OpenTelemetry**: Industry standard API/SDK.
- **W3C TraceContext**: Standard header format (traceparent).
- **Sampling**: Reduce data volume (1-10% typical).
- **Jaeger/Zipkin**: Backend storage and visualization.
- **Auto-instrumentation**: No-code tracing via Java agent.

### Key Fields in Every Span
- trace_id, span_id, parent_span_id
- service.name, operation.name
- start_time, end_time, duration
- status (OK/ERROR)
- Attributes (method, url, status_code)

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                DISTRIBUTED TRACING CHEAT SHEET                     |
+-------------------------------------------------------------------+
| CONCEPT       | DESCRIPTION                                        |
+---------------+---------------+-----------------------------------+
| Trace         | Complete request path across services               |
| Span          | Single operation (timed)                            |
| Trace ID      | Unique per request, shared across spans             |
| Span ID       | Unique per span within a trace                     |
| Parent Span ID| Links spans in tree structure                      |
| Context       | Propagation of trace context across boundaries      |
+---------------+---------------+-----------------------------------+
| W3C TRACEPARENT HEADER                                             |
+-------------------------------------------------------------------+
| traceparent: 00-<traceId>-<spanId>-<flags>                        |
| Format: version-traceId-spanId-traceFlags                         |
| Example: 00-0af7651916cd43dd8448eb211c80319c-00f067aa0ba902b7-01  |
+-------------------------------------------------------------------+
| OPENOTELEMETRY QUICK REFERENCE                                     |
+-------------------------------------------------------------------+
| Tracer                    | Creates spans                           |
| SpanBuilder               | Configures and starts spans            |
| Span                      | Timed operation, has attributes        |
| Scope                     | Makes span active in current context   |
| Attributes                | Key-value metadata on span             |
| SpanKind                  | CLIENT, SERVER, INTERNAL, PRODUCER,    |
|                           | CONSUMER                                |
| Status                    | OK, ERROR (with description)           |
| SpanProcessor             | Process spans (batch export)           |
| Sampler                   | Decision: keep or drop trace           |
+-------------------------------------------------------------------+
| SPRING BOOT 3.x CONFIG                                             |
+-------------------------------------------------------------------+
| application.yml:                                                    |
|   spring.tracing.sampling.probability=0.1                          |
|   management.tracing.enabled=true                                  |
|   management.otlp.tracing.endpoint=http://jaeger:4318/v1/traces   |
|                                                                     |
| Gradle: implementation 'io.micrometer:micrometer-tracing-bridge-   |
|         bridge-otel'                                               |
+-------------------------------------------------------------------+
| COMMANDS                                                           |
+-------------------------------------------------------------------+
| Jaeger UI: http://localhost:16686                                  |
| Jaeger Query API: GET /api/traces?service=order-service            |
| OpenTelemetry Collector: otelcol --config config.yaml              |
+-------------------------------------------------------------------+
```
