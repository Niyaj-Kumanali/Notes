# C# Delegates & Events - Interview Mastery

---

## Delegates

### What is a Delegate?

- A delegate is a **type-safe function pointer** that holds a reference to a method
- It defines a method signature that compatible methods must match
- Delegates enable **late binding** - the method to invoke is determined at runtime
- They are the foundation of events, callbacks, and LINQ in C#
- Delegates are **sealed classes** that inherit from `System.MulticastDelegate`
- Every delegate type is a distinct type - you cannot implicitly convert between delegate types even if signatures match

### How Delegates Work Internally

- When you declare a delegate, the compiler generates a class that inherits from `System.MulticastDelegate`
- The generated class contains three key fields:
  - `_target`: the object instance the method belongs to (null for static methods)
  - `_methodPtr`: a pointer to the method to invoke
  - `_invocationList`: an array of delegates used for multicast (null for single invocation)
- The delegate class has an `Invoke` method with the same signature as the delegate
- It also has `BeginInvoke` and `EndInvoke` for asynchronous invocation (legacy, rarely used now)
- When you invoke a delegate, it checks if `_invocationList` is null:
  - If null: invokes the single method via `_methodPtr`
  - If not null: iterates through the array and invokes each delegate sequentially

```csharp
// The compiler generates something conceptually like this:
class MyDelegate : MulticastDelegate
{
    public MyDelegate(object target, int methodPtr) { }

    public virtual void Invoke(string message) { }
    public virtual IAsyncResult BeginInvoke(string message, AsyncCallback callback, object state) { }
    public virtual void EndInvoke(IAsyncResult result) { }
}
```

### Creating Delegates

#### Named Method

```csharp
// Declare delegate type
public delegate void LogHandler(string message);

// Method matching the delegate signature
public static void ConsoleLog(string message)
{
    Console.WriteLine(message);
}

// Create delegate instance pointing to named method
LogHandler logger = ConsoleLog;
logger("Hello world");  // Invokes ConsoleLog
```

#### Anonymous Method (C# 2.0)

```csharp
LogHandler logger = delegate(string message)
{
    Console.WriteLine($"Log: {message}");
};

logger("Hello");  // Prints "Log: Hello"
```

#### Lambda Expression (C# 3.0)

```csharp
LogHandler logger = message => Console.WriteLine($"Log: {message}");
logger("Hello");  // Prints "Log: Hello"

// With multiple statements
LogHandler logger2 = message =>
{
    Console.WriteLine($"Timestamp: {DateTime.Now}");
    Console.WriteLine($"Log: {message}");
};
```

### Built-in Generic Delegate Types

- `Func<T, TResult>` - delegates that return a value
- `Action<T>` - delegates that return void
- `Predicate<T>` - delegates that return bool
- These eliminate the need to define custom delegate types for common patterns
- They are defined in the `System` namespace and available everywhere

### Multicast Delegates

- All delegates in C# are multicast by default (inherit from `MulticastDelegate`)
- You can combine delegates using the `+` operator or `+=`
- You can remove delegates using the `-` operator or `-=`
- The `_invocationList` array stores all combined delegates
- When invoked, each delegate in the invocation list is called sequentially

```csharp
Action<string> logger = msg => Console.WriteLine($"Logger: {msg}");
Action<string> alerter = msg => Console.Alerter(msg);
Action<string> fileWriter = msg => File.AppendAllText("log.txt", msg);

// Combine delegates
Action<string> combined = logger + alerter + fileWriter;
combined += logger;  // logger is now invoked twice

// Remove a delegate
combined -= fileWriter;
combined("Test message");  // Logger and Alerter called, Logger called again
```

### Why Multicast Delegates Return Value of Last Invoked Delegate

- When a multicast delegate has a return type (not void), only the return value of the **last** invoked delegate is preserved
- The return values of all previous delegates in the invocation list are discarded
- This is a design decision - there is no meaningful way to combine multiple return values
- The runtime iterates through `_invocationList` and stores the return value after each call, overwriting the previous one
- This is why multicast delegates with return values are discouraged - use `Action` or return `void` instead

```csharp
Func<int, int> square = x => x * x;
Func<int, int> double_ = x => x * 2;

Func<int, int> combined = square + double_;
int result = combined(5);  // Returns 10 (from double_), NOT 25 (from square)
// square(5) returns 25, but it is discarded
// double_(5) returns 10, which is the final return value
```

### Delegate Invocation vs Multicast Invocation

