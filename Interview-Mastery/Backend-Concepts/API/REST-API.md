# REST API

---

## Overview

- **Definition:** Representational State Transfer (REST) is an architectural style for designing distributed systems, introduced by Roy Fielding in his 2000 doctoral dissertation. REST APIs use HTTP as the communication protocol and treat server resources as entities that can be created, read, updated, and deleted via standard HTTP methods.

- **Why It Exists:** Before REST, web APIs were unstructured and inconsistent. REST provided a standardized, stateless, cacheable approach with a uniform interface, making it the dominant architecture for building web APIs.

- **Six Constraints:**
  - **Uniform Interface** — resources identified in requests, self-descriptive messages, HATEOAS for discoverability
  - **Stateless** — each request contains all info needed; no server-side client context
  - **Cacheable** — responses define themselves as cacheable or non-cacheable
  - **Client-Server** — separation of concerns allows independent evolution
  - **Layered System** — intermediaries (load balancers, caches) are transparent to clients
  - **Code on Demand (optional)** — servers can extend client functionality via executable code

---

## HTTP Methods & Status Codes

- **HTTP Methods and Their Semantics:**

  - **GET** — Read (idempotent, safe)
  - **POST** — Create (not idempotent)
  - **PUT** — Full update/replace (idempotent)
  - **PATCH** — Partial modification (not idempotent)
  - **DELETE** — Remove (idempotent)
  - **HEAD** — Retrieve headers only (idempotent, safe)
  - **OPTIONS** — Discover allowed methods (idempotent, safe)

- **Status Code Families:**
  - **1xx** — Informational
  - **2xx** — Success (200 OK, 201 Created, 204 No Content)
  - **3xx** — Redirection (301 Moved, 304 Not Modified)
  - **4xx** — Client Error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable)
  - **5xx** — Server Error (500 Internal, 502 Bad Gateway, 503 Unavailable)

---

## Request Processing Pipeline (Spring Boot)

- **Flow:**
  1. Embedded Tomcat/Netty accepts the TCP connection
  2. Request passes through servlet filter chain (security, logging, CORS)
  3. `DispatcherServlet` consults `HandlerMapping` to find the matching `@RequestMapping` method
  4. `HandlerAdapter` invokes the method after argument resolution (path variables, query params, body)
  5. Method returns a response entity or domain object
  6. `HttpMessageConverter` serializes the response (Jackson for JSON)
  7. Response travels back through the filter chain

- **Content Negotiation:**
  - `Accept` header — client specifies desired response format
  - `Content-Type` header — client specifies request body format
  - Spring Boot handles via `ContentNegotiationManager`

---

## Production Code Examples

### Spring Boot REST Controller

```java
@RestController
@RequestMapping("/api/v1/users")
@Slf4j
public class UserController {
    private final UserService userService;
    public UserController(UserService userService) { this.userService = userService; }

    @GetMapping
    @ResponseStatus(HttpStatus.OK)
    public Page<UserResponse> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id,asc") String[] sort) {
        Sort sorting = Sort.by(
            sort[1].equalsIgnoreCase("desc") ? Sort.Direction.DESC : Sort.Direction.ASC,
            sort[0]);
        Pageable pageable = PageRequest.of(page, size, sorting);
        return userService.findAll(pageable).map(UserMapper::toResponse);
    }

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return UserMapper.toResponse(userService.findById(id));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        return UserMapper.toResponse(userService.create(UserMapper.toEntity(request)));
    }

    @PutMapping("/{id}")
    public UserResponse updateUser(@PathVariable Long id, @Valid @RequestBody UpdateUserRequest request) {
        return UserMapper.toResponse(userService.update(id, UserMapper.toEntity(request)));
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) { userService.delete(id); }
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

### Request/Response DTOs

```java
public record CreateUserRequest(
    @NotBlank String name,
    @NotBlank @Email String email) {}

public record UserResponse(Long id, String name, String email, Instant createdAt, Instant updatedAt) {}

