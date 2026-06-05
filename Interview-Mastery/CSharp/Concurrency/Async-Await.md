# Async-Await

---

## Overview

- **Definition:** C#'s language-level asynchronous programming model (C# 5.0+) that enables non-blocking I/O using a synchronous-looking code pattern with compiler-generated state machines.
- **Why It Exists:** Frees threads during I/O wait, improving scalability for server applications and UI responsiveness. Async/await is about freeing threads, not parallelism.
- **Key Concepts:** **`async`** keyword marks methods as asynchronous; **`await`** suspends execution until the task completes; **`Task<T>`** / **`ValueTask<T>`** represent async operations; **`IAsyncEnumerable<T>`** enables async streaming; **`ConfigureAwait`** controls continuation context.

---

## Core Concepts

- **async Keyword:** Marks a method as asynchronous, enables `await` inside it, and does NOT create a new thread. The compiler rewrites the method into a state machine struct implementing `IAsyncStateMachine`.
- **await Keyword:** Suspends execution of the method until the awaited task completes. If the task is already complete, execution continues synchronously; otherwise, control returns to the caller and a continuation is scheduled.
- **Task and Task\<T\>:** Represent the "promise" of a future value. `Task` is a reference type (~80 bytes heap allocated). Tasks can be composed via `Task.WhenAll`, `Task.WhenAny`, and `Task.Run`.
- **ValueTask\<T\>:** A value-type variant that avoids heap allocation when the result is often synchronous. Can only be awaited once. Backed by `IValueTaskSource<T>` for pooling scenarios.

```csharp
async Task<string> ReadFileAsync(string path)
{
    using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
        FileShare.Read, 4096, FileOptions.Asynchronous);
    byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
    int bytesRead = await fs.ReadAsync(buffer, 0, buffer.Length);
    string result = Encoding.UTF8.GetString(buffer, 0, bytesRead);
    ArrayPool<byte>.Shared.Return(buffer);
    return result;
}
```

- **State Machine Transformation:** The compiler generates a struct with fields for captured locals, a `TaskAwaiter`, and a `_state` field tracking progress. The `MoveNext()` method uses a `switch` statement to resume at the correct await point. Exceptions are caught and set on the `TaskCompletionSource`.
- **SynchronizationContext and ConfigureAwait:** By default, after `await`, the continuation runs on the original `SynchronizationContext` (e.g., UI thread in WPF). `ConfigureAwait(false)` skips this post-back, providing ~20% faster execution and preventing deadlocks in blocking calls. ASP.NET Core has no `SynchronizationContext`, so `ConfigureAwait(false)` is effectively the default.
- **AsyncLocal\<T\>:** Flows ambient data (like request IDs) across async continuations. Copies logical call context on every await. Useful for distributed tracing and correlation IDs.

```csharp
public async Task<T> RetryAsync<T>(Func<Task<T>> operation, int maxRetries = 3, CancellationToken ct = default)
{
    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        try { return await operation().ConfigureAwait(false); }
        catch (Exception ex) when (attempt < maxRetries && IsTransient(ex))
        {
            await Task.Delay(TimeSpan.FromMilliseconds(Math.Pow(2, attempt) * 100), ct).ConfigureAwait(false);
        }
    }
    return await operation().ConfigureAwait(false);
}
```

---

## Common Mistakes

- **`async void` (except for event handlers)** — Exceptions thrown crash the process because there's no `Task` to observe the error. Always use `async Task` instead.
- **Blocking on async code** — Using `.Result` or `.Wait()` causes deadlocks when a `SynchronizationContext` is involved. Always await all the way up.
- **Missing `ConfigureAwait(false)` in library code** — Captures context unnecessarily, causing performance overhead and potential deadlocks.
- **Sequential when parallel** — Awaiting tasks one after another instead of using `Task.WhenAll` for independent operations.
- **No cancellation support** — Methods that don't accept `CancellationToken` cannot be cancelled, causing resource leaks and poor user experience.
- **Forgetting to dispose `CancellationTokenSource`** — Can cause memory leaks if not disposed properly. Use `using` or `cts.Dispose()`.
- **Async in synchronous wrapper** — Writing `public async Task<int> ComputeAsync() { return ComputeSync(); }` adds state machine overhead for no benefit. Return `Task.FromResult` directly.

