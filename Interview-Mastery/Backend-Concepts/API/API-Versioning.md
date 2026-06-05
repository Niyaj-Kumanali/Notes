# API Versioning

## 1. Executive Summary

API versioning is the practice of managing changes to an API over time while maintaining backward compatibility for existing clients. As APIs evolve, endpoints, request formats, and response structures change. Versioning provides a structured way to introduce breaking changes, deprecate old functionality, and allow clients to migrate at their own pace. Common strategies include URI versioning, header versioning, query parameter versioning, and content negotiation.

## 2. Core Theory

### Why Versioning Matters

- **Backward Compatibility**: Existing clients continue working after changes.
- **Gradual Migration**: Clients can upgrade on their own schedule.
- **Experimentation**: New features can be tested without affecting production.
- **Contract Stability**: Clients depend on a stable API contract.
- **Deprecation Path**: Old versions can be sunsetted methodically.

### Types of API Changes

- **Backward-Compatible Changes**: Adding new fields, adding new endpoints, adding optional parameters.
- **Breaking Changes**: Removing fields, changing field types, renaming endpoints, changing request/response structure, removing endpoints, changing error formats.

### Semantic Versioning for APIs

```
Major.Minor.Patch (e.g., 2.1.0)
- Major: Breaking changes
- Minor: Backward-compatible additions
- Patch: Backward-compatible bug fixes
```

## 3. Under-the-Hood Deep Dive

### Versioning Strategy Comparison

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URI Path | `/api/v1/users` | Simple, discoverable, cacheable | URI pollution |
| Query Param | `/api/users?version=1` | Single base URI | Less discoverable, caching issues |
| Header | `Accept: application/vnd.myapp.v1+json` | Clean URLs, standards-based | Harder to test, hidden complexity |
| Content Type | `Content-Type: application/vnd.myapp.v1+json` | Media type negotiation | Complex client setup |

### Custom Versioning Annotations in Spring Boot

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ApiVersion {
    String value() default "v1";
}
```

## 4. Production Code Examples

### URI Versioning with Multiple Controller Versions

```java
// Version 1 Controller
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {

    private final UserService userService;

    public UserControllerV1(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponseV1> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(UserMapperV1.toResponse(user));
    }

    @GetMapping
    public ResponseEntity<List<UserResponseV1>> getAllUsers() {
        return ResponseEntity.ok(
            userService.findAll().stream()
                .map(UserMapperV1::toResponse)
                .collect(Collectors.toList())
        );
    }
}

// Version 2 Controller (with breaking changes)
@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {

    private final UserService userService;

    public UserControllerV2(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponseV2> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(UserMapperV2.toResponse(user));
    }

    @GetMapping
    public ResponseEntity<PagedResponse<UserResponseV2>> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Page<User> users = userService.findAll(PageRequest.of(page, size));
        return ResponseEntity.ok(
            PagedResponse.from(users.map(UserMapperV2::toResponse))
        );
    }
}
```

### Versioned Response DTOs

```java
// V1 Response - flat structure
public record UserResponseV1(
    Long id,
    String name,
    String email,
    String role
) {}

// V2 Response - nested structure with additional fields
public record UserResponseV2(
    String userId,          // Changed from Long to String
    String fullName,        // Renamed from "name"
    String emailAddress,    // Renamed from "email"
    ProfileResponse profile // Nested structure
) {}

public record ProfileResponse(
    String role,
    String department,
    Instant joinedAt,
    String status
) {}
```

### Mapping Between Versions

```java
@Component
public class UserMapperV1 {

    public static UserResponseV1 toResponse(User user) {
        return new UserResponseV1(
            user.getId(),
            user.getName(),
            user.getEmail(),
            user.getRole().getName()
        );
    }
}

@Component
public class UserMapperV2 {

    public static UserResponseV2 toResponse(User user) {
        return new UserResponseV2(
            user.getUuid(),               // UUID string instead of numeric ID
            user.getName(),
            user.getEmail(),
            new ProfileResponse(
                user.getRole().getName(),
                user.getDepartment(),
                user.getCreatedAt(),
                user.getStatus().name()
            )
        );
    }
}
```

### Header Versioning Strategy

```java
@Configuration
public class HeaderVersioningConfig implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(false)
            .ignoreAcceptHeader(false)
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("application/vnd.myapp.v1+json", MediaType.APPLICATION_JSON)
            .mediaType("application/vnd.myapp.v2+json", MediaType.APPLICATION_JSON);
    }
}

// Custom handler mapping for versioned controllers
@Component
public class VersionRequestMappingHandlerMapping extends RequestMappingHandlerMapping {

