# Spring Boot Scheduling

---

## What is Scheduling?

**Spring Boot Scheduling** provides task execution capabilities using `@Scheduled` and `@EnableScheduling`. It supports **cron expressions**, **fixed rate**, **fixed delay**, and **initial delay** — enabling periodic and time-based job execution without external schedulers.

### Key Concepts:

1. **Scheduling Types**:

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

2. **Cron Expression Format**:

   The format is: `second minute hour day-of-month month day-of-week`

   | Expression | Meaning |
   |-----------|---------|
   | `0 0 * * * *` | Every hour |
   | `0 0 8 * * MON-FRI` | 8 AM weekdays |
   | `0 0/15 * * * *` | Every 15 minutes |
   | `0 0 2 * * ?` | Daily at 2 AM |
   | `0 0 0 1 1 ?` | Every Jan 1 midnight |
   | `0 0/30 9-17 * * MON-FRI` | Every 30 min during work hours weekdays |

   Special characters:
   - `*` — Any value
   - `?` — No specific value (used for day-of-month or day-of-week when the other is specified)
   - `-` — Range (e.g., `MON-FRI`)
   - `/` — Increment (e.g., `0/15` means every 15 minutes starting at 0)
   - `,` — List (e.g., `MON,WED,FRI`)
   - `L` — Last (e.g., `L` for last day of month)
   - `#` — Nth occurrence (e.g., `2#1` for first Monday)

3. **Fixed Rate vs Fixed Delay**:

   - **`fixedRate`** — The next execution starts at a fixed interval from the **start** of the previous execution. If a task takes longer than the interval, multiple executions may overlap or queue up.
   - **`fixedDelay`** — The next execution starts after a fixed delay from the **completion** of the previous execution. Tasks never overlap.

---

## Core Concepts

### 1. Thread Pool Configuration

   By default, scheduled tasks use a single-threaded executor. For concurrent tasks, configure a larger pool:

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

### 2. Dynamic Scheduling with `TaskScheduler`

   Schedule tasks programmatically at runtime:

   ```java
   @Component
   public class DynamicTaskScheduler {
       private final TaskScheduler taskScheduler;
       private final Map<String, ScheduledFuture<?>> scheduledTasks = new ConcurrentHashMap<>();

       public DynamicTaskScheduler(TaskScheduler taskScheduler) {
           this.taskScheduler = taskScheduler;
       }

       public void scheduleTask(String taskId, Runnable task, String cronExpression) {
           ScheduledFuture<?> future = taskScheduler.schedule(
               task, new CronTrigger(cronExpression));
           scheduledTasks.put(taskId, future);
       }

       public void cancelTask(String taskId) {
           ScheduledFuture<?> future = scheduledTasks.get(taskId);
           if (future != null) {
               future.cancel(false);
               scheduledTasks.remove(taskId);
           }
       }
   }
   ```

### 3. Conditional Scheduling

   Skip task execution based on conditions:

   ```java
   @Component
   public class ConditionalTasks {

       @Scheduled(cron = "0 0 2 * * ?")
       public void nightlyReport() {
           if (!featureFlags.isReportsEnabled()) {
               log.info("Reports disabled, skipping nightly task");
               return;
           }
           // Generate report
       }
   }
   ```

---

## Common Mistakes

1. **Using `fixedRate` when tasks can overrun** — If a task takes 10 seconds but `fixedRate` is 5 seconds, executions pile up or overlap. Use `fixedDelay` to prevent overlap.

2. **Cron at the same second for many tasks** — Multiple tasks trigger simultaneously, causing a thundering herd. Stagger cron times (e.g., `0 0 2 * * ?` and `0 5 2 * * ?`).