- **Single invocation**: delegate `_invocationList` is null, Invoke calls the method directly
- **Multicast invocation**: delegate `_invocationList` is populated, Invoke iterates and calls each
- Performance difference: multicast has slight overhead due to array iteration and null checks
- Exception behavior: if any delegate in a multicast chain throws, subsequent delegates are NOT invoked
- To invoke all delegates even when exceptions occur, you must iterate `GetInvocationList()` manually

```csharp
Action combined = Action1 + Action2 + Action3;

// Default invocation - stops on first exception
combined();

// Manual invocation - handles each delegate individually
foreach (Action d in combined.GetInvocationList())
{
    try
    {
        d();
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error: {ex.Message}");
    }
}
```

### Delegate Covariance and Contravariance

- **Covariance**: allows a method with a more derived return type to be assigned to a delegate expecting a less derived return type
- **Contravariance**: allows a method with less derived parameter types to be assigned to a delegate expecting more derived parameter types
- These are **return type variance** and **parameter type variance** respectively
- Covariance is declared with the `out` keyword on delegate type parameters
- Contravariance is declared with the `in` keyword on delegate type parameters

```csharp
// Covariance (out) - return type can be more derived
public delegate BaseShape ShapeFactory();
public delegate DerivedCircle CircleFactory();

// This works because DerivedCircle is a subtype of BaseShape
ShapeFactory factory = () => new DerivedCircle();  // Covariant return

// Contravariance (in) - parameter type can be less derived
public delegate void ShapeHandler(BaseShape shape);
public delegate void CircleHandler(DerivedCircle circle);

// This works because BaseShape handler can accept DerivedCircle
ShapeHandler handler = (BaseShape s) => Console.WriteLine(s.GetType());
CircleHandler circleHandler = handler;  // Contravariant parameter
```

---

## Built-in Delegate Types

### Func<T1,...,TResult>

- Takes one or more parameters and returns a value
- Variants: `Func<TResult>` through `Func<T1,...,T16,TResult>` (up to 16 parameters)
- The last type parameter is always the return type
- Most commonly used for LINQ queries, transformations, and functional operations

```csharp
Func<int, int, int> add = (a, b) => a + b;
Func<string> getTimestamp = () => DateTime.Now.ToString("HH:mm:ss");
Func<int, bool> isEven = x => x % 2 == 0;

// With LINQ
var numbers = new[] { 1, 2, 3, 4, 5 };
var doubled = numbers.Select(x => x * 2);  // Uses Func<int, int> internally
```

### Action<T1,...>

- Takes one or more parameters and returns void
- Variants: `Action` through `Action<T1,...,T16>` (up to 16 parameters)
- Used for side effects: logging, UI updates, event handlers
- Cannot be used where a return value is expected

```csharp
Action<string> log = message => Console.WriteLine(message);
Action<int, int> printSum = (a, b) => Console.WriteLine(a + b);
Action greet = () => Console.WriteLine("Hello!");

// Common use: event handlers
button.Click += (sender, e) => Console.WriteLine("Clicked!");
```

### Predicate<T>

- Takes a single parameter of type T and returns bool
- Exactly equivalent to `Func<T, bool>` but more semantic
- Used in filtering operations: `List<T>.Find`, `List<T>.Exists`, `List<T>.RemoveAll`
- Conveys intent: "this function determines a boolean condition"

```csharp
Predicate<int> isPositive = x => x > 0;
Predicate<string> isEmpty = string.IsNullOrEmpty;

var numbers = new List<int> { -1, 2, -3, 4, -5 };
int firstPositive = numbers.Find(isPositive);  // Returns 2
bool hasNegative = numbers.Exists(x => x < 0);  // Uses Predicate<int>
int removed = numbers.RemoveAll(x => x < 0);  // Removes 3 negatives
```

### Comparison<T>

- Takes two parameters of type T and returns int
- Used for custom sorting: `List<T>.Sort`, `Array.Sort`
- Returns negative if first < second, zero if equal, positive if first > second

```csharp
Comparison<string> byLength = (a, b) => a.Length.CompareTo(b.Length);
Comparison<string> reverseAlpha = (a, b) => string.Compare(b, a, StringComparison.Ordinal);

var names = new List<string> { "Charlie", "Alice", "Bob" };
names.Sort(byLength);  // Bob, Alice, Charlie
```

### Converter<TInput, TOutput>

- Takes a single parameter of type TInput and returns TOutput
- Used for type conversion: `List<T>.ConvertAll`, `Array.ConvertAll`
- Replaces manual loops for type transformations

