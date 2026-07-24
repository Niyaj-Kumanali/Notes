# ASP.NET Core Routing, Model Binding, Validation & Filters

---

## Routing

### What is Routing

- Routing is the mechanism that maps incoming HTTP requests to specific endpoint handlers
- It determines which controller action (or middleware) should process the request based on the URL and HTTP method
- Routing does NOT look at physical file paths — it matches URL patterns against route templates
- Routing works with the endpoint routing system introduced in ASP.NET Core 3.0+
- Two main approaches: **Attribute Routing** and **Conventional Routing**
- Routing is configured in `Program.cs` (or `Startup.cs` in older versions)

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();

var app = builder.Build();
app.MapControllers(); // registers attribute-routed controllers
app.Run();
```

---

### Attribute Routing

- Attribute routing lets you define routes directly on controllers and actions using attributes
- It gives you precise, granular control over URL patterns
- Most commonly used in Web APIs
- You decorate controllers with `[Route]` and actions with `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]`, etc.

```csharp
[Route("api/[controller]")]
[ApiController]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(new[] { "Product1", "Product2" });
    }

    [HttpGet("{id}")]
    public IActionResult GetById(int id)
    {
        return Ok($"Product {id}");
    }

    [HttpPost]
    public IActionResult Create([FromBody] Product product)
    {
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    [HttpPut("{id}")]
    public IActionResult Update(int id, [FromBody] Product product)
    {
        return NoContent();
    }

    [HttpDelete("{id}")]
    public IActionResult Delete(int id)
    {
        return NoContent();
    }
}
```

- `[Route("api/[controller]")]` uses the `[controller]` token which is replaced by the controller name minus the "Controller" suffix
- So `ProductsController` becomes `api/Products`
- You can combine controller-level and action-level routes:

```csharp
[Route("api/[controller]")]
[ApiController]
public class OrdersController : ControllerBase
{
    // Matches: GET /api/Orders
    [HttpGet]
    public IActionResult GetAll() { ... }

    // Matches: GET /api/Orders/5
    [HttpGet("{id}")]
    public IActionResult GetById(int id) { ... }

    // Matches: GET /api/Orders/customer/5
    [HttpGet("customer/{customerId}")]
    public IActionResult GetByCustomer(int customerId) { ... }
}
```

---

### Conventional Routing

- Conventional routing defines route patterns in middleware configuration, not on controllers
- Convention-based routes apply to MVC controllers that do NOT have `[Route]` attributes
- Defined using `app.MapControllerRoute()`

```csharp
// Program.cs
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}"
);
```

- This pattern means:
  - `{controller=Home}` — defaults to `HomeController` if not specified
  - `{action=Index}` — defaults to `Index` action if not specified
  - `{id?}` — optional parameter
- Example URLs:
  - `/` → `HomeController.Index()`
  - `/Home` → `HomeController.Index()`
  - `/Home/About` → `HomeController.About()`
  - `/Home/Contact/5` → `HomeController.Contact(5)`
- You can define multiple conventional routes:

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}"
);

app.MapControllerRoute(
    name: "admin",
    pattern: "admin/{controller=Dashboard}/{action=Index}/{id?}"
);
```

- Convention-based routing is mostly used for MVC (views), not Web APIs
- Web APIs typically use attribute routing for more explicit control

---

### Route Parameters

- Route parameters are placeholders in the route template that capture values from the URL
- They are defined using curly braces `{parameterName}`
- The captured value is automatically bound to the corresponding action parameter

```csharp
[HttpGet("{id}")]
public IActionResult GetById(int id)
{
    // The {id} in the route template maps to the int id parameter
    return Ok(id);
}

[HttpGet("{category}/{productName}")]
public IActionResult GetByCategoryAndName(string category, string productName)
{
    return Ok($"{category}/{productName}");
}
```

- Route parameters are extracted from the URL segments matching their position in the template
- If a route parameter name matches an action parameter name, model binding automatically handles it

---

### Route Constraints

- Route constraints restrict what values a route parameter can accept
- They are added after the parameter name using a colon separator: `{parameter:constraint}`
- Constraints are checked during route matching — if the value doesn't match, the route doesn't match

```csharp
// Integer only
[HttpGet("{id:int}")]
public IActionResult GetById(int id) { ... }

// Guid only
[HttpGet("{id:guid}")]
public IActionResult GetById(Guid id) { ... }

// String with min/max length
[HttpGet("{name:minlength(3)}")]
public IActionResult GetByName(string name) { ... }

[HttpGet("{name:maxlength(50)}")]
public IActionResult GetByName(string name) { ... }

// Range constraint
[HttpGet("{page:range(1,100)}")]
public IActionResult GetPage(int page) { ... }

// Alpha (letters only)
[HttpGet("{code:alpha}")]
public IActionResult GetByCode(string code) { ... }

// Regex pattern
[HttpGet("{date:regex(^\\d{{4}}-\\d{{2}}-\\d{{2}}$)}")]
public IActionResult GetByDate(string date) { ... }

// Combined constraints
[HttpGet("{id:int:range(1,1000)}")]
public IActionResult GetById(int id) { ... }
```

#### Common Route Constraints Reference

| Constraint | Description | Example |
|------------|-------------|---------|
| `int` | Integer values | `{id:int}` |
| `string` | String values (default, non-empty) | `{name:string}` |
| `guid` | GUID values | `{id:guid}` |
| `bool` | Boolean values | `{active:bool}` |
| `datetime` | DateTime values | `{date:datetime}` |
| `decimal` | Decimal values | `{price:decimal}` |
| `double` | Double values | `{rate:double}` |
| `float` | Float values | `{score:float}` |
| `long` | Long values | `{bigId:long}` |
| `minlength(n)` | Minimum string length | `{name:minlength(3)}` |
| `maxlength(n)` | Maximum string length | `{name:maxlength(100)}` |
| `length(n)` | Exact string length | `{code:length(5)}` |
| `length(min,max)` | String length range | `{code:length(3,10)}` |
| `min(n)` | Minimum value (for int) | `{page:min(1)}` |
| `max(n)` | Maximum value (for int) | `{page:max(100)}` |
| `range(min,max)` | Value within range | `{page:range(1,100)}` |
| `alpha` | Alphabetic characters only | `{slug:alpha}` |
| `regex(expr)` | Matches a regular expression | `{slug:regex(^[a-z]+$)}` |
| `required` | Value must be present | `{name:required}` |

