# Spring Boot Async Processing

---

## Overview

- **Definition:** Spring Boot Async Processing enables methods to run in a separate thread, returning control to the caller immediately. It is built on Spring's `@Async` annotation and `@EnableAsync` configuration, and is ideal for non-blocking operations like email sending, report generation, and notification dispatch.
- **Why It Exists:** To improve application responsiveness and throughput by offloading long-running or non-critical operations to background threads, freeing the main request thread to handle more user requests.
- **Key Concepts:**
  - **@Async Annotation:** Methods annotated with `@Async` execute in a separate thread. The caller returns immediately without waiting for the method to complete:

    ```java
    @EnableAsync
    @SpringBootApplication
    public class Application {}

    @Service
    public class EmailService {

        @Async
        public void sendEmail(String to, String body) {
            // Runs in a separate thread
            // Caller returns immediately
            try {
                Thread.sleep(2000); // Simulate email sending
                log.info("Email sent to {}", to);
            } catch (Exception e) {
                log.error("Failed to send email", e);
            }
        }

        @Async
        public CompletableFuture<SendResult> sendEmailWithResult(String to, String body) {
            // Returns a Future — caller can check result later
            return CompletableFuture.completedFuture(new SendResult(to, true));
        }
    }
    ```

  - **Return Types for `@Async`:**
    - **`void`** — Fire-and-forget. No way to check result or exception. Handle exceptions internally or configure `AsyncUncaughtExceptionHandler`.
    - **`CompletableFuture<T>`** — Returns a future that the caller can use to get the result or exception later. Supports composition with `thenApply`, `exceptionally`, and `allOf`.
    - **`Future<T>`** — Older interface, less flexible than `CompletableFuture`. Avoid in new code.
    - **`ListenableFuture<T>`** — Deprecated since Spring 6. Use `CompletableFuture` instead.

  - **Thread Pool Configuration:** By default, Spring uses `SimpleAsyncTaskExecutor` (creates a new thread per task — not for production). Always configure a proper thread pool:

    ```java
    @Configuration
    @EnableAsync
    public class AsyncConfig implements AsyncConfigurer {

        @Override
        public Executor getAsyncExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(5);
            executor.setMaxPoolSize(10);
            executor.setQueueCapacity(100);
            executor.setThreadNamePrefix("async-");
            executor.setWaitForTasksToCompleteOnShutdown(true);
            executor.setAwaitTerminationSeconds(30);
            executor.setRejectedExecutionHandler(new CallerRunsPolicy());
            executor.initialize();
            return executor;
        }

        @Override
        public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
            return (ex, method, params) ->
                log.error("Async method {} threw exception", method.getName(), ex);
        }
    }
    ```

---

## Core Concepts

### How `@Async` Works Under the Hood

- When a method annotated with `@Async` is called:
  - Spring's AOP proxy intercepts the call.
  - The proxy submits the method execution to the configured `TaskExecutor`.
  - The method runs in a separate thread from the thread pool.
  - The caller receives a `CompletableFuture` (if the method returns one) or `void` immediately.

### Thread Pool Executor Configuration

- Key parameters for `ThreadPoolTaskExecutor`:
  - **`corePoolSize`** — Minimum number of threads kept alive in the pool.
  - **`maxPoolSize`** — Maximum number of threads the pool can grow to.
  - **`queueCapacity`** — Number of tasks that can be queued before new threads are created.
  - **`keepAliveSeconds`** — Time excess idle threads wait before terminating.
  - **`threadNamePrefix`** — Prefix for thread names (useful for debugging and monitoring).
  - **`rejectedExecutionHandler`** — Policy when the pool and queue are full:
    - `CallerRunsPolicy` — The caller thread executes the task, providing natural backpressure.
    - `AbortPolicy` — Throws `RejectedExecutionException` (default).
    - `DiscardPolicy` — Silently discards the task.
    - `DiscardOldestPolicy` — Discards the oldest queued task.

### Exception Handling

- For `void` methods, exceptions in async methods are not propagated to the caller. Handle them using `AsyncUncaughtExceptionHandler`:

  ```java
  @Override
  public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
      return (ex, method, params) -> {
          log.error("Async method {} failed with params {}",
              method.getName(), params, ex);
          // Send alert, retry, etc.
      };
  }
  ```

- For `CompletableFuture` methods, exceptions are captured in the future:

  ```java
  CompletableFuture<Result> future = emailService.sendEmailWithResult(to, body);
  future.exceptionally(ex -> {
      log.error("Async email failed", ex);
      return new SendResult(to, false);
  });
  ```

