# Metrics and Monitoring

---

## Overview

- **Definition:** Metrics are numerical measurements collected over time (latency, error rate, CPU usage). Monitoring is the practice of collecting, visualizing, and alerting on these metrics.
- **Why It Exists:** Metrics provide quantitative insight into system health, performance, and behavior. Together with logging and tracing, they form the observability foundation for understanding and operating distributed systems.
- **Key Concepts:** **Four Golden Signals** (Latency, Traffic, Errors, Saturation), **RED Method** (Rate, Errors, Duration for services), **USE Method** (Utilization, Saturation, Errors for resources), **Counter** (monotonically increasing), **Gauge** (up/down value), **Histogram** (distribution with percentile calculation), **SLO/SLI/SLA** (target, indicator, agreement).

---

## Core Concepts

- **Metric Types:** **Counter** — only increases (requests served, errors). **Gauge** — up and down (CPU, memory, queue size). **Timer/Histogram** — measures duration distribution and calculates percentiles (P50, P95, P99). **DistributionSummary** — measures size distributions.
- **Micrometer:** Metrics instrumentation library for Spring Boot. Provides vendor-neutral facade with binders for JVM, cache, database, thread pools. Exports to Prometheus, Datadog, etc.
- **Prometheus:** Pull-based time-series database. Scrapes `/actuator/prometheus`. Uses PromQL for querying. Supports alerting rules via Alertmanager.
- **Grafana:** Dashboard visualization tool. Connects to Prometheus, Elasticsearch, Loki, etc. Used for building real-time monitoring dashboards.

```java
// Custom business metrics
Counter orderCounter = Counter.builder("orders.created")
    .description("Total orders created").register(registry);

Timer.Sample sample = Timer.start(registry);
// ... do work ...
sample.stop(Timer.builder("order.latency")
    .tag("status", "success")
    .publishPercentiles(0.5, 0.95, 0.99)
    .register(registry));
```

---

## Common Mistakes

- **Too Many Metrics** — Every metric adds cost. Only add metrics you will act on. This *looks correct* because more data seems better — the cost in storage, query time, and cognitive load accumulates silently with every new metric added.
- **High Cardinality** — Tags with user IDs or session IDs blow up metric storage. Use low-cardinality tags (endpoint, status, service). This *looks correct* because adding a `user_id` tag seems like a natural way to debug per-user issues — the cardinality explosion only becomes visible when the TSDB runs out of memory.
- **No Alerts on Metrics** — Collecting without alerting is like a smoke detector without an alarm. This *looks correct* because the dashboards are visible and someone is "watching them" — the gap between "visible on a dashboard" and "someone is looking right now" is the failure mode.
- **Alert Fatigue** — Too many false alerts cause engineers to ignore them. Alert on SLO burn rate, not raw metric thresholds. This *looks correct* because each alert rule seems reasonable in isolation — the cumulative noise of 50 well-intentioned rules is only apparent after engineers start silencing them all.
- **Ignoring Business Metrics** — Technical metrics without business context don't tell the full story. This *looks correct* because pager-duty alerts fire on technical symptoms — the team doesn't realize they're fixing the smoke detector while the building burns until they see a customer churn report.

---

## Key Design Considerations

- **SLO/SLI Framework:** Define SLIs (latency P95, error rate), set SLO targets (P95 < 200ms, errors < 0.1%), measure compliance over rolling windows. Error budget = 100% - SLO.
- **Alert on Burn Rate:** Instead of raw thresholds, alert on how fast the error budget is consumed. Burn rate > 1 means budget will be exhausted before the window ends.
- **Cardinality Control:** Never tag metrics with user IDs, request IDs, or session IDs. Use endpoint, status code, service name, and deployment version instead.
- **Push vs Pull:** Prometheus pulls metrics from targets (good for long-lived services). Push (StatsD/Datadog) works better for ephemeral workloads and batch jobs.
- **Multi-Dimensional Monitoring:** Aggregate metrics by service, endpoint, status code, and region. Use recording rules for pre-computed aggregations.
- **Monitoring Maturity:** Reactive (customer reports) → Basic (dashboards, alerts) → Proactive (SLOs, error budgets) → Predictive (anomaly detection) → Automated (self-healing).

---

## Real-World Scenarios

### Scenario 1: SLO-Based Alerting Preventing Alert Fatigue
**Context:** An operations team has 200 alert rules. Most fire daily but are ignored because they're noise. The checkout service's P99 latency alert fires 50 times/day, but only 2 of those are actionable. Engineers mute the alerts.

