# Spring MVC

---

## What is Spring MVC?

**Spring MVC** is a web framework built on the Servlet API that implements the **Model-View-Controller** pattern. It provides a clean separation between presentation logic, business logic, and data, and is the foundation for building REST APIs and web applications in Spring.

### Request Lifecycle

The journey of an HTTP request through Spring MVC follows a well-defined path:

```
HTTP Request
    ↓
DispatcherServlet (Front Controller)
    ↓
HandlerMapping → determines which controller handles the request
    ↓
Controller (handles request, returns response)
    ↓
Interceptors → pre/post processing
    ↓
HandlerAdapter → executes the controller method
    ↓
ViewResolver (if returning view name) → resolves to View
    ↓
View → renders response (JSP, Thymeleaf, JSON, etc.)
    ↓
HTTP Response
```

### Core Components

- **`DispatcherServlet`** — The front controller that intercepts all incoming requests and delegates them to handlers. It is the entry point for the entire MVC processing flow, coordinating all other components.
- **`HandlerMapping`** — Maps incoming HTTP requests to appropriate controller methods based on URL patterns, HTTP methods, headers, and parameters. Multiple `HandlerMapping` implementations can be chained.
- **`Controller`** — Handles the request, executes business logic, and returns a model or response body. Annotated with `@Controller` or `@RestController`.
- **`HandlerAdapter`** — Adapts different handler types (controllers, `HttpRequestHandler`, etc.) to the framework. It knows how to invoke the specific handler without coupling the framework to handler implementations.
- **`ViewResolver`** — Resolves logical view names (e.g., `"home"`) to actual view implementations (e.g., `home.jsp` or `home.html`). Supports prefix/suffix configuration.
- **`View`** — Renders the response, whether as HTML, JSON, XML, or other formats. The final step in the MVC flow.

### Key Annotations

- **`@Controller`** — Marks a class as an MVC controller. Methods can return view names or `ModelAndView` for server-side rendering.
- **`@RestController`** — Combination of `@Controller` and `@ResponseBody`. Every method writes directly to the HTTP response body (typically JSON). Use for REST APIs.
- **`@RequestMapping`** — Maps HTTP methods and paths to controller methods. Can be applied at class level (root path) and method level. Supports narrowing by headers, params, and media types.
- **`@GetMapping`**, **`@PostMapping`**, **`@PutMapping`**, **`@DeleteMapping`**, **`@PatchMapping`** — Shortcut annotations for specific HTTP methods. More readable than `@RequestMapping`.
- **`@RequestParam`** — Binds query parameters to method arguments. Supports default values and required flags.
- **`@PathVariable`** — Binds URL path segments to method arguments. Used for resource identifiers like `/orders/{id}`.
- **`@RequestBody`** — Deserializes the HTTP request body to a Java object using an `HttpMessageConverter`.
- **`@RequestHeader`** — Binds HTTP headers to method arguments. Useful for extracting auth tokens, content types, etc.
- **`@ResponseBody`** — Writes method return value directly to the HTTP response body using an `HttpMessageConverter`.
- **`@ResponseStatus`** — Sets the HTTP status code for the response. Use for fixed status codes like 201 Created.

---

## Core Concepts

### RESTful Controller Example