---

## Common Mistakes

- **Self-invocation bypassing the proxy** — Calling an `@Async` method from within the same class executes synchronously because the AOP proxy is not involved.
  - Why it looks correct: The method compiles and runs — the async behavior is simply absent, and no error or warning indicates the proxy was bypassed.
  - Fix: Extract to a separate bean for async behavior to work.

- **`@Async` on private methods** — Ignored because the proxy cannot intercept private methods.
  - Why it looks correct: The annotation is present on the method and the code compiles — Spring provides no warning that private `@Async` methods are silently ignored.
  - Fix: Only use `@Async` on public methods for the annotation to be effective.

- **No exception handler for void methods** — Exceptions in void async methods are silently swallowed and completely lost.
  - Why it looks correct: The caller receives an immediate return and the application continues without error — the failure is invisible.
  - Fix: Always configure `AsyncUncaughtExceptionHandler` to catch and log failures.

- **Default thread pool (unbounded)** — `SimpleAsyncTaskExecutor` creates a new thread for every call, leading to thread leaks and OOM under load.
  - Why it looks correct: During development with low traffic, the executor works fine — threads are created and garbage-collected normally. The OOM only occurs under sustained production load.
  - Fix: Always configure a proper `ThreadPoolTaskExecutor` for production.

- **Forgetting `@EnableAsync`** — Without it, `@Async` methods execute synchronously in the caller's thread, negating the performance benefit entirely.
  - Why it looks correct: The application works and the method returns the correct result — the only difference is performance, which is invisible without profiling.
  - Fix: Add `@EnableAsync` on a `@Configuration` class.

- **Not handling context propagation** — Security context, transaction context, and MDC are not propagated to the async thread by default.
  - Why it looks correct: The async method executes successfully — the missing context only manifests when audit logs lack user info or security checks fail in the async thread.
  - Fix: Use `TaskDecorator` to capture and restore context across threads.

- **Returning `void` without error handling inside the method** — Any exception inside a `void` async method is lost unless caught internally.
  - Why it looks correct: The method logic appears correct and testing with valid inputs never triggers exceptions — the error handling gap only surfaces when a production edge case causes a failure.
  - Fix: Always wrap the method body in try-catch with proper logging and alerting.

---

## Real-World Scenarios

### Scenario 1: Email Notification Service with Async Processing

- An e-commerce order service sends confirmation emails, SMS alerts, and push notifications after every order. Synchronous processing adds 3 seconds to the order response time. Users experience slow checkout.

  ```java
  @Service
  public class OrderService {
      private final NotificationService notificationService;

      @Transactional
      public Order placeOrder(CreateOrderRequest request) {
          Order order = orderRepository.save(new Order(request));
          // Return immediately — notification is async
          notificationService.sendOrderConfirmation(order);
          return order;
      }
  }

  @Service
  public class NotificationService {
      @Async
      public CompletableFuture<Void> sendOrderConfirmation(Order order) {
          emailService.send(order.getCustomerEmail(), buildEmail(order));
          smsService.send(order.getCustomerPhone(), "Order " + order.getId() + " confirmed");
          pushService.send(order.getUserId(), "Your order has been placed");
          return CompletableFuture.completedFuture(null);
      }
  }
  ```

- The checkout response time drops from 3.2 seconds to 200ms. The notifications are sent in background threads.

### Scenario 2: PDF Report Generation with User Feedback

- A SaaS analytics platform generates PDF reports. Reports can take 30-60 seconds to generate. Users should not wait synchronously — they should get a "processing" response and be notified when the report is ready.

  ```java
  @RestController
  @RequestMapping("/api/v1/reports")
  public class ReportController {
      private final ReportGenerationService reportService;

      @PostMapping
      public ResponseEntity<ReportResponse> generateReport(@RequestBody ReportRequest request) {
          String reportId = UUID.randomUUID().toString();
          reportService.generateReport(reportId, request);
          return ResponseEntity.accepted()
              .body(new ReportResponse(reportId, "Report is being generated. Check status at /api/v1/reports/" + reportId));
      }

      @GetMapping("/{reportId}")
      public ResponseEntity<ReportStatus> getStatus(@PathVariable String reportId) {
          return ResponseEntity.ok(reportService.getStatus(reportId));
      }
  }

  @Service
  public class ReportGenerationService {
      @Async("ioExecutor")
      public CompletableFuture<Void> generateReport(String reportId, ReportRequest request) {
          try {
              updateStatus(reportId, "PROCESSING");
              Report report = buildReport(request);
              reportStore.save(reportId, report);
              updateStatus(reportId, "COMPLETED");
              notificationService.notifyUser(request.getUserId(), "Report ready: " + reportId);
          } catch (Exception e) {
              updateStatus(reportId, "FAILED: " + e.getMessage());
          }
          return CompletableFuture.completedFuture(null);
      }
  }
  ```

