# Java Multithreading

---

## 1. Executive Summary

### What Is It?
Multithreading is the ability of a CPU to execute multiple threads concurrently. A thread is the smallest unit of execution within a process. Java provides built-in support for creating and managing threads via the `Thread` class and `Runnable` interface.

### Why Does It Exist?
- **CPU utilization** — keep CPU busy while one thread waits for I/O
- **Responsiveness** — UI thread stays responsive while background threads work
- **Throughput** — process multiple requests concurrently
- **Resource sharing** — threads within a process share memory (cheaper than processes)

### Thread vs Process

| Aspect | Process | Thread |
|--------|---------|--------|
| Memory | Separate address space | Shared within process |
| Creation | Heavy (fork/exec) | Lightweight |
| Context switch | Expensive (TLB flush) | Cheaper |
| Communication | IPC (pipes, sockets, shared mem) | Direct memory access |
| Isolation | Strong (can't crash other processes) | Weak (one thread crash kills process) |

### When to Use
- I/O-bound operations (file, network, database)
- Background tasks (caching, batch processing, maintenance)
- Parallel computation (CPU-bound, divisible workload)
- Responsive UI (Swing/JavaFX event thread)
- Server request handling (one thread per request)

### When NOT to Use
- CPU-bound with no parallelism (single-core, non-divisible)
- Simple sequential tasks (overhead > benefit)
- When shared mutable state is avoidable (use single-threaded)
- Real-time systems needing deterministic behavior (GC pauses)

---

## 2. Core Theory

### Creating Threads

#### 1. Extend Thread class
```java
class Worker extends Thread {
    @Override
    public void run() {
        System.out.println("Working in thread: " + Thread.currentThread().getName());
    }
}
new Worker().start();
```

#### 2. Implement Runnable (preferred)
```java
class Task implements Runnable {
    @Override
    public void run() {
        System.out.println("Task in thread: " + Thread.currentThread().getName());
    }
}
new Thread(new Task()).start();
```

#### 3. Lambda (Java 8+)
```java
new Thread(() -> System.out.println("Lambda thread")).start();
```

#### 4. Callable + Future (returns result)
```java
ExecutorService executor = Executors.newFixedThreadPool(4);
Future<Integer> future = executor.submit(() -> {
    Thread.sleep(1000);
    return 42;
});
Integer result = future.get(); // blocks until result ready
```

### Thread Lifecycle

```
          ┌──────────┐
          │   NEW    │  (after new Thread())
          └────┬─────┘
               │ start()
               ▼
          ┌──────────┐
          │ RUNNABLE │  (scheduled by OS thread scheduler)
          └────┬─────┘
               │
     ┌─────────┼─────────┐
     │         │         │
     ▼         ▼         ▼
┌────────┐ ┌────────┐ ┌──────────┐
│RUNNING │ │BLOCKED │ │ WAITING  │
└───┬────┘ │(sync)  │ │(wait/join)│
    │      └────────┘ └──────────┘
    │           │         │
    └───────────┴─────────┘
               │
               ▼
          ┌──────────┐
          │TERMINATED│
          └──────────┘
```

- **NEW**: Created but not started
- **RUNNABLE**: Eligible for execution (waiting for CPU)
- **BLOCKED**: Waiting for a monitor lock (synchronized)
- **WAITING**: Indefinitely waiting (wait(), join(), park())
- **TIMED_WAITING**: Waiting with timeout (sleep(), wait(timeout), join(timeout))
- **TERMINATED**: run() completed or exception thrown

### Thread Priority
`Thread.setPriority(Thread.NORM_PRIORITY)` (1-10). Platform-dependent — not guaranteed.

### Daemon Threads
Background threads that don't keep JVM alive. JVM exits when only daemon threads remain:
```java
thread.setDaemon(true); // Must be called before start()
```

---

## 3. Under-the-Hood Deep Dive

### Thread Stack
Each thread gets its own stack (default ~1MB on 64-bit JVM):
- Stores local variables, method calls, partial results
- Not shared between threads
- Stack size configurable: `-Xss1m`

### Thread Scheduling
- Modern JVMs use OS-level threads (1:1 mapping with kernel threads)
- Thread scheduler (OS-dependent) decides which thread runs
- Java doesn't guarantee scheduling order or fairness (unless explicitly coded)

### Context Switch Cost
- Saving/restoring registers, program counter, stack pointer
- TLB flush (memory mapping cache)
- Cache misses on resumption
- ~1-10µs per context switch

### Thread Overhead

| Resource | Cost |
|----------|------|
| Thread creation | ~1-2µs for the JVM call, OS-dependent |
| Stack memory | ~1MB (default), 1024 threads = 1GB |
| Context switch | ~1-10µs |
| Thread local | ~50ns per access |

---

## 4. Production Code Examples

### 4.1 Basic — Thread with Error Handling

```java
public class DatabaseMigrationTask implements Runnable {
    @Override
    public void run() {
        Thread currentThread = Thread.currentThread();
        currentThread.setName("migration-worker");
        currentThread.setUncaughtExceptionHandler((thread, throwable) ->
            log.error("Uncaught exception in thread {}", thread.getName(), throwable)
        );
        
        try {
            migrateData();
        } catch (Exception e) {
            log.error("Migration failed", e);
            throw new RuntimeException(e);
        }
    }
    
    private void migrateData() {
        // Migration logic
    }
}
```

### 4.2 Intermediate — Joining Threads

```java
public class ParallelDataLoader {
    public LoadResult loadAll() throws InterruptedException {
        Thread customerLoader = new Thread(this::loadCustomers);
        Thread orderLoader = new Thread(this::loadOrders);
        Thread productLoader = new Thread(this::loadProducts);
        
        customerLoader.start();
        orderLoader.start();
        productLoader.start();
        
        // Wait for all to complete
        customerLoader.join(30_000); // timeout 30s
        orderLoader.join(30_000);
        productLoader.join(30_000);
        
        return new LoadResult(
            customerLoader.isAlive() ? Status.TIMEOUT : Status.SUCCESS,
            orderLoader.isAlive() ? Status.TIMEOUT : Status.SUCCESS,
            productLoader.isAlive() ? Status.TIMEOUT : Status.SUCCESS
        );
    }
}
```

### 4.3 Advanced — Thread Pool with Callable

```java
@Service
public class ReportGenerationService {
    private final ExecutorService executor = Executors.newFixedThreadPool(4,
        new ThreadFactoryBuilder()
            .setNameFormat("report-worker-%d")
            .setDaemon(true)
            .setUncaughtExceptionHandler((t, e) -> 
                log.error("Uncaught in thread {}", t.getName(), e))
            .build());

    public ReportResult generateReport(ReportRequest request) {
        List<Callable<ReportSection>> tasks = createSectionTasks(request);
        
        try {
            List<Future<ReportSection>> futures = executor.invokeAll(tasks, 60, TimeUnit.SECONDS);
            
            List<ReportSection> results = new ArrayList<>();
            for (int i = 0; i < futures.size(); i++) {
                try {
                    results.add(futures.get(i).get(5, TimeUnit.SECONDS));
                } catch (TimeoutException e) {
                    futures.get(i).cancel(true);
                    results.add(ReportSection.timeout(tasks.get(i).getSectionName()));
                } catch (ExecutionException e) {
                    results.add(ReportSection.error(tasks.get(i).getSectionName(), e.getCause()));
                }
            }
            return ReportResult.composite(results);
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new ReportGenerationException("Report generation interrupted", e);
        }
    }
}
```

### 4.4 Bad vs Good — Thread Management

```java
// BAD: Creating thread per request (unbounded)
public void handleRequest(Request request) {
    new Thread(() -> process(request)).start(); // Thread leak!
}

// BAD: Not handling InterruptedException
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // Swallowing the interrupt — terrible!
}

// BAD: Not naming threads (impossible to debug)
new Thread(() -> doWork()).start(); // "Thread-42" tells nothing

// GOOD: Use thread pool
@Service
public class RequestHandler {
    private final ExecutorService executor = Executors.newFixedThreadPool(20);

    public void handleRequest(Request request) {
        executor.submit(() -> process(request));
    }
    
    @PreDestroy
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}

// GOOD: Handle InterruptedException properly
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // Restore interrupt flag
    // Either rethrow or handle gracefully
    throw new OperationCancelledException("Operation interrupted", e);
}
```

---

## 5. Common Mistakes (Top 10)

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Creating threads in unbounded loops | Thread leak, OOM | Use thread pool |
| 2 | Not naming threads | Impossible to debug | `setName("worker-%d")` |
| 3 | Swallowing InterruptedException | Lost interrupt signal | Restore: `Thread.currentThread().interrupt()` |
| 4 | Not shutting down executor service | Thread leak | `executor.shutdown()` in `@PreDestroy` |
| 5 | Using `Thread.stop()`/`suspend()`/`resume()` | Unsafe, deprecated | Use volatile flag |
| 6 | Thread without exception handler | Silent failures | `setUncaughtExceptionHandler()` |
| 7 | Assuming thread priority matters | OS ignores it | Don't rely on it |
| 8 | Starting thread in constructor | `this` escape before construction | Factory method |
| 9 | Not using daemon threads for background | JVM can't exit | `setDaemon(true)` |
| 10 | Thread.sleep(0) for yielding | Busy waiting | `Thread.yield()` (also limited guarantee) |

---

## 6. Cheat Sheet

```
═══ MULTITHREADING ═══════════════════════════════════════════

┌─ CREATION ─────────────────────────────────────────────────┐
│ new Thread(() -> work()).start()                            │
│ executor.submit(() -> work())                               │
│ executor.submit(() -> { return result; })  // Callable      │
└─────────────────────────────────────────────────────────────┘

┌─ THREAD STATES ────────────────────────────────────────────┐
│ NEW → RUNNABLE → RUNNING → BLOCKED/WAITING → TERMINATED    │
└─────────────────────────────────────────────────────────────┘

┌─ EXECUTOR SERVICE ─────────────────────────────────────────┐
│ newFixedThreadPool(n)  // Fixed size                        │
│ newCachedThreadPool()  // Dynamic, unbounded                │
│ newSingleThreadExecutor() // Single worker                  │
│ newScheduledThreadPool(n) // Delayed/periodic               │
│ Executors.newWorkStealingPool() // Fork-join (Java 8+)      │
└─────────────────────────────────────────────────────────────┘

┌─ RULES ────────────────────────────────────────────────────┐
│ • Use thread pools, not bare threads                         │
│ • Always name threads for debugging                          │
│ • Restore interrupt flag in InterruptedException handlers    │
│ • Shutdown executors gracefully                              │
│ • Don't swallow exceptions in threads                        │
│ • Prefer daemon threads for background tasks                 │
└─────────────────────────────────────────────────────────────┘
```
