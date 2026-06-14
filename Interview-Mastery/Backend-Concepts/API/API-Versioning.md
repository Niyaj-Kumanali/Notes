# API Versioning

---

## Overview

- **Definition:** API versioning is the practice of managing changes to an API over time while maintaining backward compatibility for existing clients. It provides a structured way to introduce breaking changes, deprecate old functionality, and allow clients to migrate at their own pace without experiencing sudden service disruptions. Versioning is essentially a contract between the API provider and consumers that guarantees stability within a given version boundary, enabling the provider to iterate on the API while consumers can plan and execute upgrades on their own schedules.

- **Why It Exists:** APIs evolve — endpoints change, request/response formats shift, and fields are added or removed. Without versioning, every change risks breaking existing clients, making it impossible to iterate safely. In a production environment with hundreds or thousands of consumers — including mobile apps that cannot be updated instantly, third-party integrations maintained by other teams, and automated systems running in unattended mode — any breaking change can cause cascading failures across the entire ecosystem. Versioning provides a safety buffer that allows both providers and consumers to evolve independently.

- **Key Goals:**
  - **Backward Compatibility** — existing clients continue working after changes are introduced, ensuring that no consumer experiences a regression simply because the API was updated. This is the foundational principle of versioning: old clients must never break when new versions are released.
  - **Gradual Migration** — clients upgrade on their own schedule rather than being forced to move synchronously with the API provider. This is crucial for mobile apps with long update cycles, enterprise clients with change-control boards, and systems with compliance requirements that prohibit frequent updates.
  - **Contract Stability** — clients depend on a stable API contract that will not change unexpectedly, giving them confidence to build integrations, write automated tests, and rely on the API for business-critical operations. A versioned contract is legally and technically binding within its lifecycle.
  - **Clear Deprecation Path** — old versions are sunsetted methodically with predictable timelines, allowing consumers to plan migrations well in advance and preventing the chaos of abrupt shutdowns. A well-defined deprecation policy includes announcement, grace period, sunset date, and post-sunset behavior.

---

## Change Types

- **Backward-Compatible Changes:** These are modifications that add functionality without breaking any existing client code. Existing integrations continue to work without modification, while new clients can take advantage of the additions.
  - Adding new fields to responses — clients that parse strictly may still break if they validate against a known schema; this is technically backward-compatible but practically requires client awareness.
  - Adding new endpoints — no existing flow is affected because no client is calling the new resource yet.
  - Adding optional parameters — existing requests without the parameter continue to work identically; the server provides default behavior when the parameter is absent.
  - Extending enumerations — adding new enum values is backward-compatible for clients that handle unknown values gracefully, but can break clients with exhaustive switch statements or strict enum validation.
  - Relaxing input constraints — making a previously required field optional, widening a validation range, or accepting additional formats for the same data.
  - Adding new HTTP methods to existing endpoints — for example, adding `PATCH` alongside `PUT` on a resource.

- **Breaking Changes:** These modifications require existing clients to update their code before they can continue using the API. Even a single breaking change without a version bump can cause production incidents across the consumer ecosystem.
  - Removing fields from responses — clients that read the removed field will receive `undefined` or `null` and may crash or produce incorrect results.
  - Changing field types — for example, changing `id` from `integer` to `string` breaks strongly typed clients, SQL queries that compare IDs, and URL construction that assumes numeric values.
  - Renaming endpoints or fields — any client code referencing the old name breaks; this includes mobile apps with hardcoded URLs, SDKs, and documentation.
  - Changing request/response structure — flattening a nested object, nesting a flat field, or changing array formats (e.g., from comma-separated to JSON array).
  - Removing endpoints — all consumers calling that endpoint receive `404 Not Found` or `410 Gone` immediately.
  - Changing error formats — clients that parse error messages, codes, or structures will misinterpret or fail to handle errors.
  - Adding new required fields to requests — existing requests that omit the new field will be rejected.
  - Changing authentication or authorization requirements — clients that were previously allowed may be denied access.
  - Modifying pagination behavior — changing default page sizes, cursor formats, or pagination metadata structure.
  - Changing rate limiting behavior or headers — automated retry logic may break if headers are renamed or removed.

- **Semantic Versioning for APIs:**
  - `Major.Minor.Patch` (e.g., 2.1.0) follows the same principles as software libraries but applied to API contracts.
  - **Major** — breaking changes that require client modifications. Incrementing the major version signals that backward compatibility is intentionally broken and clients must migrate.
  - **Minor** — backward-compatible additions such as new endpoints, optional fields, and new enum values that do not disrupt existing clients.
  - **Patch** — backward-compatible bug fixes that change behavior without altering the contract; for example, fixing a response that previously returned incorrect data while keeping the structure identical.
  - In practice, API versioning often only tracks the major version (`v1`, `v2`) because minor and patch changes are transparent to clients. However, including semantic version information in API metadata helps clients understand the nature of changes between releases.

---

## Versioning Strategies

- **URI Path Versioning:**
  - Example: `/api/v1/users`, `/api/v2/users`
  - **Pros:** simple, discoverable, CDN-cacheable, easy to test with any HTTP client, straightforward to implement with most web frameworks, and the version is immediately visible in logs and monitoring. No special headers or content negotiation logic is required.
  - **Cons:** URI pollution (the version is part of the URL structure), less RESTful (the resource identity changes with version), and can lead to URL proliferation. Some argue the resource `/api/v1/users` and `/api/v2/users` are technically different resources since they have different URLs. Additionally, changing the version requires updating all client code that constructs URLs.
  - **Best for:** Most public APIs, services with simple versioning needs, and teams that prioritize operational simplicity. This is the most widely adopted strategy across the industry.

- **Header Versioning:**
  - Example: `Accept: application/vnd.myapp.v1+json`
  - **Pros:** clean URLs (the resource path stays the same across versions), standards-based (uses HTTP content negotiation), and does not pollute the URI namespace. The version is expressed as part of the media type, which is semantically correct in REST terms.
  - **Cons:** harder to test and debug (the version is hidden in a header and not visible in the URL), requires custom header parsing, not directly cacheable by CDNs without custom configuration, and harder to discover for new API consumers. Browsers and simple tools like `curl` require explicit header configuration.
  - **Best for:** Internal APIs, services where URI cleanliness is paramount, and teams with sophisticated API gateways that can transform headers.

- **Query Parameter Versioning:**
  - Example: `/api/users?version=1`
  - **Pros:** single base URI for all versions, easy to implement, trivial to test (just append a query parameter), and straightforward to understand. It also makes it easy to switch versions during development by simply changing the parameter value.
  - **Cons:** caching issues (query parameters affect caching behavior), less discoverable, and can be problematic with URL length limits. Query parameters are often stripped by proxies or ignored by CDN caches, and they pollute analytics and logging data. The same logical resource has different representations based on a parameter, which is semantically questionable.
  - **Best for:** Simple services, internal tools, and transitional versioning before migrating to a more robust strategy.

