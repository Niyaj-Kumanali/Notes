# Exception Handling in C# - Interview Mastery

## Exception Handling Basics

### What is an Exception

- An exception is a runtime error that disrupts the normal flow of program execution
- Exceptions are thrown when something goes wrong during program execution
- They represent exceptional conditions, not normal business logic outcomes
- Every exception inherits from `System.Exception` ultimately
- Exceptions carry information: message, stack trace, inner exception, and custom data
- Exceptions should signal something unexpected, not expected control flow
- When an exception is thrown, the runtime looks for the nearest matching catch block
- If no catch block is found, the program terminates with an unhandled exception

### try, catch, finally Blocks - How They Work

- The `try` block wraps code that might throw an exception
- The `catch` block handles the exception if one occurs in the try block
- The `finally` block always executes regardless of whether an exception occurred
- Execution flow: try → (exception) → catch → finally → continue after try-catch-finally
- Execution flow: try → (no exception) → finally → continue after try-catch-finally
- You can have a try block without catch, but only if finally is present
- You can have a try block without finally, but only if catch is present
- The catch block can optionally capture the exception variable for inspection
- You can omit the exception variable: `catch { }` catches all exceptions
- The finally block runs even if a `return`, `break`, or `continue` is executed in try or catch

```csharp
try
{
    int result = 10 / int.Parse("0");
}
catch (DivideByZeroException ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}
finally
{
    Console.WriteLine("This always runs");
}
```

### Why finally Always Executes

- The finally block is guaranteed to execute by the CLR even when:
  - A `return` statement is hit in the try block
  - An exception is thrown and caught
  - An exception is thrown and not caught (finally runs before the exception propagates)
  - A `break` or `continue` statement transfers control out of the try block
- The only cases where finally does NOT run:
  - `Environment.FailFast()` is called
  - The process is killed (`Environment.Exit()`, `Process.Kill()`)
  - A `StackOverflowException` occurs (no catch or finally for this)
  - An out-of-memory exception in certain edge cases
  - The CLR itself crashes
- finally is typically used for cleanup: closing files, releasing resources, disposing objects
- The guarantee of finally execution is what makes it reliable for resource cleanup

### Multiple Catch Blocks - Order Matters

- Multiple catch blocks allow handling different exception types differently
- They are evaluated in order from top to bottom
- The FIRST matching catch block is executed, then execution jumps to finally
- Most specific exceptions must come first; more general exceptions must come last
- If a general catch comes first, specific catches below it will never be reached (compile error)
- You can only enter ONE catch block per exception; catch blocks don't fall through

```csharp
try
{
    // code that might throw various exceptions
}
catch (ArgumentNullException ex)      // Most specific
{
    // Handle null argument
}
catch (ArgumentException ex)           // Less specific (base of ArgumentNullException)
{
    // Handle bad argument
}
catch (Exception ex)                   // Most general - always last
{
    // Handle any other exception
}
```

- A catch block without a type (`catch { }`) catches all exceptions and should be last
- Mixing typed and untyped catch blocks: untyped must always be the final catch block

### When to Use catch vs When to Let Exception Propagate

- Use catch when you can meaningfully handle or recover from the exception
- Use catch when you need to log the exception and then rethrow it
- Use catch when you need to translate one exception type into another
- Let exceptions propagate when the current method cannot fix the problem
- Let exceptions propagate when the caller needs to know about the failure
- Let exceptions propagate when catching would hide the real error
- Never catch exceptions just to ignore them (empty catch blocks are almost always wrong)
- Consider the level of abstraction: lower-level code should rarely catch; higher-level code should
- Infrastructure/utility code can catch and wrap; business logic should rarely catch

---

## Exception Hierarchy

### System.Exception Base Class

- `System.Exception` is the root of all exception types in .NET
- Key properties:
  - `Message` - human-readable description of the error
  - `StackTrace` - string showing where the exception originated
  - `InnerException` - reference to the exception that caused this one
  - `Source` - name of the assembly that generated the exception
  - `HResult` - HRESULT value for the exception
  - `HelpLink` - link to help file for the exception
  - `TargetSite` - method that threw the exception
