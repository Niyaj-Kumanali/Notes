# Multithreading

---

## Overview

- **Definition:** Concurrent execution of code within a single process, leveraging multiple CPU cores for parallelism and maintaining UI responsiveness. The `System.Threading` namespace provides threads, the thread pool, and synchronization primitives.
- **Why It Exists:** Enables CPU-bound parallelism (multi-core utilization), maintains UI thread responsiveness by offloading work, and allows overlapping I/O with computation.
- **Key Concepts:** **`Thread`** (low-level OS thread, ~1MB stack), **`ThreadPool`** (reusable thread pool with hill-climbing), **`Task`** (TPL abstraction), **synchronization primitives** (`Monitor`, `Mutex`, `Semaphore`, `ReaderWriterLockSlim`), **signaling constructs** (`ManualResetEvent`, `AutoResetEvent`, `Barrier`, `CountdownEvent`), and **concurrent collections** (`ConcurrentDictionary`, `ConcurrentQueue`, `BlockingCollection`, `Channel<T>`).

---

## Core Concepts

- **Thread States:** `Unstarted → Running → WaitSleepJoin → Running → Stopped`. A thread can also transition through `Suspended → Running`.
- **Synchronization Primitives:**
  - **`lock` (Monitor):** Mutual exclusion on a reference type. Compiles to `Monitor.Enter`/`Monitor.Exit`. Every object has a sync block index in its header for lightweight locking.
  - **`Mutex`:** Cross-process mutual exclusion (kernel object). Heavier than `lock` (~5µs acquire).
  - **`Semaphore`/`SemaphoreSlim`:** Resource pool limiting. `SemaphoreSlim` stays in user mode when uncontended.
  - **`ReaderWriterLockSlim`:** Multiple concurrent readers or exclusive writer. Uses spin lock for fast path, prevents writer starvation.
  - **`Barrier`:** Phase-based synchronization where all participants signal and wait.
  - **`CountdownEvent`:** Signal when a count reaches zero (one-time use).
  - **`SpinLock`/`SpinWait`:** Busy-wait for very short critical sections (< 10ns). Avoid in general code.
  - **`Interlocked`:** Atomic operations (CAS, increment, exchange) without locking.
  - **`volatile`:** Prevents compiler/CPU reordering of reads/writes on a field. Does NOT make operations atomic.

```csharp
// Thread-safe singleton with lazy initialization
public class CacheService
{
    private static readonly Lazy<CacheService> _instance =
        new(() => new CacheService(), LazyThreadSafetyMode.ExecutionAndPublication);
    public static CacheService Instance => _instance.Value;
}
```

- **Memory Model:** .NET guarantees all data writes are visible after a lock release. Without synchronization, CPU caches may not see other threads' writes. Full memory barriers can be inserted via `Thread.MemoryBarrier()`.
- **Thread Pool Internals:** Maintains minimum threads (typically CPU count), maximum threads (default 32767), and uses hill-climbing algorithm that monitors throughput (completions/sec) to inject or retire threads every ~500ms. Thread injection is lazy — call `ThreadPool.SetMinThreads()` to avoid latency spikes under load.
- **Thread Creation Cost:** ~200 microseconds, 1MB virtual memory for stack, kernel object allocation, TLS initialization. ThreadPool dispatch is ~1-3 microseconds.

```csharp
// Producer-consumer with BlockingCollection
public class BatchProcessor
{
    private readonly BlockingCollection<WorkItem> _queue = new(
        new ConcurrentQueue<WorkItem>(), boundedCapacity: 1000);

    public void Start()
    {
        for (int i = 0; i < Environment.ProcessorCount; i++)
            Task.Run(() => ConsumerLoop());
    }

    private void ConsumerLoop()
    {
        foreach (var item in _queue.GetConsumingEnumerable())
            ProcessItem(item);
    }
}
```

---

## Common Mistakes