**Resolution:** Switch to SLO-based alerting with error budgets. Define SLI: "P99 latency of checkout endpoint < 500ms." SLO: 99.9% of requests meet this target over 30 days. Error budget: 0.1% of requests = 43 minutes of bad requests per 30 days. Alert only when the error budget burn rate exceeds 1 (budget will be exhausted within the window). This eliminates 90% of the noise — alerts fire only when the system is genuinely degrading.

```java
// Custom business metrics with Micrometer
@RestController
@RequestMapping("/api/checkout")
public class CheckoutController {
    private final MeterRegistry registry;
    private final Counter orderCounter;
    private final DistributionSummary orderValueSummary;

    public CheckoutController(MeterRegistry registry) {
        this.registry = registry;
        this.orderCounter = Counter.builder("checkout.orders.total")
            .description("Total orders placed").register(registry);
        this.orderValueSummary = DistributionSummary.builder("checkout.order.value")
            .baseUnit("USD").publishPercentiles(0.5, 0.95, 0.99).register(registry);
    }

    @PostMapping
    public ResponseEntity<OrderResponse> checkout(@RequestBody CheckoutRequest request) {
        Timer.Sample sample = Timer.start(registry);
        try {
            Order order = orderService.placeOrder(request);
            orderCounter.increment();
            orderValueSummary.record(order.getTotal().doubleValue());
            sample.stop(Timer.builder("checkout.latency")
                .tag("status", "success")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(registry));
            return ResponseEntity.ok(OrderResponse.from(order));
        } catch (Exception e) {
            sample.stop(Timer.builder("checkout.latency")
                .tag("status", "error")
                .register(registry));
            throw e;
        }
    }
}
```

### Scenario 2: Cardinality Explosion from User-Level Tags
**Context:** A metrics engineer tags all metrics with `user_id` and `session_id` for "detailed analysis." The Prometheus TSDB grows from 10GB to 500GB in a week. Queries become slow, and the monitoring system crashes.

**Resolution:** Remove high-cardinality tags. Aggregate by `endpoint`, `status_code`, `service`, and `deployment_version` instead. User-level breakdowns belong in structured logs with trace IDs, not in metrics. The TSDB stabilizes at 30GB.

### Scenario 3: Capacity Planning with Metrics Trends
**Context:** The database query latency is slowly increasing (100ms → 150ms over 6 months). No alert fires because the SLO is 500ms. At 11 months, latency hits 450ms, users notice, and an incident is declared.

**Resolution:** Implement predictive monitoring. Track growth rates of all metrics (DB query latency, connection pool usage, disk I/O, request rate). Model when each resource reaches critical thresholds. When the growth trend projects hitting the limit within 30 days, create a proactive ticket. The DB latency issue is fixed (missing index added) before it becomes an incident.

---

## Scenario-Based Questions

1. **Q: You're on call for a checkout service with 50+ alert rules. Alerts fire constantly — "P99 latency > 500ms" fires 30 times/day but the team ignores it because it autocorrects. How do you make alerts actionable?**
    - A: Replace threshold-based alerts with SLO burn-rate alerts. Define an SLO: "95% of checkout requests complete in <500ms over 30 days." Error budget: 5% = 36 hours of bad requests. Alert only when the budget is burning faster than the SLO window can sustain. Multi-window approach: if error budget burns at 10x rate (budget exhausted in 3 days), alert immediately. This catches genuine degradation while ignoring brief blips that don't threaten the SLO.
    > **Interview follow-up:** The multi-window approach alerts on burn rate over 5 minutes and 30 minutes — but what if the error rate spikes for 4 minutes and 59 seconds, just under the first window? Is this a blind spot, and how would you catch it?

2. **Q: Your Prometheus server is running out of memory. You discover that each request is tagged with `user_id`, `session_id`, and `request_id`. TSDB size is 200GB for 10 services. How do you fix this?**
   - A: Remove `user_id`, `session_id`, and `request_id` from metric tags — they create millions of unique time series (cardinality explosion). Aggressive retention reduction: move data to Thanos/Cortex with longer retention. Keep metric tags to low-cardinality dimensions: `endpoint` (20 values), `status_code` (5 values), `service` (10 values), `region` (3 values). User-level analysis belongs in tracing and logs, not metrics.

3. **Q: You deploy a new service. What are the minimum metrics you need before accepting production traffic?**
   - A: RED method: Rate (requests/sec per endpoint), Errors (error rate per endpoint + by error type), Duration (latency P50/P95/P99 per endpoint). Plus: resource metrics (CPU, memory, GC, thread pool utilization, connection pool usage). Health check: liveness and readiness endpoint responses. These give you enough to detect and diagnose most issues.

