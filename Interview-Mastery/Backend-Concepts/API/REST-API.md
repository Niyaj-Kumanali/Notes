# REST API

## 1. Executive Summary

Representational State Transfer (REST) is an architectural style for designing distributed systems, first introduced by Roy Fielding in his 2000 doctoral dissertation. REST APIs use HTTP as the communication protocol and treat server resources as entities that can be created, read, updated, and deleted via standard HTTP methods. REST is stateless, cacheable, and follows a uniform interface, making it the dominant approach for building web APIs in the industry.

## 2. Core Theory

REST is built on six architectural constraints:

- **Uniform Interface**: Resources are identified in requests, resource representations are used to manipulate resources, self-descriptive messages include metadata, and hypermedia drives application state (HATEOAS).
- **Stateless**: Each request from a client contains all the information needed to process it. No client context is stored on the server between requests.
- **Cacheable**: Responses must define themselves as cacheable or non-cacheable to improve performance.
- **Client-Server**: Separation of concerns allows the client and server to evolve independently.
- **Layered System**: A client cannot tell whether it is connected directly to the end server or an intermediary (load balancer, cache, etc.).
- **Code on Demand (optional)**: Servers can extend client functionality by transferring executable code.

### HTTP Methods and Their Semantics

| Method | CRUD Equivalent | Idempotent | Safe | Use Case |
|--------|----------------|------------|------|----------|
| GET | Read | Yes | Yes | Retrieve a resource |
| POST | Create | No | No | Create a new resource |
| PUT | Update/Replace | Yes | No | Full update of a resource |
| PATCH | Partial Update | No | No | Partial modification |
| DELETE | Delete | Yes | No | Remove a resource |
| HEAD | - | Yes | Yes | Retrieve headers only |
| OPTIONS | - | Yes | Yes | Discover allowed methods |

### HTTP Status Codes

```
1xx - Informational
2xx - Success (200 OK, 201 Created, 204 No Content)
3xx - Redirection (301 Moved Permanently, 304 Not Modified)
4xx - Client Error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity)
5xx - Server Error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable)
```

## 3. Under-the-Hood Deep Dive

### Request Processing Pipeline

When a REST API request arrives at a Spring Boot application:

1. The embedded Tomcat/Netty server accepts the TCP connection.
2. The request passes through a chain of servlet filters (security, logging, CORS).
3. `DispatcherServlet` receives the request and consults `HandlerMapping` beans to find the matching `@RequestMapping` method.
4. `HandlerAdapter` invokes the method after argument resolution (path variables, query params, request body).
5. The method returns a response entity or domain object.
6. `HttpMessageConverter` serializes the response (e.g., Jackson for JSON).
7. The response travels back through the filter chain.

### Content Negotiation

REST APIs support content negotiation via:
- `Accept` header: Client specifies desired response format (`application/json`, `application/xml`).
- `Content-Type` header: Client specifies the format of the request body.
- Spring Boot automatically handles negotiation via `ContentNegotiationManager`.

### HATEOAS (Hypermedia as the Engine of Application State)

HATEOAS adds links to API responses so clients can discover available actions dynamically.

```java
import org.springframework.hateoas.EntityModel;
import org.springframework.hateoas.Link;
import org.springframework.hateoas.server.mvc.WebMvcLinkBuilder;

@EntityModel<User> getUser(@PathVariable Long id) {
    User user = userService.findById(id);
    EntityModel<User> model = EntityModel.of(user);
    model.add(WebMvcLinkBuilder.linkTo(
        WebMvcLinkBuilder.methodOn(UserController.class).getUser(id)
    ).withSelfRel());
    model.add(WebMvcLinkBuilder.linkTo(
        WebMvcLinkBuilder.methodOn(UserController.class).getAllUsers()
    ).withRel("users"));
    return model;
}
```

## 4. Production Code Examples

### Spring Boot REST Controller