```csharp
Converter<string, int> toInt = s => int.Parse(s);
Converter<int, string> toString = x => x.ToString();

var strings = new[] { "1", "2", "3" };
int[] numbers = Array.ConvertAll(strings, toInt);  // [1, 2, 3]
```

### When to Use Each

- **Func<T>**: When you need a function that returns a value (queries, transformations, computations)
- **Action<T>**: When you need a function with side effects that returns nothing (logging, callbacks, handlers)
- **Predicate<T>**: When you need a boolean test on a single value (filtering, existence checks)
- **Comparison<T>**: When you need to compare two objects for sorting (custom sort orders)
- **Converter<TInput, TOutput>**: When you need to transform one type to another (type conversions, mappings)
- **Custom delegate**: When none of the built-in types fit your signature, or you want semantic naming

---

## Events

### What is an Event

- An event is a **wrapper around a delegate** with restricted access modifiers
- It provides a mechanism for a class to notify other classes when something of interest happens
- Events can only be raised from within the declaring class
- Outside code can only add or remove handlers using `+=` and `-=`
- Events are the C# implementation of the **Observer pattern**
- They are declared using the `event` keyword on a delegate type

```csharp
public class Button
{
    // Event declaration
    public event EventHandler Click;

    // Method that raises the event
    public void OnClick()
    {
        Click?.Invoke(this, EventArgs.Empty);
    }
}

// Subscribing to the event
var button = new Button();
button.Click += (sender, e) => Console.WriteLine("Button clicked!");
```

### How Events Prevent External Invocation and Clearing

- Without `event`, a public delegate field can be:
  - Invoked by external code: `button.Click()` (dangerous - may be null)
  - Cleared by external code: `button.Click = null` (removes all handlers)
  - Assigned by external code: `button.Click = MyHandler` (replaces all handlers)
- The `event` keyword restricts external access to only `+=` and `-=`
- External code cannot invoke the event or assign/replace it directly
- This enforces encapsulation and prevents accidental handler removal

```csharp
public class EventEmitter
{
    // Without event - DANGEROUS
    public Action OnSomethingPublic;  // Anyone can invoke or clear this

    // With event - SAFE
    public event Action OnSomething;  // Only += and -= allowed externally

    public void DoSomething()
    {
        OnSomething?.Invoke();  // Only the declaring class can raise
        OnSomethingPublic?.Invoke();  // Also works, but external code could also invoke
    }
}

var emitter = new EventEmitter();
// emitter.OnSomething();  // COMPILER ERROR - cannot invoke event from outside
// emitter.OnSomething = null;  // COMPILER ERROR - cannot assign event from outside
emitter.OnSomethingPublic();  // WORKS - but dangerous, could be null
```

### Event Accessors (add/remove)

- Events support custom `add` and `remove` accessors
- By default, events use automatic implementation (backing field with lock)
- Custom accessors allow you to add thread-safety, logging, or other behavior
- The `lock` keyword is commonly used for thread-safe event subscription

```csharp
public class ThreadSafeEmitter
{
    private readonly object _lock = new object();
    private EventHandler _myEvent;

    public event EventHandler MyEvent
    {
        add
        {
            lock (_lock)
            {
                _myEvent += value;
                Console.WriteLine($"Handler added. Total: {_myEvent?.GetInvocationList().Length ?? 0}");
            }
        }
        remove
        {
            lock (_lock)
            {
                _myEvent -= value;
                Console.WriteLine($"Handler removed. Total: {_myEvent?.GetInvocationList().Length ?? 0}");
            }
        }
    }

    public void Raise()
    {
        _myEvent?.Invoke(this, EventArgs.Empty);
    }
}
```

### Standard Event Pattern (EventHandler<T>)

- The standard pattern uses `EventHandler<TEventArgs>` as the delegate type
- `TEventArgs` should inherit from `EventArgs` (or be `EventArgs` itself)
- The event handler signature is always `void Method(object sender, TEventArgs e)`
- `sender` is the object that raised the event
- `e` contains the event data

```csharp
// Custom EventArgs
public class OrderEventArgs : EventArgs
{
    public int OrderId { get; }
    public decimal Total { get; }

    public OrderEventArgs(int orderId, decimal total)
    {
        OrderId = orderId;
        Total = total;
    }
}

// Class that raises the event
public class OrderService
{
    public event EventHandler<OrderEventArgs> OrderCreated;

    public void CreateOrder(int orderId, decimal total)
    {
        // Business logic here
        OrderCreated?.Invoke(this, new OrderEventArgs(orderId, total));
    }
}

// Subscriber
var service = new OrderService();
service.OrderCreated += (sender, e) =>
{
    Console.WriteLine($"Order {e.OrderId} created for {e.Total:C}");
};
```