### Scenario 3: Security Context Propagation Across Async Threads

- A fintech application audits every user action. The audit service runs async to avoid impacting response times. Without context propagation, the audit log cannot identify which user performed the action.

  ```java
  @Component
  public class ContextAwareTaskDecorator implements TaskDecorator {
      @Override
      public Runnable decorate(Runnable task) {
          SecurityContext securityContext = SecurityContextHolder.getContext();
          Map<String, String> mdcContext = MDC.getCopyOfContextMap();
          return () -> {
              try {
                  SecurityContextHolder.setContext(securityContext);
                  if (mdcContext != null) MDC.setContextMap(mdcContext);
                  task.run();
              } finally {
                  SecurityContextHolder.clearContext();
                  MDC.clear();
              }
          };
      }
  }

  @Configuration
  @EnableAsync
  public class AsyncConfig implements AsyncConfigurer {
      @Override
      public Executor getAsyncExecutor() {
          ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
          executor.setCorePoolSize(5);
          executor.setMaxPoolSize(10);
          executor.setQueueCapacity(100);
          executor.setTaskDecorator(new ContextAwareTaskDecorator());
          executor.setRejectedExecutionHandler(new CallerRunsPolicy());
          executor.initialize();
          return executor;
      }
  }
  ```

---

## Use Cases

- `@Async` offloads work from the request thread to improve responsiveness and throughput. These use cases cover common async patterns, thread-pool configuration, and context propagation.

- **Fire-and-forget notifications** — An order service must send confirmation emails, SMS, and push notifications after checkout without making the user wait.
  - Annotate the notification method with `@Async` and `void` return type. The request thread returns immediately; notifications run on a background thread.
  - **Avoid when:** The caller needs the result — use `CompletableFuture<T>` as the return type and combine futures if needed.

- **Long-running report generation with progress feedback** — A PDF report takes 30-60 seconds. The user should get a `202 Accepted` response and poll a status endpoint.
  - Return `CompletableFuture<Void>` from the `@Async` method. The controller returns `202 Accepted` with a report ID immediately. A separate status endpoint tracks completion.
  - **Avoid when:** The report is generated synchronously on a user action — consider WebSockets or SSE for push-based completion notification.

- **Dedicated thread pool for I/O vs. CPU tasks** — File uploads (I/O-bound) and image processing (CPU-bound) have different resource profiles. Mixing them causes thread contention.
  - Define multiple `@Bean("ioExecutor")` and `@Bean("cpuExecutor")` `ThreadPoolTaskExecutor` instances with different pool sizes. Reference them via `@Async("ioExecutor")`.
  - **Avoid when:** All async tasks have similar characteristics — a single shared executor with sensible defaults is sufficient.

- **Security context propagation to async threads** — An audit service runs async but loses the authenticated user identity, logging all actions as anonymous.
  - Implement a `TaskDecorator` that captures `SecurityContextHolder` and `MDC` from the caller thread and restores them in the worker thread. Register it on the executor.
  - **Avoid when:** The async task does not need user context — skip propagation to keep the task simpler and safer.

- **Async exception handling with `AsyncUncaughtExceptionHandler`** — An `@Async` method throws an exception that is silently swallowed because the caller's thread no longer exists.
  - Implement `AsyncConfigurer.getAsyncUncaughtExceptionHandler()` to log the exception and trigger alerts. For `CompletableFuture` returns, handle exceptions via ` exceptionally()` or `handle()`.
  - **Avoid when:** The caller uses `Future.get()` — the exception is thrown to the caller's thread on `get()`.

---

## Scenario-Based Questions

**Q: Your application processes file uploads. Each file needs virus scanning, thumbnail generation, and cloud upload. The total processing time is 15 seconds per file. Users wait on the upload endpoint for 15 seconds. How do you decouple this?**
- Use the "fire and forget" async pattern. The controller saves the file metadata, starts async processing, and returns immediately:
  ```java
  @PostMapping("/upload")
  public ResponseEntity<UploadResponse> upload(@RequestParam("file") MultipartFile file) {
      FileRecord record = fileService.saveMetadata(file);
      fileProcessingService.processAsync(record.getId(), file); // @Async
      return ResponseEntity.accepted(
          new UploadResponse(record.getId(), "File accepted for processing"));
  }
  ```