- `Exception.Data` property is an `IDictionary` for attaching custom key-value data
- All exceptions should ultimately inherit from `System.Exception`
- You can throw any class that inherits from `System.Exception`

### SystemException vs ApplicationException

- `SystemException` is the base class for exceptions thrown by the CLR or .NET runtime
- `ApplicationException` was originally intended as the base class for application-defined exceptions
- In practice, this distinction is no longer useful or recommended
- Modern guidance: just inherit from `System.Exception` directly
- .NET runtime throws both `SystemException` subtypes and other exceptions
- `ApplicationException` has no special behavior or properties
- Some Microsoft documentation still references this distinction, but it's outdated
- Most teams and frameworks inherit custom exceptions directly from `Exception`

### Common Exceptions and When Each Occurs

#### NullReferenceException
- Thrown when you try to access a member on a null object reference
- Example: `string s = null; s.Length;` throws NullReferenceException
- This is the most common exception in C# applications
- Often indicates a bug: the code assumed an object was non-null when it wasn't
- Prevent with null checks, null-conditional operator (`?.`), or null-coalescing (`??`)

```csharp
string name = null;
Console.WriteLine(name.Length); // NullReferenceException
// Better:
Console.WriteLine(name?.Length ?? 0);
```

#### ArgumentException (and ArgumentNullException, ArgumentOutOfRangeException)
- `ArgumentException` - base class for invalid argument exceptions
- `ArgumentNullException` - thrown when a required argument is null
- `ArgumentOutOfRangeException` - thrown when an argument is outside valid range
- These should be thrown by methods to indicate bad input from callers
- Example: `ArgumentNullException.ThrowIfNull(param)` in .NET 6+
- These are not bugs in your code but bugs in the caller's code

```csharp
public void SetAge(int age)
{
    if (age < 0 || age > 150)
        throw new ArgumentOutOfRangeException(nameof(age), age, "Age must be between 0 and 150");
}
```

#### InvalidOperationException
- Thrown when a method call is invalid given the object's current state
- This is the "catch-all" for state-related errors
- Example: calling `Read()` on a closed `StreamReader`
- Example: calling `MoveNext()` on an enumerator after `Reset()` fails
- Indicates the object is in a state where the requested operation cannot be performed

```csharp
var list = new List<int> { 1, 2, 3 };
var enumerator = list.GetEnumerator();
enumerator.MoveNext();
enumerator.Reset(); // May throw depending on implementation
```

#### IndexOutOfRangeException
- Thrown when you access an array or collection with an invalid index
- Example: `int[] arr = new int[3]; arr[5] = 10;`
- Example: accessing negative indices on arrays
- Does NOT apply to `List<T>` - lists throw `ArgumentOutOfRangeException` instead
- Arrays in C# are zero-based, so valid indices are 0 to Length-1
- Check bounds before accessing array elements, or use `TryGetValue` patterns

```csharp
int[] numbers = new int[5];
numbers[10] = 42; // IndexOutOfRangeException
```

#### InvalidCastException
- Thrown when an invalid type conversion is attempted
- Example: `(int)(object)"hello"` - can't unbox a string to int
- Example: direct cast from a base type to an unrelated derived type
- Use `as` operator or `is` keyword for safe casting to avoid this
- `Convert.ToInt32("abc")` throws `FormatException`, not `InvalidCastException`

```csharp
object obj = "Hello";
int num = (int)obj; // InvalidCastException
// Safe approach:
if (obj is int safeNum) { /* use safeNum */ }
```

#### OverflowException
- Thrown when an arithmetic operation produces a result too large for the target type
- Occurs with `checked` context: `checked { int x = int.MaxValue + 1; }`
- Unchecked context (default) silently wraps around without throwing
- Also thrown by `Convert.ToInt32(double)` when the double is outside int range
- Common in financial applications where overflow must be detected

```csharp
checked
{
    int x = int.MaxValue;
    int y = x + 1; // OverflowException
}
```

#### FormatException
- Thrown when the format of an argument does not meet the parameter specifications
- Example: `int.Parse("abc")` throws FormatException
- Example: `Guid.Parse("not-a-guid")` throws FormatException
- Indicates the input string doesn't match the expected format