- **Content Negotiation Versioning:**
  - Example: `Content-Type: application/vnd.myapp.v1+json`
  - **Pros:** separates the version from the URL and query string, uses standard HTTP mechanisms, and aligns with REST principles where the resource identity remains unchanged. The version is part of the representation negotiation, which is conceptually clean.
  - **Cons:** complex client setup, difficult to test without proper HTTP libraries, and less common in practice, which means developers may be unfamiliar with the pattern. It also requires both client and server to support custom media types, adding friction for simple integrations.
  - **Best for:** REST purists, APIs with complex media type requirements, and teams that already use content negotiation extensively.

- **Custom Header Versioning:**
  - Example: `X-API-Version: 1`
  - **Pros:** simple to implement and parse, explicit version declaration, and no need to modify Accept headers. Custom headers are easy to add to any HTTP client and straightforward to read in middleware.
  - **Cons:** non-standard (X- headers are deprecated by RFC 6648), can conflict with other custom headers, not cacheable by default, and requires custom server logic to parse and route. Also not discoverable through standard API documentation tools.
  - **Best for:** Internal services, transitional periods, and cases where header versioning is needed but content negotiation feels overly complex.

- **Hybrid Approach:**
  - Many production APIs combine strategies. For example, use URI path versioning as the primary mechanism (`/api/v2/orders`) but also support content negotiation for specific endpoints. The hybrid approach allows different parts of the API to evolve at different paces while maintaining a consistent consumer experience. The key is to document the strategy clearly and avoid confusing clients with too many versioning mechanisms.

---

## Production Code Examples

### URI Versioning with Multiple Controllers

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    private final UserService userService;

    public UserControllerV1(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponseV1> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(UserMapperV1.toResponse(userService.findById(id)));
    }

    @GetMapping
    public ResponseEntity<List<UserResponseV1>> getAllUsers() {
        return ResponseEntity.ok(userService.findAll().stream()
            .map(UserMapperV1::toResponse)
            .collect(Collectors.toList()));
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    private final UserService userService;

    public UserControllerV2(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponseV2> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(UserMapperV2.toResponse(userService.findById(id)));
    }

    @GetMapping
    public ResponseEntity<PagedResponse<UserResponseV2>> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Page<User> users = userService.findAll(PageRequest.of(page, size));
        return ResponseEntity.ok(PagedResponse.from(users.map(UserMapperV2::toResponse)));
    }
}
```

### Versioned DTOs

```java
// V1 - flat structure
public record UserResponseV1(
    Long id,
    String name,
    String email,
    String role
) {}

// V2 - nested structure with breaking changes
public record UserResponseV2(
    String userId,          // Changed from Long to String
    String fullName,        // Renamed from "name"
    String emailAddress,    // Renamed from "email"
    ProfileResponse profile // Nested structure replacing flat role field
) {}

public record ProfileResponse(
    String role,
    String department,
    Instant joinedAt,
    String status
) {}
```

### Header Versioning Configuration

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
```

### Content Negotiation Versioning

```java
@RestController
@RequestMapping("/api/users")
public class ContentNegotiationController {

    @GetMapping(produces = "application/vnd.myapp.v1+json")
    public ResponseEntity<List<UserResponseV1>> getAllUsersV1() {
        return ResponseEntity.ok(userService.findAll().stream()
            .map(UserMapperV1::toResponse)
            .collect(Collectors.toList()));
    }

    @GetMapping(produces = "application/vnd.myapp.v2+json")
    public ResponseEntity<PagedResponse<UserResponseV2>> getAllUsersV2(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(PagedResponse.from(
            userService.findAll(PageRequest.of(page, size))
                .map(UserMapperV2::toResponse)));
    }
}
```

### Middleware-Based Version Routing (Express.js)

```javascript
// Middleware that routes to version-specific handlers
function versionRouter(req, res, next) {
  const version = req.path.match(/^\/api\/v(\d+)\//)?.[1];

  if (!version) {
    return res.status(400).json({
      error: 'API version is required',
      message: 'Use /api/v1/ or /api/v2/ prefix',
      availableVersions: ['v1', 'v2']
    });
  }

  if (!['1', '2'].includes(version)) {
    return res.status(400).json({
      error: 'Unsupported API version',
      message: `Version v${version} is not supported`,
      supportedVersions: ['v1', 'v2']
    });
  }

  req.apiVersion = parseInt(version);
  next();
}

// Usage in Express app
app.use('/api', versionRouter);
app.use('/api/v1/users', v1UserRouter);
app.use('/api/v2/users', v2UserRouter);
```

### Version Routing with Request Header Interceptor

```java
@Component
public class VersionInterceptor implements HandlerInterceptor {
    private static final String VERSION_HEADER = "X-API-Version";
    private static final String VERSION_ATTRIBUTE = "apiVersion";

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) {
        String version = request.getHeader(VERSION_HEADER);

        if (version == null || version.isEmpty()) {
            // Default to latest version
            request.setAttribute(VERSION_ATTRIBUTE, ApiVersion.V2);
            return true;
        }

        ApiVersion apiVersion = ApiVersion.fromString(version);
        if (apiVersion == null) {
            response.setStatus(400);
            return false;
        }

        request.setAttribute(VERSION_ATTRIBUTE, apiVersion);
        return true;
    }
}

public enum ApiVersion {
    V1("1"), V2("2"), V3("3");

    private final String version;

    ApiVersion(String version) {
        this.version = version;
    }

    public static ApiVersion fromString(String v) {
        for (ApiVersion apiVersion : values()) {
            if (apiVersion.version.equals(v)) {
                return apiVersion;
            }
        }
        return null;
    }
}
```

### API Gateway Version Routing (Spring Cloud Gateway)

```java
@Bean
public RouteLocator versionedRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("users-v1", r -> r
            .path("/api/v1/users/**")
            .uri("http://users-service-v1:8080"))
        .route("users-v2", r -> r
            .path("/api/v2/users/**")
            .uri("http://users-service-v2:8080"))
        .route("header-version-routing", r -> r
            .path("/api/users/**")
            .and().header("Accept", "application/vnd.myapp.v2+json")
            .uri("http://users-service-v2:8080"))
        .build();
}
```

### Deprecation Headers

```java
@RestController
@RequestMapping("/api/v1/users")
public class DeprecatedUserController {

    @GetMapping("/{id}")
    public ResponseEntity<UserResponseV1> getUser(@PathVariable Long id) {
        return ResponseEntity.ok()
            .header("X-API-Deprecated", "true")
            .header("X-API-Sunset", "2026-12-31")
            .header("X-API-Migration", "/api/v2/users/" + id)
            .body(UserMapperV1.toResponse(userService.findById(id)));
    }
}
```

