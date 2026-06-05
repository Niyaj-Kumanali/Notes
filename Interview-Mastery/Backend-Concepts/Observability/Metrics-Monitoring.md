# Metrics and Monitoring

## 1. Executive Summary

Metrics are numerical measurements collected over time that provide insight into system health, performance, and behavior. Monitoring is the practice of collecting, visualizing, and alerting on these metrics to ensure system reliability and performance. Together, they form the observability foundation alongside logging and tracing. Key categories include application metrics (latency, error rate, throughput), system metrics (CPU, memory, disk), and business metrics (orders, users, revenue).

## 2. Core Theory

### The Four Golden Signals

Google's SRE book defines four key metrics for user-facing systems:

1. **Latency**: Time to service a request.
2. **Traffic**: Demand on the system (requests per second).
3. **Errors**: Rate of failed requests.
4. **Saturation**: How "full" the system is (resource utilization).

### RED Method (for services)

- **Rate**: Requests per second.
- **Errors**: Failed requests per second.
- **Duration**: Latency distribution.

### USE Method (for resources)

- **Utilization**: Average time resource was busy.
- **Saturation**: Degree of extra work queued.
- **Errors**: Count of error events.

### Metric Types

1. **Counter**: Monotonically increasing value (requests served, errors).
2. **Gauge**: Single numeric value that can go up and down (CPU, memory, queue size).
3. **Histogram**: Samples observations, counts them in configurable buckets (request latency).
4. **Summary**: Similar to histogram but calculates configurable quantiles over sliding time window.

## 3. Under-the-Hood Deep Dive

### Micrometer Integration

Micrometer is the metrics instrumentation library for Spring Boot. It provides a vendor-neutral facade for metrics.

```java
// Counter
Counter requestCounter = Counter.builder("api.requests")
    .tag("endpoint", "/api/orders")
    .tag("method", "POST")
    .register(meterRegistry);

requestCounter.increment();

// Gauge
Gauge.builder("jdbc.connections.active", dataSource,
        ds -> ((HikariDataSource) ds).getHikariPoolMXBean().getActiveConnections())
    .tag("pool", "primary")
    .register(meterRegistry);

// Timer
Timer.Sample sample = Timer.start(meterRegistry);
// ... do work ...
sample.stop(Timer.builder("api.latency")
    .tag("endpoint", path)
    .tag("status", String.valueOf(status))
    .publishPercentiles(0.5, 0.95, 0.99)
    .publishPercentileHistogram()
    .register(meterRegistry));

// DistributionSummary (for sizes)
DistributionSummary summary = DistributionSummary.builder("response.size")
    .baseUnit("bytes")
    .minimumExpectedValue(1L)
    .maximumExpectedValue(10_000_000L)
    .publishPercentiles(0.5, 0.95)
    .register(meterRegistry);

summary.record(response.getBody().length());
```

### Prometheus Exposition Format

Prometheus scrapes metrics from HTTP endpoints (typically /actuator/prometheus).

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
```

## 4. Production Code Examples

### Custom Business Metrics

```java
@Configuration
public class BusinessMetricsConfig {

    @Bean
    public Counter orderCreatedCounter(MeterRegistry registry) {
        return Counter.builder("orders.created")
            .description("Total number of orders created")
            .register(registry);
    }

    @Bean
    public Counter orderCancelledCounter(MeterRegistry registry) {
        return Counter.builder("orders.cancelled")
            .description("Total number of orders cancelled")
            .register(registry);
    }

    @Bean
    public Gauge pendingOrdersGauge(MeterRegistry registry,
            OrderRepository orderRepository) {
        return Gauge.builder("orders.pending", orderRepository,
                repo -> repo.countByStatus(OrderStatus.PENDING))
            .description("Number of pending orders")
            .register(registry);
    }
}

@Service
public class OrderMetricsService {

    private final Counter orderCreatedCounter;
    private final Counter orderCancelledCounter;
    private final Timer orderProcessingTimer;
    private final MeterRegistry meterRegistry;

