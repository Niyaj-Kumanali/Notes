# Spring Boot Scheduling

---

## Overview

- **Definition:** Spring Boot Scheduling provides task execution capabilities using `@Scheduled` and `@EnableScheduling`. It supports **cron expressions**, **fixed rate**, **fixed delay**, and **initial delay** — enabling periodic and time-based job execution without external schedulers.
- **Why It Exists:** To automate recurring business logic such as batch processing, report generation, data synchronization, and health checks without requiring external schedulers or manual intervention.
- **Key Concepts:**
  - **Scheduling Types:** Spring provides four scheduling modes through the `@Scheduled` annotation: `fixedRate` (fixed interval from execution start), `fixedDelay` (fixed interval from execution completion), `initialDelay` (delays the first execution), and `cron` (cron-based scheduling). Use `@EnableScheduling` on a `@Configuration` class to activate annotation-driven scheduling. Example configuration:

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

  - **Cron Expression Format:** The cron format consists of six fields: `second minute hour day-of-month month day-of-week`. Common examples include `0 0 * * * *` for every hour, `0 0 8 * * MON-FRI` for weekdays at 8 AM, and `0 0 2 * * ?` for daily at 2 AM. Special characters include `*` (any value), `?` (no specific value), `-` (range), `/` (increment), `,` (list), `L` (last), and `#` (nth occurrence).

    | Expression | Meaning |
    |-----------|---------|
    | `0 0 * * * *` | Every hour |
    | `0 0 8 * * MON-FRI` | 8 AM weekdays |
    | `0 0/15 * * * *` | Every 15 minutes |
    | `0 0 2 * * ?` | Daily at 2 AM |
    | `0 0 0 1 1 ?` | Every Jan 1 midnight |
    | `0 0/30 9-17 * * MON-FRI` | Every 30 min during work hours weekdays |

  - **Fixed Rate vs Fixed Delay:** `fixedRate` schedules the next execution at a fixed interval from the **start** of the previous execution, meaning tasks may overlap or queue up if they take longer than the interval. `fixedDelay` schedules the next execution after a fixed delay from the **completion** of the previous execution, ensuring tasks never overlap. Use `fixedRate` for time-critical operations and `fixedDelay` for tasks that must run sequentially.

---

## Core Concepts

### Thread Pool Configuration

- By default, scheduled tasks use a single-threaded executor. For concurrent tasks, configure a larger pool:

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

### Dynamic Scheduling with `TaskScheduler`

- Schedule tasks programmatically at runtime:

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

### Conditional Scheduling

- Skip task execution based on conditions:

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

- **Using `fixedRate` when tasks can overrun** — If a task takes 10 seconds but `fixedRate` is 5 seconds, executions pile up or overlap.
  - Why it looks correct: The annotation says "every 5 seconds" and it works perfectly when the task completes quickly — the overlap only manifests when a slow network or database call extends execution time.
  - Fix: Use `fixedDelay` to prevent overlap when task duration is unpredictable.

- **Cron at the same second for many tasks** — Multiple tasks triggering simultaneously cause a thundering herd.
  - Why it looks correct: All tasks use `0 0 2 * * ?` which reads as "daily at 2 AM" — the repeating second field value is visually consistent, and the performance impact is invisible in testing with few tasks.
  - Fix: Stagger cron times (e.g., `0 0 2 * * ?` and `0 5 2 * * ?`) to spread load across different seconds or minutes.

- **No error handling** — If a scheduled task throws an unhandled exception, the task stops permanently because Spring's default scheduler does not retry.
  - Why it looks correct: The task method is straightforward and appears to never throw — the unhandled exception is a rare edge case that only surfaces in production.
  - Fix: Always wrap the task body in try-catch with logging and alerting.

- **Forgetting `@EnableScheduling`** — Without this annotation on a `@Configuration` class, all `@Scheduled` annotations are silently ignored and no tasks run.
  - Why it looks correct: The code compiles, the `@Scheduled` annotation is present, and there is no error message — the tasks simply never execute.
  - Fix: Add it once at the application level.