### Automated Version Sunset Filter

```java
@Component
public class VersionSunsetFilter implements Filter {

    private static final Map<String, LocalDate> SUNSET_DATES = Map.of(
        "v1", LocalDate.of(2026, 12, 31),
        "v2", LocalDate.of(2028, 6, 30)
    );

    @Override
    public void doFilter(ServletRequest servletRequest,
                        ServletResponse servletResponse,
                        FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;

        String version = extractVersion(request.getRequestURI());
        if (version != null && SUNSET_DATES.containsKey(version)) {
            LocalDate sunsetDate = SUNSET_DATES.get(version);
            if (LocalDate.now().isAfter(sunsetDate)) {
                response.setStatus(410);
                response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                response.getWriter().write(String.format(
                    "{\"error\":\"Version %s is no longer supported\",\"migration\":\"/api/v3/\"}",
                    version));
                return;
            }
            // Still active but deprecated
            response.setHeader("X-API-Deprecated", "true");
            response.setHeader("X-API-Sunset", sunsetDate.toString());
        }

        chain.doFilter(request, response);
    }

    private String extractVersion(String uri) {
        Pattern pattern = Pattern.compile("/api/v(\\d+)/");
        Matcher matcher = pattern.matcher(uri);
        return matcher.find() ? "v" + matcher.group(1) : null;
    }
}
```

### Client Version Detection and Adaptive Responses

```java
@RestController
@RequestMapping("/api/users")
public class AdaptiveController {

    @GetMapping("/{id}")
    public ResponseEntity<?> getUser(
            @PathVariable Long id,
            @RequestHeader("X-Client-Version") String clientVersion) {

        // Parse client version to determine capabilities
        ClientVersion version = ClientVersion.parse(clientVersion);

        if (version.isOlderThan(2, 0)) {
            // Old clients get V1 response format
            UserResponseV1 response = UserMapperV1.toResponse(userService.findById(id));
            return ResponseEntity.ok()
                .header("X-API-Deprecated", "true")
                .header("X-API-Sunset", "2026-12-31")
                .header("X-Client-Upgrade-Recommended", "true")
                .body(response);
        }

        // New clients get V2 response format
        UserResponseV2 response = UserMapperV2.toResponse(userService.findById(id));
        return ResponseEntity.ok(response);
    }
}
```

### Backward Compatibility Test with OpenAPI Diff

```java
// Pseudo-code for CI/CD backward compatibility check
public class BackwardCompatibilityCheck {

    public static void main(String[] args) throws Exception {
        OpenAPI currentSpec = OpenAPIParser.parse("openapi-v1.yaml");
        OpenAPI newSpec = OpenAPIParser.parse("openapi-v2.yaml");

        List<Change> breakingChanges = OpenAPIDiff.compare(currentSpec, newSpec)
            .getBreakingChanges();

        if (!breakingChanges.isEmpty()) {
            System.err.println("BREAKING CHANGES DETECTED:");
            for (Change change : breakingChanges) {
                System.err.printf("- %s: %s%n",
                    change.getType(), change.getDescription());
            }
            System.exit(1); // Fail the build
        }

        System.out.println("No breaking changes detected. ✓");
    }
}
```

---

## Common Mistakes

- **Not versioning from day one** — Adding versioning after launch is exponentially harder because you must retroactively decide what the unversioned API means, migrate existing clients who never opted into versioning, and potentially break integrations that assumed stability. Always start with `/api/v1/` even if you are not sure you need it. The cost is negligible and the insurance is invaluable.
  - **Why it looks correct:** with no external clients yet, adding versioning feels like speculative overhead — until the first mobile app ships and a required field rename breaks every installed copy.

- **Breaking changes without version bump** — Every breaking change must increment the major version, even if it seems minor. A field rename, type change, or error format modification can silently break clients in production. The version number is a signal to consumers; ignoring this signal erodes trust in the entire API. Use automated tools in CI/CD to detect breaking changes and enforce version bumps.
  - **Why it looks correct:** the change seems small and harmless to the team that made it, and they assume clients won't notice — but automated parsers and strongly typed clients break silently when a field name or type changes.

- **Supporting too many versions** — Each active version multiplies the maintenance burden: separate controllers, DTOs, mappers, tests, documentation, and monitoring. Limit active versions to 2-3 maximum: one deprecated (migration in progress), one current (actively developed), and one preview (beta testers). Sunset versions aggressively once migration drops below a threshold (e.g., 1% of traffic).
  - **Why it looks correct:** keeping old versions running seems harmless — they're already built — and each client that hasn't migrated yet seems like a reason to keep them alive, but the cumulative maintenance cost grows linearly with each version.

- **Inconsistent versioning strategy** — Mixing URI path, header, and query parameter versioning across different endpoints creates confusion for consumers. Every developer must learn multiple patterns to interact with the API, and automated tools (API clients, SDK generators, documentation platforms) may not support all strategies uniformly. Pick one strategy, document it, and apply it consistently across all endpoints.
  - **Why it looks correct:** different teams own different endpoints, and each team picks what feels best for their use case — the inconsistency only becomes visible when a single client tries to integrate with all endpoints.

- **Forgetting to sunset old versions** — Old versions accumulate indefinitely unless actively retired. Without sunset dates, you end up maintaining v1 through v5 simultaneously, each with its own deployment complexity. Have a clear deprecation and sunset policy from the start: announce, set a date, communicate repeatedly, and then enforce the shutdown with a `410 Gone` response.
  - **Why it looks correct:** no single client seems worth breaking, so keeping every version "just in case" feels like good customer service — until the team spends 40% of every sprint backporting security fixes to five different code paths.

- **Not documenting version differences** — Clients need clear, structured documentation of what changed between versions, including migration guides, code examples, and changelogs. A simple diff at the endpoint level showing before/after request and response shapes can save hours of developer debugging. Maintain a public changelog with each release.
  - **Why it looks correct:** the team knows what changed because they wrote the code, so documentation seems like overhead for others — but every client integration team has to rediscover these differences through trial and error.

- **Removing fields without notice** — Deprecate fields first by marking them in documentation and adding warning headers to responses. Set a future removal date. Only remove the field after that date has passed and you have verified that no active clients depend on it. Monitor usage analytics to confirm zero usage before removal.
  - **Why it looks correct:** the field seems unused in the team's own applications, and removing dead code feels like good housekeeping — but silent third-party integrations and legacy scripts may still depend on it with no one monitoring their failures.

- **Assuming all clients update immediately** — Mobile apps have weeks or months of update lag, enterprise clients have change-control boards, and some clients may be abandoned. Always assume the oldest active client determines how long you must support each version. Monitor client version distribution and make data-driven sunset decisions.
  - **Why it looks correct:** in development, everyone upgrades immediately when asked, so it seems reasonable to expect the same from users — but real-world clients operate on update cycles you don't control and may never upgrade voluntarily.

