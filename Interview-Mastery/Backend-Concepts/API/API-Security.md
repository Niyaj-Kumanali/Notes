# API Security

---

## Overview

- **Definition:** API security encompasses the strategies, protocols, and practices designed to protect APIs from unauthorized access, data breaches, injection attacks, and abuse. With APIs becoming the primary interface for modern applications — powering mobile apps, web frontends, third-party integrations, and internal microservices — securing them is critical to the overall security posture of any organization. A compromised API can expose sensitive data, enable unauthorized transactions, and serve as an entry point for broader system compromise.

- **Why It Exists:** APIs expose application logic and data to external clients, making them a prime attack vector for malicious actors. Without proper security controls, APIs can be exploited for data theft, service abuse, denial of service, credential harvesting, and unauthorized access to backend systems. Unlike traditional web applications that serve HTML pages with built-in browser security boundaries, APIs return structured data (JSON, XML) that can be consumed by any client, making them more vulnerable to automated attacks and scripted exploitation. The rise of microservices has dramatically increased the API attack surface — each service is a potential entry point that must be individually secured.

- **Authentication vs Authorization:**
  - **Authentication (AuthN)** — verifying identity: "Who are you?" This is the process of confirming that the client is who they claim to be, typically through credentials (passwords, API keys), tokens (JWT), certificates, or biometric factors. Authentication happens first and establishes the identity of the principal making the request.
  - **Authorization (AuthZ)** — determining permissions: "What can you do?" Once identity is established, authorization checks what resources and actions the authenticated principal is allowed to access. This includes role-based checks (RBAC), attribute-based policies (ABAC), and resource-level ownership verification. Authorization is meaningless without authentication, but authentication alone is insufficient — both are required for proper security.

- **CIA Triad for APIs:**
  - **Confidentiality** — data is accessible only to authorized parties through encryption (TLS for transit, AES for data at rest), access controls, and proper authentication. Leaked API responses or intercepted traffic can expose sensitive user data, trade secrets, or infrastructure details.
  - **Integrity** — data is not tampered with during transit or storage, ensured through TLS signatures, request/response signing (HMAC), and checksums. An attacker who modifies an API request in transit could change transaction amounts, alter permissions, or inject malicious payloads.
  - **Availability** — the API is accessible when needed, protected through rate limiting, throttling, DDoS protection, and proper capacity planning. Availability attacks (resource exhaustion, DDoS, API abuse) can take down critical services, causing revenue loss and reputational damage.

---

## Core Concepts

### OWASP API Security Top 10 (2023)

- **Broken Object Level Authorization (BOLA)** — accessing resources (orders, messages, user profiles) without verifying that the authenticated user owns or has permission to access that specific object. This is the most common and most critical API vulnerability because APIs typically expose object IDs in URLs, and developers often add endpoint-level authorization but forget object-level checks. Example: `GET /api/orders/123` returns order 123 even though the authenticated user owns only order 456. Prevention requires ownership checks in every resource endpoint.

- **Broken Authentication** — weak or bypassed authentication mechanisms, including predictable JWT tokens, missing token validation, weak password policies, session fixation, and improper logout. Attackers exploit this to impersonate legitimate users, gaining access to their data and permissions. Prevention includes strong password policies, multi-factor authentication, secure token storage, proper session management, and account lockout after repeated failures.

- **Broken Object Property Level Authorization** — accessing or modifying individual fields of an object without proper authorization. For example, a user updates their profile via `PATCH /api/users/me` and adds `"role": "admin"` to the request body, escalating their privileges. The API applies the entire request body without filtering which fields the user is allowed to modify. Prevention requires field-level authorization or using explicit DTOs that only expose allowed fields.

- **Unrestricted Resource Consumption** — lack of rate limiting, pagination limits, or request size constraints, leading to resource exhaustion, denial of service, and financial cost spikes. An attacker can send a single request that triggers an expensive database query, or a burst of requests that overwhelms the server. Prevention requires rate limiting per user/IP/endpoint, pagination on all list endpoints, request size limits, and timeout configuration.

- **Broken Function Level Authorization** — accessing administrative or privileged functions as a regular user. For example, a non-admin user calls `DELETE /api/admin/users/123` and the request succeeds because the endpoint only checks for a valid JWT, not for the admin role. Prevention requires role-based access control (RBAC) at the function/endpoint level, typically through annotations like `@PreAuthorize("hasRole('ADMIN')")`.

- **Unrestricted Access to Sensitive Business Flows** — abusing legitimate business flows for malicious purposes, such as automated ticket purchasing (scalping), coupon abuse, fake account creation, or rating manipulation. These attacks use the API exactly as designed but at a scale that harms the business. Prevention requires rate limiting on business-critical endpoints, CAPTCHA, behavior analysis, and anomaly detection.

- **Server Side Request Forgery (SSRF)** — tricking the server into making requests to internal systems that the attacker should not be able to access directly. For example, an API that fetches URL previews can be tricked into requesting `http://169.254.169.254/latest/meta-data/` to access cloud instance metadata (containing credentials). Prevention requires URL allowlisting, blocking internal IP ranges, and disabling redirects.

- **Security Misconfiguration** — default credentials left unchanged, unpatched systems, exposed debug endpoints, verbose error messages, missing security headers, and improperly configured CORS. These are the most preventable vulnerabilities but remain common due to configuration drift, complex deployment pipelines, and lack of automated security scanning. Prevention requires regular security audits, automated configuration scanning, and hardened deployment templates.

- **Improper Inventory Management** — undocumented endpoints, old API versions still active and accessible, shadow APIs deployed by teams without central oversight, and outdated documentation that fails to reflect the current API surface. Attackers discover these endpoints through scanning, fuzzing, or analyzing client-side code. Prevention requires a complete API inventory, strict version sunset processes, and API discovery tools that detect unauthorized endpoints.

- **Unsafe Consumption of APIs** — trusting third-party API responses without validation, leading to injection attacks, data corruption, or malicious data propagation. When your API aggregates data from external services and forwards responses to clients, a compromised third-party can inject malicious content into your trusted API channel. Prevention requires schema validation, response sanitization, and treating third-party responses as untrusted input.

### Production Code Examples

#### Spring Security Filter Chain

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(sm ->
                sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/actuator/health").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/v1/users/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .addFilterBefore(new RateLimitFilter(), BasicAuthenticationFilter.class)
            .addFilterBefore(new SecurityHeadersFilter(), RateLimitFilter.class)
            .build();
    }
}
```

#### JWT Authentication Filter

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtTokenProvider tokenProvider;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtTokenProvider tokenProvider,
                                   UserDetailsService userDetailsService) {
        this.tokenProvider = tokenProvider;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {
        String token = extractToken(request);
        if (token != null && tokenProvider.validateToken(token)) {
            String username = tokenProvider.getUsernameFromToken(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities());
            authentication.setDetails(
                new WebAuthenticationDetailsSource().buildDetails(request));
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

#### Method-Level Security with Role Hierarchies

```java
@EnableMethodSecurity
@Configuration
public class MethodSecurityConfig {}