- Multiple constraints can be chained with colons:
  ```csharp
  [HttpGet("{id:int:range(1,999999)}")]
  ```

---

### Route Defaults

- Route defaults provide fallback values when a route segment is not present in the URL
- They are defined using the equals sign: `{parameter=default}`

```csharp
// Conventional routing with defaults
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}"
);

// Attribute routing defaults (less common — usually done via optional params)
[Route("api/[controller]")]
[ApiController]
public class HomeController : ControllerBase
{
    [HttpGet("{page?}")]
    public IActionResult GetPage(int page = 1)
    {
        return Ok($"Page {page}");
    }
}
```

- Defaults ensure that URLs can be shortened when the default value is acceptable
- `/Home` resolves to `HomeController.Index()` because of the defaults

---

### Optional Parameters with ?

- Optional parameters are marked with the `?` suffix in route templates
- If the segment is omitted from the URL, the parameter gets its default value (or CLR default)

```csharp
[HttpGet("{id?}")]
public IActionResult GetById(int? id)
{
    if (id == null)
        return Ok("No ID provided");
    return Ok($"ID: {id}");
}

[HttpGet("search/{term?}/{page?}")]
public IActionResult Search(string term, int? page)
{
    term ??= "all";
    page ??= 1;
    return Ok($"Search: {term}, Page: {page}");
}
```

- Without `?`, the route would not match if the segment is missing
- The `?` makes the route segment optional during matching
- In the action method, use nullable types (`int?`, `string?`) or provide default parameter values

---

### Catch-All Parameters with *

- Catch-all parameters capture the remaining URL segments (including slashes) into a single parameter
- They are defined using the `*` prefix: `{*paramName}`

```csharp
[HttpGet("files/{*path}")]
public IActionResult GetFile(string path)
{
    // URL: /files/documents/folder1/folder2/file.txt
    // path = "documents/folder1/folder2/file.txt"
    return Ok($"File path: {path}");
}

[HttpGet("api/{*wildcard}")]
public IActionResult CatchAll(string wildcard)
{
    // Catches everything after /api/
    return Ok($"Caught: {wildcard}");
}
```

- Catch-all parameters are useful for proxying, file serving, or flexible URL structures
- They are always optional by nature (an empty string matches if nothing follows)
- The captured value excludes the leading slash

---

### Route Templates vs Route Values

- **Route Templates** are the patterns defined in route attributes or conventional route configuration:
  ```csharp
  [HttpGet("api/products/{id:int}")]
  //              ^^^^^^^^^^^^^^^^^^^^^^^ this is the route template
  ```

- **Route Values** are the actual values extracted from the incoming URL at runtime:
  ```csharp
  // For URL: /api/products/42
  // Route values: { "id": "42", "controller": "Products", "action": "GetById" }
  ```

- Route values are stored in `HttpContext.Request.RouteValues` and can be accessed directly:
  ```csharp
  [HttpGet("{id}")]
    public IActionResult GetById()
    {
        var id = HttpContext.Request.RouteValues["id"];
        return Ok(id);
    }
  ```

- Template parameters become route values after the URL is matched against the template

---

### [ApiController] Attribute Behavior

- The `[ApiController]` attribute enables several automatic behaviors that simplify API development:

1. **Automatic model validation** — `ModelState.IsValid` is checked automatically; if invalid, a 400 Bad Request is returned without executing the action
2. **Infer binding sources** — parameters without explicit binding attributes get `[FromBody]` for complex types and `[FromRoute]`/`[FromQuery]` based on convention
3. **Automatic route attribute requirement** — actions must have an `[HttpGet]`, `[HttpPost]`, etc., or be in a controller with a `[Route]` attribute
4. **Problem Details response** — validation errors return `ProblemDetails` JSON by default

```csharp
[ApiController]
[Route("api/[controller]")]
public class CustomersController : ControllerBase
{
    // No need to check ModelState — it's automatic
    [HttpPost]
    public IActionResult Create(CreateCustomerRequest request)
    {
        // If request is invalid, 400 is returned before this line executes
        return Ok();
    }
}

public class CreateCustomerRequest
{
    [Required]
    public string Name { get; set; }

    [EmailAddress]
    public string Email { get; set; }
}
```

- Without `[ApiController]`, you must manually check `ModelState.IsValid`:
  ```csharp
  [HttpPost]
  public IActionResult Create(CreateCustomerRequest request)
  {
      if (!ModelState.IsValid)
          return BadRequest(ModelState);
      // ...
  }
  ```

---

### [Route] on Controller and HTTP Method Attributes on Actions

- `[Route]` on the controller defines the base route template
- `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]`, `[HttpPatch]`, `[HttpHead]`, `[HttpOptions]` on actions define the HTTP method and any additional route segments
- They combine: controller route + action route = full route

```csharp
[Route("api/v1/[controller]")]
[ApiController]
public class UsersController : ControllerBase
{
    // GET /api/v1/Users
    [HttpGet]
    public IActionResult GetAll() { ... }

    // GET /api/v1/Users/5
    [HttpGet("{id}")]
    public IActionResult GetById(int id) { ... }

    // GET /api/v1/Users/search/john
    [HttpGet("search/{term}")]
    public IActionResult Search(string term) { ... }

    // POST /api/v1/Users
    [HttpPost]
    public IActionResult Create([FromBody] User user) { ... }

    // PUT /api/v1/Users/5
    [HttpPut("{id}")]
    public IActionResult Update(int id, [FromBody] User user) { ... }

    // DELETE /api/v1/Users/5
    [HttpDelete("{id}")]
    public IActionResult Delete(int id) { ... }
}
```

- You can use `[HttpMethod("template")]` as a generic alternative:
  ```csharp
  [HttpMethod("GET", "search/{term}")]
  public IActionResult Search(string term) { ... }
  ```

- Multiple HTTP methods can be applied to the same action:
  ```csharp
  [HttpGet("{id}")]
  [HttpHead("{id}")]
  public IActionResult GetById(int id) { ... }
  ```

---

## Model Binding

### What is Model Binding

- Model binding is the process of mapping data from an HTTP request (route values, query string, form fields, request body) to action method parameters
- It happens automatically before the action method executes
- It converts string-based HTTP data into .NET types (int, bool, complex objects, collections, etc.)

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id) // 'id' is model-bound from route
{
    return Ok(id);
}