---

## Key Design Considerations

- **Maximum 3 active versions:** v1 (deprecated — migration in progress), v2 (current — actively developed), v3 (preview — opt-in beta). Never support more than three versions simultaneously. Each version adds testing, documentation, deployment, and support overhead. When a version drops below 1% of traffic, begin the sunset process immediately rather than maintaining it indefinitely.

- **Minimum 18-month deprecation period** — Give clients 18 months from the deprecation announcement to the sunset date. This accommodates enterprise procurement cycles, mobile app store review times, and development sprints. For security-critical changes, the period may be shortened to 3-6 months with enhanced communication. The deprecation period should be documented in the API terms of service so clients know what to expect.

- **Automated version sunset** — Implement a filter or gateway rule that rejects requests to sunset versions with a `410 Gone` response. The response body should include the replacement version URL and a link to the migration guide. Do not rely on manual enforcement — automate it so that sunset happens exactly on schedule without human error.

- **Version at API level, not individual endpoints** — Version the entire API surface at once, not individual endpoints. Mixing v1 and v2 endpoints in the same deployment creates confusion and breaks the contract guarantee. If only one endpoint needs a breaking change, create a new API version and deploy it as a whole. The exception is internal or experimental endpoints where per-endpoint versioning may be acceptable.

- **Internal vs External APIs** — Internal APIs (between microservices) may use different versioning strategies than external APIs (customer-facing). Internal APIs can use header versioning or semantic versioning in event payloads because the consumer base is controlled. External APIs benefit from URI path versioning because it is simpler and more discoverable for third-party developers. Document both strategies clearly if they diverge.

- **Version Lifecycle:** Development (team testing) → Preview (beta customers) → Active (general availability) → Deprecated (announced sunset) → Sunset (returns `410 Gone`). Each phase has clear entry and exit criteria. Track the lifecycle in a public roadmap so clients can plan migrations years in advance.

- **Consider capability-based versioning as an alternative** — Instead of rigid version numbers, allow clients to declare the capabilities they support via headers or query parameters. This enables more granular evolution without creating multiple full API versions. For example, a client can request `fields=v2,pagination=cursor,auth=oauth2` to opt into specific new features while keeping the rest of the contract stable.

- **Plan for versioning in the data layer** — Version-specific mappers should transform between a canonical database model and versioned DTOs. The database should never be version-aware. This design allows the API to evolve independently of the data schema. When the database schema changes, only the latest mapper needs updating; older mappers simply pass through or transform the canonical data appropriately.

---

## Real-World Scenarios

### Scenario 1: Breaking Change Migration with 100+ Clients

**Context:** Your e-commerce API has a `GET /orders` endpoint returning `{ "order_id": 123, "items": [...] }`. The business requires renaming `order_id` to `id` and moving items to a nested `order_items` structure. Over 100 active clients depend on this endpoint — mobile apps, web apps, and third-party integrations. Any breaking change will immediately break production. The product team needs this change to support a new fulfillment system that expects the restructured payload.

**Problem:** You cannot simply change the existing endpoint because 100+ clients will break the moment you deploy. The mobile app has a 2-week app store review cycle, enterprise clients have change-freeze periods, and some third-party integrations are maintained by teams in different time zones with no urgent response capability. The change is not a security fix, so you cannot justify immediate breaking. The business wants the new fulfillment system live in 6 months.

**Resolution:** Create `/api/v2/orders` with the new structure (`id` and `order_items`). Keep v1 running unchanged with the old `order_id` and `items` format. Add deprecation headers to all v1 responses: `X-API-Deprecated: true`, `X-API-Sunset: 2026-12-31`, `X-API-Migration: /api/v2/orders`. Announce the deprecation immediately via email, blog post, and API changelog — giving clients 18+ months to migrate. Contact all API key owners directly with personalized migration guides that show exact before/after code examples in their programming language. Build an API migration dashboard that lets each client see their v1 vs v2 usage, top endpoints, and migration status. Monitor v1 traffic via analytics and identify the top 10 clients by volume — assign dedicated support engineers to help them migrate. Offer a 3-month parallel run period where v1 and v2 run simultaneously and the gateway can route individual API keys to either version. After the sunset date passes, return `410 Gone` with a JSON body containing the v2 endpoint URL, migration guide link, and contact information for exceptions.

```java
// Gateway routing based on API key version preference
@Component
public class VersionRoutingFilter implements Filter {

    private final ApiKeyRepository apiKeyRepository;

    public VersionRoutingFilter(ApiKeyRepository apiKeyRepository) {
        this.apiKeyRepository = apiKeyRepository;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String apiKey = httpRequest.getHeader("X-API-Key");

        if (apiKey != null) {
            ClientConfig config = apiKeyRepository.findByKey(apiKey);
            if (config != null && config.getPreferredVersion() != null) {
                httpRequest.setAttribute("targetVersion", config.getPreferredVersion());
            }
        }

        chain.doFilter(request, response);
    }
}
```

### Scenario 2: Multiple Database Schemas for Different API Versions

**Context:** Your API evolves from v1 (flat user object with `name`, `email`) to v2 (nested `profile` object with `firstName`, `lastName`, `emailAddress`). The database already migrated to the new schema with a `profiles` table that normalizes user information. V1 clients still expect the flat format. You cannot force all clients to upgrade immediately. Additionally, some internal reporting tools depend on the old flat structure and will take months to refactor.

**Problem:** The database has moved on, but the API consumers have not. You need to serve both formats from the same data store without creating database forks or maintaining duplicate tables. V1 expects `user.name` (string) and `user.email` (string), while v2 expects `user.fullName`, `user.profile.firstName`, `user.profile.lastName`, and `user.profile.emailAddress`. A naive approach might maintain two database schemas, but that doubles storage, creates synchronization issues, and makes the data layer brittle.

**Resolution:** Keep a canonical database schema (the current v2 normalized structure). Create version-specific mappers that transform between the canonical model and each API version's DTO. The canonical `User` entity has a `Profile` object with `firstName`, `lastName`, `emailAddress`. The V1 mapper flattens these into the old format, while the V2 mapper preserves the nested structure.