4. **Q: Your SLO says "P99 latency < 1s." The current P99 is 800ms. Suddenly it drops to 200ms. Is this good news?**
    - A: Not necessarily. A sudden dramatic improvement could mean: (a) the cache is warming and serving faster (good), (b) requests are failing fast instead of being processed (bad — check error rate), or (c) the metric instrumentation is broken and under-reporting (bad). Always check error rate alongside latency. If error rate spiked at the same time, the "improvement" is actually the system failing fast and returning errors immediately.
    > **Interview follow-up:** You find that the latency drop correlates with an error rate spike — but the errors are handled gracefully (HTTP 200 with error body). Your SLI only measures HTTP status codes. How do you build a "good request" SLI that captures business-level failures without instrumenting every endpoint individually?

5. **Q: Your monitoring dashboard shows CPU at 90%, response times at 200ms P99, and error rate at 0.1%. Everything looks fine, but users report the site is slow. What's missing?**
   - A: You may be monitoring the wrong saturation metrics. High CPU with good response times could mean the database or external service is the bottleneck. Add: database query latency (P99), connection pool wait times, downstream service latency (via distributed tracing), thread pool queue depth, and external API call latency. The bottleneck is likely downstream of your application — the application is waiting for a response from a slow service.

6. **Q: Your team monitors 500+ metrics but never looks at dashboards. Incidents are discovered by customer complaints. How do you change this culture?**
   - A: Reduce to the "golden signals" — 5-10 critical metrics per service displayed on a single-pane-of-glass dashboard. Configure burn-rate alerts for SLOs — these fire before customers notice. Create a weekly "monitoring review" where the team discusses the top metrics. Add monitoring validation to the deployment pipeline: a deployment with regressions (latency increase >20%) triggers an automatic rollback. Make the dashboard the first thing engineers see when investigating an issue.

7. **Q: You need to set an SLO for a new payment service, but you don't have historical data. How do you determine the initial target?**
   - A: Start with a realistic target based on the upstream requirements. If the checkout page has a 2-second SLA, and it calls payment service, the payment SLA should be tighter (e.g., P95 < 500ms, P99 < 1s). Set a conservative initial target (e.g., 99.9% for a critical path). Measure for 2 weeks, then adjust. Err on the side of a tighter SLO — you can loosen it; tightening it later means renegotiating expectations.

8. **Q: Your metrics show 99.99% uptime, but users say the site is flaky and unreliable. How is this possible?**
   - A: Uptime measures server availability (HTTP 200 responses). It doesn't measure correctness or quality. The server may return 200 with an error message in the body, or load a page slowly but successfully. Switch to "good requests" as the SLI: requests that complete with status 2xx/3xx within the latency target and with correct business logic. "Availability" (server is up) is very different from "usability" (the system works correctly).

9. **Q: Your Prometheus setup scrapes every 15 seconds. During a 5-second traffic spike, all metrics are missed. How do you capture burst traffic?**
   - A: Reduce scrape interval to 5-10 seconds (at the cost of more storage). Use histogram buckets to capture distribution within the interval. Consider using a push gateway for short-lived jobs. For burst detection, use logs (which capture every request) for rate calculation rather than metrics. Or use Prometheus recording rules to aggregate higher-resolution data from service-level instrumentation.

10. **Q: Your team argues about whether to use counters, gauges, or histograms for a new metric. How do you decide?**
    - A: Simple rules: (1) Does it only increase? → Counter (request count, error count, bytes sent). (2) Does it go up and down? → Gauge (CPU, memory, queue depth, active connections). (3) Do you need percentiles? → Histogram (latency, payload size, wait time). (4) Do you need a distribution of values? → DistributionSummary (order amounts, file sizes, batch sizes). When in doubt, start with a Counter or Histogram — these are the most commonly useful types.

---

## Interview Questions

1. **What are the Four Golden Signals of monitoring?**
   - A: Latency (time to serve requests), Traffic (demand/throughput), Errors (failure rate), Saturation (how full the system is). Defined by Google SRE — these give a comprehensive view of system health.

2. **What is the difference between a counter and a gauge?**
   - A: Counter only increases (monotonically increasing — total requests, total errors). Use for cumulative counts. Gauge can go up and down (CPU usage, active connections, queue depth). Use for point-in-time measurements.

3. **What is the RED method?**
   - A: Rate (requests per second), Errors (failed requests per second), Duration (latency distribution: P50, P95, P99). Used for service-level monitoring. Analogy: USE method is for resources, RED method is for services.