- The client polls a status endpoint. The async method handles scan, thumbnail, and upload. If processing fails, the status shows the error and the client can retry. Never make users wait for backend processing.

**Q: Your `@Async` method returns `void`. During a production incident, exceptions are silently swallowed and no one notices that emails are not being sent. How do you ensure visibility?**
- Two changes: (a) Change return type to `CompletableFuture<Void>` so callers can check results. (b) Configure `AsyncUncaughtExceptionHandler` as a catch-all for void methods:
  ```java
  @Override
  public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
      return (ex, method, params) -> {
          log.error("Async method {} failed with params {}", method.getName(), params, ex);
          notificationService.alert("Async failure in " + method.getName() + ": " + ex.getMessage());
      };
  }
  ```
- Never leave `void` async methods without error handling. The default behavior is to silently lose exceptions — a dangerous anti-pattern.

**Q: Your async thread pool executes 50 concurrent tasks. Each task queries the database. The database connection pool (size 10) is exhausted, and tasks start failing with connection timeout errors. How do you tune the system?**
- Rule of thumb: async pool size ≤ database connection pool size. If you have 10 DB connections, the async pool should have at most 10 threads. If async tasks perform I/O (DB calls, REST calls), use a larger pool but bound it. Better approach: separate async pools by operation type — one for CPU-bound (size = core count) and one for I/O-bound (larger, but limited by downstream capacity). Monitor both pool saturation and connection pool wait times.
- **Interview follow-up:** The candidate set the async pool size to 10 to match the DB connection pool. However, the application also has 20 HTTP handler threads that may execute synchronous database queries. Under traffic, HTTP threads take 6 connections, the async pool takes 4, and throughput is fine — but a traffic spike causes HTTP threads to grab 8 connections, leaving only 2 for async tasks. The async queue backs up to 500 pending tasks, and the application runs out of memory. How would you design the sizing so HTTP threads and async threads don't compete for the same limited connection pool?

**Q: Your `@Async` method is annotated with `@Transactional`. The method runs in a separate thread, but the transaction scope is unclear. Does it join the caller's transaction or create its own?**
- The `@Async` method runs in a completely different thread. The caller's transaction context is NOT propagated (transactions are thread-bound). The `@Transactional` on the async method creates its OWN transaction. If you need the async operation to participate in the caller's transaction, you cannot use `@Async` — execute synchronously or use a distributed transaction. The async method's transaction has propagation `REQUIRED` (creates a new one since there's no existing transaction in the async thread).

**Q: You have 3 `@Async` methods in the same service class. Each needs a different thread pool. But calling one from within the same class doesn't work (self-invocation). How do you organize this?**
- Extract each pool-specific async method into its own bean:
  ```java
  @Service
  public class EmailService {
      @Async("emailExecutor")
      public CompletableFuture<Void> sendEmail(String to, String body) { ... }
  }

  @Service
  public class ReportService {
      @Async("reportExecutor")
      public CompletableFuture<Report> generateReport(Long id) { ... }
  }

  @Service
  public class NotificationService {
      @Async("pushExecutor")
      public CompletableFuture<Void> sendPush(Long userId, String message) { ... }
  }
  ```
- The caller injects each service separately. This avoids self-invocation and keeps each pool isolated.

**Q: Your async task does CPU-intensive work. The thread pool size is set to 50 on an 8-core machine. Performance is worse than synchronous execution. Why?**
- CPU-intensive tasks should have a pool size equal to the number of available cores (or cores + 1). With 50 threads on 8 cores, the CPU spends most time context-switching between threads rather than doing actual work. For CPU-bound tasks, use:
  ```java
  @Bean("cpuExecutor")
  public Executor cpuExecutor() {
      ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
      executor.setCorePoolSize(Runtime.getRuntime().availableProcessors()); // Usually 8
      executor.setMaxPoolSize(Runtime.getRuntime().availableProcessors());
      executor.setQueueCapacity(100);
      return executor;
  }
  ```
- For I/O-bound tasks (DB, REST, file reads), a larger pool (2-4x core count) is appropriate because threads spend time waiting.

