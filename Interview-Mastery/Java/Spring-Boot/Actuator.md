# Spring Boot Actuator

---

## Overview

- **Definition:** Spring Boot Actuator provides production-ready monitoring and management endpoints for Spring Boot applications. It exposes operational information about your running application — health, metrics, environment properties, thread dumps, and more — via HTTP or JMX.
- **Why It Exists:** Production applications need visibility into their internal state — health, metrics, configuration, and logging — without requiring code changes or restarts. Actuator standardizes this operational data access across all Spring Boot applications.
- **Key Concepts:**
  - **Common Endpoints:**
    - **`/actuator/health`** — Application health status (UP/DOWN). Used by container orchestrators (Kubernetes, Docker) for liveness and readiness probes to determine pod health.
    - **`/actuator/info`** — Custom application information (version, build time, contact info). Exposes metadata defined in `application.properties` or `build-info.properties`.
    - **`/actuator/metrics`** — JVM and application metrics (memory, CPU, thread count, GC pauses). Integrates with Micrometer for detailed metric collection and analysis.
    - **`/actuator/prometheus`** — Metrics in Prometheus format for scraping and alerting. Requires `micrometer-registry-prometheus` on the classpath.
    - **`/actuator/loggers`** — View and change log levels at runtime without restart. Invaluable for debugging production issues on the fly.
    - **`/actuator/threaddump`** — Thread stack dump for diagnosing deadlocks and thread contention. Captures the state of every thread in the JVM at the time of the request.
    - **`/actuator/heapdump`** — JVM heap dump for memory analysis. Useful for finding memory leaks but exposes all data in memory including PII.
    - **`/actuator/env`** — Environment properties from all property sources. Shows configuration values including secrets — must be secured.
    - **`/actuator/configprops`** — All `@ConfigurationProperties` beans with their current values. Useful for auditing configuration at runtime.
    - **`/actuator/mappings`** — All request mappings in the application. Shows which URLs map to which controller methods.
    - **`/actuator/beans`** — All Spring beans in the container. Shows bean names, scopes, types, and dependencies.
    - **`/actuator/scheduledtasks`** — All scheduled tasks and their configuration. Displays cron expressions, fixed rates, and fixed delays.
  - **Configuration:**
    ```yaml
    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus,loggers,env
            exclude: threaddump,heapdump  # sensitive
          base-path: /actuator
      endpoint:
        health:
          show-details: when-authorized
          show-components: when-authorized
      info:
        env:
          enabled: true
      metrics:
        tags:
          application: ${spring.application.name}
    ```

---

## Core Concepts

### Custom Health Indicator

- Monitor the health of specific dependencies:
  ```java
  @Component
  public class DatabaseHealthIndicator implements HealthIndicator {
      private final DataSource dataSource;

      @Override
      public Health health() {
          try (Connection conn = dataSource.getConnection()) {
              if (!conn.isValid(2)) {
                  return Health.down()
                      .withDetail("database", "Connection validation failed")
                      .build();
              }
              return Health.up()
                  .withDetail("database", "Connected")
                  .withDetail("validationQuery", conn.getMetaData().getURL())
                  .build();
          } catch (Exception e) {
              return Health.down(e)
                  .withDetail("database", "Connection failed: " + e.getMessage())
                  .build();
          }
      }
  }
  ```

### Custom Metrics with Micrometer

- Track business-specific metrics:
  ```java
  @Component
  public class OrderMetrics {
      private final Counter orderCreated;
      private final Counter paymentFailed;
      private final Timer orderProcessingTime;

      public OrderMetrics(MeterRegistry registry) {
          this.orderCreated = registry.counter("orders.created");
          this.paymentFailed = registry.counter("orders.payment.failed");
          this.orderProcessingTime = registry.timer("orders.processing.time");
      }

      public void recordOrderCreated() {
          orderCreated.increment();
      }

      public void recordPaymentFailed() {
          paymentFailed.increment();
      }

      public <T> T measureProcessingTime(Supplier<T> action) {
          return orderProcessingTime.record(action);
      }
  }
  ```

### Custom Actuator Endpoint

- Create custom management endpoints:
  ```java
  @Component
  @Endpoint(id = "feature-flags")
  public class FeatureFlagEndpoint {

      @ReadOperation
      public Map<String, Boolean> getFeatureFlags() {
          return Map.of(
              "new-checkout", true,
              "recommendations", false,
              "dark-mode", true
          );
      }

      @WriteOperation
      public void toggleFeature(@Selector String flag, boolean enabled) {
          // Toggle feature flag at runtime
      }
  }
  ```