```java
@RestController
@RequestMapping("/api/v1/users")
@Slf4j
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    @ResponseStatus(HttpStatus.OK)
    public Page<UserResponse> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id,asc") String[] sort) {

        Sort sorting = Sort.by(
            sort[1].equalsIgnoreCase("desc")
                ? Sort.Direction.DESC
                : Sort.Direction.ASC,
            sort[0]
        );
        Pageable pageable = PageRequest.of(page, size, sorting);
        return userService.findAll(pageable)
            .map(UserMapper::toResponse);
    }

    @GetMapping("/{id}")
    @ResponseStatus(HttpStatus.OK)
    public UserResponse getUser(@PathVariable Long id) {
        return UserMapper.toResponse(
            userService.findById(id)
        );
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        User user = userService.create(UserMapper.toEntity(request));
        return UserMapper.toResponse(user);
    }

    @PutMapping("/{id}")
    @ResponseStatus(HttpStatus.OK)
    public UserResponse updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        return UserMapper.toResponse(
            userService.update(id, UserMapper.toEntity(request))
        );
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### Service Layer with Exception Handling

```java
@Service
@Transactional
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException(
                "User not found with id: " + id
            ));
    }

    public User create(User user) {
        if (userRepository.existsByEmail(user.getEmail())) {
            throw new ConflictException("Email already in use: " + user.getEmail());
        }
        return userRepository.save(user);
    }

    public User update(Long id, User updatedUser) {
        User existing = findById(id);
        existing.setName(updatedUser.getName());
        existing.setEmail(updatedUser.getEmail());
        return userRepository.save(existing);
    }

    public void delete(Long id) {
        User user = findById(id);
        userRepository.delete(user);
    }
}
```

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(ConflictException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleConflict(ConflictException ex) {
        return new ErrorResponse("CONFLICT", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
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

### Request/Response DTOs with Validation

```java
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be 2-100 characters")
    String name,

    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    String email
) {}

public record UserResponse(
    Long id,
    String name,
    String email,
    Instant createdAt,
    Instant updatedAt
) {}

public record ErrorResponse(
    String code,
    String message,
    Map<String, String> details
) {
    public ErrorResponse(String code, String message) {
        this(code, message, null);
    }
}
```

### REST Client with RestClient (Spring 6.1+)

```java
@Service
public class UserApiClient {

    private final RestClient restClient;

    public UserApiClient(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .requestInterceptor((request, body, execution) -> {
                log.info("Request: {} {}", request.getMethod(), request.getURI());
                return execution.execute(request, body);
            })
            .build();
    }