### Why Events Use EventHandler<T> Instead of Custom Delegates

- `EventHandler<T>` provides a **standardized pattern** across the entire .NET ecosystem
- Using custom delegates makes your API inconsistent with other .NET code
- `EventHandler<T>` enforces the `(object sender, TEventArgs e)` signature
- It allows event data to be passed in a type-safe manner
- The `sender` parameter allows multiple subscribers to identify which object raised the event
- `EventArgs.Empty` provides a reusable instance for events with no data

### Event vs Public Delegate Field

- **Public delegate field**: external code can invoke, assign, or clear it - breaks encapsulation
- **Event**: external code can only subscribe (`+=`) or unsubscribe (`-=`) - maintains encapsulation
- Events are the correct choice for 99% of scenarios
- Public delegate fields are almost always a design mistake
- If you need to expose a callback, use a method parameter or property instead

```csharp
// BAD - Public delegate field
public class BadClass
{
    public Action<string> OnMessage;  // External code can invoke or clear
}

// GOOD - Event
public class GoodClass
{
    public event Action<string> OnMessage;  // External code can only subscribe
}

// BETTER - If you need external configuration, use a property
public class BetterClass
{
    public Action<string> MessageHandler { get; set; }  // Configurable but not invocable
}
```

---

## Anonymous Methods and Lambda Expressions

### Anonymous Methods (C# 2.0)

- Allow you to define a method inline without naming it
- Declared using the `delegate` keyword followed by parameters and body
- Can access variables from the enclosing scope (closures)
- Cannot be used with method group conversions in some cases
- More verbose than lambda expressions but functionally equivalent

```csharp
Func<int, int, int> add = delegate(int a, int b)
{
    return a + b;
};

// With no parameters
Action greet = delegate()
{
    Console.WriteLine("Hello!");
};

// Implicit parameter types (C# 3.0)
Func<string, bool> isLong = delegate(string s)
{
    return s.Length > 10;
};

// Accessing outer variable (closure)
int threshold = 5;
Func<int, bool> aboveThreshold = delegate(int x)
{
    return x > threshold;  // Captures 'threshold'
};
```

### Lambda Expressions (C# 3.0)

- A more concise syntax for inline functions
- Use `=>` (lambda operator) to separate parameters from body
- Can be expression lambdas (single expression) or statement lambdas (multiple statements)
- Implicitly typed when the compiler can infer types
- The most common way to write inline functions in modern C#

```csharp
// Expression lambda
Func<int, int, int> add = (a, b) => a + b;

// Statement lambda
Func<int, int, int> addVerbose = (a, b) =>
{
    Console.WriteLine($"Adding {a} and {b}");
    return a + b;
};

// No parameters
Action greet = () => Console.WriteLine("Hello!");

// Single parameter (parentheses optional)
Func<int, bool> isPositive = x => x > 0;

// Multiple parameters
Func<int, int, bool> areEqual = (a, b) => a == b;

// With explicit types
Func<string, int, bool> startsWith = (string s, int count) => s.Length >= count;
```

### Expression Lambdas vs Statement Lambdas

- **Expression lambda**: body is a single expression, returns the result implicitly
  - Syntax: `(params) => expression`
  - Cannot contain statements or multiple lines
  - Most concise and common form
- **Statement lambda**: body is a block with curly braces, requires explicit `return`
  - Syntax: `(params) { statements; return value; }`
  - Can contain multiple statements, local variables, loops, conditions
  - More flexible but more verbose

```csharp
// Expression lambda
Func<int, int> square = x => x * x;

// Statement lambda
Func<int, int> squareVerbose = x =>
{
    Console.WriteLine($"Squaring {x}");
    var result = x * x;
    Console.WriteLine($"Result: {result}");
    return result;
};

// Expression lambda with LINQ
var evens = numbers.Where(x => x % 2 == 0);

// Statement lambda rarely used with LINQ
var evensVerbose = numbers.Where(x =>
{
    var isEven = x % 2 == 0;
    Console.WriteLine($"Checking {x}: {isEven}");
    return isEven;
});
```

### Captured Variables (Closures)