[HttpPost]
public IActionResult Create(Order order) // 'order' is model-bound from JSON body
{
    return Ok(order);
}
```

---

### Binding Sources

#### Route Values
- Data from URL segments matching route parameters
- Bound automatically when parameter name matches a route parameter

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id) // bound from route {id}
{
    return Ok(id);
}
```

#### Query String
- Data from the `?key=value` portion of the URL
- Default binding source for simple types when not in route

```csharp
// GET /api/products?page=1&sort=name
[HttpGet]
public IActionResult GetAll(int page, string sort)
{
    // page = 1, sort = "name"
    return Ok(new { page, sort });
}
```

#### Form Data
- Data from `application/x-www-form-urlencoded` or `multipart/form-data` requests
- Common for HTML form submissions

```csharp
[HttpPost]
public IActionResult Create([FromForm] RegisterModel model)
{
    // model is populated from form fields
    return Ok(model);
}
```

#### Request Body
- Data from the request body, typically JSON or XML
- Default binding source for complex types with `[ApiController]`

```csharp
[HttpPost]
public IActionResult Create([FromBody] Product product)
{
    // product is deserialized from JSON body
    return Ok(product);
}
```

---

### Binding Source Attributes

#### [FromBody]

- Binds the parameter from the request body (JSON, XML, etc.)
- Uses input formatters configured in the application (typically `System.Text.Json`)
- Only one parameter can be bound from the body per action

```csharp
[HttpPost]
public IActionResult Create([FromBody] CreateProductRequest request)
{
    // JSON body: { "name": "Widget", "price": 9.99 }
    return Ok(request);
}
```

- The default content type for `[FromBody]` is `application/json`
- For XML, configure XML formatters and send `Content-Type: application/xml`

#### [FromQuery]

- Binds the parameter explicitly from query string parameters
- Overrides the default binding source inference

```csharp
[HttpGet]
public IActionResult Search([FromQuery] string term, [FromQuery] int page)
{
    // GET /api/search?term=hello&page=2
    return Ok(new { term, page });
}
```

#### [FromRoute]

- Binds the parameter explicitly from route values
- Useful when you need to override the default binding source

```csharp
[HttpGet("{id}")]
public IActionResult Get([FromRoute] int id)
{
    return Ok(id);
}
```

#### [FromForm]

- Binds the parameter from form data (`application/x-www-form-urlencoded` or `multipart/form-data`)
- Useful for file uploads and form submissions

```csharp
[HttpPost("upload")]
public IActionResult Upload([FromForm] IFormFile file, [FromForm] string description)
{
    // Handles multipart form data with file upload
    return Ok(new { file.FileName, description });
}
```

#### [FromServices]

