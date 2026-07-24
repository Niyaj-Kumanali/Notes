# REST API Design in ASP.NET Core

---

## RESTful API Fundamentals

### What is REST (Representational State Transfer)

- REST is an **architectural style** for designing networked applications, introduced by Roy Fielding in his 2000 doctoral dissertation
- It is **not a protocol or standard** — it is a set of constraints and principles that guide how distributed systems communicate
- REST relies on a **stateless, client-server communication protocol** — almost always HTTP
- Resources are identified by **URIs** (Uniform Resource Identifiers), and interactions use standard HTTP methods
- The key idea: instead of sending commands to the server, the client transfers a **representation** of a resource's state
- Example: `GET /api/products/5` retrieves the current state (representation) of product with ID 5

### REST Constraints

REST defines **six architectural constraints** that an API must follow:

#### 1. Client-Server Architecture

- The client and server are **separate concerns** and evolve independently
- The client handles the UI/presentation layer, the server handles data storage and business logic
- They communicate over a uniform interface (HTTP)
- This separation allows each to be developed, deployed, and scaled independently

#### 2. Stateless

- Each request from client to server must contain **all the information** needed to understand and process the request
- The server does **not store client context** between requests
- Authentication tokens (JWT), request parameters, and headers carry all necessary state
- This improves reliability, scalability, and visibility
- Example: Every API call includes an `Authorization: Bearer <token>` header — the server never stores session state

```csharp
// Stateless: server doesn't remember previous requests
[ApiController]
[Route("api/[controller]")]
[Authorize] // Token must be sent with every request
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetProduct(int id)
    {
        // Server doesn't rely on any previous request state
        // Every request is self-contained
        var product = _repository.GetById(id);
        return Ok(product);
    }
}
```

#### 3. Uniform Interface

- A standardized way of communicating between client and server
- Four sub-constraints:
  - **Resource identification through URIs**: each resource has a unique identifier (URL)
  - **Manipulation of resources through representations**: the client receives enough information to modify or delete a resource
  - **Self-descriptive messages**: each message includes enough information to describe how to process it (content type, status codes)
  - **HATEOAS** (Hypermedia as the Engine of Application State): responses include links to related resources

#### 4. Layered System

- The client cannot tell whether it is connected directly to the server or through intermediaries
- Layers can include load balancers, proxies, gateways, and CDNs
- Each layer only knows about the immediately adjacent layer
- This enables load balancing, security enforcement, and shared caches without client knowledge

#### 5. Cacheable

- Responses must explicitly indicate whether they are **cacheable or not**
- If cacheable, the client can reuse the response data for equivalent future requests
- Uses HTTP cache headers: `Cache-Control`, `ETag`, `Last-Modified`, `Expires`
- Reduces unnecessary server round-trips and improves performance

#### 6. Code on Demand (optional)

- Servers can temporarily extend client functionality by transferring **executable code** (e.g., JavaScript)
- This is the only optional constraint
- Rarely used in traditional REST APIs

### Resource-Based URLs

- URLs represent **nouns** (resources), not **verbs** (actions)
- The HTTP method defines the action, the URL defines the target

```
GET    /api/products          → Get all products
GET    /api/products/5        → Get product with ID 5
POST   /api/products          → Create a new product
PUT    /api/products/5        → Update product with ID 5
DELETE /api/products/5        → Delete product with ID 5

GET    /api/products/5/reviews       → Get reviews for product 5
POST   /api/products/5/reviews       → Add a review for product 5
GET    /api/customers/12/orders      → Get orders for customer 12
```

