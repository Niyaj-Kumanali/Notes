# Async-Await

## 1. Executive Summary

Async/await is C#'s language-level asynchronous programming model introduced in C# 5.0 and .NET Framework 4.5. It enables non-blocking I/O operations using a synchronous-looking code pattern while the runtime manages continuations via a state machine. Async/await is not about parallelism — it's about freeing threads during I/O wait, improving scalability (especially for server applications) and UI responsiveness.

## 2. Core Theory

### Key Concepts

- **`async` keyword**: Marks a method as asynchronous. Enables `await` inside it. Does NOT create a new thread.
- **`await` keyword**: Suspends execution of the method until the awaited task completes. Returns control to the caller.
- **`Task` / `Task<T>`**: Represent an asynchronous operation. The "promise" of a future value.
- **`ValueTask<T>`**: Value-type variant to reduce allocation in hot paths.
- **`IAsyncEnumerable<T>`**: Async streaming (C# 8.0+), supports `await foreach`.

### Control Flow

```csharp
async Task<string> ReadFileAsync(string path)
{
    // Thread A (synchronous part)
    byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);

    using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
        FileShare.Read, 4096, FileOptions.Asynchronous);

    // Thread A: starts async I/O, returns Task (not yet completed)
    // If I/O completes synchronously (data in cache), continues on Thread A
    // Otherwise, Thread A returns to caller; continuation runs on ThreadPool
    int bytesRead = await fs.ReadAsync(buffer, 0, buffer.Length);

    // Continuation: could be on ThreadPool, or synchronized with SynchronizationContext
    string result = Encoding.UTF8.GetString(buffer, 0, bytesRead);
    ArrayPool<byte>.Shared.Return(buffer);
    return result;
}
```

### Async Method Signatures

```csharp
// Valid return types:
async Task MyMethodAsync();           // Fire-and-forget (void equivalent)
async Task<int> MyMethodAsync();      // Returns a value
async ValueTask<int> MyMethodAsync(); // Hot-path, avoid allocation
async IAsyncEnumerable<int> MyMethodAsync(); // Stream items (C# 8+)
async void MyMethodAsync();           // ONLY for event handlers (dangerous otherwise)

// Pattern: Task-like types (custom awaitables)
// Any type with GetAwaiter()/IsCompleted/GetResult() can be awaited
```

## 3. Under-the-Hood Deep Dive

### State Machine Transformation

```csharp
// Source:
async Task<int> ExampleAsync(int a, int b)
{
    int result = await Task.Run(() => a + b);
    return result * 2;
}

// Compiler generates (simplified):
[StructLayout(LayoutKind.Auto)]
internal struct ExampleAsyncStateMachine : IAsyncStateMachine
{
    public int _a;
    public int _b;
    public int _result;
    public int _localResult;
    public TaskAwaiter<int> _awaiter;
    public int _state;

    void IAsyncStateMachine.MoveNext()
    {
        try
        {
            switch (_state)
            {
                case 0:
                    _awaiter = Task.Run(() => _a + _b).GetAwaiter();
                    if (!_awaiter.IsCompleted)
                    {
                        _state = 1;
                        _awaiter.UnsafeOnCompleted(this.MoveNext);
                        return; // Yield to caller
                    }
                    goto case 1;
                case 1:
                    _result = _awaiter.GetResult();
                    _localResult = _result * 2;
                    break;
            }
        }
        catch (Exception ex)
        {
            _taskCompletionSource.TrySetException(ex);
            return;
        }
        _taskCompletionSource.TrySetResult(_localResult);
    }
}
```

### SynchronizationContext and ConfigureAwait

```csharp
// By default, after await, the continuation runs on the original SynchronizationContext
// (e.g., UI thread in WPF/WinForms, ASP.NET request context)

// Every await has an implicit "post back" to SynchronizationContext:
await SomeAsyncMethod();  // continuation on original context

// ConfigureAwait(false) says "I don't need to resume on the original context"
await SomeAsyncMethod().ConfigureAwait(false);
// ~20% faster (no context switch), prevents deadlocks in blocking calls

// Which contexts exist:
// - WindowsFormsSynchronizationContext (UI thread)
// - DispatcherSynchronizationContext (WPF)
// - AspNetSynchronizationContext (ASP.NET Classic, .NET Framework)
// - ASP.NET Core: no context (ConfigureAwait(false) is default behavior)
// - Default: ThreadPool (TaskScheduler.Default)
```

### Task Allocation Optimization

```csharp
// Task.CompletedTask: cached completed task for void returns
// Task.FromResult<T>(value): cached for common results (bool true/false, 0, 1, etc.)
// ValueTask<T>: avoids Task allocation when result is often synchronous

// Example:
public ValueTask<int> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out int value))
        return new ValueTask<int>(value); // No Task allocation!

    return new ValueTask<int>(LoadFromDbAsync(key)); // Falls back to Task
}
```

### AsyncLocal<T>

```csharp
// Flows ambient data across async continuations (like CallContext)
AsyncLocal<Guid> _requestId = new();

async Task ProcessAsync()
{
    _requestId.Value = Guid.NewGuid();
    await Step1Async(); // _requestId.Value flows to Step1
    await Step2Async(); // still has the same value
}

// Implementation: copies the LogicalCallContext across async continuations
```

## 4. Production Code Examples

```csharp
// Async retry with exponential backoff
public async Task<T> RetryAsync<T>(
    Func<Task<T>> operation,
    int maxRetries = 3,
    CancellationToken ct = default)
{
    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        try
        {
            return await operation().ConfigureAwait(false);
        }
        catch (Exception ex) when (attempt < maxRetries && IsTransient(ex))
        {
            var delay = TimeSpan.FromMilliseconds(Math.Pow(2, attempt) * 100);
            await Task.Delay(delay, ct).ConfigureAwait(false);
        }
    }
    return await operation().ConfigureAwait(false); // Last attempt
}
```

```csharp
// Concurrent async operations with throttling
public async Task<List<TResult>> ProcessConcurrentlyAsync<T, TResult>(
    IEnumerable<T> items,
    Func<T, CancellationToken, Task<TResult>> processor,
    int maxConcurrency,
    CancellationToken ct = default)
{
    var semaphore = new SemaphoreSlim(maxConcurrency);
    var tasks = items.Select(async item =>
    {
        await semaphore.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            return await processor(item, ct).ConfigureAwait(false);
        }
        finally
        {
            semaphore.Release();
        }
    });

    return (await Task.WhenAll(tasks).ConfigureAwait(false)).ToList();
}
```

```csharp
// Async producer-consumer with Channel<T>
public class AsyncEventBus
{
    private readonly Channel<Event> _channel = Channel.CreateBounded<Event>(
        new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false
        });

    public async Task PublishAsync(Event e, CancellationToken ct)
    {
        await _channel.Writer.WriteAsync(e, ct);
    }

    public async IAsyncEnumerable<Event> SubscribeAsync(
        [EnumeratorCancellation] CancellationToken ct)
    {
        await foreach (var e in _channel.Reader.ReadAllAsync(ct))
            yield return e;
    }
}
```

```csharp
// Async factory pattern
public class AsyncResource : IAsyncDisposable
{
    private readonly FileStream _stream;

    private AsyncResource(FileStream stream)
    {
        _stream = stream;
    }

    public static async Task<AsyncResource> CreateAsync(string path)
    {
        var stream = await File.OpenReadAsync(path);
        return new AsyncResource(stream);
    }

    public async ValueTask DisposeAsync()
    {
        if (_stream != null)
            await _stream.DisposeAsync();
    }
}

// Usage:
await using var resource = await AsyncResource.CreateAsync("data.bin");
```

```csharp
// Async LINQ operators
public static class AsyncEnumerableExtensions
{
    public static async IAsyncEnumerable<TResult> SelectAwait<T, TResult>(
        this IAsyncEnumerable<T> source,
        Func<T, Task<TResult>> selector)
    {
        await foreach (var item in source)
            yield return await selector(item);
    }

    public static async IAsyncEnumerable<T> WhereAwait<T>(
        this IAsyncEnumerable<T> source,
        Func<T, Task<bool>> predicate)
    {
        await foreach (var item in source)
            if (await predicate(item))
                yield return item;
    }
}
```

## 5. Real-World Scenarios

**Scenario 1: ASP.NET Core Web API**
- Every controller action is `async Task<IActionResult>`.
- Use `IAsyncEnumerable<T>` for streaming large responses.
- DB queries via `EF Core` async methods.

**Scenario 2: File Processing Pipeline**
- `await foreach` over lines of a large file.
- Process in batches with `Channel<T>`.
- Write results asynchronously.

**Scenario 3: Microservice Orchestration**
- `Task.WhenAll` to call multiple downstream services.
- `Task.WhenAny` for race patterns (fastest response wins).
- Circuit breaker with async retry.

**Scenario 4: Real-Time UI (WPF/WinForms)**
- `async void` for event handlers (button clicks).
- `Progress<T>` and `IProgress<T>` for reporting progress to UI.
- `ConfigureAwait(true)` (default) to resume on UI thread.

## 6. Performance

```csharp
// Task allocation cost:
// - Task<T> : ~80 bytes (heap allocated)
// - ValueTask<T> : struct (stack, usually)
// - Task completion source with result: Task<T> + TCS (~200 bytes)

// Benchmark: 1M async calls
//   Task:       ~80ms, 80MB allocated
//   ValueTask:  ~40ms, 20MB allocated (when synchronous)

// Avoid async overhead on hot synchronous paths:
public Task<int> GetCountAsync()
{
    if (_cache.TryGetValue("count", out int count))
        return Task.FromResult(count); // Fast path: cached Task
    return GetCountFromDbAsync(); // Slow path: real async
}

// Or with ValueTask:
public ValueTask<int> GetCountAsync()
{
    if (_cache.TryGetValue("count", out int count))
        return new ValueTask<int>(count);
    return new ValueTask<int>(GetCountFromDbAsync());
}
```

### Async Method Inlining

```csharp
// .NET 6+: the runtime can inline async methods and elide state machine allocation
// Requires: method must be simple and frequently called
// Use NoInline attribute to prevent if problematic
```

### Avoiding Async Allocations

```csharp
// 1. Use ValueTask<T> for hot paths
// 2. Cache completed tasks: Task.CompletedTask, Task.FromResult(0)
// 3. Use IValueTaskSource<T> for pooling (advanced)
// 4. Avoid async void (creates Task, but exceptions crash process)
// 5. Use ConfigureAwait(false) in library code
```

## 7. Security

```csharp
// AsyncLocal information disclosure
AsyncLocal<string> _ambientUser = new();
_ambientUser.Value = GetSensitiveData();

async Task LeakAsync()
{
    // If the continuation runs on a different thread (ConfigureAwait(false)),
    // AsyncLocal data might flow to unexpected places
    await Task.Yield();
    string leaked = _ambientUser.Value; // Still accessible
}
// FIX: Always scope AsyncLocal carefully

// Cancel async operations to avoid resource leaks
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
try
{
    await ProcessAsync(cts.Token);
}
catch (OperationCanceledException)
{
    // Handle cancellation gracefully
}

// Avoid TaskCompletionSource.SetResult race conditions
// if (!tcs.TrySetResult(value)) { /* already completed */ }
```

## 8. Common Mistakes

```csharp
// MISTAKE 1: async void (except for event handlers)
async void FireAndForget()
{
    await Task.Delay(1000);
    throw new Exception("This crashes the process!"); // Unhandled!
}
// FIX: async Task, and handle errors

// MISTAKE 2: Blocking on async code
var result = GetDataAsync().Result; // Deadlock (if single-threaded context)
var result = GetDataAsync().GetAwaiter().GetResult(); // Still blocks thread
// FIX: await all the way up; or use .ConfigureAwait(false) if you must block

// MISTAKE 3: Missing ConfigureAwait(false) in library code
public async Task<string> ReadAsync()
{
    using var fs = File.OpenRead("data.txt");
    using var sr = new StreamReader(fs);
    return await sr.ReadToEndAsync(); // Captures context unnecessarily
}
// FIX: return await sr.ReadToEndAsync().ConfigureAwait(false);

// MISTAKE 4: Combining async with Wait/Result
Task t = DoWorkAsync();
t.Wait(); // Thread pool thread blocked!
// FIX: use await

// MISTAKE 5: No cancellation support
public async Task ProcessAsync() // Can't cancel!
{
    while (true) { await Task.Delay(1000); }
}
// FIX: accept CancellationToken

// MISTAKE 6: async in a using block
using var fs = File.OpenRead(path);
var data = await fs.ReadAsync(buffer); // fs might be disposed before continuation
// FIX: it's actually safe (using disposes on original context after await completes)

// MISTAKE 7: Sequential when you could parallelize
var r1 = await GetData1Async();  // Waits for this
var r2 = await GetData2Async();  // Then starts this
// FIX: Task.WhenAll(GetData1Async(), GetData2Async())

// MISTAKE 8: Capturing SynchronizationContext unnecessarily
// Especially in ASP.NET Core (no context), ConfigureAwait doesn't matter

// MISTAKE 9: Forgetting to dispose CancellationTokenSource
var cts = new CancellationTokenSource();
// ... use cts.Token
// FIX: using var cts = new CancellationTokenSource();

// MISTAKE 10: Mixing sync and async in custom Task-returning methods
public async Task<int> ComputeAsync()
{
    return ComputeSync(); // No await needed; the async keyword wraps in Task
}
// FIX: return Task.FromResult(ComputeSync()) without async
```

## 9. Senior Engineer Perspective

**1. Async ALL the way up.** The `Task` returned from an async method propagates up. Never block.

**2. Avoid `async` when not needed.** `Task.FromResult`, `Task.CompletedTask` for sync implementations:

```csharp
public Task SaveAsync(Data data)
{
    if (data is null) return Task.FromException(new ArgumentNullException());
    return SaveCoreAsync(data);
}
```

**3. Use `ConfigureAwait(false)` in library code, rarely in application code** (where context matters).

**4. Pooling patterns:** `ObjectPool<ValueTask>` or `IValueTaskSource<T>` for high-throughput scenarios.

**5. Async stream processing** via `IAsyncEnumerable<T>` with `Channel<T>` for producer-consumer.

**6. Exception handling:** AggregateException is thrown only on `Task.Wait`, not `await` (which unwraps the first exception).

**7. Cooperative cancellation** via `CancellationToken.Registration` for cleanup:

```csharp
await using var registration = ct.Register(() =>
{
    // Cleanup on cancellation
    _connection.Close();
});
```

## 10. Interview Questions (Easy)

1. What does the `async` keyword do?
2. What does the `await` keyword do?
3. What is the difference between `Task` and `Task<T>`?
4. What return types can an async method have?
5. What is `ConfigureAwait(false)` used for?
6. What is the difference between `Task.Run` and `async`/`await`?
7. How do you run multiple async operations in parallel?
8. What is `Task.WhenAll`?
9. What is `Task.WhenAny`?
10. What is `CancellationToken` used for?

## 11. Interview Questions (Medium)

1. Explain how the compiler transforms an async method (state machine).
2. What is `SynchronizationContext` and how does it affect async continuations?
3. When should you use `ValueTask<T>` instead of `Task<T>`?
4. Explain the difference between `Task.Delay` and `Thread.Sleep`.
5. Why is `async void` dangerous?
6. What is `AsyncLocal<T>` and when would you use it?
7. How does `await foreach` work (what interfaces are needed)?
8. What happens when an exception is thrown in an async method?
9. Explain the difference between `TaskCreationOptions.LongRunning` and a regular task.
10. What is `Yield()` and when would you use it?

## 12. Advanced Interview Questions (Hard)

1. Implement a custom awaitable type that pools its continuation state.
2. Explain the `IValueTaskSource<T>` interface and design a pooled `ValueTask<T>` source.
3. Design an async-friendly lock (AsyncLock) using `SemaphoreSlim` or channel.
4. How does the `ConfigureAwait` decision impact deadlock potential?
5. Implement an async `Lazy<T>` that prevents multiple simultaneous initializations.
6. Explain `ExecutionContext` flow vs `SynchronizationContext` flow.
7. Design an async-friendly producer-consumer pipeline with backpressure.
8. Implement `WhenAll` with concurrency limit from scratch.
9. Explain why `Task<T>` cannot be made a value type (history and constraints).
10. Design an async polling mechanism that doubles interval on failure (exponential backoff).

## 13. Interview Questions (System Design)

1. Design an async-first web server request pipeline.
2. Design a distributed async job queue with persistence and retry.
3. Design an async-first in-memory cache with background refresh.
4. Design an async file watcher that processes new files in order.
5. Design a message broker with async publish/subscribe guarantees.
6. Design an async circuit breaker for microservice calls.
7. Design an async rate limiter using token bucket algorithm.
8. Design a real-time chat system using async streams.
9. Design a streaming ETL pipeline using `IAsyncEnumerable` and channels.
10. Design an async health check aggregation system for microservices.

## 14. Expert-Level Interview Questions (Architect)

1. Design a fully async query execution engine for a database that uses `IValueTaskSource<T>` pooling to serve 100K queries/second with zero allocation.
2. Architect a distributed tracing system that flows trace context through `AsyncLocal<T>` across process boundaries via gRPC headers, handling nested spans correctly.
3. Design a cooperative async scheduler that implements priority scheduling and work stealing for a game engine's task system.
4. Architect a high-throughput async RPC framework that avoids all synchronous blocking in the hot path, using pipes and channels for zero-copy serialization.
5. Design an actor framework where actor mailboxes are fully async, with backpressure and supervision, using `Channel<T>` and `Task` composition.
6. Architect an async stream processing engine with exactly-once semantics, checkpointing, and state persistence.
7. Design a deadlock-detection and recovery system for async code that uses timeout-based locks with escalation.
8. Architect an async-first ORM that supports lazy loading, batching, and concurrent query execution without blocking.
9. Design a reactive system following the Reactive Manifesto using `IObservable<T>` and async pipelines with backpressure.
10. Architect a language-extended async profiler that tracks state machine allocations and synchronization context switches using diagnostic sources.

## 15. Debugging & Troubleshooting

```csharp
// Enable async diagnostics for stack traces:
// <AppContextSwitchOverrides value="Switch.System.Runtime.Serialization.UseNewAsyncSerialization=true"/>

// Look for "async" in call stacks:
// - State machine types: <MethodName>d__0
// - "MoveNext" frames indicate async state machine

// Common issues:
// - Deadlock: sync context blocked waiting for async result
//   -> Use ConfigureAwait(false) in library code
// - Task not observed: TaskScheduler.UnobservedTaskException
//   -> Always await tasks or attach continuations
// - Fire-and-forget with exceptions:
//   -> Use a "safe fire-and-forget" extension with logging

// Tools:
// - dotnet-trace: captures async activity
// - PerfView: "Async" group in "Thread Time" stacks
// - Visual Studio: Task Debugger (Parallel Tasks, Threads windows)
```

## 16. Comparison Section

```
+------------------------+------------------------+------------------------+
| Feature                | async/await            | BackgroundWorker       |
+------------------------+------------------------+------------------------+
| Pattern                | Continuation (state m) | Event-based            |
| Return value           | Task<T>                | RunWorkerCompleted     |
| Cancellation           | CancellationToken      | CancellationPending    |
| Progress reporting     | IProgress<T>           | ProgressChanged        |
| Composition            | await, WhenAll, WhenAny| Manual chaining        |
| Error handling         | try/catch (natural)    | RunWorkerCompleted     |
| Allocation             | State machine + Task   | Events + args          |
+------------------------+------------------------+------------------------+

+------------------------+------------------------+------------------------+
| Feature                | Task<T>                | ValueTask<T>           |
+------------------------+------------------------+------------------------+
| Type                   | Reference (class)      | Value (struct)         |
| Allocation             | Always (~80 bytes)     | None if sync result    |
| Multiple await         | Yes (cached result)    | No (once only)         |
| Blocking wait          | .Result, .Wait()      | .Result, .AsTask()     |
| Pooling                | Not easily             | IValueTaskSource<T>    |
| Best for               | General async          | Hot paths, sync results|
+------------------------+------------------------+------------------------+

+------------------------+------------------------+------------------------+
| Feature                | IAsyncEnumerable<T>    | IEnumerable<T>         |
+------------------------+------------------------+------------------------+
| Enumeration            | await foreach         | foreach                |
| Async operations       | Per-element async      | Sync only              |
| Backpressure           | Via CancellationToken  | None                   |
| Allocation             | State machine per iter | State machine per iter  |
| DB mapping             | EF Core, Dapper        | Any                    |
+------------------------+------------------------+------------------------+
```

## 17. Revision Notes

- `async` enables `await`; it does NOT create a new thread.
- The compiler generates a state machine struct from async methods.
- `await` yields control to caller if the task is not yet completed.
- `ConfigureAwait(false)` skips returning to original `SynchronizationContext`.
- `ValueTask<T>` reduces allocation when result is often synchronous.
- `async void` is only for event handlers (exceptions crash process).
- Never block on async code with `.Result` or `.Wait()` (deadlock risk).
- Use `CancellationToken` for cooperative cancellation.
- `Task.WhenAll` for concurrency, `Task.WhenAny` for racing.
- `IAsyncEnumerable<T>` + `await foreach` for async streaming.

## 18. Cheat Sheet

```
+------------------------------------------------------------------+
|                   ASYNC/AWAIT CHEAT SHEET                         |
+------------------------------------------------------------------+
| SYNTAX                                                            |
|  async Task MethodAsync()         - async void-returning method  |
|  async Task<T> MethodAsync()      - async returning T            |
|  async ValueTask<T> MethodAsync() - value-task returning         |
|  await task;                       - awaits completion           |
|  await foreach (var x in source)  - async enumeration           |
|  await using (var r = source)     - async disposal              |
+------------------------------------------------------------------+
| CREATING TASKS                                                    |
|  Task.Run(action)                  - queue work on thread pool  |
|  Task.FromResult(val)             - completed task with value   |
|  Task.FromException(ex)           - failed task                 |
|  Task.CompletedTask               - completed void task         |
|  Task.Delay(ms)                   - timer task                  |
|  Task.Yield()                     - force continuation          |
+------------------------------------------------------------------+
| COMPOSITION                                                       |
|  Task.WhenAll(t1, t2)             - wait all                    |
|  Task.WhenAny(t1, t2)             - wait first                  |
|  Task t.ContinueWith(cb)          - continuation (avoid)        |
+------------------------------------------------------------------+
| CANCELLATION                                                      |
|  CancellationTokenSource          - creates token               |
|  CancellationToken                - passed to async methods     |
|  ct.ThrowIfCancellationRequested() - throws OperationCanceled   |
|  ct.Register(action)              - callback on cancel          |
|  cts.CancelAfter(timeout)         - auto-cancel after timeout   |
+------------------------------------------------------------------+
| BEST PRACTICES                                                    |
|  async all the way up (no blocking)                              |
|  ConfigureAwait(false) in libraries                              |
|  Use CancellationToken in ALL async methods                      |
|  Prefer Task.WhenAll over sequential awaits                      |
|  Use ValueTask for synchronous hot paths                        |
|  Never use async void (except event handlers)                   |
|  Catch OperationCanceledException for cancellation              |
|  Use IAsyncDisposable for async cleanup                         |
+------------------------------------------------------------------+
| COMMON PATTERNS                                                   |
|  Retry:  await RetryAsync(operation, maxRetries)                |
|  Throttle: await ProcessConcurrentlyAsync(items, maxConc)        |
|  Timeout: await task.WithCancellation(cts.Token)                |
|  Fallback: await task.ContinueWith(c => fallback, canceled)     |
|  Race: await Task.WhenAny(task1, task2)                         |
+------------------------------------------------------------------+