    @Override
    protected HandlerMethod lookupHandlerMethod(String lookupPath,
                                               HttpServletRequest request) throws Exception {
        String version = request.getHeader("Accept");
        if (version != null) {
            request.setAttribute("apiVersion", extractVersion(version));
        }
        return super.lookupHandlerMethod(lookupPath, request);
    }

    private String extractVersion(String acceptHeader) {
        Pattern pattern = Pattern.compile("application/vnd\\.myapp\\.(v\\d+)\\+json");
        Matcher matcher = pattern.matcher(acceptHeader);
        return matcher.find() ? matcher.group(1) : "v1";
    }
}
```

### Query Parameter Versioning

```java
@RestController
@RequestMapping("/api/users")
public class UserQueryVersionController {

    private final UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<?> getUser(
            @PathVariable Long id,
            @RequestParam(defaultValue = "1") int version) {

        User user = userService.findById(id);

        return switch (version) {
            case 1 -> ResponseEntity.ok(UserMapperV1.toResponse(user));
            case 2 -> ResponseEntity.ok(UserMapperV2.toResponse(user));
            default -> throw new UnsupportedApiVersionException(version);
        };
    }
}
```

### Interceptor-Based Versioning

```java
@Component
public class VersionInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler) throws Exception {

        if (handler instanceof HandlerMethod handlerMethod) {
            ApiVersion apiVersion = handlerMethod
                .getMethodAnnotation(ApiVersion.class);
            if (apiVersion == null) {
                apiVersion = handlerMethod
                    .getBeanType().getAnnotation(ApiVersion.class);
            }

            if (apiVersion != null) {
                String requestedVersion = resolveVersion(request);
                if (!apiVersion.value().equals(requestedVersion)) {
                    response.setStatus(404);
                    response.getWriter()
                        .write("{\"error\":\"Version not found\"}");
                    return false;
                }
            }
        }
        return true;
    }

    private String resolveVersion(HttpServletRequest request) {
        // Extract from header, URI, or query param
        String fromHeader = request.getHeader("X-API-Version");
        if (fromHeader != null) return fromHeader;

        String path = request.getRequestURI();
        Pattern pattern = Pattern.compile("/api/(v\\d+)/");
        Matcher matcher = pattern.matcher(path);
        return matcher.find() ? matcher.group(1) : "v1";
    }
}
```

### Content Negotiation Versioning

```java
@RestController
@RequestMapping("/api/users")
public class ContentNegotiationController {

    @GetMapping(produces = "application/vnd.myapp.v1+json")
    public ResponseEntity<List<UserResponseV1>> getAllUsersV1() {
        return ResponseEntity.ok(
            userService.findAll().stream()
                .map(UserMapperV1::toResponse)
                .collect(Collectors.toList())
        );
    }

    @GetMapping(produces = "application/vnd.myapp.v2+json")
    public ResponseEntity<PagedResponse<UserResponseV2>> getAllUsersV2(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(
            PagedResponse.from(
                userService.findAll(PageRequest.of(page, size))
                    .map(UserMapperV2::toResponse)
            )
        );
    }
}
```

### Abstract Versioned Controller Pattern

```java
public abstract class AbstractVersionedController<T, R> {

    protected final UserService userService;

    protected AbstractVersionedController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<R> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(toResponse(user));
    }

    protected abstract R toResponse(User user);
}

@RestController
@RequestMapping("/api/v1/users")
public class UserV1Controller extends AbstractVersionedController<User, UserResponseV1> {

    public UserV1Controller(UserService userService) {
        super(userService);
    }

    @Override
    protected UserResponseV1 toResponse(User user) {
        return UserMapperV1.toResponse(user);
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserV2Controller extends AbstractVersionedController<User, UserResponseV2> {

    public UserV2Controller(UserService userService) {
        super(userService);
    }

    @Override
    protected UserResponseV2 toResponse(User user) {
        return UserMapperV2.toResponse(user);
    }
}
```

### Deprecation Headers

```java
@RestController
@RequestMapping("/api/v1/users")
public class DeprecatedUserController {

    @GetMapping("/{id}")
    public ResponseEntity<UserResponseV1> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok()
            .header("X-API-Deprecated", "true")
            .header("X-API-Sunset", "2026-12-31")
            .header("X-API-Migration", "/api/v2/users/" + id)
            .body(UserMapperV1.toResponse(user));
    }
}
```

### Versioning with a Custom Filter

```java
@Component
public class VersionRoutingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String requestURI = httpRequest.getRequestURI();
        String version = extractVersion(requestURI);