    @Timed(value = "order.processing", percentiles = {0.5, 0.95, 0.99})
    public Order processOrder(CreateOrderRequest request) {
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            Order order = createOrder(request);
            orderCreatedCounter.increment();

            // Record order value
            meterRegistry.summary("order.value", "currency", "USD")
                .record(order.getTotalAmount().doubleValue());

            sample.stop(Timer.builder("order.processing.latency")
                .tag("status", "success")
                .register(meterRegistry));

            return order;
        } catch (Exception e) {
            sample.stop(Timer.builder("order.processing.latency")
                .tag("status", "failure")
                .register(meterRegistry));
            throw e;
        }
    }

    public void cancelOrder(String orderId) {
        orderCancelledCounter.increment();
        // cancel logic
    }
}
```

### Health Indicators

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(1000)) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("validationQuery", "SELECT 1")
                    .build();
            }
            return Health.down()
                .withDetail("error", "Connection validation failed")
                .build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}

@Component
public class RedisHealthIndicator implements HealthIndicator {

    @Autowired
    private RedisConnectionFactory redisConnectionFactory;

    @Override
    public Health health() {
        try {
            RedisConnection connection = redisConnectionFactory.getConnection();
            String result = connection.ping();
            connection.close();
            return Health.up()
                .withDetail("version", result)
                .build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

### Custom Metrics with Tags

```java
@Component
public class TaggedMetricsService {

    private final MeterRegistry meterRegistry;

    public TaggedMetricsService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void recordApiCall(String endpoint, int status, long durationMs) {
        // Using tags for dimensional metrics
        Timer.builder("api.calls")
            .tag("endpoint", endpoint)
            .tag("status", String.valueOf(status))
            .tag("method", determineMethod(endpoint))
            .tag("version", "v2")
            .register(meterRegistry)
            .record(Duration.ofMillis(durationMs));
    }

    public void recordExternalApiCall(String apiName, boolean success, long durationMs) {
        Timer.builder("external.api.calls")
            .tag("api", apiName)
            .tag("result", success ? "success" : "failure")
            .register(meterRegistry)
            .record(Duration.ofMillis(durationMs));

        Counter.builder("external.api.errors")
            .tag("api", apiName)
            .register(meterRegistry);
    }
}
```

### Prometheus and Grafana Configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-apps'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets:
          - 'order-service:8080'
          - 'payment-service:8081'
          - 'user-service:8082'
```

```yaml
# Docker Compose for monitoring stack
version: '3'
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
```

### Alerting Rules

```yaml
# prometheus-alerts.yml
groups:
  - name: backend-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m])
              / rate(http_server_requests_seconds_count[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.instance }}"

      - alert: HighLatency
        expr: histogram_quantile(0.95,
              rate(http_server_requests_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P95 latency on {{ $labels.instance }}"

      - alert: ServiceDown
        expr: up{job="spring-boot-apps"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} is down"

      - alert: HighCpuUsage
        expr: system_cpu_usage > 0.9
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage > 90% on {{ $labels.instance }}"

      - alert: HighMemoryUsage
        expr: jvm_memory_used_bytes{area="heap"}
              / jvm_memory_max_bytes{area="heap"} > 0.9
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Heap usage > 90% on {{ $labels.instance }}"
```

## 5. Real-World Scenarios

### E-Commerce Monitoring Dashboard

Grafana Dashboard Panels:
1. **Traffic**: Request rate per service (rate counter).
2. **Latency**: P50/P95/P99 latency per endpoint (histogram).
3. **Errors**: Error rate per service (rate counter / total rate).
4. **Saturation**: CPU, memory, connection pools (gauges).
5. **Business**: Order creation rate, revenue, cart abandonment (custom counters).
6. **Database**: Query latency, connections, replication lag.

### Incident Response Flow

