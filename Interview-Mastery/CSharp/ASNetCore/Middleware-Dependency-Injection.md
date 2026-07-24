# ASP.NET Core Middleware & Dependency Injection

---

## 1. Dependency Injection (DI)

### What is Dependency Injection?

- Dependency Injection is a design pattern where an object receives its dependencies from an external source rather than creating them itself
- It is a form of **Inversion of Control (IoC)** — instead of a class controlling the creation of its dependencies, that responsibility is inverted and given to an external container
- Instead of `MyService` creating its own `IEmailSender`, the DI container provides it
- This decouples the consumer from the concrete implementation

```csharp
// WITHOUT DI — tightly coupled
public class OrderService
{
    private readonly SqlEmailSender _emailSender = new SqlEmailSender(); // hardcoded dependency

    public void PlaceOrder(Order order)
    {
        // process order...
        _emailSender.Send(order.CustomerEmail, "Order placed!");
    }
}

// WITH DI — loosely coupled
public class OrderService
{
    private readonly IEmailSender _emailSender;

    public OrderService(IEmailSender emailSender) // dependency is injected
    {
        _emailSender = emailSender;
    }

    public void PlaceOrder(Order order)
    {
        // process order...
        _emailSender.Send(order.CustomerEmail, "Order placed!");
    }
}
```

---

### Why DI is Important

- **Testability** — you can inject mock/stub implementations during unit testing without touching real databases or APIs
- **Loose Coupling** — components depend on abstractions (interfaces), not concrete types, making it easy to swap implementations
- **Maintainability** — changes to a dependency's implementation don't ripple through the entire codebase
- **Separation of Concerns** — each class only worries about its own logic, not how to construct its dependencies
- **Composability** — the composition root (Program.cs) wires everything together in one place, making the app's structure visible and configurable
- **Lifetime Management** — the DI container manages object lifetimes, preventing memory leaks and ensuring proper disposal

---

### Built-in DI Container in ASP.NET Core

- ASP.NET Core ships with a **built-in, lightweight DI container** — no need for third-party containers like Autofac or Ninject for most apps
- It is configured in `Program.cs` using the `IServiceCollection` interface
- Services are registered during app startup and resolved automatically by the framework
- The container is **minimal by design** — it supports constructor injection, but does NOT support property injection or method injection out of the box
- Third-party containers (Autofac, SimpleInjector) can be plugged in via `UseAutofac()` if you need advanced features like open generics or property injection

```csharp
// Program.cs — registering and using DI
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<IOrderService, OrderService>();

var app = builder.Build();

app.MapGet("/orders/{id}", async (int id, IOrderService orderService) =>
{
    var order = await orderService.GetByIdAsync(id);
    return order is not null ? Results.Ok(order) : Results.NotFound();
});

app.Run();
```

---

### Service Lifetimes

#### Singleton

- The container creates **one instance** for the entire application lifetime
- The same object is returned for every resolution, across all HTTP requests
- Best for: stateless services, caching, configuration, logging, Redis clients
- Registered with `AddSingleton<TService>()` or `AddSingleton<TService>(instance)`
- **Caution:** singleton services live for the entire app lifetime — they must be thread-safe because multiple requests will share the same instance

```csharp
// Singleton — one instance for the entire app
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddSingleton<IMyConfig, MyConfig>();  // options/config objects

public class RedisCacheService : ICacheService
{
    private readonly ConnectionMultiplexer _connection;

    public RedisCacheService(IConfiguration config)
    {
        // This constructor runs ONCE for the entire app lifetime
        _connection = ConnectionMultiplexer.Connect(config["Redis:ConnectionString"]);
    }

    public string Get(string key) => _connection.GetDatabase().GetString(key);
}
```

#### Scoped

- The container creates **one instance per HTTP request** (or per scope)
- A new instance is created at the start of the request and disposed at the end
- Best for: `DbContext`, `Unit of Work`, any service that holds request-specific state
- Registered with `AddScoped<TService>()`
- **This is the most common lifetime for data access services**

```csharp
// Scoped — one instance per HTTP request
builder.Services.AddScoped<IAppDbContext, AppDbContext>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

public class OrderController : ControllerBase
{
    private readonly IOrderRepository _repo;
    private readonly IUnitOfWork _unitOfWork;

    // Both _repo and _unitOfWork share the SAME DbContext instance
    // because they are all Scoped and within the same request
    public OrderController(IOrderRepository repo, IUnitOfWork unitOfWork)
    {
        _repo = repo;
        _unitOfWork = unitOfWork;
    }
}
```

#### Transient