- Lambdas and anonymous methods can **capture variables** from their enclosing scope
- The captured variable is shared - changes to the variable are visible to the lambda
- The variable is captured by **reference**, not by value
- This can lead to unexpected behavior in loops

```csharp
// Common issue - loop variable capture
var actions = new List<Action>();
for (int i = 0; i < 5; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
// All actions print 5, not 0-4!
// The variable 'i' is captured by reference, not value

// Fix: create a local copy
var actionsFixed = new List<Action>();
for (int i = 0; i < 5; i++)
{
    int copy = i;  // Local copy for this iteration
    actionsFixed.Add(() => Console.WriteLine(copy));
}
// Now prints 0, 1, 2, 3, 4

// Captured variable modification
int counter = 0;
Action increment = () => counter++;
increment();
Console.WriteLine(counter);  // 1
increment();
Console.WriteLine(counter);  // 2
```

### Why Lambdas are More Concise Than Anonymous Methods

- Lambdas don't require the `delegate` keyword
- Parameter types can be inferred by the compiler
- Single-parameter lambdas don't require parentheses
- Expression lambdas don't require braces or return statement
- Lambdas can be converted to expression trees for LINQ providers

```csharp
// Anonymous method - verbose
Func<int, bool> isPositiveAnon = delegate(int x)
{
    return x > 0;
};

// Lambda - concise
Func<int, bool> isPositiveLambda = x => x > 0;

// Lambda can also be converted to expression tree (for LINQ to SQL)
Expression<Func<int, bool>> isPositiveExpr = x => x > 0;
// Cannot do this with anonymous methods
```

---

## Real-World Patterns

### Observer Pattern with Events

```csharp
public class StockMarket
{
    public event EventHandler<StockChangedEventArgs> StockChanged;

    public void UpdateStock(string symbol, decimal price)
    {
        StockChanged?.Invoke(this, new StockChangedEventArgs(symbol, price));
    }
}

public class StockChangedEventArgs : EventArgs
{
    public string Symbol { get; }
    public decimal Price { get; }

    public StockChangedEventArgs(string symbol, decimal price)
    {
        Symbol = symbol;
        Price = price;
    }
}

// Multiple observers
var market = new StockMarket();

// Observer 1: Logger
market.StockChanged += (s, e) =>
    Console.WriteLine($"[LOG] {e.Symbol} changed to {e.Price}");

// Observer 2: Alert system
market.StockChanged += (s, e) =>
{
    if (e.Price > 100)
        Console.WriteLine($"[ALERT] {e.Symbol} is above 100!");
};

// Observer 3: Database writer
market.StockChanged += (s, e) =>
    Database.SaveStockUpdate(e.Symbol, e.Price);

market.UpdateStock("MSFT", 350.00m);  // All three observers notified
```

### Strategy Pattern with Delegates

```csharp
public class Sorter
{
    public void Sort<T>(List<T> items, Comparison<T> strategy)
    {
        items.Sort(strategy);
    }
}

var sorter = new Sorter();
var numbers = new List<int> { 5, 2, 8, 1, 9 };

// Strategy 1: Ascending
sorter.Sort(numbers, (a, b) => a.CompareTo(b));

// Strategy 2: Descending
sorter.Sort(numbers, (a, b) => b.CompareTo(a));

// Strategy 3: By absolute value
sorter.Sort(numbers, (a, b) => Math.Abs(a).CompareTo(Math.Abs(b)));
```

### Callback Pattern with Func<T>

```csharp
public class DataProcessor
{
    public TResult Process<TInput, TResult>(
        TInput input,
        Func<TInput, TInput> transformer,
        Func<TInput, TResult> processor)
    {
        var transformed = transformer(input);
        return processor(transformed);
    }
}

var processor = new DataProcessor();
int result = processor.Process(
    "hello",
    s => s.ToUpper(),           // Transformer: "hello" -> "HELLO"
    s => s.Length               // Processor: "HELLO" -> 5
);
// result = 5
```

### Event-Driven Architecture

```csharp
public class EventBus
{
    private readonly Dictionary<Type, List<Delegate>> _handlers = new();

    public void Subscribe<T>(Action<T> handler)
    {
        var type = typeof(T);
        if (!_handlers.ContainsKey(type))
            _handlers[type] = new List<Delegate>();
        _handlers[type].Add(handler);
    }

    public void Publish<T>(T eventData)
    {
        var type = typeof(T);
        if (_handlers.ContainsKey(type))
        {
            foreach (Action<T> handler in _handlers[type])
            {
                handler(eventData);
            }
        }
    }
}

// Usage
var bus = new EventBus();

bus.Subscribe<OrderCreatedEvent>(e => Console.WriteLine($"Order {e.OrderId} created"));
bus.Subscribe<OrderCreatedEvent>(e => EmailService.SendConfirmation(e));
bus.Subscribe<PaymentReceivedEvent>(e => InventoryService.ReserveStock(e));

bus.Publish(new OrderCreatedEvent { OrderId = 123 });
```

