# Logging & Serilog in ASP.NET Core

---

## Logging Basics

### Why Logging Matters

- **Debugging** — When something breaks in production, logs are your primary tool to understand what happened and why
- **Auditing** — Tracks who did what and when, essential for accountability and troubleshooting user-reported issues
- **Monitoring** — Provides real-time visibility into application health, error rates, and performance trends
- **Compliance** — Many industries (finance, healthcare, government) require detailed logging for regulatory compliance (GDPR, HIPAA, SOX)
- **Forensics** — After a security incident, logs are the only way to reconstruct what happened
- **Performance Analysis** — Slow queries, high latency, memory spikes — all discoverable through well-placed logs

### ILogger\<T\> Interface in ASP.NET Core

- `ILogger<T>` is the core logging abstraction in ASP.NET Core's dependency injection system
- Injected via constructor: `private readonly ILogger<MyController> _logger;`
- The generic type parameter `T` is used as the logging category (typically the class name)
- Methods: `LogTrace()`, `LogDebug()`, `LogInformation()`, `LogWarning()`, `LogError()`, `LogCritical()`
- Also supports `Log(LogLevel, EventId, Exception?, Message)` for programmatic level selection
- Works with dependency injection — register logging services with `builder.Services.AddLogging()`
- The default provider is `Microsoft.Extensions.Logging.Console` but is often replaced with Serilog

```csharp
public class OrderService : IOrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        _logger.LogInformation("Creating order for customer {CustomerId}", request.CustomerId);
        // ...
    }
}
```

### Log Levels

- **Trace** — Most verbose, fine-grained detail. Usually disabled in production. Useful for deep debugging of algorithm internals or loop iterations
- **Debug** — Diagnostic information for developers. Typically disabled in production. Variable values, method entry/exit points, branching logic
- **Information** — Normal application flow events. Application startup, request processing, significant state changes. Should be the default minimum in most applications
- **Warning** — Something unexpected but recoverable. A fallback value was used, a retry succeeded, a deprecated API was called. Not an error, but worth noting
- **Error** — Something failed. An exception was caught, a request couldn't be processed, a downstream service is unavailable. The operation failed but the app continues
- **Critical** — System-level failure. Process crash, data loss, security breach, out-of-memory. Requires immediate attention

### When to Use Each Level

| Level | Example |
|-------|---------|
| Trace | Variable values inside a loop, method parameters on every call |
| Debug | Query results, cache hit/miss, service resolution details |
| Information | "Order {OrderId} created", "User logged in", "Application started" |
| Warning | "Retry attempt 3 for external service", "Cache miss, falling back to database" |
| Error | "Failed to send email to {Email}", "Database connection failed" |
| Critical | "Ran out of memory", "Primary database unreachable", "Encryption key rotation failed" |

### Structured Logging vs String Interpolation

- **String interpolation** produces a single flat string at the call site — the logging framework cannot parse or search individual values
- **Structured logging** passes property names and values separately, letting the framework store them as distinct fields
- String interpolation: `logger.LogInformation($"Order {order.Id} for {customer.Name}");`
- Structured logging: `logger.LogInformation("Order {OrderId} for {CustomerName}", order.Id, customer.Name);`
- Structured logging is almost always preferred unless you have a specific reason not to

### Why Structured Logging Is Better

- **Searchable** — You can query logs where `OrderId = 12345` without regex on raw text
- **Filterable** — Filter by `Level = Warning` and `Service = PaymentGateway` simultaneously
- **Aggregatable** — Count occurrences of specific events, calculate percentiles for durations
- **Machine-readable** — JSON output means tools like Seq, Elasticsearch, and Splunk can index and analyze natively
- **Less disk space** — Structured data is typically smaller than equivalent string-interpolated messages with duplicated context

---

## Serilog

### What Is Serilog

- A structured logging library for .NET, designed from the ground up for structured event data
- Mature, battle-tested, and the de facto standard for structured logging in ASP.NET Core
- Stores log events as structured objects with named properties, not flat strings
- Extensible through sinks (where logs go), enrichers (what extra data is added), and formatters (how logs look)
- First-class ASP.NET Core integration via `Serilog.AspNetCore`

### Why Serilog Over Default ILogger