- The container creates a **new instance every time** it is resolved
- Each `GetService()` or constructor injection creates a fresh object
- Best for: lightweight, stateless services, validators, utilities
- Registered with `AddTransient<TService>()`
- Be careful with transient services that hold expensive resources — they are created and disposed frequently

```csharp
// Transient — new instance every time it is resolved
builder.Services.AddTransient<IValidator<Order>, OrderValidator>();
builder.Services.AddTransient<ILogger, FileLogger>();

// Each time IValidator<Order> is injected, a brand new OrderValidator is created
public class OrderController : ControllerBase
{
    private readonly IValidator<Order> _validator;  // new instance

    public OrderController(IValidator<Order> validator)
    {
        _validator = validator;  // fresh instance for this resolution
    }
}
```

---

### How to Register Services

```csharp
// Using IServiceCollection in Program.cs or a Startup class

// 1. Singleton — one instance forever
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddSingleton<IMyConfig>(new MyConfig { Key = "value" }); // existing instance

// 2. Scoped — one instance per request
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// 3. Transient — new instance every time
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();
builder.Services.AddTransient<IValidator<Order>, OrderValidator>();
```

---

### Factory-based Registration

- Use a factory delegate when you need custom logic to create the service instance
- The `IServiceProvider` parameter gives you access to the container so you can resolve other dependencies inside the factory
- Useful when you need to read configuration, pick an implementation at runtime, or pass constructor arguments

```csharp
// Factory-based registration — custom creation logic
builder.Services.AddScoped<IOrderRepository>(sp =>
{
    var connectionString = sp.GetRequiredService<IConfiguration>()
                             .GetConnectionString("DefaultConnection");
    var logger = sp.GetRequiredService<ILogger<SqlOrderRepository>>();
    return new SqlOrderRepository(connectionString, logger);
});

// Factory with conditional logic
builder.Services.AddSingleton<ICacheService>(sp =>
{
    var env = sp.GetRequiredService<IWebHostEnvironment>();
    if (env.IsDevelopment())
        return new InMemoryCacheService();  // simple in-memory for dev
    return new RedisCacheService(sp.GetRequiredService<IConfiguration>()); // Redis in prod
});
```

---

### Open Generic Registration

- Register a generic type without specifying the concrete type arguments
- The DI container resolves the correct closed type when you inject `IRepository<SomeEntity>`
- Extremely useful for repository patterns and generic CRUD services

```csharp
// Open generic registration
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
builder.Services.AddScoped(typeof(IReadOnlyRepository<>), typeof(ReadOnlyRepository<>));

// Usage — container closes the generic automatically
public class OrderController : ControllerBase
{
    private readonly IRepository<Order> _orderRepo;       // resolves Repository<Order>
    private readonly IRepository<Customer> _customerRepo; // resolves Repository<Customer>

    public OrderController(IRepository<Order> orderRepo, IRepository<Customer> customerRepo)
    {
        _orderRepo = orderRepo;
        _customerRepo = customerRepo;
    }
}
```

---

### Keyed Services (.NET 8+)

- Register multiple implementations of the same interface, distinguished by a string key
- Resolve a specific implementation using `[FromKeyedServices("key")]` attribute or `IServiceProvider.GetKeyedService<T>()`
- Useful when you have multiple strategies or implementations for the same contract

```csharp
// Keyed services — multiple implementations of the same interface
builder.Services.AddKeyedSingleton<ICacheService, RedisCacheService>("redis");
builder.Services.AddKeyedSingleton<ICacheService, MemcachedCacheService>("memcached");
builder.Services.AddKeyedScoped<IEmailSender, SmtpEmailSender>("smtp");
builder.Services.AddKeyedScoped<IEmailSender, SendGridEmailSender>("sendgrid");

// Resolving by key — using the attribute
[ApiController]
[Route("api/[controller]")]
public class NotificationController : ControllerBase
{
    [HttpPost]
    public IActionResult Send(
        [FromKeyedServices("smtp")] IEmailSender emailSender,
        NotificationDto notification)
    {
        emailSender.Send(notification.Email, notification.Message);
        return Ok();
    }
}

// Resolving by key — using IServiceProvider directly
public class Startup
{
    public void Configure(IApplicationBuilder app)
    {
        var redisCache = app.ApplicationServices.GetKeyedService<ICacheService>("redis");
    }
}
```

---

### Core DI Interfaces

#### IServiceCollection

- A collection of service descriptors used during app configuration
- You register all your services by calling extension methods on this interface
- Available in `Program.cs` via `builder.Services`
- Each call to `AddSingleton`, `AddScoped`, or `AddTransient` adds a `ServiceDescriptor` to the collection