- **Long-running tasks blocking the pool** — With the default single-threaded scheduler, one long task blocks all others from executing.
  - Why it looks correct: The application starts without errors and the long task runs to completion — the developer only notices the problem when other scheduled tasks fail to run on time.
  - Fix: Increase pool size or use `@Async` for long-running operations to free the scheduler thread.

- **Tasks not idempotent** — Scheduled tasks can be accidentally triggered multiple times in clustered deployments or during failover.
  - Why it looks correct: In single-instance development, the task runs once as expected — duplicate execution is a clustered deployment problem that doesn't surface in local testing.
  - Fix: Ensure all scheduled tasks are idempotent so duplicate executions produce correct results.

- **Default single-thread pool** — Only one task runs at a time by default. If you have multiple `@Scheduled` methods, they queue up behind a slow task.
  - Why it looks correct: The application starts without errors and all tasks eventually run — the sequential execution is only noticeable when a task takes minutes and others miss their deadlines.
  - Fix: Configure a `ThreadPoolTaskScheduler` with a sufficient pool size.

---

## Real-World Scenarios

### Scenario 1: Nightly Report Generation with Idempotent Scheduling

- A retail company generates nightly sales reports at 2 AM. The task must run exactly once across 10 Kubernetes pods. If the task fails, it should retry. The report takes 30-45 minutes.

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

- A SaaS platform lets users configure when they receive automated reports. User A wants daily at 8 AM, User B wants weekly on Monday at 9 AM. These cannot use `@Scheduled` annotations because the schedules are dynamic.

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

- A microservice checks the health of a third-party API every 30 seconds. If the API is down, the task should not spam logs — it should stop checking after 3 consecutive failures and resume when the API recovers.

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

**Q: You have a `@Scheduled(fixedRate = 5000)` task that sends heartbeat signals. The task sometimes takes 10 seconds due to network latency. What happens to the execution schedule?**
- With `fixedRate`, a new execution starts every 5 seconds from the START of the previous execution. If a task takes 10 seconds, the next execution starts 5 seconds after the previous START (not after completion), causing tasks to overlap. If your heartbeat logic is not idempotent, this causes data corruption. Fix by using `fixedDelay` instead — the next execution starts 5 seconds after the PREVIOUS one COMPLETES. For fixed-rate semantics where exact intervals are required, ensure the task never exceeds its interval or make it idempotent.

**Q: Your scheduled task runs every hour and processes pending orders. After a deployment, you notice the task has not run for 8 hours. No errors in logs. What happened?**
- The task method threw an exception that propagated up, and Spring's default scheduler silently stopped the task — it will NOT reschedule after an exception. Always wrap the task body in try-catch with logging and alerting. Use an aspect that monitors every task execution and alerts on failures to prevent silent stops.
- **Interview follow-up:** The candidate correctly identified the silent stop due to unhandled exceptions. The team adds `try-catch` with logging to the task, but two weeks later the task stops again — the try-catch was added to the scheduled wrapper method, but the wrapper calls a private helper that throws a `NullPointerException` for a specific edge case, and the wrapper's catch clause catches the generic `Exception` but the log level is `DEBUG` which is disabled in production. How would you prevent this class of silent failure across ALL scheduled tasks, not just this one?

**Q: Your Spring Boot application runs on 5 instances. A scheduled task that sends a daily summary email fires 5 times — one per instance. Customers get 5 identical emails. How do you solve this?**
- Use a distributed lock mechanism like ShedLock, which uses a shared database table or Redis to coordinate locks across instances. Only one instance acquires the lock and executes the task; if that instance fails, another picks it up after the lock expires. Alternative approaches include database `SELECT ... FOR UPDATE` row locks or Redis `SETNX` locks.

**Q: You have 15 `@Scheduled` methods in your application. Only 3 run concurrently — the other 12 wait. You didn't configure the scheduler. Why?**
- The default scheduler uses a single thread (`poolSize=1`), so all 15 tasks share one thread. If one task takes 30 minutes, the other 14 wait. Configure a `ThreadPoolTaskScheduler` with a pool size at least as large as the number of concurrent tasks you expect, using `SchedulingConfigurer`.