#### KeyNotFoundException
- Thrown when accessing a dictionary with a key that doesn't exist
- Example: `dict["nonexistent"]` throws KeyNotFoundException
- Use `TryGetValue()` or `ContainsKey()` to avoid this
- `dict.GetValueOrDefault(key)` returns default without throwing

```csharp
var dict = new Dictionary<string, int> { { "a", 1 } };
int val = dict["b"]; // KeyNotFoundException
// Safe:
dict.TryGetValue("b", out int val2); // val2 is 0, no exception
```

#### NotSupportedException
- Thrown when an operation is not supported
- Example: calling `Add()` on a read-only collection
- Example: calling `GetEnumerator()` on a non-enumerable type
- Often used in abstract base classes to indicate methods that derived classes must override

---

## throw vs throw ex

### throw Preserves Original Stack Trace

- Using `throw` alone in a catch block rethrows the same exception object
- The original stack trace from where the exception was first thrown is preserved
- This is almost always what you want when rethrowing exceptions
- The stack trace shows the complete call stack from the original throw point
- This makes debugging much easier because you can see where the problem actually occurred

```csharp
try
{
    ProcessData();
}
catch (Exception ex)
{
    Log.Error(ex); // Full stack trace available here
    throw;         // Rethrows with original stack trace preserved
}
```

### throw ex Resets Stack Trace to Current Line

- Using `throw ex` in a catch block resets the stack trace to the current line
- The original stack trace information is LOST forever
- This makes it extremely difficult to find where the exception originally occurred
- The debugger will show the exception originating from the `throw ex` line
- This is almost always a bug when used in catch blocks
- Never use `throw ex` unless you specifically want to lose the stack trace (rare)

```csharp
try
{
    ProcessData();
}
catch (Exception ex)
{
    Log.Error(ex);
    throw ex;  // BAD: Stack trace now shows THIS line as the origin
}
```

### Why Always Use 'throw' in Catch Blocks

- `throw` preserves the full debugging information about where the error occurred
- Stack traces are invaluable for diagnosing production issues
- `throw ex` makes it look like the exception came from your catch block
- Code reviewers and senior developers immediately recognize `throw ex` as a red flag
- Many static analysis tools flag `throw ex` as a code smell
- The only legitimate use of `throw ex` is extremely rare: intentionally masking the origin

### Logging Before Rethrowing

- Always log the exception before rethrowing if you need to record the error
- Log the full exception object, not just the message
- Include contextual information: what were you doing, what were the inputs
- Use structured logging where possible for better searchability

```csharp
try
{
    var result = await _database.QueryAsync(sql);
    return result;
}
catch (SqlException ex)
{
    _logger.LogError(ex, "Database query failed for SQL: {Sql}, Parameters: {Params}",
        sql, parameters);
    throw; // Preserve stack trace while logging
}
```

---

## Custom Exceptions

### How to Create Custom Exceptions

- Create a class that inherits from `Exception` or a more specific exception type
- Implement three constructors: default, message, and message+inner exception
- Optionally implement the serialization constructor for cross-appdomain support
- Mark the class as `[Serializable]` if it needs to cross appdomain boundaries
- Use the `Exception` base class properties and constructor chaining

```csharp
[Serializable]
public class OrderProcessingException : Exception
{
    public OrderProcessingException() { }

    public OrderProcessingException(string message)
        : base(message) { }

    public OrderProcessingException(string message, Exception innerException)
        : base(message, innerException) { }

    // Serialization constructor (for .NET Framework / cross-AppDomain)
    protected OrderProcessingException(System.Runtime.Serialization.SerializationInfo info,
        System.Runtime.Serialization.StreamingContext context)
        : base(info, context) { }
}
```

### Why Custom Exceptions Should Be Serializable

- Serialization allows exceptions to be transmitted across process boundaries
- Required for remoting, WCF, and certain logging frameworks
- The `[Serializable]` attribute marks the exception as safe to serialize
- The serialization constructor allows the exception to be reconstructed from serialized data
- In .NET Core/5+, serialization concerns are less critical but still good practice
- Some logging and monitoring tools rely on exception serialization

### Best Naming Convention

