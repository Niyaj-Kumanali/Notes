# API Security

---

## Overview

- **Definition:** API security encompasses the strategies, protocols, and practices designed to protect APIs from unauthorized access, data breaches, injection attacks, and abuse. With APIs becoming the primary interface for modern applications, securing them is critical.

- **Why It Exists:** APIs expose application logic and data to external clients, making them a prime attack vector. Without proper security, APIs can be exploited for data theft, service abuse, denial of service, and unauthorized access.

- **Authentication vs Authorization:**
  - **Authentication (AuthN)** — verifying identity (Who are you?)
  - **Authorization (AuthZ)** — determining permissions (What can you do?)

- **CIA Triad for APIs:**
  - **Confidentiality** — data accessible only to authorized parties (encryption, access control)
  - **Integrity** — data not tampered during transit or storage (signatures, HTTPS)
  - **Availability** — API accessible when needed (rate limiting, DDoS protection)

---

## OWASP API Security Top 10 (2023)

1. **Broken Object Level Authorization (BOLA)** — accessing objects without ownership verification
2. **Broken Authentication** — weak or bypassed authentication mechanisms
3. **Broken Object Property Level Authorization** — accessing/modifying unauthorized fields
4. **Unrestricted Resource Consumption** — lack of rate limiting leading to resource exhaustion
5. **Broken Function Level Authorization** — accessing admin functions as regular user
6. **Unrestricted Access to Sensitive Business Flows** — abusing legitimate business flows
7. **Server Side Request Forgery (SSRF)** — tricking the server into making internal requests
8. **Security Misconfiguration** — default credentials, unpatched systems, exposed debug endpoints
9. **Improper Inventory Management** — undocumented endpoints, old API versions still active
10. **Unsafe Consumption of APIs** — trusting third-party API responses without validation

---

## Production Code Examples

### Spring Security Filter Chain

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/actuator/health").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/v1/users/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .addFilterBefore(new RateLimitFilter(), BasicAuthenticationFilter.class)
            .build();
    }
}
```

### JWT Authentication Filter

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtTokenProvider tokenProvider;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtTokenProvider tokenProvider, UserDetailsService userDetailsService) {
        this.tokenProvider = tokenProvider;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {
        String token = extractToken(request);
        if (token != null && tokenProvider.validateToken(token)) {
            String username = tokenProvider.getUsernameFromToken(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
            authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader(HttpHeaders.AUTHORIZATION);
        return (bearerToken != null && bearerToken.startsWith("Bearer ")) ? bearerToken.substring(7) : null;
    }
}
```

### Method-Level Security

```java
@EnableMethodSecurity
@Configuration
public class MethodSecurityConfig {}

@RestController
@RequestMapping("/api/v1/admin")
public class AdminController {
    @GetMapping("/users")
    @PreAuthorize("hasRole('ADMIN')")
    public List<UserResponse> getAllUsers() { /* ... */ }

    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasAuthority('USER_DELETE')")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) { /* ... */ }
}
```

### Object-Level Authorization (BOLA Prevention)

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService orderService;
    private final AuthorizationService authService;

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable Long orderId,
            @AuthenticationPrincipal User user) {
        Order order = orderService.findById(orderId);
        if (!authService.canAccessOrder(user, order)) {
            return ResponseEntity.status(403).build();
        }
        return ResponseEntity.ok(OrderMapper.toResponse(order));
    }
}

