# Multithreading

## 1. Executive Summary

Multithreading in C# enables concurrent execution of code within a single process, leveraging multiple CPU cores for parallelism and maintaining responsiveness in UI applications. The `System.Threading` namespace provides `Thread`, `ThreadPool`, `Task` (via TPL), synchronization primitives (`Monitor`, `Mutex`, `Semaphore`, `ReaderWriterLockSlim`), and signaling constructs (`ManualResetEvent`, `AutoResetEvent`, `Barrier`, `CountdownEvent`). Since .NET 4.0, the Task Parallel Library (TPL) is the recommended approach over raw threads.

## 2. Core Theory

### Thread States

```
Unstarted -> Running -> WaitSleepJoin -> (resume) -> Running -> Stopped
                                     -> Suspended  -> (resume) -> Running
```

### Key Abstractions

- **Thread**: Low-level OS thread (1:1 with kernel thread). Expensive to create (~1MB stack).
- **ThreadPool**: Pool of reusable threads. `QueueUserWorkItem` or `Task.Run`.
- **Task**: Higher-level abstraction representing async work. Can use thread pool or custom scheduler.
- **Parallel LINQ (PLINQ)**: Automatic parallelization of LINQ queries (`AsParallel()`).
- **Parallel class**: `Parallel.For`, `Parallel.ForEach`, `Parallel.Invoke`.

### Synchronization Primitives

| Primitive              | Purpose                                |
|------------------------|----------------------------------------|
| `lock` (Monitor)       | Mutual exclusion (critical section)    |
| `Mutex`                | Cross-process mutual exclusion         |
| `Semaphore`/`Slim`     | Resource pool limiting                 |
| `ReaderWriterLockSlim` | Multiple readers / single writer       |
| `Barrier`              | Phase-based synchronization            |
| `CountdownEvent`       | Signal when count reaches zero         |
| `ManualResetEvent`     | Manual-reset signaling                 |
| `AutoResetEvent`       | Auto-reset signaling (one waiter)      |
| `SpinLock`             | Busy-wait lock (very short sections)   |
| `SpinWait`             | Spin-then-wait strategy                |
| `Interlocked`          | Atomic operations (CAS, increment)     |
| `Volatile`             | Memory barrier/volatile read/write     |

### Memory Model and Ordering

- .NET guarantees: All data writes are visible after a lock release.
- Without synchronization, CPU caches may not see other threads' writes.
- `volatile` prevents compiler/CPU reordering of reads/writes on a field.
- Full fences can be inserted via `Thread.MemoryBarrier()`.

## 3. Under-the-Hood Deep Dive

### Thread Creation Cost

```csharp
// Creating a thread costs:
// - 1MB virtual memory for stack (configurable via constructor)
// - Kernel object allocation (~KB)
// - Initial thread context switch
// - TLS (Thread Local Storage) initialization

// Thread creation time: ~200 microseconds (varies by OS/load)
// ThreadPool dispatch: ~1-3 microseconds
```

### Monitor (lock) Internals

```csharp
// Every object has a SyncBlock index in its header
// Object header layout (32-bit):
//   Bit 0:    lock status (0=free, 1=locked)
//   Bits 1-23: recursion count
//   Bits 24-30: waiters count
//   Bit 31:   owned by this thread?

// lock(obj) compiles to:
//   Monitor.Enter(obj, ref lockTaken)
//   try { body }
//   finally { Monitor.Exit(obj); }

// Contention: thread spins briefly (SpinWait), then blocks (WaitHandle)
```

### Thread Pool Internals

```csharp
// Thread pool maintains:
// - Minimum threads (typically CPU count)
// - Maximum threads (typically 32767)
// - Thread injection/retirement heuristics
// - Global work queue + local work-stealing queues per thread

// Hill-climbing algorithm:
// - Monitors throughput (completions/sec)
// - Injects threads when throughput increases
// - Retires threads when throughput decreases
// - Adjusts every ~500ms
```

### ReaderWriterLockSlim Internals

```csharp
// Uses a spin lock for fast path (no kernel transition)
// States:
// - Unlocked (no readers, no writer)
// - Read-locked (one or more readers)
// - Write-locked (one writer, no readers)
// - Upgradeable read (one upgradeable reader, no writer)

// Fairness: prevents writer starvation by queuing waiters
```

## 4. Production Code Examples