**Q: You schedule a task with `@Scheduled(cron = "0 0 0 * * ?")` to run at midnight. The server clock drifts by 10 minutes. The task runs at 12:10 AM instead of midnight. How do you prevent clock drift from affecting scheduled tasks?**
- Use NTP (Network Time Protocol) to sync server clocks. Add a tolerance check at the start of the task that logs a warning if the current time exceeds the expected window. For extreme precision, use a distributed scheduler like Quartz that can be configured with a separate time source independent of the system clock.

**Q: Your `@Scheduled` task runs on a fixedDelay of 10 seconds. After a database migration that takes 2 hours, the task resumes with 10 second intervals. But it tries to process 2 hours of backlog data and the system crashes. How do you prevent this?**
- Add a guard that limits how far back the task looks, processing only events within a bounded time window. Better yet, use a cron that runs once per minute and processes a limited batch within each run, preventing the catch-up avalanche. Use alerting to detect when backlog exceeds normal thresholds.
- **Interview follow-up:** The candidate proposed a bounded time window. If the database migration took 2 hours, and the guard limits the window to 5 minutes of backlog, the remaining 1 hour 55 minutes of unprocessed events are permanently lost — they exist in the database but the task will never see them because the next run only looks at events newer than its last watermark. How would you design recovery so that backlog is processed gradually over multiple runs without crashing the system, and without data loss?

**Q: You create a `@Scheduled` method in a `@Configuration` class. The method never runs. Why?**
- `@Configuration` classes are processed early in the lifecycle and the `ScheduledAnnotationBeanPostProcessor` may not scan them. Move `@Scheduled` methods to a `@Component` class. The `@Configuration` class should only contain `@Bean` definitions, not scheduled task implementations.

**Q: You need to run a task at 10:00 AM on the first business day of every month. Cron cannot easily express "first business day" (accounting for weekends and holidays). How do you implement this?**
- Run the task daily at 10 AM and add conditional logic inside the method that checks if today is the first business day by verifying the day of month, checking for weekends, and consulting a holiday calendar service. The cron expression handles the frequency while the method logic handles the business rule.

**Q: Your scheduled task generates a large report and sends it via email. The task takes 45 minutes. During this time, the HTTP thread serving the admin UI is also used by the scheduled task's thread. Which thread pool is the scheduled task using?**
- `@Scheduled` tasks use the `TaskScheduler` thread pool (single-threaded by default), which is separate from the Tomcat HTTP thread pool. However, with the default single-threaded scheduler, this long task blocks all other scheduled tasks. Increase the scheduler pool size or use `@Async` on the long-running task to offload it to a separate executor pool, freeing the scheduler thread.

**Q: Your `@Scheduled` task runs every 5 seconds and queries the database. The database goes down for 2 minutes. When it comes back, the task resumes. But the application has 50 threads BLOCKED waiting for database connections. Why?**
- With a multi-threaded scheduler and `fixedRate`, each task attempt creates a new thread that blocks on the connection timeout. With a pool of 10 and a connection timeout of 30 seconds, you get 10 threads × 30s = 300 thread-seconds of blocking. Fix by using `fixedDelay` instead of `fixedRate` so tasks don't accumulate during downtime, wrapping the task in try-catch so exceptions don't kill threads, and using a circuit breaker to stop attempting when the database is down.

---

## Interview Questions

- **What is the difference between `fixedRate` and `fixedDelay` in `@Scheduled`?**
  - `fixedRate` schedules the next execution at a fixed interval from the START of the current execution — tasks may overlap if they run longer than the interval. `fixedDelay` schedules from the COMPLETION of the current execution — tasks never overlap. Use `fixedRate` for time-critical tasks and `fixedDelay` for tasks that should not overlap.

- **What happens if a `@Scheduled` method throws an exception?**
  - Spring's scheduler catches the exception and logs it, but the task is NOT rescheduled — the `ScheduledFuture` is cancelled and the task stops running permanently. Always wrap scheduled task bodies in try-catch to prevent silent failures.

- **How do you configure the thread pool for scheduled tasks?**
  - Implement `SchedulingConfigurer` and configure `ThreadPoolTaskScheduler` with a pool size. The default is a single thread, meaning all scheduled tasks run sequentially. Set the pool size to at least the number of concurrent tasks you expect.