@Component
public class AuthorizationService {
    public boolean canAccessOrder(User user, Order order) {
        if (user.hasRole("ADMIN")) return true;
        return order.getUserId().equals(user.getId());
    }
}
```

### API Key Authentication

```java
@Component
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {
    private static final String API_KEY_HEADER = "X-API-Key";
    private final ApiKeyService apiKeyService;

    public ApiKeyAuthenticationFilter(ApiKeyService apiKeyService) { this.apiKeyService = apiKeyService; }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {
        String apiKey = request.getHeader(API_KEY_HEADER);
        if (apiKey != null && apiKeyService.isValidApiKey(apiKey)) {
            ApiKeyDetails keyDetails = apiKeyService.getKeyDetails(apiKey);
            SecurityContextHolder.getContext().setAuthentication(new ApiKeyAuthenticationToken(keyDetails));
        }
        filterChain.doFilter(request, response);
    }
}
```

### SQL Injection Prevention

```java
// UNSAFE - Never do this
@Query("SELECT u FROM User u WHERE u.name = '" + name + "'")
List<User> findByNameUnsafe(String name);

// SAFE - Use parameterized queries
@Query("SELECT u FROM User u WHERE u.name = :name")
List<User> findByName(@Param("name") String name);

// SAFE - Spring Data JPA derived queries
List<User> findByNameAndEmail(String name, String email);

// SAFE - Native queries with parameters
@Query(value = "SELECT * FROM users WHERE name = ?1", nativeQuery = true)
List<User> findByNameNative(String name);
```

### Security Headers

```java
@Component
public class SecurityHeadersFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        httpResponse.setHeader("X-Content-Type-Options", "nosniff");
        httpResponse.setHeader("X-Frame-Options", "DENY");
        httpResponse.setHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
        httpResponse.setHeader("Content-Security-Policy", "default-src 'self'");
        httpResponse.setHeader("Referrer-Policy", "strict-origin-when-cross-origin");
        chain.doFilter(request, response);
    }
}
```

### Rate Limiting with Redis

```java
@Component
public class RedisRateLimiter {
    private final RedissonClient redisson;

    public RedisRateLimiter(RedissonClient redisson) { this.redisson = redisson; }

    public boolean tryAcquire(String clientId, String endpoint) {
        RRateLimiter rateLimiter = redisson.getRateLimiter("rate:" + clientId + ":" + endpoint);
        rateLimiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.MINUTES);
        return rateLimiter.tryAcquire();
    }
}
```

---

## Common Mistakes

- **Exposing internal IDs** — use UUIDs instead of auto-increment IDs in URLs
- **Missing object-level authorization** — always verify resource ownership
- **Storing passwords in plaintext** — use bcrypt/Argon2 for hashing
- **Over-sharing error details** — return generic errors; log details internally
- **CORS misconfiguration** — don't use `Access-Control-Allow-Origin: *` with credentials
- **Missing rate limiting** — always implement rate limiting to prevent abuse
- **Using weak JWT secrets** — use strong, randomly generated secrets (256+ bits)
- **Hardcoded secrets** — never commit secrets to version control

---

## Key Design Considerations

- **Defense-in-Depth Strategy:**
  - Layer 1: Network Security (WAF, DDoS protection)
  - Layer 2: Transport Security (TLS 1.2+, HSTS)
  - Layer 3: Authentication (OAuth2, JWT, MFA)
  - Layer 4: Authorization (RBAC, ABAC, object-level checks)
  - Layer 5: Input Validation (sanitization, parameterized queries)
  - Layer 6: Rate Limiting (per-user, per-endpoint, per-IP)
  - Layer 7: Audit Logging (who, what, when, from where)
  - Layer 8: Monitoring & Alerting (anomaly detection)

- **Security Principles:**
  - **Least Privilege** — grant minimum permissions needed
  - **Default Deny** — deny by default; explicitly allow
  - **Fail Secure** — when a security control fails, deny access
  - **Never Trust User Input** — all input is potentially malicious

- **Zero Trust Architecture:**
  - Every request authenticated and authorized
  - No trust based on network location
  - Short-lived tokens with frequent rotation
  - Encrypt everything in transit and at rest
  - Continuous monitoring and anomaly detection

---

## Real-World Scenarios

### Scenario 1: Broken Object Level Authorization (BOLA) Exploit
**Context:** A social media app has `GET /api/v1/messages/{messageId}`. The endpoint fetches the message by ID without checking if the authenticated user is a participant. An attacker iterates through message IDs (1 to 100000) and reads private conversations between other users. Thousands of private messages are exposed.

**Resolution:** Implement ownership checks in every resource endpoint. The message query must include the authenticated user's ID in the WHERE clause.

```java
// Vulnerable: No ownership check
@GetMapping("/{messageId}")
public MessageResponse getMessage(@PathVariable Long messageId) {
    Message message = messageRepository.findById(messageId)
        .orElseThrow(() -> new ResourceNotFoundException("Message not found"));
    return MessageMapper.toResponse(message);
}