#### IServiceProvider

- The built-in service locator — resolves registered services at runtime
- Created internally by the DI container after all services are registered
- Used by the framework to inject dependencies into controllers, middleware, and other components
- Avoid injecting `IServiceProvider` directly (service locator anti-pattern) — prefer constructor injection

#### IServiceScope

- A scoped container that creates a bounded lifetime for resolving services
- All scoped and transient services resolved within a scope share the same instances
- When the scope is disposed, all scoped services within it are disposed
- Automatically created for each HTTP request by the framework

```csharp
// IServiceScope — creating a manual scope
using var scope = app.ApplicationServices.CreateScope();
var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
// repo and any scoped services it depends on live until scope is disposed
```

---

### Avoiding Captive Dependencies

- A **captive dependency** occurs when a longer-lived service holds a reference to a shorter-lived service
- Example: a **Singleton** service holding a **Scoped** service — the scoped service will never be disposed and will become stale across requests
- This causes memory leaks, stale data, and thread-safety issues

```csharp
// BAD — Captive dependency! Singleton holds Scoped DbContext
public class BadBackgroundService : IHostedService
{
    private readonly AppDbContext _dbContext;

    public BadBackgroundService(AppDbContext dbContext) // Scoped injected into Singleton
    {
        _dbContext = dbContext; // this DbContext will NEVER be disposed
    }                           // and will accumulate stale data across requests

    public Task StartAsync(CancellationToken ct) => Task.CompletedTask;
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}

// GOOD — Resolve scoped services within a scope
public class GoodBackgroundService : IHostedService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public GoodBackgroundService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task StartAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // use dbContext here — it will be disposed when scope is disposed
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

- Enable **scope validation** in development to detect captive dependencies at startup:

```csharp
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;       // detect captive dependencies
    options.ValidateOnBuild = true;      // validate at startup, not first resolution
});
```

---

### Why DbContext Should Be Scoped, Not Singleton

- `DbContext` is **NOT thread-safe** — multiple threads (concurrent requests) sharing a single instance causes data corruption
- `DbContext` tracks entity changes — a singleton would accumulate an ever-growing change tracker, causing memory leaks and stale data
- Each HTTP request should have its own `DbContext` to ensure request isolation and proper change tracking
- Scoped lifetime ensures proper disposal — `Dispose()` is called at the end of each request, releasing the database connection
- If you make it singleton, the same `DbContext` instance serves ALL requests simultaneously — this will break under any real concurrency

---

### Constructor Injection vs Method Injection vs Property Injection

#### Constructor Injection (Preferred)

- Dependencies are passed through the class constructor
- The DI container automatically resolves all constructor parameters
- Makes dependencies **explicit and required** — you cannot create the object without providing them
- Enables immutability (readonly fields)
- Easiest to test — just pass mocks in the constructor
- **This is the standard and recommended approach in ASP.NET Core**

```csharp
// Constructor injection — preferred
public class OrderService : IOrderService
{
    private readonly IOrderRepository _repo;
    private readonly IEmailSender _emailSender;

    public OrderService(IOrderRepository repo, IEmailSender emailSender)
    {
        _repo = repo;
        _emailSender = emailSender;
    }
}
```

#### Method Injection

- Dependencies are passed as method parameters, not through the constructor
- Used with the `[FromServices]` attribute in MVC actions or Razor Pages
- Useful when a dependency is only needed for a specific action, not the entire controller lifetime

```csharp
// Method injection — using [FromServices]
[ApiController]
[Route("api/[controller]")]
public class OrderController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(
        OrderDto dto,
        [FromServices] IOrderService orderService,    // injected per-action
        [FromServices] IEmailSender emailSender)       // injected per-action
    {
        var order = orderService.Create(dto);
        emailSender.Send(order.Email, "Order created!");
        return Ok(order);
    }
}
```

#### Property Injection

- Dependencies are set via public properties after construction
- **NOT supported by the built-in ASP.NET Core DI container**
- Available in third-party containers like Autofac
- Generally discouraged because dependencies are implicit and the object can exist in an incomplete state

---

### How to Resolve Services

- **Constructor injection** is always preferred — the framework handles it automatically for controllers, middleware, Razor Pages, etc.
- When you need manual resolution (e.g., inside a factory method or background service), use `IServiceProvider.GetService<T>()` or `GetRequiredService<T>()`
- `GetService<T>()` returns `null` if the service is not registered — use it when the service is optional
- `GetRequiredService<T>()` throws an `InvalidOperationException` if not registered — use it when the service is required
- Avoid the **Service Locator anti-pattern** — injecting `IServiceProvider` and resolving services manually throughout your code defeats the purpose of DI

```csharp
// Manual resolution — only when constructor injection is not possible
public class MyBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public MyBackgroundService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
            // use repo...
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