```
1. Prometheus Alert: "Error rate > 5% on payment-service"
2. Grafana: Check payment-service dashboard
3. See latency spike + error rate increase
4. Drill down to external API call metrics
5. Identify payment gateway timeout
6. Check circuit breaker: OPEN
7. Check logs: gateway errors
8. Root cause: Payment gateway degraded
9. Respond: Switch to backup gateway
10. Monitor: Error rate drops, latency normalizes
```

## 6. Performance

### Metrics Collection Overhead

| Operation | Overhead |
|-----------|----------|
| Counter increment | < 1 microsecond |
| Gauge read | < 1 microsecond |
| Timer sample | 1-2 microseconds |
| Histogram record | 2-5 microseconds |
| Prometheus scrape (1000 metrics) | 10-50ms |

### Best Practices for Minimizing Overhead
- Use approximate percentiles (histogram) instead of exact (summary).
- Limit number of unique tag combinations to avoid high cardinality.
- Use delta (rate) over counter in queries, not raw counters.
- Scrape at appropriate intervals (15-60s).

## 7. Security

### Metrics Security

- Never expose sensitive data in metric names or tags.
- Use authentication (Spring Security) on /actuator/prometheus.
- Restrict Prometheus access to monitoring network.
- Don't tag metrics with user IDs (high cardinality + privacy).

```java
// WRONG - exposes user IDs as tags
Counter.builder("api.calls")
    .tag("userId", userId) // High cardinality, privacy risk
    .register(registry);

// RIGHT - aggregate by endpoint
Counter.builder("api.calls")
    .tag("endpoint", endpoint)
    .register(registry);
```

## 8. Common Mistakes

### Mistake 1: Too Many Metrics
Every new metric adds cost (storage, query time, network). Only add metrics you will act on.

### Mistake 2: High Cardinality Metrics
Tags with high cardinality (user IDs, session IDs, request IDs) blow up metric storage.

### Mistake 3: No Alerts on Metrics
Collecting metrics without alerts is like having a smoke detector without an alarm.

### Mistake 4: Alert Fatigue
Too many false alerts cause engineers to ignore them.

### Mistake 5: Ignoring Business Metrics
Technical metrics without business context don't tell the full story.

## 9. Senior Engineer Perspective

### SLO / SLI / SLA Framework

- **SLI (Service Level Indicator)**: A metric that measures a specific aspect of service level (e.g., request latency P95).
- **SLO (Service Level Objective)**: Target value for an SLI (e.g., P95 latency < 200ms).
- **SLA (Service Level Agreement)**: Contract with customers based on SLOs.

### Implementation

```java
@Component
public class SloMetrics {

    private final MeterRegistry meterRegistry;
    private final double LATENCY_SLO = 200; // ms
    private final double ERROR_SLO = 0.01;  // 1% error rate

    public SloMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void recordRequest(long latencyMs, int status) {
        boolean withinSlo = latencyMs <= LATENCY_SLO && status < 500;

        Counter.builder("slo.requests.total")
            .register(meterRegistry)
            .increment();

        Counter.builder("slo.requests.good")
            .tag("within_slo", String.valueOf(withinSlo))
            .register(meterRegistry)
            .increment();

        // Track SLO compliance as a gauge
        // (good_requests / total_requests) over rolling window
        meterRegistry.gauge("slo.compliance",
            Tags.of("indicator", "latency"),
            this,
            SloMetrics::calculateCompliance);
    }

    private double calculateCompliance() {
        // Calculate from counters over rolling window
        return sloGoodCount / Math.max(sloTotalCount, 1);
    }
}
```

### Monitoring Maturity Model

| Level | Characteristics |
|-------|-----------------|
| 1. Reactive | "The customer told us it's down" |
| 2. Basic | Dashboards, basic alerts |
| 3. Proactive | SLOs, error budgets, auto-scaling |
| 4. Predictive | Anomaly detection, capacity forecasting |
| 5. Automated | Self-healing, auto-remediation |

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What is a metric?
   **A:** A numerical measurement collected over time, representing system behavior (latency, error rate, CPU usage).

2. **Q:** What is the difference between counter and gauge?
   **A:** Counter only increases (requests served). Gauge can go up and down (CPU usage, active connections).