- Always end custom exception names with "Exception"
- Examples: `OrderProcessingException`, `PaymentFailedException`, `InvalidConfigException`
- The suffix makes it immediately clear that the type is an exception
- Follows the .NET Framework convention (`ArgumentException`, `InvalidOperationException`)
- Avoid names like `OrderError` or `PaymentProblem` - they break convention

### When to Create Custom Exceptions vs Using Existing Ones

- Create custom exceptions when you need to convey specific domain-specific error information
- Create custom exceptions when callers need to handle your errors differently from others
- Create custom exceptions when you need to attach custom properties/data to the exception
- Use existing exceptions when they already describe the problem accurately
- Don't create custom exceptions just for the sake of having your own types
- `InvalidOperationException` is often sufficient for state-related errors
- `ArgumentException` and its subtypes cover most parameter validation errors
- Custom exceptions add value only when they provide meaningful semantic distinction

### Exception Filters with 'when' Keyword

- Exception filters allow you to add conditions to catch blocks without catching and rethrowing
- The `when` keyword is placed after the exception type in the catch block
- The filter expression is evaluated before the catch block body executes
- If the filter returns false, the catch block is skipped and the next catch is tried
- Filters do not unwind the stack, so stack traces are cleaner than catch+rethrow

```csharp
try
{
    await MakeHttpRequest(url);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    // Only catches 404 errors specifically
    Console.WriteLine("Resource not found");
}
catch (HttpRequestException ex)
{
    // Catches all other HttpRequestExceptions
    Console.WriteLine($"HTTP error: {ex.StatusCode}");
}
```

---

## IDisposable Pattern

### What is IDisposable

- `IDisposable` is an interface with a single method: `void Dispose()`
- It provides a deterministic way to release resources before garbage collection
- It's the C# equivalent of C++ RAII (Resource Acquisition Is Initialization)
- The `using` statement/declaration is syntactic sugar that calls `Dispose()` automatically
- IDisposable objects should always be used with `using` or explicit `Dispose()` calls
- Not disposing IDisposable objects can cause resource leaks, file handles remaining open, etc.

```csharp
public interface IDisposable
{
    void Dispose();
}
```

### Why Dispose() is Needed (Unmanaged Resources)

- Managed resources (other .NET objects) are cleaned up by the garbage collector
- Unmanaged resources (file handles, database connections, native memory) are NOT cleaned up by GC
- The GC doesn't know about unmanaged resources and cannot free them deterministically
- `Dispose()` provides a deterministic way to release unmanaged resources immediately
- Without Dispose, unmanaged resources may not be released until the process exits
- Common unmanaged resources: `FileStream`, `SqlConnection`, `Socket`, `Graphics`, `Process`

### Using Statement and Using Declaration

#### Using Statement (Classic)
- `using` statement ensures `Dispose()` is called when the block exits
- Works with any type implementing `IDisposable`
- Dispose is called in the `finally` block of the generated code
- Can have multiple resources separated by semicolons
- Resources are disposed in reverse order of declaration

```csharp
using (var stream = new FileStream("data.txt", FileMode.Open))
using (var reader = new StreamReader(stream))
{
    string content = reader.ReadToEnd();
}
// Both reader and stream are disposed here
```

#### Using Declaration (C# 8.0+)
- Simpler syntax using `using` as a variable declaration modifier
- Resource is disposed at the end of the enclosing scope (not the statement)
- Cleaner code when you only need one resource
- Still calls `Dispose()` at scope exit, which is usually the end of the method

```csharp
using var stream = new FileStream("data.txt", FileMode.Open);
string content = new StreamReader(stream).ReadToEnd();
// stream is disposed when method ends
```

### Dispose Pattern with Dispose(bool disposing)

- The full dispose pattern separates managed and unmanaged cleanup
- `Dispose(bool disposing)` is called by both `Dispose()` and the finalizer
- When `disposing` is true: called from Dispose() - safe to access other managed objects
- When `disposing` is false: called from finalizer - DO NOT access other managed objects (they may be finalized)
- Prevents double disposal with `_disposed` flag
- Used when your class holds both managed and unmanaged resources