```csharp
// Thread-safe singleton with lazy initialization
public class CacheService
{
    private static readonly Lazy<CacheService> _instance =
        new(() => new CacheService(), LazyThreadSafetyMode.ExecutionAndPublication);

    public static CacheService Instance => _instance.Value;

    private readonly ConcurrentDictionary<string, object> _cache = new();

    public T GetOrAdd<T>(string key, Func<string, T> factory) =>
        (T)_cache.GetOrAdd(key, k => factory(k)!);
}
```

```csharp
// Producer-consumer with blocking collection
public class BatchProcessor
{
    private readonly BlockingCollection<WorkItem> _queue = new(
        new ConcurrentQueue<WorkItem>(), boundedCapacity: 1000);
    private readonly CancellationTokenSource _cts = new();

    public void Start()
    {
        for (int i = 0; i < Environment.ProcessorCount; i++)
        {
            Task.Run(() => ConsumerLoop(_cts.Token));
        }
    }

    public void Enqueue(WorkItem item)
    {
        if (!_queue.TryAdd(item, 100))
            throw new TimeoutException("Queue full");
    }

    private void ConsumerLoop(CancellationToken ct)
    {
        foreach (var item in _queue.GetConsumingEnumerable(ct))
        {
            ProcessItem(item);
        }
    }

    public void Stop()
    {
        _queue.CompleteAdding();
        _cts.Cancel();
    }
}
```

```csharp
// Parallel.ForEach with throttling
public void ProcessBatch(IEnumerable<string> files)
{
    var options = new ParallelOptions
    {
        MaxDegreeOfParallelism = Environment.ProcessorCount * 2,
        CancellationToken = _cts.Token,
        TaskScheduler = TaskScheduler.Default
    };

    Parallel.ForEach(files, options, file =>
    {
        byte[] data = File.ReadAllBytes(file);
        byte[] processed = Transform(data);
        File.WriteAllBytes(file + ".out", processed);
    });
}
```

```csharp
// Fine-grained reader-writer lock for a cache
public class ThreadSafeCache<TKey, TValue> where TKey : notnull
{
    private readonly ReaderWriterLockSlim _lock = new(LockRecursionPolicy.NoRecursion);
    private readonly Dictionary<TKey, TValue> _dict = new();

    public TValue Read(TKey key)
    {
        _lock.EnterReadLock();
        try { return _dict[key]; }
        finally { _lock.ExitReadLock(); }
    }

    public void Write(TKey key, TValue value)
    {
        _lock.EnterWriteLock();
        try { _dict[key] = value; }
        finally { _lock.ExitWriteLock(); }
    }

    public TValue ReadOrWrite(TKey key, Func<TKey, TValue> factory)
    {
        _lock.EnterUpgradeableReadLock();
        try
        {
            if (_dict.TryGetValue(key, out var existing))
                return existing;

            _lock.EnterWriteLock();
            try
            {
                _dict[key] = factory(key);
                return _dict[key];
            }
            finally { _lock.ExitWriteLock(); }
        }
        finally { _lock.ExitUpgradeableReadLock(); }
    }
}
```

```csharp
// Barrier for phased parallel computation
public class ParallelPipeline
{
    private readonly Barrier _barrier;
    private double[] _data;

    public ParallelPipeline(double[] data, int threads)
    {
        _data = data;
        _barrier = new Barrier(threads, b =>
            Console.WriteLine($"Phase {b.CurrentPhaseNumber} complete"));
    }

    public void Run()
    {
        var tasks = Enumerable.Range(0, _barrier.ParticipantCount)
            .Select(i => Task.Run(() => Worker(i)))
            .ToArray();
        Task.WaitAll(tasks);
    }

    private void Worker(int threadIndex)
    {
        int chunkSize = _data.Length / _barrier.ParticipantCount;
        int start = threadIndex * chunkSize;
        int end = (threadIndex == _barrier.ParticipantCount - 1)
            ? _data.Length : start + chunkSize;

        for (int phase = 0; phase < 3; phase++)
        {
            for (int i = start; i < end; i++)
                _data[i] = PhaseTransform(_data[i], phase);
            _barrier.SignalAndWait();
        }
    }
}
```

## 5. Real-World Scenarios

**Scenario 1: High-Performance Web Server**
- Use async I/O (not threads) for scalability.
- Use `Channel<T>` for request queuing.
- Use `Parallel.ForEach` for CPU-intensive batch processing.

**Scenario 2: Real-Time Trading System**
- Use `SpinLock` for nanosecond-scale critical sections.
- Use lock-free data structures (`ConcurrentDictionary`, `Interlocked`).
- Use `ManualResetEventSlim` for signaling between high-priority threads.