- Use **plural nouns** for collections: `/api/products` (not `/api/product`)
- Use **nested resources** for relationships: `/api/customers/12/orders`
- Keep URLs **flat and predictable** — avoid deep nesting (max 2-3 levels)

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET /api/products
    [HttpGet]
    public IActionResult GetAll() { /* ... */ }

    // GET /api/products/5
    [HttpGet("{id}")]
    public IActionResult GetById(int id) { /* ... */ }

    // POST /api/products
    [HttpPost]
    public IActionResult Create([FromBody] CreateProductDto dto) { /* ... */ }

    // PUT /api/products/5
    [HttpPut("{id}")]
    public IActionResult Update(int id, [FromBody] UpdateProductDto dto) { /* ... */ }

    // DELETE /api/products/5
    [HttpDelete("{id}")]
    public IActionResult Delete(int id) { /* ... */ }

    // GET /api/products/5/reviews
    [HttpGet("{id}/reviews")]
    public IActionResult GetReviews(int id) { /* ... */ }
}
```

---

## HTTP Methods

### GET — Retrieve Resources

- Retrieves a representation of the specified resource
- **Safe**: does not modify the resource
- **Idempotent**: multiple identical requests produce the same result
- **Cacheable**: responses can be cached by the browser and intermediaries
- Should never have side effects on the server
- Response body contains the resource representation

```csharp
[HttpGet("{id}")]
[ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<IActionResult> GetProduct(int id)
{
    var product = await _context.Products.FindAsync(id);
    if (product == null)
        return NotFound();

    return Ok(_mapper.Map<ProductDto>(product));
}
```

### POST — Create Resources

- Submits an entity to the specified resource, often causing a change in state or side effects
- **Not safe**: modifies server state
- **Not idempotent**: multiple identical POST requests may create multiple resources
- The server assigns the resource ID (not the client)
- On success, return **201 Created** with a `Location` header pointing to the new resource
- Include the created resource in the response body

```csharp
[HttpPost]
[ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public async Task<IActionResult> CreateProduct([FromBody] CreateProductDto dto)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);

    var product = _mapper.Map<Product>(dto);
    _context.Products.Add(product);
    await _context.SaveChangesAsync();

    var productDto = _mapper.Map<ProductDto>(product);

    return CreatedAtAction(
        nameof(GetProduct),
        new { id = product.Id },
        productDto);
}
```

### PUT — Replace Entire Resource

- Replaces the target resource completely with the request payload
- **Safe**: no (modifies resource)
- **Idempotent**: applying the same PUT multiple times produces the same result as applying it once
- The client sends the **complete updated representation**
- If the resource doesn't exist, it can either be created (201) or return 404
- Use when the client has the full updated representation of the resource

```csharp
[HttpPut("{id}")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public async Task<IActionResult> UpdateProduct(int id, [FromBody] UpdateProductDto dto)
{
    var product = await _context.Products.FindAsync(id);
    if (product == null)
        return NotFound();

    // Replace all fields with values from DTO
    product.Name = dto.Name;
    product.Price = dto.Price;
    product.Description = dto.Description;
    product.CategoryId = dto.CategoryId;

    await _context.SaveChangesAsync();
    return NoContent();
}
```

### PATCH — Partial Update

- Applies **partial modifications** to a resource
- **Safe**: no (modifies resource)
- **Not necessarily idempotent** (depends on implementation)
- Uses JSON Merge Patch (`application/merge-patch+json`) or JSON Patch (`application/json-patch+json`)
- Only sends the fields that need to change
- More bandwidth-efficient than PUT for large resources

```csharp
// Using JSON Merge Patch
[HttpPatch("{id}")]
[Consumes("application/merge-patch+json")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
public async Task<IActionResult> PatchProduct(int id, [FromBody] JsonPatchDocument<UpdateProductDto> patchDoc)
{
    var product = await _context.Products.FindAsync(id);
    if (product == null)
        return NotFound();

    var productDto = _mapper.Map<UpdateProductDto>(product);

    patchDoc.ApplyTo(productDto, ModelState);

    if (!TryValidateModel(productDto))
        return BadRequest(ModelState);

    _mapper.Map(productDto, product);
    await _context.SaveChangesAsync();

    return NoContent();
}
```

```json
// JSON Merge Patch request body — only send fields to change
{
    "price": 29.99
}
```

```json
// JSON Patch request body — operations format
[
    { "op": "replace", "path": "/price", "value": 29.99 },
    { "op": "add", "path": "/tags", "value": ["sale"] }
]
```

### DELETE — Remove Resources

- Deletes the specified resource
- **Safe**: no (modifies resource)
- **Idempotent**: deleting the same resource multiple times has the same effect as deleting it once
- First delete returns 200 or 204, subsequent deletes can return 204 or 404
- Often returns 204 No Content on success

```csharp
[HttpDelete("{id}")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<IActionResult> DeleteProduct(int id)
{
    var product = await _context.Products.FindAsync(id);
    if (product == null)
        return NotFound();

    _context.Products.Remove(product);
    await _context.SaveChangesAsync();

    return NoContent();
}
```

### HEAD — Same as GET but No Body

- Identical to GET but the server does **not return a message body**
- Returns only the headers (Content-Type, Content-Length, cache headers, etc.)
- Useful for checking if a resource exists or getting metadata
- Safe and idempotent
- Commonly used by load balancers and caches

```csharp
[HttpHead("{id}")]
[ProducesResponseType(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<IActionResult> HeadProduct(int id)
{
    var exists = await _context.Products.AnyAsync(p => p.Id == id);
    if (!exists)
        return NotFound();

    return Ok(); // No body, just headers
}
```

### OPTIONS — Discover Allowed Methods

- Returns the HTTP methods supported by the server for a specific URL
- Used primarily for **CORS preflight requests**
- Response includes an `Allow` header listing permitted methods
- Safe and idempotent

```csharp
// ASP.NET Core handles OPTIONS automatically, but you can customize:
[HttpOptions]
[ProducesResponseType(StatusCodes.Status200OK)]
public IActionResult Options()
{
    Response.Headers.Append("Allow", "GET, POST, PUT, DELETE, PATCH, OPTIONS");
    return Ok();
}
```

---

## HTTP Status Codes

### 2xx — Success

#### 200 OK

- The request succeeded
- Used for GET, PUT, PATCH, and DELETE responses
- Response body contains the result

```csharp
[HttpGet("{id}")]
public IActionResult GetProduct(int id)
{
    var product = _service.GetById(id);
    return Ok(product); // 200 with product in body
}
```

#### 201 Created

- A new resource was successfully created
- Used for POST responses
- **Must include** a `Location` header with the URL of the new resource
- Should include the created resource in the response body

```csharp
[HttpPost]
public IActionResult CreateProduct([FromBody] CreateProductDto dto)
{
    var product = _service.Create(dto);
    return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    // 201 + Location header + resource body
}
```

#### 204 No Content

- The request succeeded but there is no content to send back
- Used for DELETE and PUT responses where no body is needed
- Commonly used when the operation was successful but nothing needs to be returned

```csharp
[HttpDelete("{id}")]
public IActionResult DeleteProduct(int id)
{
    _service.Delete(id);
    return NoContent(); // 204, no body
}
```

### 4xx — Client Errors

#### 400 Bad Request

- The server cannot process the request due to **malformed syntax**, invalid data, or missing required fields
- Use for validation failures
- Include details about what went wrong in the response body

```csharp
if (!ModelState.IsValid)
    return BadRequest(ModelState);
    // 400 + validation errors in body
```

```json
{
    "errors": {
        "Name": ["The Name field is required."],
        "Price": ["The Price field must be greater than 0."]
    }
}
```

#### 401 Unauthorized

- The client has **not authenticated** or provided invalid credentials
- The request lacks a valid authentication token
- Include a `WWW-Authenticate` header indicating the required authentication scheme
- Despite the name, this is about **authentication** (identity), not authorization

```csharp
[Authorize] // Without a valid token, returns 401
[HttpGet]
public IActionResult GetSecretData()
{
    return Ok(new { data = "secret" });
}
```

#### 403 Forbidden

- The client **is authenticated** but does **not have permission** to perform the action
- This is about **authorization** (permissions), not authentication
- Different from 401 — the server knows who you are, but won't let you do this

```csharp
[Authorize(Roles = "Admin")]
[HttpDelete("{id}")]
public IActionResult DeleteProduct(int id)
{
    // If authenticated user is not an Admin, returns 403
    _service.Delete(id);
    return NoContent();
}
```

#### 404 Not Found

- The requested resource **does not exist**
- Can also be used intentionally to avoid leaking information about resource existence
- Should not reveal whether the resource exists but is hidden vs. truly nonexistent

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetProduct(int id)
{
    var product = await _context.Products.FindAsync(id);
    if (product == null)
        return NotFound(); // 404
    return Ok(product);
}
```

#### 409 Conflict

- The request conflicts with the **current state** of the resource
- Common scenarios: duplicate entries, version conflicts (optimistic concurrency), state conflicts

```csharp
[HttpPost]
public async Task<IActionResult> CreateUser([FromBody] CreateUserDto dto)
{
    if (await _context.Users.AnyAsync(u => u.Email == dto.Email))
        return Conflict(new { message = "A user with this email already exists." });

    // ...
}
```

```csharp
// Optimistic concurrency conflict
[HttpPut("{id}")]
public async Task<IActionResult> UpdateProduct(int id, [FromBody] UpdateProductDto dto)
{
    var product = await _context.Products.FindAsync(id);
    product.Name = dto.Name;

    try
    {
        await _context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException)
    {
        return Conflict(new { message = "The resource was modified by another user. Please refresh and try again." });
    }

    return NoContent();
}
```

#### 422 Unprocessable Entity

- The request is **syntactically correct** but **semantically incorrect**
- The server understands the content type and structure, but cannot process the instructions
- Common for validation errors where the JSON is valid but the data doesn't make sense

```csharp
[HttpPost]
public IActionResult CreateOrder([FromBody] CreateOrderDto dto)
{
    // JSON is valid, but the business rules are violated
    if (dto.DeliveryDate < DateTime.UtcNow)
    {
        return UnprocessableEntity(new
        {
            message = "Delivery date must be in the future.",
            field = "DeliveryDate"
        });
    }
    // ...
}
```

### 5xx — Server Errors

#### 500 Internal Server Error

- The server encountered an **unexpected condition** that prevented it from fulfilling the request
- Typically caused by unhandled exceptions
- Never expose internal details or stack traces to the client in production

```csharp
// Global exception handler middleware
public class ExceptionHandlingMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");

            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new
            {
                message = "An internal server error occurred. Please try again later."
                // Never expose ex.Message or stack trace in production
            });
        }
    }
}
```

---

## Idempotency

### What is Idempotency

- An operation is **idempotent** if making the same request multiple times produces the **same result** as making it once
- Idempotency is about the **effect on the server state**, not about the response
- The response can differ (e.g., first GET returns 200 with data, second GET returns 404 if deleted), but the server state after the operation must be the same

### Idempotent HTTP Methods

| Method   | Safe? | Idempotent? | Cacheable? |
|----------|-------|-------------|------------|
| GET      | Yes   | Yes         | Yes        |
| HEAD     | Yes   | Yes         | Yes        |
| OPTIONS  | Yes   | Yes         | No         |
| PUT      | No    | Yes         | No         |
| DELETE   | No    | Yes         | No         |
| PATCH    | No    | Sometimes   | No         |
| POST     | No    | **No**      | No         |

### Why POST is Not Idempotent

- Multiple identical POST requests can **create multiple resources**
- Each POST is considered a new action, even with the same payload
- Example: sending the same "create order" request twice can result in two separate orders

```csharp
// Two identical POST requests = two different orders
[HttpPost]
public IActionResult CreateOrder([FromBody] CreateOrderDto dto)
{
    var order = new Order { /* ... */ };
    _context.Orders.Add(order);
    _context.SaveChanges();
    return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
}
// POST #1 → Order ID 101 created
// POST #2 → Order ID 102 created (duplicate!)
```

### Making POST Idempotent with Idempotency Keys

- The client generates a unique **idempotency key** (e.g., GUID) and includes it in the request header
- The server stores the key and the result of the first request
- If a request with the same key arrives again, the server returns the **stored result** without re-executing the operation

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder(
    [FromBody] CreateOrderDto dto,
    [FromHeader(Name = "Idempotency-Key")] string idempotencyKey)
{
    if (string.IsNullOrEmpty(idempotencyKey))
        return BadRequest("Idempotency-Key header is required.");

    // Check if we've already processed this request
    var existingResult = await _cache.GetAsync($"idempotent:{idempotencyKey}");
    if (existingResult != null)
        return Content(HttpStatusCode.OK, existingResult, "application/json");

    // Process the request
    var order = new Order { /* ... */ };
    _context.Orders.Add(order);
    await _context.SaveChangesAsync();

    var result = CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);

    // Store the result for future idempotent retries
    await _cache.SetAsync(
        $"idempotent:{idempotencyKey}",
        JsonSerializer.Serialize(order),
        TimeSpan.FromHours(24));

    return result;
}
```

### Why Idempotency Matters for Retry Logic

- Network failures can cause the client to **not receive** the server's response
- Without idempotency, retrying the request could cause **duplicate operations** (double charges, duplicate orders, etc.)
- Idempotent operations can be safely retried
- This is critical for:
  - Payment processing
  - Order creation
  - Any operation with financial or data integrity implications
- Non-idempotent operations (POST) need explicit idempotency keys to be safely retried

---

## DTOs and API Design

### What is a DTO (Data Transfer Object)

- A plain object used to **transfer data** between layers without exposing internal domain models
- Acts as a boundary contract between the API and its consumers
- Decouples the external API surface from the internal data model
- Allows different shapes for requests vs. responses

### Why Not Expose EF Entities Directly

- Entities contain **navigation properties** that can cause **circular reference** serialization issues (infinite loops)
- Entities may contain **sensitive fields** (password hashes, internal IDs, audit columns) that shouldn't be exposed
- Entity structure **couples** the API to the database schema — schema changes break the API contract
- Entities may not match the **shape** the client actually needs
- EF lazy loading can cause **N+1 query problems** when serializing

```csharp
// BAD: Exposing EF entity directly
[HttpGet("{id}")]
public IActionResult GetProduct(int id)
{
    var product = _context.Products
        .Include(p => p.Category)      // extra query
        .Include(p => p.Reviews)       // extra query
        .Include(p => p.Images)        // extra query
        .FirstOrDefault(p => p.Id == id);

    return Ok(product); // Exposes: PasswordHash, InternalNotes, Category.Reviews...
    // Circular reference exception if navigation properties reference each other
}

// GOOD: Using a DTO
[HttpGet("{id}")]
public IActionResult GetProduct(int id)
{
    var product = _context.Products.Find(id);
    if (product == null) return NotFound();

    return Ok(_mapper.Map<ProductDto>(product));
    // Only exposes: Id, Name, Price, Description, CategoryName
}
```

### Request DTOs vs Response DTOs

- **Request DTOs** (Input): define what the client sends to the server
  - May have `[Required]`, `[Range]`, `[MaxLength]` validation attributes
  - Does not include server-generated fields (Id, CreatedAt, UpdatedAt)
  - Example: `CreateProductDto`, `UpdateProductDto`

- **Response DTOs** (Output): define what the server sends back to the client
  - Includes server-generated fields (Id, CreatedAt)
  - May include computed properties (e.g., average rating, full name)
  - Example: `ProductDto`, `ProductSummaryDto`

```csharp
// Request DTO — what the client sends when creating a product
public class CreateProductDto
{
    [Required]
    [MaxLength(200)]
    public string Name { get; set; }

    [Required]
    [Range(0.01, 999999.99)]
    public decimal Price { get; set; }

    [MaxLength(2000)]
    public string Description { get; set; }

    [Required]
    public int CategoryId { get; set; }
}

// Response DTO — what the client receives
public class ProductDto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string Description { get; set; }
    public string CategoryName { get; set; }
    public DateTime CreatedAt { get; set; }
    public int ReviewCount { get; set; }
    public double AverageRating { get; set; }
}
```

### Pagination DTOs

- Never return unbounded collections — always paginate
- `PagedResult<T>` wraps the data with pagination metadata

```csharp
public class PagedResult<T>
{
    public List<T> Items { get; set; }
    public int TotalCount { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => PageNumber > 1;
    public bool HasNextPage => PageNumber < TotalPages;
}

public class PaginationQuery
{
    private const int MaxPageSize = 100;
    private int _pageSize = 10;

    public int PageNumber { get; set; } = 1;

    public int PageSize
    {
        get => _pageSize;
        set => _pageSize = Math.Min(value, MaxPageSize);
    }
}
```

```csharp
[HttpGet]
public async Task<IActionResult> GetProducts([FromQuery] PaginationQuery pagination)
{
    var query = _context.Products.AsQueryable();

    var totalCount = await query.CountAsync();
    var items = await query
        .Skip((pagination.PageNumber - 1) * pagination.PageSize)
        .Take(pagination.PageSize)
        .ToListAsync();

    var result = new PagedResult<ProductDto>
    {
        Items = _mapper.Map<List<ProductDto>>(items),
        TotalCount = totalCount,
        PageNumber = pagination.PageNumber,
        PageSize = pagination.PageSize
    };

    return Ok(result);
}
```

### API Response Wrapper Pattern

- Wrapping all responses in a consistent structure makes the API easier to consume
- Provides a uniform way to communicate success, errors, and metadata

```csharp
public class ApiResponse<T>
{
    public bool Success { get; set; }
    public string Message { get; set; }
    public T Data { get; set; }
    public List<string> Errors { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;

    public static ApiResponse<T> SuccessResponse(T data, string message = null)
    {
        return new ApiResponse<T>
        {
            Success = true,
            Data = data,
            Message = message
        };
    }

    public static ApiResponse<T> FailResponse(string message, List<string> errors = null)
    {
        return new ApiResponse<T>
        {
            Success = false,
            Message = message,
            Errors = errors ?? new List<string>()
        };
    }
}
```

```csharp
// Usage in controllers
[HttpGet("{id}")]
public async Task<IActionResult> GetProduct(int id)
{
    var product = await _service.GetByIdAsync(id);
    if (product == null)
        return NotFound(ApiResponse<ProductDto>.FailResponse("Product not found."));

    return Ok(ApiResponse<ProductDto>.SuccessResponse(product));
}
```

```json
// Success response
{
    "success": true,
    "message": null,
    "data": {
        "id": 1,
        "name": "Widget",
        "price": 9.99
    },
    "errors": [],
    "timestamp": "2026-07-24T10:30:00Z"
}

// Error response
{
    "success": false,
    "message": "Validation failed.",
    "data": null,
    "errors": ["Name is required.", "Price must be greater than 0."],
    "timestamp": "2026-07-24T10:30:00Z"
}
```

### Content Negotiation

- The client specifies what format it wants via the `Accept` header
- ASP.NET Core supports this out of the box with `AddControllers().AddXmlSerializerFormats()`
- Common formats: `application/json` (default), `application/xml`, `application/hal+json`

```csharp
// Client request
GET /api/products/5
Accept: application/xml

// Server responds in XML because the client asked for it
```

### HATEOAS (Hypermedia as the Engine of Application State)

- Responses include **links** to related resources, allowing the client to navigate the API dynamically
- The client doesn't need to hardcode URLs — it discovers them from responses
- Example: a product response includes links to update, delete, and view reviews
- Optional in most APIs but is considered the highest maturity level of REST (Richardson Maturity Model Level 3)

```json
{
    "id": 1,
    "name": "Widget",
    "price": 9.99,
    "links": [
        { "rel": "self", "href": "/api/products/1", "method": "GET" },
        { "rel": "update", "href": "/api/products/1", "method": "PUT" },
        { "rel": "delete", "href": "/api/products/1", "method": "DELETE" },
        { "rel": "reviews", "href": "/api/products/1/reviews", "method": "GET" },
        { "rel": "category", "href": "/api/categories/3", "method": "GET" }
    ]
}
```

---

## API Versioning

### Why Versioning Matters

- APIs are **contracts** with consumers — breaking changes can break client applications
- Multiple client versions may coexist (mobile app v1, web app v2, third-party integrations)
- Versioning allows you to evolve the API without breaking existing consumers
- Without versioning, even small changes (removing a field, changing a type) can cause outages

### Versioning Strategies

#### URL Path Versioning (Most Common)

- Version is embedded directly in the URL path
- Simple, explicit, easy to route
- `GET /api/v1/products`
- `GET /api/v2/products`

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult Get(int id) { /* V1 logic */ }
}

[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("2.0")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult Get(int id) { /* V2 logic with new fields */ }
}
```

#### Query String Versioning

- Version is passed as a query parameter
- `GET /api/products?api-version=2.0`
- Less clean URL but doesn't require URL changes

#### Header Versioning

- Version is specified in a custom or standard header
- `GET /api/products` with header `api-version: 2.0`
- Keeps URLs clean but less discoverable

### Breaking vs Non-Breaking Changes

**Non-breaking changes** (safe to add without a new version):
- Adding new endpoints
- Adding new optional fields to responses
- Adding new optional query parameters
- Adding new HTTP methods to existing endpoints

**Breaking changes** (require a new version):
- Removing a field from a response
- Renaming a field
- Changing a field's type
- Changing the semantics of an existing endpoint
- Making an optional field required
- Changing authentication requirements

### Deprecation Strategy

- Mark deprecated endpoints/fields with the `Deprecation` and `Sunset` headers
- Provide migration guides in documentation
- Support old versions for a defined sunset period (e.g., 6-12 months)
- Monitor usage of old versions before fully removing them
- Communicate deprecation timelines to API consumers

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jan 2027 00:00:00 GMT
Link: <https://api.example.com/docs/v2-migration>; rel="sunset"
```

---

## CORS

### What is CORS (Cross-Origin Resource Sharing)

- A security mechanism that allows or restricts web pages from one **origin** to request resources from a different origin
- An **origin** is defined by the combination of **protocol + hostname + port** (e.g., `https://example.com:443`)
- Without CORS, a web app at `https://frontend.com` cannot call an API at `https://api.backend.com` due to browser restrictions

### Why Browsers Enforce CORS (Same-Origin Policy)

- The **Same-Origin Policy** prevents a malicious website from reading data from another site
- Without it, any website could make requests to your bank's API using your browser cookies (CSRF-like attacks)
- CORS allows servers to **explicitly declare** which origins are permitted
- CORS is enforced by the **browser** — server-to-server requests are not affected

### Preflight Requests (OPTIONS)

- For "complex" requests (non-simple methods, custom headers, non-standard content types), the browser sends a **preflight OPTIONS request** first
- The preflight asks the server: "Is this cross-origin request allowed?"
- If the server responds with appropriate CORS headers, the browser proceeds with the actual request
- "Simple" requests (GET, HEAD, POST with standard headers) skip the preflight

```
// Preflight flow:
1. Browser sends: OPTIONS /api/products
   Origin: https://frontend.com
   Access-Control-Request-Method: POST
   Access-Control-Request-Headers: Content-Type, Authorization

2. Server responds:
   Access-Control-Allow-Origin: https://frontend.com
   Access-Control-Allow-Methods: GET, POST, PUT, DELETE
   Access-Control-Allow-Headers: Content-Type, Authorization
   Access-Control-Max-Age: 86400

3. Browser sends the actual POST request
```

### Configuring CORS in ASP.NET Core

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowSpecificOrigin", policy =>
    {
        policy.WithOrigins("https://frontend.com", "https://admin.com")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials(); // Allow cookies/auth headers
    });

    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
        // NOTE: AllowAnyOrigin + AllowCredentials is NOT allowed
    });
});