3. **Q:** What is a histogram in monitoring?
   **A:** Samples observations and counts them in configurable buckets, enabling percentile calculation.

4. **Q:** What is Prometheus?
   **A:** An open-source monitoring system that scrapes metrics from configured targets, stores them in a time-series database.

5. **Q:** What is Grafana?
   **A:** A visualization tool for creating dashboards from metrics stored in Prometheus, Elasticsearch, and other data sources.

6. **Q:** What is the RED method?
   **A:** Rate (requests/sec), Errors (failed requests/sec), Duration (latency distribution). Used for service monitoring.

7. **Q:** What is the USE method?
   **A:** Utilization, Saturation, Errors. Used for resource monitoring.

8. **Q:** What is an SLI, SLO, and SLA?
   **A:** SLI (Service Level Indicator - the metric), SLO (Service Level Objective - target value), SLA (Service Level Agreement - contract).

9. **Q:** What is the Micrometer library?
   **A:** A metrics instrumentation library for JVM applications, providing vendor-neutral API. Default in Spring Boot.

10. **Q:** What is the difference between white-box and black-box monitoring?
    **A:** White-box: metrics from inside the system (application metrics). Black-box: metrics from outside (ping, external health check).

### Medium

11. **Q:** How do you avoid metric cardinality explosion?
    **A:** Limit tag values (no user IDs, request IDs). Use low-cardinality tags (endpoint, status, service). Use metric naming conventions.

12. **Q:** How do you set up alerting?
    **A:** Define SLOs. Create Prometheus alert rules. Alert on SLO violation rate, not raw metric values. Use proper threshold + duration to avoid flapping.

13. **Q:** What metrics would you monitor for a database?
    **A:** Query latency, connections active/idle, I/O wait, replication lag, deadlocks, table size, cache hit ratio.

14. **Q:** How do you monitor a message queue (Kafka)?
    **A:** Consumer lag, request latency, partition count, under-replicated partitions, message rate, disk usage.

15. **Q:** What is an error budget?
    **A:** The acceptable amount of unreliability, calculated as 100% - SLO. Used to balance reliability vs feature velocity.

16. **Q:** How do you monitor user-facing latency?
    **A:** Application-level timing with Micrometer Timer. Browser performance metrics (RUM). CDN edge performance.

17. **Q:** What is the difference between rate and increase in PromQL?
    **A:** rate() calculates per-second average rate over a time range. increase() calculates total increase over a time range.

18. **Q:** How do you monitor batch jobs?
    **A:** Track: job duration (timer), items processed (counter), success/failure (counter), last success timestamp (gauge).

19. **Q:** What is a heatmap in monitoring?
    **A:** A visualization of metric distribution over time (e.g., latency heatmap showing P50/P95/P99).

20. **Q:** How do you handle monitoring during a massive traffic spike?
    **A:** Ensure monitoring system can scale (Prometheus Federation/Cortex/Thanos). Use adaptive scrape intervals. Prioritize critical metrics.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Design a metrics system that handles 10M unique time series.
    **A:** Prometheus with Thanos/Cortex for horizontal scaling. Shard by tenant or service. Downsampling: aggregate old data. Use streaming (Kafka) for real-time metrics.

2. **Q:** How do you implement multi-dimensional alerting without noise?
    **A:** Use alert aggregation: group similar alerts. Dependencies: suppress alerts for downstream if upstream is failing. Silence during maintenance windows. Flapping detection.

3. **Q:** Design a system for real-time anomaly detection on metrics.
    **A:** Baseline: moving average + standard deviation over 7-day window. Anomaly: current value > baseline + 3*stddev. ML model trained on historical patterns. Alert on anomaly, not raw threshold.

4. **Q:** How do you instrument a Java application with minimal overhead?
    **A:** Bytecode instrumentation (OpenTelemetry Java agent). Use precise @Timed annotations, not AOP on every method. Batch metric export. Use ring buffers (LMAX Disruptor).

