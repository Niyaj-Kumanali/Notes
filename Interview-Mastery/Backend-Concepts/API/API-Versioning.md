# API Versioning

---

## Overview

- **Definition:** API versioning is the practice of managing changes to an API over time while maintaining backward compatibility for existing clients. It provides a structured way to introduce breaking changes, deprecate old functionality, and allow clients to migrate at their own pace.

- **Why It Exists:** APIs evolve — endpoints change, request/response formats shift, and fields are added or removed. Without versioning, every change risks breaking existing clients, making it impossible to iterate safely.

- **Key Goals:**
  - **Backward Compatibility** — existing clients continue working after changes
  - **Gradual Migration** — clients upgrade on their own schedule
  - **Contract Stability** — clients depend on a stable API contract
  - **Clear Deprecation Path** — old versions are sunsetted methodically

---

## Change Types

- **Backward-Compatible Changes:**
  - Adding new fields to responses
  - Adding new endpoints
  - Adding optional parameters
  - Extending enumerations

- **Breaking Changes:**
  - Removing fields from responses
  - Changing field types
  - Renaming endpoints or fields
  - Changing request/response structure
  - Removing endpoints
  - Changing error formats

- **Semantic Versioning for APIs:**
  - `Major.Minor.Patch` (e.g., 2.1.0)
  - **Major** — breaking changes
  - **Minor** — backward-compatible additions
  - **Patch** — backward-compatible bug fixes

---

## Versioning Strategies

- **URI Path Versioning:**
  - Example: `/api/v1/users`, `/api/v2/users`
  - **Pros:** simple, discoverable, CDN-cacheable
  - **Cons:** URI pollution, less RESTful

- **Header Versioning:**
  - Example: `Accept: application/vnd.myapp.v1+json`
  - **Pros:** clean URLs, standards-based
  - **Cons:** harder to test, hidden complexity

- **Query Parameter Versioning:**
  - Example: `/api/users?version=1`
  - **Pros:** single base URI
  - **Cons:** caching issues, less discoverable

- **Content Negotiation Versioning:**
  - Example: `Content-Type: application/vnd.myapp.v1+json`
  - **Pros:** media type negotiation
  - **Cons:** complex client setup

---

## Production Code Examples

### URI Versioning with Multiple Controllers

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    private final UserService userService;
    public UserControllerV1(UserService userService) { this.userService = userService; }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponseV1> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(UserMapperV1.toResponse(userService.findById(id)));
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    private final UserService userService;
    public UserControllerV2(UserService userService) { this.userService = userService; }

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
public record UserResponseV1(Long id, String name, String email, String role) {}

// V2 - nested structure with breaking changes
public record UserResponseV2(
    String userId,          // Changed from Long to String
    String fullName,        // Renamed from "name"
    String emailAddress,    // Renamed from "email"
    ProfileResponse profile // Nested structure
) {}