```csharp
public class MyResource : IDisposable
{
    private bool _disposed = false;
    private IntPtr _unmanagedHandle; // unmanaged resource
    private StreamWriter _logWriter; // managed resource

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
                _logWriter?.Dispose();
            }
            // Free unmanaged resources
            if (_unmanagedHandle != IntPtr.Zero)
            {
                CloseHandle(_unmanagedHandle);
                _unmanagedHandle = IntPtr.Zero;
            }
            _disposed = true;
        }
    }

    ~MyResource()
    {
        Dispose(false);
    }
}
```

### Finalizer (~ClassName) vs Dispose()

- The finalizer (destructor) is called by the garbage collector before the object is reclaimed
- Finalizers are non-deterministic - you cannot predict when they will run
- Finalizers run on the GC thread, which can cause performance issues
- Finalizers should only be used for cleaning up unmanaged resources as a safety net
- Dispose() is deterministic - called immediately when you invoke it
- Dispose() runs on the calling thread, giving you control over when cleanup happens
- The general rule: use Dispose() for managed resources, finalizer for unmanaged resources
- If a class has no unmanaged resources, don't implement a finalizer

### When to Use IAsyncDisposable

- `IAsyncDisposable` is for types that perform async disposal operations
- Use when disposal involves async I/O (network calls, database operations, file writes)
- Has a single method: `ValueTask DisposeAsync()`
- Used with `await using` syntax instead of `using`
- Common in EF Core: `DbContext` implements `IAsyncDisposable`
- Allows freeing resources without blocking threads during disposal
- Available since .NET Core 3.0 / C# 8.0

```csharp
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync();
// connection is disposed asynchronously here
```

### GC.SuppressFinalize Usage

- `GC.SuppressFinalize(this)` tells the GC not to call the finalizer
- Called in `Dispose()` to indicate that cleanup has already been done
- Prevents the performance overhead of finalization if Dispose was called
- Without it, the object goes to the finalization queue even after Dispose
- Finalization queue processing is expensive; suppressing it when unnecessary is a significant optimization
- Always call `GC.SuppressFinalize(this)` in the `Dispose()` method if you have a finalizer

### Preventing Double Disposal

- Use a `_disposed` (or `_isDisposed`) boolean field to track disposal state
- Check `_disposed` at the start of `Dispose()` and return immediately if already disposed
- Check `_disposed` in methods that use the disposed resource and throw `ObjectDisposedException`
- Thread-safety considerations: use `_disposed` with proper synchronization or `Interlocked.CompareExchange`
- The standard dispose pattern already includes this protection

```csharp
public void Write(byte[] data)
{
    if (_disposed)
        throw new ObjectDisposedException(nameof(MyWriter));
    // proceed with write
}
```

---

## Exception Filters

### catch when (condition)

- Exception filters allow conditional catch blocks without catching and rethrowing
- The `when` clause evaluates a boolean expression before entering the catch block
- If the condition is false, the runtime skips this catch and tries the next one
- Filters can reference the caught exception variable
- Multiple filters can be chained with `||` and `&&`

```csharp
catch (IOException ex) when (ex.HResult == -2146232800)
{
    // Only handles specific IO error codes
}
```

### Why Filters Are Better Than Catch + Rethrow

- Filters do NOT unwind the stack before evaluating the condition
- With catch + rethrow, the stack is partially unwound, losing some stack trace information
- Filters evaluate the condition with the FULL original stack trace intact
- The debugger shows the original throw point, not the catch line
- Performance is slightly better because the filter short-circuits before entering the catch block
- Code is cleaner and more expressive with filters

```csharp
// BAD: catch + rethrow (unwinds stack)
catch (Exception ex)
{
    if (ex is not HttpRequestException httpEx || httpEx.StatusCode != 404)
        throw; // Stack trace is partially lost
    // handle 404
}

// GOOD: filter (preserves full stack trace)
catch (Exception ex) when (ex is HttpRequestException httpEx && httpEx.StatusCode == 404)
{
    // handle 404 with full stack trace
}
```

### How Stack Trace is Preserved with Filters

- When a filter returns false, the runtime moves to the next catch block without modifying the stack
- The stack trace remains exactly as it was when the exception was thrown
- When a filter returns true, the catch block executes with the original stack trace
- Contrast with catch+rethrow where the stack trace gets truncated at the rethrow point
- This is the primary technical advantage of exception filters