5. **Q:** Design a distributed metrics aggregation system.
    **A:** Local aggregation per instance (HDR Histogram). Push to central aggregator (Kafka). Downsample per time bucket. Query: distributed query with merge.

6. **Q:** How do you monitor 95th percentile latency accurately?
    **A:** Use HDR Histogram with configured precision (2 significant digits). Avoid computing percentiles from averages. Use enough buckets for resolution. Consider T-Digest for merging across instances.

7. **Q:** Design a system for metrics-based auto-scaling.
    **A:** Horizontal Pod Autoscaler (K8s HPA) with custom metrics. Metrics: request rate / pod, latency, queue depth. Scale out when latency > SLO. Scale in after cooldown period.

8. **Q:** How do you monitor eventual consistency lag?
    **A:** Track timestamp of last event replicated. Compare to local timestamp. Metric: replication_lag_seconds. Alert if lag > threshold.

9. **Q:** Design a cost-aware metric retention policy.
    **A:** Raw metrics: 7 days (high precision). Downsampled: 30 days (1 min resolution). Aggregated: 1 year (1 hour resolution). Archive: S3. Budget: allocate storage per service/team.

10. **Q:** How do you perform capacity planning using metrics?
    **A:** Track resource utilization over time. Model growth rate (linear/exponential). Identify saturation points. Predict: "at current growth, we'll need 20% more DB capacity in 3 months."

### System Design

11. **Q:** Design a monitoring system for a Kubernetes-based microservices platform.
    **A:** Prometheus Operator with ServiceMonitors. Node exporter, kube-state-metrics, cAdvisor. Grafana dashboards per namespace/service. Alertmanager for routing.

12. **Q:** Design an SLA monitoring system for 50+ services.
    **A:** SLI per service: latency, error rate, uptime. Prometheus recording rules for SLO compliance (30d window). Error budget dashboard. Burn rate alerts.

13. **Q:** Design a distributed health check system.
    **A:** Each service exposes /health (liveness, readiness, deep). Service mesh health checks. External synthetic checks (pingdom). Central health dashboard.

14. **Q:** Design a real-time business metrics dashboard.
    **A:** Kafka -> Flink (aggregation) -> Redis (current values) -> WebSocket (real-time updates). Metrics: active users, revenue, order rate, conversion funnel.

15. **Q:** Design a system for monitoring database query performance.
    **A:** pg_stat_statements (PostgreSQL) or Performance Schema (MySQL). Export query stats to Prometheus. Dashboard: slowest queries, most frequent, temporary file usage.

16. **Q:** Design a multi-cloud monitoring system.
    **A:** Cloud-agnostic collectors (Telegraf). Central Prometheus with remote write. Metrics tagged with cloud provider/region. Grafana multi-cloud dashboards.

17. **Q:** Design a system for monitoring third-party API dependencies.
    **A:** Proxy all external calls through a monitoring client. Metrics: latency, error rate, status code distribution. Dashboard: SLA compliance per API. Alerts: API degraded/down.

18. **Q:** Design a metrics system for IoT devices.
    **A:** MQTT for device metrics reporting. Kafka ingestion. Time-series DB (InfluxDB/TimescaleDB). Downsample by device group. Gauge: battery, signal. Counter: messages, errors.

19. **Q:** Design a system for monitoring financial transactions.
    **A:** Counters: transactions, volume, refunds. Timers: processing latency per payment method. Gauges: pending settlement amounts. Alerts: unusual patterns (decline rate spike, fraud indicators).

20. **Q:** Design a system that correlates metric anomalies with code deployments.
    **A:** Deployment info (from CI/CD) stored in database with timestamp + metadata. When metric anomaly detected, query recent deployments. Dashboard: deployment timeline overlaid on metrics.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a unified observability platform that ingests logs, metrics, and traces at 10TB/day.
    **A:** All data types in a single storage backend (e.g., Grafana Loki for logs + metrics + traces, or Elastic Observability). Common labels (service, environment, cluster). Cost: hot (7d SSD), warm (30d HDD), cold (1y S3). Query across signals.