```java
// Canonical entity (never version-aware)
@Entity
@Table(name = "users")
public class User {
    @Id
    private Long id;

    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "profile_id")
    private Profile profile;

    // Getters and setters...
}

@Entity
@Table(name = "profiles")
public class Profile {
    private String firstName;
    private String lastName;
    private String emailAddress;
    private String role;
    private String department;
    private Instant joinedAt;

    // Getters and setters...
}

// V1 mapper: flattens nested structure
public class UserMapperV1 {
    public static UserResponseV1 toResponse(User user) {
        return new UserResponseV1(
            user.getId(),
            user.getProfile().getFirstName() + " " + user.getProfile().getLastName(),
            user.getProfile().getEmailAddress(),
            user.getProfile().getRole()
        );
    }

    public static User toEntity(UserRequestV1 request) {
        User user = new User();
        Profile profile = new Profile();
        // Parse "name" into first and last
        String[] nameParts = request.name().split(" ", 2);
        profile.setFirstName(nameParts[0]);
        profile.setLastName(nameParts.length > 1 ? nameParts[1] : "");
        profile.setEmailAddress(request.email());
        profile.setRole(request.role());
        user.setProfile(profile);
        return user;
    }
}

// V2 mapper: preserves nested structure
public class UserMapperV2 {
    public static UserResponseV2 toResponse(User user) {
        return new UserResponseV2(
            user.getId(),
            new ProfileResponse(
                user.getProfile().getFirstName(),
                user.getProfile().getLastName(),
                user.getProfile().getEmailAddress(),
                user.getProfile().getRole(),
                user.getProfile().getDepartment(),
                user.getProfile().getJoinedAt()
            )
        );
    }
}
```

The database never needs to know about API versions. New versions can add fields — the V1 mapper silently omits them by not mapping fields it does not understand. The canonical schema can evolve independently, and each version's mapper bridges the gap between the canonical model and the versioned contract. This pattern keeps the data layer clean and allows adding new API fields without database changes.

### Scenario 3: Accidental Breaking Change Deployment

**Context:** A developer deploys a change to `GET /api/v1/users/{id}` that removes the `phone` field from the response. The change was meant for v2 but was incorrectly deployed to v1 due to a misconfigured CI/CD pipeline. Within minutes, support tickets flood in from clients whose parsing logic breaks because the expected field is missing. Your monitoring shows a 40x increase in 5xx errors and client-side exceptions.

**Problem:** The deployment has already happened, and the damage is in progress. Clients are actively breaking, and every minute of delay causes more errors. The developer who made the change is unavailable, and the team needs to decide: roll back, hotfix, or accept the breakage. Rolling back may be complicated if the deployment included database migrations or other irreversible changes. The incident is affecting revenue because an e-commerce checkout flow depends on the `phone` field for shipping notifications.

**Resolution:** (1) Roll back immediately to the previous deployment — this is the fastest way to restore service. If the deployment was part of a blue/green or canary release, simply redirect traffic back to the previous version. (2) If rollback is impossible (e.g., irreversible database migration already ran), add the `phone` field back as deprecated with a null value and deploy a hotfix. Return the field with a `X-API-Deprecated: true` header and a note that it will be removed in v2. (3) Audit the deployment pipeline: add version compatibility checks in CI/CD that compare the new response schema against the previous version's contract. Use OpenAPI diff tools that automatically detect removed fields and block the deployment. (4) Implement automated backward compatibility tests using OpenAPI diff tools — every PR compares the generated OpenAPI spec against the previous version's spec and fails the build if a breaking change is detected without a version bump. (5) Add a manual approval gate for changes to versioned endpoints — any modification to a `v1` controller requires senior developer review. (6) Conduct a postmortem: the root cause was a missing version check in the deployment script; add a pre-deployment hook that validates the deployed version matches the intended version.

```java
// CI/CD backward compatibility gate (pseudo-code)
public class CompatibilityGate {

    public static void main(String[] args) {
        try {
            OpenAPI oldSpec = OpenAPIParser.parse("openapi-v1-latest.yaml");
            OpenAPI newSpec = OpenAPIParser.parse("openapi-v1-new.yaml");

            List<Change> changes = OpenAPIDiff.compare(oldSpec, newSpec)
                .getAllChanges();

            List<Change> breakingChanges = changes.stream()
                .filter(Change::isBreaking)
                .collect(Collectors.toList());

            if (!breakingChanges.isEmpty()) {
                System.err.println("❌ Breaking changes detected in v1:");
                breakingChanges.forEach(c ->
                    System.err.printf("  - %s (%s)%n", c.getDescription(), c.getPath()));
                System.err.println("Blocking deployment. Move these changes to v2.");
                System.exit(1);
            }

            System.out.println("✅ No breaking changes in v1.");
        } catch (Exception e) {
            System.err.println("Error running compatibility check: " + e.getMessage());
            System.exit(1);
        }
    }
}
```

## Use Cases

- API versioning strategies trade off client convenience against server flexibility. The right approach depends on who your clients are, how frequently they update, and whether you control them.

- **Public API with hundreds of external clients** — mobile apps, third-party integrations, and partner systems
  - When to use: You cannot coordinate upgrades with clients. Use URI versioning (`/api/v2/orders`) for clear, discoverable version identities. Communicate deprecation via `Sunset` and `Deprecation` headers. Example: a payment processor API that maintains v1 for 18 months while offering v2 with improved fraud detection.
  - **Avoid when:** You control all clients and can coordinate rollouts — no versioning or header-based versioning reduces URL pollution.

- **Mobile apps with slow update cycles** — clients that may take months to upgrade
  - When to use: Mobile users don't update apps immediately, so old API versions must remain available for extended periods. Support at least two major versions concurrently and use feature detection via capability headers. Example: a ride-sharing app whose iOS/Android v3 still calls `/api/v1/rides` while v4 uses `/api/v2/rides`.
  - **Avoid when:** All users are on the latest version (e.g., a web SPA that refreshes on load) — you can deprecate faster without breaking clients.

- **Internal service contract evolution** — microservice-to-microservice API changes within the same organization
  - When to use: You need to evolve an internal API without breaking consuming services owned by other teams. Query parameter versioning (`?version=2026-06-01`) or custom header versioning (`Accept-Version: 2026-06-01`) suits internal use where clients can update their configuration easily. Example: the `inventory-service` adds a new stock-checking algorithm and publishes a new contract version consumed by `order-service`.
  - **Avoid when:** The API surface is small and changes are additive only — backward-compatible additions don't require a version bump.

- **Breaking changes for security or compliance fixes** — urgent changes that cannot wait for normal deprecation cycles
  - When to use: A security vulnerability requires immediate API changes. Create a parallel v2, apply the fix, and communicate an accelerated sunset with clear justification. Example: a user data API must remove SSN fields from responses due to new privacy regulations — deploy v2 with the fix, redirect sensitive clients, and set a 3-month v1 sunset.
  - **Avoid when:** The change is backward-compatible and could be deployed without versioning — unnecessary versioning fragments the client base.

---

## Scenario-Based Questions

