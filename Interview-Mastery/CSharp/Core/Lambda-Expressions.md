# Lambda Expressions

## 1. Executive Summary

Lambda expressions in C# are anonymous functions that can contain expressions or statement blocks and can be converted to delegates or expression trees. First introduced in C# 3.0 alongside LINQ, lambdas enable functional-style programming, closures, and concise inline function definitions. They are fundamental to LINQ, event handlers, asynchronous programming, and functional composition.

## 2. Core Theory

A lambda expression uses the `=>` operator (reads as "goes to"):

```csharp
// Expression lambda (single expression, no braces)
Func<int, int> square = x => x * x;

// Statement lambda (block body)
Func<int, int> square = x =>
{
    Console.WriteLine($"Squaring {x}");
    return x * x;
};

// With explicit parameter types
Func<int, int> add = (int a, int b) => a + b;

// With discard parameters (C# 9+)
Func<int, int, int> add = (_, _) => 0;

// Natural function type (C# 10+)
var parse = (string s) => int.Parse(s); // inferred as Func<string, int>
```

Lambda conversions:
- To a **delegate type** (`Func<>`, `Action<>`, custom delegate)
- To an **expression tree** (`Expression<Func<>>`)

## 3. Under-the-Hood Deep Dive

### Compiler Transformations

```csharp
// Source code
Func<int, int> square = x => x * x;
int result = square(5);

// Compiler generates:
internal class Lambda_Closure
{
    public static readonly Func<int, int> __func;
    public static readonly Lambda_Closure __singleton = new();

    static Lambda_Closure()
    {
        __func = __singleton.Method;
    }

    internal int Method(int x)
    {
        return x * x;
    }
}

// Usage:
int result = Lambda_Closure.__func(5);
```

### Captured Variables (Closures)

```csharp
// When lambda captures local variables:
int factor = 10;
Func<int, int> multiplier = x => x * factor;

// Compiler generates a closure class:
internal class Closure
{
    public int factor;

    internal int Method(int x)
    {
        return x * factor;
    }
}

// Usage:
Closure c = new() { factor = 10 };
Func<int, int> multiplier = c.Method;
factor = 20;
// Note: c.factor still holds 10 (value type captured by value)
```

```csharp
// IMPORTANT: Reference type / mutable value type capture
// Multiple lambdas sharing captured variables share the same closure instance

var actions = new List<Action>();
for (int i = 0; i < 5; i++)
    actions.Add(() => Console.WriteLine(i));

foreach (var a in actions) a();
// Output: 5 5 5 5 5 (C# 5+, closure is hoisted outside the loop)
// Before C# 5, the variable was inside the loop (different behavior)

// C# 5+ fix (or use foreach which creates a new variable per iteration):
for (int i = 0; i < 5; i++)
{
    int copy = i;           // Captured per iteration
    actions.Add(() => Console.WriteLine(copy));
}
// Output: 0 1 2 3 4
```

### Expression Tree Lambda

```csharp
Expression<Func<int, int>> expr = x => x * 2;

// Compiler generates:
// ParameterExpression param = Expression.Parameter(typeof(int), "x");
// BinaryExpression body = Expression.Multiply(param, Expression.Constant(2));
// Expression<Func<int, int>> expr = Expression.Lambda<Func<int, int>>(body, param);

// You can inspect and manipulate:
var body = (BinaryExpression)expr.Body;
var left = (ParameterExpression)body.Left;   // x
var right = (ConstantExpression)body.Right;  // 2
```

## 4. Production Code Examples

```csharp
// Strategy pattern with lambdas
public class PricingService
{
    private readonly Dictionary<string, Func<Order, decimal>> _strategies = new()
    {
        ["Standard"] = o => o.Total * 0.05m,
        ["Premium"] = o => o.Total > 100 ? 0 : o.Total * 0.02m,
        ["VIP"] = o => o.Total * 0.10m
    };

    public decimal CalculateDiscount(Order order, string tier) =>
        _strategies.TryGetValue(tier, out var strategy) ? strategy(order) : 0;
}
```

```csharp
// Memoization with lambda closures
public static Func<TIn, TOut> Memoize<TIn, TOut>(Func<TIn, TOut> f) where TIn : notnull
{
    var cache = new ConcurrentDictionary<TIn, TOut>();
    return input => cache.GetOrAdd(input, f);
}

// Usage:
var expensive = Memoize((int x) =>
{
    Thread.Sleep(1000);
    return x * x;
});
var a = expensive(5); // Takes 1s
var b = expensive(5); // Returns instantly from cache
```

