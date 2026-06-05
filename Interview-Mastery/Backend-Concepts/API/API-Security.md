# API Security

## 1. Executive Summary

API security encompasses the strategies, protocols, and practices designed to protect APIs from unauthorized access, data breaches, injection attacks, and abuse. With APIs becoming the primary interface for modern applications, securing them is critical. Key aspects include authentication, authorization, rate limiting, input validation, encryption, and monitoring. This guide covers the OWASP API Security Top 10, common attack vectors, and defense mechanisms using Java/Spring Boot.

## 2. Core Theory

### Authentication vs Authorization

- **Authentication** (AuthN): Verifying the identity of a client (Who are you?)
- **Authorization** (AuthZ): Determining what an authenticated client can do (What are you allowed to do?)

### The CIA Triad for APIs

- **Confidentiality**: Data is accessible only to authorized parties (encryption, access control).
- **Integrity**: Data is not tampered with during transit or storage (signatures, HTTPS).
- **Availability**: The API is accessible when needed (rate limiting, DDoS protection).

### OWASP API Security Top 10 (2023)

1. Broken Object Level Authorization (BOLA)
2. Broken Authentication
3. Broken Object Property Level Authorization
4. Unrestricted Resource Consumption
5. Broken Function Level Authorization
6. Unrestricted Access to Sensitive Business Flows
7. Server Side Request Forgery (SSRF)
8. Security Misconfiguration
9. Improper Inventory Management
10. Unsafe Consumption of APIs

### Security Layers

```
Client -> WAF -> API Gateway -> Rate Limiter -> AuthN -> AuthZ -> Input Validation -> API Logic -> Database
```

## 3. Under-the-Hood Deep Dive

### HTTPS/TLS Handshake

1. Client sends a `ClientHello` with supported TLS versions and cipher suites.
2. Server responds with `ServerHello`, its certificate, and selected cipher suite.
3. Client verifies the certificate against a trusted CA.
4. Client generates a pre-master secret, encrypts it with the server's public key.
5. Both parties derive the session key from the pre-master secret.
6. Client sends `Finished` message encrypted with the session key.
7. Server sends `Finished` message. Secure channel established.

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
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            )
            .addFilterBefore(new RateLimitFilter(), BasicAuthenticationFilter.class)
            .build();
    }
}
```

## 4. Production Code Examples

### JWT Authentication Filter

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
    public List<UserResponse> getAllUsers() {
        return userService.findAll().stream()
            .map(UserMapper::toResponse)
            .collect(Collectors.toList());
    }

    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasAuthority('USER_DELETE')")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }

    @PostMapping("/roles")
    @PreAuthorize("hasRole('SUPER_ADMIN')")
    public ResponseEntity<RoleResponse> createRole(@Valid @RequestBody CreateRoleRequest request) {
        return ResponseEntity.status(201)
            .body(RoleMapper.toResponse(roleService.create(request)));
    }
}
```

### Role and Permission Model

```java
@Entity
@Table(name = "roles")
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String name;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "role_permissions",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id"))
    private Set<Permission> permissions = new HashSet<>();
}

@Entity
@Table(name = "permissions")
public class Permission {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String name;

    @Column
    private String description;
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
    public ResponseEntity<OrderResponse> getOrder(
            @PathVariable Long orderId,
            @AuthenticationPrincipal User user) {

        Order order = orderService.findById(orderId);

        // Object-level authorization check
        if (!authService.canAccessOrder(user, order)) {
            return ResponseEntity.status(403).build();
        }

        return ResponseEntity.ok(OrderMapper.toResponse(order));
    }
}

@Component
public class AuthorizationService {

    public boolean canAccessOrder(User user, Order order) {
        // Admin can access any order
        if (user.hasRole("ADMIN")) {
            return true;
        }
        // Users can only access their own orders
        return order.getUserId().equals(user.getId());
    }

    public boolean canModifyResource(User user, Long resourceOwnerId) {
        return user.hasRole("ADMIN") || user.getId().equals(resourceOwnerId);
    }
}
```

