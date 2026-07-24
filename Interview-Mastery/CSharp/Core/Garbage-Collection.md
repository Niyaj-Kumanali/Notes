# C# Garbage Collection & Memory Management

---

## Value Types vs Reference Types

### Value Types
- Stored directly on the **stack** (unless boxed)
- Include: `struct`, `enum`, `int`, `long`, `float`, `double`, `decimal`, `bool`, `char`, `DateTime`, `TimeSpan`, `Guid`, `byte`, `short`
- Each variable gets its own **copy** of the data
- Assigned by **copying** the value
- Cannot be `null` (unless wrapped in `Nullable<T>` or `T?`)
- Have a **fixed size** known at compile time
- Inherit from `System.ValueType`
- Comparison uses **value equality** by default

### Reference Types
- Stored on the **heap**; the variable on the stack holds a **reference** (pointer)
- Include: `class`, `string`, `array`, `delegate`, `record`, `interface`, `object`
- Variables point to the **same object** in memory
- Assigned by **copying the reference**, not the object
- Can be `null`
- Size on heap varies; managed by the GC
- Inherit from `System.Object`
- Comparison uses **reference equality** by default (unless overridden)

### Where They Are Stored

| Location | What Goes There |
|----------|----------------|
| **Stack** | Value type local variables, method parameters (value types), return values (value types), local variables of reference types (the reference itself) |
| **Heap** | Reference type objects, boxed value types, closures, lambdas capturing variables, delegate targets |

### Why Value Types Are Faster for Small Data
- No heap allocation required — data lives on the stack
- No GC pressure — stack frames are cleaned up automatically when methods return
- Better CPU cache locality — contiguous memory on the stack
- No pointer dereferencing — data is accessed directly
- For small data (under ~16 bytes), the overhead of heap allocation and GC tracking outweighs any benefit of reference types

### When to Choose struct vs class
- **Use struct** when:
  - The type is small (typically under 16 bytes)
  - The type represents a single value or a small group of values (e.g., `Point`, `DateTime`, `Range`)
  - The type is immutable
  - You want to avoid heap allocation and GC pressure
  - The type will not be used in polymorphism
- **Use class** when:
  - The type is large (over 16 bytes)
  - The type has complex behavior or identity
  - You need inheritance or polymorphism
  - The type needs to be nullable
  - The type is used in collections that may resize frequently (avoid copying large structs)

### Size of Common Types