var app = builder.Build();

// Must be before MapControllers()
app.UseCors("AllowSpecificOrigin");

app.MapControllers();
app.Run();
```

### Allowing Credentials

- By default, browsers do **not** send cookies or authentication headers in cross-origin requests
- The server must set `Access-Control-Allow-Credentials: true`
- The client must set `withCredentials: true` (fetch) or `xhr.withCredentials = true` (XMLHttpRequest)
- When using credentials, you **cannot** use `AllowAnyOrigin()` — you must specify allowed origins explicitly

```csharp
// Client-side (JavaScript)
fetch('https://api.backend.com/products', {
    method: 'GET',
    credentials: 'include' // Sends cookies
});

// Or with Axios
axios.get('https://api.backend.com/products', {
    withCredentials: true
});
```

---

## API Documentation

### Swagger / OpenAPI

- Swagger is a set of tools for documenting, testing, and consuming REST APIs
- **OpenAPI** is the specification (formerly Swagger Specification) that Swagger is built on
- OpenAPI provides a machine-readable API description (JSON or YAML)
- Enables auto-generated interactive documentation (Swagger UI)
- Enables client SDK generation

### Swashbuckle / NSwag

- **Swashbuckle**: the most common Swagger package for ASP.NET Core, integrates directly with `Microsoft.AspNetCore.Mvc`
- **NSwag**: alternative that offers more advanced features (code generation, TypeScript generation)

```csharp
// Program.cs — Swashbuckle setup
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "A REST API for managing products",
        Contact = new OpenApiContact
        {
            Name = "API Support",
            Email = "support@example.com"
        }
    });

    // Add JWT authentication to Swagger
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "Bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "Enter your JWT token"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});