- **What is a cron expression and what are the fields?**
  - A cron expression has 6 fields: `second minute hour day-of-month month day-of-week`. Example: `0 0 2 * * ?` means daily at 2 AM. The `?` means no specific value, used as a wildcard when the other day field is specified.

- **How do you ensure a scheduled task runs only once across multiple application instances?**
  - Use a distributed lock — ShedLock (recommended), database `SELECT ... FOR UPDATE`, Redis `SETNX`, or ZooKeeper. ShedLock integrates via `@SchedulerLock` annotation and works with MongoDB, JDBC, Redis, and ZooKeeper.

- **What is the difference between `@Scheduled` and Quartz?**
  - `@Scheduled` is Spring's built-in annotation-based scheduler suitable for simple periodic tasks. Quartz is a full-featured enterprise-grade scheduler supporting persistent jobs that survive restarts, clustered scheduling, complex triggers with calendars and misfire instructions, and dynamic job creation. Use `@Scheduled` for simple cases; use Quartz for complex enterprise scheduling.

- **How do you dynamically schedule a task at runtime?**
  - Inject `TaskScheduler` and call `taskScheduler.schedule(task, trigger)`. For cron-based triggers, use `new CronTrigger(cronExpression)`. Keep the returned `ScheduledFuture` to cancel the task later. This pattern is useful for user-configurable schedules that cannot use static annotations.

- **How do you monitor scheduled task execution?**
  - Use an `@Around` aspect on methods annotated with `@Scheduled` to log execution time and success or failure. Record metrics with Micrometer (`Timer`, `Counter`) and expose via Actuator for dashboarding and alerting.

- **What is the default thread pool for `@Scheduled` and why is it problematic?**
  - The default uses a single-threaded executor (`poolSize=1`), meaning all scheduled tasks share one thread and a long-running task blocks all others. Always configure a larger pool with `SchedulingConfigurer` to match your concurrency needs.

- **How do you combine `@Scheduled` and `@Async`?**
  - Annotating a method with both `@Async` and `@Scheduled` executes the scheduled task on the async executor thread pool instead of the scheduler pool. This frees the scheduler thread to trigger other tasks on time, making it useful for long-running scheduled tasks.

---

## Developer Recommendations

- **Always wrap scheduled task methods in try-catch** — An unhandled exception stops the task permanently because the scheduler silently removes the task from its schedule with no logs, alerts, or retries. Try-catch with logging and alerting is the minimum safety net for reliable scheduled execution.
- **Use `fixedDelay` over `fixedRate` for tasks that must not overlap** — Database cleanup, file processing, and batch jobs should never have concurrent executions. `fixedDelay` ensures sequential execution. Use `fixedRate` only for time-independent operations like heartbeat checks.
- **Configure a dedicated thread pool for scheduled tasks** — The default single-thread pool means one slow task blocks all others. Use `SchedulingConfigurer` to set `poolSize` to at least the number of concurrent tasks and monitor pool utilization in production.
- **Use distributed locks (ShedLock) for clustered deployments** — Without coordination, every instance runs the same scheduled task simultaneously. ShedLock uses a shared database table or Redis to ensure only one instance executes a task at a time, preventing duplicate processing.
- **Make scheduled tasks idempotent** — Tasks may run multiple times due to failover, retries, or clock skew, so processing the same data twice should not cause issues. Use processed flags, upsert operations, or transactional boundaries to achieve idempotency.
- **Add metrics and monitoring to all scheduled tasks** — Track execution duration, success or failure counts, and backlog sizes. Set up alerts for task failures and abnormally long executions since scheduled tasks often run outside normal monitoring and can fail silently for days.
- **Use `@Async` + `@Scheduled` for long-running tasks** — A task that takes 30 minutes on the scheduler thread blocks all other tasks. Combine `@Async` to offload execution to a separate thread pool, freeing the scheduler thread to trigger other tasks on time.
- **Set timezone explicitly with the `zone` attribute** — The default uses the server timezone, which can cause unexpected behavior if your server is UTC but users are in a different timezone. Use `zone = "America/New_York"` to specify the intended timezone for cron-based scheduling.