public record ProfileResponse(String role, String department, Instant joinedAt, String status) {}
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
            .map(UserMapperV1::toResponse).collect(Collectors.toList()));
    }

    @GetMapping(produces = "application/vnd.myapp.v2+json")
    public ResponseEntity<PagedResponse<UserResponseV2>> getAllUsersV2(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(PagedResponse.from(
            userService.findAll(PageRequest.of(page, size)).map(UserMapperV2::toResponse)));
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
        return ResponseEntity.ok()
            .header("X-API-Deprecated", "true")
            .header("X-API-Sunset", "2026-12-31")
            .header("X-API-Migration", "/api/v2/users/" + id)
            .body(UserMapperV1.toResponse(userService.findById(id)));
    }
}
```

---

## Common Mistakes

- **Not versioning from day one** — much harder to add later
- **Breaking changes without version bump** — always bump major for breaking changes
- **Supporting too many versions** — limit active versions to 2-3
- **Inconsistent versioning strategy** — pick one strategy and stick with it
- **Forgetting to sunset old versions** — have a clear deprecation and sunset policy
- **Not documenting version differences** — maintain clear migration guides
- **Removing fields without notice** — deprecate first, remove later

---

## Key Design Considerations

- **Maximum 3 active versions:** v1 (deprecated), v2 (current), v3 (preview)
- **Minimum 18-month deprecation period** — give clients time to migrate
- **Automated version sunset** — reject requests to sunset versions
- **Version at API level**, not individual endpoints
- **Internal vs External APIs** — may use different versioning strategies
- **Version Lifecycle:** Development → Preview → Active → Deprecated → Sunset

---

## Real-World Scenarios

### Scenario 1: Breaking Change Migration with 100+ Clients
**Context:** Your e-commerce API has a `GET /orders` endpoint returning `{ "order_id": 123, "items": [...] }`. The business requires renaming `order_id` to `id` and moving items to a nested `order_items` structure. Over 100 active clients depend on this endpoint — mobile apps, web apps, and third-party integrations. Any breaking change will immediately break production.

**Resolution:** Create `/api/v2/orders` with the new structure. Keep v1 running unchanged. Add deprecation headers to v1 responses: `X-API-Deprecated: true`, `X-API-Sunset: Fri, 31 Dec 2026 23:59:59 GMT`, `X-API-Migration: /api/v2/orders`. Announce the change 18 months in advance. Contact all API key owners with migration guides. Monitor v1 usage via analytics. After the sunset date, return `410 Gone` with a link to v2 migration docs. Meanwhile, v2 also handles new features so clients have incentive to migrate.

### Scenario 2: Multiple Database Schemas for Different API Versions
**Context:** Your API evolves from v1 (flat user object with `name`, `email`) to v2 (nested `profile` object with `firstName`, `lastName`, `emailAddress`). The database already migrated to the new schema. V1 clients still expect the flat format. You cannot force all clients to upgrade immediately.

**Resolution:** Keep a canonical database schema (the current v2 structure). Create version-specific mappers that transform between the canonical model and each API version's DTO.

```java
public class UserMapperV1 {
    public static UserResponseV1 toResponse(User user) {
        return new UserResponseV1(
            user.getId(),
            user.getProfile().getFirstName() + " " + user.getProfile().getLastName(), // flatten
            user.getProfile().getEmailAddress()
        );
    }
}