---

## Common Mistakes

- **Exposing sensitive endpoints in production**
  - **Why it looks correct:** `include: "*"` is the easiest configuration and works perfectly in development — the data leakage is invisible until a security audit discovers the exposed endpoints.
  - **Fix:** Always restrict exposure to only the endpoints your operations team actually needs. Endpoints like `heapdump`, `env`, and `threaddump` expose sensitive information including PII and secrets.

- **Not securing actuator endpoints**
  - **Why it looks correct:** The endpoints are accessible to developers for debugging — the security risk only becomes apparent when the application is deployed to production with the same open configuration.
  - **Fix:** Protect actuator endpoints with Spring Security or network-level restrictions. Unsecured endpoints can leak sensitive data about your application internals.

- **Leaving `show-details: always`**
  - **Why it looks correct:** The ops team needs details to diagnose issues, and `always` is the most convenient setting — the information is shared freely until a competitor or attacker uses it for reconnaissance.
  - **Fix:** Use `when-authorized` to restrict detailed health data to authenticated users with specific roles.

- **Not creating custom health indicators**
  - **Why it looks correct:** The default `/health` returns UP and the orchestrator keeps the pod running — the false positive only matters when the downstream dependency fails and the application stops working while reporting healthy.
  - **Fix:** Add custom health checks for all critical external services including third-party APIs, message queues, and custom services. The default health check only covers basic Spring components like DataSource.

- **Not exposing metrics to Prometheus**
  - **Why it looks correct:** The application works without Prometheus — the lack of monitoring only becomes obvious during a production incident when there are no charts or alerts to analyze.
  - **Fix:** Add `micrometer-registry-prometheus` and include `prometheus` in exposed endpoints to set up proper monitoring dashboards and alerting rules for production operations.

- **No metrics tags**
  - **Why it looks correct:** Metrics render correctly in the `/actuator/metrics` endpoint — the missing tags only matter when you try to query Prometheus for `average_response_time by application` and get a combined number for all services.
  - **Fix:** Apply consistent tags (application name, environment, instance) so metrics from multiple instances and environments can be correlated or filtered in dashboards.

---

## Real-World Scenarios

### Scenario 1: Kubernetes Health Probes with Custom Readiness Check

- A payment processing service runs on Kubernetes. The default `/actuator/health` shows `UP` even when the service cannot process payments because a downstream gateway is down. Kubernetes continues to route traffic to the unhealthy pod.
  ```java
  @Component
  public class PaymentGatewayHealthIndicator implements HealthIndicator {
      private final PaymentGatewayClient gatewayClient;

      @Override
      public Health health() {
          try {
              boolean reachable = gatewayClient.ping();
              if (!reachable) {
                  return Health.down()
                      .withDetail("gateway", "Payment gateway unreachable")
                      .build();
              }
              return Health.up()
                  .withDetail("gateway", "Connected")
                  .withDetail("latency", gatewayClient.getLatencyMs() + "ms")
                  .build();
          } catch (Exception e) {
              return Health.down(e)
                  .withDetail("gateway", "Health check failed")
                  .build();
          }
      }
  }

  // Separate liveness and readiness probes
  // Liveness: is the app alive? (basic health)
  // Readiness: can it serve traffic? (includes downstream dependencies)
  @Component
  public class ReadinessState {
      private final HealthAggregator healthAggregator;

      @ReadinessCheck
      public Health checkReadiness() {
          // Aggregate all critical dependencies
      }
  }
  ```

### Scenario 2: Runtime Log Level Change for Debugging Production Issues

- A production incident causes errors in the `com.example.payment` package. To debug without restarting, an operator changes the log level to DEBUG at runtime via the Actuator endpoint.
  ```bash
  # View current log levels
  GET /actuator/loggers

  # Set DEBUG for a specific package
  POST /actuator/loggers/com.example.payment
  Content-Type: application/json

  {"configuredLevel": "DEBUG"}
  ```
- The logs immediately show detailed debug output. After debugging, the level is reset to INFO without any restart or code change. This is secured via Spring Security so only operators with `ADMIN` role can access it.

### Scenario 3: Custom Business Metrics for Monitoring an E-Commerce Platform