### When to Use Exception Filters in Production

- Use for logging-only catch blocks where you want to log and continue (rare pattern)
- Use when you need to catch only specific instances of a broad exception type
- Use when debugging requires full stack traces (always in production)
- Use for handling specific error codes or HTTP status codes from exceptions
- Use when you have multiple catch blocks for the same exception type with different conditions
- Avoid overly complex filter expressions that are hard to read
- Keep filters simple: equality checks, type checks, property comparisons

---

## Common Mistakes

### Catching Exception Without Filtering (Too Broad)

- `catch (Exception ex)` catches everything, including exceptions you don't expect
- This hides bugs that should propagate to higher-level handlers
- At minimum, use an exception filter to narrow down what you catch
- Broader catches should be at higher levels of the call stack
- Catching `Exception` at a low level masks issues that callers need to know about
- Always ask: "Can I actually handle ALL possible exceptions here?" If no, don't catch `Exception`

### Empty Catch Blocks (Swallowing Exceptions)

- An empty catch block silently ignores the error
- This hides bugs, data corruption, and resource leaks
- The caller never knows something went wrong
- At minimum, log the exception before deciding not to rethrow
- Empty catch blocks are one of the worst code smells in C#
- If you truly need to ignore an exception, document WHY with a comment

```csharp
// TERRIBLE - hides all errors silently
try
{
    SaveToDatabase(data);
}
catch { } // NEVER do this

// Better - at minimum log it
try
{
    SaveToDatabase(data);
}
catch (Exception ex)
{
    _logger.LogWarning(ex, "Failed to save to database, will retry later");
}
```

### Using Exceptions for Flow Control

- Exceptions are expensive: they capture stack traces, allocate memory, and unwind the stack
- Don't use exceptions to handle expected conditions like missing keys or null values
- Use `TryGetValue` instead of catching `KeyNotFoundException`
- Use null checks instead of catching `NullReferenceException`
- Use `int.TryParse` instead of catching `FormatException`
- Exceptions should be truly exceptional, not routine business logic paths

```csharp
// BAD: Exception for flow control
try
{
    var value = dict[key];
    Process(value);
}
catch (KeyNotFoundException)
{
    // key not found - this is EXPECTED
}

// GOOD: Check first
if (dict.TryGetValue(key, out var value))
{
    Process(value);
}
```

### Throwing Exceptions in Constructors

- Constructors that throw exceptions leave the object in a partially constructed state
- The object reference is never assigned to the caller's variable
- The finalizer will NOT run for partially constructed objects
- Validate inputs before performing any side effects in constructors
- If validation fails, throw before allocating resources or modifying state
- Consider using factory methods instead of constructors for complex initialization

### Not Disposing Resources in Finally

- If you don't use `using`, you must dispose in a `finally` block
- Disposing only in the `catch` block misses cases where no exception occurred but you still need to clean up
- Disposing only after the try-catch misses cases where an exception prevented reaching the dispose call
- `using` is always preferred over manual try-finally for disposal

```csharp
// BAD: only disposing in catch
try
{
    var reader = new StreamReader(path);
    // ...
}
catch (Exception ex)
{
    reader?.Dispose(); // What if no exception? Reader leaks!
}

// GOOD: using statement
using var reader = new StreamReader(path);
```

### Exception in Finally Block Hiding Original Exception

- If an exception is thrown in a finally block, it REPLACES the original exception
- The original exception is lost unless you catch it and handle it carefully
- Keep finally blocks simple: just resource cleanup
- Don't put complex logic in finally blocks
- If finally code can fail, wrap it in its own try-catch

```csharp
try
{
    throw new InvalidOperationException("original");
}
finally
{
    throw new ArgumentException("finally exception"); // Hides InvalidOperationException!
}
```

### Async Exceptions Not Being Observed

- In async methods, exceptions are captured in the returned `Task`
- If you don't await the task, the exception is never observed (becomes unobserved)
- Unobserved task exceptions don't crash the process by default in .NET 4.0+
- But they can cause subtle bugs: missing data, partial updates, silent failures
- Always `await` async calls or use `try-catch` around them
- `Task.WhenAll` collects all exceptions - always inspect `AggregateException.InnerExceptions`