        if (version == null) {
            version = httpRequest.getHeader("X-API-Version");
        }

        if (version == null) {
            version = "v1";
        }

        // Forward to appropriate version path
        String versionedPath = "/api/" + version +
            requestURI.replaceFirst("/api/v?\\d*", "");

        httpRequest.getRequestDispatcher(versionedPath)
            .forward(httpRequest, httpResponse);
    }

    private String extractVersion(String uri) {
        Pattern pattern = Pattern.compile("/api/v?(\\d+)");
        Matcher matcher = pattern.matcher(uri);
        return matcher.find() ? "v" + matcher.group(1) : null;
    }
}
```

## 5. Real-World Scenarios

### Scenario 1: Social Media API Evolution

```
v1 (2020):
  GET /api/v1/posts/{id} -> { id, title, body, author_id }

v2 (2022) - Breaking changes:
  GET /api/v2/posts/{id} -> { id, title, body, author: { id, name, avatar }, stats: { likes, shares, comments } }

Migration:
  - Added author detail object instead of flat author_id
  - Added stats sub-object
  - Maintained v1 endpoint for legacy clients
  - Deprecation header on v1 responses
  - v1 sunset after 2 years
```

### Scenario 2: E-Commerce Checkout API

```java
// V1 Checkout - simple flow
@PostMapping("/api/v1/checkout")
public ResponseEntity<OrderResponseV1> checkoutV1(@Valid @RequestBody CheckoutRequestV1 request) {
    Order order = orderService.createOrder(request.getItems(), request.getShippingAddressId());
    return ResponseEntity.status(201).body(new OrderResponseV1(order.getId(), "CREATED"));
}

// V2 Checkout - with payment method selection and promotions
@PostMapping("/api/v2/checkout")
public ResponseEntity<OrderResponseV2> checkoutV2(@Valid @RequestBody CheckoutRequestV2 request) {
    Order order = orderService.createOrder(
        CheckoutMapper.toOrderRequest(request));
    PaymentIntent paymentIntent = paymentService.createPaymentIntent(order);
    return ResponseEntity.status(201).body(
        new OrderResponseV2(order.getId(), "PENDING_PAYMENT", paymentIntent.getClientSecret()));
}
```

### Scenario 3: Coexistence Strategy

```yaml
# application.yml
api:
  versions:
    active:
      - v1
      - v2
      - v3
    deprecated:
      v1:
        sunset-date: 2026-12-31
        migration-path: /api/v2/users
      v2:
        sunset-date: 2027-06-30
        migration-path: /api/v3/users
```

## 6. Performance

### Versioning Performance Considerations

- **URI Versioning**: Fastest (no parsing required), cached by CDN.
- **Header Versioning**: Slightly slower (header parsing), less CDN-friendly.
- **Query Parameter Versioning**: Caching issues (same URI different version).
- **Multiple Controller Instances**: Memory overhead per version.
- **Version Routing Logic**: Minimize branching in request path.

### Optimized Version Routing

```java
@Component
public class VersionRouter {

    private final Map<String, HandlerFunction> versionHandlers = new HashMap<>();

    public VersionRouter() {
        versionHandlers.put("v1", this::handleV1);
        versionHandlers.put("v2", this::handleV2);
        versionHandlers.put("v3", this::handleV3);
    }

    public ResponseEntity<?> route(String version, Long id) {
        HandlerFunction handler = versionHandlers.get(version);
        if (handler == null) {
            return ResponseEntity.status(404).body(
                new ErrorResponse("UNSUPPORTED_VERSION",
                    "API version " + version + " is not supported"));
        }
        return handler.handle(id);
    }

    private ResponseEntity<?> handleV1(Long id) {
        return ResponseEntity.ok(UserMapperV1.toResponse(userService.findById(id)));
    }

    private ResponseEntity<?> handleV2(Long id) {
        return ResponseEntity.ok(UserMapperV2.toResponse(userService.findById(id)));
    }

    private ResponseEntity<?> handleV3(Long id) {
        return ResponseEntity.ok(UserMapperV3.toResponse(userService.findById(id)));
    }