**Q: Your application receives a traffic spike and the async queue grows to 10K pending tasks. Memory usage spikes to 90% heap. Threads start getting rejected with `RejectedExecutionException`. How do you handle this gracefully?**
- Use `CallerRunsPolicy` as the rejection handler — when the pool and queue are full, the CALLER thread executes the task (providing natural backpressure):
  ```java
  executor.setRejectedExecutionHandler(new CallerRunsPolicy());
  ```
- This slows down the request rate because the caller (e.g., HTTP handler thread) is now doing the async work. The HTTP thread pool fills up, and the load balancer stops sending requests. This is a graceful degradation. Also consider circuit breaker patterns for downstream services.
- **Interview follow-up:** The candidate suggested `CallerRunsPolicy` for natural backpressure. The async task that runs on the caller thread performs a slow I/O operation (e.g., compressing and uploading a 500MB file). The HTTP thread is now blocked for 30 seconds doing the upload. During those 30 seconds, the HTTP thread pool has one fewer thread available to serve requests, and the remaining threads may also become blocked as they hit the same backpressure. The entire application eventually stalls. What alternative rejection policy or architectural change would you use to prevent a single slow async task from taking down all HTTP request handling?

**Q: You run integration tests with `@Async` methods. Tests complete before the async method finishes, and assertions fail because the side effects haven't happened yet. How do you test async behavior deterministically?**
- Use `CompletableFuture` return types and `get()` with a timeout:
  ```java
  @Test
  void testAsyncProcessing() throws Exception {
      CompletableFuture<Result> future = service.processAsync(data);
      Result result = future.get(10, TimeUnit.SECONDS); // Wait for completion
      assertThat(result.isSuccess()).isTrue();
  }
  ```
- For void methods, use `CountDownLatch` inside the async method, or inject a `BlockingQueue` to capture results. Alternatively, in integration tests, use `TestTaskExecutor` that runs tasks synchronously:
  ```java
  @TestConfiguration
  static class TestConfig {
      @Bean(name = "ioExecutor")
      public Executor testExecutor() {
          return Runnable::run; // Runs synchronously in test
      }
  }
  ```

**Q: Your `@Async` method calls `Thread.sleep(10000)` to simulate a delay. During shutdown, the application waits 30 seconds (configured `awaitTerminationSeconds`). Users complain about slow shutdowns. How do you handle long-running tasks during shutdown?**
- Never use `Thread.sleep()` in async methods — it holds a thread doing nothing. Use scheduled delays or reactive timers instead. For shutdown, configure the executor to cancel running tasks on shutdown:
  ```java
  executor.setWaitForTasksToCompleteOnShutdown(true);
  executor.setAwaitTerminationSeconds(30);
  // Add a shutdown hook
  @PreDestroy
  public void shutdown() {
      executor.shutdown();
      try {
          if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
              executor.shutdownNow(); // Force shutdown of remaining tasks
          }
      } catch (InterruptedException e) {
          executor.shutdownNow();
      }
  }
  ```
- For critical tasks that must complete, use a separate, non-interruptible executor with a longer timeout.

**Q: You use MDC for request tracing. Async method logs don't have the correlation ID from the caller's request. How do you propagate MDC context?**
- Use a `TaskDecorator` that captures the MDC context from the caller thread and restores it in the async thread:
  ```java
  @Component
  public class MdcTaskDecorator implements TaskDecorator {
      @Override
      public Runnable decorate(Runnable task) {
          Map<String, String> contextMap = MDC.getCopyOfContextMap();
          return () -> {
              try {
                  if (contextMap != null) MDC.setContextMap(contextMap);
                  task.run();
              } finally {
                  MDC.clear();
              }
          };
      }
  }
  ```
- Register the decorator with the executor. This ensures all logs from async threads include the original request's correlation ID, making distributed tracing possible.

---

## Interview Questions

- **What is `@Async` and how does it work?**
  - `@Async` is a Spring annotation that executes a method in a separate thread. Spring creates an AOP proxy that intercepts the call and submits the method to a `TaskExecutor`. The caller returns immediately (with a `CompletableFuture` if the method returns one). Requires `@EnableAsync` on a configuration class.

- **What return types are supported by `@Async`?**
  - `void` (fire-and-forget, no result), `CompletableFuture<T>` (non-blocking future — preferred), `Future<T>` (older, blocking), `ListenableFuture<T>` (deprecated since Spring 6). Use `CompletableFuture` for composable async behavior.