---

## 2. Middleware Pipeline

### What is Middleware?

- Middleware is a component that sits in the HTTP request/response pipeline and can inspect, modify, or short-circuit incoming requests and outgoing responses
- Each middleware component has two responsibilities: **process the request** and **call the next middleware** in the pipeline (or stop it)
- Think of it as a chain of handlers, each one passing the request to the next until one handles it and the response flows back

---

### How the Pipeline Works (Russian Doll Model)

- The pipeline follows a **Russian doll (matryoshka) model** — each middleware wraps the next
- When a request arrives, it goes through middleware 1 → middleware 2 → middleware 3 → ... → endpoint
- The response then flows back: endpoint → middleware 3 → middleware 2 → middleware 1
- Each middleware can short-circuit by NOT calling `next()` — the response then flows back without reaching deeper middleware
- The `next` delegate is a `RequestDelegate` that represents the rest of the pipeline

```
Request  →  [Exception]  →  [HSTS]  →  [HTTPS]  →  [Static]  →  [Routing]  →  [Auth]  →  [Endpoint]
Response ←  [Exception]  ←  [HSTS]  ←  [HTTPS]  ←  [Static]  ←  [Routing]  ←  [Auth]  ←  [Endpoint]
```

---

### app.Use() — Adding Middleware That Can Call Next

- `app.Use()` adds inline middleware that receives `HttpContext` and a `next` delegate
- You MUST call `await next(context)` to pass the request to the next middleware, unless you intentionally want to short-circuit

```csharp
// app.Use() — inline middleware with next delegate
app.Use(async (context, next) =>
{
    // Before the next middleware
    var startTime = DateTime.UtcNow;
    context.Items["RequestStart"] = startTime;

    await next(context); // pass to next middleware

    // After the response comes back
    var elapsed = DateTime.UtcNow - startTime;
    Console.WriteLine($"Request {context.Request.Path} took {elapsed.TotalMilliseconds}ms");
});
```

---

### app.UseMiddleware<T>() — Class-based Middleware

- Register a middleware class that follows the middleware convention
- The class must have:
  - A constructor that accepts `RequestDelegate next` (and optionally other injected services)
  - An `InvokeAsync` or `Invoke` method that accepts `HttpContext` and returns `Task`
- Dependencies can be injected through the constructor (created once per app lifetime for singleton middleware, or per scope for scoped middleware)

```csharp
// Custom middleware class
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    // Constructor — receives next delegate and injected services
    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    // InvokeAsync — called for each request
    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();

        await _next(context); // call next middleware

        sw.Stop();
        _logger.LogInformation("Request {Path} completed in {Elapsed}ms",
            context.Request.Path, sw.ElapsedMilliseconds);
    }
}

// Registration
app.UseMiddleware<RequestTimingMiddleware>();
```

---

### app.Run() — Terminal Middleware

- `app.Run()` adds a terminal middleware — it NEVER calls `next()` and always produces the response
- Anything added after `app.Run()` will never execute
- Useful for catch-all handlers or fallback endpoints

```csharp
// app.Run() — terminal, never calls next
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello from terminal middleware!");
});

// Anything after Run() is NEVER reached
app.Run(async context =>  // THIS WILL NEVER EXECUTE
{
    await context.Response.WriteAsync("This is never called");
});
```

---

### app.Map() — Branching the Pipeline

- `app.Map()` creates a branch in the pipeline based on the request path
- Requests matching the given path prefix are routed to the branch; others continue down the main pipeline
- The branch has its own mini-pipeline and does NOT return to the main pipeline

```csharp
// app.Map() — branch based on path prefix
app.Map("/api/health", healthApp =>
{
    healthApp.Run(async context =>
    {
        await context.Response.WriteAsJsonAsync(new { Status = "Healthy" });
    });
});

app.Map("/api/orders", orderApp =>
{
    orderApp.UseMiddleware<AuthorizeMiddleware>();
    orderApp.MapControllers();
});

// Requests to /api/orders/... hit the order branch
// Requests to anything else continue down the main pipeline
```

---

### app.MapWhen() — Conditional Branching

- `app.MapWhen()` creates a branch based on an arbitrary condition (not just path prefix)
- The predicate receives `HttpContext` and returns `bool`
- Useful for branching based on headers, query strings, cookies, etc.