1. **Q: You need to add a breaking change to an endpoint with 100+ clients. The change is critical (security fix) and cannot wait 18 months. How do you balance security with backward compatibility?**
   - A: Create an immediate v2 with the fix and deploy it under `/api/v2/` alongside the existing v1. Backport the security fix to v1 if it can be done as an additive change — for example, adding additional validation or sanitization without changing the response format. If the fix inherently requires breaking changes, communicate urgency to all clients: email every API key owner, add prominent deprecation headers to v1 responses, and set a shorter sunset window of 3-6 months with clear justification. Offer migration assistance: provide before/after code examples in multiple programming languages, run migration workshops, and assign dedicated support engineers to the largest clients. For clients that cannot migrate within the window, offer a temporary proxy service that transforms v2 responses into v1 format for an additional 6 months. For extreme cases where the vulnerability is actively being exploited, block old clients after a short grace period (e.g., 2 weeks) and return `426 Upgrade Required` with a link to the new version. Security-critical changes justify breaking compatibility, but minimize the blast radius through communication, tooling, and transitional support.

   - **Follow-up:** Your transitional proxy that converts v2 responses to v1 format introduces additional latency and a new failure mode — if the proxy crashes, both v1 and v2 clients are affected. How do you design the proxy to fail in a way that minimizes blast radius?

2. **Q: Your team maintains 5 active API versions (v1 through v5). Each new developer adds a new version instead of evolving existing ones. The maintenance burden is crushing. How do you fix this?**
   - A: First, establish a hard limit of 3 active versions — v3 (deprecated), v4 (current), and v5 (preview). Immediately announce that v1 and v2 will be deprecated in 3 months, then enforce that deadline with automated sunset blocking. Migrate all remaining v1 and v2 clients to v4, offering dedicated migration support. Second, enforce a policy: no new versions without leadership approval and a documented justification. Developers must first attempt additive changes to the current version — new optional fields, new endpoints, new query parameters — before considering a version bump. Third, implement capability-based versioning for minor differences: instead of creating v6 for a single new field, allow clients to opt in via headers. Fourth, run a knowledge-sharing session on additive API design so the team understands how to evolve APIs without version proliferation. Finally, track version metrics in dashboards: number of versions, usage per version, and maintenance cost per version, making the problem visible to leadership.

   - **Follow-up:** You set a hard limit of 3 versions, but one enterprise client pays 40% of your revenue and refuses to migrate from v2 — do you break them or break your policy?

3. **Q: Your API uses URI path versioning (`/api/v1/users`). A client accidentally hardcodes `/api/v1/` and refuses to update. How do you migrate them without breaking their integration?**
   - A: Implement a redirect service at the API gateway level: when a request hits `/api/v1/users`, check if the client has been migrated. If yes, return a `301 Moved Permanently` redirect to `/api/users` (an unversioned alias that resolves to the latest version). For clients that do not follow redirects, such as legacy mobile apps with hardcoded URL paths, set up a transparent proxy: `/api/v1/` maps to the latest version internally by parsing the request path and rewriting it to the current version's controller. Over time, the redirect response can include a `Retry-After` header and eventually the endpoint returns `410 Gone` with a JSON body containing the new URL. For stubborn clients, offer a transitional API key that routes all `/api/v1/` traffic to the latest version internally, effectively making v1 an alias for the current version. This buys time while the client's development team schedules their migration.

4. **Q: Your database schema changed (column renamed from `email` to `email_address`). V1 expects `email`, V2 expects `email_address`. How do you design the version mapping layer?**
   - A: The database keeps the new canonical schema with `email_address`. The V1 mapper queries the `email_address` column and maps it to `email` in the response DTO, effectively aliasing the new column name to the old field. For write operations, V1 accepts `email` in the request body; the V1 mapper converts it to `email_address` before saving to the database. The V2 mapper passes through the value as `email_address` without transformation. Use an adapter pattern where a `UserRepository` always works with the canonical entity, and separate `UserMapperV1` and `UserMapperV2` classes handle the transformation on both read and write paths. For the database view layer, consider creating a database view (e.g., `user_v1`) that aliases `email_address` as `email` — this provides a clean interface for the V1 mapper without SQL-level complexity. The database never stores version-specific data, and the mapping layer is the only place where version-aware transformation occurs.

   - **Follow-up:** The V1 write mapper now needs to split `name` into `firstName`/`lastName` and map `email` to `email_address` — but what if V1 sends a `name` that can't be cleanly split (e.g., a single-word mononym)? How do you handle ambiguous or lossy transformations in the version mapper?

5. **Q: You need to support enterprise clients with custom API requirements (different fields, different auth, different rate limits). How do you avoid creating 50 API versions?**
   - A: Use capability-based versioning: clients declare their capabilities in headers such as `X-API-Capabilities: fields=v2,auth=oauth2,rate=enterprise`, and the server selects the appropriate response format, authentication method, and rate limiting policy based on those declared capabilities rather than a version number. Each enterprise client gets a configuration profile stored in a client registry: their feature toggles, field preferences, auth method, and rate limits are all configurable without creating a new API version. Implement GraphQL-style field selection for clients that need specific fields — let them request only what they need using a `?fields=id,name,email` query parameter, which eliminates the need for version-specific response shapes. For authentication and rate limiting, use configuration per API key rather than per version — each API key has an associated plan (e.g., `free`, `pro`, `enterprise`) that determines auth requirements and rate limits. This approach keeps the version count at 2-3 (deprecated, current, preview) while supporting hundreds of unique client configurations through feature flags, capability declarations, and field selection.

6. **Q: A new API version changes response field types (e.g., `id` from `Long` to `String`). This breaks clients with strict typing. How do you communicate and handle the migration?**
   - A: Add a warning header to the v1 response — `X-API-Type-Change: id will change from Long to String in v2` — so developers see the change in their actual API responses during development. Release v2 with the `String` type and provide a migration tool that automatically updates client code: the tool scans the codebase for `int` or `Long` usage on the `id` field and converts it to `string` with appropriate parsing. In the API documentation, clearly mark the type change with before/after examples in multiple languages, including handling for leading zeros, UUID-like identifiers, and large numbers that lose precision in JavaScript. Run both types in parallel during a transition period: v2 returns `String` for `id`, but also includes a deprecated `legacy_id` field with the `Long` value for 6 months. This allows clients to migrate incrementally — first add `legacy_id` handling, then switch to `id` as a string, then remove `legacy_id`. Provide a migration dashboard that shows each client's migration progress and flags any type-related errors.

   - **Follow-up:** You include a `legacy_id` field in v2 responses for backward compatibility — how long do you commit to keeping it, and what happens to clients who still depend on it after the removal date?

