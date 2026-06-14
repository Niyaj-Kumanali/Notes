# Distributed Tracing

---

## Overview

- **Definition:** Distributed tracing tracks requests as they flow through multiple services in a distributed system by assigning each request a unique trace ID propagated across service boundaries.
- **Why It Exists:** In microservices, a single request crosses many services. Without tracing, you cannot see end-to-end latency, identify which service is slow, or understand error propagation.
- **Key Concepts:** **Trace** (complete path of a request, a tree of spans), **Span** (named, timed unit of work with span ID, parent span ID, trace ID), **Context Propagation** (passing trace context via HTTP headers or message metadata), **Sampling** (capturing only a fraction of traces to control cost).

---

## Core Concepts

- **OpenTelemetry:** Industry standard combining OpenTracing and OpenCensus. Provides API, SDK, auto-instrumentation, and exporter (OTLP protocol) to backends like Jaeger or Zipkin.
- **W3C TraceContext:** Standard header format: `traceparent: 00-{traceId}-{spanId}-{flags}`. Ensures interoperability across different tracing systems.
- **Sampling Strategies:** **Head-based** (decision at trace start — simple, may miss errors), **Tail-based** (keep all, decide later — captures all errors, more resource intensive), **Probabilistic** (random %), **Rate-limiting** (fixed traces/sec).
- **SpanKind:** CLIENT (outgoing call), SERVER (incoming), INTERNAL (internal op), PRODUCER/CONSUMER (messaging).

```java
// Manual span creation
Span span = tracer.spanBuilder("createOrder")
    .setSpanKind(SpanKind.SERVER)
    .setAttribute("user.id", request.getUserId())
    .startSpan();
try (Scope scope = span.makeCurrent()) {
    Order order = new Order(request);
    reserveInventory(order);
    span.setStatus(StatusCode.OK);
    return order;
} catch (Exception e) {
    span.recordException(e);
    span.setStatus(StatusCode.ERROR);
    throw e;
} finally { span.end(); }
```

---

## Common Mistakes

- **No Sampling** — Capturing every trace creates massive data volume and cost. Use 1-10% sampling in production.
  - **Why it looks correct:** The tracing system works perfectly in dev with 100% sampling — the cost only becomes apparent in the first production billing cycle.
- **Too Many Spans** — Every method call as a span creates noise. Focus on service boundaries and significant operations.
  - **Why it looks correct:** Each span seems useful for debugging — the noise and overhead compound as every inner method adds a span to every trace.
- **No Context Propagation** — Tracing breaks when context isn't passed to async threads or message queues.
  - **Why it looks correct:** The synchronous path traces perfectly — the broken trace links in async processing are invisible since the spans appear disconnected rather than missing.
- **Missing Error Attributes** — Errors without exception details in spans are impossible to diagnose.
  - **Why it looks correct:** The span shows an error status — the developer assumes they can find details in the logs, but without a trace ID in the log entry, correlation is manual.

---

## Key Design Considerations

- **Auto-Instrumentation** — Use OpenTelemetry Java agent for zero-code tracing of HTTP, JDBC, Kafka, gRPC, Redis, etc. Covers most common libraries.
- **Trace-Log Correlation** — Include `trace_id` and `span_id` in log MDC. Query logs by trace ID to correlate with traces.
- **Trace-Based SLOs** — Define SLOs like "95% of checkout traces complete within 2s." Monitor compliance over rolling windows.
- **Sampling for Errors** — Tail-based sampling keeps 100% of ERROR traces while sampling normal ones, ensuring no errors are missed.
- **Multi-Signal Observability** — Combine traces, logs, and metrics with common labels (trace_id, service, environment) for unified debugging.

---

## Real-World Scenarios

### Scenario 1: Debugging a Slow Order Checkout
**Context:** Users report that checkout takes 5+ seconds intermittently. The order service calls inventory, payment, and shipping services. No one knows which service is slow or why.

**Resolution:** Implement distributed tracing with OpenTelemetry. Auto-instrument all services (HTTP, database, messaging). Each checkout request gets a unique trace ID. Tracing reveals that the payment service's database query (`SELECT ... FROM transactions WHERE user_id = ?`) has a missing index and takes 3 seconds for users with many transactions. The trace shows the exact span causing the slowdown.