```csharp
// app.MapWhen() — branch based on custom condition
app.MapWhen(context => context.Request.Query.ContainsKey("debug"), debugApp =>
{
    debugApp.Use(async (ctx, next) =>
    {
        await ctx.Response.WriteAsync("Debug mode active\n");
        await next(ctx);
    });
});

// Branch based on request header
app.MapWhen(context => context.Request.Headers.ContainsKey("X-Custom-Header"), customApp =>
{
    customApp.Run(async ctx =>
    {
        await ctx.Response.WriteAsync("Custom header detected!");
    });
});
```

---

### Order of Middleware Matters

- The order in which middleware is registered in `Program.cs` defines the execution order
- Incorrect order leads to unexpected behavior — for example, routing before authentication means unauthenticated users can reach endpoints
- **Standard recommended order:**

```csharp
var app = builder.Build();

// 1. Exception handling — FIRST, so it catches errors from everything downstream
app.UseExceptionHandler("/error");

// 2. HSTS — security headers for production
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

// 3. HTTPS Redirection — force HTTPS before anything else processes the request
app.UseHttpsRedirection();

// 4. Static Files — serve files directly without going through the rest of the pipeline
app.UseStaticFiles();

// 5. Routing — match the request to an endpoint
app.UseRouting();

// 6. CORS — must come after UseRouting and before UseAuthentication/UseAuthorization
app.UseCors("MyPolicy");

// 7. Authentication — identify WHO the user is
app.UseAuthentication();

// 8. Authorization — determine WHAT the user is allowed to do
app.UseAuthorization();

// 9. Rate Limiting (.NET 7+) — before endpoint execution
app.UseRateLimiter();

// 10. Endpoints — execute the matched endpoint
app.MapControllers();
```

---

### Custom Middleware Class Pattern

- Every custom middleware must follow this exact pattern to work with `UseMiddleware<T>()`:

```csharp
// The full pattern for a custom middleware class
public class CustomMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IService _someService;

    // 1. Constructor — MUST accept RequestDelegate as first parameter
    //    Additional services are injected by the DI container
    public CustomMiddleware(RequestDelegate next, IService someService)
    {
        _next = next;
        _someService = someService;
    }

    // 2. InvokeAsync — MUST be public, accept HttpContext, return Task
    //    DO NOT put constructor logic here — this runs per request
    public async Task InvokeAsync(HttpContext context)
    {
        // --- Before next middleware ---

        // Read request
        var body = await new StreamReader(context.Request.Body).ReadToEndAsync();

        // Modify request
        context.Request.Headers["X-Custom"] = "value";

        // Call next middleware in the pipeline
        await _next(context);

        // --- After response comes back ---

        // Read/modify response
        var responseBody = await new StreamReader(context.Response.Body).ReadToEndAsync();
        Console.WriteLine($"Response: {responseBody}");
    }
}

// Register it
app.UseMiddleware<CustomMiddleware>();
```

---

### Inline Middleware (app.Use with Delegate)

- For simple, one-off middleware that doesn't need a full class
- Use `app.Use()` with a lambda — it receives `HttpContext` and `Func<Task> next`
- Good for logging, headers, simple branching logic

```csharp
// Inline middleware — quick and simple
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("X-Custom-Header", "MyValue");
    await next();
});

// Inline middleware — short-circuit example (does NOT call next)
app.Use(async (context, next) =>
{
    if (!context.Request.Headers.ContainsKey("Authorization"))
    {
        context.Response.StatusCode = 401;
        await context.Response.WriteAsync("Unauthorized");
        return; // short-circuit — next middleware is never called
    }
    await next();
});
```

---

## 3. Common Middleware

### ExceptionHandler Middleware

- Catches unhandled exceptions from downstream middleware and endpoints
- Must be registered FIRST (or very early) in the pipeline to catch errors from all other middleware
- In development, `UseDeveloperExceptionPage()` shows a detailed error page
- In production, `UseExceptionHandler("/error")` redirects to an error endpoint

```csharp
// Development — detailed exception page
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

// Production — redirect to error controller
app.UseExceptionHandler("/error");

// Error controller
[ApiController]
public class ErrorController : ControllerBase
{
    [Route("/error")]
    public IActionResult HandleError()
    {
        var exception = HttpContext.Features.Get<IExceptionHandlerFeature>()?.Error;
        return Problem(
            detail: exception?.Message,
            statusCode: 500,
            title: "An error occurred"
        );
    }
}
```

---

### HSTS Middleware

