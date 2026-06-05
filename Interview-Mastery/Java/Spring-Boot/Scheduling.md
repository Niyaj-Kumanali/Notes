# Spring Boot Scheduling

---

## 1. Executive Summary

Spring Boot provides task scheduling via `@Scheduled` and `@EnableScheduling`. It supports cron expressions, fixed rate, fixed delay, and initial delay.

### Cron Expression Format
`second minute hour day-of-month month day-of-week`

| Expression | Meaning |
|-----------|---------|
| `0 0 * * * *` | Every hour |
| `0 0 8 * * MON-FRI` | 8 AM weekdays |
| `0 0/15 * * * *` | Every 15 minutes |
| `0 0 2 * * ?` | Daily at 2 AM |
| `0 0 0 1 1 ?` | Every Jan 1 midnight |

---

## 2. Core Theory

### Scheduling Types

```java
@Component
@EnableScheduling
public class ScheduledTasks {
    
    @Scheduled(fixedRate = 5000) // Every 5 seconds (regardless of execution time)
    public void runEvery5Seconds() { /* ... */ }
    
    @Scheduled(fixedDelay = 5000) // 5 seconds after previous execution finishes
    public void run5sAfterPrevious() { /* ... */ }
    
    @Scheduled(initialDelay = 10000, fixedRate = 60000) // 10s initial, then every 60s
    public void delayedStart() { /* ... */ }
    
    @Scheduled(cron = "0 0 2 * * ?") // Daily at 2 AM
    public void dailyReport() { /* ... */ }
}
```

### Thread Pool Configuration

```java
@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {
    
    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setThreadNamePrefix("scheduled-task-");
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(30);
        scheduler.initialize();
        registrar.setTaskScheduler(scheduler);
    }
}
```

---

## 3. Common Mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Fixed rate without thread pool | Tasks queue up if previous overruns | Use fixedDelay or increase pool |
| 2 | Cron at same second for many tasks | Thundering herd | Stagger cron times |
| 3 | No error handling | Task silently fails after first exception | try-catch in task |
| 4 | Forgetting @EnableScheduling | Nothing runs | Add annotation to config |
| 5 | Long-running tasks block pool | Other scheduled tasks delayed | Dedicated thread pool or async |
| 6 | Tasks not idempotent | Duplicate execution | Idempotency check |

---

## 4. Cheat Sheet

```
═══ SCHEDULING ════════════════════════════════════════════════

┌─ ANNOTATIONS ──────────────────────────────────────────────┐
│ @EnableScheduling — on @Configuration class                 │
│ @Scheduled(fixedRate=5000) — every 5s                      │
│ @Scheduled(fixedDelay=5000) — 5s after previous ends      │
│ @Scheduled(cron="0 0 2 * * ?") — cron expression           │
│ @Scheduled(initialDelay=10000) — delay before first run    │
└─────────────────────────────────────────────────────────────┘

┌─ CONFIGURATION ────────────────────────────────────────────┐
│ ThreadPoolTaskScheduler — configure pool size              │
│ SchedulingConfigurer — programmatic configuration          │
│ poolSize=1 (default) — only one thread                     │
│ Increase for concurrent scheduled tasks                    │
└─────────────────────────────────────────────────────────────┘

┌─ BEST PRACTICES ───────────────────────────────────────────┐
│ • Always wrap in try-catch to prevent silent failures       │
│ • Use fixedDelay for non-overlapping tasks                  │
│ • Configure thread pool for concurrent tasks                │
│ • Make tasks idempotent                                     │
│ • Add logging at start and end                              │
│ • Consider distributed locks for clustered deployment      │
└─────────────────────────────────────────────────────────────┘
```