@RestController
@RequestMapping("/api/v1/admin")
public class AdminController {

    @GetMapping("/users")
    @PreAuthorize("hasRole('ADMIN')")
    public List<UserResponse> getAllUsers() {
        return userService.findAll().stream()
            .map(UserMapper::toResponse)
            .collect(Collectors.toList());
    }

    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasAuthority('USER_DELETE')")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteById(id);
        return ResponseEntity.noContent().build();
    }

    @PostMapping("/bulk-import")
    @PreAuthorize("hasRole('SUPER_ADMIN')")
    public ResponseEntity<ImportResult> bulkImport(@RequestBody List<UserImport> users) {
        return ResponseEntity.ok(userService.bulkImport(users));
    }
}

// Role hierarchy: SUPER_ADMIN includes ADMIN, ADMIN includes USER
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.fromHierarchy("ROLE_SUPER_ADMIN > ROLE_ADMIN > ROLE_USER");
}
```

#### Object-Level Authorization (BOLA Prevention) with AOP

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService orderService;
    private final AuthorizationService authService;

    public OrderController(OrderService orderService,
                          AuthorizationService authService) {
        this.orderService = orderService;
        this.authService = authService;
    }

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderResponse> getOrder(
            @PathVariable Long orderId,
            @AuthenticationPrincipal User user) {
        Order order = orderService.findById(orderId);
        if (!authService.canAccessOrder(user, order)) {
            log.warn("Unauthorized access attempt: user={} order={}",
                user.getId(), orderId);
            return ResponseEntity.status(403).build();
        }
        return ResponseEntity.ok(OrderMapper.toResponse(order));
    }
}

@Component
public class AuthorizationService {
    private static final Logger log = LoggerFactory.getLogger(AuthorizationService.class);

    public boolean canAccessOrder(User user, Order order) {
        if (user.hasRole("ADMIN")) {
            return true; // Admins can access all orders
        }
        boolean isOwner = order.getUserId().equals(user.getId());
        boolean isParticipant = order.getParticipants().contains(user.getId());
        return isOwner || isParticipant;
    }
}
```

#### API Key Authentication with Rotation

```java
@Component
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {
    private static final String API_KEY_HEADER = "X-API-Key";
    private final ApiKeyService apiKeyService;

    public ApiKeyAuthenticationFilter(ApiKeyService apiKeyService) {
        this.apiKeyService = apiKeyService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {
        String apiKey = request.getHeader(API_KEY_HEADER);
        if (apiKey != null && apiKeyService.isValidApiKey(apiKey)) {
            ApiKeyDetails keyDetails = apiKeyService.getKeyDetails(apiKey);
            SecurityContextHolder.getContext()
                .setAuthentication(new ApiKeyAuthenticationToken(keyDetails));
        }
        filterChain.doFilter(request, response);
    }
}

@Service
public class ApiKeyService {
    private final ApiKeyRepository repository;

    public void rotateKey(String oldKey) {
        String newKey = generateSecureApiKey();
        repository.deactivateKey(oldKey);
        repository.saveKey(newKey, LocalDate.now().plusDays(90));
        // Notify customer with new key via secure channel
    }

    private String generateSecureApiKey() {
        byte[] keyBytes = new byte[32];
        new SecureRandom().nextBytes(keyBytes);
        return "sk_" + Base64.getUrlEncoder().withoutPadding().encodeToString(keyBytes);
    }
}
```

#### SQL Injection Prevention

```java
// UNSAFE - Never do this — string concatenation allows injection
@Query("SELECT u FROM User u WHERE u.name = '" + name + "'")
List<User> findByNameUnsafe(String name);

// SAFE - Use parameterized queries with named parameters
@Query("SELECT u FROM User u WHERE u.name = :name")
List<User> findByName(@Param("name") String name);

// SAFE - Spring Data JPA derived queries (automatically parameterized)
List<User> findByNameAndEmail(String name, String email);

// SAFE - Native queries with positional parameters
@Query(value = "SELECT * FROM users WHERE name = ?1", nativeQuery = true)
List<User> findByNameNative(String name);

// SAFE - Criteria API for dynamic queries
public List<User> searchUsers(String name, String email) {
    CriteriaBuilder cb = entityManager.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);
    List<Predicate> predicates = new ArrayList<>();
    if (name != null) {
        predicates.add(cb.equal(root.get("name"), name));
    }
    if (email != null) {
        predicates.add(cb.equal(root.get("email"), email));
    }
    query.where(predicates.toArray(new Predicate[0]));
    return entityManager.createQuery(query).getResultList();
}
```

#### Security Headers Filter

```java
@Component
public class SecurityHeadersFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain)
            throws IOException, ServletException {
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        httpResponse.setHeader("X-Content-Type-Options", "nosniff");
        httpResponse.setHeader("X-Frame-Options", "DENY");
        httpResponse.setHeader("Strict-Transport-Security",
            "max-age=31536000; includeSubDomains; preload");
        httpResponse.setHeader("Content-Security-Policy",
            "default-src 'self'; frame-ancestors 'none'");
        httpResponse.setHeader("Referrer-Policy",
            "strict-origin-when-cross-origin");
        httpResponse.setHeader("Cache-Control", "no-store");
        httpResponse.setHeader("Pragma", "no-cache");
        chain.doFilter(request, response);
    }
}
```

#### Rate Limiting with Redis

```java
@Component
public class RedisRateLimiter {
    private final RedissonClient redisson;

    public RedisRateLimiter(RedissonClient redisson) {
        this.redisson = redisson;
    }

    public boolean tryAcquire(String clientId, String endpoint) {
        RRateLimiter rateLimiter = redisson.getRateLimiter(
            "rate:" + clientId + ":" + endpoint);
        rateLimiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.MINUTES);
        return rateLimiter.tryAcquire();
    }
}
```

#### Input Validation and Sanitization

```java
@Component
public class InputSanitizer {

    // Sanitize string fields to prevent XSS and injection
    public static String sanitize(String input) {
        if (input == null) return null;
        // Strip HTML tags
        String cleaned = Jsoup.clean(input, Safelist.none());
        // Encode special characters
        cleaned = StringEscapeUtils.escapeHtml4(cleaned);
        // Remove control characters
        cleaned = cleaned.replaceAll("[\\p{Cn}\\p{Cc}&&[^\\n\\t]]", "");
        return cleaned.trim();
    }

    // Validate email format
    public static boolean isValidEmail(String email) {
        if (email == null || email.length() > 254) return false;
        return EmailValidator.getInstance().isValid(email);
    }

    // Sanitize filename to prevent path traversal
    public static String sanitizeFilename(String filename) {
        // Remove path separators
        String cleaned = filename.replaceAll("[/\\\\]", "");
        // Remove path traversal sequences
        cleaned = cleaned.replaceAll("\\.\\.", "");
        // Remove null bytes
        cleaned = cleaned.replace("\0", "");
        return cleaned;
    }
}
```