- **Locking on a public object or string** — Strings are interned (shared across the process), and public objects allow external code to participate in the lock, causing deadlocks. Always lock on a private `readonly object`. This *looks correct* because the lock compiles and runs without error in testing, but a string literal `"lockObj"` is interned globally — any other code in the process using the same literal shares the same monitor, turning unrelated critical sections into a deadlock source.
- **Nested locking causing deadlock** — Locking `a` then `b` in one thread and `b` then `a` in another guarantees deadlock. Fix by locking in a consistent order (by hash code or ID). This *looks correct* because each individual transaction acquires locks as needed and the code compiles without warnings, but the circular wait condition is a classic deadlock that only manifests under concurrent load and may pass all unit tests.
- **Thread pool starvation from blocking tasks** — Calling `Thread.Sleep` or blocking on a thread pool thread prevents it from processing other work items. Use async `Task.Delay` instead. This *looks correct* because `Thread.Sleep` is the familiar synchronous API and the code appears to pause execution as intended, but it occupies a thread pool thread that could otherwise serve hundreds of other queued work items, starving the pool under load.
- **Not handling `AbandonedMutexException`** — A `Mutex` can be abandoned if the owner crashes, throwing this exception on the next waiter. This *looks correct* because a `Mutex` appears to be just a cross-process `lock`, but if the owning thread terminates without releasing it, the mutex is abandoned — subsequent waiters must catch `AbandonedMutexException` and determine whether the protected resource is in a consistent state.
- **`volatile` does NOT make operations atomic** — `counter++` is read-modify-write even on a `volatile` field. Use `Interlocked.Increment`. This *looks correct* because `volatile` suggests the field is always up-to-date across threads, but it only prevents compiler reordering and ensures cache coherence on individual reads and writes — the read-modify-write of `counter++` can still interleave with another thread's increment, causing lost updates.
- **`async void` in non-UI contexts** — Exceptions crash the process. Use `async Task`. This *looks correct* because `async void` compiles and runs like `async Task` in the happy path, but there is no `Task` to observe exceptions — an unhandled exception propagates to the `SynchronizationContext` and terminates the process, with no stack trace logged to the standard error handler.
- **`Thread.Abort()` is dangerous and obsolete** — Use `CancellationToken` for cooperative cancellation. This *looks correct* because `Abort` immediately terminates the thread and appears to solve unresponsive code, but it throws `ThreadAbortException` at an arbitrary point, potentially leaving locks held, state corrupted, and `finally` blocks partially executed — making it impossible to recover gracefully.

```csharp
// Deadlock example: inconsistent lock ordering
void Transfer(Account a, Account b, decimal amount)
{
    lock (a) { lock (b) { /* transfer */ } }
}
// Fix: always lock in order of account ID
```

---

## Key Design Considerations

- **Prefer TPL over raw threads** — Tasks are lighter (~100 bytes vs ~1MB stack), more flexible, integrate with async/await, and use the thread pool with work-stealing for better load balancing.
- **Use `ConcurrentDictionary` instead of `Dictionary` + `lock`** — Uses striped locking (per-bucket), optimized for concurrent access patterns.
- **Know Amdahl's Law:** `Speedup = 1 / ((1-P) + P/N)` — Parallel speedup is fundamentally limited by the sequential portion of the workload.
- **Work-stealing vs dedicated threads** — Thread pool uses local work-stealing queues per thread for better cache locality and load distribution.
- **Consider Dataflow (`ActionBlock`, `TransformBlock`)** — Higher-level abstraction for pipelining and producer-consumer with parallelism control.
- **Use `Channel<T>` over `BlockingCollection`** — Async-first, supports backpressure, and integrates with `IAsyncEnumerable<T>`.
- **Minimum threads** — Call `ThreadPool.SetMinThreads(workerThreads: 16, completionPortThreads: 16)` to avoid latency spikes during load bursts.

```csharp
ThreadPool.SetMinThreads(workerThreads: 16, completionPortThreads: 16);
```

---

## Real-World Scenarios

### Scenario 1: High-Performance Order Matching Engine
**Context:** A trading platform needs to match buy/sell orders across multiple instruments with microsecond-level latency. Must handle concurrent order submission and maintain thread safety.