- Binds the parameter from the dependency injection container
- The parameter type must be registered in the DI container
- The parameter is NOT treated as a model binding source

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id, [FromServices] ILogger<ProductsController> logger)
{
    logger.LogInformation("Getting product {Id}", id);
    return Ok(id);
}
```

- This avoids the need to inject services via the constructor (though constructor injection is still preferred)

#### [FromHeader]

- Binds the parameter from a specific HTTP request header

```csharp
[HttpGet]
public IActionResult Get([FromHeader(Name = "X-Request-Id")] string requestId)
{
    return Ok($"Request: {requestId}");
}
```

- You can bind multiple headers:
  ```csharp
  [HttpGet]
  public IActionResult Get(
      [FromHeader(Name = "X-Tenant-Id")] string tenantId,
      [FromHeader(Name = "Accept-Language")] string language)
  {
      return Ok(new { tenantId, language });
  }
  ```

---

### Complex Type Binding

- When an action parameter is a complex type (a class with properties), ASP.NET Core binds values from multiple sources by default
- With `[ApiController]`, complex types default to `[FromBody]` (JSON body)
- Without `[ApiController]`, the framework attempts to bind each property from the best matching source

```csharp
public class SearchFilter
{
    public string Term { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public string SortBy { get; set; }
}

// With [ApiController] — expects JSON body by default
[HttpPost("search")]
public IActionResult Search([FromBody] SearchFilter filter)
{
    return Ok(filter);
}

// Without [ApiController] — binds from query string by default
[HttpGet("search")]
public IActionResult Search([FromQuery] SearchFilter filter)
{
    // GET /api/search?term=hello&page=1&pageSize=20&sortBy=name
    return Ok(filter);
}
```

- Each property can be bound independently from different sources without explicit attributes (when using `[FromQuery]` on complex types)

---

### Collection Binding

- Model binding supports binding to arrays, lists, and dictionaries
- Collections can come from query strings, route values, or request body

```csharp
// Query string: /api/items?ids=1&ids=2&ids=3
[HttpGet]
public IActionResult GetItems([FromQuery] List<int> ids)
{
    return Ok(ids);
}

// Query string: /api/filter?tags=csharp&tags=dotnet&tags=aspnet
[HttpGet]
public IActionResult Filter([FromQuery] string[] tags)
{
    return Ok(tags);
}

// JSON body array
[HttpPost]
public IActionResult CreateMany([FromBody] List<Product> products)
{
    return Ok(products);
}
```

- For dictionary binding from query strings:
  ```
  /api/data?dict[key1]=value1&dict[key2]=value2
  ```

---

## Model Validation

### [ApiController] Auto-Validation

- When `[ApiController]` is applied, ASP.NET Core automatically validates the model state before the action executes
- If `ModelState.IsValid` is `false`, a `400 Bad Request` response is returned immediately
- The action method is never called when validation fails
- The response body contains validation errors in `ProblemDetails` format

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateOrderRequest request)
    {
        // This line is only reached if request is valid
        return Ok();
    }
}

public class CreateOrderRequest
{
    [Required(ErrorMessage = "Product name is required")]
    [StringLength(100, MinimumLength = 3)]
    public string ProductName { get; set; }

    [Range(1, 1000, ErrorMessage = "Quantity must be between 1 and 1000")]
    public int Quantity { get; set; }

    [Required]
    [EmailAddress]
    public string CustomerEmail { get; set; }
}
```

- The automatic 400 response looks like:
  ```json
  {
    "type": "https://tools.ietf.org/html/rfc7807",
    "title": "One or more validation errors occurred.",
    "status": 400,
    "errors": {
      "ProductName": ["Product name is required"],
      "Quantity": ["Quantity must be between 1 and 1000"]
    }
  }
  ```

---

### ModelState.IsValid

- Without `[ApiController]`, you must check `ModelState.IsValid` manually
- `ModelState` contains all validation errors and metadata about the binding

```csharp
[HttpPost]
public IActionResult Create(CreateOrderRequest request)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }

    // Process valid request...
    return Ok();
}
```

- You can access specific errors:
  ```csharp
  if (!ModelState.ContainsKey("ProductName"))
  {
      // No error for ProductName
  }

  var errors = ModelState["ProductName"]?.Errors;
  foreach (var error in errors)
  {
      Console.WriteLine(error.ErrorMessage);
  }
  ```

---

### Custom Validation Attributes

- You can create custom validation attributes by inheriting from `ValidationAttribute`
- Override the `IsValid` method to implement custom validation logic
- Attributes are checked automatically during model validation

```csharp
public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(object value, ValidationContext context)
    {
        if (value is DateTime date)
        {
            if (date <= DateTime.UtcNow)
            {
                return new ValidationResult(ErrorMessage ?? "Date must be in the future");
            }
            return ValidationResult.Success;
        }

        return new ValidationResult("Invalid date value");
    }
}

// Usage
public class CreateEventRequest
{
    [Required]
    public string Name { get; set; }

    [FutureDate(ErrorMessage = "Event date must be in the future")]
    public DateTime EventDate { get; set; }
}
```

- Another example — cross-field validation:
  ```csharp
  public class DateRangeAttribute : ValidationAttribute
  {
      private readonly string _endDateProperty;

      public DateRangeAttribute(string endDateProperty)
      {
          _endDateProperty = endDateProperty;
      }

      protected override ValidationResult IsValid(object value, ValidationContext context)
      {
          var startDate = value as DateTime?;
          var endDateProp = context.ObjectType.GetProperty(_endDateProperty);
          var endDate = endDateProp?.GetValue(context.ObjectInstance) as DateTime?;

          if (startDate.HasValue && endDate.HasValue && startDate > endDate)
          {
              return new ValidationResult("Start date must be before end date");
          }

          return ValidationResult.Success;
      }
  }

  public class EventRequest
  {
      [DateRange(nameof(EndDate))]
      public DateTime StartDate { get; set; }

      public DateTime EndDate { get; set; }
  }
  ```

---

### IValidatableObject Interface

- `IValidatableObject` allows complex cross-field validation inside the model class itself
- Implement `Validate` to return a collection of `ValidationResult`

```csharp
public class CreateEventRequest : IValidatableObject
{
    [Required]
    public string Name { get; set; }

    public DateTime StartDate { get; set; }
    public DateTime EndDate { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (StartDate >= EndDate)
        {
            yield return new ValidationResult(
                "End date must be after start date",
                new[] { nameof(EndDate) });
        }

        if (EndDate - StartDate > TimeSpan.FromDays(30))
        {
            yield return new ValidationResult(
                "Event cannot last more than 30 days",
                new[] { nameof(EndDate) });
        }
    }
}
```

- `IValidatableObject` is evaluated after individual property validation attributes
- It's useful when validation depends on multiple properties together

---

### FluentValidation Library

- FluentValidation is a popular third-party library for building strongly-typed validation rules
- It separates validation logic from model classes
- Rules are defined in separate validator classes that implement `AbstractValidator<T>`

```csharp
// Install: FluentValidation.AspNetCore
// Program.cs
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Validator class
public class CreateProductValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .Length(3, 100).WithMessage("Name must be between 3 and 100 characters");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than 0")
            .LessThan(100000).WithMessage("Price must be less than 100,000");

        RuleFor(x => x.SKU)
            .Matches(@"^[A-Z]{3}-\d{4}$").WithMessage("SKU must be in format XXX-0000");

        RuleFor(x => x.EndDate)
            .GreaterThan(x => x.StartDate)
            .When(x => x.EndDate.HasValue)
            .WithMessage("End date must be after start date");
    }
}

// Model
public class CreateProductRequest
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string SKU { get; set; }
    public DateTime? StartDate { get; set; }
    public DateTime? EndDate { get; set; }
}
```

- FluentValidation integrates seamlessly with `[ApiController]` auto-validation
- It replaces data annotation validation entirely or works alongside it

---

### Returning 400 Bad Request

- With `[ApiController]`, 400 is returned automatically
- Without it, you return it manually:

```csharp
// Manual
if (!ModelState.IsValid)
{
    return BadRequest(ModelState);
}

// Or as ProblemDetails
return ValidationProblem(ModelState);

// Custom error response
if (!ModelState.IsValid)
{
    var errors = ModelState
        .Where(e => e.Value.Errors.Any())
        .ToDictionary(
            e => e.Key,
            e => e.Value.Errors.Select(err => err.ErrorMessage).ToArray());

    return BadRequest(new { errors });
}
```

---

### Custom Validation Error Responses

- You can customize the validation error response format globally

```csharp
// Program.cs
builder.Services.Configure<ApiBehaviorOptions>(options =>
{
    options.InvalidModelStateResponseFactory = context =>
    {
        var errors = context.ModelState
            .Where(e => e.Value.Errors.Any())
            .Select(e => new
            {
                Field = e.Key,
                Errors = e.Value.Errors.Select(err => err.ErrorMessage)
            });

        return new BadRequestObjectResult(new
        {
            Success = false,
            Message = "Validation failed",
            Errors = errors
        });
    };
});
```

---

## Filters

### What Are Filters

- Filters are components that run at specific points in the ASP.NET Core request pipeline
- They provide a way to add cross-cutting concerns (logging, caching, authorization, error handling) without modifying action methods
- Filters can short-circuit the pipeline (prevent the action from executing)
- They are similar to middleware but operate at the action level, not the request level

---

### Filter Execution Order

- Filters execute in a specific, predictable order:

1. **Authorization Filters** — Run first. Can short-circuit the entire request.
2. **Resource Filters** — Run after authorization. Can short-circuit. Good for caching.
3. **Action Filters** — Run before and after the action method. Cannot short-circuit the action, but can modify the result.
4. **Exception Filters** — Run when an unhandled exception occurs.
5. **Result Filters** — Run before and after the action result executes (e.g., before writing the response body).

```
Request → Authorization → Resource → [Action Method] → Action → Result → Response
                         ↑                              ↑                 ↑
                    (before action)              (after action)    (before/after result)
                                                          ↑
                                                    Exception (if thrown)
```

- The order matters: authorization runs first, so it can reject requests before any other filter runs

---

### Authorization Filters

- Run first in the pipeline
- Determine whether the request is authorized
- Can short-circuit the request by returning a `401 Unauthorized` or `403 Forbidden`
- The `[Authorize]` attribute is the most common authorization filter

```csharp
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class SecureController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok("You are authorized");
    }
}

[Authorize(Roles = "Admin")]
[ApiController]
[Route("api/[controller]")]
public class AdminController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok("Admin only");
    }
}

[Authorize(Policy = "MinimumAge21")]
[ApiController]
[Route("api/[controller]")]
public class AdultController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok("Age 21+ only");
    }
}
```

- Custom authorization filters implement `IAuthorizationFilter` or `IAsyncAuthorizationFilter`

---

### Resource Filters

- Run after authorization but before model binding and action execution
- Can short-circuit the pipeline by returning a cached result
- Common use cases: caching, logging, rate limiting

```csharp
public class MyResourceFilter : IResourceFilter
{
    public void OnResourceExecuting(ResourceExecutingContext context)
    {
        // Runs BEFORE model binding and action execution
        // Can inspect/modify the request
        // Can short-circuit by setting context.Result
    }

    public void OnResourceExecuted(ResourceExecutedContext context)
    {
        // Runs AFTER the action and result filters
        // Can inspect/modify the result
    }
}

// Usage as an attribute
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public class CacheAttribute : Attribute, IResourceFilter
{
    private readonly int _duration;

    public CacheAttribute(int duration)
    {
        _duration = duration;
    }

    public void OnResourceExecuting(ResourceExecutingContext context)
    {
        var cacheKey = context.HttpContext.Request.Path.ToString();
        var cached = MemoryCache.Get(cacheKey) as string;

        if (cached != null)
        {
            context.Result = new ContentResult
            {
                Content = cached,
                ContentType = "application/json"
            };
        }
    }

    public void OnResourceExecuted(ResourceExecutedContext context)
    {
        if (context.Result is ContentResult contentResult)
        {
            var cacheKey = context.HttpContext.Request.Path.ToString();
            MemoryCache.Set(cacheKey, contentResult.Content, _duration);
        }
    }
}

[HttpGet("{id}")]
[Cache(60)]
public IActionResult GetById(int id) { ... }
```

---

### Action Filters

- Run before and after the action method executes
- Cannot short-circuit the action (the action always executes), but can modify the input or output
- Common use cases: logging, input transformation, response modification

```csharp
public class LogActionFilter : IActionFilter
{
    private readonly ILogger<LogActionFilter> _logger;

    public LogActionFilter(ILogger<LogActionFilter> logger)
    {
        _logger = logger;
    }

    public void OnActionExecuting(ActionExecutingContext context)
    {
        // Runs BEFORE the action method
        _logger.LogInformation(
            "Executing action {Action} on {Controller}",
            context.ActionDescriptor.RouteValues["action"],
            context.ActionDescriptor.RouteValues["controller"]);

        // You can access action arguments
        foreach (var arg in context.ActionArguments)
        {
            _logger.LogInformation("Arg: {Key} = {Value}", arg.Key, arg.Value);
        }
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        // Runs AFTER the action method
        if (context.Exception != null)
        {
            _logger.LogError(context.Exception, "Action threw an exception");
        }
        else
        {
            _logger.LogInformation("Action executed successfully");
        }
    }
}
```

#### IActionFilter vs IAsyncActionFilter

- `IActionFilter` uses synchronous methods (`OnActionExecuting`, `OnActionExecuted`)
- `IAsyncActionFilter` uses a single async method (`OnActionExecutionAsync`)
- Use `IAsyncActionFilter` when you need to perform async operations (database calls, HTTP requests)

```csharp
public class AsyncActionFilter : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        // Runs BEFORE the action
        Console.WriteLine("Before action");

        // Call next() to execute the action and remaining filters
        var resultContext = await next();

        // Runs AFTER the action
        if (resultContext.Exception != null)
        {
            Console.WriteLine($"Exception: {resultContext.Exception.Message}");
        }
        else
        {
            Console.WriteLine("After action");
        }
    }
}
```

- **Important**: If you don't call `next()`, the action will NOT execute. This is how you can "short-circuit" from an action filter by setting `context.Result` before calling `next()`.

---

### Exception Filters

- Run when an unhandled exception occurs during action execution
- Do NOT have a "before" and "after" — they only run on exception
- Common use cases: global error handling, logging exceptions, returning custom error responses

```csharp
public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;

    public GlobalExceptionFilter(ILogger<GlobalExceptionFilter> logger)
    {
        _logger = logger;
    }

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Unhandled exception occurred");

        context.Result = new ObjectResult(new
        {
            Error = "An unexpected error occurred",
            Message = context.Exception.Message
        })
        {
            StatusCode = 500
        };

        context.ExceptionHandled = true; // Prevents other handlers from processing
    }
}
```

- Mark the exception as handled by setting `context.ExceptionHandled = true`
- If you don't set it, the default developer exception page or error handler may still run

---

### Result Filters

- Run before and after the action result is executed (e.g., before and after `ObjectResult` writes to the response)
- Common use cases: modifying the response, adding headers, performance measurement

```csharp
public class TimingResultFilter : IResultFilter
{
    private readonly ILogger<TimingResultFilter> _logger;
    private Stopwatch _stopwatch;

    public TimingResultFilter(ILogger<TimingResultFilter> logger)
    {
        _logger = logger;
    }

    public void OnResultExecuting(ResultExecutingContext context)
    {
        // Runs BEFORE the result is written to the response
        _stopwatch = Stopwatch.StartNew();

        // Add custom headers
        context.HttpContext.Response.Headers.Append("X-Custom-Header", "MyValue");
    }

    public void OnResultExecuted(ResultExecutedContext context)
    {
        // Runs AFTER the result is written to the response
        _stopwatch.Stop();
        _logger.LogInformation(
            "Result executed in {ElapsedMs}ms",
            _stopwatch.ElapsedMilliseconds);
    }
}
```

---

### Filter Factories and Attribute Filters

- Filters can be registered as attributes so they can be applied directly to controllers or actions
- The `ServiceFilter` and `TypeFilter` attributes instantiate filters and resolve their dependencies

```csharp
// Register the filter as a service
builder.Services.AddScoped<LogActionFilter>();

// Apply using [ServiceFilter]
[ServiceFilter(typeof(LogActionFilter))]
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() { ... }
}
```

- `[TypeFilter]` creates a new instance per request (like `AddTransient`):
  ```csharp
  [TypeFilter(typeof(LogActionFilter))]
  public IActionResult Get() { ... }
  ```

- `[ServiceFilter]` uses the registered DI service (must be registered with `AddScoped`/`AddSingleton`/`AddTransient`)

---

### ServiceFilter vs TypeFilter

| Feature | `[ServiceFilter]` | `[TypeFilter]` |
|---------|-------------------|----------------|
| DI Registration | Must be registered in DI container | Resolved through DI automatically |
| Lifetime | Follows registered lifetime | New instance per request (transient) |
| Use Case | Shared filter instances | Filter with dependencies not in DI |
| Performance | Slightly faster (pre-registered) | Slightly slower (creates each time) |

```csharp
// ServiceFilter — requires explicit DI registration
builder.Services.AddScoped<LogActionFilter>();

[ServiceFilter(typeof(LogActionFilter))]
public IActionResult Get() { ... }

// TypeFilter — resolves dependencies automatically
// No DI registration needed
[TypeFilter(typeof(LogActionFilter))]
public IActionResult Get() { ... }
```

---

### [FilterOrder] Attribute

- Controls the execution order of filters of the same type
- Lower numbers execute first
- Useful when multiple filters of the same type are applied

```csharp
[FilterOrder(1)]
public class FirstFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        Console.WriteLine("First filter - executing");
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        Console.WriteLine("First filter - executed");
    }
}

[FilterOrder(2)]
public class SecondFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        Console.WriteLine("Second filter - executing");
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        Console.WriteLine("Second filter - executed");
    }
}

// Apply both
[ServiceFilter(typeof(FirstFilter))]
[ServiceFilter(typeof(SecondFilter))]
[ApiController]
[Route("api/[controller]")]
public class OrderController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok();
    }
}

// Output:
// First filter - executing
// Second filter - executing
// (action runs)
// Second filter - executed
// First filter - executed
```

---

### Creating Custom Filters

- Custom filters are created by implementing the appropriate filter interface
- Common pattern: create an attribute class that implements the filter interface

```csharp
// Custom action filter as an attribute
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public class ValidateModelAttribute : Attribute, IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            context.Result = new BadRequestObjectResult(context.ModelState);
        }
    }

    public void OnActionExecuted(ActionExecutedContext context) { }
}

// Custom async exception filter
public class CustomExceptionFilter : IAsyncExceptionFilter
{
    public Task OnExceptionAsync(ExceptionContext context)
    {
        context.Result = new ObjectResult(new
        {
            Error = "Something went wrong",
            Details = context.Exception.Message
        })
        {
            StatusCode = 500
        };

        context.ExceptionHandled = true;
        return Task.CompletedTask;
    }
}

// Custom result filter as an attribute
[AttributeUsage(AttributeTargets.Method)]
public class AddCacheHeaderAttribute : Attribute, IResultFilter
{
    private readonly int _seconds;

    public AddCacheHeaderAttribute(int seconds)
    {
        _seconds = seconds;
    }

    public void OnResultExecuting(ResultExecutingContext context)
    {
        context.HttpContext.Response.Headers.Append(
            "Cache-Control",
            $"public, max-age={_seconds}");
    }

    public void OnResultExecuted(ResultExecutedContext context) { }
}
```

---

### Global vs Controller-Level vs Action-Level Filters

#### Global Filters
- Apply to ALL actions in the application
- Registered in `Program.cs`
- Execute for every request

```csharp
// Program.cs
builder.Services.AddScoped<LogActionFilter>();
builder.Services.AddScoped<GlobalExceptionFilter>();

var app = builder.Build();

app.Services.GetRequiredService<ILogger<Program>>()
    .LogInformation("Configuring global filters");

// Global filters
app.MapControllers().WithMetadata(
    typeof(LogActionFilter),
    typeof(GlobalExceptionFilter));
```

- Or using the filter collection:
  ```csharp
  builder.Services.AddControllers(options =>
  {
      options.Filters.Add<LogActionFilter>();         // Global
      options.Filters.Add<GlobalExceptionFilter>();   // Global
      options.Filters.Add(typeof(CustomExceptionFilter));
  });
  ```

#### Controller-Level Filters
- Apply to ALL actions within a specific controller
- Applied as attributes on the controller class

```csharp
[ServiceFilter(typeof(LogActionFilter))]
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // LogActionFilter applies to ALL actions in this controller
    [HttpGet]
    public IActionResult GetAll() { ... }

    [HttpGet("{id}")]
    public IActionResult GetById(int id) { ... }
}
```

#### Action-Level Filters
- Apply to a single action method
- Applied as attributes on the action method

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() { ... }