    public UserResponse getUser(Long id) {
        return restClient.get()
            .uri("/api/v1/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
                throw new ApiClientException("Client error: " + response.getStatusText());
            })
            .body(UserResponse.class);
    }

    public UserResponse createUser(CreateUserRequest request) {
        return restClient.post()
            .uri("/api/v1/users")
            .body(request)
            .retrieve()
            .body(UserResponse.class);
    }
}
```

### Paginated Response Pattern

```java
public record PagedResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean first,
    boolean last
) {
    public static <T> PagedResponse<T> from(Page<T> page) {
        return new PagedResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.isFirst(),
            page.isLast()
        );
    }
}
```

### Filtering, Sorting, and Searching

```java
@GetMapping
public PagedResponse<UserResponse> searchUsers(
        @RequestParam(required = false) String name,
        @RequestParam(required = false) String email,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "id,asc") String[] sort) {

    Specification<User> spec = Specification.where(null);

    if (name != null) {
        spec = spec.and((root, query, cb) ->
            cb.like(cb.lower(root.get("name")), "%" + name.toLowerCase() + "%"));
    }
    if (email != null) {
        spec = spec.and((root, query, cb) ->
            cb.equal(root.get("email"), email));
    }

    Sort sorting = Sort.by(
        sort[1].equalsIgnoreCase("desc")
            ? Sort.Direction.DESC : Sort.Direction.ASC,
        sort[0]
    );
    Pageable pageable = PageRequest.of(page, size, sorting);

    return PagedResponse.from(
        userRepository.findAll(spec, pageable).map(UserMapper::toResponse)
    );
}
```

## 5. Real-World Scenarios

### Scenario 1: Social Media Platform API

Designing a Twitter-like API requires careful resource modeling:

```
POST /api/v1/tweets              - Create a tweet
GET  /api/v1/tweets/{id}         - Get a tweet
GET  /api/v1/tweets/{id}/replies - Get replies to a tweet
POST /api/v1/tweets/{id}/like    - Like a tweet
POST /api/v1/tweets/{id}/retweet - Retweet
GET  /api/v1/feed                - Get user's timeline feed
```

### Scenario 2: E-Commerce Order System

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<OrderResponse> placeOrder(
            @AuthenticationPrincipal User user,
            @Valid @RequestBody PlaceOrderRequest request) {
        Order order = orderService.placeOrder(user.getId(), request);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(order.getId())
            .toUri();
        return ResponseEntity.created(location).body(OrderMapper.toResponse(order));
    }

    @PostMapping("/{orderId}/cancel")
    public ResponseEntity<Void> cancelOrder(
            @PathVariable Long orderId,
            @AuthenticationPrincipal User user) {
        orderService.cancel(orderId, user.getId());
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/{orderId}/status")
    public ResponseEntity<OrderStatusResponse> getOrderStatus(
            @PathVariable Long orderId) {
        return ResponseEntity.ok(
            new OrderStatusResponse(orderService.getStatus(orderId))
        );
    }
}
```

### Scenario 3: Bulk Operations

```java
@PostMapping("/bulk")
public ResponseEntity<List<UserResponse>> bulkCreate(
        @Valid @RequestBody List<@Valid CreateUserRequest> requests) {
    List<User> users = requests.stream()
        .map(UserMapper::toEntity)
        .collect(Collectors.toList());
    return ResponseEntity.ok(
        userService.bulkCreate(users).stream()
            .map(UserMapper::toResponse)
            .collect(Collectors.toList())
    );
}

@DeleteMapping("/bulk")
public ResponseEntity<Void> bulkDelete(@RequestBody List<Long> ids) {
    userService.bulkDelete(ids);
    return ResponseEntity.noContent().build();
}
```

## 6. Performance

### Connection Pooling

```java
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

### Caching with Spring Cache

```java
@EnableCaching
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager(
            "users", "roles", "permissions"
        );
        cacheManager.setAllowNullValues(false);
        return cacheManager;
    }
}

@Service
public class CachedUserService {

    @Cacheable(value = "users", key = "#id", unless = "#result == null")
    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
    }

    @CachePut(value = "users", key = "#user.id")
    public User update(User user) {
        return userRepository.save(user);
    }

    @CacheEvict(value = "users", key = "#id")
    public void delete(Long id) {
        userRepository.deleteById(id);
    }
}
```

### Response Compression

```yaml
server:
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html
    min-response-size: 2048
```

### Database Optimization

- Use pagination with `LIMIT`/`OFFSET` or keyset pagination.
- Add proper database indexes for frequently queried columns.
- Use `@EntityGraph` for eager fetching of relationships.
- Avoid N+1 queries by using `JOIN FETCH` or `@BatchSize`.
- Use read replicas for read-heavy workloads.

### Asynchronous Request Processing

```java
@RestController
public class AsyncController {

    @GetMapping("/async/users")
    public CompletableFuture<List<UserResponse>> getUsers() {
        return CompletableFuture.supplyAsync(() ->
            userService.findAll().stream()
                .map(UserMapper::toResponse)
                .collect(Collectors.toList())
        );
    }
}
```

## 7. Security

### CORS Configuration

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            .allowedHeaders("*")
            .exposedHeaders("X-Total-Count")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

### Rate Limiting

```java
@Component
public class RateLimitingInterceptor implements HandlerInterceptor {