**Scenario 3: Background Job Processor**
- Use `Hangfire`/`Quartz.NET` or implement with `Channel<T>` + `Task.Run`.
- Use `SemaphoreSlim` to limit concurrent job execution.

**Scenario 4: Log Processing Pipeline**
- Producer thread reads from file -> `BlockingCollection` -> consumer threads parse -> `ActionBlock` (from Dataflow) to write.
- Use `CountdownEvent` to signal when batch is complete.

## 6. Performance

```csharp
// Lock vs Interlocked vs SpinLock
// Short critical sections (< 10ns): SpinLock or Interlocked
// Medium sections (10-1000ns): lock (Monitor)
// Long sections (> 1ms): SemaphoreSlim, ReaderWriterLockSlim
// Cross-process: Mutex, Semaphore (kernel objects)

// Thread count rule of thumb:
// CPU-bound: threads <= Environment.ProcessorCount
// IO-bound: threads > ProcessorCount (overlap I/O wait)

// False sharing: avoid by padding frequently updated fields
[StructLayout(LayoutKind.Explicit)]
public struct PaddedCounter
{
    [FieldOffset(0)] public long Value;
    [FieldOffset(64)] private long _pad1; // Prevent false sharing with adjacent data
    [FieldOffset(128)] private long _pad2;
}
```

### Synchronization Cost Comparison

| Primitive              | Acquire (contended) | Acquire (uncontended) | Memory |
|------------------------|---------------------|----------------------|--------|
| `Interlocked.Increment`| ~5ns                | ~5ns                 | None   |
| `SpinLock`             | ~20ns               | ~5ns                 | None   |
| `lock` (Monitor)       | ~100ns              | ~15ns                | Header |
| `ReaderWriterLockSlim` | ~200ns              | ~25ns                | Object |
| `SemaphoreSlim`        | ~200ns              | ~30ns                | Object |
| `AutoResetEvent`       | ~1us (kernel)       | ~100ns               | Kernel |
| `Mutex`                | ~5us (kernel)       | ~1us                 | Kernel |

## 7. Security

```csharp
// Avoid exposing synchronization objects
public class SharedState
{
    private readonly object _lock = new();
    private int _counter;

    // BAD: callers can deadlock
    public object LockObject => _lock;

    // GOOD: encapsulate locking
    public void Increment()
    {
        lock (_lock) _counter++;
    }
}

// Thread impersonation: do not cache security context
// Use SecurityContext.Run for async flow
```

## 8. Common Mistakes

```csharp
// MISTAKE 1: Locking on a public object or string
public class BadLock
{
    private readonly string _lockName = "mylock"; // String interning causes sharing!
    public void DoWork() { lock (_lockName) { } }
}
// FIX: lock on a private readonly object

// MISTAKE 2: Nested locking causing deadlock
void Transfer(Account a, Account b, decimal amount)
{
    lock (a) { lock (b) { /* transfer */ } } // Deadlock if Transfer(a,b) and Transfer(b,a)
}
// FIX: lock in consistent order (by hash code or ID)

// MISTAKE 3: Thread pool starvation from blocking tasks
Task.Run(() =>
{
    Thread.Sleep(1000); // Blocks a thread pool thread!
});
// FIX: use Task.Delay (async) or dedicated long-running task

// MISTAKE 4: Not handling AbandonedMutexException
// Mutex can be abandoned if owner crashes

// MISTAKE 5: Volatile does NOT make operations atomic
volatile int counter;
counter++; // Still read-modify-write (not atomic)
// FIX: Interlocked.Increment(ref counter)

// MISTAKE 6: Double-checked locking without volatile (pre-.NET 2.0)
// FIX: use Lazy<T> or volatile for the instance field

// MISTAKE 7: async void (fire-and-forget) in non-UI contexts
// FIX: async Task instead

// MISTAKE 8: Thread.Abort() is dangerous and obsolete
// FIX: use CancellationToken for cooperative cancellation
```

## 9. Senior Engineer Perspective

**1. Prefer TPL over raw threads.** Tasks are lighter, more flexible, and integrate with async/await.

**2. Use `ConcurrentDictionary` instead of `Dictionary` + `lock`.** For most scenarios, it's optimized better.

**3. Consider work-stealing vs dedicated threads.** Thread pool with work-stealing queues gives better load balancing.

**4. Know Amdahl's Law:** `Speedup = 1 / ((1 - P) + P/N)`. Parallel speedup is limited by sequential portion.

**5. Use `ValueTask` for synchronization-free hot paths** to avoid allocation.

**6. For high-performance scenarios, consider Dataflow (ActionBlock, TransformBlock)** as a higher-level abstraction for pipelining.