```csharp
public class OrderBook
{
    private readonly SortedSet<LimitOrder> _bids = new(OrderComparer.Descending); // Max heap by price
    private readonly SortedSet<LimitOrder> _asks = new(OrderComparer.Ascending);  // Min heap by price
    private readonly object _lock = new();

    public MatchResult SubmitOrder(LimitOrder order)
    {
        lock (_lock) // Short critical section — lock is appropriate
        {
            var oppositeBook = order.Side == Side.Buy ? _asks : _bids;
            var matches = new List<Trade>();
            
            while (oppositeBook.Count > 0 && CanMatch(order, oppositeBook.Min))
            {
                var best = oppositeBook.Min;
                int fillQuantity = Math.Min(order.RemainingQuantity, best.RemainingQuantity);
                
                matches.Add(new Trade(best.OrderId, order.OrderId, best.Price, fillQuantity));
                order.Reduce(fillQuantity);
                best.Reduce(fillQuantity);
                
                if (best.RemainingQuantity == 0) oppositeBook.Remove(best);
                if (order.RemainingQuantity == 0) break;
            }
            
            if (order.RemainingQuantity > 0)
                (order.Side == Side.Buy ? _bids : _asks).Add(order);
                
            return new MatchResult(matches, order.RemainingQuantity == 0);
        }
    }
}
```

### Scenario 2: Parallel Image Processing Pipeline
**Context:** A batch photo processing service needs to resize, watermark, and compress 10,000 images. Each operation is CPU-bound and independent.

```csharp
public class ImageBatchProcessor
{
    public async Task ProcessBatchAsync(string[] imagePaths, CancellationToken ct)
    {
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct
        };

        await Parallel.ForEachAsync(imagePaths, options, async (path, token) =>
        {
            try
            {
                using var image = await Image.LoadAsync(path, token);
                
                // Each image processed independently on its own thread
                var resized = image.Clone(ctx => ctx.Resize(800, 0));
                ApplyWatermark(resized);
                
                var outputPath = Path.Combine(_outputDir, Path.GetFileName(path));
                await resized.SaveAsJpegAsync(outputPath, token);
                
                Interlocked.Increment(ref _processedCount); // Thread-safe counter
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                Interlocked.Increment(ref _failedCount);
                _logger.LogError(ex, "Failed to process {Path}", path);
            }
        });
    }
}
```

### Scenario 3: Distributed Task Queue with Work-Stealing
**Context:** A compute cluster node processes tasks from a shared queue. Each node has multiple worker threads that should steal work from overloaded peers.

```csharp
public class WorkStealingScheduler
{
    private readonly ThreadLocal<ConcurrentQueue<Action>> _localQueues = new(true);
    private readonly ConcurrentQueue<Action> _globalQueue = new();
    private readonly CancellationTokenSource _cts = new();

    public void Start(int workerCount)
    {
        for (int i = 0; i < workerCount; i++)
        {
            var thread = new Thread(WorkerLoop) { Name = $"Worker-{i}", IsBackground = true };
            thread.Start();
        }
    }

    private void WorkerLoop()
    {
        var localQueue = _localQueues.Value;
        
        while (!_cts.IsCancellationRequested)
        {
            // 1. Try local queue (LIFO — good cache locality)
            if (localQueue.TryDequeue(out var task)) { task(); continue; }
            
            // 2. Try stealing from another worker's local queue
            foreach (var otherQueue in _localQueues.Values)
            {
                if (otherQueue != localQueue && otherQueue.TryDequeue(out task))
                { task(); goto next; }
            }
            
            // 3. Fall back to global queue
            if (_globalQueue.TryDequeue(out task)) { task(); continue; }
            
            Thread.SpinWait(100); // No work available, back off
            next:;
        }
    }
}
```

---

## Scenario-Based Questions

1. **Q: You are building a real-time chat server that handles 100K concurrent connections. How do you design the threading model?**
   A: Use async I/O with `SocketAsyncEventArgs` or Kestrel's transport layer — never dedicate a thread per connection (10K threads would consume 10GB+ stack space). Use the thread pool for CPU-bound work (message serialization). For broadcasting to large groups, use `ConcurrentDictionary<Guid, Channel<Message>>` and write to each channel in a loop. Consider `Pipelines` for zero-copy network I/O. Use `ReaderWriterLockSlim` for infrequent connection list modifications.