3. **No error handling** — If a scheduled task throws an unhandled exception, the task stops permanently (Spring's default scheduler does not retry). Always wrap the body in try-catch.

4. **Forgetting `@EnableScheduling`** — Without it, `@Scheduled` annotations are ignored.

5. **Long-running tasks blocking the pool** — With the default single-threaded scheduler, one long task blocks all others. Increase pool size or use `@Async` for long-running operations.

6. **Tasks not idempotent** — Scheduled tasks can be accidentally triggered multiple times (e.g., in clustered deployments). Ensure tasks are idempotent.

7. **Default single-thread pool** — Only one task runs at a time. If you have multiple `@Scheduled` methods, they may queue up behind a slow task.

---

## Real-World Scenarios

### Scenario 1: Nightly Report Generation with Idempotent Scheduling

A retail company generates nightly sales reports at 2 AM. The task must run exactly once across 10 Kubernetes pods. If the task fails, it should retry. The report takes 30-45 minutes.

```java
@Component
public class NightlyReportTask {
    private final ReportService reportService;
    private final DistributedLock distributedLock;

    @Scheduled(cron = "0 0 2 * * ?", zone = "America/New_York")
    public void generateNightlyReport() {
        String lockKey = "nightly-report-lock";
        Optional<Lock> lock = distributedLock.tryLock(lockKey, Duration.ofHours(1));
        if (lock.isEmpty()) {
            log.info("Another instance is already generating the report");
            return; // Another instance has the lock
        }
        try {
            log.info("Starting nightly report generation");
            reportService.generateReport(LocalDate.now().minusDays(1));
            log.info("Nightly report generated successfully");
        } catch (Exception e) {
            log.error("Nightly report failed", e);
            // Alert on-call — no silent failures
            alertService.sendAlert("Nightly report generation failed: " + e.getMessage());
        } finally {
            lock.get().unlock();
        }
    }
}
```

### Scenario 2: Dynamic Scheduling for User-Configurable Jobs

A SaaS platform lets users configure when they receive automated reports. User A wants daily at 8 AM, User B wants weekly on Monday at 9 AM. These cannot use `@Scheduled` annotations because the schedules are dynamic.

```java
@Component
public class DynamicJobScheduler {
    private final TaskScheduler taskScheduler;
    private final ConcurrentHashMap<String, ScheduledFuture<?>> jobs = new ConcurrentHashMap<>();

    public void scheduleUserReport(String userId, String cronExpression) {
        // Cancel existing job for this user if any
        cancelJob(userId);

        ScheduledFuture<?> future = taskScheduler.schedule(
            () -> generateAndSendReport(userId),
            new CronTrigger(cronExpression)
        );
        jobs.put(userId, future);
    }

    public void cancelJob(String userId) {
        ScheduledFuture<?> existing = jobs.remove(userId);
        if (existing != null) {
            existing.cancel(false);
        }
    }

    private void generateAndSendReport(String userId) {
        // Generate user-specific report
        log.info("Generating report for user {}", userId);
    }
}
```

### Scenario 3: Scheduled Health Check with Circuit Breaker

A microservice checks the health of a third-party API every 30 seconds. If the API is down, the task should not spam logs — it should stop checking after 3 consecutive failures and resume when the API recovers.

```java
@Component
public class HealthCheckTask {
    private final ExternalApiClient apiClient;
    private final AtomicInteger failureCount = new AtomicInteger(0);

    @Scheduled(fixedDelay = 30_000)
    public void checkExternalApiHealth() {
        if (failureCount.get() >= 3) {
            log.warn("Circuit open for external API — {} consecutive failures. Skipping check.",
                failureCount.get());
            return;
        }
        try {
            boolean healthy = apiClient.ping();
            if (healthy) {
                failureCount.set(0);
                log.info("External API is healthy");
            } else {
                int failures = failureCount.incrementAndGet();
                log.warn("External API unhealthy (failure #{})", failures);
            }
        } catch (Exception e) {
            int failures = failureCount.incrementAndGet();
            log.error("External API check failed (failure #{})", failures, e);
        }
    }
}
```

---

## Scenario-Based Questions

1. **Q: You have a `@Scheduled(fixedRate = 5000)` task that sends heartbeat signals. The task sometimes takes 10 seconds due to network latency. What happens to the execution schedule?**
   A: With `fixedRate`, a new execution starts every 5 seconds from the START of the previous execution. If a task takes 10 seconds, the next execution starts 5 seconds after the previous START (not after completion). This means tasks overlap — two or more heartbeat tasks run concurrently. If your heartbeat logic is not idempotent (e.g., it increments a counter), this causes data corruption. Fix: Use `fixedDelay` instead — the next execution starts 5 seconds after the PREVIOUS one COMPLETES. For heartbeat of fixed-rate semantics (must run at exact intervals), ensure the task never exceeds its interval, or make it idempotent.

2. **Q: Your scheduled task runs every hour and processes pending orders. After a deployment, you notice the task has not run for 8 hours. No errors in logs. What happened?**
   A: The task method threw an exception that propagated up. Spring's default scheduler (`ScheduledThreadPoolExecutor`) catches exceptions and silently stops the task — it will NOT reschedule after an exception. The fix is always wrapping the task body in try-catch:
   ```java
   @Scheduled(cron = "0 0 * * * ?")
   public void processPendingOrders() {
       try {
           orderService.processPending();
       } catch (Exception e) {
           log.error("Failed to process pending orders", e);
           alertService.sendAlert("Scheduled task failed: processPendingOrders");
       }
   }
   ```
   Add monitoring: an aspect that logs every task execution and alerts on failures.

3. **Q: Your Spring Boot application runs on 5 instances. A scheduled task that sends a daily summary email fires 5 times — one per instance. Customers get 5 identical emails. How do you solve this?**
   A: Use a distributed lock mechanism. ShedLock is the most common solution:
   ```java
   @Scheduled(cron = "0 0 8 * * ?")
   @SchedulerLock(name = "dailySummaryEmail", lockAtLeastFor = "10m", lockAtMostFor = "30m")
   public void sendDailySummary() {
       userService.sendDailySummaryEmail();
   }
   ```
   ShedLock uses a shared database table or Redis to coordinate locks. Only one instance acquires the lock and executes the task. If that instance fails, another instance picks it up after the lock expires. Alternative: Use a database `SELECT ... FOR UPDATE` row lock or a Redis `SETNX` lock.

4. **Q: You have 15 `@Scheduled` methods in your application. Only 3 run concurrently — the other 12 wait. You didn't configure the scheduler. Why?**
   A: The default scheduler uses a single thread (`poolSize=1`). All 15 tasks share one thread. If one task takes 30 minutes, the other 14 wait. Configure a `ThreadPoolTaskScheduler` with a sufficient pool size:
   ```java
   @Configuration
   @EnableScheduling
   public class SchedulingConfig implements SchedulingConfigurer {
       @Override
       public void configureTasks(ScheduledTaskRegistrar registrar) {
           ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
           scheduler.setPoolSize(10); // At least as many as concurrent tasks
           scheduler.setThreadNamePrefix("scheduled-");
           scheduler.setWaitForTasksToCompleteOnShutdown(true);
           scheduler.setAwaitTerminationSeconds(30);
           scheduler.initialize();
           registrar.setTaskScheduler(scheduler);
       }
   }
   ```

5. **Q: You schedule a task with `@Scheduled(cron = "0 0 0 * * ?")` to run at midnight. The server clock drifts by 10 minutes. The task runs at 12:10 AM instead of midnight. How do you prevent clock drift from affecting scheduled tasks?**
   A: Use NTP (Network Time Protocol) to sync server clocks. For tasks that must run at exact times, add a tolerance check at the start of the task:
   ```java
   @Scheduled(cron = "0 0 0 * * ?")
   public void midnightTask() {
       LocalTime now = LocalTime.now();
       if (now.isAfter(LocalTime.of(0, 5))) {
           log.warn("Scheduled task running late: {}", now);
           // Still execute, but log the drift
       }
       // task logic
   }
   ```
   For extreme precision, use a distributed scheduler like Quartz that can be configured with a separate time source.

6. **Q: Your `@Scheduled` task runs on a fixedDelay of 10 seconds. After a database migration that takes 2 hours, the task resumes with 10 second intervals. But it tries to process 2 hours of backlog data and the system crashes. How do you prevent this?**
   A: Add a guard that limits how far back the task looks:
   ```java
   @Scheduled(fixedDelay = 10_000)
   public void processEvents() {
       Instant cutoff = Instant.now().minus(5, ChronoUnit.MINUTES); // Max 5 min backlog
       List<Event> events = eventRepository.findUnprocessedSince(cutoff);
       if (events.size() > 1000) {
           log.warn("Processing {} events — potential backlog issue", events.size());
           alertService.sendAlert("Large event backlog detected");
       }
       for (Event event : events) {
           processEvent(event);
       }
   }
   ```
   Better: Instead of fixedDelay, use a cron that runs once per minute, and within each run, process a limited batch. This prevents the "catch-up avalanche."

7. **Q: You create a `@Scheduled` method in a `@Configuration` class. The method never runs. Why?**
   A: `@Configuration` classes are usually enhanced with CGLIB proxies. However, `@Scheduled` on a method in a `@Configuration` class may not be processed because `@Configuration` classes are processed early in the lifecycle. The `ScheduledAnnotationBeanPostProcessor` may not scan them. Move `@Scheduled` methods to a `@Component` class. The `@Configuration` class should only contain `@Bean` definitions, not scheduled task implementations.

8. **Q: You need to run a task at 10:00 AM on the first business day of every month. Cron cannot easily express "first business day" (accounting for weekends and holidays). How do you implement this?**
   A: Run the task daily at 10 AM and add conditional logic inside the method:
   ```java
   @Scheduled(cron = "0 0 10 * * ?")
   public void firstBusinessDayTask() {
       LocalDate today = LocalDate.now();
       if (today.getDayOfMonth() != 1) return; // Not the 1st
       if (today.getDayOfWeek() == SATURDAY) return; // Skip Sat
       if (today.getDayOfWeek() == SUNDAY) return;  // Skip Sun
       // Check holidays via a HolidayCalendar service
       if (holidayCalendar.isHoliday(today)) return;
       // This is the first business day of the month
       executeMonthlyTask();
   }
   ```
   For more complex scheduling (like "second Tuesday"), use a library like Quarts or `CronExpression` with custom logic. The cron expression handles the frequency; the method logic handles the business rule.

9. **Q: Your scheduled task generates a large report and sends it via email. The task takes 45 minutes. During this time, the HTTP thread serving the admin UI is also used by the scheduled task's thread. Which thread pool is the scheduled task using?**
   A: By default, `@Scheduled` tasks use the `TaskScheduler` thread pool (single-threaded by default). This is separate from the Tomcat HTTP thread pool. The scheduled task does NOT use HTTP threads. However, if the default scheduler has only 1 thread, this long task blocks ALL other scheduled tasks. The fix is to increase the scheduler pool size (as in Q4) or use `@Async` on the long-running task:
   ```java
   @Async
   @Scheduled(cron = "0 0 2 * * ?")
   public CompletableFuture<Void> generateReportAsync() {
       // Runs on the async executor pool, not the scheduler pool
       reportService.generateReport();
       return CompletableFuture.completedFuture(null);
   }
   ```
   This frees the scheduler thread to trigger other tasks.

10. **Q: Your `@Scheduled` task runs every 5 seconds and queries the database. The database goes down for 2 minutes. When it comes back, the task resumes. But the application has 50 threads BLOCKED waiting for database connections. Why?**
    A: The scheduled task has a `fixedRate = 5000`. With a single-threaded scheduler, tasks are sequential — the next task does not start until the previous one completes (or fails). But with a multi-threaded scheduler (pool size > 1), if the database goes down, each task throws an exception (blocking for the connection timeout). With a pool of 10 and a connection timeout of 30 seconds, you get 10 threads × 30s = 300 thread-seconds of blocking. Fix: Use `fixedDelay` instead of `fixedRate` so tasks don't accumulate during downtime. Always wrap the task in try-catch so the exception doesn't kill the scheduler thread. Use a circuit breaker to stop attempting when the database is down.

---

## Interview Questions

1. **What is the difference between `fixedRate` and `fixedDelay` in `@Scheduled`?** 
   A: `fixedRate` schedules the next execution at a fixed interval from the START of the current execution — tasks may overlap if they run longer than the interval. `fixedDelay` schedules from the COMPLETION of the current execution — tasks never overlap. Use `fixedRate` for time-critical tasks and `fixedDelay` for tasks that should not overlap.

2. **What happens if a `@Scheduled` method throws an exception?** 
   A: Spring's scheduler catches the exception and logs it, but the task is NOT rescheduled — the `ScheduledFuture` is cancelled. The task stops running permanently. Always wrap scheduled task bodies in try-catch to prevent this.

3. **How do you configure the thread pool for scheduled tasks?** 
   A: Implement `SchedulingConfigurer` and configure `ThreadPoolTaskScheduler` with a pool size. The default is a single thread, which means all scheduled tasks run sequentially. Set the pool size to at least the number of concurrent tasks you expect.

4. **What is a cron expression and what are the fields?** 
   A: A cron expression has 6 fields: `second minute hour day-of-month month day-of-week`. Example: `0 0 2 * * ?` means "daily at 2 AM." The `?` means "no specific value" (used when you specify day-of-week but not day-of-month, or vice versa).

5. **How do you ensure a scheduled task runs only once across multiple application instances?** 
   A: Use a distributed lock — ShedLock (recommended), database `SELECT ... FOR UPDATE`, Redis `SETNX`, or ZooKeeper. ShedLock integrates via `@SchedulerLock` annotation and works with MongoDB, JDBC, Redis, and ZooKeeper.

6. **What is the difference between `@Scheduled` and Quartz?** 
   A: `@Scheduled` is Spring's built-in, annotation-based scheduler suitable for simple periodic tasks. Quartz is a full-featured, enterprise-grade scheduler that supports: persistent jobs (survive restarts), clustered scheduling, complex triggers (calendars, misfire instructions), and dynamic job creation. Use `@Scheduled` for simple cases; use Quartz for complex enterprise scheduling.

7. **How do you dynamically schedule a task at runtime?** 
   A: Inject `TaskScheduler` and call `taskScheduler.schedule(task, trigger)`. For cron-based triggers, use `new CronTrigger(cronExpression)`. Keep the returned `ScheduledFuture` to cancel the task later. This is useful for user-configurable schedules.

8. **How do you monitor scheduled task execution?** 
   A: Use an `@Around` aspect on methods annotated with `@Scheduled` to log execution time and success/failure. Record metrics with Micrometer (`Timer`, `Counter`). Expose via Actuator.

9. **What is the default thread pool for `@Scheduled` and why is it problematic?** 
   A: The default uses a single-threaded executor (`poolSize=1`). All scheduled tasks share one thread. A long-running task blocks all other tasks. Always configure a larger pool with `SchedulingConfigurer`.

10. **How do you combine `@Scheduled` and `@Async`?** 
    A: Annotating a method with both `@Async` and `@Scheduled` executes the scheduled task on the async executor thread pool, not the scheduler pool. This frees the scheduler thread to trigger other tasks. Useful for long-running scheduled tasks.

---

## Developer Recommendations

- **Always wrap scheduled task methods in try-catch** — An unhandled exception stops the task permanently. The scheduler silently removes the task from its schedule. No logs, no alerts, no retries. Try-catch with logging and alerting is the minimum safety net.
- **Use `fixedDelay` over `fixedRate` for tasks that must not overlap** — Database cleanup, file processing, and batch jobs should never have concurrent executions. `fixedDelay` ensures sequential execution. Use `fixedRate` only for time-independent operations like heartbeat checks.
- **Configure a dedicated thread pool for scheduled tasks** — The default single-thread pool means one slow task blocks all others. Use `SchedulingConfigurer` to set `poolSize` to at least the number of concurrent tasks. Monitor pool utilization.
- **Use distributed locks (ShedLock) for clustered deployments** — Without coordination, every instance runs the same scheduled task simultaneously. ShedLock uses a shared database table or Redis to ensure only one instance executes a task at a time.
- **Make scheduled tasks idempotent** — Tasks may run multiple times due to failover, retries, or clock skew. Processing the same data twice should not cause issues. Use "processed" flags, upsert operations, or transactional boundaries.
- **Add metrics and monitoring to all scheduled tasks** — Track execution duration, success/failure counts, and backlog sizes. Set up alerts for task failures and abnormally long executions. Scheduled tasks often run outside normal monitoring and can fail silently for days.
- **Use `@Async` + `@Scheduled` for long-running tasks** — A task that takes 30 minutes on the scheduler thread blocks all other tasks. Combine `@Async` to offload execution to a separate thread pool, freeing the scheduler thread to trigger other tasks on time.
- **Set timezone explicitly with the `zone` attribute** — Default uses the server's timezone. If your server is UTC but users are in EST, your "daily at 2 AM" task runs at 2 AM UTC ≠ 2 AM EST. Use `zone = "America/New_York"` to specify the intended timezone.
