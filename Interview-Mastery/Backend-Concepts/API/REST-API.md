# REST API

---

## Overview

**REST (Representational State Transfer)** is an architectural style for designing distributed systems, introduced by Roy Fielding in his 2000 doctoral dissertation at UC Irvine. REST defines a set of six constraints that guide how web services should behave, treating server resources as entities that can be created, read, updated, and deleted via standard HTTP methods. The key insight of REST is that by following these constraints, you build systems that are scalable, evolvable, and visible to intermediaries like caches and proxies. REST has become the dominant architectural style for web APIs because it leverages the existing HTTP infrastructure, is simple to understand and implement, and works with virtually any programming language or platform.

**Why It Exists:** Before REST, web APIs were largely unstructured and inconsistent. SOAP (Simple Object Access Protocol) was the dominant standard, but it was heavyweight, relied on XML exclusively, and required strict contract-first development with WSDL definitions. REST provided a simpler, more flexible alternative using existing HTTP semantics. By leveraging standard HTTP methods (GET, POST, PUT, DELETE, PATCH), status codes (200, 201, 404, 500), and headers, REST made APIs more intuitive and reduced the learning curve for new developers. The stateless constraint simplified server implementations and improved scalability, while the cacheable constraint enabled performance optimizations that were difficult with earlier approaches. Today, REST is the default choice for most public APIs, and its principles influence even non-RESTful designs.