### Decoupling Services with Events

```csharp
public class UserService
{
    public event EventHandler<UserRegisteredEventArgs> UserRegistered;

    public void Register(string email, string password)
    {
        // Create user
        var user = new User { Email = email };

        // Raise event - services are decoupled
        UserRegistered?.Invoke(this, new UserRegisteredEventArgs(user));
    }
}

// These services don't know about each other
var userService = new UserService();

// Email service subscribes
userService.UserRegistered += (s, e) =>
    EmailService.SendWelcomeEmail(e.User.Email);

// Analytics service subscribes
userService.UserRegistered += (s, e) =>
    AnalyticsService.TrackRegistration(e.User.Id);

// CRM service subscribes
userService.UserRegistered += (s, e) =>
    CrmService.CreateContact(e.User);

// None of these services are coupled to each other
// They only depend on the UserService event
userService.Register("user@example.com", "password123");
```

---

## Common Mistakes

### Not Unsubscribing from Events (Memory Leaks)

- If a subscriber doesn't unsubscribe, the publisher holds a reference to the subscriber
- This prevents the garbage collector from collecting the subscriber
- Long-lived publishers with short-lived subscribers cause the most leaks
- Always unsubscribe in `Dispose` or when the subscriber is destroyed
- Use `WeakEventManager` (WPF/Maui) for automatic cleanup in certain scenarios

```csharp
// LEAK: Subscriber never unsubscribed
public class LeakyForm : Form
{
    public LeakyForm()
    {
        GlobalEventBus.UserLoggedIn += OnUserLoggedIn;  // Never unsubscribed
    }

    private void OnUserLoggedIn(object sender, EventArgs e)
    {
        // This form stays alive as long as GlobalEventBus exists
    }
}

// FIX: Unsubscribe properly
public class GoodForm : Form
{
    public GoodForm()
    {
        GlobalEventBus.UserLoggedIn += OnUserLoggedIn;
    }

    protected override void OnFormClosed(FormClosedEventArgs e)
    {
        GlobalEventBus.UserLoggedIn -= OnUserLoggedIn;
        base.OnFormClosed(e);
    }
}
```

### Raising Events Without Null Check

- If no handlers are subscribed, the event delegate is null
- Invoking a null delegate throws `NullReferenceException`
- Always use the null-conditional operator `?.Invoke()` or check for null first

```csharp
// WRONG - throws NullReferenceException if no subscribers
public void OnClick()
{
    Click();  // CRASH if Click is null
}

// CORRECT - null-conditional operator
public void OnClick()
{
    Click?.Invoke(this, EventArgs.Empty);
}

// CORRECT - explicit null check (needed for thread safety sometimes)
public void OnClick()
{
    var handler = Click;
    if (handler != null)
    {
        handler(this, EventArgs.Empty);
    }
}
```

### Using Delegates for Simple Callbacks (Use Func Instead)

- Custom delegate types add unnecessary complexity for simple callbacks
- `Func<T>` and `Action<T>` are sufficient for most scenarios
- Only define custom delegates when you need semantic naming or the signature is complex
- Custom delegates make the API harder to understand for developers familiar with standard patterns

```csharp
// UNNECESSARY - custom delegate for simple callback
public delegate int CalculateDelegate(int a, int b);
public int Calculate(CalculateDelegate calc) => calc(2, 3);

// BETTER - use Func
public int Calculate(Func<int, int, int> calc) => calc(2, 3);

// GOOD - custom delegate with semantic meaning
public delegate void StockPriceChangedHandler(string symbol, decimal oldPrice, decimal newPrice);
// This adds clarity when the signature is complex and the delegate is used extensively
```

### Not Using WeakEventManager for Long-Lived Subscribers

- In WPF/Maui, long-lived objects subscribing to events on short-lived objects can cause memory issues
- `WeakEventManager` holds weak references to subscribers, allowing garbage collection
- Important for scenarios where the publisher outlives the subscriber