```java
// Auto-instrumentation — zero code changes needed
// Add OpenTelemetry Java agent to JVM args:
// -javaagent:opentelemetry-javaagent.jar

// Manual span creation for custom business logic
@Service
public class CheckoutService {
    private final Tracer tracer;

    public Order checkout(CheckoutRequest request) {
        Span span = tracer.spanBuilder("checkout")
            .setAttribute("user.id", request.getUserId())
            .setAttribute("order.total", request.getTotal().toString())
            .startSpan();
        try (Scope scope = span.makeCurrent()) {
            validateInventory(request.getItems());  // Automatically traced
            Order order = createOrder(request);
            span.setStatus(StatusCode.OK);
            return order;
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally { span.end(); }
    }
}
```

### Scenario 2: Tracing Across Kafka Messages
**Context:** An order event flows through Kafka: Order Service → Payment Service → Shipping Service. When a shipping delay occurs, it's unclear if the issue is in publishing, consuming, or processing.

**Resolution:** OpenTelemetry auto-instruments Kafka producers and consumers. The trace context is propagated in Kafka record headers. The trace shows: produce time in OrderService → queue time (offset from produce to consume) → process time in PaymentService → produce to Shipping → process time. This identifies whether the delay is in queueing, processing, or network.

### Scenario 3: Trace-Log Correlation for Incident Response
**Context:** An outage occurs. Errors are logged across 10 services. Without correlation, engineers spend hours manually matching timestamps to understand the request flow.

**Resolution:** Configure all services to emit structured JSON logs with `trace_id` and `span_id` in MDC. During the incident, search the log aggregator for the trace ID of an error trace. All logs for that request appear, ordered by span hierarchy, across all services. The root cause is identified in minutes instead of hours.

## Use Cases

- **Latency bottleneck identification** — debugging why a checkout request takes 10 seconds
  - Trace waterfall shows each span's duration across services. The longest span identifies the bottleneck — a slow database query, a blocked HTTP call, or network latency.
  - **Avoid when:** your system is a single service with no async calls — application performance monitoring (APM) of a single service is sufficient.

- **Error root cause analysis** — finding which service returns an error and why
  - Traces capture error context across the entire request path. Drill into the failing span to see the exception, parameters, and state at the point of failure.
  - **Avoid when:** the error is consistent and easily reproduced locally — local debugging may be faster than tracing analysis.

- **Dependency mapping** — discovering service dependencies, critical paths, and dead or unused services
  - Auto-instrumentation generates a live dependency graph. Shows which services call which, call frequency, error rates, and latency per edge.
  - **Avoid when:** service topology is already well-documented and stable — tracing for dependency discovery has diminishing returns.

- **SLA/SLO compliance monitoring** — measuring end-to-end latency for critical user journeys
  - Traces measure the complete request flow from entry point to response. Compare P50, P95, and P99 latencies against business SLAs.
  - **Avoid when:** you only need service-level latency — RED metrics (Rate, Errors, Duration) per service are sufficient for SLO tracking.

- **Sampling-based performance analysis** — understanding representative request performance without storing every trace
  - Head-based sampling (fixed rate, e.g., 1%) or tail-based sampling (keep all errors + sample of slow traces) balances detail with storage cost.
  - **Avoid when:** every single request must be traceable for audit — use head-based sampling with 100% sampling for audited transactions only.

---

## Scenario-Based Questions

1. **Q: Users report that checkout takes 10 seconds intermittently. You suspect it's a specific downstream service, but traditional monitoring shows all services are healthy. How do you identify the root cause?**
    - A: Implement distributed tracing with OpenTelemetry. Auto-instrument all services. Each checkout request creates a trace with spans for each service call. The trace shows the exact duration of each span. Look at the trace waterfall — the service with the longest span is the bottleneck. Drill into that span's attributes (database query, HTTP URL) to find the root cause. Use tail-based sampling to capture all error traces.
    - **Interview follow-up:** Tail-based sampling keeps all error traces, but what defines an "error"? A downstream service returning a 400 status might be correct business logic, not a system failure. How do you distinguish between "expected errors" that can be sampled and "real errors" that must be kept?

2. **Q: You have 500 microservices. Tracing every request generates 10TB of data per day. Storage costs are exploding. How do you reduce data volume while keeping useful traces?**
   - A: Implement sampling. Head-based probabilistic sampling (e.g., 1% of all traces) for general monitoring. Tail-based sampling to keep 100% of error traces (regardless of rate) and 10% of slow traces (>P95 latency). Use rate-limiting sampling to cap at 100 traces/second for high-traffic endpoints. Configure different sampling rates per service — critical services (payments) get higher sampling than trivial ones (health checks).

3. **Q: Your tracing system shows spans from most services, but Kafka message processing appears as disconnected spans — they don't connect to the parent trace. What's broken?**
   - A: Context propagation across Kafka is not working. OpenTelemetry auto-instruments Kafka for producer/consumer, but you may need to configure it explicitly. Ensure the Kafka instrumentation is enabled (`otel.instrumentation.kafka.enabled=true`). Verify that the producer injects trace context into Kafka record headers, and the consumer extracts it. Check the OpenTelemetry agent version — older versions may not support Kafka headers.