#### CSRF Protection for State-Changing Endpoints

```java
@Configuration
@EnableWebSecurity
public class CsrfSecurityConfig {

    @Bean
    public SecurityFilterChain csrfFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .csrfTokenRequestHandler(new SpaCsrfTokenRequestHandler()))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/**").permitAll()
                .anyRequest().authenticated()
            )
            .build();
    }
}
```

### Key Design Considerations

- **Defense-in-Depth Strategy:** Security cannot rely on a single layer. Implement multiple overlapping security controls so that if one layer fails, the next layer provides protection. The layers should be:
  - Layer 1: Network Security — Web Application Firewall (WAF), DDoS protection, network segmentation, IP allowlisting/blocklisting
  - Layer 2: Transport Security — TLS 1.2+ minimum, HSTS headers, certificate pinning, mutual TLS for service-to-service
  - Layer 3: Authentication — OAuth2/OIDC, JWT with short expiration, multi-factor authentication, API keys with rotation
  - Layer 4: Authorization — RBAC for function-level access, ABAC for attribute-based policies, object-level ownership checks, field-level filtering
  - Layer 5: Input Validation — parameterized queries, input sanitization, schema validation, allowlist-based validation
  - Layer 6: Rate Limiting — per-user, per-IP, per-endpoint, global limits with graduated responses (slow down, CAPTCHA, block)
  - Layer 7: Audit Logging — structured JSON logs, correlation IDs, tamper-proof storage, real-time alerting
  - Layer 8: Monitoring & Alerting — anomaly detection, behavioral analysis, automated incident response, regular penetration testing

- **Security Principles:**
  - **Least Privilege** — Grant the minimum permissions needed for each user, service, or function to perform its task. A user who only needs to read their own profile should not have write access to other users' profiles. A microservice that only needs to publish messages should not have admin database credentials.
  - **Default Deny** — Deny access by default; explicitly allow only what is necessary. Start with an empty allowlist and add permissions as needed. This is safer than starting with a permissive policy and trying to block specific actions.
  - **Fail Secure** — When a security control fails (e.g., a database connection error, a certificate validation failure), deny access by default. A fail-open behavior (allowing access when the check fails) is a common security vulnerability. Always assume the worst when a security check cannot be completed.
  - **Never Trust User Input** — All input from clients is potentially malicious until validated. This includes API request bodies, query parameters, path variables, headers, file uploads, and even data from trusted third parties (defense-in-depth). Validate everything on the server side — never rely on client-side validation for security.

- **Zero Trust Architecture:**
  - Every request is authenticated and authorized, regardless of network location — requests from within the VPC are not automatically trusted.
  - No trust is based on network location — an attacker who breaches the network perimeter should still face authentication and authorization controls at every service boundary.
  - Short-lived tokens with frequent rotation — JWT access tokens expire in 15 minutes, refresh tokens are rotated on every use, API keys are rotated every 90 days.
  - Encrypt everything in transit (TLS 1.2+) and at rest (AES-256) — no unencrypted communication is permitted even within internal networks.
  - Continuous monitoring and anomaly detection — baseline normal traffic patterns and alert on deviations that may indicate compromise or abuse.
  - Micro-segmentation — each service runs in its own security context with minimal network access to other services, enforced by service mesh policies (mTLS, authorization policies).

- **API Security Testing:**
  - Static analysis (SAST) — scan code for security vulnerabilities during development (SQL injection, hardcoded secrets, insecure deserialization).
  - Dynamic analysis (DAST) — run automated security scanners against running APIs to detect misconfigurations, injection flaws, and authentication bypasses.
  - Dependency scanning — continuously monitor third-party libraries for known vulnerabilities (CVEs) using tools like OWASP Dependency-Check, Snyk, or GitHub Dependabot.
  - Penetration testing — engage external security researchers to manually test your API for business logic flaws and complex attack chains that automated tools miss.
  - Fuzz testing — send malformed, unexpected, or random data to API endpoints to discover crashes, memory leaks, or unexpected behavior.

---

## Common Mistakes

- **Exposing internal IDs** — Using auto-increment database IDs (1, 2, 3) in API URLs makes resources predictable and enumerable. An attacker can iterate through IDs to discover the total number of users, orders, or messages in the system. Use UUIDs or ULIDs instead — they are globally unique, unpredictable, and reveal no information about the underlying data volume or ordering.
  - This *looks correct* because: sequential IDs are the default in every database, they're fast and small, and the enumeration risk only becomes visible after an attacker scripts a full scrape of your user base.

- **Missing object-level authorization** — Endpoint-level authorization (checking if a user has the USER role) is not the same as object-level authorization (checking if a user owns the specific order). The most common API vulnerability (BOLA) occurs because developers secure endpoints but forget to check resource ownership. Every resource lookup must verify that the authenticated user is authorized to access that specific object.
  - This *looks correct* because: the user is authenticated and the endpoint requires a valid role, so it seems like access is controlled — but authentication alone doesn't verify that user A can access user B's private resource.

- **Storing passwords in plaintext** — Passwords must never be stored in recoverable form. Use adaptive hashing algorithms like bcrypt (cost factor 12+), Argon2id, or PBKDF2 with a per-user salt. Never use MD5, SHA-1, or unsalted hashes. Hash passwords immediately upon receipt — never log, transmit, or store the plaintext password.
  - This *looks correct* because: in early development the database is local and "nobody can access it," so storing passwords as-is feels simpler and hashing seems like premature optimization — until the database is compromised and every user's password is immediately exposed.

- **Over-sharing error details** — Detailed error messages (stack traces, SQL queries, internal server names) help attackers understand the system. Return generic error messages to clients and log full details internally. For example, return "Invalid credentials" instead of "User not found" vs "Wrong password" — the latter enables username enumeration.
  - This *looks correct* because: detailed error messages help during development debugging, and it seems helpful to tell users exactly what went wrong — but that same helpfulness tells attackers which usernames are valid.

- **CORS misconfiguration** — Using `Access-Control-Allow-Origin: *` with credentials (`Access-Control-Allow-Credentials: true`) is a security vulnerability. If credentials are required, the origin must be explicitly specified. An overly permissive CORS policy allows any website to make authenticated requests to your API on behalf of the user.
  - This *looks correct* because: `*` is the simplest value and works during local development, and the credentials+wildcard combination seems like two independent settings — their dangerous interaction is not obvious from reading the configuration.

- **Missing rate limiting** — Without rate limiting, a single client can exhaust server resources, cause financial cost spikes (cloud bills), or scrape all data. Implement rate limiting at multiple levels: per API key, per user, per IP, per endpoint, and globally. Use different limits for different endpoints (login endpoints need stricter limits than read-only endpoints).
  - This *looks correct* because: during development and initial launch, traffic is low and rate limiting seems unnecessary — the first bot attack or data scraping incident demonstrates its value in a way that's hard to miss.