```csharp
// Fluent builder with lambda
public class EmailBuilder
{
    private string _to, _subject, _body;
    public EmailBuilder To(string to) { _to = to; return this; }
    public EmailBuilder Subject(string s) { _subject = s; return this; }
    public EmailBuilder Body(Func<string, string> bodyFactory)
    {
        _body = bodyFactory(_subject);
        return this;
    }
    public void Send() => Console.WriteLine($"To: {_to}, Subj: {_subject}, Body: {_body}");
}

new EmailBuilder()
    .To("user@example.com")
    .Subject("Welcome")
    .Body(subj => $"Hello, {subj}!")
    .Send();
```

```csharp
// Delegates with lambda for callbacks
public class AsyncProcessor
{
    public Task ProcessAsync<T>(IEnumerable<T> items,
        Func<T, CancellationToken, Task> processor,
        Action<T, Exception> onError,
        Action onComplete,
        CancellationToken ct = default)
    {
        return Task.Run(async () =>
        {
            foreach (var item in items)
            {
                try
                {
                    await processor(item, ct);
                }
                catch (Exception ex) when (!ct.IsCancellationRequested)
                {
                    onError(item, ex);
                }
            }
            onComplete();
        }, ct);
    }
}
```

## 5. Real-World Scenarios

**Scenario 1: Middleware Pipeline (ASP.NET Core)**
```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");
    await next();
    Console.WriteLine("After");
});
```

**Scenario 2: Event Aggregator with Weak References**
```csharp
public class EventAggregator
{
    private readonly ConcurrentDictionary<Type, List<WeakReference<Delegate>>> _handlers = new();

    public void Subscribe<T>(Action<T> handler) where T : class
    {
        _handlers.GetOrAdd(typeof(T), _ => new())
            .Add(new WeakReference<Delegate>(handler));
    }

    public void Publish<T>(T message) where T : class
    {
        if (_handlers.TryGetValue(typeof(T), out var handlers))
        {
            foreach (var wr in handlers.ToList())
            {
                if (wr.TryGetTarget(out var del))
                    ((Action<T>)del)(message);
                else
                    handlers.Remove(wr); // Cleanup dead references
            }
        }
    }
}
```

**Scenario 3: Specification Pattern**
```csharp
public static class Specification
{
    public static Func<T, bool> And<T>(this Func<T, bool> left, Func<T, bool> right) =>
        x => left(x) && right(x);

    public static Func<T, bool> Or<T>(this Func<T, bool> left, Func<T, bool> right) =>
        x => left(x) || right(x);

    public static Func<T, bool> Not<T>(this Func<T, bool> spec) =>
        x => !spec(x);
}

Func<Product, bool> isExpensive = p => p.Price > 100;
Func<Product, bool> isInStock = p => p.Stock > 0;
var filter = isExpensive.And(isInStock).Not();
```

## 6. Performance

```csharp
// Static lambda methods: cached, no allocation
Func<int, int> staticLambda = static x => x * x;  // C# 9+

// Capturing lambda: allocates closure on first invocation, cached thereafter
int factor = 10;
Func<int, int> capturing = x => x * x * factor;

// Non-capturing lambda: cached in static field via compiler
Func<int, int> nonCapturing = x => x * x;  // No GC allocation after JIT

// Benchmark: allocating lambdas in a hot loop
for (int i = 0; i < 1_000_000; i++)
{
    Process(x => x * x);  // BAD: Allocates new delegate each time
}

// FIX: Cache the delegate
static int Square(int x) => x * x;
var cached = (Func<int, int>)Square;
for (int i = 0; i < 1_000_000; i++)
    Process(cached);
```

### Allocation Costs

| Lambda Type            | Closure     | Delegate Alloc | Caching              |
|------------------------|-------------|----------------|----------------------|
| Static (no capture)    | None        | Once (cached)  | Compiler caches      |
| Instance method ref    | None        | Once (cached)  | Compiler caches      |
| Capture locals (value) | New closure | New delegate   | Per capture site     |
| Capture locals (ref)   | Same closure| New delegate   | Multiple share       |
| Expression tree lambda | Expression  | New nodes      | Always allocates     |