- **Sinks** — Dozens of pre-built sinks: Console, File, Seq, Elasticsearch, SQL Server, Application Insights, Datadog, etc.
- **Enrichers** — Add context automatically: thread ID, machine name, environment, correlation ID, custom properties
- **Structured data** — Natively stores objects, collections, and nested structures as queryable properties
- **Destructuring** — Automatically extracts properties from complex objects for structured storage
- **Configuration** — Rich configuration from code, JSON files, environment variables, or any `IConfiguration` source
- **Output templates** — Fine-grained control over log output format
- **Performance** — Batching, buffering, and async sinks minimize performance impact

### How to Configure Serilog in ASP.NET Core

```csharp
// Program.cs (.NET 6+)
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Add Serilog as the logging provider
builder.Host.UseSerilog((context, services, configuration) =>
{
    configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName()
        .WriteTo.Console();
});

var app = builder.Build();
// ...
app.Run();
```

- `ReadFrom.Configuration()` allows Serilog settings to come from `appsettings.json`
- `ReadFrom.Services()` enables sinks that depend on DI (like `IDbConnection`)
- `Enrich.FromLogContext()` adds properties from `LogContext.PushProperty()` calls (including correlation IDs)
- Must call `Serilog.Log.CloseAndFlush()` on application shutdown to avoid data loss

### Serilog Sinks