```csharp
// BAD: Fire-and-forget, exception is lost
_ = SaveDataAsync(data);

// GOOD: Await and handle
try
{
    await SaveDataAsync(data);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Save failed");
}
```

---

## Best Practices

- Always catch specific exceptions, not broad `Exception` type
- Always use `throw` in catch blocks, never `throw ex`, to preserve stack traces
- Always dispose IDisposable resources using `using` statements or explicit `Dispose()` in `finally`
- Use exception filters (`when`) for logging patterns and conditional catch blocks
- Log exceptions with full context: message, stack trace, inner exception, and relevant parameters
- Don't catch exceptions you can't handle meaningfully - let them propagate to appropriate handlers
- Validate inputs early using guard clauses and throw appropriate argument exceptions
- Don't use exceptions for normal control flow - use conditional checks instead
- Implement `IAsyncDisposable` for types with async cleanup needs
- Follow the naming convention: `SomethingException` for custom exception types
- Include the standard three constructors in custom exceptions
- Call `GC.SuppressFinalize(this)` in `Dispose()` when a finalizer is implemented
- Consider thread safety when implementing the dispose pattern
- Document the exceptions your public methods can throw using XML documentation
- Use `ArgumentNullException.ThrowIfNull()` for null parameter validation in .NET 6+
- Consider using `ExceptionDispatchInfo` when you need to rethrow from a different context
- In async code, always observe task exceptions by awaiting them
- Use `ValueTask` carefully - only await once; for multiple awaits, convert to `Task`

---

## Interview Questions

### Basic Exception Handling

1. **What is the difference between `throw` and `throw ex`?**
   - `throw` preserves the original stack trace; `throw ex` resets the stack trace to the current line. Always use `throw` in catch blocks to maintain debugging information.

2. **Why does the `finally` block always execute?**
   - The CLR guarantees finally execution to ensure resource cleanup. It runs even with `return`, `break`, or `continue`. Only fails on `Environment.FailFast()`, `StackOverflowException`, or process termination.

3. **Can you have a try block without a catch block?**
   - Yes, if you have a `finally` block. The try-finally pattern is valid and useful when you need cleanup but don't want to handle the exception (let it propagate).

4. **What happens if an exception is thrown in a finally block?**
   - The exception from finally replaces any original exception. The original exception is lost. Keep finally blocks simple and consider wrapping risky cleanup in try-catch.

5. **What is the order of catch blocks when handling multiple exception types?**
   - Most specific exceptions must come first, most general last. The first matching catch block is executed. Putting a general catch first causes a compile error because subsequent catches become unreachable.

### Custom Exceptions

6. **How do you create a proper custom exception in C#?**
   - Inherit from `Exception`, implement three constructors (default, message, message+inner), mark as `[Serializable]`, and implement the serialization constructor. Name it with the `Exception` suffix.

7. **Why should custom exceptions be serializable?**
   - Serialization allows exceptions to cross process and AppDomain boundaries (remoting, WCF, logging). The `[Serializable]` attribute and serialization constructor enable this.

8. **When should you create a custom exception vs using an existing one?**
   - Create custom when you need domain-specific error information, callers need to distinguish your errors, or you need custom properties. Use existing exceptions when they already accurately describe the problem.

9. **What is the correct naming convention for custom exceptions?**
   - Always use the `Exception` suffix: `OrderProcessingException`, `PaymentFailedException`. Never use generic names like `OrderError`.

### Exception Filters

10. **What is the advantage of exception filters over catch + rethrow?**
    - Filters evaluate without unwinding the stack, preserving the full original stack trace. With catch+rethrow, the stack trace is truncated. Filters are also slightly more performant and produce cleaner code.

11. **Write an example of an exception filter that only catches 404 errors.**
    ```csharp
    catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
    {
        // Handle 404
    }
    ```

12. **Can you use complex logic in a `when` clause?**
    - Yes, but keep it simple. You can use `&&`, `||`, method calls, and property comparisons. Avoid complex logic that makes the code hard to read.

### IDisposable Pattern