- The operations team needs real-time visibility into order volumes, payment failures, and processing times. They use Prometheus + Grafana for dashboards.
  ```java
  @Component
  public class OrderMetrics {
      private final Counter orderCreatedCounter;
      private final Counter paymentFailedCounter;
      private final Timer orderProcessingTimer;
      private final DistributionSummary orderValueSummary;

      public OrderMetrics(MeterRegistry registry) {
          this.orderCreatedCounter = Counter.builder("orders.created.total")
              .description("Total orders created").register(registry);
          this.paymentFailedCounter = Counter.builder("orders.payment.failed")
              .description("Failed payment attempts")
              .tag("cause", "insufficient_funds")
              .register(registry);
          this.orderProcessingTimer = Timer.builder("orders.processing.duration")
              .description("Order processing time")
              .publishPercentiles(0.5, 0.95, 0.99)
              .register(registry);
          this.orderValueSummary = DistributionSummary.builder("orders.value")
              .description("Distribution of order values")
              .baseUnit("USD")
              .register(registry);
      }
  }
  ```

---

## Use Cases

- Actuator endpoints provide runtime visibility into Spring Boot applications. These use cases show how to use health checks, metrics, logging, and configuration exposure for production operations.

- **Kubernetes health probes (liveness + readiness)** — A container orchestrator needs to know whether a pod is alive (liveness) and whether it can serve traffic (readiness, including downstream dependency health).
  - Configure `management.endpoint.health.group.readiness.include=db,redis,paymentGateway` and use separate probe paths: `/actuator/health/liveness` and `/actuator/health/readiness`. Implement custom `HealthIndicator` beans for each dependency.
  - **Avoid when:** Your app has no external dependencies — the default `health` endpoint with no custom indicators is sufficient.

- **Runtime log level debugging** — A production incident causes errors in a specific package. Changing the log level to DEBUG normally requires a restart and redeploy.
  - Use `POST /actuator/loggers/com.example.payment` with body `{"configuredLevel": "DEBUG"}`. Logs immediately show debug output. Reset to `INFO` when done. Secure this endpoint with `ADMIN` role.
  - **Avoid when:** Logs contain sensitive data — restrict access via `management.endpoint.loggers.roles=ADMIN` and audit all log-level changes.

- **Custom business metrics for monitoring dashboards** — Operations needs real-time visibility into order volumes, payment failures, and processing latencies for Grafana dashboards.
  - Inject `MeterRegistry` and create `Counter`, `Timer`, and `DistributionSummary` meters. Expose via `/actuator/prometheus` using `micrometer-registry-prometheus`. Configure Prometheus to scrape the endpoint.
  - **Avoid when:** You only need JVM-level metrics (heap, threads, GC) — Actuator's built-in metrics are sufficient; no custom code needed.

- **Runtime environment inspection for debugging** — A configuration issue in production (wrong database URL, feature flag not set) needs diagnosis without accessing the server directly.
  - Use `GET /actuator/env` to view all property sources and their values. Use `GET /actuator/configprops` to see `@ConfigurationProperties` beans and their current values.
  - **Avoid when:** The endpoint exposes secrets — use `management.endpoint.env.keys-to-sanitize=*password*,*secret*,*key*` to redact sensitive values.

- **Thread dump and heap dump for performance analysis** — A production service experiences thread contention or memory leaks. Restarting would destroy the evidence.
  - Use `GET /actuator/threaddump` for a thread stack dump to diagnose deadlocks. Use `GET /actuator/heapdump` to download a heap dump for OOM analysis with Eclipse MAT.
  - **Avoid when:** The `/actuator/heapdump` exposes PII — restrict access to operators and never enable it in environments with sensitive user data without RBAC.

---

## Scenario-Based Questions

1. **Q: Your Kubernetes cluster reports the pod as healthy (`/actuator/health` returns `UP`), but the application returns 503 errors for all API calls because the Redis cache cluster is down. How do you make the health check reflect actual readiness?**
   - Create a comprehensive health indicator that aggregates all critical dependencies:
     ```java
     @Component
     public class RedisHealthIndicator implements HealthIndicator {
         private final RedisTemplate<String, String> redisTemplate;
         @Override
         public Health health() {
             try {
                 String pong = redisTemplate.getConnectionFactory().getConnection().ping();
                 return "PONG".equals(pong)
                     ? Health.up().build()
                     : Health.down().withDetail("redis", "ping failed").build();
             } catch (Exception e) {
                 return Health.down(e).build();
             }
         }
     }
     ```
   - Then configure Kubernetes readiness probe to use `/actuator/health/readiness` and ensure the custom `RedisHealthIndicator` is included. Set `management.endpoint.health.show-details=always` for the readiness probe.