- **Console** — Writes to standard output. Essential for containerized apps and development. Configurable output template
- **File** — Writes to rolling log files. Supports daily, size-based, and custom rolling policies
- **Seq** — Ships logs to Seq server for structured log search, dashboards, and alerting. Excellent for development and production
- **Elasticsearch** — Writes to Elasticsearch indices for use with Kibana or OpenSearch Dashboards
- **SQL Server** — Writes log events to a SQL Server table. Useful when you already have SQL infrastructure
- **Application Insights** — Sends telemetry to Azure Application Insights for cloud-native monitoring
- **MongoDB** — Writes to MongoDB collections for flexible schema and horizontal scaling

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning",
        "Microsoft.Hosting.Lifetime": "Information"
      }
    },
    "WriteTo": [
      { "Name": "Console" },
      {
        "Name": "File",
        "Args": {
          "path": "Logs/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30
        }
      },
      {
        "Name": "Seq",
        "Args": {
          "serverUrl": "http://localhost:5341"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithThreadId"],
    "Properties": {
      "Application": "MyWebApi"
    }
  }
}
```

### Serilog Enrichers

- Enrichers attach additional properties to every log event automatically
- **FromLogContext** — Picks up properties pushed via `LogContext.PushProperty()` (correlation IDs, user IDs, etc.)
- **WithMachineName** — Adds the hostname of the server
- **WithThreadId** — Adds the managed thread ID (useful for diagnosing concurrency issues)
- **WithEnvironmentName** — Adds `ASPNETCORE_ENVIRONMENT` value
- **WithExceptionDetails** — Adds structured exception properties (inner exceptions, data dictionary)
- Custom enrichers implement `ILogEventEnricher` and can add any property you need

```csharp
public class CorrelationIdEnricher : ILogEventEnricher
{
    private readonly ICorrelationIdProvider _provider;

    public CorrelationIdEnricher(ICorrelationIdProvider provider)
    {
        _provider = provider;
    }

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        logEvent.AddPropertyIfAbsent(
            propertyFactory.CreateProperty("CorrelationId", _provider.CorrelationId));
    }
}
```

### Log Formatting and Output Templates

- Output templates control how log events appear in the console or file sinks
- Properties are referenced with `{PropertyName}` in the template
- Use `:format` for numeric and date formatting: `{Timestamp:yyyy-MM-dd HH:mm:ss}`
- Use `-` prefix for minimal output: `{Properties:j}` outputs all properties as JSON
- Default console template: `[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{Properties:j}{NewLine}{Exception}`

```
[10:30:45 INF] Order {OrderId:0} created for customer {CustomerId}
[10:30:45 WRN] Retry attempt {Attempt} for service {ServiceName}
[10:30:46 ERR] Payment failed {PaymentId} - {ErrorMessage}
```

### File Rolling

- **Daily** (`rollingInterval: "Day"`) — New file each day: `log-20260724.txt`, `log-20260725.txt`
- **Hourly** (`rollingInterval: "Hour"`) — New file each hour. Useful for high-volume logging
- **Size-based** — New file when current file reaches a size limit: `fileSizeLimitBytes: 10_000_000`
- **Custom** — Implement `IRollingFileSink` for custom rolling logic
- Always set `retainedFileCountLimit` to avoid filling disk: `retainedFileCountLimit: 31` keeps one month
- Combined rolling: daily rolling with size limit per file

```json
{
  "Name": "File",
  "Args": {
    "path": "Logs/log-.txt",
    "rollingInterval": "Day",
    "fileSizeLimitBytes": 52428800,
    "retainedFileCountLimit": 30,
    "restrictedToMinimumLevel": "Information"
  }
}
```

### Minimum Log Level Configuration

- Set globally via configuration: `"MinimumLevel": { "Default": "Information" }`
- Override per namespace to reduce noise from verbose libraries:
  - `"Microsoft.AspNetCore": "Warning"`
  - `"Microsoft.EntityFrameworkCore": "Warning"`
  - `"System.Net.Http": "Warning"`
- Can be changed at runtime without restarting the application
- Per-sink minimum level allows different sinks to capture different levels (e.g., Console = Information, File = Debug)

### Logging Configuration from appsettings.json

- Serilog reads its configuration from `appsettings.json` (or environment-specific variants)
- Separates logging configuration from application code
- Enables different configurations per environment without code changes
- Environment variables can override JSON values (useful for containerized deployments)
- Supports hierarchical configuration: `Serilog:WriteTo:0:Name` format for arrays

---

## Structured Logging

### What Is Structured Logging

- Instead of embedding values directly in a message template, structured logging passes named placeholders
- The logging framework stores each property separately alongside the message template
- Result: a log event is a structured object with a template and a dictionary of named properties
- Enables tools to index, search, and aggregate on individual properties

```csharp
// Bad: string interpolation — produces "Order 12345 for customer John Smith"
_logger.LogInformation($"Order {order.Id} for customer {customer.Name}");

// Good: structured logging — produces template "Order {OrderId} for customer {CustomerName}"
// with properties OrderId=12345, CustomerName="John Smith"
_logger.LogInformation("Order {OrderId} for customer {CustomerName}", order.Id, customer.Name);
```

### How Serilog Handles Structured Data

- Serilog captures the message template and parameters as-is, without string formatting
- Complex objects are automatically destructured (default depth) to extract their properties
- Collections are captured as structured arrays
- Nested objects have their properties flattened into the log event
- The `Destructurama.Attributed` package adds attributes to control destructuring behavior

```csharp
var order = new Order
{
    Id = 12345,
    Customer = new Customer { Name = "Alice", Email = "alice@example.com" },
    Items = new List<OrderItem> { /* ... */ }
};

// Serilog will store order.Id, order.Customer.Name, order.Customer.Email as separate properties
_logger.LogInformation("Order created: {@Order}", order);
```

### @ (Destructured) vs $ (Stringified) Operators

- **`@`** — Destructures the object, storing its properties as individual named properties in the log event
  - `{@Order}` stores `Order.Id`, `Order.Customer.Name`, etc. as top-level properties
  - Enables searching: `WHERE OrderId = 12345` in Seq or Elasticsearch
- **`$`** — Converts the object to its `ToString()` representation, stored as a single string property
  - `{Order$}` stores the entire object as a single string value in the `Order` property
  - Useful for simple objects where you want a human-readable summary
- Default (no operator) — Uses the type's `ToString()` method, stored as a string

```csharp
_logger.LogInformation("Full order details: {@Order}", order);    // Destructured — properties extracted
_logger.LogInformation("Order summary: {Order$}", order);          // Stringified — ToString() output
_logger.LogInformation("Order ID: {OrderId}", order.Id);           // Explicit property
```

### Why Structured Logging Enables Powerful Queries

- Each property becomes a searchable field in your log store
- Seq, Elasticsearch, Splunk, and other tools can filter and aggregate on these fields
- You can answer questions like: "Show all errors where CustomerId = 42 in the last 24 hours"
- Aggregations: "Count orders per CustomerId per day"
- Alerts: "Alert when ErrorRate for ServiceName = PaymentGateway exceeds 5% in 5 minutes"
- Without structured logging, you'd need regex or text search on flat strings — unreliable and slow

### Searching Logs by Property Values

- In Seq: `CustomerId = 42 and Level = 'Error'`
- In Elasticsearch/Kibana: `properties.CustomerId:42 AND properties.Level:"Error"`
- In SQL Server sink: `SELECT * FROM Logs WHERE CustomerId = 42 AND Level = 'Error'`
- With flat string logging, you'd have to search for "Customer 42" in the message text, which matches false positives and is slow

### Performance Considerations

- Structured logging is actually faster than string interpolation because it avoids string formatting when the log level is disabled
- If `Information` level is disabled, the `LogInformation` call returns immediately without formatting the message template
- With string interpolation, the string is always formatted regardless of whether the log level is enabled
- Serilog's property capture is allocation-efficient for common cases
- Batching sinks (like `Seq`) buffer events and send them asynchronously, minimizing I/O impact

---

## Correlation IDs

### What Is a Correlation ID

- A unique identifier (typically a GUID or UUID) assigned to a single request or operation
- Stays with the request as it flows through middleware, services, external calls, and message queues
- Enables tracing the complete lifecycle of a request across multiple components
- Typically generated at the entry point (middleware) and propagated via HTTP headers or message metadata

### Why Correlation IDs Matter

- **Request tracing** — Follow a single request through multiple services in a microservice architecture
- **Debugging** — When an error occurs, the correlation ID links all related log entries together
- **Performance analysis** — Track total request duration across service boundaries
- **User support** — Users report "it didn't work" — the correlation ID lets you find the exact logs
- **Distributed systems** — Essential when a request touches 5+ services; without it, correlation is nearly impossible
- **Compliance** — Some regulations require traceability of operations end-to-end

### How to Implement in ASP.NET Core (Middleware)

```csharp
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;
    private const string CorrelationIdHeader = "X-Correlation-Id";

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Use incoming correlation ID if present, otherwise generate a new one
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString("D");

        // Store in HttpContext for current request
        context.Items["CorrelationId"] = correlationId;

        // Add to response headers
        context.Response.OnStarting(() =>
        {
            context.Response.Headers[CorrelationIdHeader] = correlationId;
            return Task.CompletedTask;
        });

        // Push into Serilog LogContext so all log events include it
        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}