**7. Thread injection in thread pool is lazy.** Call `ThreadPool.SetMinThreads()` to avoid latency spikes under load:

```csharp
ThreadPool.SetMinThreads(workerThreads: 16, completionPortThreads: 16);
```

**8. Use `Channel<T>` for producer-consumer** (preferred over `BlockingCollection` in modern code).

## 10. Interview Questions (Easy)

1. What is a thread? How does it differ from a process?
2. How do you create a new thread in C#?
3. What is the ThreadPool and why use it?
4. What is a race condition?
5. What does the `lock` statement do?
6. What is a deadlock?
7. How does `Monitor.Enter` / `Monitor.Exit` relate to `lock`?
8. What is the difference between `Thread.Sleep` and `Task.Delay`?
9. What is a `Mutex` and how does it differ from a `lock`?
10. What is the purpose of `Interlocked` class?

## 11. Interview Questions (Medium)

1. Explain the difference between `Thread` and `Task`.
2. What is the difference between `ConcurrentQueue<T>` and `Queue<T>` with `lock`?
3. How does `ReaderWriterLockSlim` improve performance over `lock`?
4. Explain the concept of thread safety and immutability.
5. What is a `Barrier` and when would you use it?
6. How does `SemaphoreSlim` differ from `Semaphore`?
7. Explain the volatile keyword and memory barriers.
8. What is thread-local storage (`ThreadLocal<T>`, `ThreadStaticAttribute`)?
9. How does `SpinLock` work and when is it appropriate?
10. What is the `CountdownEvent` and how does it differ from `ManualResetEvent`?

## 12. Advanced Interview Questions (Hard)