4. **Q: You manually create spans for business operations, but junior developers keep forgetting to close spans, causing dangling spans that break the trace. How do you enforce proper span management?**
   - A: Use OpenTelemetry's auto-instrumentation as much as possible — it handles span lifecycle correctly. For manual spans, enforce the try-with-resources pattern using code reviews and static analysis. Create a wrapper utility that ensures spans are always closed:

```java
public static <T> T trace(String name, Map<String, String> attributes, Supplier<T> block) {
    Span span = tracer.spanBuilder(name).startSpan();
    attributes.forEach(span::setAttribute);
    try (Scope scope = span.makeCurrent()) {
        return block.get();
    } catch (Exception e) {
        span.recordException(e); span.setStatus(StatusCode.ERROR); throw e;
    } finally { span.end(); }
}
```

5. **Q: Your microservices run asynchronously — the order service pushes a message to Kafka and the payment service processes it minutes later. The trace doesn't connect them. How do you correlate asynchronous processing?**
   - A: OpenTelemetry's Kafka instrumentation handles this. Trace context is serialized into the Kafka message headers by the producer. When the consumer deserializes the message, it extracts the context and creates a child span linked to the parent trace. The trace shows a gap (the queue time) between the producer span end and consumer span start. This gap is the time the message spent in Kafka, which is useful for monitoring consumer lag.

6. **Q: Your tracing backend (Jaeger) is down. Do traces still propagate through services?**
    - A: Yes. Tracing instrumentation is non-blocking and should never affect application performance or reliability. If the exporter can't reach the backend, spans are dropped (or buffered if configured with memory/disk buffer). Trace context propagation via HTTP headers continues regardless — services pass trace IDs downstream even if spans aren't exported. The application works normally; you just temporarily lose visibility.
    - **Interview follow-up:** If the exporter drops spans because the backend is unreachable, the application is healthy but you have no visibility — exactly when you need tracing most. If you buffer spans in memory, what happens to the buffer during a sustained backend outage? Does it cause OOM?

7. **Q: You deploy a new service that doesn't use any tracing library. Requests through this service appear as broken traces — no spans from this service. How do you fix this?**
   - A: Add the OpenTelemetry Java agent to the new service's JVM arguments. Auto-instrumentation covers HTTP, database, messaging, and many other libraries without code changes. If the service is in a different language (Node.js, Python, Go), use the appropriate OpenTelemetry SDK and auto-instrumentation package. The trace context is propagated via standard W3C headers, so it works across languages.

8. **Q: Your tracing shows that 95% of checkout traces complete in 1 second, but 5% take 10+ seconds. All services show similar latency distributions independently. How does tracing help find the root cause?**
   - A: The trace waterfall reveals whether the slow requests are slow in the same service every time or in different services. If a specific combination of input data causes slowness (e.g., users with 10K+ orders), the trace attributes (user ID, order count) from the first service's span will correlate with slow downstream spans. Use trace-based analytics: group traces by user tier, look for patterns in the slow traces.

9. **Q: Your system uses gRPC for inter-service communication. OpenTelemetry's auto-instrumentation doesn't capture gRPC spans. How do you add tracing for gRPC?**
   - A: OpenTelemetry supports gRPC instrumentation via the `opentelemetry-instrumentation-grpc` library. For Spring Boot gRPC (grpc-spring-boot-starter), add the OpenTelemetry gRPC instrumentation dependency. Configure client and server interceptors that propagate trace context via gRPC metadata. The auto-instrumentation agent includes gRPC instrumentation when the library is on the classpath.

10. **Q: You need to trace requests that start from a mobile app and go through your API Gateway. How do you propagate the trace ID from the mobile client?**
    - A: The mobile app generates a trace ID and sends it in the `traceparent` HTTP header. The API Gateway and downstream services recognize the W3C TraceContext header and continue the trace. On the mobile side, use OpenTelemetry's mobile SDK (or manually generate a trace ID). If the mobile app can't add tracing headers, the API Gateway generates the trace ID as the entry point. All subsequent services propagate it.

---

## Interview Questions

1. **What is distributed tracing?**
   - A: A technique that tracks a request across multiple services by propagating a unique trace ID, collecting timed spans (operations) from each service to reconstruct the end-to-end request path and identify bottlenecks.