| Type | Size (bytes) | Notes |
|------|-------------|-------|
| `bool` | 1 | CLR stores as 1 byte |
| `byte` | 1 | |
| `sbyte` | 1 | |
| `char` | 2 | UTF-16 |
| `short` / `ushort` | 2 | |
| `int` / `uint` | 4 | |
| `long` / `ulong` | 8 | |
| `float` | 4 | Single-precision IEEE 754 |
| `double` | 8 | Double-precision IEEE 754 |
| `decimal` | 16 | High-precision, used for financial |
| `DateTime` | 8 | Ticks since 01/01/0001 |
| `TimeSpan` | 8 | Ticks |
| `Guid` | 16 | 128-bit identifier |
| `IntPtr` | 4 or 8 | Platform-dependent (pointer-sized) |
| `nint` / `nuint` | 4 or 8 | Native-sized integer (C# 9+) |
| Reference type ref | 4 or 8 | Size of a pointer on the platform |

---

## Stack vs Heap

### Stack
- A **LIFO** (Last In, First Out) data structure
- Extremely fast — just moving a stack pointer
- Automatically managed — memory is reclaimed when a method returns
- **Thread-local** — each thread gets its own stack (typically 1 MB default)
- Stores:
  - Local variables of value types
  - Method parameters (value types, passed by value)
  - Return addresses
  - Intermediate expression results
  - Method call frames (activation records)
- Cannot be resized dynamically
- No need for garbage collection

### Heap
- A pool of memory used for **dynamic allocation**
- Managed by the **Garbage Collector (GC)**
- **Shared across threads** — any thread can allocate and access heap objects
- Stores:
  - All reference type objects
  - Boxed value types
  - Closures and lambda captures
  - Large structs that are boxed
  - Delegates
  - String literals (interned)
- Allocations are more expensive than stack (must find free space, update GC data structures)
- Memory is reclaimed by the GC, not automatically

### What Goes on Stack
- Local value type variables: `int x = 5;`
- Value type method parameters: `void Foo(int x)`
- Value type return values (temporary)
- Local variables of reference types (just the reference/object pointer):
  ```csharp
  string name = "Alice"; // name is on stack, "Alice" is on heap
  ```
- Stack-allocated arrays: `Span<int> arr = stackalloc int[100];`
- Method call frames

### What Goes on Heap
- Reference type objects: `var list = new List<int>();`
- Boxing: `object o = 42;` (42 is boxed on the heap)
- Closures: variables captured by lambdas
  ```csharp
  int offset = 10;
  Func<int, int> add = x => x + offset; // offset is captured in a closure on the heap
  ```
- String objects (including string literals, though interns are shared)
- Dynamic allocations: `new`, `Array.CreateInstance()`, etc.

### Stack Overflow
- Occurs when the stack exceeds its size limit (usually 1 MB)
- Causes: infinite recursion, very deep recursion, large stack-allocated arrays
  ```csharp
  void Recurse() { Recurse(); } // StackOverflowException
  ```
- `StackOverflowException` is **uncatchable** — the process terminates
- Prevention:
  - Limit recursion depth
  - Use iterative algorithms instead of recursive ones
  - Use `stackalloc` carefully
  - Increase stack size with `/STACK` linker option (rarely done)

### Out of Memory
- Occurs when the GC cannot allocate enough contiguous memory
- Causes: memory leaks (GC roots holding references), excessive memory usage, fragmented heap
- `OutOfMemoryException` is catchable
- Prevention:
  - Dispose objects properly
  - Use `using` statements
  - Release references when done
  - Use object pooling for frequent allocations
  - Monitor memory usage

---

## Boxing and Unboxing

### Boxing
- The process of converting a **value type** to `object` or any **interface type**
- Involves:
  1. Allocating memory on the **heap** (enough for the value type + object header)
  2. **Copying** the value type data into the newly allocated heap memory
  3. Returning a **reference** to the heap object
- Creates a **new object** every time — the original value is not modified
- Example:
  ```csharp
  int x = 42;
  object boxed = x; // Boxing: x is copied to the heap
  ```

### Unboxing
- The process of converting `object` (or interface) back to a **value type**
- Involves:
  1. **Type checking** — verifying the object is actually the target value type
  2. **Copying** the value from the heap to the stack
- Throws `InvalidCastException` if the type doesn't match
- Example:
  ```csharp
  object boxed = 42;
  int y = (int)boxed; // Unboxing: type check + copy from heap to stack
  ```

### Performance Cost of Boxing
- **Heap allocation** — every box allocates memory that must be garbage collected
- **Memory copy** — the value is copied twice (box and unbox)
- **GC pressure** — more objects on the heap means more GC work
- **Cache misses** — heap objects may not be in CPU cache
- Benchmark: boxing/unboxing can be **50-100x slower** than direct value type operations
- In tight loops, boxing can cause significant performance degradation

### When Boxing Happens Implicitly
- Storing a value type in an `object` variable
- Calling an interface method on a value type that doesn't implement it
- Passing value types to methods that accept `object` or `params object[]`
  ```csharp
  Console.WriteLine(42); // int is boxed to call WriteLine(object)
  ```
- Adding value types to non-generic collections (`ArrayList`, `Hashtable`)
- Using value types with `dynamic`
- Using value types in string formatting: `$"Value is {myStruct}"`

### How to Avoid Boxing
- Use **generics** (`List<T>` instead of `ArrayList`)
- Use **generic interfaces** (`IComparable<T>` instead of `IComparable`)
- Use `Span<T>` and `ReadOnlySpan<T>` for zero-allocation operations
- Use `Memory<T>` for heap-friendly span-like operations
- Use `stackalloc` for small stack allocations
- Use `ConcurrentBag<T>`, `Queue<T>`, `Stack<T>` instead of non-generic versions
- Use `ArrayPool<T>` for temporary buffers
- Consider using `Unsafe.SizeOf<T>()` for compile-time size checks

---

## Garbage Collector

### What GC Does
- Provides **automatic memory management** for .NET applications
- Tracks which objects are still in use (reachable) and which are no longer needed
- Reclaims memory from unreachable objects
- Manages heap memory allocation and deallocation
- Compacts the heap to reduce fragmentation
- Handles finalization for objects with finalizers
- Manages the Large Object Heap (LOH)

### Generational Collection
- The GC uses a **generational model** based on the observation that **most objects die young**
- Three generations:
  - **Gen 0**: Newly allocated objects. Collected most frequently. Most objects die here.
  - **Gen 1**: Objects that survived one Gen 0 collection. Acts as a buffer between short-lived and long-lived objects.
  - **Gen 2**: Objects that survived Gen 1 collection. Collected least frequently. Long-lived objects live here.

### Why Generational
- Most objects are short-lived (temporary variables, iterators, closures)
- Scanning only recently allocated objects (Gen 0) is much faster than scanning the entire heap
- Gen 0 collections are very fast and frequent
- Gen 2 collections are expensive and infrequent
- The cost of collection is proportional to the number of **live** objects, not dead ones
- Typical pattern: collect Gen 0 frequently, promote survivors, eventually collect Gen 2 rarely

### Large Object Heap (LOH)
- Objects **>= 85,000 bytes** are allocated on the LOH
- LOH is treated as **Gen 2** (only collected during Gen 2 collections)
- LOH is **not compacted by default** (to avoid copying large blocks of memory)
- Can be forced to compact with `GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce`
- LOH fragmentation can cause `OutOfMemoryException` even when free memory exists
- Large arrays and large string allocations are common LOH residents
- .NET 4.5.1+ allows LOH to be compacted on demand

### GC.Collect()
- Forces garbage collection of all generations
- **Almost never call it** in production code:
  - The GC is optimized to collect at the right time
  - Calling it manually can cause unnecessary full collections
  - Can actually **hurt performance** by disrupting GC's heuristics
- Acceptable use cases:
  - Benchmarks and performance testing
  - After releasing a large number of objects (rare)
  - Diagnosing memory issues
  - In test code to verify cleanup behavior
- `GC.Collect(int generation, GCCollectionMode mode, bool blocking)` provides more control

### IDisposable vs GC for Deterministic Cleanup
- **GC / Finalizer**: Non-deterministic — you cannot predict when the finalizer runs
  - Use for releasing **unmanaged resources** as a safety net
  - Finalizer runs on a separate thread, blocking other finalizations
  - Adds overhead (object must be promoted to Gen 1+)
- **IDisposable**: Deterministic — you control when cleanup happens
  - Use for releasing **managed resources** (database connections, file handles, etc.)
  - Call `Dispose()` explicitly or use `using` statement
  - No performance overhead
  - Pattern:
    ```csharp
    public class MyResource : IDisposable
    {
        private bool _disposed = false;

        public void Dispose()
        {
            Dispose(true);
            GC.SuppressFinalize(this);
        }

        protected virtual void Dispose(bool disposing)
        {
            if (!_disposed)
            {
                if (disposing)
                {
                    // Free managed resources
                }
                // Free unmanaged resources
                _disposed = true;
            }
        }

        ~MyResource()
        {
            Dispose(false);
        }
    }
    ```

### WeakReference and WeakReference<T>
- **WeakReference**: Allows holding a reference to an object **without preventing GC** from collecting it
- The object can be collected at any time when there are no strong references
- Use cases:
  - Caching: keep objects in cache only if they haven't been collected
  - Object tracking: track objects without preventing their collection
  - Memory-sensitive applications
- `WeakReference<T>` (generic, available in .NET Framework 4.5+):
  - Type-safe weak reference
  - Better performance than non-generic `WeakReference`
- Example:
  ```csharp
  var data = new byte[1024 * 1024]; // 1 MB
  var weakRef = new WeakReference<byte[]>(data);
  data = null; // Release strong reference

  GC.Collect();

  if (weakRef.TryGetTarget(out byte[] retrieved))
  {
      Console.WriteLine("Object survived GC");
  }
  else
  {
      Console.WriteLine("Object was collected");
  }
  ```

---

## GC Internals

### Mark and Sweep Algorithm
- Two-phase process:
  1. **Mark phase**: Starting from GC roots, traverse all reachable objects and mark them as live
  2. **Sweep phase**: Scan the heap and reclaim memory from unmarked (unreachable) objects
- After sweeping, the heap may be **fragmented** (holes between live objects)
- Compaction is performed to reduce fragmentation (see below)
- The GC uses a **card table** to track cross-generational references (Gen 2 → Gen 0) to avoid scanning the entire Gen 2 on every Gen 0 collection

### GC Roots
- The starting points for reachability analysis — if an object is reachable from any root, it is alive:
  - **Static fields**: `static object _instance;` — always reachable as long as the assembly is loaded
  - **Local variables and parameters**: In active method call frames on the stack
  - **CPU registers**: Currently active values in registers
  - **Thread stack frames**: All active stack frames for all threads
  - **Finalization queue**: Objects with finalizers that haven't been finalized yet
  - **GCHandle entries**: Objects pinned or referenced via `GCHandle`
  - **Managed stack references**: References from the JIT compiler's stack walks
  - **Application domains**: Domain-wide roots
  - **Runtime infrastructure**: Internal CLR data structures
- If an object is not reachable from any root, it is eligible for garbage collection

### Compaction
- After sweep, the heap may have **fragmentation** (free gaps between live objects)
- Compaction moves live objects together to fill gaps
- Benefits:
  - Creates larger contiguous free blocks
  - Reduces fragmentation
  - Improves allocation performance (bump pointer allocation)
- Gen 0 and Gen 1 are **always compacted** (small heaps, acceptable cost)
- Gen 2 is compacted **on demand** or when fragmentation is severe
- LOH is **not compacted by default** (large objects are expensive to move)
- Compaction is expensive:
  - Requires updating all references to moved objects
  - Requires GC suspension (all threads must be paused)
  - Large heaps take longer to compact

### Concurrent / Background GC
- .NET uses **concurrent GC** to minimize pause times
- Most of the GC work runs on a **separate background thread**
- Application threads are only briefly suspended at certain points
- Gen 0 and Gen 1 collections are very fast (few milliseconds)
- Gen 2 collections can run concurrently (background GC)
- Background GC allows the application to continue running while Gen 2 collection is in progress
- Configurable via `GCSettings latency mode`

### Server vs Workstation GC
- **Workstation GC** (default for desktop apps):
  - Single heap per process
  - GC runs on the requesting thread
  - Lower memory overhead
  - Better for UI responsiveness
  - One GC thread
- **Server GC** (default for ASP.NET, services):
  - One heap per logical processor (CPU core)
  - Dedicated GC threads (one per heap)
  - More aggressive collection
  - Higher throughput for server workloads
  - Higher memory usage (each heap has its own allocation context)
  - Better for high-throughput, multi-threaded applications
- Configure via `runtimeconfig.json` or `app.config`:
  ```xml
  <configuration>
    <runtime>
      <gcServer enabled="true"/>
      <gcConcurrent enabled="true"/>
    </runtime>
  </configuration>
  ```

### GC Latency Modes
- `GCSettings.LatencyMode` controls how aggressive the GC is:
  - `LowLatency`: Minimizes Gen 2 collections. Use for latency-sensitive operations.
  - `SustainedLowLatency`: Minimizes Gen 2 collections for longer periods. Use for sustained low-latency scenarios.
  - `Batch`: Maximizes throughput. GC runs at specific times. Good for batch processing.
  - `Interactive`: Balanced mode (default). Prioritizes app responsiveness.
  - `NoGCRegion`: A region where GC is disabled (use with extreme caution)
- Example:
  ```csharp
  var previousMode = GCSettings.LatencyMode;
  GCSettings.LatencyMode = GCLatencyMode.LowLatency;

  try
  {
      // Critical latency-sensitive code
  }
  finally
  {
      GCSettings.LatencyMode = previousMode;
  }
  ```

---

## Memory Management Best Practices

### Avoid Large Allocations in Hot Paths
- Allocate objects outside of loops when possible
- Reuse objects instead of creating new ones
- Be aware of implicit allocations:
  - Lambda captures (each capture creates a closure class)
  - String concatenation (each `+` creates a new string)
  - LINQ queries (iterators, closures)
  - Boxing (value type → object)
  - `params` arrays
- Profile allocations in hot paths using `dotMemory`, `PerfView`, or `dotnet-trace`

### Use ArrayPool<T> for Temporary Buffers
- `System.Buffers.ArrayPool<T>` provides reusable arrays
- Avoids repeated allocation and GC of large arrays
- Ideal for temporary buffers that are frequently allocated and released
- Example:
  ```csharp
  var pool = ArrayPool<byte>.Shared;
  byte[] buffer = pool.Rent(1024);
  try
  {
      // Use buffer
  }
  finally
  {
      pool.Return(buffer);
  }
  ```
- The returned array may contain old data — always clear it if needed
- Pool is thread-safe and designed for concurrent access

### Use Span<T> and Memory<T> for Zero-Allocation Slicing
- `Span<T>` provides a type-safe, memory-safe view over a contiguous region of memory
- No heap allocation — `Span<T>` is a `ref struct` (stack-only)
- Perfect for slicing arrays, strings, and buffers without copying
- `Memory<T>` is the heap-friendly counterpart that can be used in async methods
- Example:
  ```csharp
  int[] array = { 1, 2, 3, 4, 5 };
  Span<int> slice = array.AsSpan().Slice(2, 3); // {3, 4, 5} — no allocation
  ```

### Use stackalloc for Small Stack Allocations
- Allocates memory on the stack instead of the heap
- Zero GC pressure — memory is freed when the method returns
- Limit to small sizes (typically under 1 KB) to avoid stack overflow
- Returns `Span<T>` or `Span<byte>`
- Example:
  ```csharp
  Span<int> numbers = stackalloc int[100]; // 400 bytes on stack
  for (int i = 0; i < numbers.Length; i++)
  {
      numbers[i] = i;
  }
  ```

### Avoid Finalizers
- Finalizers have significant performance overhead:
  - Object must be promoted to Gen 1 or higher
  - Finalizer runs on a separate thread, blocking other finalizations
  - Two-pass GC: first pass queues for finalization, second pass actually collects
- Only use finalizers when you **must** release unmanaged resources as a safety net
- Prefer `IDisposable` + `using` for deterministic cleanup
- If you do have a finalizer, always implement `IDisposable` with the full pattern

### Use Object Pooling
- `Microsoft.Extensions.ObjectPool` or custom pooling
- Reuse expensive objects instead of creating new ones
- Good for:
  - Large objects
  - Objects with expensive initialization
  - Frequently allocated/deallocated objects
  - Objects that hold unmanaged resources
- Example:
  ```csharp
  var pool = new DefaultObjectPoolProvider().CreateStringBuilderPool();
  var sb = pool.Get();
  try
  {
      sb.Append("Hello");
      string result = sb.ToString();
  }
  finally
  {
      pool.Return(sb);
  }
  ```

### Pre-size Collections
- Specify initial capacity when creating collections
- Avoids repeated internal array resizing and copying
- Example:
  ```csharp
  // Bad: internal array resizes multiple times
  var list = new List<int>();
  for (int i = 0; i < 10000; i++) list.Add(i);

  // Good: allocated once
  var list = new List<int>(10000);
  for (int i = 0; i < 10000; i++) list.Add(i);
  ```
- Same applies to `Dictionary`, `HashSet`, `StringBuilder`, etc.
- `StringBuilder` has a constructor that takes initial capacity

### Additional Best Practices
- **Use `readonly struct`** when possible — allows the compiler to avoid defensive copies
- **Use `in` parameters** for large structs — passes by reference, avoids copying
- **Use `ref return` and `ref struct`** — avoid unnecessary copies
- **String interning** — reuse string literals; use `string.Intern()` for frequently used strings
- **Avoid `dynamic`** — causes boxing and runtime dispatch overhead
- **Use `ValueTask`** instead of `Task` for synchronous completions — avoids allocation
- **Use `Span<T>`-based APIs** — modern .NET APIs accept spans for zero-allocation processing
- **Monitor with `GC.GetTotalMemory()`** — useful for debugging memory issues

---

## Span<T> and Memory<T>

### Span<T> is Stack-Only (ref struct)
- `Span<T>` is a `ref struct` — it can only exist on the stack
- Cannot be stored in fields of classes or structs (unless the struct is also a `ref struct`)
- Cannot be used as:
  - Fields in a class or regular struct
  - Return values from async methods
  - Elements of arrays
  - Generic type arguments (constraints prevent this)
- Size: typically 16-24 bytes on the stack (pointer + length)
- Zero allocation — it's just a view over existing memory
- Represents a contiguous region of memory (array, string, stackalloc, etc.)

### Memory<T> Can Be Heap-Allocated
- `Memory<T>` is a regular struct (not `ref struct`)
- Can be stored in fields, used in async methods, and passed to lambdas
- Can wrap:
  - Arrays (`T[]`)
  - Native memory (`NativeMemory`)
  - `MemoryManager<T>` for custom memory sources
  - `string` (via `MemoryMarshal`)
- Slightly more overhead than `Span<T>` (may involve array + offset + length)
- Use when you need to **defer** or **store** a memory view

### Span<T> Cannot Be Captured in Lambdas
- Since `Span<T>` is a `ref struct`, it **cannot be captured** in lambda expressions or local functions
- This is because lambdas may be converted to heap-allocated delegate objects
- Use `Memory<T>` when you need to capture memory in lambdas:
  ```csharp
  // This won't compile:
  Span<int> span = stackalloc int[10];
  Action action = () => span[0] = 1; // Error: cannot capture Span<T>

  // Use Memory<T> instead:
  Memory<int> memory = new int[10];
  Action action = () => memory.Span[0] = 1; // Works
  ```

### Why Span<T> Exists
- Provides **zero-allocation slicing** of arrays, strings, and buffers
- Before `Span<T>`, slicing required creating new arrays or using `ArraySegment<T>` (which has overhead)
- Enables high-performance, allocation-free string processing:
  ```csharp
  // Old way: allocates new string
  string substring = input.Substring(5, 10);

  // New way: zero allocation
  ReadOnlySpan<char> slice = input.AsSpan().Slice(5, 10);
  ```
- Enables safe, bounds-checked memory access without `unsafe` code
- Bridges the gap between managed and unmanaged memory
- Powers modern .NET APIs: `Utf8JsonReader`, `PipeReader`, `SearchValues<T>`, etc.

---

## Nullable Types and Reference Types

### Nullable Value Types (Nullable<T> / T?)
- Value types cannot normally be `null`
- `Nullable<T>` wraps a value type to allow `null`
- Shorthand: `T?` (e.g., `int?`, `DateTime?`, `bool?`)
- Has `HasValue` and `Value` properties
- Unboxing to nullable: handles `null` gracefully
- Example:
  ```csharp
  int? age = null;
  if (age.HasValue)
  {
      Console.WriteLine(age.Value);
  }

  // Null coalescing
  int actualAge = age ?? 0;

  // Null conditional
  int length = name?.Length ?? 0;
  ```
- Nullable value types are still structs (value type semantics)
- Two representations of `null`:
  - `HasValue = false` (default)
  - `Value = default(T)` with `HasValue = false`

### Nullable Reference Types (C# 8+)
- Allows you to express whether a reference type is expected to be `null`
- Enabled with `#nullable enable` or project-level setting
- Two types:
  - `T?` — may be null (nullable)
  - `T` — should not be null (non-nullable)
- This is a **compile-time** feature only — no runtime enforcement
- The compiler issues warnings (not errors) for potential null issues
- Example:
  ```csharp
  #nullable enable

  string name = "Alice";    // Non-nullable — compiler warns if null
  string? nickname = null;  // Nullable — explicitly allows null

  void ProcessName(string name) { }   // Expects non-null
  void ProcessNick(string? nick) { }  // Accepts null
  ```
- The `?` annotation changes how the compiler analyzes null flows
- Nullable reference types are erased at runtime — `string?` and `string` are the same type

### #nullable enable/disable
- `#nullable enable` — enables nullable reference type analysis from this point
- `#nullable disable` — disables it (default for older code)
- Can be set per-file or project-wide in `.csproj`:
  ```xml
  <PropertyGroup>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  ```
- Use `#nullable restore` to revert to the project-level setting
- Migration strategy:
  1. Enable at project level
  2. Fix warnings incrementally
  3. Use `#nullable disable` for files you haven't migrated yet
- Common annotations: `?`, `null!` (null-forgiving), `[NotNull]`, `[MaybeNull]`

---

## Interview Questions

### Q1: What is the difference between stack and heap?
- **Stack**: Fast, LIFO, automatic cleanup, thread-local, stores value type locals and method frames
- **Heap**: Dynamic, GC-managed, shared across threads, stores reference type objects and boxed values
- Stack allocation is essentially free (just moving a pointer); heap allocation requires finding free space and GC tracking

### Q2: What is boxing and unboxing?
- **Boxing**: Converting a value type to `object` — allocates on heap, copies value, returns reference
- **Unboxing**: Converting `object` to value type — type check, then copy from heap to stack
- Both are expensive: boxing causes heap allocation + GC pressure; unboxing requires type verification

### Q3: What are the three generations of the GC?
- **Gen 0**: Newly allocated objects. Collected most frequently. Most objects die here.
- **Gen 1**: Survived one Gen 0 collection. Acts as a buffer/intermediate generation.
- **Gen 2**: Survived Gen 1 collection. Long-lived objects. Collected least frequently.
- The generational model is based on the "weak generational hypothesis" — most objects die young

### Q4: What is the Large Object Heap (LOH)?
- Objects >= 85,000 bytes are allocated on the LOH
- LOH is treated as Gen 2 (only collected during Gen 2 collections)
- LOH is **not compacted by default** (expensive to move large objects)
- LOH fragmentation can cause allocation failures even with free memory available

### Q5: When should you call GC.Collect()?
- **Almost never** in production code
- The GC is self-tuning and optimizes collection timing
- Manual calls can disrupt GC heuristics and hurt performance
- Acceptable: benchmarks, diagnostics, after releasing massive amounts of objects

### Q6: What is the difference between IDisposable and finalizers?
- **IDisposable**: Deterministic cleanup. You call `Dispose()` explicitly or use `using`. No performance overhead. Handles managed resources.
- **Finalizer**: Non-deterministic. GC calls it at unpredictable time. Runs on separate thread. Object is promoted to Gen 1+. Only for unmanaged resources as safety net.
- Always prefer `IDisposable` over finalizers

### Q7: What is Span<T> and why does it exist?
- `Span<T>` is a `ref struct` that provides a type-safe, bounds-checked view over contiguous memory
- Enables **zero-allocation slicing** of arrays, strings, and buffers
- Stack-only: cannot be stored in heap objects, captured in lambdas, or used in async methods
- Exists to eliminate unnecessary heap allocations in performance-critical code

### Q8: Why are strings immutable in C#?
- `string` is a reference type but behaves like a value type (immutable)
- Immutability enables: string interning, thread safety, safe use as dictionary keys, security
- Each modification creates a new string — use `StringBuilder` for concatenation in loops
- `string.Intern()` reuses a single instance of identical strings

### Q9: What is the difference between a value type and a reference type?
- **Value type**: Stored on stack, copied by value, each variable has its own copy, inherits from `ValueType`
- **Reference type**: Stored on heap, variable holds a reference (pointer), multiple variables can reference the same object, inherits from `Object`

### Q10: When would you use a struct instead of a class?
- When the type is **small** (under ~16 bytes), **immutable**, and represents a **single value**
- When you want to **avoid heap allocation** and GC pressure
- When the type won't be used in polymorphism
- Examples: `Point`, `DateTime`, `Range`, `KeyValuePair`

### Q11: What is the difference between ArrayPool<T> and a regular array?
- `ArrayPool<T>` provides **reusable** array instances from a pool
- Avoids repeated allocation and GC of large temporary arrays
- You `Rent()` an array, use it, then `Return()` it to the pool
- Returned arrays may contain old data — always clear or overwrite before use

### Q12: How does the GC handle memory fragmentation?
- After sweep, free gaps exist between live objects
- **Compaction** moves live objects together to fill gaps
- Gen 0 and Gen 1 are always compacted
- Gen 2 is compacted on demand or when severe fragmentation occurs
- LOH is not compacted by default (expensive to move large objects)

### Q13: What are GC roots?
- The starting points for reachability analysis
- Include: static fields, local variables on active stack frames, CPU registers, thread stacks, finalization queue, GCHandle entries
- An object is alive if reachable from any GC root; unreachable objects are collected

### Q14: What is the difference between server and workstation GC?
- **Workstation**: Single heap, GC on requesting thread, lower memory overhead, better for UI
- **Server**: One heap per logical processor, dedicated GC threads, higher throughput, better for server workloads
- Configure via `runtimeconfig.json` or `app.config`

### Q15: What happens when a finalizer throws an exception?
- The exception is caught by the CLR and **ignored** (logged as an unhandled exception)
- The finalizer terminates for that object
- Other finalizers may still run, but the finalization thread may be affected
- Never throw exceptions from finalizers

### Q16: What is the purpose of GC.SuppressFinalize()?
- Tells the GC that the object has already been cleaned up via `Dispose()`
- Prevents the GC from queuing the object for finalization
- Performance optimization: avoids the overhead of running the finalizer
- Always call `GC.SuppressFinalize(this)` in the `Dispose()` method

### Q17: Can value types be null?
- Normal value types cannot be `null`
- `Nullable<T>` (or `T?`) wraps a value type to allow `null`
- Reference types (which are always heap-allocated) can be `null`
- With nullable reference types (`#nullable enable`), reference types have a `?` annotation to indicate nullable

### Q18: What is the 85,000-byte threshold for LOH?
- Objects >= 85,000 bytes are allocated on the Large Object Heap
- The threshold was chosen based on page size and typical allocation patterns
- LOH objects are only collected during Gen 2 collections
- LOH is not compacted by default

### Q19: How do closures affect memory?
- When a lambda captures a local variable, the compiler generates a **closure class** on the heap
- The closure class holds references to all captured variables
- The closure object lives as long as the delegate exists
- This can cause unexpected allocations and extend the lifetime of captured variables
- Use `static` lambdas (C# 9+) when no captures are needed:
  ```csharp
  Func<int, int> add = static x => x + 1; // No closure allocation
  ```

### Q20: What is the difference between weak and strong references?
- **Strong reference**: Prevents GC from collecting the object. Default for all references.
- **Weak reference**: Allows GC to collect the object when no strong references exist. Object may be collected at any time.
- `WeakReference<T>` provides type-safe weak references
- Use cases: caches, object tracking, memory-sensitive applications

### Q21: Why is string concatenation in loops bad for performance?
- Each `+` on strings creates a **new string object** on the heap
- In a loop, this creates N intermediate string objects, all of which become garbage
- Use `StringBuilder` for building strings in loops:
  ```csharp
  // Bad
  string result = "";
  for (int i = 0; i < 1000; i++) result += i.ToString();

  // Good
  var sb = new StringBuilder();
  for (int i = 0; i < 1000; i++) sb.Append(i);
  string result = sb.ToString();
  ```

### Q22: What is the difference between Memory<T> and Span<T>?
- **Span<T>**: Stack-only (`ref struct`). Zero allocation. Cannot be stored in fields, captured in lambdas, or used in async methods.
- **Memory<T>**: Regular struct. Can be stored in fields, captured in lambdas, and used in async methods. May involve heap allocation for some sources.
- Use `Span<T>` when possible; use `Memory<T>` when you need to store or defer the memory view.

### Q23: What does "GC latency mode" control?
- Controls how aggressively the GC collects objects
- `LowLatency`: Minimizes Gen 2 pauses. Use for latency-sensitive code.
- `SustainedLowLatency`: Same but for longer periods.
- `Batch`: Maximizes throughput. GC runs at specific times.
- `Interactive`: Balanced mode (default).
- `NoGCRegion`: Temporarily disables GC (use with extreme caution).

### Q24: What happens when you assign a value type to a variable of type object?
- **Boxing** occurs: the value type is copied to the heap, wrapped in an object header, and the reference is stored
- This is an implicit conversion that causes heap allocation
- To avoid it, use generics or `Span<T>`

### Q25: How do you prevent StackOverflowException?
- Avoid infinite recursion (use base cases)
- Limit recursion depth or convert to iterative approach
- Be careful with `stackalloc` — keep allocations small (< 1 KB)
- Increase stack size with `/STACK` linker option (last resort)
- `StackOverflowException` is **uncatchable** — the process terminates immediately

---

## Quick Reference Table

| Concept | Key Point |
|---------|-----------|
| Value type | Stack, copy by value, fast for small data |
| Reference type | Heap, copy by reference, polymorphism |
| Boxing | Value → heap object (expensive) |
| Unboxing | Heap object → value (type check + copy) |
| Gen 0 | New objects, collected frequently |
| Gen 1 | One collection survived, buffer generation |
| Gen 2 | Long-lived objects, collected rarely |
| LOH | Objects >= 85,000 bytes, not compacted |
| IDisposable | Deterministic cleanup via `Dispose()` |
| Finalizer | Non-deterministic cleanup by GC |
| Span<T> | Zero-allocation, stack-only memory view |
| Memory<T> | Heap-friendly memory view, can be stored |
| ArrayPool<T> | Reusable array pooling |
| stackalloc | Stack allocation, zero GC pressure |
| GC roots | Static fields, stack variables, CPU registers |
| Compaction | Moves objects to reduce fragmentation |
| Server GC | One heap per core, higher throughput |
| Workstation GC | Single heap, lower memory, UI-friendly |