// In pipeline
app.UseSwagger();
app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
    options.RoutePrefix = "swagger"; // Set to empty string to serve at root
});
```

### Securing Swagger in Production

- Never expose Swagger UI in production without access controls
- Use environment checks to conditionally enable Swagger

```csharp
// Only enable Swagger in development
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Or use basic authentication for Swagger
app.UseSwaggerUI(options =>
{
    // Only enable in non-production
    if (!app.Environment.IsProduction())
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "API V1");
        options.RoutePrefix = "swagger";
    }
});
```

### API Documentation Best Practices

- Document **all** endpoints with request/response examples
- Document all **status codes** each endpoint can return
- Include **authentication requirements** in the docs
- Provide **code samples** in popular languages (C#, JavaScript, Python)
- Keep documentation **in sync** with code — use XML comments and attributes
- Use ` ProducesResponseType` attributes for accurate Swagger documentation

```csharp
/// <summary>
/// Gets a product by its unique identifier.
/// </summary>
/// <param name="id">The product ID</param>
/// <returns>The product details</returns>
/// <response code="200">Returns the product</response>
/// <response code="404">Product not found</response>
[HttpGet("{id}")]
[ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<IActionResult> GetProduct(int id) { /* ... */ }
```

---

## Common REST Mistakes

### Using Verbs in URLs

- **Wrong**: `GET /api/getProducts`, `POST /api/createProduct`, `DELETE /api/deleteProduct/5`
- **Correct**: `GET /api/products`, `POST /api/products`, `DELETE /api/products/5`
- The HTTP method **is** the verb — the URL should only identify the resource
- Verbs in URLs indicate a misunderstanding of REST principles

### Not Returning Proper Status Codes

- Returning `200 OK` for everything (including errors) forces the client to parse the body to determine if something went wrong
- Returning `400` when `422` is more appropriate
- Not returning `201 Created` for successful POST operations
- Returning `200` for DELETE when `204 No Content` is more semantic

### Not Versioning APIs

- Making breaking changes without versioning forces all consumers to update simultaneously
- Even internal APIs should be versioned — team changes and service dependencies evolve
- Start with v1 from day one, even if it feels premature

### Exposing Internal Database Schema

- API field names should match **client needs**, not database column names
- Don't expose `user_id` when the client expects `userId` — use DTOs to map
- Never expose internal fields like `password_hash`, `is_deleted`, `created_by_internal_system`

### Not Handling Partial Updates Correctly

- Using PUT when you mean PATCH, or vice versa
- PUT should replace the **entire** resource; PATCH should update only **provided** fields
- Not handling `null` values in PATCH — distinguishing "don't change" from "set to null"

### Ignoring Caching Headers

- Not setting `Cache-Control`, `ETag`, or `Last-Modified` headers wastes bandwidth and performance
- GET responses should include appropriate cache directives
- Use `ETag` for conditional requests (avoid re-downloading unchanged data)

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetProduct(int id)
{
    var product = await _service.GetByIdAsync(id);

    // Generate ETag from product's last updated time
    var etag = $"\"{product.UpdatedAt.Ticks}\"";

    // Return 304 Not Modified if client has current version
    if (Request.Headers.IfNoneMatch.ToString() == etag)
        return StatusCode(StatusCodes.Status304NotModified);

    Response.Headers.ETag = etag;
    Response.Headers.CacheControl = "private, max-age=60";

    return Ok(_mapper.Map<ProductDto>(product));
}
```

### Not Validating Input

- Never trust client input — validate everything
- Use `[Required]`, `[Range]`, `[MaxLength]`, `[RegularExpression]` attributes on DTOs
- Validate business rules (not just data format) — use a validation library like FluentValidation
- Return clear, actionable error messages

```csharp
public class CreateOrderValidator : AbstractValidator<CreateOrderDto>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.CustomerId).GreaterThan(0);
        RuleFor(x => x.Items).NotEmpty().WithMessage("Order must have at least one item.");
        RuleFor(x => x.Items).Must(items =>
            items.All(i => i.Quantity > 0 && i.Quantity <= 1000))
            .WithMessage("Item quantity must be between 1 and 1000.");
        RuleFor(x => x.DeliveryDate)
            .GreaterThan(DateTime.UtcNow.AddDays(1))
            .WithMessage("Delivery date must be at least 2 days in the future.");
    }
}
```

---

## Interview Questions

### Question 1: What are the key constraints of REST?

**Answer:**

- **Client-Server**: The client and server are independent and can evolve separately. The client handles the UI, the server handles data and logic.
- **Stateless**: Each request contains all the information the server needs. No session state is stored between requests. Authentication tokens (JWT) are sent with every request.
- **Uniform Interface**: Resources are identified by URIs, and interactions use standard HTTP methods. Responses are self-descriptive and include hypermedia links.
- **Layered System**: The client cannot tell if it's connected directly to the server or through intermediaries (load balancers, proxies, CDNs).
- **Cacheable**: Responses explicitly indicate whether they can be cached, improving performance and reducing unnecessary server requests.
- **Code on Demand** (optional): Servers can temporarily extend client functionality with executable code.

### Question 2: What is the difference between PUT and PATCH?

**Answer:**

- **PUT** replaces the **entire** resource with the new representation. You must send all fields. If a field is omitted, it may be reset to its default or null value.
- **PATCH** applies **partial** modifications to a resource. You only send the fields that need to change.
- PUT is always idempotent. PATCH is not necessarily idempotent (though it can be if implemented carefully).
- Use PUT when the client has the complete updated representation. Use PATCH when the client only wants to change specific fields.

```csharp
// PUT: must send everything
{ "name": "Widget", "price": 9.99, "description": "Updated", "categoryId": 3 }

// PATCH: only send what changes
{ "price": 19.99 }
```

### Question 3: Which HTTP methods are idempotent and why?

**Answer:**

- **Idempotent**: GET, HEAD, OPTIONS, PUT, DELETE
- GET is idempotent because fetching the same resource multiple times doesn't change the server state
- PUT is idempotent because sending the same complete replacement always results in the same resource state
- DELETE is idempotent because deleting an already-deleted resource has no additional effect
- HEAD and OPTIONS are idempotent because they are safe (read-only) operations
- **POST is NOT idempotent** because multiple identical requests create multiple resources
- PATCH is not necessarily idempotent — it depends on the implementation

### Question 4: What is the difference between 401 Unauthorized and 403 Forbidden?

**Answer:**

- **401 Unauthorized**: The client has **not provided valid authentication** credentials. The server doesn't know who the client is. Typically includes a `WWW-Authenticate` header indicating the required scheme.
- **403 Forbidden**: The client **is authenticated** (the server knows who they are) but **lacks permission** to perform the requested action.
- In short: 401 = "I don't know who you are." 403 = "I know who you are, but you can't do this."

### Question 5: What is a DTO and why should you use one?

**Answer:**

- A **Data Transfer Object** is a plain object used to transfer data between layers (API ↔ client).
- Reasons to use DTOs:
  - **Security**: Avoids exposing sensitive fields (password hashes, internal notes)
  - **Decoupling**: The API contract is independent of the database schema. Schema changes don't break the API.
  - **Performance**: Prevents N+1 query problems from EF navigation properties. Only fetches what's needed.
  - **Shape control**: Different operations can return different shapes (summary vs. detail)
  - **Validation**: Request DTOs can have validation attributes specific to each operation

### Question 6: How would you implement CORS in an ASP.NET Core API?

**Answer:**

```csharp
// In Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins("https://frontend.com")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

app.UseCors("AllowFrontend"); // Before MapControllers
```

Key points:
- CORS is enforced by the **browser**, not the server
- Preflight OPTIONS requests are sent automatically by the browser for "complex" requests
- You must specify explicit origins when using `AllowCredentials()` — `AllowAnyOrigin()` + `AllowCredentials()` is not allowed
- Place `UseCors()` before `MapControllers()` in the middleware pipeline

### Question 7: What is idempotency and how do you make POST requests idempotent?

**Answer:**

- **Idempotency** means that making the same request multiple times produces the same result as making it once.
- POST is not idempotent — sending the same POST request twice creates two resources.
- To make POST idempotent, use an **idempotency key**:
  1. The client generates a unique key (GUID) and includes it in a request header
  2. The server checks if the key has been seen before
  3. If yes, return the stored response from the first request
  4. If no, process the request, store the result with the key, and return it
- This prevents duplicate operations during network retries
- Essential for payment processing, order creation, and any financial operations

### Question 8: What are the common status codes you should use and when?

**Answer:**

| Status Code | When to Use |
|-------------|-------------|
| 200 OK | Successful GET, PUT, PATCH |
| 201 Created | Successful POST (resource created). Include `Location` header |
| 204 No Content | Successful DELETE or PUT with no response body |
| 400 Bad Request | Validation errors, malformed request body |
| 401 Unauthorized | Missing or invalid authentication credentials |
| 403 Forbidden | Authenticated but not authorized for this action |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | Duplicate resource, concurrency conflict |
| 422 Unprocessable Entity | Valid JSON but semantically incorrect data |
| 500 Internal Server Error | Unhandled server exception |

### Question 9: How should you version a REST API?

**Answer:**

- **URL versioning** (most common): `GET /api/v1/products`
- **Query string versioning**: `GET /api/products?api-version=1.0`
- **Header versioning**: `GET /api/products` with `api-version: 1.0` header
- Version when making **breaking changes**: removing fields, changing types, changing semantics
- Non-breaking changes (adding fields, new endpoints) don't require versioning
- Use **deprecation headers** (`Deprecation`, `Sunset`) to signal when a version will be removed
- Support old versions for a reasonable period (6-12 months)
- Monitor usage to determine when old versions can be safely retired

### Question 10: What are the most common REST API mistakes?

**Answer:**

- **Using verbs in URLs**: `/api/getProducts` instead of `GET /api/products`
- **Not versioning**: breaking changes without versioning break consumers
- **Wrong status codes**: returning 200 for everything instead of proper 4xx/5xx codes
- **Exposing entities**: returning EF entities directly instead of DTOs
- **No input validation**: trusting client data without validation
- **No pagination**: returning unbounded collections
- **Ignoring caching**: not setting Cache-Control or ETag headers
- **Inconsistent naming**: mixing camelCase and PascalCase in JSON responses
- **No error format**: returning raw exception messages or inconsistent error shapes

### Question 11: How do you handle pagination in a REST API?

**Answer:**

- Use **cursor-based** or **offset-based** pagination
- Offset-based: `GET /api/products?page=2&pageSize=20`
- Cursor-based: `GET /api/products?cursor=eyJpZCI6MTAwfQ&limit=20` (better for large datasets)
- Return a consistent response shape:
  - The items/results
  - Total count
  - Current page number and page size
  - Total pages
  - Links to next/previous pages
- Always set a **maximum page size** to prevent abuse (e.g., max 100)
- Use `PagedResult<T>` wrapper to keep the response consistent

### Question 12: What is HATEOAS and when would you use it?

**Answer:**

- **HATEOAS** (Hypermedia as the Engine of Application State) means API responses include **links** to related resources and available actions
- A product response might include links: `self`, `update`, `delete`, `reviews`, `category`
- The client discovers available actions by following links rather than hardcoding URLs
- This is the highest maturity level of REST (Richardson Maturity Model Level 3)
- When to use:
  - Public APIs consumed by many unknown clients
  - APIs where discoverability is important
  - Complex domain models with many relationships
- When to skip:
  - Simple internal APIs
  - When you control both client and server (tight coupling is acceptable)
  - When performance is critical and link resolution adds overhead

### Question 13: How do you handle errors consistently in a REST API?

**Answer:**

- Use a **consistent error response format** across all endpoints
- Always return the appropriate **HTTP status code**
- Include a human-readable **message** and machine-readable **error codes**
- Include **validation errors** as a list of field-level errors for 400 responses
- Never expose **internal details** (stack traces, database errors, connection strings) in production
- Use **global exception handling middleware** to catch unhandled exceptions

```json
{
    "success": false,
    "message": "Validation failed.",
    "errors": [
        { "field": "Name", "message": "Name is required." },
        { "field": "Price", "message": "Price must be greater than 0." }
    ]
}
```

### Question 14: What is content negotiation and how does it work?

**Answer:**

- Content negotiation is the process where the **client and server agree** on the format of the response
- The client sends an `Accept` header specifying preferred formats: `Accept: application/json` or `Accept: application/xml`
- The server checks if it can produce the requested format and responds accordingly
- If the server can't produce the requested format, it returns `406 Not Acceptable`
- In ASP.NET Core, configure with `AddControllers().AddXmlSerializerFormats()`
- Also applies to request bodies: the `Content-Type` header tells the server what format the client is sending

### Question 15: How do you secure a REST API?

**Answer:**

- **HTTPS everywhere**: encrypt all traffic with TLS
- **Authentication**: verify identity (JWT tokens, OAuth 2.0, API keys)
- **Authorization**: verify permissions (roles, policies, resource-based)
- **Input validation**: validate and sanitize all inputs
- **Rate limiting**: prevent abuse with throttling
- **CORS**: restrict which origins can access the API
- **Headers**: set security headers (`X-Content-Type-Options`, `X-Frame-Options`, CSP)
- **Logging**: log all authentication attempts and suspicious activity
- **Never expose**: stack traces, internal IDs, database schema in responses
- **Use `[Authorize]` globally**: require authentication by default, opt out with `[AllowAnonymous]`

```csharp
// Global authorization — require auth by default
builder.Services.AddControllers(options =>
{
    options.Filters.Add(new AuthorizeFilter());
});

// Public endpoints opt out
[AllowAnonymous]
[HttpGet("health")]
public IActionResult HealthCheck() => Ok("Healthy");
```

### Question 16: What is the difference between authentication and authorization?

**Answer:**

- **Authentication** answers: "Who are you?" — verifying the identity of the client (login, token validation)
- **Authorization** answers: "What can you do?" — determining if the authenticated client has permission for the requested action
- 401 Unauthorized is returned for **authentication** failures (missing/invalid credentials)
- 403 Forbidden is returned for **authorization** failures (authenticated but not permitted)
- They are separate concerns and should be handled independently
- Example: a regular user is authenticated (known identity) but gets 403 when trying to access admin endpoints (not authorized)

### Question 17: How do you handle large file uploads in a REST API?

**Answer:**

- Use `multipart/form-data` content type
- Configure Kestrel limits and request size limits in `Program.cs`
- Stream the file to storage rather than loading it entirely into memory
- Use `IFormFile` for small files, streaming for large files
- Return 201 Created with a resource URL pointing to the uploaded file
- Validate file type, size, and content before processing
- Consider chunked uploads for very large files

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.MaxRequestBodySize = 104857600; // 100 MB
});