A typical REST API controller following best practices:

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping
    public ResponseEntity<List<Order>> getAll(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        PageResult<Order> result = orderService.findAll(page, size);
        return ResponseEntity.ok()
            .header("X-Total-Count", String.valueOf(result.total()))
            .body(result.items());
    }

    @GetMapping("/{id}")
    public ResponseEntity<Order> getById(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Order create(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.create(request);
    }

    @PutMapping("/{id}")
    public Order update(@PathVariable Long id,
                        @Valid @RequestBody UpdateOrderRequest request) {
        return orderService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        orderService.delete(id);
    }
}
```

### Content Negotiation

Spring automatically negotiates the response format based on:
- **`Accept` header** — Client specifies the desired content type.
- **URL suffix** — e.g., `/users.json`, `/users.xml`.
- **Query parameter** — e.g., `?format=json`.

```java
@GetMapping(value = "/users/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}
```

### Global Exception Handler

Handle exceptions consistently across all controllers:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> new FieldError(e.getField(), e.getDefaultMessage()))
            .toList();
        return new ErrorResponse("VALIDATION_ERROR", "Validation failed", errors);
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex) {
        log.error("Unhandled exception", ex);
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
}
```

### CORS Configuration

Cross-Origin Resource Sharing configuration for frontend access:

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://frontend.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

### Interceptor

Interceptors allow pre/post processing of requests:

```java
@Component
public class RequestLoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) {
        request.setAttribute("startTime", System.currentTimeMillis());
        MDC.put("requestId", UUID.randomUUID().toString());
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) {
        long startTime = (Long) request.getAttribute("startTime");
        long duration = System.currentTimeMillis() - startTime;
        log.info("{} {} completed in {}ms with status {}",
            request.getMethod(), request.getRequestURI(),
            duration, response.getStatus());
    }
}
```

### Async Request Processing

For long-running operations without blocking the request thread:

```java
@GetMapping("/async")
public DeferredResult<String> asyncProcess() {
    DeferredResult<String> result = new DeferredResult<>(5000L); // 5s timeout

    executor.submit(() -> {
        try {
            Thread.sleep(2000);
            result.setResult("Done!");
        } catch (Exception e) {
            result.setErrorResult("Failed");
        }
    });

    result.onTimeout(() -> result.setErrorResult("Timeout"));
    return result;
}
```

---

## Common Mistakes

- **Forgetting `@ResponseBody` or not using `@RestController`** — Methods return view names instead of JSON, causing `404` or template resolution errors. This *looks correct* because the method compiles and returns an object — the framework error message (404) does not mention the missing `@ResponseBody`, making the root cause hard to identify. Always use `@RestController` for REST APIs.
- **Returning entities directly** — Circular JSON references (`@JsonBackReference` / `@JsonManagedReference` needed) and over-fetching cause serialization errors. This *looks correct* because returning an entity works during development with small datasets — the `LazyInitializationException` only appears in production when the Hibernate session closes before Jackson serializes the response. Use DTOs instead of entities in controller responses.
- **Not using `@Valid` or `@Validated` on request bodies** — Invalid input passes through to the service layer without validation. This *looks correct* because the DTO has `@NotNull` annotations, and the controller method accepts it — developers expect validation to happen automatically, but Spring only activates it when `@Valid` is present on the parameter. Always validate at the controller boundary with `@Valid`.
- **Exposing internal IDs in URLs** — Sequential IDs in paths are predictable and pose a security risk. This *looks correct* because auto-increment IDs are natural identifiers, and the endpoint works perfectly — the security risk is invisible until a malicious actor enumerates IDs. Consider using UUIDs for public-facing resources.
- **Inconsistent error handling** — Each controller returns different error formats, making API clients harder to build. This *looks correct* because each developer handles errors in their controller, and every endpoint works correctly when there are no errors — the problem only surfaces when API clients must parse multiple error formats. Use `@RestControllerAdvice` for consistent error responses across all endpoints.
- **Not setting response status codes** — All successful responses return 200 by default. This *looks correct* because 200 OK is a valid HTTP status, and the client receives the data — the lack of `201 Created` or `204 No Content` is invisible to developers testing with browser dev tools. Use `@ResponseStatus` or `ResponseEntity` for correct and meaningful status codes.
- **Missing CORS configuration for frontend access** — The browser blocks cross-origin requests without proper CORS headers. This *looks correct* because backend-to-backend calls and curl requests work fine — the CORS error only appears in the browser's JavaScript console, which backend developers may not check. Configure `WebMvcConfigurer` with `addCorsMappings` for frontend access.
- **Large request bodies without size limits** — Can cause out-of-memory errors. This *looks correct* because the application handles small payloads without issues during testing — the OOM only occurs under attack or with a large legitimate upload in production. Set `spring.servlet.multipart.max-request-size` and use `@Size` on request DTOs to enforce limits.

---

## Real-World Scenarios

### Scenario 1: REST API Versioning for a Breaking Change

Your mobile API has a `GET /api/orders` endpoint that returns orders. The frontend team needs a new field `fulfillmentStatus` added, but changing the existing `status` field would break v1 clients. You need to support both versions simultaneously for 6 months.

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {
    @GetMapping
    public List<OrderV1> getAll() { /* Returns orders with original status field */ }
}

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 {
    @GetMapping
    public List<OrderV2> getAll() { /* Returns orders with new fulfillmentStatus */ }
}
```

Route v1 clients to `/api/v1/orders` and v2 clients to `/api/v2/orders`. After the migration period, remove v1 controller and redirect v1 URLs to v2.

### Scenario 2: File Upload Service with Progress Tracking

A document management system allows users to upload large PDFs (up to 500MB). Users need upload progress and must receive clear error messages for oversized or invalid files.

```java
@RestController
@RequestMapping("/api/v1/documents")
public class DocumentController {
    @PostMapping("/upload")
    public ResponseEntity<UploadResponse> uploadFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam("documentType") String type) {
        if (file.isEmpty()) {
            return ResponseEntity.badRequest()
                .body(new UploadResponse("error", "File is empty"));
        }
        if (file.getSize() > 500_000_000) {
            return ResponseEntity.status(PAYLOAD_TOO_LARGE)
                .body(new UploadResponse("error", "File exceeds 500MB limit"));
        }
        Document doc = documentService.store(file, type);
        return ResponseEntity.ok(new UploadResponse("success", doc.getId().toString()));
    }

    @GetMapping("/upload/progress/{uploadId}")
    public ResponseEntity<UploadProgress> getProgress(@PathVariable String uploadId) {
        return ResponseEntity.ok(uploadService.getProgress(uploadId));
    }
}
```

### Scenario 3: Request Logging Interceptor for Compliance

A fintech application must log every API request and response (method, URI, status, duration, user ID) for regulatory compliance. The log must include the complete request and response body (for non-sensitive endpoints).

```java
@Component
public class AuditLogInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) {
        if (handler instanceof HandlerMethod hm) {
            AuditContext ctx = new AuditContext(
                UUID.randomUUID().toString(),
                request.getMethod(),
                request.getRequestURI(),
                Instant.now()
            );
            request.setAttribute("auditCtx", ctx);
            MDC.put("requestId", ctx.requestId());
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        AuditContext ctx = (AuditContext) request.getAttribute("auditCtx");
        if (ctx != null) {
            log.info("AUDIT: method={} uri={} status={} duration={}ms user={}",
                ctx.method(), ctx.uri(), response.getStatus(),
                Duration.between(ctx.startTime(), Instant.now()).toMillis(),
                SecurityContextHolder.getContext().getAuthentication()?.getName());
        }
        MDC.remove("requestId");
    }
}
```

---

## Scenario-Based Questions

1. **Q: You are building a REST API for an e-commerce platform. A `GET /api/orders` endpoint that previously returned 20 fields now needs to return 35 fields. Some are N+1-loaded from lazy associations. The response time has grown from 200ms to 4s. How do you fix this without breaking existing clients?**
   A: First, create a DTO projection that only selects the fields actually needed for the response — stop returning JPA entities directly. Use `@JsonView` to define different views for different client versions:
   ```java
   public class OrderViews {
       public static class Basic {}
       public static class Detailed extends Basic {}
   }
   @JsonView(OrderViews.Basic.class)
   @GetMapping("/orders")
   public List<OrderDTO> getOrders() { ... }
   ```
   Second, add pagination if not present. Third, enable response compression. Fourth, add specific `@EntityGraph` or `JOIN FETCH` queries to eliminate N+1. Clients that need only the basic view get the 200ms response; clients that opt into detailed view accept the 4s cost or use async patterns.

2. **Q: Your `@RestControllerAdvice` handles validation errors, but the error response format is different from the rest of your team's controllers. Some endpoints return `{"error": "message"}`, others return `{"code": 400, "details": "..."}`. How do you enforce a single format?**
   A: Create a shared `ErrorResponse` record and mandate its use across all controllers via the `@RestControllerAdvice`:
   ```java
   public record ErrorResponse(String code, String message, Instant timestamp,
                               String path, Map<String, List<String>> fieldErrors) {}
   ```
   Then set `server.error.include-message=never` to disable Spring's default error responses. Configure `@RestControllerAdvice` to handle ALL exception types and always return `ErrorResponse`. Use an ArchUnit test that fails if any controller returns `ResponseEntity` with an inline body instead of `ErrorResponse`.

3. **Q: You have a controller method that accepts a `@RequestBody` and a `@RequestParam`. When `@RequestParam` is missing, the client gets a 400 with "Required request parameter is missing". You want to return your custom `ErrorResponse` format instead. How?**
   A: Handle `MissingServletRequestParameterException` in your `@RestControllerAdvice`:
   ```java
   @ExceptionHandler(MissingServletRequestParameterException.class)
   @ResponseStatus(HttpStatus.BAD_REQUEST)
   public ErrorResponse handleMissingParam(MissingServletRequestParameterException ex) {
       return new ErrorResponse("MISSING_PARAM",
           "Required parameter '" + ex.getParameterName() + "' of type " +
           ex.getParameterType() + " is missing",
           Instant.now(), request.getRequestURI(), null);
   }
   ```
   Also handle `ServletRequestBindingException` as a parent for other binding errors. This ensures your custom format covers all parameter-related errors.

   > **Interview follow-up:** The candidate added `MissingServletRequestParameterException` to the global handler. If the application has 50 endpoints and a new developer forgets to handle a different binding exception (e.g., `TypeMismatchException`), the client gets Spring's default 400 response instead of the custom format. How would you ensure ALL HTTP 400 errors use the custom format, including ones you haven't explicitly handled?

4. **Q: Your API uses `@RequestParam(defaultValue = "0") int page` for pagination. A client sends `page=-1` and receives a 200 with no data. How do you validate this properly?**
   A: Add `@Validated` at the controller class level and use validation annotations on parameters:
   ```java
   @RestController
   @Validated
   public class OrderController {
       @GetMapping("/orders")
       public List<Order> getOrders(
               @RequestParam @Min(0) int page,
               @RequestParam @Min(1) @Max(100) int size) {
           // ...
       }
   }
   ```
   Then handle `ConstraintViolationException` in your `@RestControllerAdvice` to return a structured error. `page=-1` will now return 400 with a clear message instead of silently returning empty results.

5. **Q: Your team has a custom `HandlerInterceptor` that measures request duration. A developer notices that `postHandle` is not called for requests that result in a 401 Unauthorized. Why?**
   A: Security filter exceptions cause the request to fail before the handler method executes. The `HandlerInterceptor.postHandle()` is only called after the handler successfully executes. For requests rejected by Spring Security filters, neither `preHandle`, `postHandle`, nor even the controller method are called. Use `afterCompletion()` instead — it's always called regardless of success or failure:
   ```java
   @Override
   public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                               Object handler, Exception ex) {
       long duration = System.currentTimeMillis() - (Long) request.getAttribute("startTime");
       log.info("{} {} -> {} ({}ms)", request.getMethod(), request.getRequestURI(),
           response.getStatus(), duration);
   }
   ```

6. **Q: Your REST API exposes a `PATCH /api/orders/{id}` for partial updates. Some clients send the full object (10 fields), others send only 2 fields. The service layer throws validation errors for null fields that the client didn't intend to change. How do you implement partial updates correctly?**
   A: Use `@JsonView` or a `Map<String, Object>` approach. The cleanest pattern is to accept a `Map` and apply only the present fields via Jackson's `updateValue`:
   ```java
   @PatchMapping("/{id}")
   public Order partialUpdate(@PathVariable Long id,
                              @RequestBody Map<String, Object> updates) {
       Order existing = orderService.findById(id);
       objectMapper.updateValue(existing, updates);
       return orderService.save(existing);
   }
   ```
   Or, use a dedicated partial update DTO where all fields are nullable and the service merges non-null values into the existing entity. This avoids triggering validation on fields the client didn't send.

7. **Q: You need to implement a request rate limiter per API key. A naive approach adds a `synchronized` block in the controller — now throughput is 10 req/s instead of 1000 req/s. How do you implement this efficiently?**
   A: Never rate-limit in the controller — use a `HandlerInterceptor` with a non-blocking rate limiter:
   ```java
   @Component
   public class RateLimitingInterceptor implements HandlerInterceptor {
       private final Cache<String, RateLimiter> limiters = Caffeine.newBuilder()
           .expireAfterAccess(1, TimeUnit.HOURS)
           .build();

       @Override
       public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                                Object handler) {
           String apiKey = request.getHeader("X-Api-Key");
           RateLimiter limiter = limiters.get(apiKey, k -> RateLimiter.create(100.0));
           if (!limiter.tryAcquire()) {
               response.setStatus(429);
               response.setHeader("Retry-After", "1");
               return false;
           }
           return true;
       }
   }
   ```
   Use Guava's `RateLimiter` or Bucket4j for thread-safe, non-blocking rate limiting. This keeps the controller clean and the rate limiter out of the business logic.

8. **Q: A junior developer writes `@GetMapping("/orders")` returning `List<Order>` (JPA entity). The entity has `@OneToMany` lazy associations. Jackson serialization triggers `LazyInitializationException` for every order. How do you fix this at the architecture level?**
   A: Ban returning JPA entities from controllers entirely. Enforce a policy that all controller methods return DTOs. Use MapStruct or manual mapping in the service layer:
   ```java
   @GetMapping("/orders")
   public List<OrderResponse> getOrders() {
       return orderService.findAll().stream()
           .map(orderMapper::toResponse)
           .toList();
   }
   ```
   The DTO only includes the fields that should be serialized. No lazy-loading surprises, no circular JSON references, no over-fetching. Add an ArchUnit test to enforce `@RestController` methods never return entity types.

   > **Interview follow-up:** The candidate suggested banning entities from controllers with ArchUnit. If a `@RestController` method returns `ResponseEntity<Order>` (wrapping the entity inside `ResponseEntity`), a naive ArchUnit rule checking the return type of the method may miss it. How would you write an ArchUnit rule that catches entities wrapped in `ResponseEntity`, `CompletableFuture`, or `Mono`?

9. **Q: You need to accept both `application/json` and `application/xml` requests, and return responses in the same format the client sent. Some endpoints should only support JSON. How do you configure this?**
   A: Configure content negotiation globally, then restrict specific endpoints:
   ```yaml
   spring:
     mvc:
       contentnegotiation:
         favor-parameter: false
         favor-path-extension: false
         media-types:
           json: application/json
           xml: application/xml
   ```
   Add `jackson-dataformat-xml` to the classpath for XML support. For JSON-only endpoints, use:
   ```java
   @GetMapping(value = "/orders", produces = MediaType.APPLICATION_JSON_VALUE)
   public List<Order> getOrders() { ... }
   ```
   Spring will respond with 406 Not Acceptable if a client requests XML from a JSON-only endpoint.

10. **Q: You have a controller that returns `ResponseEntity<Resource<ByteArrayResource>>` for file downloads. Memory usage spikes to 500MB when users download large files. How do you stream large files without loading them into memory?**
    A: Use `StreamingResponseBody` or `Resource` with `InputStreamResource`:
    ```java
    @GetMapping("/files/{id}")
    public ResponseEntity<StreamingResponseBody> downloadFile(@PathVariable Long id) {
        File file = fileService.getFile(id);
        return ResponseEntity.ok()
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .header("Content-Disposition", "attachment; filename=\"" + file.getName() + "\"")
            .contentLength(file.length())
            .body(outputStream -> {
                try (InputStream is = new FileInputStream(file)) {
                    is.transferTo(outputStream);
                }
            });
    }
    ```
    `StreamingResponseBody` writes directly to the HTTP response output stream without buffering the entire file in memory. Set `server.servlet.session.timeout` appropriately for large downloads.

---

## Interview Questions

1. **What is the DispatcherServlet and what is its role?** 
   A: The `DispatcherServlet` is the front controller in Spring MVC. It intercepts all incoming HTTP requests and delegates them to the appropriate components: `HandlerMapping` to find the controller, `HandlerAdapter` to execute it, and `ViewResolver` to render the response. It is the entry point for the entire MVC request processing pipeline.

2. **What is the difference between `@Controller` and `@RestController`?** 
   A: `@Controller` marks a class as an MVC controller whose methods return view names (JSP, Thymeleaf). `@RestController` is a convenience annotation that combines `@Controller` and `@ResponseBody` — every method writes directly to the HTTP response body (typically JSON). Use `@RestController` for REST APIs.

3. **What is the request lifecycle in Spring MVC?** 
   A: HTTP Request → `DispatcherServlet` → `HandlerMapping` (determines controller) → Interceptors (`preHandle`) → `HandlerAdapter` → Controller method → Interceptors (`postHandle`) → `ViewResolver` (if returning view name) → View rendering → Interceptors (`afterCompletion`) → HTTP Response.

4. **What is the difference between `@RequestParam` and `@PathVariable`?** 
   A: `@RequestParam` extracts query parameters (`/api/users?role=admin`). `@PathVariable` extracts URI path segments (`/api/users/123`). Use `@PathVariable` for resource identifiers and `@RequestParam` for filtering, sorting, and pagination.

5. **How do you handle file uploads in Spring MVC?** 
   A: Use `MultipartFile` as a controller method parameter. Configure limits in `application.yml`: `spring.servlet.multipart.max-file-size=10MB`. Handle `MultipartException` in `@RestControllerAdvice`. For large files, use streaming with `InputStreamResource` or `StreamingResponseBody`.

6. **What is a `HandlerInterceptor` and how do you use it?** 
   A: `HandlerInterceptor` provides pre-processing (`preHandle`), post-processing (`postHandle`), and completion callbacks (`afterCompletion`) for requests. Implement `HandlerInterceptor` and register it via `WebMvcConfigurer.addInterceptors()`. Use for logging, authentication checks, rate limiting, or request timing.

7. **How do you implement CORS in Spring MVC?** 
   A: Configure a `WebMvcConfigurer` bean that overrides `addCorsMappings()`:
   ```java
   @Configuration
   public class CorsConfig implements WebMvcConfigurer {
       @Override
       public void addCorsMappings(CorsRegistry registry) {
           registry.addMapping("/api/**").allowedOrigins("https://frontend.com");
       }
   }
   ```
   Or use `@CrossOrigin` on specific controllers or methods.

8. **What is content negotiation in Spring MVC?** 
   A: Content negotiation determines the response format (JSON, XML, HTML) based on the `Accept` header, URL suffix, or query parameter. Spring automatically selects the appropriate `HttpMessageConverter`. Add `jackson-dataformat-xml` for XML support.

9. **How do you validate request bodies in Spring MVC?** 
   A: Annotate the request body parameter with `@Valid` or `@Validated` and add validation annotations (`@NotBlank`, `@NotNull`, `@Size`) on the DTO fields. Handle `MethodArgumentNotValidException` in `@RestControllerAdvice` to return structured field-level errors.

10. **What is `ResponseEntity` and when should you use it?** 
    A: `ResponseEntity` represents the full HTTP response — status code, headers, and body. Use it when you need to: (a) set a non-default status code, (b) add custom headers (pagination, caching), or (c) conditionally return different statuses. For simple cases, `@ResponseStatus` is sufficient.

---

## Developer Recommendations

- **Use DTOs instead of entities in controller responses** — Returning JPA entities directly causes lazy-loading exceptions, circular JSON references, and over-fetching. DTOs are explicit about what gets serialized, prevent `LazyInitializationException`, and decouple the API contract from the database model. Use MapStruct for mapping.
- **Prefer `@RestController` over `@Controller` for REST APIs** — `@Controller` requires `@ResponseBody` on every method, which is easily forgotten and causes 404 errors (Spring tries to resolve a view name). `@RestController` defaults all methods to response body mode and is unambiguous.
- **Use `ResponseEntity` for fine-grained control over status codes and headers** — `@ResponseStatus` is fine for fixed status codes, but `ResponseEntity` lets you conditionally set status, add caching headers, pagination headers (`X-Total-Count`), and content disposition for file downloads in a fluent API.
- **Always add `@Valid` or `@Validated` to `@RequestBody` parameters** — Without it, validation annotations on DTOs are silently ignored. The method receives an invalid object without any error. Always handle `MethodArgumentNotValidException` in `@RestControllerAdvice`.
- **Use `HandlerInterceptor` for cross-cutting concerns, not controllers** — Logging, rate limiting, authentication checks, and request timing in every controller method is repetitive and error-prone. `HandlerInterceptor` centralizes these concerns. Remember `afterCompletion()` for cleanup that must run even on errors.
- **Use `@ExceptionHandler` in `@RestControllerAdvice` for consistent error responses** — Without a global handler, every controller may return errors in different formats. Centralize all exception handling in one place. Handle specific exceptions before generic ones.
- **Configure CORS explicitly for production** — Never use `allowedOrigins("*")` in production — it opens the door for CSRF and data theft. Specify exact origins. If you need to support multiple origins dynamically, implement a `CorsConfigurationSource` that reads from a configuration source.
- **Use `StreamingResponseBody` for large file downloads** — Returning `byte[]` or `ByteArrayResource` for large files loads the entire file into heap memory, causing OOM under concurrency. `StreamingResponseBody` streams directly to the response output stream with minimal memory footprint.