### API Key Authentication

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
            ApiKeyAuthenticationToken authToken =
                new ApiKeyAuthenticationToken(keyDetails);
            SecurityContextHolder.getContext().setAuthentication(authToken);
        }

        filterChain.doFilter(request, response);
    }
}
```

## 5. Real-World Scenarios

### Scenario 1: Payment API Idempotency

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> processPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @Valid @RequestBody PaymentRequest request) {

    // Check if this request was already processed
    Optional<PaymentResult> existing = idempotencyService
        .getResult(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(PaymentMapper.toResponse(existing.get()));
    }

    // Process the payment
    PaymentResult result = paymentService.process(request);

    // Store the result for idempotency
    idempotencyService.storeResult(idempotencyKey, result);

    return ResponseEntity.status(201).body(PaymentMapper.toResponse(result));
}
```

### Scenario 2: Multi-Tenant Data Isolation

```java
@Entity
@Table(name = "documents")
public class Document {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String tenantId;

    private String title;
    private String content;

    // Tenant ID is set automatically
    @PrePersist
    public void prePersist() {
        if (tenantId == null) {
            this.tenantId = TenantContext.getCurrentTenantId();
        }
    }
}

@Component
public class TenantFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String tenantId = httpRequest.getHeader("X-Tenant-Id");

        if (tenantId == null || tenantId.isBlank()) {
            throw new BadRequestException("X-Tenant-Id header is required");
        }

        TenantContext.setCurrentTenantId(tenantId);
        try {
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}
```

### Scenario 3: Secure File Upload

```java
@PostMapping("/files/upload")
public ResponseEntity<FileResponse> uploadFile(
        @RequestParam("file") MultipartFile file) {

    // Validate file size
    if (file.getSize() > MAX_FILE_SIZE) {
        throw new BadRequestException("File exceeds maximum size of 10MB");
    }

    // Validate file extension
    String filename = file.getOriginalFilename();
    String extension = filename.substring(filename.lastIndexOf(".") + 1).toLowerCase();
    if (!ALLOWED_EXTENSIONS.contains(extension)) {
        throw new BadRequestException("File type not allowed: " + extension);
    }

    // Scan for malware (pseudo)
    if (!virusScanner.isClean(file)) {
        throw new BadRequestException("File failed security scan");
    }

    // Sanitize filename
    String sanitizedFilename = sanitizeFilename(filename);

    // Store with a random name to prevent path traversal
    String storedName = UUID.randomUUID() + "." + extension;
    Path targetPath = storageDir.resolve(storedName).normalize();

    if (!targetPath.startsWith(storageDir)) {
        throw new SecurityException("Path traversal detected");
    }

    file.transferTo(targetPath.toFile());

    return ResponseEntity.status(201)
        .body(new FileResponse(storedName, sanitizedFilename, file.getSize()));
}

private String sanitizeFilename(String filename) {
    return filename.replaceAll("[^a-zA-Z0-9.-]", "_");
}
```

## 6. Performance

### Security Performance Considerations

- **Token Validation Caching**: Cache JWT validation results to reduce CPU overhead.
- **Rate Limiting**: Use distributed rate limiting (Redis) for multi-instance deployments.
- **Connection Pooling**: Reuse HTTPS connections via connection pooling.
- **Asymmetric Crypto**: Use ECDSA over RSA for faster JWT signing/verification.
- **Read Replicas**: Offload authentication queries to read replicas.

### JWT Token Validation with Caching

```java
@Component
public class CachedJwtTokenProvider {

    private final CacheManager cacheManager;

    public CachedJwtTokenProvider(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    public boolean validateToken(String token) {
        Cache cache = cacheManager.getCache("jwt-validation");
        Cache.ValueWrapper cached = cache.get(token);

        if (cached != null) {
            return (boolean) cached.get();
        }

        boolean valid = performValidation(token);
        cache.put(token, valid);
        return valid;
    }

    private boolean performValidation(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(publicKey)
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

### Optimized Rate Limiting with Redis

```java
@Component
public class RedisRateLimiter {