public record ErrorResponse(String code, String message, Map<String, String> details) {
    public ErrorResponse(String code, String message) { this(code, message, null); }
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
            }).build();
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
        return restClient.post().uri("/api/v1/users").body(request)
            .retrieve().body(UserResponse.class);
    }
}
```

### Pagination Pattern

```java
public record PagedResponse<T>(List<T> content, int page, int size,
    long totalElements, int totalPages, boolean first, boolean last) {
    public static <T> PagedResponse<T> from(Page<T> page) {
        return new PagedResponse<>(page.getContent(), page.getNumber(), page.getSize(),
            page.getTotalElements(), page.getTotalPages(), page.isFirst(), page.isLast());
    }
}
```

---

## Common Mistakes

- **Using GET for mutations** — always use the correct HTTP method
- **Inconsistent error responses** — standardize error format across all endpoints
- **Ignoring idempotency** — PUT/DELETE should be idempotent; POST should not
- **Exposing internal IDs** — use UUIDs instead of auto-increment IDs
- **No pagination** — always paginate list endpoints
- **Returning stack traces in production** — never expose internal error details
- **Not versioning APIs** — always version from day one
- **Incorrect HTTP status codes** — use 201 for create, 204 for delete
- **N+1 query problem** — use JOIN FETCH or EntityGraph
- **No input validation** — always validate and sanitize all inputs

---

## Key Design Considerations

- **Richardson Maturity Model:**
  - Level 0: Swamp of POX — HTTP as tunnel (one URI, one method)
  - Level 1: Resources — multiple URIs but single HTTP method
  - Level 2: HTTP Verbs — proper use of methods and status codes
  - Level 3: Hypermedia Controls — HATEOAS for discoverability

- **Best Practices:**
  - Use plural nouns for resources (`/users` not `/user`)
  - Limit nested resources to 2-3 levels
  - Implement query complexity limits
  - Use OpenAPI/Swagger for contract-first design
  - Design for graceful degradation with circuit breakers
  - Log request IDs, trace IDs, and response times
  - Never break existing clients; use additive changes

---

## Real-World Scenarios

### Scenario 1: Idempotent Payment Processing
**Context:** Your e-commerce payment endpoint `POST /api/v1/orders/{id}/pay` charges the user's card. Network timeouts cause the mobile client to retry automatically. Without idempotency, each retry triggers a new charge — users are double-charged, support tickets surge, and the finance team demands a fix.

**Resolution:** Implement idempotency keys. The client generates a UUID (`Idempotency-Key` header) for each payment attempt. The server stores the key with the processing result in Redis (TTL: 24 hours). On the first request, process the payment and cache the result. On retry with the same key, return the cached result without processing again. This guarantees exactly-once processing.

```java
@PostMapping("/{orderId}/pay")
public ResponseEntity<PaymentResponse> payOrder(
        @PathVariable Long orderId,
        @RequestHeader("Idempotency-Key") UUID idempotencyKey,
        @Valid @RequestBody PaymentRequest request,
        @AuthenticationPrincipal User user) {
    // Check if this idempotency key was already processed
    Optional<PaymentResponse> cached = idempotencyService.getResult(idempotencyKey);
    if (cached.isPresent()) {
        return ResponseEntity.ok(cached.get());
    }
    // Process payment and cache the result
    PaymentResponse result = paymentService.processPayment(orderId, user.getId(), request);
    idempotencyService.cacheResult(idempotencyKey, result, Duration.ofHours(24));
    return ResponseEntity.status(HttpStatus.CREATED).body(result);
}
```

### Scenario 2: N+1 Query Problem in Order Listing
**Context:** `GET /api/v1/orders` returns a list of orders. The response includes order items, but the implementation iterates through orders and queries items one by one: `SELECT * FROM items WHERE order_id = ?`. With 100 orders, this generates 101 database queries. Response time is 5 seconds.

**Resolution:** Use batch fetching with JOIN or `IN` clause. Replace N+1 queries with a single batch query: `SELECT * FROM items WHERE order_id IN (:orderIds)`. Map results in memory.

```java
// Before: N+1 queries
public List<OrderResponse> findAll() {
    List<Order> orders = orderRepository.findAll();  // 1 query
    return orders.stream().map(order -> {
        List<Item> items = itemRepository.findByOrderId(order.getId()); // N queries
        return OrderMapper.toResponse(order, items);
    }).toList();
}