```csharp
// Problem: Long-lived service holds strong reference to short-lived view
public class DataService
{
    public event EventHandler<DataEventArgs> DataLoaded;
}

// WeakEventManager solution (WPF)
WeakEventManager<DataService, EventArgs>.AddHandler(
    dataService,
    nameof(DataService.DataLoaded),
    OnDataLoaded);

private void OnDataLoaded(object sender, DataEventArgs e)
{
    // Called normally, but subscriber can be garbage collected
}
```

### Capturing Variables in Closures Causing Unexpected Behavior

- Loop variable capture is the most common closure bug
- All iterations share the same variable reference
- Fix by creating a local copy inside the loop
- Be cautious with `this` capture in lambdas inside constructors

```csharp
// BUG: All buttons trigger same handler
var buttons = new[] { button1, button2, button3 };
for (int i = 0; i < buttons.Length; i++)
{
    buttons[i].Click += (s, e) => Console.WriteLine($"Button {i} clicked");
}
// All print "Button 3 clicked"

// FIX: Local copy
for (int i = 0; i < buttons.Length; i++)
{
    int index = i;  // Local copy
    buttons[i].Click += (s, e) => Console.WriteLine($"Button {index} clicked");
}

// BUG: Capturing 'this' in constructor
public class MyClass
{
    public MyClass()
    {
        // 'this' is not fully constructed yet
        someEvent += (s, e) => this.DoSomething();
    }
}
```

### Forgetting That Multicast Delegates Return Only Last Value

- Developers often expect all return values to be collected
- Only the return value of the last invoked delegate is preserved
- Use `GetInvocationList()` and iterate if you need all return values
- Design events to return void to avoid this confusion

```csharp
Func<int, int> addTen = x => x + 10;
Func<int, int> multiplyByTwo = x => x * 2;

Func<int, int> combined = addTen + multiplyByTwo;
int result = combined(5);  // Returns 10 (from multiplyByTwo), NOT 15

// If you need all results:
var results = combined.GetInvocationList()
    .Cast<Func<int, int>>()
    .Select(d => d(5))
    .ToList();  // [15, 10]
```

---

## Interview Questions

### Q1: What is the difference between a delegate and an event?

- A **delegate** is a type that holds a reference to a method - it can be invoked, assigned, and combined
- An **event** is a wrapper around a delegate that restricts external access to only `+=` and `-=`
- External code can invoke a public delegate but cannot invoke an event
- Events enforce encapsulation - only the declaring class can raise the event
- Delegates are used for callbacks, events are used for notifications

### Q2: Explain the difference between Func, Action, and Predicate

- **Func<T, TResult>**: takes parameters, returns a value - used for queries and computations
- **Action<T>**: takes parameters, returns void - used for side effects and handlers
- **Predicate<T>**: takes one parameter, returns bool - used for filtering and existence checks
- `Predicate<T>` is functionally identical to `Func<T, bool>` but more semantic
- `Func` and `Action` support up to 16 parameters, `Predicate` only supports one

### Q3: What happens with multicast delegate return values?

- Only the return value of the **last** invoked delegate is preserved
- All previous return values are discarded
- This is a design decision - there's no standard way to aggregate multiple return values
- If you need all return values, use `GetInvocationList()` and iterate manually
- This is why events and callbacks typically return void

### Q4: Describe the standard event pattern in C#

- Uses `EventHandler<TEventArgs>` as the delegate type
- `TEventArgs` inherits from `EventArgs` (or uses `EventArgs` directly for no data)
- Handler signature is always `void Method(object sender, TEventArgs e)`
- `sender` identifies the object that raised the event
- `e` contains the event-specific data
- Use `EventArgs.Empty` when no data is needed

### Q5: What are closures and why do they matter?

- A **closure** is when a lambda or anonymous method captures variables from its enclosing scope
- The captured variable is shared by reference, not by value
- This allows the lambda to access and modify variables even after the enclosing method returns
- Closures can cause bugs in loops where the loop variable is captured
- Fix by creating a local copy of the variable inside the loop body

### Q6: What is the difference between expression lambdas and statement lambdas?

- **Expression lambda**: body is a single expression, result is returned implicitly
  - Syntax: `(x) => x * x`
- **Statement lambda**: body is a block with curly braces, requires explicit return
  - Syntax: `(x) { var r = x * x; return r; }`
- Expression lambdas can be converted to expression trees (used by LINQ providers)
- Statement lambdas are needed when multiple statements or local variables are required
- Expression lambdas are more common and concise

### Q7: How do delegate covariance and contravariance work?