    [HttpGet("{id}")]
    [ServiceFilter(typeof(LogActionFilter))] // Only applies to this action
    public IActionResult GetById(int id) { ... }
}
```

#### Execution Order Summary

```
Global filters → Controller filters → Action filters
```

- Global filters execute first, then controller-level, then action-level
- This order applies to both "before" and "after" phases

---

## API Versioning

### What is API Versioning

- API versioning is the practice of supporting multiple versions of an API simultaneously
- It allows you to introduce breaking changes in new versions while keeping old versions working
- Clients can choose which version of the API to use
- Essential for public APIs where you cannot control all clients

---

### URL Versioning

- The version is embedded in the URL path
- Most common and most explicit approach

```csharp
// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.ReportApiVersions = true;
})
.AddApiVersioning();

// V1 Controller
[ApiVersion("1.0")]
[Route("api/v1/[controller]")]
[ApiController]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(new[] { new { Id = 1, Name = "Product V1" } });
    }
}

// V2 Controller
[ApiVersion("2.0")]
[Route("api/v2/[controller]")]
[ApiController]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(new[] { new { Id = 1, Name = "Product V2", Price = 9.99 } });
    }
}
```

- Clients call: `GET /api/v1/products` or `GET /api/v2/products`

---

### Query String Versioning

- The version is passed as a query string parameter
- Less explicit but simpler to implement

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});

// Single controller handles multiple versions
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/[controller]")]
[ApiController]
public class ProductsController : ControllerBase
{
    [HttpGet, MapToApiVersion("1.0")]
    public IActionResult GetAllV1()
    {
        return Ok(new[] { new { Id = 1, Name = "Product V1" } });
    }

    [HttpGet, MapToApiVersion("2.0")]
    public IActionResult GetAllV2()
    {
        return Ok(new[] { new { Id = 1, Name = "Product V2", Price = 9.99 } });
    }
}
```