```csharp
// Problem: async over sync
public async Task<int> ComputeAsync() { return ComputeSync(); } // Bad
// Fix: Task.FromResult
public Task<int> ComputeAsync() { return Task.FromResult(ComputeSync()); }
```

- **Not observing task exceptions** — Fire-and-forget tasks can throw unobserved exceptions, triggering `TaskScheduler.UnobservedTaskException`. Always attach continuations or await tasks.

---

## Key Design Considerations

- **Async all the way up** — Never block on async code. The `Task` returned from an async method propagates up the call stack. Blocking defeats the purpose of async.
- **Avoid `async` when not needed** — Use `Task.FromResult`, `Task.CompletedTask`, `Task.FromException` for synchronous implementations of async interfaces.
- **`ConfigureAwait(false)` in library code, rarely in application code** — Library code should not need to resume on a specific context; application code (UI) needs the context.
- **Pooling patterns** — Use `ValueTask<T>` with `IValueTaskSource<T>` for high-throughput scenarios to minimize allocations. Consider `ObjectPool<T>` for reusable buffers.
- **Async stream processing** — Use `Channel<T>` with `IAsyncEnumerable<T>` for producer-consumer pipelines with backpressure.
- **Exception handling** — `AggregateException` is thrown only with `Task.Wait`, not `await` (which unwraps the first exception). Use `try/catch` naturally.
- **Cooperative cancellation** — Pass `CancellationToken` to all async methods. Register cleanup callbacks via `ct.Register()`.

```csharp
await using var registration = ct.Register(() => _connection.Close());
```

---

## Real-World Scenarios

### Scenario 1: High-Throughput API Gateway with Rate Limiting
**Context:** An API gateway processing 50K requests/second needs per-tenant rate limiting without blocking threads.

```csharp
public class RateLimitingMiddleware
{
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _rateLimiters = new();
    private readonly int _maxConcurrent;
    private readonly TimeSpan _timeout;

    public async Task<T> ExecuteAsync<T>(string tenantId, Func<Task<T>> operation, CancellationToken ct)
    {
        var semaphore = _rateLimiters.GetOrAdd(tenantId, _ => new SemaphoreSlim(_maxConcurrent));
        
        if (!await semaphore.WaitAsync(_timeout, ct).ConfigureAwait(false))
            throw new RateLimitExceededException($"Tenant {tenantId} exceeded rate limit");
        
        try
        {
            return await operation().ConfigureAwait(false);
        }
        finally
        {
            semaphore.Release();
        }
    }
}
```

### Scenario 2: Async Producer-Consumer Pipeline with Backpressure
**Context:** A log processing system reads from a high-volume source, transforms data, and writes to a database. Need backpressure to prevent memory overflow.

```csharp
public class LogPipeline
{
    private readonly Channel<LogEntry> _channel = Channel.CreateBounded<LogEntry>(
        new BoundedChannelOptions(10_000) { FullMode = BoundedChannelFullMode.Wait });

    public async Task ProduceAsync(IAsyncEnumerable<LogEntry> source, CancellationToken ct)
    {
        await foreach (var entry in source.WithCancellation(ct))
            await _channel.Writer.WriteAsync(entry, ct);
        _channel.Writer.Complete();
    }

    public async Task ConsumeAsync(CancellationToken ct)
    {
        var batch = new List<LogEntry>(100);
        await foreach (var entry in _channel.Reader.ReadAllAsync(ct))
        {
            batch.Add(entry);
            if (batch.Count >= 100)
            {
                await BulkInsertAsync(batch, ct);
                batch.Clear();
            }
        }
        if (batch.Count > 0) await BulkInsertAsync(batch, ct);
    }

    private async Task BulkInsertAsync(List<LogEntry> batch, CancellationToken ct)
    {
        using var conn = new SqlConnection(_connectionString);
        await conn.OpenAsync(ct);
        // Bulk insert logic
    }
}
```

### Scenario 3: Graceful Shutdown with Cancellation Propagation
**Context:** A microservice must shut down within 5 seconds when Kubernetes sends SIGTERM, preserving in-flight work.