    @FunctionalInterface
    interface HandlerFunction {
        ResponseEntity<?> handle(Long id);
    }
}
```

## 7. Security

### Version-Specific Security

```java
@Configuration
@EnableWebSecurity
public class VersionSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                // V1 endpoints require only basic auth
                .requestMatchers("/api/v1/**").authenticated()
                // V2 endpoints require specific scope
                .requestMatchers("/api/v2/**")
                    .hasAuthority("SCOPE_api:read")
                // V3 endpoints require elevated permissions
                .requestMatchers("/api/v3/**")
                    .hasAuthority("SCOPE_api:admin")
            )
            .build();
    }
}
```

## 8. Common Mistakes

- **Not versioning from day one**: Harder to add later.
- **Breaking changes without version bump**: Always bump major version for breaking changes.
- **Supporting too many versions**: Limit active versions to 2-3.
- **Inconsistent versioning strategy**: Pick one strategy and stick with it.
- **Forgetting to sunset old versions**: Have a clear deprecation and sunset policy.
- **Versioning at the wrong level**: Version at API level, not individual endpoints.
- **Not documenting version differences**: Maintain clear migration guides.
- **Removing fields without notice**: Deprecate fields first, remove later.

## 9. Senior Engineer Perspective

### API Evolution Strategy

```mermaid
Flow:
1. Add field to response (backward-compatible)
2. Mark old field as deprecated with sunset header
3. Add new endpoint structure alongside old one
4. Migrate internal services and clients
5. Remove deprecated features after sunset date
```

### Best Practices

- **Maximum 3 active versions**: v1 (deprecated), v2 (current), v3 (preview).
- **Minimum 18-month deprecation period**: Give clients time to migrate.
- **Automated version sunset**: Automatically reject requests to sunset versions.
- **Versioning in the contract**: OpenAPI specs should document all versions.
- **Internal vs External APIs**: Use different versioning strategies.
- **Graceful degradation**: Old versions get basic support, new versions get full features.

### Version Lifecycle

```
Development -> Preview -> Active -> Deprecated -> Sunset
```

## 10. Interview Questions (Easy)

1. What is API versioning and why is it needed?
2. Name four common API versioning strategies.
3. What is the difference between backward-compatible and breaking changes?
4. What is semantic versioning for APIs?
5. How does URI path versioning work?
6. What is the purpose of a deprecation header?
7. What is query parameter versioning?
8. What are the pros and cons of URI versioning?
9. What is header versioning?
10. How many active API versions should you typically support?

## Medium

1. Compare URI path versioning vs header versioning.
2. How do you handle versioning in Spring Boot?
3. What is content negotiation versioning?
4. How do you implement API deprecation with sunset dates?
5. What is the strangler fig pattern and how does it apply to API versioning?
6. How would you handle simultaneous version support in a database?
7. How do you manage version-specific request validation?
8. What is the difference between versioning at the API vs resource level?
9. How do you test multiple API versions?
10. How do you document versioned APIs?

## 11. Advanced Interview Questions (Hard)

1. Design a versioning system that supports both URI and header versioning simultaneously.
2. How would you implement a gradual migration from v1 to v2 without downtime?
3. Design an API that supports both JSON and Protocol Buffers across versions.
4. How do you handle database schema changes when supporting multiple API versions?
5. Implement a request transformer that converts v1 requests to v2 internally.
6. Design a canary deployment strategy for API version rollouts.
7. How would you implement version-specific rate limiting and throttling?
8. Design an automated version backwards compatibility testing framework.
9. How do you handle versioning in an event-driven microservices architecture?
10. Implement a version-aware API gateway.

## System Design

1. Design an API versioning system for a SaaS platform with 1000+ customers.
2. Design a version management system for a public API with millions of consumers.
3. Design a multi-version API gateway with request routing and transformation.
4. Design an API deprecation notification system.
5. Design a version-aware documentation system.
6. Design a backward compatibility testing pipeline for CI/CD.
7. Design an API version analytics and monitoring system.
8. Design a version negotiation protocol for real-time APIs.
9. Design a schema evolution system for gRPC APIs.
10. Design a versioned webhook delivery system.

## 12. Expert-Level Interview Questions (Architect-Level)

1. Design a versionless API strategy where all changes are backward-compatible by default using extensible schemas, and breaking changes are handled through capability negotiation rather than version numbers.
2. How would you implement a polyglot API versioning system across multiple microservices written in different languages, with a unified versioning contract?
3. Design an API versioning strategy for an event-sourced system where the event schema evolves over time independently of the API schema.
4. How do you handle versioning in a CQRS architecture where commands and queries evolve at different rates?
5. Design a system that automatically generates version migration code and documentation from schema diffs between API versions.
6. How would you implement versioning for a real-time streaming API (WebSocket/gRPC) where the protocol itself evolves?
7. Design a distributed API version registry that allows clients to discover available versions and their capabilities dynamically.
8. How do you handle versioning in a BFF (Backend for Frontend) pattern where each client type may need different API evolution paths?
9. Design an API versioning strategy for a regulatory compliance system where historical data must be accessible via older API versions indefinitely.
10. How would you implement a self-healing API version migration system that detects client usage patterns and automatically adapts version support?

## 13. Debugging & Troubleshooting

### Common Issues

- **404 Not Found for valid paths**: Check version prefix and routing configuration.
- **Wrong version returned**: Verify header parsing and routing logic.
- **Deprecated version still in use**: Monitor usage and communicate sunset dates.
- **Version mismatch between services**: Align version contracts across all services.
- **Caching returns wrong version**: Ensure caching keys include version.

### Version Metrics

```java
@Component
public class VersionMetrics {