- Clients call: `GET /api/products?api-version=2.0`

---

### Header Versioning

- The version is specified in a custom HTTP header

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ApiVersionReader = ApiVersionReader.FromHeader("X-Api-Version");
});

// Usage: add header X-Api-Version: 2.0
```

---

### Media Type Versioning

- The version is embedded in the `Accept` header using a custom media type

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ApiVersionReader = ApiVersionReader.FromMediaType(
        "application/vnd.myapp.product.v2+json");
});

// Client sends:
// Accept: application/vnd.myapp.product.v2+json
```

---

### Implementing with Asp.Versioning.Mvc

```csharp
// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true; // adds api-supported-versions header
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new QueryStringApiVersionReader("api-version"),
        new HeaderApiVersionReader("X-Api-Version"));
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// Controller with versioning
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class CustomersController : ControllerBase
{
    [HttpGet, MapToApiVersion("1.0")]
    public IActionResult GetV1()
    {
        return Ok(new[] { new { Id = 1, Name = "Customer V1" } });
    }

    [HttpGet, MapToApiVersion("2.0")]
    public IActionResult GetV2()
    {
        return Ok(new[] { new
        {
            Id = 1,
            Name = "Customer V2",
            Email = "customer@example.com"
        }});
    }
}

// Default version when no version specified
[ApiController]
[ApiVersionNeutral] // works for ALL versions
[Route("api/[controller]")]
public class HealthController : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok("Healthy");
}
```