```csharp
public class GracefulShutdownHostedService : IHostedService
{
    private readonly ILogger _logger;
    private CancellationTokenSource _shutdownCts = new();

    public async Task StartAsync(CancellationToken ct)
    {
        _ = RunMainLoopAsync(_shutdownCts.Token);
        await Task.CompletedTask;
    }

    private async Task RunMainLoopAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                var workItem = await _workQueue.DequeueAsync(ct);
                await ProcessAsync(workItem, ct);
            }
            catch (OperationCanceledException) when (ct.IsCancellationRequested)
            {
                _logger.LogInformation("Shutdown requested, draining remaining items...");
                await DrainRemainingWorkAsync(TimeSpan.FromSeconds(3));
                break;
            }
        }
    }

    public async Task StopAsync(CancellationToken ct)
    {
        _logger.LogInformation("Initiating graceful shutdown...");
        _shutdownCts.Cancel();
        
        // Give in-flight work 5 seconds to complete
        await Task.Delay(TimeSpan.FromSeconds(5), ct).ContinueWith(_ => { });
        _shutdownCts.Dispose();
    }
}
```

---

## Scenario-Based Questions

1. **Q: You are building a web API that calls three external services. Each service is independent and fails ~5% of the time. How do you handle this efficiently and resiliently?**
   A: Use `Task.WhenAll` to parallelize the three calls. Wrap each in a try/catch that returns a fallback value on failure. Apply a timeout per call via `CancellationTokenSource.CreateLinkedTokenSource` with a per-task timeout. Use a circuit breaker pattern (e.g., `Polly`) for each service to avoid hammering failing services. Consider `ConfigureAwait(false)` to avoid context capture. Edge case: one service timeout should not cancel the others — use independent CTS per service.

2. **Q: You have an ASP.NET Core endpoint that uploads large files (500MB+). How do you avoid memory exhaustion and thread starvation?**
   A: Stream the request body directly to disk using `Request.Body.CopyToAsync(fileStream)` without buffering in memory. Disable request body buffering. Use `Stream.PipeReader` for zero-copy processing. Set `IISClientMaxRequestBodySize` and `KestrelLimits.MaxRequestBodySize`. For antivirus scanning, chain `Stream` decorators. Never read the entire file into a `byte[]` — this causes LOH fragmentation and GC pressure.

3. **Q: You are designing a background job processor that handles 1000 jobs/second. Each job does I/O with some CPU work. How do you maximize throughput?**
   A: Use `Channel<T>` as a bounded producer-consumer with multiple concurrent consumers. Set `BoundedChannelFullMode.Wait` for backpressure. Use `ValueTask` for frequently synchronous operations (cache checks). Pin consumer count to `Environment.ProcessorCount * 2` for I/O-heavy workloads. Use `ConfigureAwait(false)` throughout. Monitor channel count and apply dynamic scaling. For CPU-bound portions, offload to `Task.Run` sparingly.

4. **Q: You are implementing an async cache-aside pattern. How do you prevent the thundering herd problem when a popular cache key expires?**
   A: Use `AsyncLazy<T>` or `SemaphoreSlim(1,1)` per key to serialize cache regeneration. When the cache misses, the first caller acquires the semaphore and regenerates; subsequent callers await the same operation. Use `ConcurrentDictionary<string, SemaphoreSlim>` for per-key locking. Consider `IDistributedCache` with `GetOrCreateAsync` pattern. Trade-off: complexity vs. reduced database load. Edge case: handle semaphore disposal on key eviction.

5. **Q: You need to call a paginated REST API that returns 10K pages. How do you process results concurrently with bounded parallelism?**
   A: Use `Parallel.ForEachAsync` (.NET 6+) with `MaxDegreeOfParallelism` set to a sensible concurrency limit (e.g., 10). Alternatively, use a `SemaphoreSlim` to throttle `Task.WhenAll`. Implement page enumeration as an `IAsyncEnumerable<int>` yielding page numbers. Each page fetch and processing is independent. Handle retries per page with exponential backoff. Cancel remaining fetches on first unrecoverable error.

6. **Q: You are building a real-time dashboard that polls 10 data sources every 5 seconds. How do you implement the polling loop without drift?**
   A: Use a `PeriodicTimer` (.NET 6+) which avoids drift (unlike `Task.Delay` in a loop — drift accumulates). Start all 10 polls concurrently with `Task.WhenAll`. Use a `CancellationTokenSource` tied to the application lifetime. If a poll takes longer than 5 seconds, log a warning but don't skip the next interval — `PeriodicTimer` handles this naturally. For time-sensitive data, consider `IAsyncEnumerable<T>` with `Channel<T>`.

