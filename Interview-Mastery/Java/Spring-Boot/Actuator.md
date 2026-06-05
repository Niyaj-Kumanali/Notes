# Spring Boot Actuator

---

## 1. Executive Summary

Spring Boot Actuator provides production-ready monitoring and management endpoints for Spring Boot applications. It exposes operational information via HTTP or JMX.

---

## 2. Core Theory

### Common Endpoints

| Endpoint | Path | What It Shows |
|----------|------|---------------|
| Health | `/actuator/health` | Application health status |
| Info | `/actuator/info` | Custom application info |
| Metrics | `/actuator/metrics` | JVM, system, custom metrics |
| Prometheus | `/actuator/prometheus` | Prometheus-format metrics |
| Loggers | `/actuator/loggers` | View/change log levels at runtime |
| Threaddump | `/actuator/threaddump` | Thread stack dump |
| Heapdump | `/actuator/heapdump` | JVM heap dump |
| Env | `/actuator/env` | Environment properties |
| Configprops | `/actuator/configprops` | @ConfigurationProperties |
| Mappings | `/actuator/mappings` | Request mappings |
| Beans | `/actuator/beans` | All Spring beans |
| Scheduledtasks | `/actuator/scheduledtasks` | Scheduled tasks |
| Caches | `/actuator/caches` | Cache information |
| Health Path | `/actuator/health/{component}` | Specific health component |

### Configuration

```yaml
# application.yml
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

### Custom Health Indicator

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

### Custom Metric

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
        return orderProcessingTime.record(() -> action.get());
    }
}
```

---

## 3. Cheat Sheet

```
═══ SPRING BOOT ACTUATOR ══════════════════════════════════════

┌─ ESSENTIAL ENDPOINTS ─────────────────────────────────────┐
│ /actuator/health         — Health check (readiness+liveness)│
│ /actuator/info           — Application info                │
│ /actuator/metrics        — Application metrics             │
│ /actuator/prometheus     — Prometheus scrape endpoint       │
│ /actuator/loggers        — Change log level at runtime      │
│ /actuator/env            — Environment properties          │
└─────────────────────────────────────────────────────────────┘

┌─ CONFIGURATION ────────────────────────────────────────────┐
│ management.endpoints.web.exposure.include — which endpoints │
│ management.endpoint.health.show-details  — details level   │
│ management.info.env.enabled              — info from env   │
│ management.metrics.tags.*                — common metrics  │
└─────────────────────────────────────────────────────────────┘

┌─ CUSTOMIZATION ────────────────────────────────────────────┐
│ HealthIndicator        — custom health check               │
│ MeterBinder            — custom metrics                    │
│ InfoContributor        — custom info                       │
│ @ReadOperation         — custom Actuator endpoint          │
└─────────────────────────────────────────────────────────────┘

┌─ BEST PRACTICES ───────────────────────────────────────────┐
│ • Expose only needed endpoints in production                │
│ • Secure actuator endpoints (Spring Security)               │
│ • Set health show-details to when-authorized               │
│ • Create custom health checks for downstream services       │
│ • Export to Prometheus for production monitoring            │
│ • Use metrics for tracking business KPIs                   │
└─────────────────────────────────────────────────────────────┘
```