```

- Register in the pipeline: `app.UseMiddleware<CorrelationIdMiddleware>();`
- Must be registered early in the pipeline to capture all downstream logs
- The `LogContext.PushProperty()` call adds the correlation ID to every log event within its scope
- The `using` block ensures the property is removed when the request completes

### AsyncLocal\<T\> for Flowing Correlation Context

- `AsyncLocal<T>` stores data that flows with the async execution context
- Each async call inherits the value from its parent, but changes in a child don't affect the parent
- Used internally by `LogContext` to propagate properties across async boundaries
- Alternative to passing correlation ID through every method parameter
- Works across `Task.Run()`, `await`, and other async patterns

```csharp
public static class CorrelationContext
{
    private static readonly AsyncLocal<string> _correlationId = new();

    public static string CorrelationId
    {
        get => _correlationId.Value;
        set => _correlationId.Value = value;
    }
}
```

- Be careful: `AsyncLocal` does not automatically flow across thread pool boundaries in all cases
- `ExecutionContext` flows with `await` by default but may not flow with `Task.Run()` without explicit capture

### Serilog Enrichers for Correlation ID

- Instead of using `LogContext.PushProperty()` in middleware, you can create a dedicated enricher
- The enricher can pull the correlation ID from `HttpContext.Items`, a service, or `AsyncLocal<T>`
- Register the enricher in Serilog configuration for cleaner separation of concerns
- Enables reuse across different ASP.NET Core applications

### Request Logging Middleware Pattern

- Serilog provides built-in request logging: `app.UseSerilogRequestLogging()`
- Logs the HTTP method, path, status code, and elapsed time for every request
- Captures the request as a structured event with all properties available for search
- Can be configured to exclude health check endpoints or static files

```csharp
app.UseSerilogRequestLogging(options =>
{
    options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers.UserAgent.ToString());
        return Task.CompletedTask;
    };
});
```

---

## Logging Best Practices

### Log at the Right Level

- **Trace** — Almost never used in production; enable only when deep-debugging a specific issue
- **Debug** — Disable in production; use during development to inspect variable values and flow
- **Information** — The baseline for production; captures meaningful application events
- **Warning** — Something unexpected happened but the system recovered; worth reviewing periodically
- **Error** — Something failed and the user was affected; requires investigation
- **Critical** — System is in a compromised state; requires immediate intervention
- If you're not sure, err on the side of too little rather than too much — you can always increase verbosity later

### Include Structured Context

- Always include IDs that link to your data: `OrderId`, `CustomerId`, `UserId`, `CorrelationId`
- Include the operation or method name for easy identification: `"Processing payment"` not just `"Processing"`
- Include relevant metadata: retry count, elapsed time, service name, error code
- Use `BeginScope` or `LogContext.PushProperty` for request-level context that applies to all log events

```csharp
using (var scope = _logger.BeginScope(new Dictionary<string, object>
{
    ["UserId"] = currentUser.Id,
    ["CorrelationId"] = correlationId,
    ["TenantId"] = tenant.Id
}))
{
    _logger.LogInformation("Processing order {OrderId}", order.Id);
    // Both UserId and CorrelationId appear in every log event within this scope
}
```

### Never Log Sensitive Data

- **Passwords** — Never log user passwords, even hashed ones
- **API keys and tokens** — Never log bearer tokens, API keys, or connection strings with passwords
- **PII** — Avoid logging email addresses, phone numbers, social security numbers, credit card numbers
- **Health data** — Medical records, diagnoses, treatment details (HIPAA violation)
- **Financial data** — Account numbers, balances, transaction details beyond what's needed
- Use redaction or masking when you need partial visibility: `card=****1234`, `email=a***@example.com`
- Serilog's `Destructurama` and custom sinks can enforce redaction policies

### Log Exceptions with Full Stack Trace

- Always pass the exception object to the logging method: `_logger.LogError(ex, "Failed to process order {OrderId}", order.Id);`
- Never call `ex.ToString()` and log the string — lose structured exception properties
- Serilog captures exception type, message, stack trace, inner exceptions, and data dictionary as structured properties
- Avoid: `_logger.LogError(ex.ToString());` — loses the structured exception data

```csharp
// Bad — exception is stringified, losing structure
_logger.LogError(ex.ToString());