---

## Common Mistakes

### Not Using [ApiController]

```csharp
// BAD: Manual ModelState checking required
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateProductRequest request)
    {
        if (!ModelState.IsValid) // Must check manually
            return BadRequest(ModelState);
        // ...
    }
}

// GOOD: Auto-validation with [ApiController]
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateProductRequest request)
    {
        // ModelState is automatically validated
        return Ok();
    }
}
```

---

### Wrong Filter Order

```csharp
// BAD: Putting logic that belongs in middleware as a filter
public class AuthenticationFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        // Authentication should be middleware, not an action filter
        // Use IAuthorizationFilter or middleware instead
    }
    public void OnActionExecuted(ActionExecutedContext context) { }
}

// BAD: Using exception filter for logging only
public class LoggingExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        // If this filter doesn't handle the exception,
        // it should be middleware or a different filter type
        Log.Error(context.Exception);
        // Forgot: context.ExceptionHandled = true;
    }
}
```

---

### Not Handling Model Validation Errors

```csharp
// BAD: Ignoring validation
[HttpPost]
public IActionResult Create([FromBody] CreateProductRequest request)
{
    // No validation check, no [ApiController], invalid data passes through
    var product = MapToProduct(request);
    _db.Products.Add(product);
    return Ok(product);
}

// GOOD: Proper validation handling
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateProductRequest request)
    {
        // Validation is automatic with [ApiController]
        var product = MapToProduct(request);
        _db.Products.Add(product);
        return Ok(product);
    }
}
```

---

### Using Action Filters When Middleware Would Be Better

- Middleware runs for every request in the pipeline (including non-MVC requests)
- Filters only run for MVC actions
- Use middleware for: authentication, CORS, static files, logging of all requests
- Use filters for: action-specific concerns, model validation, result modification

```csharp
// BAD: Logging all requests in an action filter
public class RequestLoggingFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        // Only runs for MVC actions, misses static files, health checks, etc.
        _logger.LogInformation("Request: {Path}", context.HttpContext.Request.Path);
    }
    public void OnActionExecuted(ActionExecutedContext context) { }
}

// GOOD: Use middleware for request logging
app.Use(async (context, next) =>
{
    _logger.LogInformation("Request: {Path}", context.Request.Path);
    await next();
});
```

---

### Over-Complicated Routing

```csharp
// BAD: Unnecessarily complex routes
[Route("api/v1/company/{companyId:guid}/department/{departmentId:int}/employees/{employeeId:long}")]
public IActionResult GetEmployee(Guid companyId, int departmentId, long employeeId) { ... }

// GOOD: Simpler, more RESTful routes
[Route("api/employees/{id}")]
public IActionResult GetEmployee(long id) { ... }

// BAD: Redundant route constraints in conventional routing
app.MapControllerRoute(
    name: "default",
    pattern: "{controller:regex(^Home$)}/{action:regex(^Index$)}/{id?}"
);

// GOOD: Use sensible defaults
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}"
);
```

---

## Interview Questions

### Q1: What is routing in ASP.NET Core and how does it differ from middleware?

**Answer:**
- Routing maps incoming URLs to endpoint handlers (controller actions)
- Middleware is a pipeline component that processes every request
- Routing is selective (matches specific URL patterns), middleware is global
- Routing determines WHICH handler processes the request; middleware determines HOW the request is processed
- Routing was decoupled from middleware in ASP.NET Core 3.0+ using endpoint routing

---

### Q2: What are the two main types of routing in ASP.NET Core?

**Answer:**
- **Attribute Routing**: Routes are defined on controllers/actions using `[Route]`, `[HttpGet]`, etc. Most common for Web APIs. Gives granular control.
- **Conventional Routing**: Routes are defined in middleware configuration using patterns like `{controller=Home}/{action=Index}/{id?}`. Used for MVC views. Less explicit.

---

### Q3: What does the [ApiController] attribute do?

**Answer:**
- Automatic model validation (returns 400 if `ModelState` is invalid)
- Automatic binding source inference (`[FromBody]` for complex types, `[FromRoute]` for route params)
- Requires route attributes on actions (or a `[Route]` on the controller)
- Returns `ProblemDetails` format for errors
- Eliminates the need to manually check `ModelState.IsValid`

---

### Q4: What are route constraints and give some examples?

**Answer:**
- Route constraints restrict what values a route parameter can accept
- Examples: `{id:int}`, `{id:guid}`, `{name:minlength(3)}`, `{page:range(1,100)}`, `{slug:alpha}`, `{date:regex(^\\d{4}-\\d{2}-\\d{2}$)}`
- Constraints are checked during route matching; non-matching routes are skipped
- Multiple constraints can be chained: `{id:int:range(1,100)}`

---

### Q5: What is the difference between [FromBody], [FromQuery], [FromRoute], and [FromForm]?

**Answer:**
- `[FromBody]`: Binds from request body (JSON/XML). Used for complex types in POST/PUT.
- `[FromQuery]`: Binds from query string parameters (`?key=value`).
- `[FromRoute]`: Binds from route values (URL segments).
- `[FromForm]`: Binds from form data (`multipart/form-data` or `application/x-www-form-urlencoded`).
- Each explicitly tells the framework where to look for the value.