// Secure: Ownership check built into the query
@GetMapping("/{messageId}")
public MessageResponse getMessage(@PathVariable Long messageId,
        @AuthenticationPrincipal User user) {
    Message message = messageRepository.findByIdAndParticipantId(messageId, user.getId())
        .orElseThrow(() -> new ResourceNotFoundException("Message not found"));
    return MessageMapper.toResponse(message);
}
```

Use a centralized `AuthorizationService.canAccessResource(user, resourceId, resourceType)` to avoid duplicating checks. Apply at both the service and repository level for defense-in-depth.

### Scenario 2: Rate Limiting Bypass & Credential Stuffing
**Context:** Your login endpoint has rate limiting at 5 attempts/minute per IP. An attacker uses a botnet of 10,000 IPs to perform credential stuffing — each IP tries 5 different password combinations per minute. 50,000 login attempts per minute pass through the rate limiter. User accounts with weak passwords are compromised.

**Resolution:** Implement multi-layer rate limiting: (1) Per-IP: 5 attempts/minute (stops single-source brute force). (2) Per-account: 5 attempts/minute regardless of source IP (stops distributed credential stuffing). (3) Per-device fingerprint: detected via TLS fingerprint, headers, and timing patterns. (4) Global: 10,000 login attempts/minute across all sources. Additionally, require CAPTCHA after 3 failed attempts on any account.

```java
@Component
public class LoginRateLimiter {
    private final RedisTemplate<String, String> redis;