// Bad — exception is not included at all
_logger.LogError("Failed to process order");

// Good — exception is structured, with context
_logger.LogError(ex, "Failed to process order {OrderId} for customer {CustomerId}",
    order.Id, order.CustomerId);
```

### Use BeginScope for Request-Level Context

- `BeginScope` creates a logging scope that adds properties to all log events within it
- Ideal for per-request context: correlation ID, user ID, tenant ID, request path
- Scope is automatically disposed when the block exits (or when the request completes)
- Works with Serilog's `Enrich.FromLogContext()` to add scope properties to the log event

### Log Incoming Requests and Outgoing Responses

- Log the request (method, path, headers, body) at Information or Debug level
- Log the response (status code, body, duration) at the same level
- Log external service calls (URL, method, status, duration) to diagnose integration issues
- Be careful with request/response bodies — they can be large and may contain sensitive data
- Use sampling or conditional logging for high-volume endpoints

### Performance Impact of Logging

- Structured logging with Serilog is fast when the log level is disabled (early return, no formatting)
- Batching sinks (Seq, Elasticsearch) buffer events and send them in bulk — minimal I/O impact
- Console logging is synchronous by default — avoid in production or use `Async` wrapper
- File logging with rolling is efficient but consider async wrappers for high-throughput scenarios
- Avoid string concatenation in log calls — always use structured templates
- Profile your application with logging enabled to ensure it's not the bottleneck

---

## Common Mistakes

### Using String Interpolation Instead of Structured Logging

- Produces a flat string that cannot be searched or filtered by individual properties
- Forces string formatting even when the log level is disabled
- Loses type information — numbers become strings, dates become strings
- Makes log aggregation tools (Seq, Elasticsearch) far less useful
- Always prefer: `_logger.LogInformation("Order {OrderId} created", order.Id);`

### Logging Too Much in Production

- Verbose logging creates noise that makes finding real issues difficult
- High-volume logs consume disk space and increase I/O load
- Log every decision point in a loop can produce thousands of events per second
- Set appropriate minimum levels per namespace — `Microsoft.AspNetCore` should be `Warning`
- Use log level overrides to reduce noise from verbose libraries

### Logging Sensitive Data

- Passwords, tokens, PII, health data, financial data should never appear in logs
- Even in development, practice good habits — logs can accidentally end up in shared environments
- Use structured logging to log only the properties you need, not entire objects blindly
- Implement redaction at the sink level for defense in depth

### Not Logging Enough for Debugging

- Under-logging is worse than over-logging for production debugging
- Log key decision points, state changes, and error conditions
- Include enough context to reproduce the issue: IDs, timestamps, user information
- Use Debug level for detailed diagnostic information that's only enabled when needed

### Ignoring Log File Rotation

- Without rotation, log files grow unbounded until disk is full
- Application crashes when disk is full — a logging failure causes a system failure
- Always configure file rolling with size or time limits
- Set `retainedFileCountLimit` to control how many old files to keep
- Monitor disk usage even with rotation enabled

### Synchronous Logging in Async Code

- Console and File sinks are synchronous by default
- Synchronous I/O in async methods defeats the purpose of async and can cause thread pool starvation
- Use `Serilog.Sinks.Async` to wrap any sink with an async buffer
- Alternatively, use `WriteTo.Async(wt => wt.Console())` configuration

```csharp
// Wrapping sinks with async buffer
.WriteTo.Async(a =>
{
    a.Console();
    a.File("Logs/log-.txt", rollingInterval: RollingInterval.Day);
})
```

---

## Real-World Patterns

### Request/Response Logging Middleware

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        var requestId = Guid.NewGuid().ToString("N");

        context.Items["RequestId"] = requestId;

        try
        {
            _logger.LogInformation(
                "HTTP {RequestMethod} {RequestPath} started from {RemoteIp}",
                context.Request.Method,
                context.Request.Path,
                context.Connection.RemoteIpAddress?.ToString());

            await _next(context);

            stopwatch.Stop();

            _logger.LogInformation(
                "HTTP {RequestMethod} {RequestPath} completed with {StatusCode} in {ElapsedMs}ms",
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            _logger.LogError(ex,
                "HTTP {RequestMethod} {RequestPath} failed after {ElapsedMs}ms",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds);

            throw;
        }
    }
}
```