## 7. Security

```csharp
// Avoid capturing sensitive data in closures
string password = GetPassword();
Func<bool> checkPassword = input => input == password; // password held in closure!

// For expression trees, sanitize inputs before building
public Expression<Func<T, bool>> BuildPredicate<T>(string propertyName, object value)
{
    if (value is string s && s.Length > 200)
        throw new ArgumentException("Value too long");
    // ... build expression tree safely
}

// Prevent delegate invocation from untrusted sources
public class SafeInvoker
{
    private readonly Func<Task> _action;
    private readonly TimeSpan _timeout;

    public async Task InvokeSafelyAsync()
    {
        using var cts = new CancellationTokenSource(_timeout);
        try
        {
            var task = _action();
            await task.WithCancellation(cts.Token);
        }
        catch (OperationCanceledException)
        {
            throw new TimeoutException("Delegate timed out");
        }
    }
}
```

## 8. Common Mistakes

```csharp
// MISTAKE 1: Closure over loop variable (C# 4 and earlier behavior)
var list = new List<Action>();
for (int i = 0; i < 5; i++)
    list.Add(() => Console.WriteLine(i));
foreach (var a in list) a();
// Outputs: 5 5 5 5 5
// FIX (modern C# is fine, but for old code): capture copy per iteration

// MISTAKE 2: Accidental variable capture causing memory leaks
public class Leaky
{
    public event Action OnSomething;
    public void Subscribe()
    {
        var bigData = new byte[1024 * 1024];
        OnSomething += () => Console.WriteLine(bigData.Length); // bigData can't be GC'd!
    }
}

// MISTAKE 3: Modifying captured variable after delegate creation
int x = 5;
Func<int> f = () => x;
x = 10;
Console.WriteLine(f()); // 10, not 5!

// MISTAKE 4: Overuse of lambda expressions vs local functions
Func<int, int> f1 = x => x * x;     // Allocates delegate
int F2(int x) => x * x;              // Local function, no allocation
// Prefer local functions when you don't need to pass the delegate around

// MISTAKE 5: Expression tree with non-serializable captures
Expression<Func<int>> expr = () => GetValue(); // GetValue captured must be serializable
// FIX: only use simple member access that the provider understands

// MISTAKE 6: Returning lambda from method captures locals in surprising ways
public Func<int> CreateAdder(int a)
{
    int b = 10;
    return x => a + b + x; // a and b captured, both in closure
}
```

## 9. Senior Engineer Perspective

**1. Prefer local functions over lambdas when you don't need delegates.** Local functions are methods, not delegates — zero allocation.