7. **Q: You are implementing an async lock for a distributed system. How does it differ from a single-process `SemaphoreSlim`?**
   A: Single-process `SemaphoreSlim` is per-instance — two servers cannot coordinate. For distributed locking, use `Redis` (`RedLock` algorithm via `StackExchange.Redis`), Azure Blob lease, or database `sp_getapplock`. Distributed locks need lease expiration (to handle crashed holders), fencing tokens (to prevent delayed requests), and clock drift tolerance. Never assume a distributed lock is perfectly reliable — design for the case where two nodes both believe they hold the lock.

8. **Q: You are processing a stream of 1M events from Kafka. Each event requires an async DB write. How do you batch efficiently?**
   A: Accumulate events into a channel or buffer. When the buffer reaches a threshold (e.g., 100 events) or a time window elapses (e.g., 100ms), flush as a batch using `SqlBulkCopy` or `EF Core ExecuteUpdate`. Use `Channel<T>` with a `BatchAsync` extension that yields batches. For exactly-once semantics, use idempotency keys. For at-least-once, track offsets and commit after successful batch. Handle partial batch failures by retrying individual items.

9. **Q: You have a legacy synchronous library that can't be made async. How do you integrate it into an async codebase without thread pool starvation?**
   A: Offload synchronous calls to a dedicated thread (not the thread pool) using `Task.Run` with a custom `TaskScheduler` that has a limited number of dedicated threads. Use `SemaphoreSlim` to cap concurrency. Never call `.Result` or `.Wait()` — this ties up a thread pool thread AND blocks, doubling the damage. Consider `IOPriority` hints on Windows. Measure: if the library calls are short (< 50ms), they may be acceptable on the thread pool with limited parallelism.

10. **Q: You are debugging an ASP.NET app that hangs under load. Deadlock is suspected. How do you diagnose and fix it?**
    A: Capture a memory dump (or use `dotnet-dump`), load in WinDbg or `dotnet-dump analyze`, and run `!syncblk` to find blocked threads. Look for threads waiting on `Monitor.Enter` while holding another lock. Common pattern: blocking on async code (`.Result`/`.Wait()`) in a UI context. Fix: use `ConfigureAwait(false)` in library code, avoid blocking calls entirely, and ensure async-all-the-way-up. Also check `ThreadPool` starvation via `ThreadPool.GetAvailableThreads` metrics.

---

## Interview Questions

1. **What is the difference between `Task` and `ValueTask<T>`?** 
   A: `Task<T>` is a reference type (~80 bytes heap allocated) that can be awaited multiple times. `ValueTask<T>` is a value type that avoids allocation when the result is synchronous. `ValueTask<T>` can only be awaited once and cannot be used with `WhenAll`/`WhenAny` without calling `.AsTask()`.

2. **What does `ConfigureAwait(false)` do?**
   A: It tells the runtime not to marshal the continuation back to the original `SynchronizationContext` or `TaskScheduler`. This avoids context switch overhead and prevents deadlocks when blocking on async code. Library code should use it; UI application code should not.

3. **How does the compiler transform an `async` method?**
   A: The compiler generates a struct implementing `IAsyncStateMachine` with a `_state` field, captured locals as fields, and a `MoveNext()` method using a `switch` statement. The state machine is boxed only when an incomplete task is awaited.

4. **What is `AsyncLocal<T>` used for?**
   A: It stores ambient data that flows across async continuations via `ExecutionContext.Copy()`. Common uses: correlation IDs for distributed tracing, transaction scopes, and `ILogger` scopes.

5. **Explain the difference between `Task.WhenAll` and `Task.WhenAny`.**
   A: `WhenAll` returns a task that completes when ALL provided tasks complete — the result is an array of all results. `WhenAny` returns when ANY task completes — the result is the first completed task. Use `WhenAll` for fan-out parallelism; use `WhenAny` for timeouts or first-response-wins patterns.

6. **What causes an `async void` method to crash the process?**
   A: Exceptions thrown in `async void` methods cannot be caught because there is no `Task` to observe the exception. The exception is re-thrown on the `SynchronizationContext`, which typically crashes the process. Always use `async Task` except for event handlers.

7. **How does `IAsyncEnumerable<T>` differ from `IEnumerable<T>`?**
   A: `IAsyncEnumerable<T>` supports asynchronous iteration with `await foreach`. Each element can be fetched asynchronously via `MoveNextAsync()` returning `ValueTask<bool>`. It supports cancellation via `WithCancellation()` and integrates with `Channel<T>`.