    public boolean isAllowed(String username, String clientIp) {
        String perIpKey = "rate:ip:" + clientIp;
        String perUserKey = "rate:user:" + username;

        // Per-IP limit: 5 per minute
        Long ipAttempts = redis.opsForValue().increment(perIpKey);
        if (ipAttempts == 1) redis.expire(perIpKey, 1, TimeUnit.MINUTES);
        if (ipAttempts > 5) return false;

        // Per-account limit: 5 per minute (regardless of IP)
        Long userAttempts = redis.opsForValue().increment(perUserKey);
        if (userAttempts == 1) redis.expire(perUserKey, 1, TimeUnit.MINUTES);
        return userAttempts <= 5;
    }
}
```

### Scenario 3: Third-Party API Compromise Propagation
**Context:** Your API aggregates weather data from a third-party service and forwards the response directly to clients. The third-party service is compromised — an attacker injects malicious JavaScript into the weather description field. Your API blindly forwards the response to all 10,000 concurrent users. XSS attack propagates through your trusted API.

**Resolution:** Never trust third-party API responses. Always validate, sanitize, and transform before forwarding. Implement response schema validation that strips unexpected fields and sanitizes string content.

---

## Scenario-Based Questions

1. **Q: A user discovers they can access other users' order details by changing the order ID in the URL (`GET /api/orders/123` -> `/api/orders/456`). What's the vulnerability, and how do you fix it comprehensively without adding checks to every endpoint?**
   - A: This is Broken Object Level Authorization (BOLA), OWASP API Security #1. Fix: (1) Implement a centralized `AuthorizationService` with `canAccessOrder(userId, orderId)` that's automatically applied via AOP or a Spring Security `@PostAuthorize` annotation. (2) Use a base repository class that always filters by the authenticated user's ID. (3) For JPA: add `@EntityGraph` or `WHERE user_id = :principalId` to queries. (4) Never rely on the client to restrict access — always verify on the server. (5) Use UUIDs instead of sequential IDs to make guessing harder (defense-in-depth, not a replacement for authorization).

2. **Q: Your login endpoint has rate limiting per IP, but attackers rotate through thousands of IPs to perform credential stuffing. How do you stop this without blocking legitimate users behind a shared NAT IP?**
   - A: (1) Per-account rate limiting: track failed attempts per username, not per IP. Lock the account after 5 failures regardless of source IP. (2) Device fingerprinting: use TLS fingerprint, browser headers, and timing patterns to identify the client behind the IP. (3) CAPTCHA after 2 failed attempts — this stops automated tools while letting humans through. (4) Use WebAuthn/passkeys for high-value accounts. (5) Monitor for credential stuffing patterns: same username tried from many IPs in rapid succession — this is credential stuffing, not a single attacker.

3. **Q: Your API returns `404 Not Found` for non-existent resources but `403 Forbidden` for resources the user doesn't own. An attacker can determine which resource IDs exist by comparing status codes. How do you fix this information leak?**
   - A: Return `404 Not Found` for both cases. The user doesn't need to know the difference between "resource doesn't exist" and "you don't have access." Implementation: in the service layer, catch all authorization failures and throw a generic `ResourceNotFoundException`. This prevents ID enumeration. For audit purposes, log the actual reason (auth failure vs not found) internally but always return 404 to the client.

4. **Q: A third-party API your service depends on is compromised. Your API blindly forwards the malicious response to clients, resulting in an XSS attack. How do you design your API to prevent this class of vulnerability?**
   - A: This is OWASP #10 (Unsafe Consumption of APIs). (1) Validate third-party responses against a strict schema — reject any unexpected fields or content types. (2) Sanitize all string fields: strip HTML tags, encode special characters, reject executable content. (3) Never forward third-party responses directly — always transform them into your own response DTOs. (4) Treat the third-party API as an untrusted external system, subject to the same security controls as user input. (5) Implement a circuit breaker: if the third-party API returns malformed responses, fail closed rather than propagating potentially malicious data.

5. **Q: Your file upload endpoint is used to upload profile pictures. An attacker uploads a file named `../../etc/passwd` with a PHP shell inside. How do you secure this endpoint?**
   - A: (1) Validate file size: limit to 5MB for profile pictures. (2) Whitelist allowed MIME types: `image/jpeg`, `image/png`, `image/webp` only. (3) Validate magic bytes (not just extension): read the first bytes and verify they match the claimed format. (4) Sanitize filename: strip path traversal sequences (`../`, `..\\`), use a random UUID as the stored filename. (5) Store files outside the web root — serve through a controlled endpoint with authorization. (6) Run virus scanning on upload. (7) Re-encode the image server-side to strip any embedded payloads.

6. **Q: Your API returns different error messages: "Invalid email" for unknown emails and "Invalid password" for known emails with wrong passwords. Attackers use this to enumerate valid user accounts. How do you fix this without degrading UX?**
   - A: Return a single generic message: "Invalid email or password." For UX, you can differentiate on the client side with progressive disclosure. On the server: (1) Always hash the password, even if the user doesn't exist (constant-time response). (2) Use the same database query that checks both email existence and password validity together. (3) Log the actual failure reason internally for security monitoring but never expose it in the response. (4) Add a random delay to prevent timing attacks.

7. **Q: Your multi-tenant SaaS application must ensure Tenant A cannot access Tenant B's data. How do you enforce this at the architecture level, not just in application code?**
   - A: (1) Database-level isolation: use a `tenant_id` column in every table and enforce it in a repository base class that automatically appends `WHERE tenant_id = ?` to every query. (2) Use separate database schemas per tenant (PostgreSQL schemas) for stronger isolation. (3) Set the tenant context in a `ThreadLocal` via a filter — extract `X-Tenant-Id` from the JWT. (4) Use Spring Data JPA's `@TenantFilter` annotation or Hibernate's multi-tenancy feature. (5) Add integration tests that specifically test cross-tenant access. (6) Audit: log all cross-tenant access attempts for detection. Never rely on developers remembering to add `WHERE tenant_id = ?` manually.

8. **Q: Your API fetches user-provided URLs to generate previews (link preview feature). An attacker provides `http://169.254.169.254/latest/meta-data/` to access the cloud metadata service (SSRF). How do you prevent this?**
   - A: (1) Whitelist allowed URL patterns: only allow `https://` URLs to known domains. (2) Block internal IP ranges: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.0/8, 169.254.0.0/16 (AWS metadata). (3) Resolve the hostname to an IP and check against blocklist before fetching. (4) Disable redirect following at the HTTP client level — attackers can use redirects to bypass the initial URL check. (5) Use a dedicated network segment for outbound requests with restrictive egress rules. (6) Set a timeout (5 seconds) to prevent slow loris attacks.