- **Using weak JWT secrets** — JWT tokens signed with weak secrets can be cracked offline. Use strong, randomly generated secrets of at least 256 bits (32 bytes). Rotate signing keys periodically. Use RS256 (asymmetric) instead of HS256 (symmetric) for multi-service environments so the signing key is not shared with every service.
  - This *looks correct* because: a short secret or a simple passphrase is easy to remember and configure, and the token validation works correctly — offline cracking isn't visible in normal operation until someone extracts the secret from a leaked config file.

- **Hardcoded secrets** — Secrets committed to version control are a matter of when, not if, they will be exposed. Use environment variables for local development, secrets management services (HashiCorp Vault, AWS Secrets Manager) for production, and never hardcode secrets in source code. Implement pre-commit hooks to scan for secret patterns.
  - This *looks correct* because: the repository is private, and embedding the secret in code is the fastest way to get running — the exposure only happens when the repo is accidentally made public or a contractor with access leaks it.

- **Trusting user input without validation** — Every input from the client is potentially malicious — this includes request bodies, query parameters, headers, file uploads, and even seemingly safe data types like integers or enums. Validate format, length, range, and type on every input field. Use allowlists rather than blocklists for validation.
  - This *looks correct* because: the frontend already validates everything, and most users are legitimate — the assumption that "nobody would send malicious data" holds until someone sends a SQL injection payload through a seemingly harmless search field.

- **Neglecting audit logging** — Without comprehensive audit logs, you cannot detect ongoing attacks, investigate incidents, or meet compliance requirements. Log all authentication attempts (success and failure), authorization failures, data access, and configuration changes with timestamps, user IDs, IP addresses, and action details.
  - This *looks correct* because: logging adds storage and performance overhead, and the system works fine without it — the value of audit logs is only realized during an incident, when it's already too late to start collecting them.

---

## Real-World Scenarios

### Scenario 1: Broken Object Level Authorization (BOLA) Exploit

- **Context:** A social media app has a `GET /api/v1/messages/{messageId}` endpoint that fetches direct messages by their database ID. The endpoint was designed to allow users to retrieve their own messages, but the developer only checked that the user is authenticated — not that the user is a participant in the conversation. The message IDs are sequential integers starting from 1. The application has 500,000 active users exchanging millions of private messages daily.

- **Problem:** An attacker registers a legitimate account and discovers that by changing the message ID in the URL (e.g., `GET /api/v1/messages/1001` → `GET /api/v1/messages/1002`), they can read other users' private messages. The API does not check whether the authenticated user is a sender or recipient of the message. The attacker writes a simple script that iterates through message IDs 1 to 100,000 and exfiltrates private conversations between other users — including personal information, business discussions, and potentially sensitive communications. The attacker can do this without triggering any alarms because all requests are authenticated with a valid token.

- **Resolution:** Implement object-level authorization by ensuring that every resource query includes the authenticated user's identity as a filter condition. The repository query must verify that the user is a participant in the message before returning it. Additionally, use a centralized `AuthorizationService` that can be applied consistently across all resource endpoints.

```java
// Vulnerable: No ownership check — any authenticated user can access any message
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
    Message message = messageRepository
        .findByIdAndParticipantIdsContaining(messageId, user.getId())
        .orElseThrow(() -> new ResourceNotFoundException("Message not found"));
    return MessageMapper.toResponse(message);
}

// Secure: Centralized authorization service applied via AOP
@GetMapping("/{messageId}")
@PostAuthorize("returnObject.body.senderId == authentication.principal.id || " +
               "returnObject.body.recipientId == authentication.principal.id")
public ResponseEntity<MessageResponse> getMessage(@PathVariable Long messageId) {
    Message message = messageRepository.findById(messageId)
        .orElseThrow(() -> new ResourceNotFoundException("Message not found"));
    return ResponseEntity.ok(MessageMapper.toResponse(message));
}
```

- Beyond the code fix, conduct an audit of all endpoints to identify similar missing ownership checks. Add automated security tests that specifically test for BOLA by attempting to access another user's resources. Implement API security scanning in CI/CD that detects common BOLA patterns.

### Scenario 2: Rate Limiting Bypass and Credential Stuffing

- **Context:** Your login endpoint has rate limiting at 5 attempts per minute per IP address. The application is a popular e-commerce platform with 2 million registered users. An attacker obtains a list of 50,000 email and password combinations from a previous data breach. Using a botnet of 10,000 compromised machines distributed across different ISPs and countries, the attacker performs credential stuffing — each bot IP tries 5 different password combinations per minute against different user accounts. The per-IP rate limiter sees each IP making only 5 requests per minute, which is within the configured limit. However, aggregate traffic is 50,000 login attempts per minute — far exceeding what legitimate users generate. Within hours, accounts with weak or reused passwords are compromised, and the attacker begins making fraudulent purchases and extracting stored payment information.

- **Problem:** Per-IP rate limiting alone is completely ineffective against distributed credential stuffing because each IP address stays well within the individual limit. The security control was designed for the wrong threat model — it assumed a single attacker machine rather than a coordinated botnet. The compromised accounts trigger fraud alerts, customer support tickets, and payment chargebacks. The security team discovers the attack only after multiple users report unauthorized transactions. By then, hundreds of accounts have been compromised, and the financial and reputational damage is significant.

- **Resolution:** Implement multi-layer rate limiting that addresses distributed attacks:

```java
@Component
public class LoginRateLimiter {
    private final RedisTemplate<String, String> redis;

    public LoginRateLimiter(RedisTemplate<String, String> redis) {
        this.redis = redis;
    }

    public RateLimitResult isAllowed(String username, String clientIp,
                                     String deviceFingerprint) {
        // Layer 1: Per-IP limit — 5 per minute (stops single-source brute force)
        String perIpKey = "rate:ip:" + clientIp;
        Long ipAttempts = redis.opsForValue().increment(perIpKey);
        if (ipAttempts == 1) redis.expire(perIpKey, 1, TimeUnit.MINUTES);
        if (ipAttempts > 5) {
            return RateLimitResult.blocked("IP rate limit exceeded");
        }

        // Layer 2: Per-account limit — 5 per minute regardless of source IP
        // This is the critical layer for credential stuffing
        String perUserKey = "rate:user:" + username;
        Long userAttempts = redis.opsForValue().increment(perUserKey);
        if (userAttempts == 1) redis.expire(perUserKey, 1, TimeUnit.MINUTES);
        if (userAttempts > 5) {
            return RateLimitResult.blocked("Account rate limit exceeded");
        }

        // Layer 3: Require CAPTCHA after 2 failed attempts
        if (userAttempts > 2) {
            return RateLimitResult.captchaRequired("CAPTCHA required");
        }

        // Layer 4: Detect credential stuffing pattern — same username from many IPs
        String ipSetKey = "rate:user:ips:" + username;
        redis.opsForSet().add(ipSetKey, clientIp);
        redis.expire(ipSetKey, 5, TimeUnit.MINUTES);
        Long uniqueIps = redis.opsForSet().size(ipSetKey);
        if (uniqueIps > 10) {
            // Same account attempted from 10+ IPs in 5 minutes — credential stuffing
            log.warn("Credential stuffing detected: username={}, ips={}", username, uniqueIps);
            return RateLimitResult.blocked("Suspicious activity detected");
        }

        // Layer 5: Global rate limit
        String globalKey = "rate:global:login";
        Long globalAttempts = redis.opsForValue().increment(globalKey);
        if (globalAttempts == 1) redis.expire(globalKey, 1, TimeUnit.MINUTES);
        if (globalAttempts > 10000) {
            return RateLimitResult.blocked("Global rate limit exceeded");
        }

        return RateLimitResult.allowed();
    }
}
```