    private final RateLimiter rateLimiter;

    public RateLimitingInterceptor(RateLimiter rateLimiter) {
        this.rateLimiter = rateLimiter;
    }

    @Override
    public boolean preHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler) throws Exception {
        String clientIp = request.getRemoteAddr();
        if (!rateLimiter.tryAcquire(clientIp)) {
            response.setStatus(429);
            response.getWriter().write("Too many requests");
            return false;
        }
        return true;
    }
}
```

### Input Validation and Sanitization

```java
@PostMapping
public ResponseEntity<UserResponse> createUser(
        @Valid @RequestBody CreateUserRequest request) {
    // Validate that email doesn't contain malicious patterns
    if (request.email().matches(".*[<>].*")) {
        throw new BadRequestException("Email contains invalid characters");
    }
    // Sanitize name
    String sanitizedName = HtmlUtils.htmlEscape(request.name());
    CreateUserRequest sanitized = new CreateUserRequest(
        sanitizedName, request.email()
    );
    return ResponseEntity.status(201).body(
        UserMapper.toResponse(userService.create(UserMapper.toEntity(sanitized)))
    );
}
```

## 8. Common Mistakes

- **Using GET for mutations**: Always use the correct HTTP method.
- **Inconsistent error responses**: Standardize error response format across all endpoints.
- **Ignoring idempotency**: PUT and DELETE should be idempotent; POST should not.
- **Exposing internal IDs**: Use UUIDs or opaque identifiers instead of auto-increment IDs.
- **No pagination for list endpoints**: Always paginate list responses.
- **Returning stack traces in production**: Never expose internal error details.
- **Not versioning APIs**: Always version your API from day one.
- **Incorrect HTTP status codes**: Use appropriate status codes (201 for create, 204 for delete).
- **N+1 query problem**: Use JOIN FETCH or EntityGraph for relationships.
- **No input validation**: Always validate and sanitize all inputs.

## 9. Senior Engineer Perspective

### API Design Maturity Levels (Richardson Maturity Model)

```
Level 0: The Swamp of POX - Using HTTP as a tunnel (single URI, one method)
Level 1: Resources          - Multiple URIs but single HTTP method
Level 2: HTTP Verbs         - Proper use of HTTP methods and status codes
Level 3: Hypermedia Controls - HATEOAS enabling discoverability
```

### Design Considerations

- **Plural vs Singular Nouns**: Use plural (`/users` not `/user`).
- **Nested Resources**: Limit nesting to 2-3 levels (`/users/{id}/orders/{orderId}`).
- **Query Complexity**: Implement query complexity limits to prevent expensive queries.
- **API Contract First**: Use OpenAPI/Swagger to define the contract before implementation.
- **Graceful Degradation**: Design for partial failures; use circuit breakers.
- **Observability**: Log request IDs, trace IDs, and response times.
- **Backward Compatibility**: Never break existing clients; use additive changes.

### OpenAPI Contract Example

```yaml
openapi: 3.1.0
info:
  title: User Management API
  version: 1.0.0
paths:
  /api/v1/users:
    get:
      summary: List all users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
        - name: size
          in: query
          schema:
            type: integer
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
    post:
      summary: Create a new user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