### Exception Logging with Context

```csharp
public class OrderService : IOrderService
{
    private readonly ILogger<OrderService> _logger;
    private readonly IPaymentGateway _paymentGateway;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        try
        {
            _logger.LogInformation(
                "Creating order for customer {CustomerId} with {ItemCount} items",
                request.CustomerId,
                request.Items.Count);

            var order = await ProcessOrderAsync(request);

            await _paymentGateway.ChargeAsync(order.Total);

            _logger.LogInformation(
                "Order {OrderId} created successfully with total {Total:C}",
                order.Id,
                order.Total);

            return order;
        }
        catch (PaymentDeclinedException ex)
        {
            _logger.LogWarning(ex,
                "Payment declined for order attempt by customer {CustomerId}: {Reason}",
                request.CustomerId,
                ex.Reason);

            throw new OrderCreationException("Payment was declined", ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Unexpected error creating order for customer {CustomerId}",
                request.CustomerId);

            throw;
        }
    }
}
```

### Audit Logging for Compliance

```csharp
public class AuditLogService : IAuditLogService
{
    private readonly ILogger<AuditLogService> _logger;

    public AuditLogService(ILogger<AuditLogService> logger)
    {
        _logger = logger;
    }

    public void LogAccess(string userId, string resource, string action, bool granted)
    {
        // Use Information level — audit logs are always important
        _logger.LogInformation(
            "AUDIT: User {UserId} attempted {Action} on {Resource} — {Result}",
            userId,
            action,
            resource,
            granted ? "GRANTED" : "DENIED");
    }

    public void LogDataModification(string userId, string entityType, string entityId,
        string operation, object changes)
    {
        _logger.LogInformation(
            "AUDIT: User {UserId} performed {Operation} on {EntityType} {EntityId} — {@Changes}",
            userId,
            operation,
            entityType,
            entityId,
            changes);
    }
}
```