4. **What is the USE method?**
   - A: Utilization (% time resource is busy), Saturation (degree of queued work), Errors (error count). Used for resource monitoring (CPU, disk, memory, network). Every resource should have USE metrics.

5. **What is an SLO and how do you measure it?**
   - A: Service Level Objective — a target for an SLI (Service Level Indicator). Example: "P95 latency < 200ms" (SLO) measured as "95% of requests complete in <200ms over 30 days" (SLI). Error budget = 100% - SLO target.

6. **How do you avoid metric cardinality explosion?**
   - A: Limit tag values to low-cardinality dimensions: endpoint (10-50 values), status code (5 values), service name (10-50). Never use user IDs, request IDs, session IDs, or timestamps as tags. Use logs or traces for high-cardinality data.

7. **What is the difference between a histogram and a summary?**
   - A: Histogram counts observations in configurable buckets on the client side; percentiles are calculated from bucket counts. Summary calculates quantiles server-side over a sliding window. Histograms are mergeable (good for aggregating across instances); summaries are not.

8. **How do you set up alerting for a service?**
   - A: Define SLOs → Configure Prometheus alert rules → Route via Alertmanager. Alert on SLO burn rate (how fast error budget is consumed) not raw thresholds. Multi-window: faster burn → more urgent alert. Example: "Error budget burn rate > 10x over 5 minutes" → critical alert.

9. **What metrics would you monitor for a database?**
   - A: Query latency (P50, P95, P99), active vs idle connections, I/O wait time, replication lag, cache hit ratio (buffer pool), deadlock rate, connection pool utilization, table/index size growth, slow query count.

10. **How do you monitor for a massive traffic spike?**
    - A: Ensure monitoring scales independently (Prometheus with Thanos/Cortex for long-term storage). Use adaptive scrape intervals. Prioritize critical metrics over verbose ones. Validate that the monitoring infrastructure doesn't compete with the application for resources. Use pre-computed recording rules for expensive queries.

---

## Developer Recommendations

- **Focus on the RED method (Rate, Errors, Duration) for every service** — These three metrics provide enough information to detect and diagnose most issues. Every service should export: requests/sec per endpoint, error rate per endpoint + error type, and latency distribution (P50/P95/P99). Add SLOs for these metrics and alert on burn rates. Before adding any other metric, ensure RED metrics are implemented correctly. One team skipped this and added 200 custom business metrics first — when their payment service slowed down, they couldn't tell whether it was a rate spike, an error surge, or a latency regression because none of the RED fundamentals were in place.

- **Never tag metrics with high-cardinality values** — User IDs, request IDs, session IDs, and email addresses as metric tags cause cardinality explosions that crash monitoring systems. A tag with 10K unique values creates 10K time series. At 100 services × 100 metrics × 10K values = 100M time series = unusable monitoring. Use structured logs or traces for high-cardinality data; use metrics for aggregated, low-cardinality signals.

- **Alert on SLO burn rate, not static thresholds** — Static threshold alerts ("P99 > 500ms") are noisy and frequently ignored. SLO burn-rate alerts calculate how fast the error budget is being consumed. If the 30-day window can tolerate 43 minutes of bad requests, a 5-minute spike isn't actionable. Only alert when the burn rate exceeds a threshold that threatens the SLO. This eliminates 90%+ of alert noise.

- **Start with RED metrics, add business metrics, then custom metrics** — RED metrics tell you if the service is technically healthy. Business metrics (orders/sec, revenue/sec, signups/sec, cart abandonment rate) tell you if the system is delivering value. Custom metrics (cache hit ratio, connection pool utilization, queue depth) help diagnose the cause. This hierarchy ensures you measure what matters: first whether the system works, then whether the business runs.

- **Use histograms for latency metrics, not averages** — Average latency hides 90% of problems. A 100ms average could mean 99% of requests complete in 10ms and 1% complete in 9 seconds. Use histograms with percentile reporting (P50, P95, P99, P999) to capture the full distribution. Configure percentile buckets appropriate to your service: for a payment API, P50 < 200ms, P99 < 1s, P999 < 3s.

- **Instrument metrics in your code, don't rely only on infrastructure monitoring** — Infrastructure metrics (CPU, memory, disk) tell you about the server, not the application. Code-level metrics (request latency per endpoint, error rate per business operation, cache hit ratio, queue depth, thread pool utilization) tell you about the actual service health. Use Micrometer in Spring Boot for framework-provided metrics (JVM, database, cache) and add custom metrics for business operations.