- Additionally, implement WebAuthn/passkeys for high-value accounts, enable multi-factor authentication, monitor for credential stuffing patterns in real-time, and require password reset for accounts accessed from unfamiliar locations.

### Scenario 3: Third-Party API Compromise Propagation (Unsafe API Consumption)

- **Context:** Your e-commerce platform aggregates product reviews from a third-party service called "ReviewHub" and displays them on your product pages via `GET /api/v1/products/{id}/reviews`. Your API receives the review data from ReviewHub and forwards it directly to your clients as JSON — essentially acting as a pass-through. You trust ReviewHub because they are a well-known service and you have a signed contract with them. Your API does not validate, sanitize, or transform the data before forwarding it. Your application serves 1 million daily active users.

- **Problem:** ReviewHub suffers a supply chain attack — an attacker compromises their review submission endpoint and injects malicious JavaScript into the review text field. When your API fetches reviews from ReviewHub, it forwards the malicious payload directly to your clients. The review text containing `<script>alert('xss')</script>` is sent as part of your trusted API response. Any client application that renders this HTML without sanitization (e.g., a web app that uses `innerHTML` to display reviews) executes the injected JavaScript. The attacker can now steal session cookies, redirect users to phishing pages, or deface your site — all through a trusted third-party integration. Your security reputation is damaged because the attack appears to come from your API.

- **Resolution:** Never trust third-party API responses — they are an untrusted input source subject to the same security controls as user input. Implement a validation and transformation layer for all third-party data:

```java
@Service
public class ReviewAggregationService {

    private final ReviewHubClient reviewHubClient;
    private final ObjectMapper objectMapper;

    public ReviewAggregationService(ReviewHubClient reviewHubClient,
                                    ObjectMapper objectMapper) {
        this.reviewHubClient = reviewHubClient;
        this.objectMapper = objectMapper;
    }

    public List<ReviewResponse> getReviews(String productId) {
        // Fetch raw response from third-party
        String rawResponse = reviewHubClient.fetchReviews(productId);

        // Step 1: Validate response structure against expected schema
        try {
            JsonNode root = objectMapper.readTree(rawResponse);
            validateSchema(root);
        } catch (Exception e) {
            log.error("Invalid response schema from ReviewHub: {}", e.getMessage());
            throw new ExternalServiceException("Invalid review data received");
        }

        // Step 2: Deserialize with strict schema
        ReviewHubResponse hubResponse;
        try {
            hubResponse = objectMapper.readValue(rawResponse, ReviewHubResponse.class);
        } catch (JsonProcessingException e) {
            log.error("Failed to deserialize ReviewHub response: {}", e.getMessage());
            throw new ExternalServiceException("Malformed review data");
        }

        // Step 3: Transform to internal DTO — never forward third-party DTOs directly
        return hubResponse.getReviews().stream()
            .map(this::sanitizeAndTransform)
            .collect(Collectors.toList());
    }

    private ReviewResponse sanitizeAndTransform(ReviewHubItem item) {
        // Sanitize all string fields
        String sanitizedText = sanitizeHtml(item.getText());
        String sanitizedAuthor = sanitizeHtml(item.getAuthorName());
        String sanitizedTitle = sanitizeHtml(item.getTitle());

        return new ReviewResponse(
            item.getId(),
            sanitizedTitle,
            sanitizedText,
            sanitizedAuthor,
            item.getRating(),
            item.getCreatedAt()
        );
    }

    private String sanitizeHtml(String input) {
        if (input == null) return null;
        // Strip all HTML tags and encode special characters
        return Jsoup.clean(input, Safelist.none());
    }

    private void validateSchema(JsonNode root) {
        // Verify expected structure
        if (!root.has("reviews") || !root.get("reviews").isArray()) {
            throw new ValidationException("Missing 'reviews' array");
        }
        for (JsonNode review : root.get("reviews")) {
            if (!review.has("id") || !review.has("text")) {
                throw new ValidationException("Review missing required fields");
            }
            // Validate field types
            if (!review.get("rating").isInt() || review.get("rating").asInt() < 1
                || review.get("rating").asInt() > 5) {
                throw new ValidationException("Invalid rating value");
            }
        }
    }
}
```

- Beyond the code fix, implement a circuit breaker for the third-party integration: if ReviewHub returns malformed responses repeatedly, fail closed (return cached reviews) rather than propagating potentially malicious data. Monitor third-party response quality and alert on anomalies. Consider using a separate, sandboxed network segment for external API calls.

## Use Cases

- API security spans authentication, authorization, rate limiting, and input validation. Choose the right controls based on who your clients are and what they can access.

- **Securing public-facing REST/GraphQL APIs** — protecting endpoints exposed to the internet from unauthorized access and abuse
  - When to use: Your API is accessible from the open internet and handles sensitive data. Implement OAuth2 with short-lived access tokens, enforce TLS 1.3, apply rate limiting per client, and validate all inputs. Example: a payment gateway API that uses OAuth2 client credentials for machine-to-machine auth and tokenized card data.
  - **Avoid when:** The API is internal-only on a trusted network — mutual TLS or a service mesh with mTLS may be simpler than full OAuth2 flows.

- **Implementing OAuth2 / OIDC for SSO** — delegating authentication to a trusted identity provider
  - When to use: Users need to log in via Google, GitHub, or a corporate IdP. Use the Authorization Code flow with PKCE for public clients and the Client Credentials flow for server-to-server communication. Example: a SaaS platform that lets users sign in with their company's Okta account via OIDC.
  - **Avoid when:** You control both the client and the server on the same backend — a simpler API key or session-based auth may suffice.

- **Rate limiting and abuse prevention** — protecting APIs from excessive or malicious traffic
  - When to use: You need to ensure fair usage, prevent DDoS, or protect downstream databases from request spikes. Implement token bucket or sliding window rate limiting per API key or IP. Example: a weather API that allows 1000 requests/hour for free tier and 100,000/hour for enterprise.
  - **Avoid when:** The API is consumed only by internal services with predictable traffic patterns — rate limiting adds unnecessary complexity.

- **API key management for third-party developers** — issuing, rotating, and revoking credentials for external consumers
  - When to use: You provide an API for external developers to build integrations. Generate unique API keys per developer, support key rotation, and allow instant revocation. Example: a mapping service that issues API keys to mobile apps and tracks usage per key for billing.
  - **Avoid when:** You have only one or two known consumers — static credentials with IP whitelisting may be simpler.