2. **Q: You have a legacy WinForms app that freezes during a long computation. How do you make it responsive without a full rewrite?**
   A: Use `Task.Run` to offload the computation to the thread pool. Use `IProgress<T>` (wraps `SynchronizationContext.Post`) to report progress back to the UI thread. Disable the "Start" button before the task and re-enable in the continuation. For cancellation, use `CancellationTokenSource` and pass the token to the task. If the computation supports chunked processing, yield periodically to check cancellation.

3. **Q: You are optimizing a high-frequency trading system where lock contention is the bottleneck. How do you reduce it?**
   A: Use lock-free data structures (`ConcurrentDictionary`, `Interlocked` operations, `SpinLock` for sub-microsecond sections). Partition data by symbol or instrument so that different threads rarely contend on the same lock (striped locking). Use `ReaderWriterLockSlim` for read-dominated data. Consider `MemoryBarrier`-based lock-free algorithms for simple state. Profile with `System.Threading.CountdownEvent` to measure contention via ETW events.
   > **Interview follow-up:** If a lock-free CAS loop fails repeatedly under high contention, what metric tells you whether the backoff strategy is adequate versus whether the algorithm is fundamentally unscalable?

4. **Q: You are debugging a production crash with `StackOverflowException` in a multithreaded service. What could cause it?**
   A: Recursive lock-free pattern bugs (e.g., CAS loop that never succeeds because of contention). Deep call chains in thread pool threads due to aggressive inlining + deep async state machines. Unbounded recursion in a parallel algorithm. Deadlock recovery logic that retries infinitely. Fix: dump analysis with `!clrstack` to find the repeating call pattern, add recursion limits, and review lock-free algorithms for correctness.

5. **Q: You are migrating from `lock` to `ReaderWriterLockSlim` for a configuration cache. What trade-offs should you consider?**
   A: `ReaderWriterLockSlim` allows unlimited concurrent readers when no writer exists — great for config (read 1000x/sec, write 1x/hour). But it's ~2x slower than `lock` for the exclusive (write) case. It also has more overhead for very short critical sections. Only beneficial when reads significantly outnumber writes AND the read section is non-trivial (> 1µs). For a simple dictionary lookup, `lock` may be faster due to lower overhead.
   > **Interview follow-up:** Under what pattern of read/write interleaving does `ReaderWriterLockSlim` exhibit writer starvation despite its fairness guarantees, and how do you detect it in production?

6. **Q: You are building a backtesting engine that processes years of tick data. How do you parallelize it correctly?**
   A: Partition data by time window (e.g., one day per partition) — ensure no cross-partition dependencies. Use `Parallel.ForEach` on the partition list. Each partition runs sequentially (tick data is ordered). Aggregate results using `Interlocked` or a lock-protected list. Challenge: some strategies need look-back across partitions — implement a warm-up period or overlapping partitions. Use `ImmutableArray<T>` for strategy parameters to avoid synchronization.

7. **Q: You need to implement a thread-safe lazy-initialized singleton. Why prefer `Lazy<T>` over double-checked locking?**
   A: `Lazy<T>` with `LazyThreadSafetyMode.ExecutionAndPublication` guarantees single execution and publication of the result — the runtime handles all memory barriers correctly. Double-checked locking is error-prone (must `volatile` the field) and performance varies by .NET version. `Lazy<T>` also supports exception caching (if the factory throws, subsequent accesses re-throw the same exception). Only use manual double-checked locking if you need the singleton to be re-created after failure.

8. **Q: You are designing a thread pool for a game engine where predictability matters more than throughput. How is it different from .NET's ThreadPool?**
   A: .NET's ThreadPool optimizes for throughput via hill-climbing (varies thread count dynamically). A game engine needs fixed thread count (core count - 1) to prevent oversubscription. Use dedicated threads with known affinitized cores. Use spin-waiting for short tasks (avoids context switch latency, ~1-2µs). Use fiber-like cooperative scheduling within threads to avoid kernel transitions. No dynamic thread injection — it causes frame time spikes.