8. **What is the `TaskCompletionSource<T>` pattern?**
   A: It creates a `Task<T>` that you manually control by calling `SetResult()`, `SetException()`, or `SetCanceled()`. Used for bridging legacy async patterns (APM/EAP) to TAP and for implementing custom async primitives like `AsyncLock` or `ValueTask<T>` sources.

9. **How does `SemaphoreSlim.WaitAsync` prevent thread pool starvation?**
   A: Unlike `Monitor.Enter` (which blocks the thread), `WaitAsync` returns a `Task` that completes when the semaphore is acquired. The calling thread is freed to process other work items during the wait, preventing thread pool exhaustion.

10. **What is the async state machine boxing behavior?**
    A: The state machine struct is boxed (heap allocated) only when `MoveNext()` is called asynchronously — i.e., when an awaited task is incomplete. If all awaited tasks complete synchronously, the struct stays on the stack and no boxing occurs. This is the "fast path" optimization.

---

## Developer Recommendations

- **Prefer `ConfigureAwait(false)` in library code, omit it in application code** — Library code should not depend on a specific `SynchronizationContext`. Adding `ConfigureAwait(false)` provides ~20% throughput improvement and prevents deadlocks. In UI application code, the context is needed for thread affinity (e.g., updating controls).

- **Use `CancellationToken` in all async method signatures** — Without it, callers cannot cancel in-flight operations, leading to resource leaks and poor UX. Always pass `CancellationToken` through the call chain. Use `CancellationToken.None` as a default if the method's callers rarely need cancellation.

- **Avoid `async void` except for event handlers** — `async void` exceptions crash the process. Return `Task` instead so errors are observable. For top-level event handlers (button clicks), wrap the body in a try/catch to log exceptions.

- **Use `Channel<T>` for producer-consumer over `BlockingCollection<T>`** — `Channel<T>` is async-first, supports backpressure via bounded capacity, integrates with `IAsyncEnumerable<T>`, and has lower overhead. `BlockingCollection<T>` blocks threads, which is antithetical to async patterns.

- **Prefer `Parallel.ForEachAsync` over `Task.WhenAll` + manual throttling** — `Parallel.ForEachAsync` (.NET 6+) provides built-in `MaxDegreeOfParallelism` without requiring manual `SemaphoreSlim` management. It handles cancellation and is optimized for async workloads.

- **Measure before optimizing async allocation** — The async state machine allocation (~80-120 bytes) is negligible for most workloads. Prematurely replacing `Task<T>` with `ValueTask<T>` adds complexity and constraints. Profile with BenchmarkDotNet to identify true bottlenecks.

- **Never block on async code with `.Result` or `.Wait()`** — This causes thread pool starvation and deadlocks in contexts with `SynchronizationContext`. Use `await` all the way up. If you absolutely must block (e.g., console app `Main`), use `GetAwaiter().GetResult()` or switch to `await Main` (C# 7.1+).

- **Use `TaskCompletionSource` carefully** — Only one thread should call `SetResult`/`SetException`. Calling it multiple times throws. Use `TrySetResult` for patterns where completion might race. Consider `TaskCompletionSource<T>.RunContinuationsAsynchronously` to avoid running continuations on the completing thread.

---

## Key Design Considerations (Additional)

- **Async method inlining (.NET 6+):** The runtime can elide state machine allocation for simple async methods that are frequently called. Use `[MethodImpl(MethodImplOptions.NoInlining)]` if problematic.
- **Avoid async void:** Only for event handlers. All other async methods must return `Task`, `Task<T>`, or `ValueTask<T>`.
- **ValueTask restrictions:** Cannot be awaited multiple times, cannot be used with `Task.WhenAll`/`WhenAny` without calling `.AsTask()`, and cannot block with `.Result`.
- **Channel\<T\> for producer-consumer:** Async-compatible, bounded with backpressure, supports `IAsyncEnumerable<T>` via `ReadAllAsync()`.

---

## Comparison Summary

- **async/await vs BackgroundWorker:** async/await uses continuation-based state machines (no thread during wait) vs event-based pattern; async/await has natural error handling via try/catch; async/await supports composition via `WhenAll`/`WhenAny`.
- **Task vs ValueTask:** Reference type (always allocates) vs value type (allocates only when wrapping a Task); multiple await supported only for Task; ValueTask best for synchronous hot paths.
- **IAsyncEnumerable vs IEnumerable:** Supports per-element async operations, `await foreach`, and cancellation for backpressure.