```

## 10. Interview Questions (Easy)

1. What does REST stand for and who introduced it?
2. What are the six constraints of REST architecture?
3. What is the difference between PUT and PATCH?
4. What HTTP status code should be returned for a successful resource creation?
5. What does idempotent mean in the context of HTTP methods?
6. Which HTTP methods are safe (no side effects)?
7. What is the purpose of the OPTIONS HTTP method?
8. What is content negotiation in REST APIs?
9. What is the difference between 401 Unauthorized and 403 Forbidden?
10. What is the purpose of the Accept header in an HTTP request?

## Medium

1. What is HATEOAS and why is it important?
2. How do you handle pagination in a REST API?
3. What is the Richardson Maturity Model?
4. How do you implement versioning in a REST API?
5. What is the N+1 query problem and how do you solve it?
6. How do you handle partial updates in REST?
7. What are the best practices for designing REST API error responses?
8. How does content negotiation work in Spring Boot?
9. What is the difference between `@RestController` and `@Controller`?
10. How do you implement sorting and filtering in REST APIs?

## 11. Advanced Interview Questions (Hard)

1. How would you design a REST API that supports both JSON and Protocol Buffers?
2. Implement a request deduplication mechanism for POST requests.
3. How do you handle distributed transactions across multiple REST API calls?
4. Design an API that supports bulk operations with atomicity guarantees.
5. How would you implement cursor-based pagination vs offset-based pagination?
6. What strategies exist for handling concurrent updates to the same resource?
7. How do you design an API that supports partial responses (fields filtering)?
8. Implement an idempotency key system for a payment API.
9. How do you handle API evolution without breaking existing clients?
10. Design a rate-limiting strategy for a multi-tenant REST API.

## System Design

1. Design a REST API for a real-time collaborative document editing service.
2. Design a REST API for a ride-sharing platform (Uber/Lyft).
3. Design a REST API for a social media feed system.
4. Design a REST API for an e-commerce order management system.
5. Design a REST API for a video streaming platform.
6. Design a REST API for a hotel booking system with availability search.
7. Design a REST API for a payment processing system.
8. Design a REST API for a notification delivery service.
9. Design a REST API for a content moderation pipeline.
10. Design a REST API for a multi-tenant SaaS platform.

## 12. Expert-Level Interview Questions (Architect-Level)

1. How would you design a globally distributed REST API that maintains consistency across regions while minimizing latency?
2. Design an event-driven REST API where resources emit state change events that multiple downstream systems consume asynchronously.
3. How do you evolve a monolithic REST API into microservices without breaking existing clients? Describe the strangler fig pattern in detail.
4. Design an API versioning strategy that supports both URI versioning and header versioning simultaneously, with graceful deprecation.
5. How would you implement a REST API with support for transactional guarantees across multiple resource types using the saga pattern?
6. Design a hypermedia-driven API that enables clients to navigate complex workflows without prior knowledge of endpoint URLs.
7. How do you design a REST API that handles backpressure from slow clients without degrading the entire system?
8. Design an API gateway that routes requests to different backend services based on resource type, while providing unified authentication, rate limiting, and caching.
9. How would you implement API analytics and usage billing for a REST API exposed as a paid product?
10. Design a versionless API strategy where backward compatibility is maintained indefinitely through extensible schemas and capability negotiation.

## 13. Debugging & Troubleshooting

### Common Issues and Solutions

- **Slow response times**: Check database queries, add indexes, enable caching, use async processing.
- **Connection timeouts**: Check connection pool settings, database availability, network latency.
- **Deserialization errors**: Verify JSON structure matches DTO fields, check for unknown properties.
- **CORS errors**: Verify CORS configuration, check allowed origins and methods.
- **415 Unsupported Media Type**: Check Content-Type header matches the expected media type.

### Debugging with Spring Boot Actuator

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,httptrace,loggers,env
  endpoint:
    health:
      show-details: always
```

### Request Tracing

```java
@Component
public class RequestTracingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String requestId = UUID.randomUUID().toString();
        httpRequest.setAttribute("requestId", requestId);
        MDC.put("requestId", requestId);

        long start = System.currentTimeMillis();
        try {
            chain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - start;
            log.info("{} {} {} {} {}ms",
                requestId,
                httpRequest.getMethod(),
                httpRequest.getRequestURI(),
                ((HttpServletResponse) response).getStatus(),
                duration);
            MDC.remove("requestId");
        }
    }
}
```

## 14. Comparison Section

### REST vs GraphQL