    private final MeterRegistry meterRegistry;

    public VersionMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void recordVersionUsage(HttpServletRequest request) {
        String version = extractVersion(request.getRequestURI());
        if (version == null) {
            version = request.getHeader("X-API-Version");
        }
        meterRegistry.counter("api.requests", "version", version).increment();
    }

    public void recordDeprecatedCall(String version) {
        meterRegistry.counter("api.deprecated.calls", "version", version).increment();
    }
}
```

## 14. Comparison Section

### URI Versioning vs Header Versioning

| Aspect | URI Versioning | Header Versioning |
|--------|---------------|-------------------|
| Simplicity | Very simple | Complex |
| Discoverability | High (visible in URL) | Low (hidden in headers) |
| Caching | CDN-friendly | CDN-unfriendly |
| RESTful Purity | Less pure (URLs change) | More pure (same URL) |
| Default Version | Easy (v1 implied) | Harder to determine |
| Browser Testing | Easy (type in URL) | Requires tools |
| API Documentation | Clear version separation | Version logic needed |

### Versioning Strategy Decision Matrix

```
                    URI    Header    Query    Content-Type
Simple               Y       N         Y          N
Cacheable            Y       N         N          Y
Standards-based      N       Y         N          Y
Discoverable         Y       N         Y          N
Easy to test         Y       N         Y          N
```

## 15. Revision Notes

- Four main strategies: URI, Header, Query Param, Content-Type
- Semantic versioning: Major.Minor.Patch
- Backward-compatible: adding fields, endpoints, optional params
- Breaking: removing/renaming fields, changing types, removing endpoints
- Support max 3 versions: deprecated, active, preview
- Minimum 18-month deprecation period
- Use deprecation headers: `X-API-Deprecated`, `X-API-Sunset`, `X-API-Migration`
- Spring Boot: separate controllers per version or routing logic
- Strangler fig pattern for gradual migration
- Always version from day one

## 16. Cheat Sheet

```
+------------------------------------------------------------------+
| API VERSIONING CHEAT SHEET                                       |
+------------------------------------------------------------------+
| VERSIONING STRATEGIES                                            |
|   URI Path  : /api/v1/resource  /api/v2/resource                |
|   Header    : Accept: application/vnd.myapp.v1+json             |
|   Query     : /api/resource?version=1                            |
|   Content   : Content-Type: application/vnd.myapp.v1+json        |
+------------------------------------------------------------------+
| CHANGE TYPES                                                     |
|   Backward-Compatible: Add fields, add endpoints, add optional   |
|   Breaking          : Remove/rename fields, change types,        |
|                       remove endpoints, change error format      |
+------------------------------------------------------------------+
| DEPRECATION HEADERS                                              |
|   X-API-Deprecated: true                                        |
|   X-API-Sunset    : 2026-12-31                                  |
|   X-API-Migration : /api/v2/resource/{id}                       |
+------------------------------------------------------------------+
| SPRING BOOT - URI VERSIONING                                     |
|   @RestController                                                |
|   @RequestMapping("/api/v1/users")  public class V1Controller   |
|   @RequestMapping("/api/v2/users")  public class V2Controller   |
+------------------------------------------------------------------+
| SPRING BOOT - CONTENT NEGOTIATION                                |
|   @GetMapping(produces = "application/vnd.myapp.v1+json")       |
+------------------------------------------------------------------+
| VERSION LIFECYCLE                                                |
|   Dev -> Preview -> Active -> Deprecated -> Sunset              |
+------------------------------------------------------------------+
| BEST PRACTICES                                                   |
|   1. Version from day one                                        |
|   2. Max 3 active versions                                       |
|   3. Min 18-month deprecation period                             |
|   4. Use deprecation headers                                     |
|   5. Monitor version usage                                       |
|   6. Automate sunset enforcement                                 |
+------------------------------------------------------------------+
```