**2. Use `static` lambdas (C# 9+) to prevent accidental captures.** The compiler errors if you reference `this` or locals.

```csharp
// Static lambda: cannot access instance members or locals
Func<int, int> f = static x => x * x;
```

**3. Expression tree lambdas are compile-time generated.** They cannot contain:
- Assignment operators (`=`)
- `await` (use `Expression.LambdaAsync` in preview)
- Dynamic operations
- `Span<T>` / `ref struct` captures (can't be boxed)

**4. Closure lifetime extends GC lifetime of all captured variables.** Be mindful of large object graphs in closures.

**5. Use `UnsafeAccessor` for truly high-performance scenarios** instead of reflection-based delegates.

**6. Lambda caching in hot paths:**

```csharp
// Good: compiler caches non-capturing lambdas
private static readonly Func<int, int> _square = static x => x * x;

// For capturing, consider a struct-based approach to avoid heap alloc
public readonly struct Adder
{
    private readonly int _a;
    public Adder(int a) => _a = a;
    public int Add(int b) => _a + b;
}
// Usage: new Adder(5).Add // Method group, no closure
```

## 10. Interview Questions (Easy)

1. What is a lambda expression in C#?
2. How do you write a lambda with no parameters?
3. What is the difference between an expression lambda and a statement lambda?
4. How do you specify explicit parameter types in a lambda?
5. What delegate types can a lambda be converted to?
6. What is a closure?
7. How do you pass a lambda to a method?
8. What is the `Func<T, TResult>` delegate?
9. What is the `Action<T>` delegate?
10. What is the difference between `Func` and `Action`?

## 11. Interview Questions (Medium)

1. Explain what happens when a lambda captures a local variable (compiler transformation).
2. What is the difference between `Expression<Func<T,bool>>` and `Func<T,bool>`?
3. How does the compiler cache lambdas? When are they cached vs re-allocated?
4. What is the closure problem with `for` loops and how was it fixed in C# 5?
5. Explain the difference between capturing value types and reference types.
6. What is a `static` lambda (C# 9+) and why use it?
7. Can you use `ref`, `out`, or `in` parameters in lambdas?
8. How do you create a lambda that returns `async Task`?
9. What restrictions exist for expression tree lambdas vs delegate lambdas?
10. Explain how `Func<int, int> f = int.Parse;` works (method group conversion).

## 12. Advanced Interview Questions (Hard)

1. Implement a `Memoize` function using closures and demonstrate thread safety.
2. Design an expression tree visitor that rewrites all constant string comparisons to case-insensitive.
3. Explain the memory layout of a closure class with multiple captured variables.
4. How would you implement `DynamicMethod` via `ILGenerator` to create a delegate equivalent to a lambda? Compare performance.
5. Write a lambda-based retry utility with exponential backoff using closures.
6. Explain how the C# compiler decides whether to cache a lambda in a static field vs an instance field.
7. Design a type-safe event bus using lambdas and interface dispatch.
8. How does `Nullable<T>` interact with lambda type inference?
9. Implement a currying utility using lambda expressions in C#.
10. Explain the interplay between `Span<T>` and lambdas (why ref structs can't be captured).

## 13. Interview Questions (System Design)

1. Design a rules engine where rules are expressed as lambdas and evaluated over dynamic objects.
2. Design a middleware pipeline using lambdas as middleware components.
3. Design a distributed tracing system using lambdas as span creators.
4. Design a pipeline executor that composes lambdas into a single call chain.
5. Design a functional configuration system using lambdas for lazy evaluation.
6. Design an event sourcing system where projections are lambdas.
7. Design a type-safe HTTP API client using lambdas for request configuration.
8. Design a workflow engine with lambdas as step definitions.
9. Design a caching layer that uses lambdas for cache key generation.
10. Design a policy-based authorization framework using lambdas.

## 14. Expert-Level Interview Questions (Architect)

1. Design a dynamic expression tree compiler that generates optimized IL for LINQ providers with caching and partial evaluation.
2. Architect a system that distributes lambda execution across a cluster using serialized expression trees.
3. Design a zero-allocation functional pipeline library using struct-delegates and `IFunction<in T, out TResult>` interfaces.
4. Architect a safe sandbox for executing user-supplied lambdas with resource limits and type constraints.
5. Design a system that uses expression tree rewriting to implement automatic caching, logging, and metrics for arbitrary methods.
6. Architect a functional reactive UI framework where all bindings are compile-time checked lambdas.
7. Design a distributed actor system where actors are defined as lambdas with closures serialized for migration.
8. Architect an AOP framework using expression tree weaving at compile time.
9. Design a type provider system that generates lambda-based APIs from schema definitions (like GraphQL).
10. Architect a query optimizer that rewrites expression trees using algebraic laws (commutativity, associativity, distributivity).

## 15. Debugging & Troubleshooting

```csharp
// Debugging lambdas: use breakpoints inside statement lambdas
var filtered = list.Where(x =>
{
    bool result = x > 5;  // Set breakpoint here
    return result;
});

// The "Closure" type appears in stack traces
// Look for "<>c__DisplayClass0_0" in call stack

// SOS dump analysis
// !DumpHeap -type DisplayClass  (find closure objects)
// !ClrStack -p                  (show parameter values)

// Common issues:
// - Memory leak: event handlers capturing large objects
// - Delegate allocation in hot paths: profile with ETW
// - Expression tree not translatable: "LINQ could not be translated" error
// - Unexpected shared state: multiple lambdas capturing same variable
```

## 16. Comparison Section

```
+------------------------+----------------------------+----------------------------+
| Feature                | Lambda (delegate)          | Expression Tree            |
+------------------------+----------------------------+----------------------------+
| Representation         | IL code                    | Expression node tree       |
| Execution              | Direct JIT'd               | Compiled at runtime        |
| Inspection             | Can't inspect body         | Full introspection         |
| Modification           | Can't modify               | Can rewrite nodes          |
| Translation            | Not translatable           | Translatable (SQL etc.)    |
| Closure support        | Full                        | Limited (no assignment)    |
| Allocation             | Closure + delegate          | Expression tree nodes      |
| Best for               | In-memory operations       | Query providers / analysis |
+------------------------+----------------------------+----------------------------+

+------------------------+----------------------------+----------------------------+
| Aspect                 | Lambda Expression          | Local Function             |
+------------------------+----------------------------+----------------------------+
| Syntax                 | x => x * x                 | int F(int x) => x * x;     |
| Delegate creation      | Yes (if assigned to Func)  | No (unless converted)      |
| Allocation             | Yes (delegate + closure)   | Zero (method on type)      |
| Capture                | Automatic closure          | Automatic closure          |
| Recursion              | No (must use delegate)     | Yes                        |
| Generics               | Inferred                   | Explicit                   |
| ref/in/out params      | No                         | Yes                        |
| Span<T> capture        | No                         | Yes                        |
| Conditional / loops    | Statement lambda only      | Natural                    |
+------------------------+----------------------------+----------------------------+
```

## 17. Revision Notes

- Lambda syntax: `(parameters) => expression` or `(parameters) => { statements; }`.
- Compiler generates a closure class when lambda captures variables.
- Non-capturing lambdas cached in static fields (zero allocation after first use).
- Capturing lambdas allocate a new closure and delegate each time (unless cached manually).
- `static` lambdas (C# 9+) prevent accidental captures.
- Expression tree lambdas (`Expression<Func<>>`) are compile-time generated node trees.
- Local functions are preferred over lambdas when no delegate is needed (zero allocation).
- Closure over loop variable fixed in C# 5 (`foreach` always new variable; `for` uses same).
- Delegates can point to lambdas, static methods, instance methods, or method groups.
- `UnsafeAccessor` provides low-level access without closure overhead.

## 18. Cheat Sheet

```
+------------------------------------------------------------------+
|                   LAMBDA EXPRESSIONS CHEAT SHEET                  |
+------------------------------------------------------------------+
| SYNTAX                                                            |
|  Expression:     (params) => expression                           |
|  Statement:      (params) => { statements; return x; }            |
|  No params:      () => expression                                 |
|  Single param:   x => expression  (parens optional)               |
|  Explicit types: (int x, string y) => x + y.Length               |
|  Static:         static x => x * x  (no capture)                 |
|  Async:          async (x) => await ProcessAsync(x)              |
|  Discard:        (_, _) => 0  (C# 9+)                            |
+------------------------------------------------------------------+
| DELEGATES                                                         |
|  Func<T, R>      - takes T, returns R                            |
|  Action<T>       - takes T, returns void                         |
|  Predicate<T>    - takes T, returns bool                         |
|  Expression<T>   - expression tree version                       |
+------------------------------------------------------------------+
| CAPTURE RULES                                                     |
|  Value types:    Copied into closure (unless ref)                |
|  Reference types: Reference captured (mutation visible)          |
|  this:           Captured implicitly if accessing instance       |
|  Static lambda:  No capture allowed (compiler enforced)          |
+------------------------------------------------------------------+
| COMMON PATTERNS                                                   |
|  Filter:   list.Where(x => x > 5)                                |
|  Map:      list.Select(x => x.ToString())                        |
|  Reduce:   list.Aggregate(0, (acc, x) => acc + x)                |
|  ForEach:  list.ForEach(x => Console.WriteLine(x))               |
|  Sort:     list.Sort((a, b) => a.CompareTo(b))                   |
|  Event:    button.Click += (s, e) => HandleClick();              |
+------------------------------------------------------------------+
| CLOSURE LIFETIME                                                  |
|  Closure created when lambda captures local/instance variable     |
|  Closure lives as long as the delegate reference lives            |
|  Multiple lambdas can share the same closure class                |
|  Closure keeps ALL captured variables alive (not just used)      |
+------------------------------------------------------------------+
| PERFORMANCE TIPS                                                  |
|  Use static lambda if no capture needed                          |
|  Cache delegates in static readonly fields                       |
|  Prefer local functions over lambdas when possible               |
|  Avoid capturing large objects in long-lived delegates           |
|  Expression trees always allocate (use compiled lambda for perf) |
+------------------------------------------------------------------+