**Six Constraints:**
- **Uniform Interface** — Resources are identified in requests (URIs), representations are self-descriptive (media types), and HATEOAS (Hypermedia as the Engine of Application State) enables discoverability. This constraint simplifies the overall system architecture and improves visibility of interactions. The uniform interface is what distinguishes REST from other network-based architectures — it decouples clients from servers by standardizing the communication contract.
- **Stateless** — Each request from a client contains all the information needed by the server to process that request. The server does not store any client context between requests. Session state is kept entirely on the client. This constraint improves scalability (any server can handle any request), reliability (no session replication needed), and visibility (intermediaries can inspect requests without session context). The trade-off is increased request size (more data per request) and the need for clients to manage their own session state.
- **Cacheable** — Responses must implicitly or explicitly define themselves as cacheable or non-cacheable. A well-cached API can eliminate many client-server interactions, improving performance and reducing server load. Cache directives via `Cache-Control`, `Expires`, and `ETag` headers give clients and intermediaries clear guidance on caching behavior. The trade-off is potential staleness — cached data may become outdated, requiring careful TTL management and invalidation strategies.
- **Client-Server** — Separation of concerns allows the client and server to evolve independently. The client doesn't need to know about data storage, and the server doesn't need to know about the user interface. This simplifies server implementation, improves portability of the client across platforms, and allows the server to scale independently of the client. This constraint is fundamental to the web's success — browsers (clients) and web servers evolve at different rates.
- **Layered System** — Intermediaries like load balancers, caches, and proxies can be inserted transparently between clients and servers. Each layer only interacts with adjacent layers, and no layer has knowledge of layers beyond its immediate neighbors. This enables load balancing, shared caches, security enforcement, and other cross-cutting concerns without modifying the application. The constraint improves system scalability and security but can add latency overhead from passing through multiple layers.
- **Code on Demand (optional)** — Servers can extend client functionality by transferring executable code, such as JavaScript applets or scripts. This is the only optional constraint and is rarely used in API design today. It allows clients to be extended with new capabilities after deployment, reducing the number of features that must be pre-implemented. The trade-off is reduced visibility (intermediaries can't easily inspect executable code) and potential security concerns.

---

## HTTP Methods & Status Codes

**HTTP Methods and Their Semantics:**
- **GET** — Retrieve a resource representation. GET requests are idempotent (multiple identical requests produce the same result) and safe (no side effects on the server). GET responses are cacheable by default. GET should never modify server state. Use GET for reading data, retrieving search results, and fetching resource representations.
- **POST** — Submit an entity to a resource, often causing a state change or side effect on the server. POST is not idempotent — submitting the same POST request multiple times may create multiple resources or trigger multiple actions. Use POST for creating resources, submitting form data, performing actions that don't fit other methods, and anything with side effects.
- **PUT** — Replace the target resource with the request payload. PUT is idempotent — sending the same PUT request N times has the same effect as sending it once (the resource ends up in the same state). PUT requires the client to send the complete resource representation. Use PUT for full updates where the client knows the resource identity.
- **PATCH** — Apply partial modifications to a resource. PATCH is not necessarily idempotent (applying the same patch twice may produce different results depending on the patch format). Use PATCH for partial updates where the client only sends changed fields. JSON Merge Patch (RFC 7396) and JSON Patch (RFC 6902) are standard patch formats.
- **DELETE** — Remove the specified resource. DELETE is idempotent — deleting an already-deleted resource typically returns 404 or 410 but does not change server state. DELETE may trigger cascading deletion of related resources.
- **HEAD** — Identical to GET but returns only response headers, no response body. HEAD is idempotent and safe. Useful for checking resource existence, metadata, cache validity, and content length before making a full GET request.
- **OPTIONS** — Returns the HTTP methods supported by the target resource. OPTIONS is idempotent and safe. Used for CORS preflight requests and API discovery. The response's `Allow` header lists supported methods.

**Status Code Families:**
- **1xx (Informational):** Request received, continuing to process. 100 Continue (client should continue sending body), 101 Switching Protocols (used for WebSocket upgrade). Rarely seen in REST APIs.
- **2xx (Success):** The action was successfully received, understood, and accepted. 200 OK (standard success for GET, PUT, PATCH), 201 Created (resource created via POST — include Location header), 202 Accepted (request accepted but processing not complete — used for async operations), 204 No Content (success with no body — used for DELETE).
- **3xx (Redirection):** Further action needed. 301 Moved Permanently (resource has new URI, update links), 304 Not Modified (cached version is still valid — used with ETag/If-None-Match), 307 Temporary Redirect (temporary URI, keep using original).
- **4xx (Client Error):** The request contains bad syntax or cannot be fulfilled. 400 Bad Request (malformed syntax, validation errors), 401 Unauthorized (authentication required or failed), 403 Forbidden (authenticated but not authorized), 404 Not Found (resource doesn't exist), 405 Method Not Allowed (wrong HTTP method), 409 Conflict (resource state conflict — e.g., version mismatch), 410 Gone (resource permanently removed), 415 Unsupported Media Type (wrong Content-Type), 422 Unprocessable Entity (semantic validation errors), 429 Too Many Requests (rate limit exceeded).
- **5xx (Server Error):** The server failed to fulfill an apparently valid request. 500 Internal Server Error (unexpected server failure), 502 Bad Gateway (upstream server returned invalid response), 503 Service Unavailable (temporarily overloaded or down for maintenance), 504 Gateway Timeout (upstream server timed out).

---

## Request Processing Pipeline (Spring Boot)

**Flow:**
1. An embedded servlet container (Tomcat, Jetty, or Undertow) accepts the TCP connection on port 8080 (or configured port). The container's acceptor thread picks up the connection and hands it to a worker thread from the Tomcat thread pool (default 200).
2. The request passes through the servlet filter chain configured in Spring Security and application filters. Filters handle cross-cutting concerns: security authentication (JWT/OAuth2 validation, session management), request logging (request ID generation, MDC population), CORS validation (origin checking, preflight handling), request wrapping (XSS sanitization, decompression), and rate limiting (token bucket check). Filters execute in order and can short-circuit the chain by returning a response directly (e.g., authentication failure returns 401).
3. `DispatcherServlet` consults `HandlerMapping` implementations to find the matching `@RequestMapping` method. HandlerMapping evaluates: `@RequestMapping` URI patterns (path matching with variables), HTTP method matching (GET vs POST), request parameters and headers condition, `Accept` header content negotiation for `produces` matching, and custom conditions (tenant resolution, feature flags).
4. `HandlerAdapter` invokes the matched controller method after argument resolution. Spring resolves method parameters from the request: `@PathVariable` from URI template, `@RequestParam` from query string, `@RequestBody` via HttpMessageConverter deserialization, `@RequestHeader` from HTTP headers, `@ModelAttribute` from form data or query parameters, `Principal` from security context, `BindingResult` for validation errors. Spring also handles: automatic validation via `@Valid`/`@Validated` annotations (throws MethodArgumentNotValidException), parameter conversion (String to Long, etc.), and custom argument resolvers.
5. The controller method executes business logic and returns a response entity or domain object. The method can return: `ResponseEntity<T>` (full control over status, headers, body), `@ResponseBody` annotated return value (auto-serialized), `ModelAndView` (for server-rendered views), or `void` with response writing.
6. `HttpMessageConverter` serializes the response body. Spring Boot auto-configures: Jackson for JSON (MappingJackson2HttpMessageConverter — configures ObjectMapper with date formats, naming strategies, serialization features), JAXB/Jackson XML for XML (if jackson-dataformat-xml is on classpath), StringHttpMessageConverter for plain text, ByteArrayHttpMessageConverter for binary data. Content negotiation selects the converter based on `Accept` header and `produces` annotation.
7. The response travels back through the filter chain (invoking after-request filters), then through the servlet container's response processing (compression if configured), and finally over the network to the client. The entire pipeline is synchronous by default but can be made asynchronous using `DeferredResult`, `Callable`, or Spring WebFlux.

**Content Negotiation:**
- The `Accept` header specifies the desired response format (e.g., `Accept: application/json`, `Accept: application/xml`, `Accept: application/vnd.myapp.v1+json` for versioning). Multiple types with quality factors are supported (`Accept: text/html;q=0.9, application/json;q=0.8`).
- The `Content-Type` header specifies the request body format (e.g., `Content-Type: application/json` for POST/PUT/PATCH requests). The server rejects requests with unsupported content types (415 Unsupported Media Type).
- Spring Boot handles negotiation via `ContentNegotiationManager`, which supports multiple strategies: Accept header (default), query parameter (`?format=json`), path extension (`.json`, `.xml`), and fixed content type. Configuration via `spring.mvc.contentnegotiation.*` properties or `WebMvcConfigurer.configureContentNegotiation()`.
- Custom media types enable API versioning: `application/vnd.myapp.v1+json`, `application/vnd.myapp.v2+json`. The controller uses `produces = "application/vnd.myapp.v1+json"` on specific methods.

---

## Production Code Examples

### Spring Boot REST Controller with Full Error Handling

```java
@RestController
@RequestMapping("/api/v1/users")
@Slf4j
public class UserController {
    // Inject the service layer dependency via constructor injection
    // Constructor injection is preferred over field injection for testability
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }

    /**
     * GET /api/v1/users — Retrieve paginated list of users.
     * Supports sorting, pagination, and returns a page wrapper.
     * Default page=0, size=20, sort=id,asc
     */
    @GetMapping
    @ResponseStatus(HttpStatus.OK)
    public Page<UserResponse> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id,asc") String[] sort) {
        // Parse the sort parameter: "id,asc" → sort by id ascending
        Sort sorting = Sort.by(
            sort[1].equalsIgnoreCase("desc") ? Sort.Direction.DESC : Sort.Direction.ASC,
            sort[0]);
        Pageable pageable = PageRequest.of(page, size, sorting);
        return userService.findAll(pageable).map(UserMapper::toResponse);
    }

    /**
     * GET /api/v1/users/{id} — Retrieve single user by ID.
     * Returns 404 if user not found (via service exception).
     */
    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return UserMapper.toResponse(userService.findById(id));
    }

    /**
     * POST /api/v1/users — Create a new user.
     * Returns 201 Created with the created user in the response body.
     * Request body validated via @Valid annotation.
     */
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        User user = userService.create(UserMapper.toEntity(request));
        return UserMapper.toResponse(user);
    }

    /**
     * PUT /api/v1/users/{id} — Full update/replace of user.
     * Idempotent: same request sent twice produces same result.
     */
    @PutMapping("/{id}")
    public UserResponse updateUser(
            @PathVariable Long id, 
            @Valid @RequestBody UpdateUserRequest request) {
        return UserMapper.toResponse(userService.update(id, UserMapper.toEntity(request)));
    }

    /**
     * DELETE /api/v1/users/{id} — Delete a user.
     * Returns 204 No Content — no response body.
     * Idempotent: deleting already deleted user returns 404.
     */
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### Global Exception Handler for Consistent Error Responses

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    /**
     * Handle ResourceNotFoundException — returns 404 with error details.
     * Example: User not found, Order not found.
     */
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    /**
     * Handle validation errors from @Valid annotations.
     * Returns 400 with field-level error details in a map.
     * Each field error includes the field name and default message.
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        return new ErrorResponse("VALIDATION_ERROR", "Validation failed", errors);
    }

    /**
     * Catch-all handler for unhandled exceptions.
     * Returns 500 with generic message — never expose stack traces.
     * Full exception is logged server-side for debugging.
     */
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex) {
        log.error("Unhandled exception", ex);
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
    
    /**
     * Handle HttpMessageNotReadableException (malformed JSON body).
     */
    @ExceptionHandler(HttpMessageNotReadableException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleMalformedJson(HttpMessageNotReadableException ex) {
        return new ErrorResponse("MALFORMED_JSON", "Request body is malformed or contains invalid data");
    }
    
    /**
     * Handle constraint violations (validation on path variables, query params).
     */
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleConstraintViolation(ConstraintViolationException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getConstraintViolations()
            .forEach(v -> errors.put(v.getPropertyPath().toString(), v.getMessage()));
        return new ErrorResponse("CONSTRAINT_VIOLATION", "Parameter validation failed", errors);
    }
}
```

### Request/Response DTOs with Java Records

```java
// Request DTO for creating a user — uses Jakarta Bean Validation annotations
// Records are immutable, concise, and ideal for DTOs
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be 2-100 characters")
    String name,
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    String email,
    
    @Pattern(regexp = "USER|ADMIN|MODERATOR", message = "Role must be USER, ADMIN, or MODERATOR")
    String role
) {}

// Response DTO — never expose internal entity directly
public record UserResponse(
    Long id, 
    String name, 
    String email, 
    String role, 
    Instant createdAt, 
    Instant updatedAt
) {}

// Standard error response format for all endpoints
public record ErrorResponse(
    String code, 
    String message, 
    Map<String, String> details
) {
    // Convenience constructor for simple errors without field-level details
    public ErrorResponse(String code, String message) {
        this(code, message, null);
    }
}
```

### REST Client with RestClient (Spring 6.1+) — Modern HTTP Client

```java
@Service
public class UserApiClient {
    private final RestClient restClient;
    private static final Logger log = LoggerFactory.getLogger(UserApiClient.class);

    /**
     * Configure RestClient with base URL, default headers, and logging interceptor.
     * RestClient is the modern replacement for RestTemplate (Spring 6.1+).
     */
    public UserApiClient(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            // Add request/response logging via interceptor
            .requestInterceptor((request, body, execution) -> {
                log.info("Request: {} {}", request.getMethod(), request.getURI());
                long start = System.currentTimeMillis();
                ClientResponse response = execution.execute(request, body);
                long duration = System.currentTimeMillis() - start;
                log.info("Response: {} ({}ms)", response.statusCode(), duration);
                return response;
            })
            .requestFactory(new JdkClientHttpRequestFactory()) // Java 11+ HTTP client
            .build();
    }

    /**
     * GET request with path variable and error handling.
     * Throws ApiClientException on 4xx responses.
     */
    public UserResponse getUser(Long id) {
        return restClient.get()
            .uri("/api/v1/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
                throw new ApiClientException("Client error: " + response.getStatusText());
            })
            .body(UserResponse.class);
    }

    /**
     * POST request with JSON body.
     * Automatically serializes CreateUserRequest to JSON.
     */
    public UserResponse createUser(CreateUserRequest request) {
        return restClient.post()
            .uri("/api/v1/users")
            .body(request)
            .retrieve()
            .body(UserResponse.class);
    }
    
    /**
     * PUT request for full update with ExchangeFilterFunction for logging.
     */
    public UserResponse updateUser(Long id, UpdateUserRequest request) {
        return restClient.put()
            .uri("/api/v1/users/{id}", id)
            .body(request)
            .retrieve()
            .body(UserResponse.class);
    }
    
    /**
     * DELETE request returning void.
     */
    public void deleteUser(Long id) {
        restClient.delete()
            .uri("/api/v1/users/{id}", id)
            .retrieve()
            .toBodilessEntity();
    }
}
```

### Pagination Pattern with Generic Wrapper

```java
/**
 * Generic paginated response wrapper.
 * Encapsulates content, pagination metadata, and navigation links.
 * @param <T> the type of content elements
 */
public record PagedResponse<T>(
    List<T> content,       // The page content
    int page,              // Current page number (0-based)
    int size,              // Page size
    long totalElements,    // Total number of elements across all pages
    int totalPages,        // Total number of pages
    boolean first,         // Is this the first page?
    boolean last,          // Is this the last page?
    String sort            // Sort specification
) {
    public static <T> PagedResponse<T> from(Page<T> page) {
        return new PagedResponse<>(
            page.getContent(), 
            page.getNumber(), 
            page.getSize(),
            page.getTotalElements(), 
            page.getTotalPages(), 
            page.isFirst(), 
            page.isLast(),
            page.getSort().toString()
        );
    }
}
```

---

## Common Mistakes

- **Using GET for mutations** — Using GET to perform actions that modify server state (e.g., `GET /deleteUser?id=123`) breaks HTTP semantics, makes caching unpredictable, and can be triggered accidentally by search engine crawlers or browser prefetching. Always use the correct HTTP method for the operation: GET for reads, POST for creates, PUT for full updates, PATCH for partial updates, DELETE for removals. This *looks correct* because: developers see URLs as action endpoints rather than resource identifiers, and GET is the simplest method to trigger from any tool, including a browser address bar.
- **Inconsistent error responses** — Returning different error shapes from different endpoints forces clients to parse error messages heuristically. Standardize on a single error format (like the `ErrorResponse` record above) across all endpoints. Include a machine-readable error code, a human-readable message, and optional field-level details. This allows clients to handle errors programmatically rather than parsing text. This *looks correct* because: each endpoint's error format seems naturally tailored to its specific operation, and the inconsistency only becomes painful when a client needs to parse errors from a dozen different endpoints programmatically.
- **Ignoring idempotency** — PUT and DELETE should be idempotent: sending the same request twice produces the same server state. POST should never be idempotent. If a POST request is received twice (due to retries), it should either be rejected (via idempotency key) or create a duplicate resource. The HTTP specification is clear about these semantics, and clients depend on them for correct behavior. This *looks correct* because: duplicate requests seem statistically unlikely in normal operation, and the consequences of double-processing aren't visible until a payment is charged twice or a duplicate order ships.
- **Exposing internal IDs** — Using auto-increment database IDs (1, 2, 3) in URLs allows clients to enumerate resources and guess valid IDs. Use UUIDs or other non-sequential identifiers for exposed resource IDs. UUIDs also simplify client-side ID generation and make it safe to expose in URLs without leaking information about data volume or ordering. This *looks correct* because: auto-increment integers are free, fast, and the default in every database — the enumeration risk only becomes obvious when an attacker scripts through your entire user base.
- **No pagination** — Returning unbounded lists is a performance disaster: a client requesting `/users` with 10 million records will crash the server (memory), waste network bandwidth, and provide a terrible user experience. Always paginate list endpoints from day one, even if the current dataset is small. Use cursor-based pagination for real-time feeds and offset-based for static datasets. This *looks correct* because: with a small initial dataset of a few hundred records, "we'll add pagination later" seems reasonable — until the user base grows and a single request suddenly loads hundreds of thousands of records.
- **Returning stack traces in production** — Exposing stack traces reveals your application internals, library versions, and potential security vulnerabilities. Always return generic error messages to clients and log the full details server-side. Use a global exception handler that catches Throwable and translates it to a sanitized error response. This *looks correct* because: stack traces are invaluable during development for rapid debugging, and developers may not realize they expose internal package names, file paths, and library versions to any client that sends a malformed request.
- **Not versioning APIs** — Without versioning, any breaking change immediately breaks all existing clients. Adding versioning later is painful because you must retroactively decide what "no version" means. Always version from day one with a simple prefix: `/api/v1/`. The cost is minimal (one URI segment) and the benefit is the ability to evolve the API without breaking clients. This *looks correct* because: when only one client exists and you control it, versioning feels like unnecessary overhead — until a second team integrates or a breaking change is urgently needed.
- **Incorrect HTTP status codes** — Returning 200 for a created resource (should be 201), returning 200 for a deleted resource (should be 204), or returning 500 for a client error (should be 4xx) confuses clients and breaks standards-based tooling. Status codes are part of your API contract — use them correctly. A simple rule: 2xx for success, 3xx for redirects, 4xx for client mistakes, 5xx for server failures. This *looks correct* because: the response body reaches the client regardless of the status code, so developers treat codes as a cosmetic detail rather than a machine-readable contract that automated clients depend on.
- **N+1 query problem** — The most common performance issue in REST APIs. A list endpoint fetches N parent entities and then makes N additional queries for related data. Fix with `JOIN FETCH`, `@EntityGraph`, or batch fetching. Monitor database query counts in production — sudden increases indicate N+1 regressions. This *looks correct* because: the code produces correct results in development with 5 test records, and the N+1 pattern only reveals itself under production-scale data volumes where each loop triggers a separate database round trip.
- **No input validation** — Never trust client input. Every field should be validated for type, format, length, range, and business rules. Use Bean Validation annotations (`@NotBlank`, `@Email`, `@Size`, `@Pattern`) on request DTOs. Sanitize string inputs to prevent XSS, SQL injection, and other injection attacks. Validate before any business logic executes. This *looks correct* because: the frontend already validates all inputs, so backend validation seems redundant — until someone calls the API directly with curl, Postman, or an automated script that bypasses the browser entirely.

---

## Key Design Considerations

**Richardson Maturity Model:**
- **Level 0: Swamp of POX (Plain Old XML)** — HTTP is used as a transport tunnel, usually with a single URI and single HTTP method (typically POST). All operations go through the same endpoint; the action is encoded in the request body. This is essentially RPC over HTTP, not REST. Many SOAP and XML-RPC services operate at this level.
- **Level 1: Resources** — Multiple URIs (one per resource) but still using a single HTTP method (usually GET). The server has different endpoints for different resources but hasn't adopted HTTP verbs. This is a step forward — resources are identifiable — but the API can't leverage HTTP semantics properly.
- **Level 2: HTTP Verbs** — Proper use of HTTP methods (GET, POST, PUT, DELETE) and status codes (200, 201, 204, 404). This is where most REST APIs operate. The API uses the full HTTP protocol to express intent, making it self-descriptive and cacheable. Level 2 is sufficient for most production APIs.
- **Level 3: Hypermedia Controls (HATEOAS)** — Responses include links to related actions and resources, enabling clients to discover the API dynamically. This is the highest maturity level and the least commonly implemented. Links tell clients what they can do next without out-of-band documentation. While powerful, HATEOAS adds complexity that most APIs don't need.

**Best Practices:**
- Use plural nouns for resources (`/users` not `/user`, `/orders` not `/getOrders`). Nouns represent resources; verbs represent actions and belong in HTTP methods. The URI identifies the resource; the HTTP method identifies the operation.
- Limit nested resources to 2-3 levels maximum (`/users/{id}/orders` is fine; `/users/{id}/orders/{orderId}/items/{itemId}/details` is not). Deep nesting creates complex URIs that are hard to navigate and cache. Consider flattening complex hierarchies with query parameters or composite endpoints.
- Implement query complexity limits to prevent expensive queries from degrading performance. Restrict the number of filterable fields, maximum page size, and depth of included relationships. Return 400 or 413 if a query is too complex.
- Use OpenAPI/Swagger for contract-first design. The OpenAPI specification serves as the single source of truth for your API contract. Generate server stubs and client SDKs from the spec. This ensures consistency between documentation and implementation.
- Design for graceful degradation with circuit breakers. A downstream service failure should not cascade to your API clients. Use Resilience4j or Spring Cloud Circuit Breaker to fail fast and return cached or degraded responses when dependencies are unhealthy.
- Log request IDs, trace IDs, and response times for observability. Generate a unique request ID for each incoming request, propagate it to downstream services, and include it in all log entries. This enables correlating logs across services for debugging.
- Never break existing clients; use additive changes. Add new fields to responses, add new optional parameters, and add new endpoints. Only create a new API version when additive changes are impossible. Mark deprecated fields with documentation and sunset headers.

---

## Real-World Scenarios

### Scenario 1: Idempotent Payment Processing with Distributed Locking
**Context:** Your e-commerce payment endpoint `POST /api/v1/orders/{id}/pay` charges the user's credit card through a third-party payment gateway. Network timeouts cause the mobile client to automatically retry the request. Without idempotency, each retry triggers a new charge against the user's card. Users are double-charged, support tickets surge, and the finance team demands an immediate fix. The problem is compounded because the payment gateway itself is eventually consistent — a 200 OK response might mean "received" not "processed," and the actual charge result arrives via webhook minutes later.

**Resolution:** Implement idempotency keys with a two-phase approach. The client generates a UUID (`Idempotency-Key` header) for each payment attempt. The server stores the key with the processing result in Redis (TTL: 24 hours with automatic extension for pending transactions). On the first request, create a pending record, process the payment, and update the record with the result. On retry with the same key, check Redis — if the key exists with a result, return the cached result without re-processing. If the key exists with a pending status, wait briefly for completion. If the key doesn't exist (expired), proceed with processing. This guarantees exactly-once processing even with retries and eventually consistent gateways.

```java
@PostMapping("/{orderId}/pay")
public ResponseEntity<PaymentResponse> payOrder(
        @PathVariable Long orderId,
        @RequestHeader("Idempotency-Key") UUID idempotencyKey,
        @Valid @RequestBody PaymentRequest request,
        @AuthenticationPrincipal User user) {
    // Phase 1: Check if this idempotency key was already processed
    // Use SET NX EX for atomic claim — only one instance gets to process
    String lockKey = "payment:idempotency:" + idempotencyKey;
    Boolean acquired = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "PENDING", Duration.ofMinutes(5));
    
    if (Boolean.FALSE.equals(acquired)) {
        // Another request is processing or already processed
        // Wait and return the result (or pending status)
        Optional<PaymentResponse> cached = idempotencyService.getResult(idempotencyKey);
        if (cached.isPresent()) {
            return ResponseEntity.ok(cached.get());
        }
        return ResponseEntity.accepted()
            .body(new PaymentResponse("PROCESSING", "Payment is being processed"));
    }
    
    try {
        // Phase 2: Process payment and cache the result
        PaymentResponse result = paymentService.processPayment(orderId, user.getId(), request);
        idempotencyService.cacheResult(idempotencyKey, result, Duration.ofHours(24));
        return ResponseEntity.status(HttpStatus.CREATED).body(result);
    } catch (Exception e) {
        // On failure, release the idempotency key so the client can retry
        redisTemplate.delete(lockKey);
        throw e;
    }
}
```

### Scenario 2: N+1 Query Problem in Order Listing with Batch Fetching
**Context:** Your `GET /api/v1/orders` endpoint returns a list of orders with associated order items, customer details, and payment status. The JPA implementation lazily loads relationships, causing cascading N+1 queries. Fetching 100 orders triggers: 1 query for orders + 100 queries for items (one per order) + 100 queries for customers + 100 queries for payments = 301 database queries. Response time is 6 seconds. The database CPU spikes to 95% under moderate traffic. Adding more application servers doesn't help because the bottleneck is in the database query volume, not application processing capacity.

**Resolution:** Use batch fetching with JOIN queries or `IN` clauses. Replace the cascade of individual queries with a few targeted batch queries and assemble the results in application memory. Use JPA's `@EntityGraph` or `JOIN FETCH` to eagerly load relationships in a single query. For complex graphs, use multiple batch queries and manual result assembly. The number of queries drops from 301 to 4 (or fewer), and response time drops from 6 seconds to 150ms.

```java
// Before: N+1 queries — lazy loading triggers individual queries per entity
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // findAll() fetches only orders
    // Accessing order.getItems() triggers separate queries for each order
}

// Service layer with N+1 problem
public List<OrderResponse> findAll() {
    List<Order> orders = orderRepository.findAll();  // 1 query
    return orders.stream().map(order -> {
        List<Item> items = itemRepository.findByOrderId(order.getId()); // N queries
        Customer customer = customerRepository.findById(order.getCustomerId()); // N queries
        return OrderMapper.toResponse(order, items, customer);
    }).toList();
}

// After: Batch fetching — 2 queries total
public List<OrderResponse> findAll() {
    // Step 1: Fetch all orders with eager loading via EntityGraph or JOIN FETCH
    List<Order> orders = orderRepository.findAllWithItemsAndCustomer(); // 1 query
    
    // Step 2: Assemble results — no additional queries needed
    return orders.stream()
        .map(order -> OrderMapper.toResponse(
            order, 
            order.getItems(),    // Already loaded
            order.getCustomer()  // Already loaded
        ))
        .toList();
}

// Repository with EntityGraph
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @EntityGraph(attributePaths = {"items", "customer", "payment"})
    @Query("SELECT o FROM Order o")
    List<Order> findAllWithRelations();
}

// Alternative: Manual batch fetching for more control
public List<OrderResponse> findAll() {
    List<Order> orders = orderRepository.findAll();                        // 1 query
    List<Long> orderIds = orders.stream().map(Order::getId).toList();
    Map<Long, List<Item>> itemsByOrder = itemRepository
        .findByOrderIdIn(orderIds)                                         // 1 batch query
        .stream().collect(Collectors.groupingBy(Item::getOrderId));
    Map<Long, Customer> customersById = customerRepository
        .findByIdIn(orders.stream().map(Order::getCustomerId).toList())   // 1 batch query
        .stream().collect(Collectors.toMap(Customer::getId, Function.identity()));
    return orders.stream()
        .map(order -> OrderMapper.toResponse(
            order,
            itemsByOrder.getOrDefault(order.getId(), List.of()),
            customersById.get(order.getCustomerId())))
        .toList();
}
```

### Scenario 3: Cursor-Based Pagination for Real-Time Social Feed
**Context:** A social media application implements a feed endpoint `GET /api/v1/feed?page=0&size=20` using offset-based pagination. As users create new posts continuously, the page boundaries shift between requests. A user loads page 1 (20 posts), then page 2 — but between the two requests, 5 new posts were created. The user sees 5 posts from page 1 duplicated on page 2 (the new posts pushed old ones to page 2). Users are frustrated by duplicate content, and the infinite scroll UI breaks because it shows the same posts repeatedly. Additionally, `OFFSET 100000` on large datasets forces the database to scan and discard rows, causing slow queries.

**Resolution:** Transition to cursor-based pagination. The client sends the last seen post's ID or timestamp as a cursor parameter. The server returns posts created before that cursor, ensuring stable page boundaries that don't shift when new content is inserted. The response includes `hasNextPage` (boolean) and `nextCursor` (opaque string) for the client to use in the next request. The database query uses `WHERE created_at < :cursor ORDER BY created_at DESC LIMIT :limit`, which is efficient with an index on `created_at` and avoids the offset scanning problem.

```java
@GetMapping("/feed")
public ResponseEntity<FeedResponse> getFeed(
        @AuthenticationPrincipal User user,
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") int limit) {
    // Fetch limit + 1 items to determine if there's a next page
    List<Post> posts = feedService.getPostsAfter(user.getId(), cursor, limit + 1);
    
    // If we fetched more than limit, there are more pages
    boolean hasNextPage = posts.size() > limit;
    if (hasNextPage) {
        posts = posts.subList(0, limit);
    }
    
    // The cursor is the last post's sort key (created_at timestamp)
    String nextCursor = hasNextPage 
        ? posts.get(posts.size() - 1).getCreatedAt().toString() 
        : null;
    
    // Return posts with pagination metadata
    FeedResponse response = new FeedResponse(
        posts.stream()
            .map(PostMapper::toResponse)
            .toList(),
        nextCursor,
        hasNextPage
    );
    
    return ResponseEntity.ok()
        .header("X-Has-Next-Page", String.valueOf(hasNextPage))
        .body(response);
}
```

---

## Scenario-Based Questions

1. **Q: Your social media API fetches a user's feed by aggregating posts from 500 followed users. The database query times out at 30 seconds. How do you redesign this endpoint for <200ms response?**
   - A: Pre-compute the feed asynchronously using a fan-out-on-write approach. When a user posts, insert the post ID into the feed cache of all followers. Store feeds in Redis sorted sets (score = timestamp) for O(log N) reads. For high-profile users with millions of followers, switch to a pull model: their posts are stored in a separate timeline sorted set, and the feed reader merges this with the pre-computed feed from regular users. Use cursor-based pagination with timestamps as cursors. This pushes the computational cost from read time (when the user waits) to write time (when the post is created), making reads consistently fast. The trade-off is that feed pre-computation consumes storage — acceptable given the dramatic read performance improvement.

> **Interview follow-up:** What happens to the fan-out-on-write approach when a user with 10 million followers posts — do you really write to 10 million sorted sets synchronously, and how does that affect the poster's experience?

2. **Q: An e-commerce order placement endpoint receives duplicate requests from network retries, causing double charges. You implement idempotency keys, but your service is stateless and runs on 10 instances. How do you ensure idempotency across all instances?**
   - A: Store idempotency keys in a shared Redis instance accessible to all application instances. Use Redis' atomic `SET key value NX EX <ttl>` command (SET if Not eXists, with expiry) to claim the key. The first instance to execute the SET gets exclusive processing rights. Subsequent requests with the same key find the key exists and return the cached result. For atomic claim and result storage, use a Lua script: `if redis.call("SETNX", KEYS[1], ARGV[1]) == 1 then redis.call("EXPIRE", KEYS[1], 86400) return 1 else return 0 end`. For critical payment scenarios, add a database-level unique constraint on `(idempotency_key)` as a backup — the constraint catches any race conditions that slip past Redis.

3. **Q: Your REST API is slow because mobile clients call `GET /orders` then `GET /orders/{id}/items` for each order (N+1 network requests). How do you optimize without forcing clients to change?**
   - A: Implement a server-side embedding mechanism. Support an `?include=items` query parameter on the orders endpoint: when present, the server JOINs orders with items and returns everything in a single response. This follows the JSON:API specification's `?include` convention. For more complex cases, create a composite endpoint `GET /orders-with-items` that returns denormalized data. An API gateway can also aggregate responses from multiple downstream services. The trade-off is that embedding increases response payload size — only embed when the client explicitly requests it. Monitor response sizes and set limits to prevent abuse.

> **Interview follow-up:** If you support `?include=items.product.vendor.address`, how do you prevent a malicious client from triggering a 15-way join that takes down your database?

4. **Q: You need to support JSON and XML from the same REST API, but XML clients report missing fields while JSON clients get all data. What's happening?**
   - A: The XML serialization likely fails on fields without proper XML annotations. Jackson's default serialization works well for JSON but may not handle certain types (like `Instant`, `Optional`, or collections) correctly for XML. Solutions: (1) Use `jackson-dataformat-xml` instead of JAXB — the same Jackson annotations work for both JSON and XML. (2) Annotate fields with `@JsonProperty` and `@JacksonXmlProperty` for consistent naming. (3) Test both serialization formats in integration tests by sending `Accept: application/json` and `Accept: application/xml` and comparing outputs. (4) Consider dropping XML support if usage is low (<1% of traffic) — the maintenance cost of dual serialization often exceeds the benefit.

5. **Q: Your API returns 500 errors intermittently under load. Error logs show "Connection pool exhausted" but CPU is only 30%. What's the real bottleneck and how do you fix it?**
   - A: The database connection pool is exhausted. Low CPU (30%) indicates the application is waiting for database connections, not computing. Causes: (1) Connection pool too small — HikariCP defaults to 10 connections. Calculate pool size as: `poolSize = Tn × (Cm - 1) + 1`, where Tn = max threads and Cm = max connections per thread. With Tomcat's 200 threads and estimated 30% blocking, target ~60 connections. (2) Slow queries holding connections longer than necessary — optimize queries, add indexes, or set `connectionTimeout` lower so requests fail fast rather than queue. (3) Connection leaks — threads not returning connections to pool due to missing `finally` blocks or unclosed statements. Enable HikariCP's leak detection: `leakDetectionThreshold=60000` logs a stack trace when connections are held too long.

6. **Q: A mobile client needs offline support. Users create orders offline, and sync when connectivity returns. How do you design conflict resolution?**
   - A: Use ETags for optimistic concurrency. The client sends `If-Match` with the last known ETag of resources it's updating. The server rejects with 409 Conflict if the resource changed since the client's last sync. Implement last-write-wins with server timestamps as the simplest resolution. For commutative operations (incrementing counters, adding items to a set), design endpoints as CRDTs (Conflict-Free Replicated Data Types) where the order of syncing doesn't matter. Provide a `/sync` endpoint that returns all changes since a timestamp using `Last-Modified` headers. For critical conflicts like price changes or inventory depletion, return 409 with both versions — the client must present the user with choices. The sync endpoint should support pagination and incremental sync to handle large datasets efficiently.

7. **Q: How do you handle partial updates without forcing clients to send the full resource, while still being RESTful?**
   - A: (1) Use PATCH with JSON Merge Patch (RFC 7396): `PATCH /users/1` with body `{"name": "new name"}` — only specified fields change, omitted fields remain as-is. (2) Use JSON Patch (RFC 6902) for complex operations: `[{"op": "replace", "path": "/name", "value": "new name"}]`. (3) For simpler implementations, use POST with a `/users/{id}/fields` sub-resource pattern. Spring Boot supports `@PatchMapping` with custom merge logic. Implement a `mergePatch()` method that reads the current entity from the database, applies only the fields present in the patch, and saves. Validate that the patch doesn't contain read-only fields (ID, createdAt). Return the full updated resource in the response so the client can update its local state.

8. **Q: Your payment API charges users via a third-party gateway. The gateway sometimes returns 200 OK but the charge actually failed (eventually consistent). Users see "Payment Successful" but no money was charged. How do you handle this?**
   - A: Don't trust the gateway's synchronous response for critical operations. Implement a webhook callback pattern: the gateway processes asynchronously and sends a webhook with the final status. Return `202 Accepted` immediately from the charge endpoint and update the order status when the webhook arrives. Add a "pending" order state — show "Payment processing" to the user until the webhook confirms success or failure. Implement a reconciliation job that polls the gateway for pending transactions every few minutes. Use idempotency keys to prevent duplicate webhook processing — the gateway sends an idempotency key with each webhook, and your server deduplicates by that key. Store both the initial response and the webhook result, and alert if they conflict.

> **Interview follow-up:** The webhook might arrive before the 202 response is returned to the client — how do you handle the race condition where the client polls and sees "paid" before their original request has finished?

9. **Q: You are migrating from REST to GraphQL. REST endpoints have been public for 3 years with 500+ clients. How do you handle this transition safely?**
   - A: Run both systems in parallel using the strangler fig pattern. Keep all existing REST endpoints unchanged. Add a `/graphql` endpoint alongside them. Build GraphQL resolvers that delegate to the same underlying service layer — no need to rewrite business logic. Route new features through GraphQL, keep old features on REST. Send migration guides to clients with clear timelines and code examples. Announce REST deprecation with 18+ months warning. Add `Sunset` and `Deprecated` headers to REST responses. Monitor REST usage per client — only sunset endpoints when traffic drops to zero. Consider using an OpenAPI-to-GraphQL wrapper that auto-generates GraphQL schemas from existing REST endpoints for clients that can't migrate immediately.

10. **Q: How do you design rate limiting for a multi-tenant REST API where one tenant's burst traffic shouldn't affect others?**
    - A: Implement multi-layered rate limiting using the token bucket algorithm in Redis. Layer 1: Tenant-level limits — each tenant gets `X` requests per minute stored in `tenant:<id>:rate`. Layer 2: Endpoint-level limits — expensive endpoints (reports = 10/min) have lower limits than cheap ones (list = 1000/min). Layer 3: Global limits to protect infrastructure from all tenants combined. Use a sliding window algorithm to prevent traffic spikes at window boundaries (fixed windows allow burst at the start of each window). Return `429 Too Many Requests` with `Retry-After` header and `X-RateLimit-*` headers for transparency. Implement priority queues — paid tenants get higher limits and priority during contention. For the sliding window, use a sorted set with timestamps as scores: `ZREMRANGEBYSCORE key (now - window) now`, `ZCARD key`, `ZADD key now member`.

---

## Interview Questions

1. **What are the six constraints of REST?**
   - A: Uniform Interface (resources identified by URI, self-descriptive messages, HATEOAS), Stateless (no server-side session context), Cacheable (responses define cacheability), Client-Server (separation of concerns, independent evolution), Layered System (intermediaries transparent to clients), Code on Demand (optional — executable code transfer). These constraints ensure scalability, modifiability, and visibility in distributed systems.

2. **What is the difference between PUT and PATCH?**
   - A: PUT replaces the entire resource — it is idempotent because sending the same full representation N times produces the same resource state. PATCH applies a partial modification — it may or may not be idempotent depending on the patch format (JSON Merge Patch is not idempotent; JSON Patch can be). PUT requires the client to send the complete resource; PATCH sends only the changes. Choose PUT for full replacements where the client knows the entire resource state; choose PATCH for partial updates with large resources where sending the full representation is expensive.

3. **What is HATEOAS?**
   - A: Hypermedia as the Engine of Application State, representing Level 3 of the Richardson Maturity Model. Responses include links to related actions (e.g., after creating an order, the response includes links to pay, cancel, or view status). This enables clients to discover the API dynamically without out-of-band documentation. The server tells the client what actions are available based on the current state. Rarely implemented in practice due to added complexity, but powerful for truly evolvable APIs.

4. **How do you implement pagination in REST?**
   - A: Offset pagination (`?page=0&size=20`) is simple and supports random page access but breaks when new records are inserted (pages shift) and performs poorly on large offsets (OFFSET 100000 scans and discards rows). Cursor-based pagination (`?cursor=2024-01-01T00:00:00Z&limit=20`) is stable against concurrent inserts and efficient (O(log N) via indexed sort key) but doesn't support random page access. Keyset pagination (`?after=id:1000`) is efficient for databases that can't use offset. Return `hasNextPage` and `totalElements` (only when necessary — COUNT queries are expensive). Always paginate list endpoints from day one.

5. **What are idempotency keys and why are they important?**
   - A: An `Idempotency-Key` header (UUID) ensures exactly-once processing despite retries. The server checks if the key was already processed; if so, returns the cached result instead of processing again. Critical for payment, order placement, and any mutation with side effects. Store keys in Redis with TTL matching the business window (typically 24 hours). Use atomic SET NX EX to claim a key and prevent race conditions across multiple server instances.

6. **How do you handle errors in REST APIs?**
   - A: Use appropriate HTTP status codes for each error type: 400 for validation errors, 401 for authentication failures, 403 for authorization failures, 404 for not found, 409 for conflicts, 422 for unprocessable entities, 429 for rate limiting, 500 for server errors. Return a consistent error response format with a machine-readable code, human-readable message, and optional field-level details. Never expose stack traces or internal implementation details. Log the full error server-side with a correlation ID.

7. **What is content negotiation?**
   - A: The mechanism where the client and server agree on the response format. The client specifies desired formats via the `Accept` header (`Accept: application/json`). The server selects the appropriate `HttpMessageConverter` based on its capabilities. Supports multiple formats (JSON, XML, YAML, etc.) from a single endpoint. Also supports API versioning via custom media types (`application/vnd.myapp.v1+json`). Spring Boot's `ContentNegotiationManager` handles this automatically.

8. **How do you secure a REST API?**
   - A: TLS for all communications (HTTPS only). Authentication via JWT or OAuth2 with token validation on every request. Authorization via role/permission checks at the endpoint and object level. Input validation and sanitization on all inputs. Rate limiting per user and per endpoint. Security headers: HSTS, CSP, X-Content-Type-Options, X-Frame-Options. Never expose internal IDs or stack traces. API keys for third-party clients with proper scoping.

9. **What is the N+1 query problem in REST APIs?**
   - A: When serializing a list of N resources, each resource triggers an additional database query for related data. For example, fetching 100 orders triggers 1 query for the orders list plus 100 individual queries for each order's items (1 + 100 = 101 queries). Fix with batch fetching: `JOIN FETCH`, `@EntityGraph`, or batch loading with `IN` clauses. Also known as the "SELECT N+1" problem in ORM contexts. Monitor via Spring Boot's datasource proxy or Hibernate query statistics.

10. **How do you version a REST API?**
    - A: URI path versioning (`/api/v1/users`) is the most common approach — simple, discoverable, and CDN-friendly. Header versioning (`Accept: application/vnd.myapp.v1+json`) keeps URLs clean and follows REST principles more closely. Support a maximum of 3 active versions: deprecated, current, and preview. Deprecate with 18+ months notice using `Sunset` and `Deprecated` headers. Never break existing clients without a migration path.

---

## Developer Recommendations

- **Always implement idempotency for mutation endpoints** — `POST /payments` and `POST /orders` will inevitably receive duplicate requests from network retries, mobile app retries, and client timeouts. An `Idempotency-Key` header prevents double charges and duplicate orders at minimal implementation cost. Store keys in Redis with TTL matching the business window (typically 24 hours). Use atomic `SET NX EX` to claim the key — the first instance to execute the SET gets to process; others return the cached result. This is more reliable than relying on clients to deduplicate. For database-backed idempotency, use a unique constraint on `(idempotency_key)` as a backup. The cost: one Redis call per request. The benefit: elimination of an entire class of data integrity bugs.

- **Use cursor-based pagination over offset-based for production APIs** — Offset pagination has two critical flaws: it breaks when new records are inserted between page requests (pages shift, creating duplicates and missed records), and it performs poorly on large datasets (`OFFSET 100000` forces the database to scan and discard 100,000 rows). Cursor-based pagination using a unique sortable field (`created_at`, `id`) is O(log N) and stable regardless of concurrent inserts. The trade-off is no random page access (no "go to page 5") — users rarely jump to arbitrary pages in real-world applications. Implement cursors as opaque strings (base64-encoded timestamps or IDs) so clients don't try to manipulate them.

- **Batch database queries to avoid N+1 problems** — The N+1 problem is the single most common performance issue in REST APIs, silently multiplying database load by orders of magnitude. When returning a list of resources with related data, always batch-fetch the related data using `IN` clauses or JOIN queries. Use Spring's `@EntityGraph` or Hibernate's `JOIN FETCH` for JPA. For GraphQL, use DataLoader. Enable Hibernate SQL logging in development (`spring.jpa.show-sql=true`) and count queries per request. Set up query count monitoring in production — a sudden increase in queries-per-request indicates an N+1 regression. This single optimization resolves 80% of "slow API" complaints.

- **Design for API evolution from day one** — Start versioning before you think you need it. The cost of adding versioning later (breaking existing clients, coordinated deploys, migration scripts, retroactive contract decisions) dwarfs the minimal upfront cost of prefixing URIs with `/api/v1/`. Use additive-only changes for as long as possible: add new fields to responses (clients ignore them), add new optional parameters, add new endpoints. Deprecate before removing, never remove without a documented migration path. Keep a maximum of 3 active versions. Add `Sunset` and `Deprecated` headers to responses so clients can programmatically know when a version is being retired.

- **Use appropriate HTTP status codes consistently** — Every status code tells the client what to do next: 201 for creation (client knows the resource was created and can find it via the Location header), 204 for deletion (no content to return — avoid the common mistake of returning 200), 400 for bad request (client should fix the request before retrying), 401 for unauthenticated (client should provide credentials), 403 for unauthorized (client is known but lacks permission), 409 for conflict (client should retry with updated data — use with ETags), 422 for validation errors (client should fix specific fields as described in the response). Inconsistent status codes force clients to parse error messages heuristically — always use the correct code. A simple cheat sheet: success codes vary by operation type; client errors (4xx) should indicate what the client needs to change; server errors (5xx) should be rare and indicate the server failed.

- **Implement rate limiting before you need it** — A single abusive client, bug in a mobile app, or misconfigured webhook can take down your entire API. Implement rate limiting from the first deployment, not after the first outage. Use the token bucket algorithm with Redis for distributed rate limiting across multiple instances. Set per-tenant, per-endpoint, and global limits. Return `429 Too Many Requests` with `Retry-After` header (seconds until the client can retry) and `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers for transparency. Monitor rate limit hit rates — a rising trend indicates a client with a bug, an attacker probing your API, or a legitimate client that needs higher limits. Give paying customers higher limits but always enforce some cap.

---

*Last updated: 2026-06-06. Key concepts: uniform interface, stateless communication, HTTP methods and status codes, idempotency, pagination strategies, content negotiation, HATEOAS, error handling, rate limiting, API evolution.*