1. Implement a lock-free stack using `Interlocked.CompareExchange`.
2. Explain the memory model guarantees in .NET: acquire/release semantics.
3. Design a work-stealing queue (like .NET ThreadPool's local queues).
4. Implement double-checked locking correctly in modern C#.
5. Explain how `Lazy<T>` with `LazyThreadSafetyMode.ExecutionAndPublication` works internally.
6. Design a scalable multi-producer, single-consumer queue.
7. Explain false sharing and how to mitigate it in .NET.
8. Implement a `ManualResetEventSlim` equivalent from scratch.
9. Design a lock-free hash table with resize capability.
10. Explain the thread pool hill-climbing algorithm in depth.

## 13. Interview Questions (System Design)

1. Design a high-throughput message queue using multithreading primitives.
2. Design a web crawler that respects robots.txt using parallel processing.
3. Design a job scheduler with dependencies and parallel execution.
4. Design a distributed counter service with eventual consistency.
5. Design a real-time monitoring dashboard with concurrent data ingestion.
6. Design a parallel ETL pipeline with backpressure handling.
7. Design a multi-threaded game server with lock-free state management.
8. Design a distributed lock service using leases and fencing tokens.
9. Design a thread-safe in-memory event store with optimistic concurrency.
10. Design a parallel ray tracer using work-stealing.

## 14. Expert-Level Interview Questions (Architect)

1. Design a lock-free, wait-free, linearizable concurrent dictionary for a trading exchange with nanosecond latency requirements.
2. Architect a user-mode scheduler (like Go's goroutine scheduler) on top of .NET threads, implementing M:N threading (M user tasks on N OS threads) with work-stealing and stack copying.
3. Design a distributed consensus algorithm (Raft/Paxos) using .NET synchronization primitives with exactly-once semantics.
4. Architect a real-time stream processing engine (like Flink) using channels, barriers for checkpointing, and exactly-once state snapshots.
5. Design a lock-free memory allocator for a game engine that avoids GC pauses entirely.
6. Architect a multi-version concurrency control (MVCC) engine for an in-memory database using .NET synchronization primitives.
7. Design a distributed transaction coordinator using two-phase commit across process boundaries with failure recovery.
8. Architect a thread-sanitizer-like tool for .NET that detects data races at runtime using happens-before tracking.
9. Design a reactive programming framework that implements the Reactive Manifesto (responsive, resilient, elastic, message-driven) using actors and threads.
10. Architect a real-time bidding exchange handling 1M+ bids/second using lock-free data structures and CPU pinning.

## 15. Debugging & Troubleshooting

```csharp
// Detect deadlocks in dump:
// - Use SOS: !locks, !syncblk, !dlk
// - Look for threads in WaitSleepJoin state
// - Check SyncBlock ownership

// ETW traces for contention:
// - Microsoft-Windows-DotNETRuntime (ContentionKeyword)
// - PerfView: "Thread Time" with "Contention" stack

// Common debugging tools:
// - Visual Studio Parallel Stacks window
// - WinDbg + SOS
// - dotnet-dump analyze
// - PerfView
// - Concurrency Visualizer (VS extension)

// Code patterns for debugging:
Thread.CurrentThread.Name = "Worker-" + id; // Name threads for stack traces
```

## 16. Comparison Section

```
+---------------------+------------------+------------------+
| Feature             | Thread           | Task             |
+---------------------+------------------+------------------+
| Abstraction level   | OS thread        | Promise/task     |
| Creation cost       | ~1MB stack       | ~100 bytes       |
| Scheduler           | OS kernel        | TaskScheduler    |
| Return value        | No               | Yes (Task<T>)    |
| Composition         | Manual (Join)    | ContinueWith     |
| Cancellation        | Thread.Abort     | CancellationToken|
| Async support       | No               | Yes (async/await)|
| Pool integration    | No               | Yes (default)    |
+---------------------+------------------+------------------+

+--------------------+-------------------+-------------------+
| Primitive          | User-mode         | Kernel-mode       |
+--------------------+-------------------+-------------------+
| SpinLock           | Yes (busy-wait)   | No                |
| Interlocked        | Yes (CAS)         | No                |
| Monitor (lock)     | Spin then kernel  | WaitHandle        |
| ReaderWriterSlim   | Spin then kernel  | AutoResetEvent    |
| SemaphoreSlim      | Spin then kernel  | Semaphore         |
| ManualResetEvent   | No                | Yes               |
| Mutex              | No                | Yes               |
+--------------------+-------------------+-------------------+
```

## 17. Revision Notes

- Prefer `Task` over `Thread`, TPL over raw threading.
- `lock` = `Monitor.Enter`/`Exit`. Use private `object` as lock target.
- `async void` only for event handlers; use `async Task` otherwise.
- `ConcurrentDictionary`, `ConcurrentQueue` for thread-safe collections.
- `Interlocked` for atomic operations on primitives.
- `CancellationToken` for cooperative cancellation (not `Thread.Abort`).
- `ReaderWriterLockSlim` for read-heavy scenarios.
- Thread pool minimum threads can be increased via `SetMinThreads`.
- Avoid `lock(this)`, `lock(typeof(T))`, `lock(string)`.
- Deadlock prevention: fixed lock ordering, timeout, lock hierarchy.

## 18. Cheat Sheet

```
+------------------------------------------------------------------+
|                  MULTITHREADING CHEAT SHEET                       |
+------------------------------------------------------------------+
| CREATION                                                          |
|  new Thread(Start).Start()         - raw thread                  |
|  ThreadPool.QueueUserWorkItem(cb)  - pool thread                 |
|  Task.Run(action)                  - TPL task                    |
|  Parallel.For/ForEach/Invoke       - data parallelism            |
+------------------------------------------------------------------+
| SYNCHRONIZATION                                                    |
|  lock (obj) { ... }                - mutual exclusion             |
|  Monitor.Enter/Exit(obj)           - equivalent to lock           |
|  Interlocked.Increment(ref x)      - atomic increment            |
|  Interlocked.CompareExchange(ref x, val, cmp) - CAS               |
|  volatile int _field;              - compiler reordering barrier  |
+------------------------------------------------------------------+
| SIGNALING                                                          |
|  AutoResetEvent / ManualResetEvent - one/all waiters              |
|  ManualResetEventSlim              - user-mode MRSE              |
|  Barrier(n)                         - phased sync                |
|  CountdownEvent(n)                 - count to zero               |
|  SemaphoreSlim(n)                  - resource pool              |
+------------------------------------------------------------------+
| CONCURRENT COLLECTIONS                                             |
|  ConcurrentDictionary<K,V>         - thread-safe dictionary       |
|  ConcurrentQueue<T>               - lock-free FIFO               |
|  ConcurrentStack<T>               - lock-free LIFO               |
|  ConcurrentBag<T>                 - unordered, thread-local      |
|  BlockingCollection<T>            - bounded producer-consumer    |
+------------------------------------------------------------------+
| BEST PRACTICES                                                     |
|  Avoid shared state (prefer immutability)                        |
|  Lock as little as possible (fine-grained locking)                |
|  Always acquire locks in same order (deadlock prevention)         |
|  Use CancellationToken for cancellation                          |
|  Prefer async/await over blocking threads                         |
|  Use ConcurrentDictionary instead of Dictionary + lock           |
|  Set ThreadPool min threads to avoid latency spikes              |
+------------------------------------------------------------------+