9. **Q: Your API supports both cookie-based session auth (for web clients) and JWT bearer token auth (for mobile/API clients). How do you design the authentication architecture so authorization logic is shared?**
   - A: (1) Use a `SecurityContextRepository` that tries multiple strategies: first check for a JWT in the `Authorization` header, then check for a session cookie. (2) Both strategies produce the same `Authentication` object (same principal, same authorities). (3) Authorization (role checks, `@PreAuthorize`) works identically regardless of how the user authenticated. (4) Spring Security supports this natively: configure both `oauth2ResourceServer` for JWT and `httpBasic` or `formLogin` for session auth. (5) The authentication filters are in a chain — the first to match sets the context. (6) Ensure the CORS and CSRF configuration is compatible with both auth methods.

10. **Q: You discover that a developer accidentally committed a file containing database credentials to a public GitHub repository. What's your incident response plan?**
    - A: (1) Immediately rotate the compromised credentials (database password, connection strings). (2) Use `git filter-branch` or BFG Repo-Cleaner to remove the file from git history. (3) Force-push the cleaned history (coordinate with the team to rebase any open PRs). (4) Check GitHub's audit log for any forks or clones of the repo between commit and removal. (5) Add the credential pattern to `.gitignore` and implement pre-commit hooks (git-secrets, truffleHog) to prevent recurrence. (6) Rotate ALL credentials, not just the leaked one — assume the entire credential store is compromised if the file contained multiple secrets. (7) Conduct a root cause analysis: why was the file in the repo? Improve secret management (vault, environment variables, Kubernetes secrets).

---

## Interview Questions