2. **Q: You expose `/actuator/heapdump` to diagnose a memory leak. A security audit flags this as a critical vulnerability because anyone can download the heap dump containing customer PII. How do you fix this?**
   - Three-layer defense: (a) Disable the heapdump endpoint: `management.endpoint.heapdump.enabled=false`. (b) If you must keep it, secure with Spring Security — only `ADMIN` role can access it. (c) Run actuator on a separate port exposed only to internal networks:
     ```yaml
     management:
       server:
         port: 8081
       endpoints:
         web:
           exposure:
             include: health,info,metrics
             exclude: heapdump,env,threaddump
     ```
   - The separate port should not be exposed to the public internet.
   - **Interview follow-up:** The candidate proposed a separate management port on 8081. A developer runs the application locally using `--server.port=8080` and a shared IDE run config that was configured before the management port was added. The developer is unaware of the management port change and spends 2 hours debugging why the application appears to start but never becomes healthy (they're checking port 8080, not 8081). How would you design the configuration so that the separate management port is only enabled in production profiles, and what logging or startup banner would you add to make the port separation visible to developers?

3. **Q: Your application has 50+ metrics. Developers keep adding new ones inconsistently — some use dots, some use underscores; some tag with `application`, some don't. Prometheus queries become confusing. How do you standardize?**
   - Enforce naming conventions with a custom `MeterFilter`:
     ```java
     @Bean
     public MeterFilter standardizeMetrics() {
         return new MeterFilter() {
             @Override
             public MeterFilterReply accept(Meter.Id id) {
                 String name = id.getName();
                 if (!name.startsWith("app.")) {
                     return MeterFilterReply.DENY;
                 }
                 return MeterFilterReply.ACCEPT;
             }
             @Override
             public Meter.Id map(Meter.Id id) {
                 return id.withName(id.getName().replace(" ", "_").toLowerCase())
                     .withTag(Tag.of("application", "${spring.application.name}"));
             }
         };
     }
     ```
   - Add a CI check that fails if metrics don't follow the convention. Document common tags: `application`, `environment`, `instance`.

4. **Q: You need to monitor the 95th percentile response time of a specific external API call. How do you implement this with Actuator/Micrometer?**
   - Use a `Timer` with percentile publishing:
     ```java
     @Component
     public class ExternalApiMetrics {
         private final Timer apiTimer;
         public ExternalApiMetrics(MeterRegistry registry) {
             this.apiTimer = Timer.builder("external.api.duration")
                 .description("External API call duration")
                 .tag("endpoint", "/api/v1/orders")
                 .publishPercentiles(0.5, 0.95, 0.99)
                 .sla(Duration.ofMillis(100), Duration.ofMillis(500), Duration.ofSeconds(1))
                 .register(registry);
         }
         public <T> T measure(Supplier<T> call) {
             return apiTimer.record(call);
         }
     }
     ```
   - The `publishPercentiles` enables percentile calculation on the client side. The `sla` boundaries allow counting how many calls fall into each latency bucket.

5. **Q: You upgrade Spring Boot from 2.x to 3.x. The `/actuator/health` endpoint now returns only `{"status": "UP"}` without component details. The ops team relied on those details. What changed?**
   - Spring Boot 3.x changed the default for `management.endpoint.health.show-details` from `always` to `when-authorized`. Set it explicitly:
     ```yaml
     management:
       endpoint:
         health:
           show-details: always
           show-components: always
     ```
   - Or better, use `when-authorized` and configure the roles that can see details:
     ```yaml
     management:
       endpoint:
         health:
           show-details: when-authorized
           roles: ADMIN, OPERATOR
     ```

6. **Q: Your `/actuator/prometheus` endpoint returns data, but Prometheus reports "target down" for your application. What could be wrong?**
   - The Prometheus server likely cannot reach the endpoint. Check: (a) Network connectivity — is there a firewall blocking the scrape port? (b) Prometheus configuration — is the `scrape_configs` target URL correct? (c) Scrape timeout — if the application is slow to respond, Prometheus may timeout. Increase `scrape_timeout` in Prometheus config. (d) TLS — if the app requires HTTPS but Prometheus scrapes HTTP. (e) Authentication — if the endpoint requires auth headers. Add `basic_auth` to the Prometheus scrape config.

7. **Q: You deploy a new version and notice that `jvm.memory.used` keeps increasing but never decreases. The heap dump shows normal object sizes. What might be the actual memory leak?**
   - The metric `jvm.memory.used` includes all memory regions, not just heap. The leak might be in: (a) Metaspace — classloader leak (undeployed applications in containers), (b) Direct memory — direct ByteBuffer allocations not released, (c) Native memory — JNI calls, thread stacks, or native libraries. Use `jvm.buffer.memory.used` to check direct and mapped buffers. Use `jvm.memory.nonheap.used` for metaspace. If heap metrics are stable but total memory grows, the leak is likely off-heap.

8. **Q: Your team has 20 microservices. Each exposes actuator endpoints. The ops team cannot keep track of which endpoints each service exposes. How do you standardize?**
   - Create a shared library that all microservices use, which configures actuator consistently:
     ```java
     @Configuration
     public class StandardActuatorConfig {
         @Bean
         public MeterFilter commonTags() {
             return MeterFilter.commonTags(
                 Tag.of("application", "${spring.application.name}"),
                 Tag.of("team", "platform"),
                 Tag.of("environment", "${spring.profiles.active}")
             );
         }
     }
     ```
   - Then use Spring Boot Admin or a service mesh (Istio, Linkerd) to aggregate metrics and health from all services in a single dashboard. Standardize endpoint exposure across all services via a shared config in a ConfigMap.

9. **Q: You configure `management.endpoints.web.exposure.include=*` for development. The build pipeline accidentally deploys this to production. What is the impact and how do you prevent it?**
   - Impact: `*` exposes all endpoints including `heapdump` (full JVM memory dump with PII), `env` (environment variables with secrets), `threaddump` (running threads), `loggers` (change log levels), `shutdown` (shut down the application). Mitigation: (a) Use separate config per profile — `management.endpoints.web.exposure.include=health,info,metrics,prometheus` for production. (b) Add a `@ConditionalOnCloudPlatform` or profile check that refuses to start with dangerous exposure in production. (c) Use ArchUnit to test that the production config does not include sensitive endpoints.
   - **Interview follow-up:** The candidate suggested ArchUnit to test production config. A year later, the team migrates from `application.properties` to a Spring Cloud Config server. The ArchUnit test reads the local `application-production.properties` file, but the effective configuration is now served remotely by the config server — the local file only contains overrides. The ArchUnit test passes because the local file uses safe endpoint exposure, but the config server delivers `include=*`, which is the actual configuration in production. How would you write a runtime check that validates the effective (not just the file-based) actuator exposure on application startup in production?

10. **Q: Your application is slow and you suspect a thread pool exhaustion. How do you use Actuator to diagnose this in production?**
    - Check multiple endpoints: (a) `/actuator/metrics/jvm.threads.live` — number of live threads. (b) `/actuator/metrics/jvm.threads.peak` — peak thread count. (c) `/actuator/metrics/tomcat.threads.current` — current Tomcat threads. (d) `/actuator/threaddump` — full thread dump to see blocked/waiting threads. Compare with configured pool sizes:
      ```bash
      # Quick diagnostic
      curl -s /actuator/metrics/jvm.threads.live | jq '.measurements[0].value'
      curl -s /actuator/threaddump | jq '.threads | map(select(.threadState == "BLOCKED")) | length'
      ```
    - If thread count is near the pool limit and many threads are BLOCKED, you have a contention issue. Check `/actuator/health` for database connection pool health.

---

## Interview Questions

- **What is Spring Boot Actuator?**
  - A: Spring Boot Actuator provides production-ready monitoring and management endpoints for Spring Boot applications. It exposes health, metrics, environment properties, thread dumps, and more via HTTP or JMX. It integrates with Micrometer for metrics and supports Prometheus, Graphite, and other monitoring systems.

- **What are the most commonly used Actuator endpoints?**
  - A: `/health` (application health), `/info` (custom app info), `/metrics` (JVM and app metrics), `/prometheus` (Prometheus-formatted metrics), `/loggers` (view/change log levels), `/threaddump` (thread dump), `/heapdump` (heap dump), `/env` (environment properties), `/beans` (Spring beans), `/mappings` (request mappings).

- **How do you secure Actuator endpoints?**
  - A: Use Spring Security to restrict access: permit `/health` and `/info` for all, require `ADMIN` role for others. Run actuator on a separate management port (`management.server.port=8081`) not exposed externally. Use `management.endpoints.web.exposure.include` to only expose necessary endpoints.

- **What is a custom HealthIndicator and when do you create one?**
  - A: A `HealthIndicator` reports the health of a specific component. Create one for external dependencies not covered by Spring Boot's auto-configured indicators — message queues, third-party APIs, custom services. Implement `HealthIndicator` and return `Health.up()` or `Health.down()` with details.

- **How do you expose custom metrics with Micrometer?**
  - A: Inject `MeterRegistry` and create counters, timers, gauges, or distribution summaries. Example:
    ```java
    Counter counter = Counter.builder("orders.created").register(registry);
    Timer timer = Timer.builder("api.latency").publishPercentiles(0.95).register(registry);
    ```
  - Metrics are automatically exposed via `/actuator/metrics` and `/actuator/prometheus`.

- **What is the difference between `/actuator/health` and `/actuator/info`?**
  - A: `/health` indicates application status (UP/DOWN/UNKNOWN) and is used by orchestrators for liveness/readiness probes. `/info` shows arbitrary application metadata (version, build time, contact info) and has no impact on operational status.

- **How do you change log levels at runtime using Actuator?**
  - A: POST to `/actuator/loggers/{package}` with body `{"configuredLevel": "DEBUG"}`. The change is immediate and persists until the application restarts. This is invaluable for debugging production issues without redeployment.

- **How do you expose Prometheus metrics from a Spring Boot application?**
  - A: Add `micrometer-registry-prometheus` dependency and include `prometheus` in `management.endpoints.web.exposure.include`. Metrics are available at `/actuator/prometheus`. Configure Prometheus to scrape this endpoint.

- **What is Micrometer's `MeterRegistry` and how does it work?**
  - A: `MeterRegistry` is the central registry for metrics. It creates and manages meters (Counter, Timer, Gauge, DistributionSummary). Micrometer can send metrics to multiple monitoring systems simultaneously (Prometheus, Datadog, Graphite, etc.) via registry binders. Spring Boot automatically configures the registry.

- **How do you create a custom Actuator endpoint?**
  - A: Annotate a class with `@Endpoint(id = "myendpoint")`. Add `@ReadOperation` for GET, `@WriteOperation` for POST, `@DeleteOperation` for DELETE methods. The endpoint automatically appears in `/actuator/myendpoint` when exposed.

---

## Developer Recommendations

- **Never expose sensitive endpoints in production**
  - Endpoints like `heapdump`, `env`, `threaddump`, and `shutdown` expose critical data. Heap dumps contain PII and secrets. The `env` endpoint shows environment variables including database passwords and API keys. Always use a whitelist approach: `include: health,info,metrics,prometheus` and explicitly exclude dangerous endpoints.

- **Run Actuator on a separate management port**
  - `management.server.port=8081` isolates management endpoints from the public API port. This makes it easier to firewall the management port and prevents accidental exposure of sensitive endpoints through reverse proxy configuration.

- **Add custom HealthIndicators for all external dependencies**
  - The default `/health` only covers Spring components (DataSource, Redis, etc.). Add health checks for your third-party API integrations, message queues, and custom services. This makes Kubernetes liveness/readiness probes actually useful.

- **Use Micrometer's `MeterFilter` to enforce naming conventions**
  - Without standardization, metrics become a mess of inconsistent names and tags. Apply filters that prefix all metrics (`app.*`), replace spaces with underscores, and add common tags (`application`, `environment`, `instance`).

- **Publish percentiles (p50, p95, p99) for latency metrics**
  - Average latency is misleading — a few slow requests can skew it. Percentiles reveal the true user experience. Use `Timer.publishPercentiles(0.5, 0.95, 0.99)` to track the median and tail latencies.

- **Secure loggers endpoint with `ADMIN` role**
  - The `/loggers` endpoint allows changing log levels at runtime. This is powerful for debugging but dangerous — a malicious actor could set `root` logger to `TRACE` and flood the disk with logs. Always restrict this to operators.

- **Use `@Timed` annotation for automatic timing**
  - Instead of manually creating Timers, use `@Timed(value = "myapp.orders.create", percentiles = {0.5, 0.95})` on controller or service methods. Spring AOP automatically records execution time. Combine with `@TimedSet` for class-level defaults.

- **Monitor GC metrics proactively**
  - Track `jvm.gc.pause` (pause time), `jvm.gc.memory.allocated` (allocation rate), and `jvm.gc.live.data.size` (tenured generation usage). A sharp increase in allocation rate or pause time is an early warning of memory problems.