[HttpPost("upload")]
public async Task<IActionResult> Upload(IFormFile file)
{
    if (file == null || file.Length == 0)
        return BadRequest("No file uploaded.");

    var path = Path.Combine("uploads", file.FileName);

    using var stream = new FileStream(path, FileMode.Create);
    await file.CopyToAsync(stream);

    return CreatedAtAction(nameof(GetFile), new { name = file.FileName }, new { path });
}
```

### Question 18: What are API rate limiting strategies?

**Answer:**

- **Fixed window**: Allow N requests per time window (e.g., 100 requests per minute, resets every minute)
- **Sliding window**: More smooth distribution, counts requests from the last N seconds
- **Token bucket**: Clients get tokens that replenish over time, each request consumes a token
- **Leaky bucket**: Requests queue up and are processed at a fixed rate
- Return `429 Too Many Requests` with a `Retry-After` header when limits are exceeded
- Use `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers to inform clients
- In ASP.NET Core, use the `AspNetCoreRateLimit` NuGet package or custom middleware

```csharp
// Custom simple rate limiting middleware
public class RateLimitMiddleware
{
    private static readonly ConcurrentDictionary<string, RateInfo> _clients = new();
    private const int MaxRequests = 100;
    private const int WindowSeconds = 60;

    public async Task InvokeAsync(HttpContext context)
    {
        var clientId = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        var now = DateTime.UtcNow;

        var info = _clients.AddOrUpdate(clientId,
            new RateInfo { Count = 1, WindowStart = now },
            (key, existing) =>
            {
                if ((now - existing.WindowStart).TotalSeconds > WindowSeconds)
                    return new RateInfo { Count = 1, WindowStart = now };
                existing.Count++;
                return existing;
            });

        context.Response.Headers.Add("X-RateLimit-Limit", MaxRequests.ToString());
        context.Response.Headers.Add("X-RateLimit-Remaining", (MaxRequests - info.Count).ToString());

        if (info.Count > MaxRequests)
        {
            context.Response.StatusCode = 429;
            context.Response.Headers.Add("Retry-After", WindowSeconds.ToString());
            return;
        }

        await _next(context);
    }
}
```

---

*This covers the essential topics for REST API design interviews in ASP.NET Core. Focus on understanding the "why" behind each principle, not just the "what". Interviewers value candidates who can explain trade-offs and make informed design decisions.*