1. **What is the OWASP API Security Top 10?**
   - A: The top 10 API security risks including BOLA (#1), Broken Authentication (#2), Broken Object Property Level Authorization (#3), Unrestricted Resource Consumption (#4), Broken Function Level Authorization (#5), and others like SSRF, misconfiguration, and unsafe API consumption.

2. **What is BOLA and how do you prevent it?**
   - A: Broken Object Level Authorization — accessing resources without ownership verification. Prevent by always verifying the authenticated user owns the requested resource. Use centralized authorization checks, repository-level filtering, and never trust user-provided IDs without auth verification.

3. **What is the difference between authentication and authorization?**
   - A: Authentication (AuthN) verifies identity — "Who are you?" Authorization (AuthZ) determines permissions — "What can you do?" Authentication comes first, then authorization checks what the authenticated principal is allowed to access.

4. **How do you protect against credential stuffing?**
   - A: Per-account rate limiting (5 attempts/minute), CAPTCHA after failed attempts, account lockout, WebAuthn/passkeys, credential monitoring (Have I Been Pwned), and multi-factor authentication. Per-IP rate limiting alone is insufficient — attackers use botnets.

5. **What is SSRF and how do you prevent it?**
   - A: Server-Side Request Forgery — tricking the server into making requests to internal systems. Prevent by URL allowlisting, blocking internal IP ranges, disabling redirects, using a dedicated network segment for outbound requests, and never passing user input directly to URL fetchers.

6. **How do you securely handle file uploads?**
   - A: Validate file size and MIME type (check magic bytes, not just extension). Sanitize filenames (remove path traversal). Store outside web root with random filenames. Serve through controlled download endpoints with authorization. Run malware scanning.

7. **What security headers should every API response include?**
   - A: `Strict-Transport-Security` (HSTS), `X-Content-Type-Options: nosniff`, `Content-Security-Policy`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`. These prevent common browser-level attacks.

8. **How do you implement multi-tenancy security?**
   - A: Use a tenant ID in every query, extracted from the authentication context. Apply tenant filtering at the repository level (automatically appended to all queries). Use separate database schemas for stricter isolation. Never trust the client to specify their tenant ID.

9. **How do you design a defense-in-depth security strategy?**
   - A: Layer 1: Network (WAF, DDoS protection). Layer 2: Transport (TLS 1.2+). Layer 3: Authentication (OAuth2, JWT). Layer 4: Authorization (RBAC, object-level checks). Layer 5: Input validation. Layer 6: Rate limiting. Layer 7: Audit logging. Layer 8: Monitoring and alerting.

10. **What should you do if credentials are accidentally committed to a public repository?**
    - A: Immediately rotate the leaked credentials. Remove the file from git history (BFG Repo-Cleaner). Check for forks/clones. Implement pre-commit secret scanning. Add the credential pattern to `.gitignore`. Conduct root cause analysis.

---

## Developer Recommendations

- **Always implement object-level authorization — it's the #1 API vulnerability** — BOLA (OWASP #1) affects most APIs because developers add authorization at the endpoint level but forget at the object level. Every resource lookup must verify the authenticated user owns or has access to that specific resource. Use a centralized `AuthorizationService` and apply it consistently. Never rely on obfuscation (UUIDs) as a substitute for authorization — UUIDs prevent guessing but don't prevent authenticated users from accessing data they shouldn't see.

- **Implement per-account rate limiting, not just per-IP** — Per-IP rate limiting stops simple brute force but not distributed credential stuffing (attackers use botnets with thousands of IPs). Per-account rate limiting tracks failed attempts by username regardless of source IP, stopping distributed attacks cold. Combine both: 5 attempts/minute per IP AND 5 attempts/minute per account. Add CAPTCHA after 2 failures to allow legitimate users who forgot their password to retry.

- **Never trust third-party API responses** — OWASP #10 (Unsafe Consumption of APIs) is often overlooked. A compromised third-party service can inject malicious data into your API. Always validate third-party responses against a strict schema, sanitize all string fields, and transform responses into your own DTOs. Treat third-party APIs as untrusted input sources — apply the same validation that you apply to user input.

- **Use security headers from day one** — `Strict-Transport-Security`, `X-Content-Type-Options`, `Content-Security-Policy`, `X-Frame-Options`, and `Referrer-Policy` headers are zero-cost security improvements that prevent entire classes of attacks. Add them as a filter in your API gateway or web server configuration. These headers tell browsers how to handle your API responses safely.

- **Implement a centralized audit logging system** — Every security-relevant action (login, logout, data access, permission changes, failed authorization attempts) should be logged with: who, what, when, from where (IP), and outcome (success/failure). Use structured logging (JSON) with correlation IDs. Store logs in a tamper-proof system (SIEM, immutable log store). Audit logs are essential for incident response, forensics, and compliance (PCI DSS, SOC 2, GDPR).

- **Use a secrets management system, not environment variables** — Environment variables are better than hardcoded secrets but still leak through error messages, logs, and process listings. Use a dedicated secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) with automatic rotation, access auditing, and encryption. Never commit secrets to version control — use pre-commit hooks to enforce this.
