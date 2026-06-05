# Spring Boot Async Processing

---

## 1. Executive Summary

Spring Boot supports asynchronous method execution via `@Async` and `@EnableAsync`. Methods run in a separate thread, returning immediately to the caller.

---

## 2. Core Theory

### @Async

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

### Thread Pool Configuration

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

## 3. Common Mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Calling @Async method from same class | Self-invocation bypasses proxy | Extract to separate bean |
| 2 | @Async on private method | Ignored | Use public methods |
| 3 | No exception handler | Silent failures | AsyncUncaughtExceptionHandler |
| 4 | Default thread pool (unbounded) | Thread leak, OOM | Configure pool explicitly |
| 5 | Forgetting @EnableAsync | Method runs synchronously | Add annotation |
| 6 | Returning void without error handling | Lost exceptions | Return CompletableFuture or handle internally |

---

## 4. Cheat Sheet

```
═══ ASYNC PROCESSING ═════════════════════════════════════════

┌─ ANNOTATIONS ──────────────────────────────────────────────┐
│ @EnableAsync — on @Configuration                           │
│ @Async — on public method to run asynchronously            │
│ @Async — return void or CompletableFuture<T>               │
└─────────────────────────────────────────────────────────────┘

┌─ CONFIGURATION ────────────────────────────────────────────┐
│ ThreadPoolTaskExecutor — corePoolSize, maxPoolSize, queue  │
│ CallerRunsPolicy — backpressure when pool full             │
│ AsyncUncaughtExceptionHandler — log unhandled exceptions   │
└─────────────────────────────────────────────────────────────┘

┌─ RULES ────────────────────────────────────────────────────┐
│ • @Async on public methods only                             │
│ • Self-invocation bypasses proxy (extract to separate bean) │
│ • Always configure thread pool explicitly                   │
│ • Handle exceptions inside async method or via handler      │
│ • For results, return CompletableFuture<T>                  │
└─────────────────────────────────────────────────────────────┘
```