- Adds the `Strict-Transport-Security` header, telling browsers to only use HTTPS
- Should only be used in **production** — development browsers don't respect HSTS for localhost
- Must come early in the pipeline, before HTTPS redirection

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}
```

---

### HTTPS Redirection Middleware

- Redirects all HTTP requests to HTTPS (307 redirect)
- Must come early, after HSTS but before routing and other middleware
- Configured with `UseHttpsRedirection()` or a custom redirect status code

```csharp
app.UseHttpsRedirection(); // redirects HTTP → HTTPS
```

---

### Static Files Middleware

- Serves static files (HTML, CSS, JS, images) from `wwwroot/` without hitting the rest of the pipeline
- Must come early — if it's after routing, static files would be routed to controllers unnecessarily
- Falls through to the next middleware if no matching static file is found

```csharp
app.UseStaticFiles(); // serves from wwwroot/
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(Path.Combine(env.ContentRootPath, "CustomStatic")),
    RequestPath = "/static"
});
```

---

### Routing Middleware

- Matches incoming requests to endpoints (controllers, minimal API, Razor Pages)
- Must come BEFORE authentication, authorization, and endpoints
- `UseRouting()` is actually implicit in .NET 6+ minimal hosting — but it's good practice to be explicit

```csharp
app.UseRouting();

// Endpoints are mapped after UseRouting()
app.MapControllers();
app.MapGet("/hello", () => "Hello World!");
```

---

### CORS Middleware

- Handles Cross-Origin Resource Sharing headers
- Must come AFTER `UseRouting()` and BEFORE `UseAuthentication()`/`UseAuthorization()`
- Without proper order, CORS preflight requests may be rejected

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("MyPolicy", policy =>
    {
        policy.WithOrigins("https://example.com")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

app.UseCors("MyPolicy"); // Must be after UseRouting, before UseAuthorization
```

---

### Authentication Middleware

- Identifies WHO the user is (reads the token/cookie and sets `HttpContext.User`)
- Must come AFTER routing and CORS, but BEFORE authorization
- If you put auth before routing, it runs for every request including static files

```csharp
app.UseAuthentication(); // identifies the user
app.UseAuthorization();  // checks permissions
```

---

### Authorization Middleware

- Determines WHAT the authenticated user is allowed to do
- Checks `[Authorize]` attributes, policies, and roles
- Must come AFTER authentication

```csharp
app.UseAuthorization();
```

---

### Request/Response Logging Middleware

- Logs incoming requests and outgoing responses for debugging and monitoring
- Can log method, path, status code, headers, body, and timing
- Use `app.Use()` for simple logging or a custom middleware class for structured logging

```csharp
app.Use(async (context, next) =>
{
    var method = context.Request.Method;
    var path = context.Request.Path;
    var startTime = DateTime.UtcNow;

    Console.WriteLine($"[REQ] {method} {path}");

    await next();

    var elapsed = DateTime.UtcNow - startTime;
    Console.WriteLine($"[RES] {context.Response.StatusCode} {method} {path} ({elapsed.TotalMilliseconds:F1}ms)");
});
```

---

### Rate Limiting Middleware (.NET 7+)

- Limits the number of requests a client can make within a time window
- Built-in support for fixed window, sliding window, token bucket, and concurrency limiters
- Comes AFTER routing but BEFORE authorization (so it applies to endpoint execution)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.PermitLimit = 10;           // 10 requests
        opt.Window = TimeSpan.FromSeconds(60); // per 60 seconds
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 2;
    });
});

app.UseRateLimiter();

app.MapGet("/api/data", () => "data")
    .RequireRateLimiting("fixed");