- **What is the difference between `SimpleAsyncTaskExecutor` and `ThreadPoolTaskExecutor`?**
  - `SimpleAsyncTaskExecutor` creates a new thread for every task — unbounded, not for production. `ThreadPoolTaskExecutor` uses a bounded thread pool with configurable core size, max size, queue, and rejection policy. Always use `ThreadPoolTaskExecutor` in production.

- **What is the self-invocation problem with `@Async`?**
  - Calling an `@Async` method from within the same class bypasses the AOP proxy, so the method runs synchronously. Fix by extracting the `@Async` method to a separate bean. Self-invocation affects all AOP-based annotations (`@Transactional`, `@Cacheable`, `@Async`).

- **How do you handle exceptions in `@Async` methods?**
  - For `void` methods, implement `AsyncUncaughtExceptionHandler` in `AsyncConfigurer`. For `CompletableFuture<T>` methods, the future captures the exception and can be handled with `future.exceptionally()` or `future.whenComplete()`.

- **How do you configure a custom thread pool for `@Async`?**
  - Implement `AsyncConfigurer.getAsyncExecutor()` or define a `@Bean` of type `ThreadPoolTaskExecutor`. Use `@Async("executorBeanName")` to specify which pool to use. Configure core pool size, max pool size, queue capacity, thread prefix, and rejection policy.

- **What is `TaskDecorator` and when do you use it?**
  - `TaskDecorator` is an interface that wraps a `Runnable` before it is executed by the thread pool. Use it to propagate context (SecurityContext, MDC, request attributes) from the caller thread to the async thread. Set it via `executor.setTaskDecorator(decorator)`.

- **How do `@Async` and `@Transactional` interact?**
  - Each creates its own AOP proxy. When both are on the same method, the transaction is local to the async thread — the caller's transaction does not propagate. The async method runs in its own transaction. Order of interceptors: `@Transactional` wrapper around `@Async` wrapper.

- **How do you test `@Async` methods?**
  - Use `CompletableFuture` return types and `future.get(timeout)` in tests. For void methods, use `CountDownLatch` or `Awaitility`. For integration tests, replace the async executor with a synchronous one (`Runnable::run`) using `@TestConfiguration`.

- **What thread pool rejection policies are available and when would you use each?**
  - `AbortPolicy` (default — throws `RejectedExecutionException`), `CallerRunsPolicy` (caller thread executes the task — backpressure), `DiscardPolicy` (silently discards), `DiscardOldestPolicy` (discards oldest queued task). Use `CallerRunsPolicy` for graceful degradation under load.

---

## Developer Recommendations

- **Use `CompletableFuture<T>` over `void` for `@Async` methods** — Void async methods silently swallow exceptions. `CompletableFuture` captures failures and allows callers to check results, compose futures, and chain callbacks. The overhead is minimal and the debugging benefit is enormous.
- **Always configure a custom `ThreadPoolTaskExecutor`** — The default `SimpleAsyncTaskExecutor` creates unlimited threads, causing OOM and thread leaks. Configure core/max pool size, queue capacity, thread naming prefix, and a rejection policy (`CallerRunsPolicy` for backpressure).
- **Use separate thread pools for different workloads** — CPU-intensive (pool size = core count) and I/O-bound (pool size = 2-4x core count) tasks have different optimal pool sizes. Mixing them in one pool leads to either CPU underutilization or thread contention.
- **Propagate context (SecurityContext, MDC) via `TaskDecorator`** — Async threads lose the caller's security context, MDC, and request attributes. Without propagation, audit logs lack user info and tracing breaks. A `TaskDecorator` captures context before the thread switch and restores it in the async thread.
- **Implement `AsyncUncaughtExceptionHandler` for void async methods** — By default, exceptions in void `@Async` methods are completely lost. The handler logs the error, sends alerts, and provides visibility into failures that would otherwise go unnoticed.
- **Use `CallerRunsPolicy` as the rejection handler** — When the async pool is full, `AbortPolicy` (default) throws an exception to the caller. `CallerRunsPolicy` executes the task on the caller thread, providing natural backpressure. The HTTP thread pool fills up, and the load balancer redirects traffic.
- **Size async thread pools relative to the downstream capacity** — If async tasks call a database with a connection pool of 10, the async pool should not exceed 10. More threads just queue up waiting for connections. Monitor downstream resource usage.
- **Prefer `ApplicationEvent` + `@Async` + `@TransactionalEventListener` for fire-and-forget side effects** — Instead of calling `@Async` methods directly from service code, publish an event. The event listener can be async and transactional. This decouples the primary operation from side effects (emails, notifications, audit).