| Aspect | REST | GraphQL |
|--------|------|---------|
| Data Fetching | Fixed structure, may over/under-fetch | Client specifies exact data needed |
| Endpoints | Multiple endpoints (one per resource) | Single endpoint |
| Caching | Native HTTP caching | Requires custom caching |
| Learning Curve | Low | Medium |
| Tooling | Mature ecosystem | Growing ecosystem |
| Real-time | WebSockets for real-time | Subscriptions for real-time |
| File Upload | Multipart upload | Requires custom handling |
| Versioning | URI/header based | Evolve schema, deprecate fields |

### REST vs gRPC

| Aspect | REST | gRPC |
|--------|------|------|
| Protocol | HTTP/1.1 (HTTP/2 optional) | HTTP/2 (mandatory) |
| Data Format | JSON (human-readable) | Protocol Buffers (binary) |
| Performance | Moderate | High |
| Streaming | Via WebSockets | Native bidirectional streaming |
| Code Generation | Manual/OpenAPI generators | Built-in from proto files |
| Browser Support | Native | Requires gRPC-web proxy |
| Contract | OpenAPI/Swagger | Proto files |

## 15. Revision Notes

- REST = Representational State Transfer, 6 constraints (Uniform Interface, Stateless, Cacheable, Client-Server, Layered System, Code on Demand)
- HTTP Methods: GET (read), POST (create), PUT (replace), PATCH (partial), DELETE (remove)
- Idempotent: GET, PUT, DELETE, HEAD, OPTIONS
- Safe: GET, HEAD, OPTIONS
- Use plural nouns, proper HTTP status codes, version from day one
- Always paginate, use consistent error format, avoid N+1 queries
- Spring Boot: `@RestController`, `@RequestMapping`, `@Valid`, `@ExceptionHandler`
- HATEOAS for discoverability, OpenAPI for documentation
- Richardson Maturity Model: Level 0-3

## 16. Cheat Sheet

```
+------------------------------------------------------------------+
| REST API CHEAT SHEET                                             |
+------------------------------------------------------------------+
| HTTP METHODS                                                     |
|   GET    /users        -> 200 OK                                 |
|   GET    /users/{id}   -> 200 OK                                 |
|   POST   /users        -> 201 Created (+ Location header)        |
|   PUT    /users/{id}   -> 200 OK                                 |
|   PATCH  /users/{id}   -> 200 OK                                 |
|   DELETE /users/{id}   -> 204 No Content                         |
+------------------------------------------------------------------+
| STATUS CODES                                                     |
|   200 OK            201 Created      204 No Content              |
|   301 Moved         304 Not Modified  400 Bad Request            |
|   401 Unauthorized  403 Forbidden     404 Not Found              |
|   409 Conflict      422 Unprocessable 429 Too Many Requests      |
|   500 Internal      502 Bad Gateway   503 Service Unavailable    |
+------------------------------------------------------------------+
| SPRING BOOT ANNOTATIONS                                          |
|   @RestController    @RequestMapping   @GetMapping              |
|   @PostMapping       @PutMapping       @DeleteMapping           |
|   @PatchMapping      @Valid            @RequestParam            |
|   @PathVariable      @RequestBody      @ResponseStatus           |
|   @RestControllerAdvice               @ExceptionHandler         |
+------------------------------------------------------------------+
| NAMING CONVENTIONS                                               |
|   /api/v1/resources                                              |
|   /api/v1/resources/{id}                                         |
|   /api/v1/resources/{id}/sub-resources                           |
|   ?page=0&size=20&sort=name,desc                                |
+------------------------------------------------------------------+
| ERROR RESPONSE FORMAT                                            |
| {                                                                |
|   "code": "NOT_FOUND",                                           |
|   "message": "User not found with id: 123",                      |
|   "details": null                                                 |
| }                                                                |
+------------------------------------------------------------------+
| CACHE ANNOTATIONS                                                |
|   @Cacheable    caches method result                              |
|   @CachePut     updates cache                                    |
|   @CacheEvict   removes from cache                                |
|   @Caching      groups multiple cache annotations                |
+------------------------------------------------------------------+
```