// After: Batch fetching (2 queries total)
public List<OrderResponse> findAll() {
    List<Order> orders = orderRepository.findAll();           // 1 query
    List<Long> orderIds = orders.stream().map(Order::getId).toList();
    Map<Long, List<Item>> itemsByOrder = itemRepository
        .findByOrderIdIn(orderIds)                           // 1 batch query
        .stream().collect(Collectors.groupingBy(Item::getOrderId));
    return orders.stream()
        .map(order -> OrderMapper.toResponse(order, itemsByOrder.getOrDefault(order.getId(), List.of())))
        .toList();
}
```

### Scenario 3: Cursor-Based Pagination for Real-Time Feed
**Context:** A social media feed `GET /api/v1/feed?page=0&size=20` uses offset pagination. As new posts are created, the page boundaries shift. A user on page 2 sees the same posts they already saw on page 1 (because new posts pushed old ones to page 2). Users are frustrated by duplicate content.

**Resolution:** Switch to cursor-based pagination. The client sends the last seen post's ID or timestamp as a cursor. The server returns posts after that cursor, with `hasNextPage` and `nextCursor` in the response.

```java
@GetMapping("/feed")
public ResponseEntity<FeedResponse> getFeed(
        @AuthenticationPrincipal User user,
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") int limit) {
    List<Post> posts = feedService.getPostsAfter(user.getId(), cursor, limit + 1);
    boolean hasNextPage = posts.size() > limit;
    if (hasNextPage) posts = posts.subList(0, limit);
    String nextCursor = hasNextPage ? posts.get(posts.size() - 1).getCreatedAt().toString() : null;
    return ResponseEntity.ok(new FeedResponse(posts, nextCursor, hasNextPage));
}
```

---

## Scenario-Based Questions

1. **Q: Your social media API fetches a user's feed by aggregating posts from 500 followed users. The database query times out at 30 seconds. How do you redesign this endpoint for <200ms response?**
   - A: (1) Pre-compute the feed asynchronously — when a user logs in or posts, write the feed to Redis sorted set (score = timestamp). Reading is O(log N) — just fetch the top 20. (2) Fan-out on write: when a user posts, insert into all followers' feed caches. (3) For high-profile users (millions of followers), switch to pull model: followers fetch posts from the celebrity's timeline, and the server merges it with the pre-computed feed. (4) Use cursor-based pagination to avoid offset shifting.

2. **Q: An e-commerce order placement endpoint receives duplicate requests from network retries, causing double charges. You implement idempotency keys, but your service is stateless and runs on 10 instances. How do you ensure idempotency across all instances?**
   - A: Store idempotency keys in a shared data store (Redis) accessible to all instances. Use `SET idempotency:<key> <status> NX EX 86400` (NX = only set if not exists) to atomically claim the key. First instance to execute the SET gets to process; others return the cached result. This is a distributed lock with TTL. For critical payments, use a database-level unique constraint on `(idempotency_key)` as backup.

3. **Q: Your REST API is slow because mobile clients call `GET /orders` then `GET /orders/{id}/items` for each order (N+1 network requests). How do you optimize without forcing clients to change?**
   - A: (1) Support embedding: `GET /orders?include=items` — the server joins orders with items and returns them in one structure. (2) Create a composite endpoint: `GET /orders-with-items` that returns everything in one call. (3) Use JSON:API spec's `?include=items` for standardized embedding. (4) Implement an API gateway that aggregates responses from multiple services. Trade-off: embedding increases response size — only embed when requested.

4. **Q: You need to support JSON and XML from the same REST API, but XML clients report missing fields while JSON clients get all data. What's happening?**
   - A: XML serialization likely fails on fields without proper annotations. Solutions: (1) Use Jackson's XML extension (`jackson-dataformat-xml`) instead of JAXB — same annotations work for both JSON and XML. (2) Annotate fields with `@JsonProperty` and `@JacksonXmlProperty` for consistent serialization. (3) Test both formats in integration tests. (4) Consider dropping XML support if usage is low (<1% of traffic) — the maintenance cost often exceeds the benefit.

5. **Q: Your API returns 500 errors intermittently under load. Error logs show "Connection pool exhausted" but CPU is only 30%. What's the real bottleneck and how do you fix it?**
   - A: The database connection pool is exhausted. 30% CPU means the app is waiting for database connections, not computing. Causes: (1) Connection pool too small (default HikariCP = 10). Fix: increase to `poolSize = Tomcat max-threads × (1 - blocking factor)`. With 200 threads and 30% blocking, target ~60 connections. (2) Slow queries holding connections longer. Fix: optimize queries, add indexes, increase `connectionTimeout`. (3) Connection leaks — not returning connections to pool. Fix: use connection pool monitoring (HikariCP metrics) and add leak detection.

6. **Q: A mobile client needs offline support. Users create orders offline, and sync when connectivity returns. How do you design conflict resolution?**
   - A: (1) Use ETags for optimistic concurrency — client sends `If-Match` header with the last known ETag. Server rejects if the resource changed since the client last synced. (2) Last-write-wins (simplest): server timestamps win. (3) CRDT (Conflict-Free Replicated Data Types): design endpoints to accept commutative operations — order of syncing doesn't matter. (4) Provide a `/sync` endpoint that returns all changes since a timestamp, using `Last-Modified` headers. (5) For critical conflicts (price changes, inventory depletion), return 409 Conflict with both versions — client must resolve.

7. **Q: How do you handle partial updates without forcing clients to send the full resource, while still being RESTful?**
   - A: (1) Use PATCH with JSON Merge Patch (RFC 7396): `PATCH /users/1` with body `{"name": "new name"}` — only the specified fields change. (2) Use JSON Patch (RFC 6902) for more complex operations: `[{ "op": "replace", "path": "/name", "value": "new name" }]`. (3) For simpler implementations, use POST with a `/users/{id}/fields` sub-resource pattern. (4) Spring Boot supports `@PatchMapping` with `HttpMessageConverter` — implement a custom `mergePatch()` that merges the patch into the existing entity.

8. **Q: Your payment API charges users via a third-party gateway. The gateway sometimes returns 200 OK but the charge actually failed (eventually consistent). Users see "Payment Successful" but no money was charged. How do you handle this?**
   - A: (1) Don't trust the gateway's initial response for critical operations. Implement a webhook callback pattern: the gateway processes asynchronously and sends a webhook with the final status. (2) Return `202 Accepted` immediately and update the order status when the webhook arrives. (3) Implement a reconciliation job that polls the gateway for pending transactions. (4) Add a "pending" order state — show "Payment processing" to the user until the webhook confirms. (5) Use idempotency keys to prevent duplicate webhook processing.

9. **Q: You are migrating from REST to GraphQL. REST endpoints have been public for 3 years with 500+ clients. How do you handle this transition safely?**
   - A: (1) Run both systems in parallel. Keep REST endpoints unchanged. Add `/graphql` endpoint. (2) Implement the strangler fig pattern: route new features through GraphQL, keep old REST for existing clients. (3) Build GraphQL resolvers that delegate to existing REST services internally — no need to rewrite business logic. (4) Send migration guides to clients and announce REST deprecation timeline (18+ months). (5) Monitor REST usage and only sunset endpoints when traffic drops to zero. (6) Use OpenAPI-to-GraphQL wrapper for automatic GraphQL schema generation from existing REST APIs.

10. **Q: How do you design rate limiting for a multi-tenant REST API where one tenant's burst traffic shouldn't affect others?**
    - A: (1) Tenant-level rate limiting using token bucket per tenant in Redis. Each tenant gets a `tenant:<id>:rate` key with a counter and TTL. (2) Endpoint-level rate limiting: different limits for expensive endpoints (reports = 10/min) vs cheap ones (list = 1000/min). (3) Global rate limiting to protect infrastructure. (4) Use sliding window (not fixed window) to prevent traffic spikes at window boundaries. (5) Return `429 Too Many Requests` with `Retry-After` header and `X-RateLimit-*` headers for transparency. (6) Implement priority queues — paid tenants get higher limits and priority during contention.

---

## Interview Questions

1. **What are the six constraints of REST?**
   - A: Uniform Interface, Stateless, Cacheable, Client-Server, Layered System, Code on Demand (optional). These ensure scalability, modifiability, and visibility in distributed systems.

2. **What is the difference between PUT and PATCH?**
   - A: PUT replaces the entire resource (idempotent). PATCH applies a partial modification (not necessarily idempotent). PUT sends the full resource; PATCH sends only the changes. PUT is idempotent — sending it N times has the same effect as once.

3. **What is HATEOAS?**
   - A: Hypermedia as the Engine of Application State (Level 3 Richardson Maturity Model). Responses include links to related actions, enabling clients to discover the API dynamically without out-of-band documentation.

4. **How do you implement pagination in REST?**
   - A: Offset pagination (`?page=0&size=20`) for static datasets. Cursor-based pagination (`?cursor=2024-01-01T00:00:00Z&limit=20`) for real-time feeds and large datasets. Keyset pagination for databases without offset performance issues. Return total count only when necessary (it's expensive).

5. **What are idempotency keys and why are they important?**
   - A: An `Idempotency-Key` header (UUID) ensures exactly-once processing despite retries. The server checks if the key was processed; if so, returns the cached result. Critical for payment, order placement, and any operation with side effects.

6. **How do you handle errors in REST APIs?**
   - A: Use appropriate HTTP status codes (201 for create, 400 for validation, 401 for auth, 404 for not found, 409 for conflict, 422 for unprocessable, 500 for server errors). Return a consistent error response format with code, message, and optional details. Never expose stack traces.

7. **What is content negotiation?**
   - A: The server determines the response format based on the client's `Accept` header. Clients specify desired format (JSON, XML, etc.). The server selects the appropriate `HttpMessageConverter`. Also supports versioning via custom media types.

8. **How do you secure a REST API?**
   - A: Use TLS for all communications. Implement authentication (JWT/OAuth2) and authorization (role/permission checks). Validate and sanitize all inputs. Rate limit per user and endpoint. Use security headers (HSTS, CSP, X-Content-Type-Options). Never expose internal IDs or stack traces.

9. **What is the N+1 query problem in REST APIs?**
   - A: When serializing a list of N resources, each resource triggers an additional query. Example: fetching 100 orders, then querying items for each order individually — 101 queries total. Fix: batch fetching, JOIN queries, or using a graph-based query layer.

10. **How do you version a REST API?**
    - A: URI path versioning (`/api/v1/users`) — most common, CDN-friendly. Header versioning (`Accept: application/vnd.myapp.v1+json`) — clean URLs, standards-based. Support maximum 3 active versions. Deprecate with 18+ months notice and sunset headers.

---

## Developer Recommendations

- **Always implement idempotency for mutation endpoints** — `POST /payments` and `POST /orders` will inevitably receive duplicate requests from network retries, mobile app retries, and client timeouts. An `Idempotency-Key` header prevents double charges and duplicate orders at minimal implementation cost. Store keys in Redis with TTL matching the business window (typically 24 hours). This is more reliable than relying on clients to deduplicate.

- **Use cursor-based pagination over offset-based for production APIs** — Offset pagination breaks when new records are inserted (pages shift, duplicates appear) and is slow on large offsets (`OFFSET 100000` scans and discards rows). Cursor-based pagination (using a unique sortable field like `created_at` or `id`) is O(log N) and stable regardless of concurrent inserts. Trade-off: no random page access (no "go to page 5"). Acceptable for most real-world APIs — users rarely jump to arbitrary pages.

- **Batch database queries to avoid N+1** — The N+1 problem is the single most common performance issue in REST APIs. When returning a list of resources that include related data, always batch-fetch the related data using `IN` clauses or JOIN queries. Use Spring's `@EntityGraph` or Hibernate's `JOIN FETCH` for JPA. For GraphQL, use DataLoader. Monitor query counts in production — a sudden increase in database queries per request indicates N+1 regression.

- **Design for API evolution from day one** — Start versioning before you think you need it. The cost of adding versioning later (breaking existing clients, coordinated deploys, migrations) dwarfs the minimal upfront cost of prefixing URIs with `/api/v1/`. Use additive-only changes for as long as possible. Deprecate before removing. Keep max 3 active versions. Add `Sunset` and `Deprecated` headers to responses.

- **Use appropriate HTTP status codes consistently** — Every status code tells the client what to do next: 201 for creation (client knows the resource was created), 204 for deletion (no content to return), 400 for bad request (client should fix the request), 401 for unauthenticated (client should log in), 403 for unauthorized (client lacks permission), 409 for conflict (client should retry with updated data), 422 for validation (client should fix specific fields). Inconsistent status codes force clients to parse error messages — always use the correct code.

- **Implement rate limiting before you need it** — A single abusive client can take down your entire API. Implement rate limiting from the first deployment. Use the token bucket algorithm with Redis for distributed rate limiting. Set per-tenant, per-endpoint, and global limits. Return `429 Too Many Requests` with `Retry-After` and `X-RateLimit-*` headers. Monitor rate limit hit rates — a rising trend indicates a client with a bug or an attacker probing your API.**
   A: Use `GET /api/v1/feed?page=0&size=20` returning a paginated response. Implement cursor-based pagination for real-time feeds to avoid duplicates when new posts are created. Use keyset pagination for better performance on large datasets.

2. **Q: An e-commerce order placement endpoint is receiving duplicate requests due to network retries, causing double charges. How do you solve this?**
   A: Implement idempotency using an `Idempotency-Key` header. The client generates a unique key per request. The server checks if the key was already processed; if so, returns the cached response instead of processing again.

3. **Q: Your REST API is slow because clients are making many sequential calls to fetch related data (e.g., get orders, then get items for each order). How do you optimize?**
   A: Implement bulk endpoints (`GET /orders?ids=1,2,3` with items included), add composite resources, support embedding via query params (`?include=items,user`), or use GraphQL for flexible client-driven queries.

4. **Q: You need to support both JSON and XML responses from the same REST API. How would you implement this?**
   A: Use content negotiation. Clients send `Accept: application/json` or `Accept: application/xml`. Spring Boot's `ContentNegotiationManager` automatically selects the appropriate `HttpMessageConverter`. Configure both Jackson and JAXB converters.

5. **Q: Your API returns 500 errors intermittently on high load. How do you diagnose the issue?**
   A: Enable request tracing with `MDC.put("requestId", requestId)`, log response times, monitor database connection pool usage (HikariCP metrics), check for thread pool exhaustion, and use Spring Boot Actuator for real-time metrics.

6. **Q: A mobile client needs offline support and data synchronization. How does this affect your REST API design?**
   A: Use optimistic locking with `ETag` and `If-Match` headers for conflict detection. Implement `Last-Modified` headers for incremental sync. Provide a `/sync` endpoint that returns changes since a timestamp.

7. **Q: How do you handle partial updates without forcing clients to send the full resource?**
   A: Use PATCH with JSON Patch (RFC 6902) or JSON Merge Patch (RFC 7396). Alternatively, use a `/users/{id}/fields` endpoint pattern where clients send only the changed fields.

8. **Q: Your payment API must ensure that a charge is processed exactly once, even if the client retries. What pattern do you use?**
   A: Idempotency keys. The client generates a UUID and sends it as `Idempotency-Key` header. The server stores the key with the result. On retry with the same key, return the stored result. Expire keys after 24 hours.

9. **Q: You are migrating from REST to GraphQL. How do you handle this transition without breaking existing clients?**
   A: Run both systems in parallel. Keep REST endpoints unchanged. Add a `/graphql` endpoint alongside. Redirect clients incrementally. Use the strangler fig pattern: route some traffic to GraphQL while keeping REST alive for legacy clients.

10. **Q: How do you design a rate-limiting strategy for a multi-tenant REST API?**
    A: Use token bucket or sliding window algorithm. Apply per-tenant, per-endpoint, and per-IP limits. Store counters in Redis for distributed rate limiting. Return `429 Too Many Requests` with `Retry-After` header. Queue excess requests when possible.
