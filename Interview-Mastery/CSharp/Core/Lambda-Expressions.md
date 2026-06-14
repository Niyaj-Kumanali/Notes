# Lambda Expressions

---

## Overview

- **Definition:** Anonymous functions in C# that can contain expressions or statement blocks and can be converted to delegates or expression trees.
- **Why It Exists:** Enables functional-style programming, concise inline function definitions, closures (capturing variables), and is fundamental to LINQ, event handlers, and functional composition.
- **Key Concepts:** **Expression lambda** (`x => x * x`), **statement lambda** (`x => { return x * x; }`), **closure** (captured variables hoisted to a compiler-generated class), **expression tree** (`Expression<Func<T>>` — inspectable/translatable), **delegate conversion** (to `Func`, `Action`, custom delegates), and **static lambdas** (C# 9+, no capture allowed).

---

## Core Concepts

- **Lambda Syntax:** Uses `=>` operator. Expression lambdas have a single expression (no braces). Statement lambdas use `{ }` with optional `return`. Parameter types can be inferred or explicit. Discard parameters (`_`) ignore inputs (C# 9+). Natural function types (C# 10+) infer delegate type from context.

```csharp
Func<int, int> square = x => x * x;                                  // Expression lambda
Func<int, int> square = x => { Console.WriteLine($"x={x}"); return x * x; };  // Statement lambda
var parse = (string s) => int.Parse(s);                              // Natural function type
```

- **Compiler Transformation (Non-Capturing):** The compiler generates a sealed class with a static cached delegate field pointing to a static method. No allocation on subsequent calls.

```csharp
// Source: Func<int, int> square = x => x * x;
// Compiler generates:
internal class Lambda_Closure
{
    public static readonly Func<int, int> __func = new Lambda_Closure().Method;
    internal int Method(int x) => x * x;
}
```

- **Captured Variables (Closures):** When a lambda captures local variables, the compiler generates a closure class with fields for each captured variable. The lambda body becomes an instance method on the closure class. Value types are copied into the closure; reference types retain the reference (mutations visible).

```csharp
// Multiple lambdas capturing the same variable share the same closure instance
var actions = new List<Action>();
for (int i = 0; i < 5; i++)
    actions.Add(() => Console.WriteLine(i));
foreach (var a in actions) a(); // Outputs: 5 5 5 5 5 (C# 5+ hoists outside loop)
```

- **Expression Tree Lambdas:** `Expression<Func<int, int>> expr = x => x * 2;` compiles to expression node objects (not IL). Nodes can be inspected, manipulated, and translated (e.g., EF Core translates to SQL). Restrictions: no assignment operators, no `await`, no `Span<T>` captures, no `dynamic` operations.

```csharp
// Expression tree inspection
Expression<Func<int, int>> expr = x => x * 2;
var body = (BinaryExpression)expr.Body;
var left = (ParameterExpression)body.Left;    // x
var right = (ConstantExpression)body.Right;   // 2
```

---

## Common Mistakes

- **Closure over loop variable** — In C# 4 and earlier, capturing `for` loop variable captured the same variable (all iterations get the final value). C# 5+ hoists `for` loop variables outside the loop, but `foreach` always creates a new variable per iteration. To be safe, manually copy: `int copy = i; list.Add(() => Console.WriteLine(copy));`
  - **Why it looks correct:** the lambda references `i` by name and the code reads naturally, but the compiler generates a single closure instance shared across all iterations, so every lambda sees the final value of `i` after the loop completes.
- **Accidental variable capture causing memory leaks** — Capturing a large object in an event handler lambda prevents GC of that object until the delegate is unsubscribed. The closure keeps ALL captured variables alive, not just the ones used.
  - **Why it looks correct:** the lambda only references one small field from the captured object, but the closure holds a reference to the entire object graph reachable from the captured variable, preventing GC of everything.
- **Modifying captured variable after delegate creation** — The closure captures the variable, not its value. Changes after delegate creation are visible when the delegate executes.
  - **Why it looks correct:** the variable assignment appears complete at the point of delegate creation, but the closure holds a reference to the variable's storage location, so any subsequent assignment changes what the delegate reads.
- **Overuse of lambdas vs local functions** — Lambdas allocate a delegate each time (unless cached). Local functions are methods on the enclosing type — zero allocation. Prefer local functions when you don't need to pass the delegate as an argument.
  - **Why it looks correct:** a lambda and a local function appear syntactically similar, but every lambda in a hot path allocates a delegate object (and a closure if capturing), while a local function is just a method call with no heap allocation.
- **Expression tree with non-serializable captures** — If an expression tree captures a method call that the provider cannot translate, it throws `InvalidOperationException`. Keep expression tree captures to simple member access.
  - **Why it looks correct:** the expression compiles to valid `Expression` nodes, but the database provider cannot translate arbitrary method calls to SQL and throws a runtime exception when the query is executed.
- **`Span<T>` cannot be captured in lambdas** — `ref struct` types cannot be boxed (required for closure), so they cannot be captured.
  - **Why it looks correct:** `Span<T>` is used naturally in the enclosing method, but the compiler enforces this restriction at compile time — the error message "cannot use ref struct type inside lambda expression" surfaces immediately rather than causing a runtime failure.

```csharp
// Memory leak via event handler
public class Leaky
{
    public event Action OnSomething;
    public void Subscribe()
    {
        var bigData = new byte[1024 * 1024];
        OnSomething += () => Console.WriteLine(bigData.Length); // bigData can't be GC'd!
    }
}
```

---

## Key Design Considerations

- **Prefer local functions over lambdas when delegates aren't needed** — Local functions are methods on the enclosing type, zero allocation. Lambdas allocate a delegate (and closure if capturing). Convert a local function to a delegate only when necessary.
- **Use `static` lambdas (C# 9+) to prevent accidental captures** — The compiler errors if you reference `this` or local variables. Ensures the lambda is cached and prevents unintended GC roots.

```csharp
Func<int, int> f = static x => x * x; // Compiler error if capture attempted
```

- **Expression tree lambdas cannot contain** — Assignment operators (`=`), `await`, `dynamic` operations, or `Span<T>` / `ref struct` captures. These limitations exist because expression trees represent operations as nodes, not IL.
- **Closure lifetime extends GC lifetime of captured variables** — A delegate holding a closure prevents GC of all captured variables. Be mindful of large object graphs in long-lived delegates.
- **Lambda caching in hot paths** — Non-capturing lambdas are automatically cached by the compiler in a static field. For capturing lambdas, manually cache the delegate or use a struct-based approach.

```csharp
// Compiler caches non-capturing lambdas in static field
private static readonly Func<int, int> _square = static x => x * x;
```

---

## Real-World Scenarios

### Scenario 1: Dynamic Query Builder with Expression Trees
**Context:** An ETL tool needs to build dynamic filter expressions at runtime based on user-configured rules, then apply them to in-memory collections and translate to SQL for EF Core.

```csharp
public class DynamicFilterBuilder
{
    public Expression<Func<T, bool>> BuildFilter<T>(IEnumerable<FilterRule> rules)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        Expression? body = null;

        foreach (var rule in rules)
        {
            var property = Expression.Property(parameter, rule.PropertyName);
            var constant = Expression.Constant(Convert.ChangeType(rule.Value, property.Type));
            var comparison = rule.Operator switch
            {
                "==" => Expression.Equal(property, constant),
                "!=" => Expression.NotEqual(property, constant),
                ">" => Expression.GreaterThan(property, constant),
                "<" => Expression.LessThan(property, constant),
                "Contains" => Expression.Call(property, "Contains", null, constant),
                _ => throw new NotSupportedException($"Operator {rule.Operator} not supported")
            };
            body = body == null ? comparison : Expression.AndAlso(body, comparison);
        }

        return body == null
            ? x => true
            : Expression.Lambda<Func<T, bool>>(body, parameter);
    }
}

// Usage: combine expression trees for in-memory + EF Core translation
var filter = builder.BuildFilter<Product>(rules);
var results = _context.Products.Where(filter).ToList(); // EF translates to SQL WHERE
var local = inMemoryProducts.Where(filter.Compile()).ToList(); // In-memory filter
```

### Scenario 2: Memoized Caching Layer with Closures
**Context:** A reporting service calls a slow external API. Results for the same parameters should be cached with configurable TTL, using closures for clean encapsulation.

```csharp
public class MemoizedCache<TKey, TValue> where TKey : notnull
{
    private readonly ConcurrentDictionary<TKey, CachedValue> _cache = new();
    private readonly Func<TKey, Task<TValue>> _factory;
    private readonly TimeSpan _ttl;

    public MemoizedCache(Func<TKey, Task<TValue>> factory, TimeSpan ttl)
    {
        _factory = factory;
        _ttl = ttl;
    }

    public async Task<TValue> GetOrCreateAsync(TKey key)
    {
        // Fast path: check for valid cached value
        if (_cache.TryGetValue(key, out var cached) && !cached.IsExpired)
            return cached.Value;

        // Slow path: regenerate — use Lazy<Task> to prevent thundering herd
        var lazy = new Lazy<Task<CachedValue>>(async () =>
        {
            var value = await _factory(key);
            return new CachedValue(value, DateTime.UtcNow.Add(_ttl));
        });

        var newCached = _cache.GetOrAdd(key, _ => lazy.Value).Result;
        if (newCached.IsExpired) // Race: another thread may have refreshed
        {
            _cache.TryUpdate(key, await lazy.Value, newCached);
        }
        return newCached.Value;
    }

    private record CachedValue(TValue Value, DateTime ExpiresAt)
    {
        public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
    }
}

// Usage
var cache = new MemoizedCache<string, Report>(async id => await api.GetReportAsync(id), ttl: TimeSpan.FromMinutes(5));
var report = await cache.GetOrCreateAsync("report-123");
```

### Scenario 3: Pipeline Composition with Function Delegates
**Context:** An image processing pipeline needs to compose transformations (resize, watermark, filter) dynamically. Each step is a lambda, and the pipeline is composed at runtime.

```csharp
public class ImagePipeline
{
    private readonly List<Func<Image, Image>> _steps = new();
    private Func<Image, Image>? _compiled;

    public ImagePipeline AddStep(Func<Image, Image> transform)
    {
        _steps.Add(transform);
        _compiled = null; // Invalidate cache
        return this;
    }

    public Image Process(Image source)
    {
        _compiled ??= _steps.Aggregate((acc, step) => img => step(acc(img)));
        return _compiled(source);
    }
}

// Usage
var pipeline = new ImagePipeline()
    .AddStep(img => img.Resize(800, 0))
    .AddStep(img => img.ApplyWatermark("© 2026"))
    .AddStep(img => img.ApplyFilter("sepia"));

var result = pipeline.Process(sourceImage);
```

## Use Cases

- **Functional composition in LINQ** — passing predicates, selectors, and aggregators to LINQ methods
  - Lambda expressions (`x => x.Property > 5`) are the standard way to define LINQ query logic. Concise inline syntax for simple transformations.
  - **Avoid when:** the logic spans multiple lines or has side effects — a named method or local function improves readability.

- **Event handlers and callbacks** — subscribing to events with minimal boilerplate
  - Inline lambda handlers (`button.Click += (s, e) => DoSomething();`) avoid creating separate handler methods for trivial logic.
  - **Avoid when:** the handler must be unsubscribed later — store the delegate in a field for removal with `-=`.

- **Asynchronous task continuation** — defining work to run after a task completes
  - Lambda passed to `ContinueWith` or used within `async` anonymous methods. Captures closure variables for the continuation's context.
  - **Avoid when:** the continuation is complex — use `await` with a named async method instead.

- **Pipeline/chaining patterns** — sequential processing of data through transform steps
  - Compose lambdas into a processing pipeline. Each step is a lambda that transforms the output of the previous step. Enables clean separation of concerns.
  - **Avoid when:** the pipeline has branching logic (if/else) — a strategy pattern or visitor pattern is more maintainable than conditional lambdas.

- **Capturing local state in delegates** — remembering values from the enclosing scope when the delegate executes later
  - Closure captures variables by reference. Enables event handlers and callbacks that carry context without needing explicit parameter passing.
  - **Avoid when:** the captured variable changes before the delegate executes — capture a copy in a local variable to avoid unintended aliasing.

---

## Scenario-Based Questions

1. **Q: You are building a real-time event processing pipeline where 100K events/sec are routed through a chain of transforms (filter → enrich → transform → publish). Each transform is a lambda. How do you minimize allocation overhead?**
   - **A:** Use static lambdas (C# 9+ `static x => ...`) for transform steps that don't capture variables — the compiler caches the delegate in a static field, allocating once. For steps that need captured state (e.g., a threshold value), use a struct-based approach with `Func<...>` pointing to a method on a reusable struct. Avoid statement lambdas in hot paths (they always allocate). Pre-compose the pipeline using `Aggregate` into a single delegate that chains all steps — this eliminates per-event delegate dispatch overhead.

2. **Q: You have a memory leak suspected from event handlers with lambdas. The subscriber is never garbage collected. How do you diagnose and fix?**
   - **A:** Take a memory dump and analyze with `dotnet-dump analyze`: run `!dumpheap -type <>c__DisplayClass` to find closure instances. The closure holds references to ALL captured variables, including the subscriber object (`this`). Fix: use a weak event pattern (e.g., `WeakEventManager` from `Microsoft.Toolkit.Mvvm`), or manually unsubscribe: `button.Click -= OnClick`. For lambdas, store the delegate in a field and unsubscribe via `button.Click -= _handler`. Consider `static` lambdas that don't capture `this`.
   - **Interview follow-up:** If the subscriber holds the only reference to the delegate, can a weak event pattern still leak if the publisher outlives the subscriber — and if so, what's the alternative?

3. **Q: You are writing an EF Core query with a complex `Where` clause that combines optional filters. Building the expression tree dynamically is error-prone — how do you design it?**
   - **A:** Start with a `true` expression (`.Where(x => true)`). Conditionally append filter expressions using `Expression.AndAlso`. Use a reusable `AndAlso` extension method that combines two `Expression<Func<T, bool>>` by replacing parameters. For nullable filters: `if (filter.MinPrice.HasValue) query = query.Where(p => p.Price >= filter.MinPrice.Value)`. This avoids complex expression tree manipulation for simple cases. For truly dynamic rules (user-defined), build the expression tree with `Expression` APIs.

4. **Q: You are refactoring a codebase that passes lambdas to `Task.Run` in a loop. The closures capture the loop variable incorrectly. What happens and how do you fix it?**
   - **A:** If using a `for` loop in C# < 9 or with explicit delegate creation: `for (int i = 0; i < 10; i++) Task.Run(() => Work(i))` — all tasks see the final value of `i` (10). Fix: capture a copy per iteration: `int captured = i; Task.Run(() => Work(captured))`. In C# 9+, this is the default behavior for `for` loops. For `foreach`, C# 5+ already creates a new variable per iteration. Always be explicit about the capture intent.

5. **Q: You need to pass a lambda as a parameter to a method, but the lambda captures a `Span<byte>`. The compiler refuses. How do you work around this?**
   - **A:** `Span<T>` is a `ref struct` — it cannot be boxed, so it cannot be captured in a lambda's closure. Workarounds: (1) Use a local function instead — local functions can capture `Span<T>` because they're methods on the enclosing type without heap allocation. (2) Pass the span as a parameter instead of capturing it: `MemoryOwner<byte> owner; Process(buffer => HandleBuffer(owner.Span, buffer))`. (3) For async code, copy the span content to a pooled array and capture that.

6. **Q: You are profiling a hot path where a non-capturing lambda is allocated on every call despite the compiler's caching. Why?**
   - **A:** The compiler caches non-capturing lambdas when assigned to a `Func<>`/`Action<>` delegate type. However, if the lambda is converted to a custom delegate type (e.g., `Func<int, int> f = x => x * x;` is cached; `MyDelegate f = x => x * x;` may NOT be cached), the compiler may not cache. Also, if the lambda is created inside a generic method or as part of a LINQ expression, caching behavior varies. Fix: explicitly cache in a static readonly field: `private static readonly Func<int, int> _square = x => x * x;`.
   - **Interview follow-up:** How does the compiler determine whether a lambda is "the same" for caching purposes — does identical source text in two locations share one delegate, or does each location get its own?

7. **Q: You are implementing a retry mechanism using a lambda that captures a `CancellationToken`. The token changes on each retry. What happens?**
   - **A:** The closure captures the token variable, not the token value. If the variable is reassigned (e.g., creating a new linked CTS per retry), the lambda sees the latest value only if it's captured as a mutable variable. Better: capture the token at the point of lambda creation — create a local copy inside the retry loop: `var ct = currentCt; Task.Run(() => Work(ct))`. Or, pass the token as a parameter to the lambda: `Task.Run(ct => Work(ct), ct)`.

8. **Q: You need to serialize and deserialize a C# expression tree for a rules engine that runs on a different machine. How do you approach this?**
   - **A:** Use `System.Linq.Expressions` serialization via a library like `Serialize.Linq`. Alternatively, define the rules as a DSL (JSON/YAML) and build expression trees from them — this is more portable and version-tolerant. For example: represent filters as `{ "field": "Price", "op": "gt", "value": 100 }`. Build the expression tree from these rule objects. This avoids serializing compiled IL and works across .NET versions.

9. **Q: You are writing a library that uses `Expression<Func<T, bool>>` for filtering. How do you compose two expression trees with `OR` without an external library?**
   - **A:** Use `Expression.OrElse` and combine the parameters. Since the two expressions have different `ParameterExpression` instances, you must replace them: create a new parameter, invoke both expressions with the new parameter, then combine:

```csharp
public static Expression<Func<T, bool>> Or<T>(
    this Expression<Func<T, bool>> left,
    Expression<Func<T, bool>> right)
{
    var param = Expression.Parameter(typeof(T));
    var body = Expression.OrElse(
        Expression.Invoke(left, param),
        Expression.Invoke(right, param));
    return Expression.Lambda<Func<T, bool>>(body, param);
}
```

This uses `Invoke` which may not translate to SQL in LINQ-to-SQL providers. For EF Core, use `Expression.AndAlso`/`OrElse` with parameter replacement via `ParameterRebinder` — a class that walks the tree replacing parameters.

10. **Q: You are debugging why a lambda inside a loop captures the same variable for all iterations. You find the IL shows a single closure shared across all iterations. Why?**
     - **A:** The compiler hoists variables captured by multiple lambdas in the same scope into a single closure. If a `for` loop variable is captured by lambdas created in different iterations, all lambdas share the same closure instance and thus the same variable. In C# 5+, `foreach` loop variables are scoped per iteration (new variable each time). For `for` loops, the variable is scoped outside the loop body. Fix: declare a local variable inside the loop: `for (int i = 0; i < n; i++) { int copy = i; list.Add(() => Console.WriteLine(copy)); }`.
    - **Interview follow-up:** If two lambdas in the same method capture disjoint sets of variables, does the compiler generate one closure or two — and what determines the answer?

---

## Interview Questions

1. **What is a lambda expression?**
   - **A:** An anonymous function that can contain expressions or statements and can be converted to a delegate or expression tree. Syntax: `(parameters) => expression` or `(parameters) => { statements; }`.

2. **What is a closure?**
   - **A:** A lambda that captures variables from its enclosing scope. The compiler generates a class (closure) with fields for each captured variable and creates an instance on the heap. The captured variables stay alive as long as the delegate reference exists.

3. **What is the difference between `Func<T, bool>` and `Expression<Func<T, bool>>`?**
   - **A:** `Func<T, bool>` is a delegate — compiled IL that executes directly. `Expression<Func<T, bool>>` is an expression tree — a tree of node objects representing the code. Expression trees can be inspected, modified, and translated (e.g., to SQL) but must be compiled to execute.

4. **What is a static lambda (C# 9+)?**
   - **A:** A lambda declared with the `static` keyword that cannot capture variables from the enclosing scope: `static x => x * x`. The compiler errors if any capture is attempted. Guarantees zero allocation (compiler caches the delegate in a static field).

5. **Can lambdas capture `ref`, `out`, or `in` parameters?**
   - **A:** No. Lambda parameters cannot have `ref`, `out`, or `in` modifiers. Local functions support these modifiers, but lambdas don't because they can be converted to delegate types that don't support ref-like parameters.

6. **What is a method group conversion?**
   - **A:** Creating a delegate from a method name: `Func<int, int> f = int.Parse;`. The compiler resolves overloads and generates a delegate pointing to the method. If the method is static, no closure is allocated.

7. **Why can't lambdas capture `Span<T>`?**
   - **A:** `Span<T>` is a `ref struct` — it's stack-only and cannot be boxed. Lambdas capture variables by storing them in a heap-allocated closure class, which requires boxing. `ref struct` types cannot be boxed, so they cannot be captured. Local functions can capture `Span<T>` because they don't require heap allocation.

8. **Explain the difference between expression lambda and statement lambda.**
   - **A:** Expression lambda has a single expression as the body: `x => x * x`. Statement lambda uses braces with optional return: `x => { return x * x; }`. Expression lambdas can be converted to both delegates and expression trees; statement lambdas can only be delegates.

9. **What is the `Invoke` method on an expression tree used for?**
   - **A:** `Expression.Invoke` applies a lambda expression to arguments. Used when composing expression trees: `Expression.Invoke(leftExpression, argumentExpression)`. EF Core can translate `Invoke` in some cases, but it may cause client evaluation. Prefer direct parameter replacement for better SQL translation.

10. **How does the compiler cache non-capturing lambdas?**
    - **A:** The compiler generates a sealed class with a static readonly field holding the cached delegate. The delegate is created once via a static initializer. All uses of the same non-capturing lambda expression reference this cached instance. This applies per lambda expression literal, not per identical content.

---

## Developer Recommendations

- **Prefer local functions over lambdas when you don't need a delegate** — Local functions are methods on the enclosing type with zero allocation. Lambdas allocate a delegate (and a closure if capturing). A local function can always be converted to a delegate when needed (`var d = (Func<int, int>)LocalFunc;`), but this still allocates. Use lambdas only when passing as an argument to a method that expects a delegate.
  - **Production story:** A high-throughput API once used lambdas for every request validation step, allocating thousands of delegates per second and causing frequent Gen 0 GC collections — switching to local functions eliminated the allocation entirely.

- **Use `static` lambdas (C# 9+) to prevent accidental captures** — Marking a lambda as `static` forces the compiler to error if `this` or local variables are referenced. This prevents unintended closures that extend object lifetimes and prevents delegate re-allocation. Make it a habit to start with `static` and remove it only when a capture is intentional.

- **Avoid capturing large objects in event handler lambdas** — The closure keeps ALL captured variables alive as long as the delegate is subscribed. If an event handler lambda captures `this` (implicitly through instance method access), the entire object cannot be GC'd until the event is unsubscribed. Store the delegate in a field for manual unsubscription or use weak event patterns.
  - **Production story:** A chat service once leaked hundreds of megabytes because a long-lived `ConnectionManager` event handler captured a `Room` object with a large message buffer — the `Room` could not be collected until the `ConnectionManager` was disposed, keeping the buffer alive indefinitely.

- **Prefer expression trees over reflection for dynamic member access** — Building an `Expression<Func<T, TResult>>` and compiling it is ~10x faster than `PropertyInfo.GetValue` for repeated access. The compilation cost is amortized over subsequent invocations. Use for serializers, mappers, and dynamic property accessors in hot paths.

- **Be explicit about captured variable lifetime** — A lambda that captures local variables extends their lifetime to match the delegate's lifetime. In long-lived delegates (e.g., `BackgroundService` loops), captured state accumulates. Use scope-limiting: declare captured variables in the narrowest possible scope, or use static lambdas with explicit parameters to pass state.

- **Use expression tree manipulation for dynamic queries, not string concatenation** — Building dynamic LINQ queries by concatenating strings is fragile and insecure. Use `System.Linq.Expressions` APIs (`Expression.AndAlso`, `Expression.Equal`, etc.) to build filter predicates programmatically. Libraries like `System.Linq.Dynamic.Core` can help for simple cases.

- **Prefer `Func<...>` / `Action<...>` over custom delegate types in public APIs** — Using standard delegate types reduces learning curve and improves interoperability with LINQ and the BCL. Custom delegate types require explicit conversion. Reserve custom delegates for cases where named parameters improve readability significantly (e.g., `delegate bool Filter<in T>(T item)`).

---

## Lambda Allocation Cost

| Lambda Type | Closure | Delegate Alloc | Caching |
|---|---|---|---|
| Static (no capture) | None | Once | Compiler caches |
| Instance method ref | None | Once | Compiler caches |
| Capturing (value) | New closure | New delegate | Per evaluation |
| Capturing (ref) | Same closure | New delegate | Shared |
| Expression tree | Expression nodes | New nodes | Always allocates |

## Lambda vs Local Function

| Aspect | Lambda | Local Function |
|---|---|---|
| Delegate creation | Yes (if assigned) | No (unless converted) |
| Allocation | Closure + delegate | Zero (method on type) |
| Recursion | No (use delegate ref) | Yes |
| `ref`/`out` params | No | Yes |
| `Span<T>` capture | No | Yes |