2. **Q:** How do you implement a monitoring system that survives the monitored system failing?
    **A:** Separate monitoring infrastructure (different AWS account/VPC). Active monitoring from multiple locations. Redundant Prometheus instances. Air-gapped monitoring for critical systems.

3. **Q:** Design a self-adapting alerting system that adjusts thresholds based on patterns.
    **A:** ML-based threshold detection. Baseline: rolling window of 7 days. Seasonal adjustment: compare to same time/day last week. Anomaly detection: 3-sigma or MAD (Median Absolute Deviation). Suppress known patterns.

4. **Q:** How do you design monitoring for a system with 99.999% uptime requirement?
    **A:** Every component monitored (no blind spots). Synthetic probes from multiple locations. Canary metrics comparison. Health checks every 10s. Alerts within 30s. Automated remediation (circuit breaker, restart, failover).

5. **Q:** Design a cost-optimized monitoring strategy for a startup vs an enterprise.
    **A:** Startup: Prometheus + Grafana (OSS), basic metrics, critical alerts only. Enterprise: Thanos/Cortex, multi-cluster, long retention, SLO tracking, external probes, APM integration.

6. **Q:** How do you design monitoring for a serverless/FaaS architecture?
    **A:** Distributed tracing as primary observability. CloudWatch/Cloud Monitoring metrics. Custom metrics via metric API (no long-lived agent). Business metrics in application logs.

7. **Q:** Design an approach to reduce alert fatigue by 90%.
    **A:** Alert on SLO burn rate (not raw metric). Aggregate related alerts (dedup by root cause). Silence known issues (maintenance windows, known bugs). Tiered alerting: P1 escalates, P5 goes to dashboard only.

8. **Q:** How do you measure and improve the MTTR using metrics from the monitoring system?
    **A:** Metric: time from alert to mitigation. Tag each incident with: service, root cause type, mitigation action. Dashboard: MTTR trend, top causes, resolve efficiency per team. Target: reduce MTTR by X% quarterly.

9. **Q:** Design a chaos engineering monitoring validation system.
    **A:** During chaos experiment, verify monitoring detects the injected failure. Metrics: alert latency, false negatives (uncaught failures), false positives (alert without cause). Score: monitoring coverage percentage.

10. **Q:** How do you design a monitoring system for a zero-downtime deployment strategy?
    **A:** Blue/green or canary metrics comparison. Deploy to canary, compare error rate / latency / traffic. Automated rollback if canary metrics deviate. Deployment dashboard: metric delta.

## 13. Debugging & Troubleshooting

### Common Monitoring Issues

**Issue: Missing metrics in Prometheus**
- Check /actuator/prometheus endpoint accessibility.
- Check Prometheus scrape configuration.
- Check target up status in Prometheus targets.
- Check for firewall/network issues.

**Issue: High cardinality causing Prometheus memory issues**
- Identify high-cardinality labels: `top_metrics_by_cardinality()`.
- Reduce or remove problematic tags.
- Use recording rules for aggregation.

**Issue: Alerts not firing**
- Check alert rule expression: does it return data?
- Check alert state in Prometheus.
- Check Alertmanager configuration.
- Check silenced/route configuration.

**Issue: Grafana dashboard not loading**
- Check Prometheus datasource connectivity.
- Check query timeout.
- Check dashboard JSON for errors.

## 14. Comparison Section

### Monitoring Solutions

| Solution | Type | Storage | Query | Scaling | Cost |
|----------|------|---------|-------|---------|------|
| Prometheus | Pull | TSDB | PromQL | Federation/Thanos | Free |
| Datadog | Push | Cloud | Custom | SaaS | $$$ |
| New Relic | Agent | Cloud | NRQL | SaaS | $$$ |
| Grafana Cloud | Mixed | Cloud | PromQL/LogQL | SaaS | $$ |
| AWS CloudWatch | Agent | Cloud | CloudWatch Insights | AWS | $$ |

### Counter vs Gauge vs Histogram vs Summary