---

## Scenario-Based Questions

- **Q:** A user discovers they can access other users' order details by changing the order ID in the URL (`GET /api/orders/123` -> `/api/orders/456`). What's the vulnerability, and how do you fix it comprehensively without adding checks to every endpoint?
  - This is Broken Object Level Authorization (BOLA), OWASP API Security #1, and the most common API vulnerability. To fix it comprehensively, implement a centralized `AuthorizationService` with a method like `canAccessOrder(userId, orderId)` that is automatically applied to all resource endpoints using Spring AOP or a `@PostAuthorize` annotation — this ensures no endpoint is missed. Use a base repository class that automatically includes the authenticated user's ID in every query (e.g., `WHERE user_id = :principalId` appended to all repository methods), so the database itself enforces ownership. For JPA, use `@EntityGraph` or `@TenantFilter` to inject ownership conditions. Never rely on the client to restrict access — always verify on the server. Use UUIDs instead of sequential IDs as a defense-in-depth measure (they make guessing harder but are not a replacement for authorization).

> **Interview follow-up:** Your centralized `AuthorizationService` now supports a "delegate access" feature where User A can view User B's orders — the simple `WHERE user_id = :principalId` query no longer works because the owner and the delegate are different. How do you model shared or delegated ownership in the authorization layer without making every query a complex OR-based condition?

- **Q:** Your login endpoint has rate limiting per IP, but attackers rotate through thousands of IPs to perform credential stuffing. How do you stop this without blocking legitimate users behind a shared NAT IP?
  - Implement per-account rate limiting that tracks failed attempts by username regardless of source IP — lock the account after 5 failures. This stops distributed credential stuffing because every attempt against the same account counts toward a single per-account counter, no matter how many IPs are used. Add device fingerprinting using TLS fingerprint, browser headers, timing patterns, and WebAuthn credentials to identify the client behind the IP — this helps distinguish between a legitimate user behind a NAT and an attacker. Require CAPTCHA after 2 failed attempts from any source, which stops automated tools while letting humans through. Use WebAuthn or passkeys for high-value accounts (e-commerce, banking) — these are phishing-resistant and immune to credential stuffing. Finally, monitor for credential stuffing patterns: the same username tried from many IPs in rapid succession is a strong indicator of an ongoing attack.

> **Interview follow-up:** You lock an account after 5 failed attempts — what stops an attacker from deliberately locking every user's account and causing a denial-of-service against your entire user base?

- **Q:** Your API returns `404 Not Found` for non-existent resources but `403 Forbidden` for resources the user doesn't own. An attacker can determine which resource IDs exist by comparing status codes. How do you fix this information leak?
  - Return `404 Not Found` for both cases — the user does not need to know the difference between "the resource doesn't exist" and "you don't have access to this resource." Implement this by having the service layer catch all authorization failures and throw a single `ResourceNotFoundException` that maps to a 404 response regardless of the underlying cause. This prevents attackers from enumerating valid resource IDs by observing response codes. For internal audit purposes, log the actual reason (authorization failure vs. resource not found) to a secure, tamper-proof log store with the user ID, resource ID, and reason — this maintains security visibility without leaking information to clients. Apply the same principle to login endpoints: return "Invalid email or password" for both unknown emails and wrong passwords to prevent username enumeration.

- **Q:** A third-party API your service depends on is compromised. Your API blindly forwards the malicious response to clients, resulting in an XSS attack. How do you design your API to prevent this class of vulnerability?
  - This is OWASP #10 (Unsafe Consumption of APIs). Never forward third-party responses directly to clients — always deserialize, validate, and transform them into your own internal DTOs, then serialize your DTOs as the response. Validate third-party responses against a strict JSON schema that defines required fields, types, and value ranges — reject any response that does not match the schema. Sanitize all string fields by stripping HTML tags, encoding special characters, and rejecting executable content using a library like Jsoup or OWASP Java HTML Sanitizer. Treat the third-party API as an untrusted external system that is subject to the same security controls as direct user input — validate everything. Implement a circuit breaker: if the third-party API returns malformed responses repeatedly, fail closed by serving cached data or returning an error rather than propagating potentially malicious data. Finally, use a dedicated network segment for outbound requests with restrictive egress rules to limit blast radius if the third-party service is compromised.

> **Interview follow-up:** You fail closed when the third-party API returns malformed data — but your cached data is 6 hours old, and the product team is getting complaints that inventory levels are wrong. How do you balance serving potentially stale-but-safe cached data against serving validated-but-correct data when the third-party service has a partial, recoverable failure?

- **Q:** Your file upload endpoint is used to upload profile pictures. An attacker uploads a file named `../../etc/passwd` with a PHP shell inside. How do you secure this endpoint?
  - Implement multiple layers of defense: validate file size (limit to 5MB for profile pictures) to prevent resource exhaustion; whitelist allowed MIME types (`image/jpeg`, `image/png`, `image/webp` only) and verify the actual file content via magic bytes (first bytes of the file), not just the file extension or Content-Type header — an attacker can rename a PHP file to .jpg. Sanitize the filename by stripping path traversal sequences (`../`, `..\\`), removing null bytes, and generating a random UUID as the stored filename — the original filename is never used for storage. Store files outside the web root directory so they cannot be executed directly by the web server — serve them through a controlled download endpoint that performs authorization checks. Run virus scanning on all uploaded files using ClamAV or a similar service. Re-encode the image server-side using a library like ImageMagick or Java's ImageIO to strip any embedded metadata or payloads — re-encoding produces a clean image file from the decoded pixel data, removing any steganographic or injection payloads.

> **Interview follow-up:** You re-encode uploaded images server-side — but a user's profile picture is now slightly blurry after JPEG compression, and they're complaining about quality loss. How do you balance security with image fidelity?

- **Q:** Your API returns different error messages: "Invalid email" for unknown emails and "Invalid password" for known emails with wrong passwords. Attackers use this to enumerate valid user accounts. How do you fix this without degrading UX?
  - Return a single generic message: "Invalid email or password" for all login failures. This prevents attackers from determining which emails are registered. For UX, you can differentiate on the client side with progressive disclosure — for example, after the first failed attempt, show a "Forgot password?" link that works for both known and unknown accounts (sending a reset link only if the account exists, without revealing which case). On the server side, always hash the password even if the user does not exist (constant-time comparison to prevent timing attacks) — query the user by email, and if not found, hash a dummy password before returning the generic error. Log the actual failure reason (user not found vs wrong password) internally for security monitoring but never expose it in the API response. Also add a random delay of 0-500ms to prevent timing-based enumeration where the response time differs between existing and non-existing users.