### Performance Logging (Request Duration)

```csharp
public class PerformanceLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceLoggingMiddleware> _logger;

    private static readonly HashSet<string> ExcludedPaths = new(StringComparer.OrdinalIgnoreCase)
    {
        "/health",
        "/health/ready",
        "/metrics",
        "/swagger"
    };

    public PerformanceLoggingMiddleware(RequestDelegate next,
        ILogger<PerformanceLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (ExcludedPaths.Contains(context.Request.Path))
        {
            await _next(context);
            return;
        }

        var stopwatch = Stopwatch.StartNew();

        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            var elapsed = stopwatch.ElapsedMilliseconds;

            var logLevel = elapsed switch
            {
                > 5000 => LogLevel.Warning,
                > 2000 => LogLevel.Information,
                _ => LogLevel.Debug
            };

            _logger.Log(logLevel,
                "Request {Method} {Path} completed in {ElapsedMs}ms with status {StatusCode}",
                context.Request.Method,
                context.Request.Path,
                elapsed,
                context.Response.StatusCode);
        }
    }
}
```

### Log Aggregation with Seq

- Seq is a purpose-built server for structured log events
- Provides a web UI for searching, filtering, and dashboards
- Supports SQL-like queries on log properties: `WHERE Level = 'Error' AND DurationMs > 1000`
- Alerts: send email or webhook when error rate exceeds threshold
- Free for single-user development; commercial licenses for teams
- Self-hosted or Seq Cloud (SaaS)

```csharp
// Seq configuration
.WriteTo.Seq(
    serverUrl: "http://localhost:5341",
    apiKey: "your-api-key",
    batchPostingLimit: 50,
    period: TimeSpan.FromSeconds(2))
```

- Seq queries for common scenarios:
  - All errors: `Level = 'Error'`
  - Slow requests: `ElapsedMs > 2000`
  - Failed payments: `SourceContext = 'PaymentService' AND Level = 'Error'`
  - Specific user: `UserId = 'user-123'`
  - Time range: `@t > '2026-07-24T00:00:00'`

---

## Interview Questions

### 1. What are the benefits of structured logging over traditional string-based logging?

Structured logging stores log events as queryable objects with named properties, enabling powerful searches, filters, and aggregations. Traditional string logging produces flat text that requires regex to search and cannot be aggregated by property. Structured logging also avoids string formatting when the log level is disabled, improving performance.

### 2. How would you configure Serilog in an ASP.NET Core application?

Add `Serilog.AspNetCore` NuGet package, call `builder.Host.UseSerilog()` in Program.cs, configure sinks, enrichers, and minimum levels either in code or via `appsettings.json`. Read configuration from `context.Configuration` for environment-specific overrides. Always call `Log.CloseAndFlush()` on shutdown.

### 3. Explain the difference between @ and $ operators in Serilog message templates.

The `@` operator destructures an object, extracting its properties as individual structured fields in the log event — enabling search and aggregation on those properties. The `$` operator converts the object to its `ToString()` representation and stores it as a single string property. Use `@` for objects you want to query; use `$` for simple display strings.

### 4. What are the six log levels in ASP.NET Core and when would you use each?

Trace (finest detail, disabled in production), Debug (diagnostic info for developers), Information (normal application flow events), Warning (unexpected but recoverable situations), Error (failed operations that affect users), Critical (system-level failures requiring immediate attention).

### 5. How would you implement correlation IDs in a microservices architecture?

Generate a correlation ID at the first service, propagate it via HTTP headers (`X-Correlation-Id`) to downstream services. Use `LogContext.PushProperty()` or a Serilog enricher to attach it to all log events. Use `AsyncLocal<T>` to flow the correlation ID through async execution contexts. Include it in response headers for client-side tracing.

### 6. What is the performance impact of logging in a high-throughput application?

Structured logging with Serilog has minimal impact when the log level is disabled (early return, no formatting). Batching sinks buffer events and send asynchronously. Console logging is synchronous — use async wrappers. Avoid string concatenation. Profile with logging enabled. Most performance issues come from synchronous sinks, not the logging framework itself.

### 7. How do you prevent logging sensitive data like passwords and API keys?