2. **What is the difference between a trace and a span?**
   - A: A trace is the complete end-to-end request path — a tree of spans. A span is a single timed operation within that trace (e.g., an HTTP call, database query, or function execution). Spans have parent-child relationships forming the trace tree.

3. **What is context propagation?**
   - A: The mechanism of passing trace context (trace ID, span ID) across service boundaries via HTTP headers (`traceparent`), message metadata (Kafka headers), or gRPC metadata. Ensures all spans from a single request link to the same trace.

4. **What is OpenTelemetry?**
   - A: The industry standard observability framework combining OpenTracing and OpenCensus. Provides API, SDK, auto-instrumentation, and a vendor-agnostic protocol (OTLP) for exporting traces, metrics, and logs to any backend (Jaeger, Zipkin, Datadog, etc.).

5. **How do you trace a Kafka message?**
   - A: OpenTelemetry auto-instruments Kafka producers and consumers. Trace context is injected into Kafka record headers as `traceparent` on produce. On consume, the context is extracted, creating a child span linked to the parent trace.

6. **What is head-based vs tail-based sampling?**
   - A: Head-based: sampling decision made at trace start (e.g., 1% of all traces). Simple but may miss errors. Tail-based: buffer all traces, decide later — keeps 100% of errors and slow traces while sampling normal traces. More resource-intensive but captures all incidents.

7. **How do you correlate traces with logs?**
   - A: Include `trace_id` and `span_id` in the logging MDC. Configure the logging framework to emit these fields in structured JSON logs. Search logs by trace ID to see all log entries for a specific request across all services.

8. **What is the W3C traceparent header format?**
   - A: `traceparent: 00-{32-char traceId}-{16-char spanId}-{2-char flags}`. Example: `00-0af7651916cd43dd8448eb211c80319c-00f067aa0ba902b7-01`. The flags field indicates sampling (01 = sampled).

9. **How do you propagate context across async boundaries?**
   - A: Store the current context before the async operation, restore it in the async thread. OpenTelemetry's `Context.wrap(Runnable)` or `Context.current().makeCurrent()` handles this. Auto-instrumentation wraps thread pools automatically.

10. **Design a tracing system for 500+ microservices.**
    - A: OpenTelemetry auto-instrumentation on all services. OpenTelemetry Collector per cluster for batching, filtering, and retries. Kafka as a buffering layer between collectors and backend. Jaeger or Grafana Tempo backend with object storage (S3/GCS). Sampling: head-based 1% for general, tail-based for errors. Trace-log-metric correlation via common attributes (service, trace_id).

---

## Developer Recommendations

- **Use OpenTelemetry auto-instrumentation as the default, manual spans only for business logic** — Auto-instrumentation covers HTTP, gRPC, database calls, messaging, and caching without any code changes. Add manual spans only for business operations that the auto-instrumentation can't capture (e.g., "processOrder" or "applyDiscount"). This gives 90% of tracing value with 10% of the effort.
  - **Production story:** A team spent 3 weeks hand-instrumenting every service before discovering the OpenTelemetry Java agent would have covered 95% of it in one afternoon.

- **Propagate trace context everywhere, including async and messaging** — The most common tracing failure is broken context propagation. OpenTelemetry handles this for standard patterns, but verify: HTTP headers (`traceparent`), Kafka/RabbitMQ message headers, gRPC metadata, and async thread pools (`ExecutorService` wrap). Without propagation, traces break at service boundaries, and you lose end-to-end visibility.

- **Include trace_id and span_id in all structured log output** — A trace without logs is missing context; logs without a trace ID are impossible to correlate. Configure MDC to include `trace_id`, `span_id`, `service`, and `environment` in every log entry. This enables "click from trace to logs" debugging. OpenTelemetry's auto-instrumentation automatically populates MDC.

- **Use tail-based sampling to capture every error trace** — Head-based probabilistic sampling misses most errors because errors are rare (typically 0.1-1% of traffic). Tail-based sampling buffers all traces and keeps 100% of error traces. This ensures no error goes undiagnosed. The extra storage cost for error traces is negligible compared to the debugging time saved.

- **Set meaningful span attributes for business context** — Auto-instrumentation sets technical attributes (HTTP method, URL, status code). Add business attributes that help debugging: `user.id`, `order.id`, `payment.amount`, `error.category`. These attributes make traces searchable and actionable. Don't add PII or high-cardinality attributes (user IDs are usually fine within a trace system).

- **Monitor tracing system health separate from application health** — The tracing pipeline (agent → collector → backend) can fail without affecting the application. Monitor: trace ingestion rate (sudden drop means instrumentation failure), span export latency, collector queue depth, and backend storage utilization. Without this, you may lose visibility during the very incidents you need tracing for.