---

### Q6: What are the five types of filters and in what order do they execute?

**Answer:**
1. **Authorization Filters** — Run first, determine if request is authorized, can short-circuit
2. **Resource Filters** — Run after auth, before action (caching, logging)
3. **Action Filters** — Run before/after action method (logging, input transformation)
4. **Exception Filters** — Run when unhandled exception occurs (global error handling)
5. **Result Filters** — Run before/after result execution (response modification, headers)

---

### Q7: How do you create a custom filter in ASP.NET Core?

**Answer:**
- Implement the appropriate filter interface (`IActionFilter`, `IExceptionFilter`, `IResultFilter`, etc.)
- Optionally create an attribute class that implements the interface
- Register it as a service or use `[ServiceFilter]`/`[TypeFilter]`
- For async operations, implement `IAsyncActionFilter` or `IAsyncExceptionFilter`

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class MyCustomFilter : Attribute, IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context) { }
    public void OnActionExecuted(ActionExecutedContext context) { }
}
```

---

### Q8: What is the difference between ServiceFilter and TypeFilter?

**Answer:**
- `[ServiceFilter]`: Requires explicit DI registration (`AddScoped`, `AddTransient`, `AddSingleton`). Uses the registered instance. Faster.
- `[TypeFilter]`: Creates a new instance per request. Resolves dependencies through DI automatically. No manual registration needed. Slower but more flexible.

---

### Q9: What is model binding and what are its sources?

**Answer:**
- Model binding maps HTTP request data to action method parameters
- Sources: route values, query string, form data, request body, headers, DI container
- With `[ApiController]`, complex types default to `[FromBody]`
- Without `[ApiController]`, the framework uses conventions to determine the source

---

### Q10: How does catch-all parameter routing work?

**Answer:**
- Catch-all parameters capture remaining URL segments into a single parameter
- Defined using `*` prefix: `{*path}`
- Example: `[HttpGet("files/{*path}")]` matches `/files/a/b/c.txt` and captures `a/b/c.txt`
- Captured value does not include the leading slash
- Useful for proxying, file serving, flexible URL structures

---

### Q11: Explain the difference between IActionFilter and IAsyncActionFilter.

**Answer:**
- `IActionFilter` has two synchronous methods: `OnActionExecuting` (before) and `OnActionExecuted` (after)
- `IAsyncActionFilter` has a single async method: `OnActionExecutionAsync` which receives an `ActionExecutionDelegate`
- Use `IAsyncActionFilter` when performing async operations (DB calls, HTTP requests)
- Both allow modifying the request before the action and the result after the action

---

### Q12: What is API versioning and what are the common strategies?

**Answer:**
- API versioning supports multiple API versions simultaneously, allowing breaking changes
- **URL versioning**: `/api/v1/products` — most common, explicit
- **Query string versioning**: `?api-version=1.0` — simple
- **Header versioning**: Custom header `X-Api-Version: 2.0`
- **Media type versioning**: Custom `Accept` header — `application/vnd.myapp.v2+json`
- Implemented with `Asp.Versioning.Mvc` package

---

### Q13: What happens when model validation fails with [ApiController]?

**Answer:**
- `ModelState.IsValid` is checked automatically before the action executes
- If invalid, a `400 Bad Request` is returned with `ProblemDetails` JSON body
- The action method is never called
- The response includes a dictionary of field names and their validation errors
- This behavior can be customized via `ApiBehaviorOptions.InvalidModelStateResponseFactory`

---

### Q14: How do you apply filters globally vs controller-level vs action-level?

**Answer:**
- **Global**: Register in `Program.cs` via `builder.Services.AddControllers(options => { options.Filters.Add<...>(); })`
- **Controller-level**: Apply as attribute on the controller class: `[ServiceFilter(typeof(MyFilter))]`
- **Action-level**: Apply as attribute on the action method: `[ServiceFilter(typeof(MyFilter))]`
- Execution order: Global → Controller → Action

---

### Q15: What is the difference between a resource filter and an action filter?

**Answer:**
- **Resource filters** run BEFORE model binding and after the action/result. They can short-circuit the entire pipeline. Used for caching.
- **Action filters** run AFTER model binding but before/after the action method. They cannot prevent model binding. Used for logging, input/output modification.
- Resource filters have access to `ResourceExecutingContext`; action filters have access to `ActionExecutingContext`.

---

### Q16: How do you implement custom validation in ASP.NET Core?

**Answer:**
- **Data Annotations**: Create custom attributes inheriting from `ValidationAttribute` and overriding `IsValid`
- **IValidatableObject**: Implement `Validate` method in the model class for cross-field validation
- **FluentValidation**: Use `AbstractValidator<T>` to define rules in separate validator classes
- All approaches integrate with `[ApiController]` auto-validation

---

### Q17: What does the `?` mean in route templates?

**Answer:**
- It makes a route parameter optional
- Without `?`, the route won't match if that segment is missing from the URL
- Example: `{id?}` — the route matches both `/products/5` and `/products`
- Use nullable types (`int?`) in the action method to handle optional parameters

---

### Q18: Can you use multiple HTTP method attributes on a single action? When would you?

**Answer:**
- Yes: `[HttpGet("{id}")]` and `[HttpHead("{id}")]` can coexist on the same action
- Useful when the same logic handles different HTTP methods
- Common case: `[HttpGet]` and `[HttpHead]` — HEAD returns same headers as GET but no body
- Less common: `[HttpPost]` and `[HttpPut]` for upsert operations

---

### Q19: How do exception filters differ from exception handling middleware?

**Answer:**
- Exception filters only catch exceptions in MVC actions (not middleware, not Razor Pages outside MVC)
- Middleware catches ALL exceptions in the pipeline
- Exception filters can access MVC-specific context (`ActionContext`, `ExceptionContext`)
- For global error handling, middleware or `IExceptionHandler` is preferred
- Exception filters are useful for MVC-specific error responses (custom ProblemDetails)

---

### Q20: What is the `ModelState` dictionary and what does it contain?

**Answer:**
- `ModelState` tracks validation state for each parameter/property during model binding
- Contains: validation errors, raw values, attempted values, model state entries
- Each entry can have multiple `ModelError` objects with messages
- Accessible via `ModelState.IsValid`, `ModelState["fieldName"].Errors`
- With `[ApiController]`, invalid `ModelState` automatically returns 400
- Without it, you check `ModelState.IsValid` manually