- **Q:** Your multi-tenant SaaS application must ensure Tenant A cannot access Tenant B's data. How do you enforce this at the architecture level, not just in application code?
  - Implement tenant isolation at multiple architectural layers. At the database layer, use a `tenant_id` column in every table and enforce it in a repository base class that automatically appends `WHERE tenant_id = ?` to every query — this prevents a developer from forgetting to filter by tenant. For PostgreSQL, use row-level security policies that automatically filter by the current tenant ID set in the session, providing a database-level guarantee even if the application code is flawed. For stronger isolation, use separate database schemas per tenant (PostgreSQL schemas) or separate databases entirely, which provides physical data separation. At the application layer, extract the tenant ID from the JWT or authentication context using a filter and set it in a `ThreadLocal` tenant context — all repositories and services pull the tenant ID from this context automatically. Use Spring Data JPA's `@TenantFilter` annotation or Hibernate's multi-tenancy feature with a `CurrentTenantIdentifierResolver` that resolves the tenant ID from the security context. Add integration tests that specifically test cross-tenant access by creating two tenants and verifying that each tenant's API calls return only their own data. Never rely on developers remembering to add `WHERE tenant_id = ?` manually — automate it at the framework level.

> **Interview follow-up:** You use a `ThreadLocal` to hold the tenant context, but a background job that processes data across all tenants doesn't set a tenant ID — every query returns zero results because the filter automatically appends `WHERE tenant_id = NULL`. How do you distinguish between "this request is for a specific tenant" and "this request crosses all tenants" without introducing an "admin bypass" that could be abused?

- **Q:** Your API fetches user-provided URLs to generate previews (link preview feature). An attacker provides `http://169.254.169.254/latest/meta-data/` to access the cloud metadata service (SSRF). How do you prevent this?
  - Implement a strict URL allowlist that only permits `https://` URLs to known, verified domains — reject any URL that does not match the allowlist. Before making the request, resolve the hostname to an IP address and check it against a blocklist of internal IP ranges: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.0/8, 169.254.0.0/16 (AWS metadata), and 100.64.0.0/10 (CGNAT). Also block IPv6 loopback and link-local addresses. Disable redirect following in the HTTP client — attackers can use open redirectors or DNS rebinding to bypass the initial URL check by having the first request pass validation while a redirect points to an internal service. Set a short timeout (5 seconds maximum) to prevent slow loris attacks and resource exhaustion. Run the URL fetcher in a dedicated network segment (e.g., a separate VPC or subnet) with restrictive egress rules that only allow outbound connections to the internet on port 443, and no ingress from the fetcher to internal services. Finally, cache and serve the preview, so the URL is fetched at most once per URL to prevent repeated SSRF probing.

> **Interview follow-up:** Your URL allowlist blocks `http://` and only allows `https://`, but a legitimate customer needs to preview content from an internal knowledge base hosted on `http://` — how do you handle this edge case without opening the SSRF vector?

- **Q:** Your API supports both cookie-based session auth (for web clients) and JWT bearer token auth (for mobile/API clients). How do you design the authentication architecture so authorization logic is shared?
  - Use a `SecurityContextRepository` that tries multiple authentication strategies in sequence: first check for a JWT in the `Authorization` header (for mobile/API clients), then check for a session cookie (for web clients). Both authentication paths should produce the same type of `Authentication` object with the same principal type and same authorities — this ensures that authorization logic (role checks, `@PreAuthorize` annotations, method security) works identically regardless of how the user authenticated. Spring Security supports this natively: configure both `oauth2ResourceServer` for JWT and `formLogin` or `httpBasic` for session-based auth in the same filter chain — Spring Security tries each filter in order and the first one that matches sets the `SecurityContext`. The key is that the `UserDetails` loaded by both paths contains the same roles and permissions, so the `SecurityContext` contains an identical `Authentication` object. Ensure CORS and CSRF configurations are compatible with both auth methods — for example, CSRF protection applies only to cookie-based auth (browsers send cookies automatically), while JWT auth is inherently CSRF-safe (the token must be explicitly included in the `Authorization` header).

- **Q:** You discover that a developer accidentally committed a file containing database credentials to a public GitHub repository. What's your incident response plan?
  - Immediately rotate the compromised credentials — change the database password, invalidate the connection strings, and redeploy all services that used the compromised credentials. Remove the file from git history using `git filter-branch` or BFG Repo-Cleaner to scrub it from all commits, not just the latest. Force-push the cleaned history to GitHub, but coordinate with the team first because this invalidates all existing clones and requires everyone to rebase their branches. Check GitHub's audit log for any forks or clones of the repository that were created between the commit and the removal — anyone who forked the repo before the cleanup still has the credentials. Add the credential file pattern to `.gitignore` and implement pre-commit hooks using tools like git-secrets or truffleHog that scan for secret patterns before allowing a commit to proceed. Rotate ALL credentials that were in the committed file, not just the one that was leaked — if the file contained database passwords, API keys, and cloud access keys, assume all are compromised because an attacker who sees one may have captured the entire file. Conduct a root cause analysis to understand why the credentials were in the source code in the first place, and improve the secret management process: use a vault (HashiCorp Vault, AWS Secrets Manager), environment variables, or Kubernetes secrets instead of configuration files in the repository.

---

## Interview Questions