9. **Q: You are implementing a rate limiter in a multithreaded server. How do you maintain accurate counts under high concurrency?**
   A: Use `Interlocked.Increment` on a counter for each sliding window bucket. For a token bucket algorithm, use `Interlocked.CompareExchange` (CAS) in a `while` loop to atomically update remaining tokens. For a distributed rate limiter, use Redis `INCR` with `EXPIRE`. For high precision, use `long` ticks via `Stopwatch.GetTimestamp()`. Trade-off: `Interlocked` operations are ~5ns but lack waiting semantics — combine with `SemaphoreSlim` for blocking when rate is exceeded.

10. **Q: You have a thread pool starvation issue — response times spike to 30s under load. How do you diagnose and fix it?**
     A: Capture `ThreadPool` metrics: `ThreadPool.GetAvailableThreads` shows zero workers. Common causes: blocking calls on thread pool threads (`.Result`, `lock` held for long I/O), too many long-running tasks, or insufficient min threads. Fix: ensure no blocking calls in async code, increase `ThreadPool.SetMinThreads` to prevent latency spikes, and use `TaskCreationOptions.LongRunning` for truly long CPU-bound work. Monitor `clr!ThreadPoolWorkerThreadWait` in ETW traces.
    > **Interview follow-up:** If you increase `SetMinThreads` to 100 but the starvation persists, what diagnostic step distinguishes between "not enough threads" and "threads are blocked and not completing work"?

---

## Interview Questions

1. **What is the difference between `lock` and `Monitor`?**
   A: `lock(obj) { body }` is syntactic sugar for `Monitor.Enter(obj, ref lockTaken)` in a `try` block with `Monitor.Exit(obj)` in `finally`. `Monitor` additionally provides `TryEnter` with timeout and `Pulse`/`Wait` for signaling between threads.

2. **What does `volatile` do?**
   A: It prevents compiler and CPU reordering of reads/writes on a field. Every volatile read has acquire semantics; every volatile write has release semantics. It does NOT make compound operations like `counter++` atomic.

3. **Explain the thread pool hill-climbing algorithm.**
   A: The thread pool monitors throughput (completions per second). It periodically adds a thread and measures throughput change. If throughput increases, it adds more; if it decreases, it removes threads. This converges to the optimal thread count dynamically. Adjustments happen approximately every 500ms.

4. **How does `Interlocked.Increment` work at the CPU level?**
   A: It uses a CPU-level atomic instruction (LOCK XADD on x86, LDXR/STXR on ARM) that reads, increments, and writes the value in a single uninterruptible operation. This avoids the cost of a full memory barrier and lock acquisition.

5. **What is false sharing and how do you prevent it?**
   A: False sharing occurs when threads on different cores modify variables that share a CPU cache line (typically 64 bytes). Each modification invalidates the cache line for other cores. Mitigate by padding fields with `[FieldOffset]` to ensure independent fields are on separate cache lines.

6. **What is the difference between `AutoResetEvent` and `ManualResetEvent`?**
   A: `AutoResetEvent` automatically resets to non-signaled after releasing a single waiting thread (like a turnstile). `ManualResetEvent` stays signaled until manually reset, releasing all waiting threads simultaneously. `AutoResetEvent` is for one-at-a-time signaling; `ManualResetEvent` is for broadcast-style signaling.

7. **How does `ConcurrentDictionary` achieve thread safety?**
   A: It uses striped locking — the internal bucket array is divided into regions, each protected by a separate lock. Read operations are mostly lock-free (volatile reads). Write operations lock only the relevant stripe, allowing concurrent access to different regions. Resizing acquires all locks.

8. **What is a deadlock and how do you prevent it?**
   A: A deadlock occurs when two or more threads each hold a lock the other needs. Prevention: always acquire locks in a consistent global order (by hash code or ID), use `Monitor.TryEnter` with timeout, and avoid nested locks when possible. Detection: ETW events via `Monitor.LockContention` or `!syncblk` in WinDbg.

9. **Explain `TaskCreationOptions.LongRunning`.**
   A: It tells the TPL to create a dedicated thread (not use the thread pool) for the task. Use for long-running CPU-bound operations that would otherwise monopolize a thread pool thread. Without it, the thread pool might add more threads to compensate, causing oversubscription.