7. **Q: How do you test backward compatibility in CI/CD when deploying a new API version?**
   - A: Record the OpenAPI spec of the current production version as a baseline artifact. When building the new version, generate the new OpenAPI spec via integration tests that exercise all endpoints. Use automated OpenAPI diff tools (e.g., `openapi-diff`, `oas-diff`) to compare the baseline and new specs — the tool should flag any breaking changes such as removed fields, type changes, new required fields, or renamed endpoints. Run consumer-driven contract tests using a framework like Pact: each client defines their expected contract in a Pact file, and the CI pipeline verifies that the new API version satisfies all existing consumer contracts.

   - **Follow-up:** Your Pact contract tests pass because the response schema matches, but a client's integration fails in production because a field that was previously always present is now `null` for some resources — how do you test for semantic compatibility beyond structural schema matching? Replay recorded production requests from the current version against the new version and verify that they all return successful responses (status 2xx) with the expected structure. Implement a canary deployment that routes 1% of production traffic to the new version and monitors for error rate increases — if any consumer receives an unexpected response, the canary automatically rolls back. Combine all these checks into a single CI gate that must pass before any versioned API change is deployed.

8. **Q: Your mobile app cannot update frequently (some users on versions 2+ years old). You need to make breaking API changes. How do you support ancient mobile clients?**
   - A: Support multiple API versions for as long as the oldest mobile version remains above your usage threshold (typically 1% of active users). Plan for API versions to live 2-3 years, which means careful additive-only development for long periods — use optional fields, new endpoints, and feature detection rather than breaking changes. Implement client version detection via a `X-Client-Version` header that the mobile app sends with every request; the server adapts responses based on the client's capabilities by maintaining a capability matrix (e.g., "version < 3.0 cannot handle nested objects"). When a mobile version drops below 1% of active users, send in-app update prompts that require the user to update before continuing. For the remaining users on very old versions, offer a transitional proxy service that translates the latest API responses backward into the format the old client expects — this is technically debt but avoids breaking paying customers. Set a hard cutoff date for each major mobile version and communicate it through multiple channels (email, push notification, in-app banner) at least 6 months in advance.

   - **Follow-up:** Your transitional proxy service now has to maintain backward compatibility logic for 3 different mobile API versions — who owns this proxy, and how do you prevent it from becoming its own unmaintainable monolith?

9. **Q: How do you handle versioning for internal event-driven communication (Kafka/RabbitMQ) between microservices?**
   - A: Use a schema registry (Confluent Schema Registry, Apicurio, or AWS Glue Schema Registry) that stores and validates event schemas centrally, ensuring that producers and consumers agree on the data format. Define compatibility modes carefully: backward compatibility means new consumers can read events produced by old producers (you can add optional fields); forward compatibility means old consumers can read events produced by new producers (you can add optional fields but cannot remove or rename existing ones); full compatibility means both directions work. Set backward compatibility as the default for most events — new versions can add optional fields but cannot remove or change existing required fields. Each event envelope includes a `schema_version` or `event_version` integer field as the first field, so consumers can read the version before attempting to deserialize the rest of the payload. Services implement a version-aware deserialization strategy: read the version field first, then select the appropriate deserializer or DTO. Use the "tolerant reader" pattern — consumers ignore fields they do not understand and provide sensible defaults for missing optional fields. For major breaking changes that require new required fields, create a new event type (e.g., `order-created-v2` instead of modifying `order-created`) and run both event types in parallel during migration.

10. **Q: Your public API has 1M+ daily active clients. You need to deprecate v1 of an endpoint. How do you execute the deprecation plan?**
    - A: Announce deprecation 18+ months in advance via email to all registered API key owners, a blog post on the developer portal, and a changelog entry in the API documentation. Add `X-API-Deprecated: true` and `X-API-Sunset: <date>` headers to all v1 responses from the moment of announcement — every single response carries the deprecation signal. Send quarterly reminder emails with personalized migration statistics showing each client's v1 vs v2 usage and progress. Build a migration dashboard on the developer portal where clients can see their own v1 usage trends, migration status, and known compatibility issues. Six months before sunset, add a warning message in the JSON response body: `{"warning": "This API version will be sunset on 2026-12-31. Migrate to v2: https://docs.example.com/migration"}` — this ensures even clients that ignore headers see the notice. Three months before sunset, start returning `410 Gone` for 1% of v1 requests (gradual rollout) to alert monitoring systems. On the sunset date, return `410 Gone` for all v1 requests with a JSON body containing the v2 endpoint URL, migration guide link, and a contact email for extension requests. Maintain a small whitelist for clients with approved extensions (e.g., government contracts with longer procurement cycles) but charge a premium to incentivize migration.

---

## Interview Questions

1. **What is API versioning and why is it needed?**
   - A: API versioning is the practice of managing changes to an API over time while maintaining backward compatibility for existing clients. It allows introducing breaking changes, deprecating old functionality, and letting clients migrate at their own pace. Without versioning, every change risks breaking existing clients, which is unacceptable in production environments with diverse consumers including mobile apps, third-party integrations, and enterprise systems.

2. **What are the common API versioning strategies?**
   - A: The four main strategies are URI path versioning (`/api/v1/users`), header versioning (`Accept: application/vnd.myapp.v1+json`), query parameter versioning (`?version=1`), and content negotiation versioning. URI path versioning is the most common because it is simple, cacheable, and discoverable. Header versioning keeps URLs clean but is harder to test and debug. Each strategy has trade-offs, and some APIs combine multiple strategies in a hybrid approach.

3. **What counts as a breaking change?**
   - A: Breaking changes include removing fields from responses, changing field types, renaming endpoints or fields, modifying request/response structures, removing endpoints, changing error formats, and adding new required fields to requests. Adding new fields, adding new endpoints, and adding optional parameters are backward-compatible. Any change that would cause an existing client to fail if they processed the response without code changes is breaking.

4. **How many API versions should you support simultaneously?**
   - A: Maximum 3: one deprecated version (migration in progress), one current version (actively developed), and one preview version (opt-in beta). Supporting more than 3 versions creates an unsustainable maintenance burden with multiplied controllers, DTOs, mappers, tests, and documentation. Old versions should be sunset methodically as their traffic drops below 1% of total API usage.

5. **How do you deprecate an API version?**
   - A: Announce the deprecation 18+ months in advance through email, blog posts, and changelogs. Add `X-API-Deprecated: true` and `X-API-Sunset: <date>` headers to all responses from the moment of announcement. Send periodic migration reminders with client-specific statistics. After the sunset date, return `410 Gone` for all requests to the deprecated version with a JSON body containing the replacement URL and migration guide link.

6. **What is the difference between URI path versioning and header versioning?**
   - A: URI path versioning (`/api/v1/users`) is simple, cacheable, and discoverable — the version is visible in the URL, making it easy to debug and test. Header versioning (`Accept: application/vnd.myapp.v1+json`) keeps URLs clean from a REST perspective but hides the version in an HTTP header, making it harder to test, debug, and cache. URI path is recommended for most public APIs due to its operational simplicity.