- **Covariance** (out): allows a method returning a more derived type to be used where a less derived return type is expected
- **Contravariance** (in): allows a method accepting a less derived parameter type to be used where a more derived parameter type is expected
- Covariance preserves assignment compatibility for return types
- Contravariance preserves assignment compatibility for parameter types
- Generic delegates like `Func` and `Action` already support these via `in` and `out` keywords

### Q8: How do you prevent memory leaks from events?

- Always unsubscribe from events when the subscriber is destroyed
- Use `Dispose` pattern to clean up event subscriptions
- Use `WeakEventManager` for scenarios where the publisher outlives the subscriber
- Be cautious with anonymous lambdas - they cannot be easily unsubscribed
- Named methods or storing the delegate reference allows proper unsubscription

### Q9: Why should you always check for null before invoking an event?

- If no handlers are subscribed, the event delegate field is null
- Invoking a null delegate throws `NullReferenceException`
- Use `EventName?.Invoke(sender, args)` for null-safe invocation
- Alternatively, capture the delegate in a local variable and check for null (thread-safe pattern)
- The null-conditional operator is the modern, concise approach

### Q10: What is the difference between named methods and anonymous methods for delegates?

- **Named methods**: have a name, can be referenced multiple times, easier to debug and test
- **Anonymous methods**: defined inline, cannot be referenced by name, good for one-time use
- Named methods improve readability when the logic is complex
- Anonymous methods and lambdas are better for simple, short callbacks
- Named methods can be used with `RemoveAll` and other methods that need to unsubscribe

### Q11: Can you explain how the invocation list works in multicast delegates?

- The `_invocationList` array stores multiple delegate instances
- When delegates are combined with `+` or `+=`, they are added to this array
- When invoked, the runtime iterates through the array and calls each delegate
- If any delegate throws an exception, subsequent delegates are not invoked
- `GetInvocationList()` returns a copy of the array for manual iteration
- Removing a delegate with `-` or `-=` removes the first matching instance from the array

### Q12: When would you use a custom delegate instead of Func/Action?

- When the delegate represents a specific concept (e.g., `StockPriceChangedHandler`)
- When the delegate is used extensively throughout the codebase
- When you want to provide meaningful documentation through the type name
- When the signature is complex and `Func` with many type parameters becomes unreadable
- When you need to enforce a specific contract that `Func`/`Action` cannot express

### Q13: How do you handle exceptions in multicast delegates?

- By default, if one delegate throws, subsequent delegates are skipped
- Use `GetInvocationList()` and iterate manually with try-catch for each delegate
- This ensures all handlers are invoked even if some fail
- Consider logging failures without rethrowing to maintain resilience
- Some frameworks provide built-in exception handling for events

### Q14: What is the purpose of the `sender` parameter in event handlers?

- `sender` is the object that raised the event
- It allows a single handler to serve multiple event sources
- The handler can cast `sender` to the appropriate type to access source-specific data
- It enables polymorphic event handling where the handler behavior depends on the source
- Typically cast using `as` or `(Type)sender` with null checking

### Q15: How do event accessors (add/remove) differ from regular events?

- Regular events use automatic backing fields with default add/remove behavior
- Custom accessors allow you to implement thread-safe subscription using locks
- Custom accessors can add logging, validation, or other cross-cutting concerns
- The `add` accessor is called when using `+=`, `remove` when using `-=`
- You can override only `add` or only `remove` if needed
- Custom accessors are common in WPF and other UI frameworks

### Q16: What is the difference between delegate equality and reference equality?

- Two delegates are equal if they point to the same method on the same target
- Delegate equality compares both `_target` and `_methodPtr`
- Two different delegate instances pointing to the same method are considered equal
- This matters when removing delegates - the correct instance must be used
- Lambda expressions create new delegate instances each time, complicating equality
- For reliable removal, store the delegate reference in a variable

### Q17: Can events be invoked from derived classes?

- No, events can only be raised from the declaring class
- Derived classes cannot invoke events defined in the base class
- If you need derived classes to raise events, define a protected `OnEventName` method
- The method checks for null and invokes the event
- This is the standard pattern in .NET frameworks

```csharp
public class BaseClass
{
    public event EventHandler MyEvent;

    protected virtual void OnMyEvent()
    {
        MyEvent?.Invoke(this, EventArgs.Empty);
    }
}

public class DerivedClass : BaseClass
{
    public void DoSomething()
    {
        OnMyEvent();  // Calls base class method which raises the event
    }
}
```