10. **How does `SpinLock` differ from `lock`?**
    A: `SpinLock` busy-waits (spins in a loop) instead of context-switching. It's faster for very short critical sections (< 10ns) but wastes CPU cycles on contention. `lock` (Monitor) yields the thread on contention, which is better for longer sections. Use `SpinLock` only after profiling confirms `lock` is the bottleneck.

---

## Developer Recommendations

- **Prefer `Task` over raw `Thread` for most scenarios** — A `Thread` has ~1MB stack and takes ~200µs to create. A `Task` uses the thread pool (~100 bytes, ~1µs dispatch). Tasks integrate with async/await, support cancellation, and enable composition (`WhenAll`, `WhenAny`). Reserve raw `Thread` for long-running CPU-bound operations that need a dedicated OS thread. A monitoring agent once created a raw `Thread` per connection for 10K concurrent connections, consuming 10GB+ of virtual memory just for thread stacks and crashing the process with `OutOfMemoryException` before reaching 5K connections.

- **Use `SemaphoreSlim` for async-compatible synchronization** — Unlike `Monitor` (which blocks the thread), `SemaphoreSlim.WaitAsync()` returns a task that completes when the semaphore is acquired. This frees the thread during the wait, preventing thread pool starvation. Use it for resource pooling and rate limiting in async code.

- **Set `ThreadPool.SetMinThreads` at application startup** — The default minimum thread count equals CPU count, causing latency spikes when burst traffic arrives. Set to at least `Environment.ProcessorCount * 4` for I/O-bound services. This prevents the hill-climbing algorithm from injecting threads too slowly during load spikes. An API gateway once received a sudden traffic burst that queued 500 requests while the thread pool slowly injected threads one at a time every 500ms — the first requests timed out at 30 seconds before the pool had enough threads to process the backlog.

- **Avoid `lock(this)` or locking on public types** — Lock on a private `readonly object` field. Locking public objects allows external code to participate in the lock, potentially causing deadlocks. Locking on strings is especially dangerous due to string interning (two identical string literals share the same object).

- **Use `ConcurrentDictionary` instead of `Dictionary` + manual locking** — `ConcurrentDictionary` uses striped locking for fine-grained concurrency. Its `GetOrAdd` and `AddOrUpdate` methods provide atomic read-modify-write that would require complex double-checked locking with `Dictionary`.

- **Use `Channel<T>` for producer-consumer over `BlockingCollection<T>`** — `Channel<T>` is async-first, supports backpressure, and integrates with `IAsyncEnumerable<T>`. `BlockingCollection<T>` blocks consumer threads, making it unsuitable for async pipelines. `Channel<T>` also offers `BoundedChannelFullMode` for various backpressure strategies.

- **Measure lock contention before optimizing** — Profile with `dotnet-trace` and look for `Monitor.Contention` events or use `PerfView`. Changing `lock` to `ReaderWriterLockSlim` or `Interlocked` adds complexity. Only optimize when contention is proven to be the bottleneck (> 5% of CPU time or high count of contention events).

---

## Lock Contention Costs

| Primitive | Acquire (uncontended) | Acquire (contended) |
|---|---|---|
| `Interlocked.Increment` | ~5ns | ~5ns |
| `SpinLock` | ~5ns | ~20ns |
| `lock` (Monitor) | ~15ns | ~100ns |
| `ReaderWriterLockSlim` | ~25ns | ~200ns |
| `SemaphoreSlim` | ~30ns | ~200ns |
| `AutoResetEvent` | ~100ns | ~1µs |
| `Mutex` | ~1µs | ~5µs |

## Synchronization Strategy

- **Short critical sections (<10ns):** `Interlocked` or `SpinLock`
- **Medium sections (10-1000ns):** `lock` (Monitor)
- **Long sections (>1ms):** `SemaphoreSlim` or `ReaderWriterLockSlim`
- **Cross-process:** `Mutex`, `Semaphore` (kernel objects)
- **CPU-bound parallelism:** Threads ≤ `Environment.ProcessorCount`
- **I/O-bound:** More threads than cores to overlap I/O wait