7. **How do you handle database schema changes across API versions?**
   - A: Use a canonical database schema that represents the latest version of the data model. Create version-specific mappers (V1Mapper, V2Mapper) that transform between canonical entities and versioned DTOs. Old versions map canonical data to their expected format, silently ignoring fields they do not understand. The database never stores version-specific data, keeping the data layer clean and allowing schema evolution independent of API versions.

8. **What is capability-based versioning?**
   - A: Capability-based versioning replaces rigid version numbers with client-declared capabilities, typically via headers like `X-API-Capabilities: fields=v2,pagination=cursor`. The server selects the appropriate response format based on the declared capabilities rather than a version number. This supports diverse client needs without version proliferation, enabling granular evolution where different aspects of the API can evolve independently.

9. **How do you test backward compatibility?**
   - A: Use contract testing (Pact) where each client defines their expected contract and CI verifies no client is broken by new changes. Use OpenAPI diff tools that compare the generated spec against the previous version's spec and flag breaking changes. Replay recorded production requests against new versions and verify backward-compatible responses. Combine these with canary deployments that monitor real traffic for compatibility issues before full rollout.

10. **How do you version events in event-driven architectures?**
    - A: Use a schema registry (Confluent Schema Registry, Apicurio) that stores event schemas and enforces compatibility modes. Include a schema version ID or `event_version` field as the first field in every event envelope so consumers can identify the version before deserialization. Use backward-compatible changes whenever possible — add optional fields rather than removing or renaming existing ones. Services implement tolerant readers that handle multiple event versions by ignoring unknown fields and providing defaults for missing optional fields.

---

## Developer Recommendations

- **Version from day one** — Adding versioning after launch is exponentially harder because you must retroactively decide what the unversioned API means, how to handle existing clients that never opted into versioning, and how to communicate the change. The cost of adding `v1` to your API path from the first commit is effectively zero — one extra URI segment in your routing configuration. The benefit is the ability to make breaking changes later without ceremony, retroactive communication, or client breakage. Even if you never need a v2, the version prefix is invisible insurance that costs nothing. When you inevitably do need to make a breaking change, having `/api/v1/` already in place means you simply add `/api/v2/` and begin the migration process with zero disruption to existing clients.
  - **Production story:** A mobile payment startup launched without versioning — 18 months and 5M users later, a required security field rename forced them to either break every installed app or maintain a complex header-based version detection layer across all servers, taking 3 months of engineering time to implement retroactively.

- **Use additive changes as long as possible** — Before creating v2, ask whether the change can be made backward-compatible. Add the new field alongside the old one in the response and mark the old field as deprecated using headers and documentation. Accept both input formats during a transition period so clients can switch at their own pace. Use default values for new optional parameters so existing requests continue to work. Only create a new API version when additive changes are truly impossible — for example, when you need to change a field type, remove a field entirely, or fundamentally alter the request/response contract. This "additive-first" approach delays the versioning tax: each new version adds maintenance overhead, documentation burden, and client migration costs, so you want to avoid creating one until absolutely necessary. Many APIs can evolve for years with only additive changes if designed carefully from the start.

- **Keep maximum 3 active versions** — Each active version approximately doubles the maintenance burden because you need separate controllers, DTOs, request/response mappers, test suites, documentation sets, and monitoring dashboards. Supporting 5 versions means 5x the code for request handling, 5x the test maintenance, and 5x the documentation updates every time a change is made. Enforce a hard limit of 3 versions: one deprecated (migration in progress), one current (actively developed), and one preview (opt-in beta). Set hard sunset dates when each version is created, not when it becomes burdensome. Automate version sunset with middleware that returns `410 Gone` after the deadline — do not rely on manual enforcement. Be ruthless about sunsetting: if a version has less than 1% traffic and a replacement exists, shut it down. The long tail of low-traffic versions consumes disproportionate maintenance effort.

- **Use a version mapping layer, not separate databases** — Never fork your database per API version. This anti-pattern creates synchronization issues, data inconsistency, doubled storage costs, and complex ETL pipelines to keep forks in sync. Instead, maintain a single canonical database schema that represents your current understanding of the data model. Create version-specific mappers (UserMapperV1, UserMapperV2, UserMapperV3) that transform between canonical database entities and versioned API DTOs. The mapper pattern is a pure transformation layer with no side effects — easy to test, easy to maintain, and easy to extend. When the database schema evolves, only the latest mapper needs updating; older mappers still reference canonical fields and expose them in the expected legacy format. This pattern keeps the data layer completely version-unaware and allows the API to evolve independently from the database.
  - **Production story:** A SaaS company maintained separate PostgreSQL databases per API version for 2 years — when a critical security patch required changing a column type, they had to migrate 5 databases instead of one, and the v3 database that nobody remembered still had the vulnerable column type 8 months after the patch was applied.

- **Add deprecation headers from the moment you announce** — As soon as you announce that a version is deprecated, add `X-API-Deprecated: true` and `X-API-Sunset: <ISO date>` headers to every response from that version. These headers allow clients to build automated monitoring and alerting around deprecation — sophisticated clients can detect them and trigger their own migration workflows programmatically. Additionally, add an `X-API-Migration` header that points to the replacement endpoint URL, giving clients a direct link to the new version's documentation. Don't rely solely on blog posts, emails, or documentation updates — embed the deprecation signal in every single API response so it is impossible to miss. After the sunset date passes, return `410 Gone` with a JSON response body containing the migration URL, contact information, and a link to the migration guide. Consider adding a `Retry-After` header with the sunset date so automated systems can schedule their migration.

- **Automate backward compatibility checks in CI/CD** — Use OpenAPI diff tools (such as `openapi-diff` or `oas-diff`) that compare the generated OpenAPI spec of the new build against the spec of the previous production version. The CI pipeline should fail the build if a breaking change is detected in an existing version (e.g., changes to v1 or v2) without a corresponding version bump. This prevents the most common versioning mistake: accidental breaking changes deployed to the wrong version. Pair this with consumer-driven contract tests using a tool like Pact — each client registers their expected contract, and CI verifies that no client's contract is broken by the new deployment. Together, these automated gates enforce versioning discipline across the entire team without relying on manual code reviews or tribal knowledge. They also serve as documentation: the CI failure message tells developers exactly what changed and why it is breaking, educating the team about API design principles over time.

- **Design your API for evolution from the start** — Use extensible data formats such as maps, flexible enumerations with an "unknown" fallback value, and wrapper objects that can accommodate new fields without structural changes. Avoid exposing internal database identifiers directly — use opaque string identifiers (UUIDs) so you can change the underlying storage without breaking the API contract. Use hypermedia links (HATEOAS) to decouple clients from hardcoded URL patterns — clients follow links rather than constructing URLs, so endpoint renames are transparent. Design response objects with optional fields as the default — make everything optional unless there is a compelling reason to make it required. This evolutionary design philosophy means you can add new fields, endpoints, and capabilities without creating new versions for years, drastically reducing the versioning tax on your team and your consumers.