- **What is the OWASP API Security Top 10?**
  - The OWASP API Security Top 10 is a list of the most critical security risks for APIs, published by the Open Web Application Security Project. The 2023 edition includes Broken Object Level Authorization (#1), Broken Authentication (#2), Broken Object Property Level Authorization (#3), Unrestricted Resource Consumption (#4), Broken Function Level Authorization (#5), Unrestricted Access to Sensitive Business Flows (#6), Server Side Request Forgery (#7), Security Misconfiguration (#8), Improper Inventory Management (#9), and Unsafe Consumption of APIs (#10). This list should be the starting point for any API security audit.

- **What is BOLA and how do you prevent it?**
  - Broken Object Level Authorization (BOLA) is the #1 API security risk — it occurs when an API endpoint allows a user to access a resource (order, message, document) without verifying that the user owns or has permission to access that specific object. Prevent it by always verifying ownership in every resource endpoint: include the authenticated user's ID in repository queries, use a centralized `AuthorizationService`, and apply `@PostAuthorize` annotations. Never rely on obfuscation (UUIDs, unpredictable IDs) as a substitute for authorization checks.

- **What is the difference between authentication and authorization?**
  - Authentication (AuthN) verifies identity — "Who are you?" — through credentials, tokens, or biometrics. Authorization (AuthZ) determines permissions — "What can you do?" — based on roles, attributes, or policies. Authentication always comes first; authorization checks what the authenticated principal is allowed to access. Both are required and serve different purposes in the security model.

- **How do you protect against credential stuffing?**
  - Per-account rate limiting (5 attempts/minute per username, not per IP) is the primary defense — this stops distributed attacks that use thousands of IPs. Add CAPTCHA after 2 failed attempts, implement account lockout after 5 failures, and consider WebAuthn or passkeys for high-value accounts (these are phishing-resistant). Monitor for credential stuffing patterns like the same username attempted from many IPs in rapid succession.

- **What is SSRF and how do you prevent it?**
  - Server-Side Request Forgery (SSRF) tricks the server into making requests to internal systems by providing a malicious URL. Prevent it by URL allowlisting (only allow `https://` to known domains), blocking internal IP ranges (10.0.0.0/8, 169.254.0.0/16, 127.0.0.0/8, etc.), disabling HTTP redirects in the client, using a dedicated network segment for outbound requests with restrictive egress rules, and never passing user input directly to a URL fetcher without validation.

- **How do you securely handle file uploads?**
  - Validate file size (max 5MB), whitelist MIME types and verify magic bytes (not just extension), sanitize filenames to remove path traversal sequences, store files outside the web root with random UUID filenames, serve through authorized download endpoints, run virus scanning, and re-encode images server-side to strip embedded payloads. Never trust the filename, Content-Type header, or extension provided by the client.

- **What security headers should every API response include?**
  - `Strict-Transport-Security` (HSTS — enforce HTTPS), `X-Content-Type-Options: nosniff` (prevent MIME sniffing), `Content-Security-Policy` (restrict resources the browser can load), `X-Frame-Options: DENY` (prevent clickjacking), and `Referrer-Policy: strict-origin-when-cross-origin` (control referrer information). These headers are zero-cost security improvements that prevent entire classes of browser-based attacks.

- **How do you implement multi-tenancy security?**
  - Use a tenant ID extracted from the authentication context and apply it automatically at the repository level using a base repository class or framework filter. Use separate database schemas or row-level security policies for stronger isolation. Never trust the client to specify their tenant ID — derive it from the security context. Add integration tests that specifically verify cross-tenant isolation cannot be bypassed.

- **How do you design a defense-in-depth security strategy?**
  - Layer multiple independent security controls: network security (WAF, DDoS protection), transport security (TLS 1.2+), authentication (OAuth2, JWT, MFA), authorization (RBAC, object-level checks), input validation (parameterized queries), rate limiting (per-user, per-endpoint), audit logging (who, what, when), and monitoring (anomaly detection). Each layer provides protection if the layer above is bypassed.

- **What should you do if credentials are accidentally committed to a public repository?**
  - Immediately rotate the compromised credentials — this is the most urgent action because the credentials are exposed. Remove the file from git history using BFG Repo-Cleaner or `git filter-branch`. Check for forks or clones that may have captured the credentials. Implement pre-commit secret scanning (git-secrets, truffleHog) to prevent recurrence. Conduct a root cause analysis and improve the secret management process to use a vault or secrets manager instead of config files in the repository.

---

## Developer Recommendations

- **Always implement object-level authorization — it is the #1 API vulnerability** — BOLA (OWASP #1) affects most APIs because developers add authorization at the endpoint level (checking if a user has the USER role) but forget at the object level (checking if the user owns this specific order). Every resource lookup must verify that the authenticated user owns or has permission to access that specific resource instance. Use a centralized `AuthorizationService` with methods like `canAccessOrder(user, orderId)` and apply it consistently across all endpoints using AOP or a base controller class. Never rely on obfuscation (UUIDs, unpredictable IDs) as a substitute for authorization — UUIDs prevent random guessing but do not prevent an authenticated user from accessing a resource that does not belong to them. The difference between endpoint authorization and object authorization is the difference between "is this user logged in?" and "is this user allowed to see this specific record?" — both are required. A healthcare startup used UUIDs instead of sequential IDs for patient records and considered the API "secure" — until a penetration tester registered as a patient, changed the UUID in the URL to another patient's ID, and accessed 5,000 medical records. UUIDs made guessing harder but didn't prevent authorized access to unauthorized data.

- **Implement per-account rate limiting, not just per-IP** — Per-IP rate limiting stops simple brute force from a single machine, but it is completely ineffective against distributed credential stuffing where attackers use botnets with thousands of IPs, each making requests well within the per-IP limit. Per-account rate limiting tracks failed attempts by username regardless of source IP, stopping distributed attacks cold because every attempt against the same account counts toward a single counter. Combine both strategies: 5 attempts per minute per IP AND 5 attempts per minute per account. Add CAPTCHA after 2 failed attempts to allow legitimate users who forgot their password to retry while blocking automated tools. For high-value accounts, implement WebAuthn or passkeys — these are phishing-resistant credentials that cannot be stuffed because they are bound to the device and domain. A social media platform with 200M users relied solely on per-IP rate limiting — during a credential stuffing campaign using 50,000 residential proxies, 1.2M accounts were compromised before the team realized every single IP was making only 3 requests per minute, well under the limit.

- **Never trust third-party API responses** — OWASP #10 (Unsafe Consumption of APIs) is often overlooked because third-party services are implicitly trusted. A compromised third-party service can inject malicious data into your API, and since the data comes from a "trusted" source, it bypasses normal input validation. Always validate third-party responses against a strict JSON schema, sanitize all string fields using an HTML sanitizer to strip malicious content, and transform the data into your own response DTOs instead of forwarding the third-party DTOs directly. Treat third-party APIs as untrusted input sources that are subject to the same validation, sanitization, and security controls as direct user input. Additionally, implement a circuit breaker that fails closed (serves cached data) when the third-party API returns malformed or suspicious responses.

- **Use security headers from day one** — `Strict-Transport-Security`, `X-Content-Type-Options`, `Content-Security-Policy`, `X-Frame-Options`, and `Referrer-Policy` headers are zero-cost security improvements that prevent entire classes of browser-based attacks. Add them as a filter in your API gateway, web server configuration, or application framework from the very first deployment. These headers cost nothing to implement (a few lines of configuration), require no maintenance, and provide immediate protection against common web vulnerabilities like clickjacking, MIME sniffing, and protocol downgrade attacks. There is no legitimate reason to omit security headers from any API response — make them a mandatory part of your deployment checklist.

- **Implement a centralized audit logging system** — Every security-relevant action (login success/failure, logout, data access, permission changes, failed authorization attempts, configuration modifications) should be logged with: who (user ID, API key), what (action, resource, change), when (ISO 8601 timestamp with timezone), from where (source IP, user agent), and outcome (success or failure with reason). Use structured logging in JSON format with correlation IDs that span microservices so you can trace a single user's actions across the entire system. Store logs in a tamper-proof system (SIEM, immutable log store, append-only database) that prevents attackers from covering their tracks. Audit logs are not optional — they are essential for incident response, forensic investigation, and compliance with regulations like PCI DSS, SOC 2, GDPR, and HIPAA.

- **Use a secrets management system, not environment variables** — Environment variables are better than hardcoded secrets but they still leak through error messages, stack traces, process listings, debugging tools, and log aggregation systems. Use a dedicated secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager) with automatic rotation, access auditing, fine-grained permissions, and encryption at rest and in transit. Never commit secrets to version control — implement pre-commit hooks using git-secrets or truffleHog that scan for patterns like `password=`, `-----BEGIN RSA PRIVATE KEY-----`, or `AKIA[0-9A-Z]{16}` (AWS access keys) and block the commit if any are found. Rotate secrets on a regular schedule (every 90 days) and immediately when a team member with access leaves the organization. A secrets manager also provides access logging so you can see who accessed which secret and when, which is critical for incident response.