13. **What is the purpose of the `using` statement?**
    - It's syntactic sugar that ensures `Dispose()` is called automatically when the block exits. It generates a try-finally where the finally calls `Dispose()`.

14. **What is the difference between `IDisposable.Dispose()` and a finalizer?**
    - `Dispose()` is deterministic and called explicitly or via `using`. The finalizer is non-deterministic, called by GC before reclamation, and only for unmanaged resource cleanup as a safety net.

15. **What does `GC.SuppressFinalize(this)` do and why is it important?**
    - It tells the GC not to queue the object for finalization. Called in `Dispose()` to prevent the performance overhead of finalization when cleanup is already done. Without it, objects go to the finalization queue unnecessarily.

16. **When should you use `IAsyncDisposable` instead of `IDisposable`?**
    - When disposal involves async operations (database connections, network streams, file I/O). Use `await using` syntax. Common with EF Core's `DbContext`.

17. **How do you prevent double disposal in the dispose pattern?**
    - Use a `_disposed` boolean flag. Check it at the start of `Dispose()` and return immediately if already disposed. Also check it in methods that use the resource and throw `ObjectDisposedException`.

### Async Exception Handling

18. **How do exceptions work in async methods?**
    - Exceptions are captured in the returned `Task`. When the task is awaited, the exception is rethrown. If the task is never awaited (fire-and-forget), the exception may go unobserved and silently lost.

19. **What is `AggregateException` and when does it occur?**
    - `AggregateException` wraps multiple exceptions. Occurs with `Task.WhenAll()` when multiple tasks fail, or with `Task.Wait()` on faulted tasks. Access individual exceptions via `InnerExceptions`.

20. **How do you handle exceptions from `Task.WhenAll`?**
    - Catch `AggregateException` and iterate over `InnerExceptions`, or use `.GetAwaiter().GetResult()` on individual tasks, or use a helper method to unwrap the AggregateException.

### Common Exceptions

21. **What is the difference between `NullReferenceException` and `ArgumentNullException`?**
    - `NullReferenceException` is a system exception thrown when accessing members on null. `ArgumentNullException` is an argument validation exception thrown by methods when a required parameter is null. Throw `ArgumentNullException`; never throw `NullReferenceException` explicitly.

22. **When would you throw `InvalidOperationException`?**
    - When an operation is invalid given the object's current state. Example: calling `Read()` on a closed stream, or attempting to enumerate a collection that's being modified.

23. **What is the difference between `ArgumentException`, `ArgumentNullException`, and `ArgumentOutOfRangeException`?**
    - `ArgumentException` is the base for argument exceptions. `ArgumentNullException` specifically for null arguments. `ArgumentOutOfRangeException` for arguments outside valid range. Use the most specific one.

### Advanced Topics

24. **What is `ExceptionDispatchInfo` and when would you use it?**
    - `ExceptionDispatchInfo.Capture(ex).Throw()` rethrows an exception while preserving the original stack trace, even across different contexts. Useful when you need to rethrow from a different location without losing stack information, or when working with `ValueTask` exceptions.

25. **How does exception handling differ between synchronous and asynchronous code?**
    - In synchronous code, exceptions bubble up the call stack immediately. In async code, exceptions are captured in the `Task` and only thrown when awaited. Not awaiting means exceptions go unobserved. `async` methods behave like returning `Task`, so exceptions are captured, not thrown immediately.

26. **What happens if a constructor throws an exception?**
    - The object is never fully constructed; the caller gets no reference. The finalizer will NOT run for the failed object. Any resources allocated before the exception in the constructor must be cleaned up within the constructor's try-finally, since the object never reaches the caller.

27. **What is the difference between `SystemException` and `ApplicationException`?**
    - Both are base classes for exceptions. `SystemException` is for runtime-thrown exceptions; `ApplicationException` was meant for application-defined exceptions. In modern .NET, this distinction is obsolete. Just inherit from `Exception` directly.

28. **How would you implement a global exception handler in a .NET application?**
    - For ASP.NET Core: use `app.UseExceptionHandler()` middleware. For console apps: wrap `Main` in try-catch or use `AppDomain.CurrentDomain.UnhandledException`. For WPF: `Application.DispatcherUnhandledException`. For WinForms: `Application.ThreadException`.