    private final StringRedisTemplate redisTemplate;
    private final RedissonClient redisson;

    public RedisRateLimiter(StringRedisTemplate redisTemplate,
                           RedissonClient redisson) {
        this.redisTemplate = redisTemplate;
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

## 7. Security

### Common Attack Vectors and Defenses

### SQL Injection Prevention with JPA

```java
// UNSAFE - Never do this
@Query("SELECT u FROM User u WHERE u.name = '" + name + "'")
List<User> findByNameUnsafe(String name);

// SAFE - Use parameterized queries
@Query("SELECT u FROM User u WHERE u.name = :name")
List<User> findByName(@Param("name") String name);

// SAFE - Use Spring Data JPA derived queries
List<User> findByNameAndEmail(String name, String email);

// SAFE - Use native queries with parameters
@Query(value = "SELECT * FROM users WHERE name = ?1", nativeQuery = true)
List<User> findByNameNative(String name);
```

### CSRF Protection

```java
@Configuration
@EnableWebSecurity
public class CsrfConfig {

    @Bean
    public SecurityFilterChain csrfFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
            )
            .build();
    }
}
```

### XSS Prevention

```java
@Component
public class XSSFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        chain.doFilter(
            new XSSRequestWrapper((HttpServletRequest) request),
            response
        );
    }
}

public class XSSRequestWrapper extends HttpServletRequestWrapper {

    public XSSRequestWrapper(HttpServletRequest request) {
        super(request);
    }

    @Override
    public String getParameter(String name) {
        String value = super.getParameter(name);
        return sanitize(value);
    }

    @Override
    public String[] getParameterValues(String name) {
        String[] values = super.getParameterValues(name);
        if (values == null) return null;
        return Arrays.stream(values)
            .map(this::sanitize)
            .toArray(String[]::new);
    }

    private String sanitize(String value) {
        if (value == null) return null;
        return value
            .replaceAll("<script>", "")
            .replaceAll("</script>", "")
            .replaceAll("on\\w+\\s*=", "")
            .replaceAll("javascript:", "");
    }
}
```

### Security Headers

```java
@Configuration
public class SecurityHeadersConfig {

    @Bean
    public WebMvcConfigurer securityHeaders() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/**")
                    .allowedMethods("GET", "POST", "PUT", "DELETE");
            }
        };
    }

    @Bean
    public FilterRegistrationBean<SecurityHeadersFilter> securityHeadersFilter() {
        FilterRegistrationBean<SecurityHeadersFilter> registrationBean =
            new FilterRegistrationBean<>();
        registrationBean.setFilter(new SecurityHeadersFilter());
        registrationBean.addUrlPatterns("/*");
        return registrationBean;
    }
}

public class SecurityHeadersFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        httpResponse.setHeader("X-Content-Type-Options", "nosniff");
        httpResponse.setHeader("X-Frame-Options", "DENY");
        httpResponse.setHeader("X-XSS-Protection", "0");
        httpResponse.setHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
        httpResponse.setHeader("Content-Security-Policy",
            "default-src 'self'; script-src 'self'; style-src 'self'");
        httpResponse.setHeader("Referrer-Policy", "strict-origin-when-cross-origin");
        chain.doFilter(request, response);
    }
}
```

### Secret Management

```java
@Component
public class SecretManager {

    private final Map<String, String> secrets = new ConcurrentHashMap<>();

    public SecretManager() {
        // In production, load from AWS Secrets Manager / HashiCorp Vault
        loadSecrets();
    }

    private void loadSecrets() {
        // Never hardcode secrets
        String dbPassword = System.getenv("DB_PASSWORD");
        String jwtSecret = System.getenv("JWT_SECRET");
        String apiKey = System.getenv("API_KEY");

        secrets.put("db.password", dbPassword);
        secrets.put("jwt.secret", jwtSecret);
        secrets.put("api.key", apiKey);
    }

    public String getSecret(String key) {
        return secrets.get(key);
    }
}
```

## 8. Common Mistakes

- **Exposing internal IDs**: Use UUIDs instead of auto-increment IDs in URLs.
- **Missing object-level authorization**: Always verify the authenticated user owns the resource.
- **Storing passwords in plaintext**: Use bcrypt/Argon2 for password hashing.
- **Over-sharing error details**: Return generic error messages; log details internally.
- **CORS misconfiguration**: Don't use `Access-Control-Allow-Origin: *` with credentials.
- **Missing rate limiting**: Always implement rate limiting to prevent abuse.
- **Using weak JWT secrets**: Use strong, randomly generated secrets (256+ bits).
- **Not validating redirect URIs**: Open redirects can be used for phishing.
- **Disabled security in profile**: Never `security.ignored=/**` even in dev.
- **Hardcoded secrets**: Never commit secrets to version control.

## 9. Senior Engineer Perspective

### Defense-in-Depth Strategy

```
Layer 1: Network Security (WAF, DDoS protection, VPN)
Layer 2: Transport Security (TLS 1.2+, HSTS)
Layer 3: Authentication (OAuth2, JWT, MFA)
Layer 4: Authorization (RBAC, ABAC, object-level checks)
Layer 5: Input Validation (sanitization, parameterized queries)
Layer 6: Rate Limiting (per-user, per-endpoint, per-IP)
Layer 7: Audit Logging (who, what, when, from where)
Layer 8: Monitoring & Alerting (anomaly detection)
```

### Security by Design Principles

- **Least Privilege**: Grant the minimum permissions needed.
- **Default Deny**: Deny access by default; explicitly allow.
- **Secure by Default**: Security should be opt-out rather than opt-in.
- **Fail Secure**: When a security control fails, deny access.
- **Separation of Concerns**: Separate authentication, authorization, and business logic.
- **Never Trust User Input**: All input is potentially malicious.

### Zero Trust Architecture for APIs

- Every request must be authenticated and authorized.
- No request is trusted based on network location.
- Use short-lived tokens and rotate frequently.
- Encrypt everything in transit and at rest.
- Continuous monitoring and anomaly detection.

## 10. Interview Questions (Easy)

1. What is the difference between authentication and authorization?
2. What is the purpose of HTTPS/TLS in API security?
3. What is CORS and why is it needed?
4. What is the difference between 401 and 403 HTTP status codes?
5. What is CSRF and how do you prevent it?
6. What is SQL injection and how do you prevent it?
7. What is XSS and how do you prevent it?
8. What are security headers and give three examples?
9. What is the principle of least privilege?
10. What is input validation and why is it important?

## Medium

1. How does JWT token validation work?
2. What is the OWASP API Security Top 10?
3. How do you implement role-based access control (RBAC)?
4. What is the difference between symmetric and asymmetric encryption?
5. How do you handle API key rotation?
6. What is rate limiting and what algorithms are used?
7. How do you implement secure password storage?
8. What is the difference between authentication and session management?
9. How does the Spring Security filter chain work?
10. What is object-level authorization and how do you implement it?

## 11. Advanced Interview Questions (Hard)

1. How would you implement attribute-based access control (ABAC) in a Spring Boot application?
2. Design a secure API key generation system with automatic rotation and revocation.
3. How do you prevent brute force attacks on login endpoints while maintaining performance?
4. Implement a secure file upload system resistant to all OWASP file upload attacks.
5. How would you design a session management system for a distributed microservices architecture?
6. Implement a defense against JWT replay attacks.
7. How do you secure an API against SSRF (Server Side Request Forgery)?
8. Design a multi-factor authentication system for a REST API.
9. How would you implement a secure webhook delivery system with signature verification?
10. Design a secrets management system for a microservices deployment on Kubernetes.

## System Design

1. Design an API security gateway that handles authentication, rate limiting, and auditing.
2. Design a zero-trust API architecture for a multi-cloud deployment.
3. Design a security monitoring and alerting system for a high-throughput API.
4. Design a cross-origin authentication flow using OAuth2 and PKCE.
5. Design a secure multi-tenant API with tenant isolation at the data and network level.
6. Design an API security testing pipeline integrated with CI/CD.
7. Design a distributed rate limiting system that works across data centers.
8. Design a secure API for financial transactions with PCI DSS compliance.
9. Design an API key management system with usage tracking and billing.
10. Design a security incident response system for automated threat detection.

## 12. Expert-Level Interview Questions (Architect-Level)

1. Design a comprehensive API security strategy for a global financial platform serving millions of users across multiple jurisdictions with varying regulatory requirements (GDPR, PCI-DSS, SOC2).
2. How would you implement a real-time threat detection system for API abuse using behavioral analysis and machine learning, without adding significant latency?
3. Design a secure cross-service authentication system for a service mesh with 200+ microservices, where services may be written in different languages and deployed across multiple clouds.
4. How do you balance security with developer experience? Design an API that is both highly secure and easy for third-party developers to integrate with.
5. Design a system for managing API credentials at scale (millions of API keys) with automatic rotation, revocation, granular permissions, and usage auditing.
6. How would you implement a defense-in-depth strategy for an API that processes sensitive PII data, including encryption at rest, in transit, and during processing (homomorphic encryption)?
7. Design a security architecture for an API that exposes write operations to untrusted third-party clients while preventing data exfiltration, mass assignment, and parameter tampering.
8. How do you handle security incident response for a compromised API? Design the complete incident response plan including detection, containment, recovery, and post-mortem.
9. Design a federated identity system that allows users to authenticate across multiple independent API platforms using a single identity provider while maintaining tenant isolation.
10. How would you design an API security compliance framework that automatically enforces security policies across all APIs in an organization and generates compliance reports for auditors?

## 13. Debugging & Troubleshooting

### Common Security Issues

- **Authentication failures**: Check token expiration, signing keys, user status.
- **Authorization denied**: Verify role/permission assignments, method security annotations.
- **CORS errors**: Check allowed origins, methods, and credentials settings.
- **Rate limit exceeded**: Check rate limit configuration and reset logic.
- **SSL handshake failures**: Verify certificate chain, TLS version, cipher suites.

### Security Audit Logging

```java
@Component
@Aspect
public class SecurityAuditAspect {

    private final AuditLogger auditLogger;

    @Around("@annotation(auditable)")
    public Object audit(ProceedingJoinPoint joinPoint, Auditable auditable) throws Throwable {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String username = auth != null ? auth.getName() : "anonymous";

        long start = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();
            auditLogger.log(AuditEvent.builder()
                .action(auditable.action())
                .resource(auditable.resource())
                .username(username)
                .success(true)
                .duration(System.currentTimeMillis() - start)
                .build());
            return result;
        } catch (Exception e) {
            auditLogger.log(AuditEvent.builder()
                .action(auditable.action())
                .resource(auditable.resource())
                .username(username)
                .success(false)
                .error(e.getMessage())
                .duration(System.currentTimeMillis() - start)
                .build());
            throw e;
        }
    }
}
```

## 14. Comparison Section

### JWT vs Session-Based Authentication

| Aspect | JWT | Session |
|--------|-----|---------|
| State | Stateless (client holds the token) | Stateful (server stores session) |
| Scalability | Highly scalable (no server-side storage) | Requires shared session store (Redis) |
| Revocation | Difficult (until token expires) | Instant (delete session) |
| Payload | Can store user data in token | Only session ID stored client-side |
| Size | Larger (contains claims) | Small (just session ID) |
| Security | Token must be stored securely | Session cookie has HttpOnly flag |

### OAuth2 vs API Keys

| Aspect | OAuth2 | API Keys |
|--------|--------|----------|
| Use Case | Third-party access, delegated auth | Server-to-server, simple access |
| Granularity | Scoped permissions | Typically all-or-nothing |
| Rotation | Automatic via refresh tokens | Manual |
| Complexity | High (multiple grant types) | Low |
| Audit Trail | Yes (who authorized what) | Limited |

### Security Headers

| Header | Purpose | Value |
|--------|---------|-------|
| Strict-Transport-Security | Enforce HTTPS | max-age=31536000 |
| X-Content-Type-Options | Prevent MIME sniffing | nosniff |
| X-Frame-Options | Prevent clickjacking | DENY |
| Content-Security-Policy | Prevent XSS | default-src 'self' |
| X-XSS-Protection | XSS filter (deprecated) | 0 |

## 15. Revision Notes

- AuthN = verification, AuthZ = permissions
- OWASP Top 10: BOLA is #1
- Spring Security filter chain orders filters
- Always use parameterized queries for SQL
- Rate limiting prevents resource exhaustion
- Use bcrypt/Argon2 for passwords (not MD5/SHA)
- JWT: stateless but hard to revoke
- Session: stateful but easy to revoke
- Security headers add browser-level protection
- Never trust user input

## 16. Cheat Sheet

```
+------------------------------------------------------------------+
| API SECURITY CHEAT SHEET                                         |
+------------------------------------------------------------------+
| AUTHENTICATION vs AUTHORIZATION                                  |
|   AuthN = Who you are   AuthZ = What you can do                  |
+------------------------------------------------------------------+
| SPRING SECURITY FILTER CHAIN (order matters):                    |
|   ChannelProcessingFilter                                        |
|   SecurityContextPersistenceFilter                                |
|   LogoutFilter                                                   |
|   UsernamePasswordAuthenticationFilter                            |
|   DefaultLoginPageGeneratingFilter                               |
|   BasicAuthenticationFilter                                      |
|   RequestCacheAwareFilter                                        |
|   SecurityContextHolderAwareRequestFilter                         |
|   AnonymousAuthenticationFilter                                  |
|   SessionManagementFilter                                        |
|   ExceptionTranslationFilter                                     |
|   FilterSecurityInterceptor                                      |
+------------------------------------------------------------------+
| COMMON ANNOTATIONS                                               |
|   @PreAuthorize("hasRole('ADMIN')")                              |
|   @PreAuthorize("hasAuthority('USER_DELETE')")                   |
|   @PostAuthorize("returnObject.owner == authentication.name")    |
|   @Secured("ROLE_ADMIN")                                         |
|   @RolesAllowed("ADMIN")                                         |
+------------------------------------------------------------------+
| PASSWORD HASHING                                                 |
|   BCryptPasswordEncoder   -> bcrypt (adaptive)                   |
|   SCryptPasswordEncoder   -> scrypt (memory-hard)                |
|   Pbkdf2PasswordEncoder   -> PBKDF2                              |
|   Argon2PasswordEncoder   -> Argon2 (recommended)                |
+------------------------------------------------------------------+
| SECURITY HEADERS                                                 |
|   Strict-Transport-Security: max-age=31536000                    |
|   X-Content-Type-Options: nosniff                                |
|   X-Frame-Options: DENY                                          |
|   Content-Security-Policy: default-src 'self'                    |
|   Referrer-Policy: strict-origin-when-cross-origin               |
+------------------------------------------------------------------+
| COMMON ATTACKS & DEFENSES                                        |
|   SQL Injection  -> Parameterized queries / JPA                  |
|   XSS           -> Input sanitization / CSP headers              |
|   CSRF          -> CSRF tokens / SameSite cookies                |
|   BOLA          -> Object-level auth checks                      |
|   Rate Abuse    -> Rate limiting / Throttling                    |
|   MITM          -> HTTPS / TLS                                   |
+------------------------------------------------------------------+
```