public class UserMapperV2 {
    public static UserResponseV2 toResponse(User user) {
        return new UserResponseV2(
            user.getId(),
            new ProfileResponse(
                user.getProfile().getFirstName(),
                user.getProfile().getLastName(),
                user.getProfile().getEmailAddress()
            )
        );
    }
}
```

The database never needs to know about API versions. New versions can add fields — V1 mapper silently omits them.

### Scenario 3: Accidental Breaking Change Deployment
**Context:** A developer deploys a change to `GET /api/v1/users/{id}` that removes the `phone` field from the response. The change was meant for v2 but was incorrectly deployed to v1. Within minutes, support tickets flood in from clients whose parsing logic breaks because the expected field is missing.

**Resolution:** (1) Roll back immediately to the previous deployment. (2) If rollback is impossible (e.g., database migration already ran), add the `phone` field back as deprecated with a null value and deploy a hotfix. (3) Audit the deployment pipeline: add version compatibility checks in CI/CD that compare the new response schema against the previous version's contract. (4) Implement automated backward compatibility tests using OpenAPI diff tools — every PR compares the generated OpenAPI spec against the previous version.

---

## Scenario-Based Questions

1. **Q: You need to add a breaking change to an endpoint with 100+ clients. The change is critical (security fix) and cannot wait 18 months. How do you balance security with backward compatibility?**
   - A: (1) Create an immediate v2 with the fix. (2) Backport the security fix to v1 if possible (additive change). (3) Communicate urgency: email all API key owners, add prominent deprecation headers, set a shorter sunset window (3-6 months). (4) Offer migration assistance: provide code examples, migration scripts, and direct support for large clients. (5) For extreme cases, block old clients after a grace period and return `426 Upgrade Required` with a link to the new version. Security-critical changes justify breaking compatibility — but minimize the blast radius.

2. **Q: Your team maintains 5 active API versions (v1 through v5). Each new developer adds a new version instead of evolving existing ones. The maintenance burden is crushing. How do you fix this?**
   - A: (1) Set a hard limit of 3 active versions (e.g., v3 deprecated, v4 current, v5 preview). (2) Immediately sunset the oldest versions: announce that v1 and v2 will be deprecated in 3 months. Migrate remaining clients. (3) Enforce a policy: no new versions without leadership approval. First attempt additive changes to the current version. (4) Implement automated sunset: reject requests to versions past their sunset date. (5) Use capability-based versioning for minor differences instead of creating new versions.

3. **Q: Your API uses URI path versioning (`/api/v1/users`). A client accidentally hardcodes `/api/v1/` and refuses to update. How do you migrate them without breaking their integration?**
   - A: (1) Run a redirect service: `/api/v1/users` returns a `301 Moved Permanently` to `/api/users` (unversioned — latest version). (2) The redirect response includes the new URI. (3) Over time, the client's library may follow redirects (HTTP clients often do). (4) For clients that don't follow redirects, offer a dedicated proxy endpoint that maps their requests to the latest version. (5) Eventually, the redirect responds with `410 Gone` after a reasonable grace period.

4. **Q: Your database schema changed (column renamed from `email` to `email_address`). V1 expects `email`, V2 expects `email_address`. How do you design the version mapping layer?**
   - A: (1) Database keeps the new canonical schema (`email_address`). (2) V1 mapper queries `email_address` and maps it to `email` in the response DTO. (3) V2 mapper passes it through as `email_address`. (4) For writes: V1 accepts `email`, V1 mapper converts it to `email_address` before saving. (5) Use an adapter pattern: `UserRepository` always works with canonical entities; `UserMapperV1` and `UserMapperV2` handle the transformation. The database never stores version-specific data.

5. **Q: You need to support enterprise clients with custom API requirements (different fields, different auth, different rate limits). How do you avoid creating 50 API versions?**
   - A: (1) Use capability-based versioning: clients declare capabilities in headers (`X-API-Capabilities: fields=v2,auth=oauth2,rate=enterprise`). The server selects response format based on capabilities, not version numbers. (2) Feature flags per client: each enterprise client gets a configuration profile. (3) GraphQL-style field selection: allow clients to request specific fields. (4) For auth and rate limits, use configuration per API key rather than per version. This keeps the version count at 2-3 while supporting diverse needs.

6. **Q: A new API version changes response field types (e.g., `id` from `Long` to `String`). This breaks clients with strict typing. How do you communicate and handle the migration?**
   - A: (1) Add a `X-API-Type-Change` header to v1 responses warning clients. (2) Release v2 with the `String` type. (3) Provide a migration tool: `MigrateClient -source=v1 -target=v2` that automatically updates client code. (4) In the API documentation, clearly mark the type change and provide code examples in multiple languages showing the migration. (5) Run both types in parallel: v2 returns `String`, but also include the `Long` version as a deprecated field (`legacy_id`) for 6 months.

7. **Q: How do you test backward compatibility in CI/CD when deploying a new API version?**
   - A: (1) Record the OpenAPI spec of the current version. (2) When deploying the new version, generate the new OpenAPI spec. (3) Use OpenAPI diff tools (e.g., `openapi-diff`) to compare: flag any breaking changes (removed fields, type changes, new required fields). (4) Run contract tests: replay recorded requests from the current version against the new version and verify backward-compatible responses. (5) Use consumer-driven contract tests (Pact) — each client defines their expected contract; CI verifies no client is broken.

8. **Q: Your mobile app cannot update frequently (some users on versions 2+ years old). You need to make breaking API changes. How do you support ancient mobile clients?**
   - A: (1) Support multiple API versions for as long as the oldest mobile version is active. (2) When a mobile version falls below 1% of active users, send in-app update prompts. (3) Use feature detection: the mobile app sends its version in a header (`X-Client-Version: 3.2.1`). The server adapts responses based on client capabilities. (4) Plan for API versions to live 2-3 years. This means careful additive-only development for long periods. (5) When sunsetting a version, block the oldest app versions and show a "Please update" screen.

9. **Q: How do you handle versioning for internal event-driven communication (Kafka/RabbitMQ) between microservices?**
   - A: (1) Use a schema registry (Confluent Schema Registry, Apicurio) to manage event schemas. (2) Define compatibility modes: backward (new consumers read old events), forward (old consumers read new events), or full (both directions). (3) Set backward compatibility as default — new event versions can add optional fields but cannot remove or change existing required fields. (4) Each event includes a schema version ID or `event_version` field. (5) Services handle multiple event versions in their deserialization logic — always read the version field first.

10. **Q: Your public API has 1M+ daily active clients. You need to deprecate v1 of an endpoint. How do you execute the deprecation plan?**
    - A: (1) Announce deprecation 18+ months in advance via email, blog post, and API changelog. (2) Add `X-API-Deprecated: true` and `X-API-Sunset: <date>` headers to all v1 responses from day one. (3) Send quarterly reminders to API key owners with migration stats. (4) Provide a migration dashboard showing v1 vs v2 usage per client. (5) 6 months before sunset, add a warning message in the response body. (6) On sunset date, return `410 Gone` with a JSON body containing the v2 endpoint URL and migration guide.

---

## Interview Questions

1. **What is API versioning and why is it needed?**
   - A: API versioning manages changes to an API over time while maintaining backward compatibility. It allows introducing breaking changes, deprecating old functionality, and letting clients migrate at their own pace. Without versioning, every change risks breaking existing clients.

2. **What are the common API versioning strategies?**
   - A: URI path versioning (`/api/v1/users`), header versioning (`Accept: application/vnd.myapp.v1+json`), query parameter versioning (`?version=1`), and content negotiation. URI path is the most common and CDN-friendly.

3. **What counts as a breaking change?**
   - A: Removing fields, changing field types, renaming endpoints or fields, changing request/response structures, removing endpoints, changing error formats, adding new required fields. Adding fields, adding endpoints, and adding optional parameters are backward-compatible.

4. **How many API versions should you support simultaneously?**
   - A: Maximum 3: one deprecated (migration in progress), one current, one preview. Supporting more creates unsustainable maintenance burden. Sunset old versions methodically.

5. **How do you deprecate an API version?**
   - A: Announce 18+ months in advance. Add `X-API-Deprecated` and `X-API-Sunset` headers. Send periodic notifications. Provide migration guides. After sunset, return `410 Gone`.

6. **What is the difference between URI path versioning and header versioning?**
   - A: URI path (`/api/v1/users`) is simple, cacheable, and discoverable. Header versioning (`Accept: application/vnd.myapp.v1+json`) keeps URLs clean but is harder to test and debug. URI path is recommended for most APIs.

7. **How do you handle database schema changes across API versions?**
   - A: Use a canonical database schema (the latest version). Create version-specific mappers that transform between canonical entities and versioned DTOs. Old versions simply ignore fields they don't understand.

8. **What is capability-based versioning?**
   - A: Instead of version numbers, clients declare capabilities via headers. The server selects the appropriate response format. This supports diverse client needs without version proliferation.

9. **How do you test backward compatibility?**
   - A: Use contract testing (Pact), OpenAPI diff tools in CI/CD, and consumer-driven contracts. Record current version responses and verify new versions don't break them.

10. **How do you version events in event-driven architectures?**
    - A: Use a schema registry (Confluent, Apicurio) with compatibility modes. Include schema version in the event envelope. Use backward-compatible changes (add optional fields only). Services handle multiple event versions.

---

## Developer Recommendations

- **Version from day one** — Adding versioning after launch is painful: you must retroactively decide what "no version" means, migrate clients, or break them. Prefix your API with `/api/v1/` from the first commit. The cost is one extra URI segment. The benefit is the ability to make breaking changes later without ceremony. Even if you never need it, the insurance is worth the zero cost.

- **Use additive changes as long as possible** — Before creating v2, ask: can the change be made backward-compatible? Add the new field alongside the old one. Mark old fields as deprecated. Support both input formats. Only create a new version when additive changes are impossible (type changes, removed fields, fundamental contract changes). This delays the versioning tax and keeps your API surface manageable.

- **Keep maximum 3 active versions** — Each version doubles your maintenance burden: request mapping, DTOs, mappers, tests, documentation. Support at most 3 versions (deprecated, current, preview). Set hard sunset dates. Automate version sunset — reject requests after the deadline. Don't negotiate exceptions; document the timeline clearly.

- **Use a version mapping layer, not separate databases** — Never fork your database per API version. Keep a canonical schema in the database. Create version-specific mappers (V1Mapper, V2Mapper) that transform between canonical entities and versioned DTOs. This keeps your data layer clean and allows adding new API fields without database changes.

- **Add deprecation headers from the moment you announce** — `X-API-Deprecated: true` and `X-API-Sunset: <date>` tell clients programmatically that a version is going away. Don't just blog about it — embed the information in every response. Clients can build monitoring around these headers and proactively migrate. Add `X-API-Migration` header pointing to the replacement. After sunset, return `410 Gone` with a JSON body containing the migration URL.

- **Automate backward compatibility checks in CI/CD** — Use OpenAPI diff tools that compare the generated spec against the previous version's spec. Fail the build if a breaking change is detected without a version bump. This prevents accidental breaking changes (the most common versioning mistake) and enforces discipline across the team.