Never log password fields or tokens directly. Use structured logging to log only specific properties, not entire objects. Implement redaction at the sink level. Use attributes from `Destructurama.Attributed` to mark properties as sensitive. Establish code review practices to catch sensitive data in log statements.

### 8. Explain the Serilog enricher concept with an example.

Enrichers attach additional properties to every log event automatically. For example, `WithMachineName()` adds the hostname, `FromLogContext()` picks up properties from `LogContext.PushProperty()`, and a custom enricher implementing `ILogEventEnricher` can add any property — like user ID or tenant ID — without passing it through every log call.

### 9. How would you set up different log levels for different namespaces?

Use the `Override` section in configuration: set `Default` to `Information`, then override specific namespaces to `Warning` or `Debug`. For example, set `Microsoft.AspNetCore` to `Warning` to suppress verbose framework logs while keeping your application logs at `Information`. This can be done in `appsettings.json` or programmatically.

### 10. What is file rolling and why is it important?

File rolling creates new log files based on time intervals (daily, hourly) or file size limits, preventing any single log file from growing unbounded. Without it, disk space fills up and the application crashes. Configure `rollingInterval`, `fileSizeLimitBytes`, and `retainedFileCountLimit` to control rolling behavior and retention.

### 11. Describe the request logging middleware pattern in ASP.NET Core.

Serilog provides `UseSerilogRequestLogging()` which logs HTTP method, path, status code, and duration for every request. Custom middleware can add correlation IDs, user information, request bodies (for debugging), and performance metrics. Place it early in the pipeline to capture all downstream activity. Use `EnrichDiagnosticContext` to add custom properties.

### 12. How does `BeginScope` work and when should you use it?

`BeginScope` creates a logging scope that adds properties to all log events within the scope's lifetime. Use it for request-level context like correlation ID, user ID, or tenant ID. The scope is disposed when the block exits or the request completes. Works with Serilog's `Enrich.FromLogContext()` to include scope properties in structured log events.

### 13. Why should you never use string interpolation in log statements?

String interpolation forces string formatting at the call site even when the log level is disabled, wasting CPU cycles. It produces flat strings that cannot be searched or filtered by individual values. It loses type information. Structured logging templates are parsed once and reused, avoiding repeated formatting, and the properties remain individually queryable.

### 14. What is the difference between LogError and LogWarning?

`LogError` indicates a failure that affected the operation or user — something broke. `LogWarning` indicates something unexpected happened but the system recovered or the operation completed — a degraded state, not a failure. Errors trigger alerts; warnings are reviewed periodically. Misusing levels makes monitoring and alerting unreliable.

### 15. How would you handle logging in a distributed system with multiple services?

Propagate correlation IDs across service boundaries via HTTP headers or message metadata. Use a centralized log aggregation system (Seq, Elasticsearch, Splunk) that all services ship logs to. Include service name as an enricher. Use structured logging consistently across all services. Implement distributed tracing (OpenTelemetry) alongside logging for full request visualization.

### 16. Explain how async logging works and why it matters.

Async logging uses a background thread or channel to buffer log events and write them to sinks without blocking the calling thread. This prevents synchronous I/O from degrading application performance, especially under high load. Serilog's `Serilog.Sinks.Async` wraps any sink with async buffering. Critical for preventing thread pool starvation in async ASP.NET Core applications.

### 17. What metrics should you monitor through your logging infrastructure?

Request rate and latency percentiles (p50, p95, p99), error rate by endpoint, exception frequency by type, slow query counts, retry rates for external services, disk usage of log files, log ingestion rate to aggregation systems, and correlation of errors to specific deployments or changes.

---

## Key Takeaways

- Use structured logging templates (`{Property}`) everywhere — never string interpolation
- Configure Serilog with sinks for your needs (Console for dev, Seq for production, File for retention)
- Set minimum log levels per namespace to reduce noise from framework libraries
- Implement correlation IDs early — they're essential for distributed tracing
- Never log sensitive data — use redaction and structured logging to control what's stored
- Configure file rolling and retention to prevent disk exhaustion
- Use async logging wrappers in production to avoid performance degradation
- Log at the right level — Information for normal flow, Error for failures, Critical for system issues
- Include sufficient context in every log event: IDs, timing, operation names
- Use `BeginScope` to attach request-level context to all log events within a request