```

---

## 4. Common Mistakes

### Wrong Middleware Order

- Placing `UseAuthentication()` before `UseRouting()` causes auth to run before the route is determined — static files and health checks may be rejected
- Placing `UseExceptionHandler()` after `UseRouting()` means routing errors won't be caught
- Placing `UseCors()` after `UseAuthorization()` means CORS preflight requests get rejected
- **Always follow the standard order: Exception → HSTS → HTTPS → Static → Routing → CORS → Auth → Authorization → Rate Limiting → Endpoints**

### Captive Dependency

- A singleton service injecting a scoped service (e.g., `AppDbContext`) creates a captive dependency
- The scoped service is never disposed, accumulates stale state, and causes data corruption under concurrency
- **Fix:** use `IServiceScopeFactory` to create a scope when you need scoped services from a singleton
- **Prevention:** enable scope validation: `options.ValidateScopes = true`

### Not Calling next() in Middleware

- Forgetting to call `await next(context)` causes the pipeline to stop — downstream middleware and endpoints never execute
- This is called **short-circuiting** — it's sometimes intentional (e.g., auth rejection) but usually a bug
- **Always make sure your middleware calls `next()` unless you deliberately want to stop the pipeline**

### Doing Heavy Work in Middleware

- Middleware runs for **every request** that passes through it
- Doing expensive operations (complex queries, file I/O, external API calls) in middleware blocks ALL requests and degrades performance
- Keep middleware lightweight — logging, header manipulation, simple checks
- Heavy work should be in services, background workers, or endpoint-specific logic

### Not Using IDisposable in Middleware

- If your middleware holds unmanaged resources or disposable dependencies, implement `IDisposable`
- The middleware instance lives for the app's lifetime (singleton) — resources held by it won't be cleaned up unless explicitly disposed
- In practice, most middleware should not hold resources — use DI lifetimes correctly instead

---

## 5. Interview Questions

### Q1: What is the difference between Singleton, Scoped, and Transient service lifetimes?

- **Singleton** creates one instance for the entire app lifetime. **Scoped** creates one instance per HTTP request. **Transient** creates a new instance every time it is resolved.
- Singleton is thread-safe by nature of having one instance — the service itself must be thread-safe. Scoped services are safe within a single request. Transient services are always safe since each resolution is independent.
- Choose Singleton for configuration, caching, and lightweight stateless services. Choose Scoped for data access (DbContext). Choose Transient for lightweight utilities and validators.

### Q2: What is a captive dependency and how do you prevent it?

- A captive dependency is when a longer-lived service holds a reference to a shorter-lived service (e.g., Singleton holding Scoped)
- Prevent it by using `IServiceScopeFactory` to create scopes in singletons, or by changing the shorter-lived service's lifetime to match or exceed the longer-lived one
- Enable `ValidateScopes = true` on the DI container to detect captive dependencies at startup

### Q3: Why should DbContext be Scoped and not Singleton?

- DbContext is not thread-safe — multiple concurrent requests sharing one instance causes data corruption
- DbContext tracks entity changes — a singleton accumulates an ever-growing change tracker, causing memory leaks and stale data
- Each request should work with its own isolated set of tracked entities, which Scoped provides

### Q4: What is the order of middleware in ASP.NET Core and why does it matter?

- Standard order: ExceptionHandler → HSTS → HTTPS → Static Files → Routing → CORS → Authentication → Authorization → Rate Limiting → Endpoints
- Order matters because each middleware wraps the next — an exception handler must be first to catch errors from all downstream middleware
- Authentication must come before Authorization because you need to identify the user before checking permissions
- CORS must come after Routing and before Authorization so preflight requests aren't rejected

### Q5: How does the middleware pipeline work?

- It follows a Russian doll model — each middleware calls `next()` to pass the request to the next component
- A middleware can short-circuit by not calling `next()` — the response then flows back without reaching deeper middleware
- Each middleware can modify the request before passing it on, and modify the response after it comes back

### Q6: What is the difference between app.Use(), app.Run(), and app.Map()?

- **app.Use()** adds middleware that receives the next delegate and can call it to continue the pipeline
- **app.Run()** adds terminal middleware that never calls next — the pipeline stops there
- **app.Map()** branches the pipeline based on request path — matching requests go into a separate sub-pipeline

### Q7: What are keyed services and when would you use them?

- Keyed services (.NET 8+) let you register multiple implementations of the same interface, each identified by a string key
- You resolve a specific implementation using `[FromKeyedServices("key")]` or `GetKeyedService<T>()`
- Use when you have multiple strategies (e.g., multiple email providers, multiple cache backends) and want to choose at runtime

### Q8: What is constructor injection and why is it preferred?

- Dependencies are passed through the class constructor by the DI container
- It makes dependencies explicit and required — the class cannot be instantiated without them
- It enables easy unit testing — just pass mocks in the constructor
- It supports immutability through readonly fields
- Method injection and property injection are alternatives but are less explicit and harder to test

### Q9: How do you register an open generic in the DI container?

- Use `services.AddScoped(typeof(IRepository<>), typeof(Repository<>))` — the container resolves the concrete type based on the generic argument at resolution time
- When you inject `IRepository<Order>`, the container creates `Repository<Order>`
- This avoids having to register every possible generic type combination manually

### Q10: What is the difference between IServiceCollection, IServiceProvider, and IServiceScope?

- **IServiceCollection** is used during configuration to register services (addTransient, addScoped, addSingleton)
- **IServiceServiceProvider** is the runtime service locator that resolves registered services
- **IServiceScope** creates a bounded lifetime for scoped services — all services within a scope share instances, and disposal cleans them up

### Q11: How do you resolve services manually when constructor injection is not available?

- Use `IServiceScopeFactory.CreateScope()` to create a scope, then `scope.ServiceProvider.GetRequiredService<T>()` to resolve
- This is common in background services (BackgroundService, IHostedService) where DI is not available
- Always dispose the scope when done to clean up scoped services

### Q12: What happens if you forget to call next() in middleware?

- The pipeline short-circuits — all downstream middleware and endpoints are skipped
- The response flows back through the current middleware and any middleware above it in the pipeline
- This is sometimes intentional (e.g., returning 401 without reaching the endpoint) but is usually a bug

### Q13: What is the difference between middleware and filters in ASP.NET Core?

- **Middleware** operates at the HTTP pipeline level — it sees every request before routing, including static files and non-MVC requests
- **Filters** (action filters, result filters, exception filters) operate at the MVC/Razor Pages level — they only run for matched endpoints
- Use middleware for cross-cutting concerns that apply to ALL requests (logging, headers, auth). Use filters for endpoint-specific concerns (model validation, caching, logging specific to controllers)

### Q14: How do you create custom middleware in ASP.NET Core?

- Create a class with a constructor that accepts `RequestDelegate next` and optional injected services
- Add an `InvokeAsync(HttpContext context)` method that calls `await _next(context)` to continue the pipeline
- Register with `app.UseMiddleware<YourMiddleware>()`
- The middleware class must NOT have constructor logic that runs per-request — put per-request logic in `InvokeAsync`

### Q15: What is the service locator anti-pattern and why should you avoid it?

- Service locator is when you inject `IServiceProvider` and manually call `GetService<T>()` throughout your code instead of using constructor injection
- It hides dependencies — you can't see what a class needs just by looking at its constructor
- It makes testing harder — you need to set up the entire container instead of passing mocks
- It defeats the purpose of DI — the container should manage dependencies, not your business logic

### Q16: What is the difference between GetService<T>() and GetRequiredService<T>()?

- `GetService<T>()` returns `null` if the service is not registered — use for optional dependencies
- `GetRequiredService<T>()` throws `InvalidOperationException` if the service is not registered — use for required dependencies
- Always prefer `GetRequiredService<T>()` unless the dependency is truly optional, to fail fast on misconfiguration

### Q17: How does rate limiting middleware work and when would you use it?

- Rate limiting middleware (.NET 7+) restricts how many requests a client can make within a given time window
- Supports fixed window, sliding window, token bucket, and concurrency limiters
- Use it to protect APIs from abuse, ensure fair usage, and prevent overload
- Register with `AddRateLimiter()` and apply with `.RequireRateLimiting("policyName")`

### Q18: When would you use app.MapWhen() instead of app.Map()?

- `app.Map()` branches based on request path prefix only
- `app.MapWhen()` branches based on an arbitrary predicate — headers, query strings, cookies, etc.
- Use `MapWhen()` when path-based branching is not sufficient, like "branch if the request has an X-Debug header"

### Q19: What does UseStaticFiles() do and why does it need to come early?

- It serves files from `wwwroot/` (CSS, JS, images) directly without going through routing or controllers
- If placed after routing, every static file request would go through the full pipeline unnecessarily
- It falls through to the next middleware if no matching static file is found, so it doesn't break other requests

### Q20: How do you inject services into middleware registered with UseMiddleware<T>()?

- The middleware's constructor can accept any registered service — the DI container resolves them automatically
- Services injected into the middleware constructor have the middleware's lifetime (singleton by default for UseMiddleware)
- If you need scoped services, resolve them from `IServiceScopeFactory` in the `InvokeAsync` method

---

## Quick Reference

| Lifetime   | Instances | Disposal           | Thread-Safe? | Use For                          |
|------------|-----------|--------------------|--------------|----------------------------------|
| Singleton  | 1 forever | App shutdown       | Must be      | Config, Cache, Redis client      |
| Scoped     | 1/request | End of request     | Yes (per req)| DbContext, Unit of Work           |
| Transient  | N/A       | GC / Scope dispose | Always       | Validators, lightweight services  |

| Middleware Method | Purpose                     | Calls Next? |
|------------------|-----------------------------|-------------|
| `app.Use()`      | Inline middleware           | Optional    |
| `app.UseMiddleware<T>()` | Class-based middleware | Optional    |
| `app.Run()`      | Terminal middleware          | Never       |
| `app.Map()`      | Branch by path              | N/A (branch)|
| `app.MapWhen()`  | Branch by condition         | N/A (branch)|