| Type | Use Case | Operations | Percentile |
|------|----------|------------|------------|
| Counter | Cumulative count | Increment | No |
| Gauge | Instant value | Set | No |
| Histogram | Observation distribution | Record | Yes (from buckets) |
| Summary | Distribution with quantiles | Record | Yes (server-side) |

### Push vs Pull Monitoring

| Aspect | Pull (Prometheus) | Push (StatsD/Datadog) |
|--------|------------------|----------------------|
| Discovery | Service discovery | Agent configuration |
| Scalability | Scrape from each target | Aggregation needed |
| Reliability | Survives target restarts | Can lose data on crash |
| Firewall | Needs access to targets | Agent pushes out |
| Best for | Long-lived services | Ephemeral workloads |

## 15. Revision Notes

### Quick Recap
- **Metrics**: Numeric time-series data.
- **Four Golden Signals**: Latency, Traffic, Errors, Saturation.
- **RED**: Rate, Errors, Duration (services).
- **USE**: Utilization, Saturation, Errors (resources).
- **Micrometer**: Metrics facade for Spring Boot.
- **Prometheus**: Time-series database + query language (PromQL).
- **Grafana**: Dashboard visualization.
- **SLO/SLI/SLA**: Target, indicator, agreement.
- **Alerting**: Prometheus alert rules -> Alertmanager -> PagerDuty.

### Key Metrics to Monitor
- Application: Request rate, latency (P50/P95/P99), error rate.
- System: CPU, memory, disk, network.
- JVM: Heap usage, GC pause duration, thread count.
- Database: Query latency, connections, cache hit ratio.
- Queue: Consumer lag, message rate, queue depth.
- Business: Orders, revenue, users, conversion rate.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|               METRICS AND MONITORING CHEAT SHEET                   |
+-------------------------------------------------------------------+
| GOLDEN SIGNALS | DESCRIPTION                         | EXAMPLE     |
+----------------+-------------------------------------+-------------+
| Latency        | Time to serve request               | P95 < 200ms |
| Traffic        | Demand on system                    | 1000 req/s  |
| Errors         | Failed requests                     | < 1%        |
| Saturation     | How full the system is              | CPU < 70%   |
+----------------+-------------------------------------+-------------+
| MICROMETER TYPES                                                |
+-------------------------------------------------------------------+
| Counter        | Incrementing count (total requests)               |
| Gauge          | Up/down value (active connections)                |
| Timer          | Latency + count (request duration)                |
| Distribution-  | Distribution of values (response size)            |
| Summary        |                                                      |
+-------------------------------------------------------------------+
| SPRING BOOT ACTUATOR ENDPOINTS                                     |
+-------------------------------------------------------------------+
| /actuator/health          | Combined health status                  |
| /actuator/metrics         | List available metrics                 |
| /actuator/metrics/{name}  | Specific metric details                |
| /actuator/prometheus      | Prometheus-formatted metrics           |
| /actuator/info            | Application info                       |
+-------------------------------------------------------------------+
| PROMQL QUERY EXAMPLES                                              |
+-------------------------------------------------------------------+
| Rate of requests:                                                   |
|   rate(http_server_requests_seconds_count[5m])                      |
| P95 latency:                                                        |
|   histogram_quantile(0.95,                                          |
|     rate(http_server_requests_seconds_bucket[5m]))                  |
| Error rate:                                                         |
|   sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))  |
|   / sum(rate(http_server_requests_seconds_count[5m]))               |
| Availability:                                                       |
|   up{job="spring-boot-apps"}                                        |
+-------------------------------------------------------------------+
| COMMON ALERTS                                                       |
+-------------------------------------------------------------------+
| [Critical] Error rate > 5% for 5min                                |
| [Warning]  P95 latency > 2s for 10min                              |
| [Critical] Service down (up == 0) for 1min                         |
| [Warning]  CPU > 90% for 10min                                     |
| [Warning]  Heap > 90% for 10min                                    |
| [Critical] Disk > 95%                                               |
+-------------------------------------------------------------------+
```
