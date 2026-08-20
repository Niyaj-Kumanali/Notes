# Spring Security Questions

## Questions

1. What is Spring Security?
2. Difference between authentication and authorization.
3. What is security filter chain?
4. How does Spring Security process a request?
5. What is `SecurityContextHolder`?
6. What is `Authentication` object?
7. What is `GrantedAuthority`?
8. What is `UserDetails`?
9. What is `UserDetailsService`?
10. What is `AuthenticationProvider`?
11. What is `PasswordEncoder`?
12. Why use BCrypt?
13. What is JWT?
14. How does JWT authentication work?
15. What should a JWT contain?
16. What should a JWT not contain?
17. How do you validate JWT signature?
18. Difference between access token and refresh token.
19. How do you revoke JWT?
20. What is token rotation?
21. Where should JWT be stored?
22. Why is storing JWT in localStorage risky?
23. What is CSRF?
24. When can CSRF be disabled?
25. What is CORS?
26. How do you configure CORS?
27. Difference between `hasRole()` and `hasAuthority()`.
28. What is method-level security?
29. What is `@PreAuthorize`?
30. What is `@PostAuthorize`?
31. What is OAuth2?
32. Difference between OAuth2 and JWT.
33. What is OpenID Connect?
34. What is resource server?
35. What is authorization server?
36. How do you secure actuator endpoints?
37. How do you handle unauthorized response?
38. Difference between 401 and 403.
39. How do you implement RBAC?
40. How do you implement permission-based access?
41. How do you secure APIs behind a load balancer?
42. How do you handle `Authorization` header through proxies?
43. What is session fixation?
44. What is stateless session management?
45. How do you mix stateful UI and stateless API security?

---

## Answers

1. What is Spring Security?
   - **Answer:**
      - A framework for authentication, authorization, and protection against common web vulnerabilities
      - In a typical enterprise application, Spring Security can be used with JWT to secure REST APIs, configure security filter chains, and apply role-based access control for different user types
      - The filter chain architecture consists of `SecurityFilterChain` which replaced the old `WebSecurityConfigurerAdapter`, and a custom `JwtAuthenticationFilter` can be written that extends `OncePerRequestFilter` to intercept and validate tokens on every request
      - `SecurityFilterChain` beans are matched by request matchers (e.g., `requestMatcher("/api/**")`), allowing different filter configurations for public and private endpoints, and filters are executed in order with each handling a single security concern such as authentication, authorization, CSRF protection, or exception translation

2. Difference between authentication and authorization.
   - **Answer:**
      - Authentication verifies who you are (identity), authorization verifies what you can do (permissions)
      - For example, users log in with username/password (authentication), and the JWT contains roles like `ROLE_ADMIN` or `ROLE_PARTNER` (authorization) to control access to different API endpoints
      - Authentication creates a `UsernamePasswordAuthenticationToken` containing principal, credentials, and authorities, then stores it in `SecurityContextHolder` via `AuthenticationManager.authenticate()`, while authorization checks `GrantedAuthority` objects against the required permissions using `@PreAuthorize`, `.hasRole()`, or filter-chain-level access rules

3. What is security filter chain?
   - **Answer:**
      - The security filter chain is a sequence of filters that intercept every HTTP request to apply authentication, authorization, CSRF, CORS, and other security logic
      - A typical setup uses a custom filter chain with `JwtAuthenticationFilter` placed before `UsernamePasswordAuthenticationFilter` to validate JWT tokens on every API call
      - The filter chain is ordered with each `SecurityFilterChain` bean matched by `requestMatcher()` — a request matching `/api/**` uses one chain while `/actuator/**` uses another, and multiple filter chains can be used to separate public endpoints (no auth required) from private ones (JWT validation mandatory)

4. How does Spring Security process a request?
   - **Answer:**
      - Each request passes through the security filter chain
      - If a JWT is present, a custom filter extracts the token, validates the signature, loads user details, and creates an `Authentication` object stored in `SecurityContextHolder`
      - If JWT is missing or invalid, the request is rejected with 401 before reaching the controller
      - The full flow is: incoming request → `DelegatingFilterProxy` (delegates to Spring-managed bean) → `FilterChainProxy` (selects matching `SecurityFilterChain`) → custom filters (e.g., `JwtAuthenticationFilter`) → `UsernamePasswordAuthenticationFilter` (for login form) → `ExceptionTranslationFilter` (handles auth/denial exceptions) → `FilterSecurityInterceptor` (final authorization check)

5. What is `SecurityContextHolder`?
   - **Answer:**
      - `SecurityContextHolder` stores the `SecurityContext` of the currently authenticated user using a `ThreadLocal`
      - The logged-in user's details can be accessed in service layers by calling `SecurityContextHolder.getContext().getAuthentication()` to retrieve the username and roles for audit logging
      - Three storage modes exist: `MODE_THREADLOCAL` (default, one context per request thread), `MODE_INHERITABLETHREADLOCAL` (child threads inherit the parent's context, used for `@Async` methods), and `MODE_GLOBAL` (single shared context, only for single-threaded apps). A common issue is context loss with `@Async` where the spawned thread has no authentication — this can be fixed by calling `SecurityContextHolder.setStrategyName(Mode.INHERITABLETHREADLOCAL)` in the security configuration

6. What is `Authentication` object?
   - **Answer:**
      - `Authentication` represents the authenticated user's identity and authorities
      - After successful JWT validation, a `UsernamePasswordAuthenticationToken` is created containing the user principal, credentials (null for JWT), and granted authorities, then set in the `SecurityContextHolder`
      - Three key methods: `getPrincipal()` returns the user object (e.g., `UserDetails` or username string), `getCredentials()` returns the password or token string, and `getAuthorities()` returns the collection of `GrantedAuthority` objects. The `isAuthenticated()` flag starts as `false` during authentication and is set to `true` by the `AuthenticationManager` after successful verification

7. What is `GrantedAuthority`?
   - **Answer:**
      - `GrantedAuthority` represents a permission granted to the user, like a role or a specific action permission
      - Database roles like `ADMIN`, `PARTNER`, and `VIEWER` can be mapped to `SimpleGrantedAuthority` objects in the `UserDetails` implementation for authorization checks
      - Role-based authorities follow the convention `ROLE_ADMIN`, `ROLE_PARTNER` and are checked with `hasRole()`, while permission-based authorities use custom strings like `REPORT_EXPORT` or `USER_DELETE` and are checked with `hasAuthority()`. `hasRole('ADMIN')` can be used for broad admin-only endpoints and `hasAuthority('REPORT_EXPORT')` for fine-grained permission checks on specific actions that require explicit permission

8. What is `UserDetails`?
   - **Answer:**
      - `UserDetails` is Spring Security's interface representing the user principal with username, password, authorities, and account status flags
      - A custom `UserDetails` implementation can wrap a JPA `User` entity and delegate to it for authentication and authorization
      - The interface defines `getAuthorities()` (returns `Collection<GrantedAuthority>`), `isAccountNonExpired()` (account not past expiration), `isAccountNonLocked()` (account not locked), `isCredentialsNonExpired()` (password not past expiration), and `isEnabled()` (account active). The account locking methods can be used to implement a failed-login lockout — after 5 consecutive failed attempts `enabled` can be set to `false` on the `UserDetails` object, and unlocked after a 15-minute cooldown

9. What is `UserDetailsService`?
   - **Answer:**
      - `UserDetailsService` is a core interface that loads user-specific data by username
      - A typical implementation queries the database via a JPA repository, maps the user entity to `UserDetails`, and returns it for authentication
      - The single method `loadUserByUsername(String username)` must return a fully populated `UserDetails` or throw `UsernameNotFoundException`. User details can be cached in Redis with a 5-minute TTL to avoid querying the database on every login request, reducing database load while still allowing role changes to propagate within minutes

10. What is `AuthenticationProvider`?
   - **Answer:**
      - `AuthenticationProvider` processes a specific type of authentication request
      - In a JWT-based application, the `DaoAuthenticationProvider` handles username/password login, and a custom `JwtAuthenticationProvider` can validate the JWT token and return the `Authentication` object
      - To implement a custom provider, override `authenticate(Authentication auth)` which contains the authentication logic and `supports(Class<?> clazz)` which checks if this provider handles the given authentication type. Multiple providers can be registered using `AuthenticationManagerBuilder` — `DaoAuthenticationProvider` for form login and a custom `JwtAuthenticationProvider` for API token validation, so Spring Security delegates to the correct provider based on the authentication token type

11. What is `PasswordEncoder`?
   - **Answer:**
      - `PasswordEncoder` handles password hashing and verification
      - `BCryptPasswordEncoder` is commonly used for hashing user passwords before storing them in the database, and Spring Security uses the same encoder to verify passwords during login authentication
      - Three common implementations: `BCryptPasswordEncoder` (adaptive hashing with built-in salt, preferred for most apps), `SCryptPasswordEncoder` (memory-hard function, better against GPU attacks), and `Pbkdf2PasswordEncoder` (NIST-recommended, standard-based). BCrypt is popular for its adaptive salt and configurable work factor, and the strength parameter (cost factor) can be configured to 12 in the security config, balancing security with response time — each login hash takes under 1 second

12. Why use BCrypt?
   - **Answer:**
      - BCrypt is a slow, salted hashing algorithm designed specifically for passwords
      - Using `BCryptPasswordEncoder` with strength 12 makes brute-force attacks impractical — even if the database is compromised, the attacker cannot reverse the hashes easily
      - BCrypt embeds the salt directly in the hash output (the 22-character string after the `$2a$12$` prefix), so no separate salt storage is needed. The strength factor controls the number of rounds as `2^strength`, so increasing from 10 to 12 quadruples computation time. Strength 12 is a common choice based on profiling: under 1 second per hash on production hardware, ensuring login latency stays acceptable while making offline brute-force attacks computationally infeasible

13. What is JWT?
   - **Answer:**
      - JWT is a JSON-based token used for stateless authentication consisting of a header, payload, and signature
      - After login the server returns a signed JWT containing the user ID, roles, and expiration time, which the client sends in the `Authorization` header for subsequent requests
      - A JWT has three Base64URL-encoded parts separated by dots: the header specifies the algorithm (e.g., `HS256`) and token type, the payload contains claims (user data, expiration, issuer), and the signature is created by hashing `header + payload` with a secret key using the algorithm specified in the header. The `io.jsonwebtoken` (jjwt) library can be used to create tokens with `Jwts.builder().signWith(key).compact()` and parse them with `Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token)`

14. How does JWT authentication work?
   - **Answer:**
      - The client sends credentials to a login endpoint; the server validates them, creates a signed JWT, and returns it
      - A typical login endpoint accepts username/password, authenticates via `AuthenticationManager`, generates a JWT with 24-hour expiry, and returns it in the response body
      - For subsequent requests the full flow is: client sends `Authorization: Bearer <token>` header → `JwtAuthenticationFilter` extracts the token string → validates the signature using the signing key, checks expiry (`exp` claim), and verifies the issuer (`iss` claim) → parses claims into a `Claims` object → builds a `UsernamePasswordAuthenticationToken` with the subject as principal and roles from the `roles` claim → sets it in `SecurityContextHolder`. If any step fails, the filter throws `JwtException` and the request never reaches the controller

15. What should a JWT contain?
   - **Answer:**
      - A JWT should contain minimal claims: user ID, roles/permissions, issued-at time, and expiration time
      - Common claims include `sub` (user ID), `roles` (comma-separated roles), `iat` (issued at), `exp` (expiration), and `iss` (issuer = application name)
      - Standard registered claims include `iss` (issuer), `sub` (subject/user ID), `aud` (audience), `exp` (expiration), `nbf` (not-before), `iat` (issued-at), and `jti` (unique token ID for revocation). Custom claims should be minimal — keeping the payload small reduces HTTP header overhead, since large JWTs increase latency on every request

16. What should a JWT not contain?
   - **Answer:**
      - A JWT should never contain sensitive information like passwords, credit card numbers, or personally identifiable information (PII) because the payload is base64-encoded, not encrypted
      - Only non-sensitive data like user ID and roles should be stored — never the password hash or personal details
      - Anyone with the token can decode the Base64URL payload and read all claims without needing the signing key — the signature only prevents tampering, not reading. If sensitive claims are required, use JWE (JWT Encryption) with `Jwts.builder().encryptWith(key, algorithm)` to produce an encrypted token where the payload is ciphertext. Token size should also be kept reasonable since large payloads cause HTTP header overhead, and very long tokens can hit URL length limits if passed as query parameters

17. How do you validate JWT signature?
   - **Answer:**
      - The server validates the signature using the same secret key that signed the token
      - The jjwt library's `Jwts.parserBuilder().setSigningKey(secretKey).build().parseClaimsJws(token)` automatically validates the signature, expiry, and throws exceptions for tampered or expired tokens
      - HMAC signing uses a shared secret (same key signs and verifies — suitable for single-server or trusted environments), while RSA signing uses a public/private key pair (private key signs, public key verifies — better for distributed systems where multiple services validate tokens). The signing secret should be stored in environment variables via `@Value("${jwt.secret}")` rather than hardcoding it in source, and edge cases should be handled by catching specific exceptions: `SignatureException` for tampered tokens, `ExpiredJwtException` for expired tokens, `MalformedJwtException` for malformed structure, and `UnsupportedJwtException` for unsupported algorithms, returning a 401 JSON response for each

18. Difference between access token and refresh token.
   - **Answer:**
      - Access tokens are short-lived (15-30 minutes) and used to authenticate API requests; refresh tokens are long-lived (7-30 days) and used to obtain new access tokens without re-login
      - A common pattern is refresh token rotation where each refresh request invalidates the old refresh token and issues a new pair
      - The security trade-off is that longer access tokens increase the vulnerability window if leaked (an attacker has more time to use them), while refresh tokens reduce login frequency but require secure storage. Refresh tokens can be stored in a database with expiry dates and a `revoked` boolean flag, enabling server-side revocation even though the access token itself is stateless

19. How do you revoke JWT?
   - **Answer:**
      - Stateless JWTs cannot be revoked directly — they remain valid until expiry
      - A Redis blacklist of revoked JWT IDs (`jti`) can be maintained until their natural expiry, with a filter check at the beginning of the security chain to reject blacklisted tokens
      - Alternative approaches: short token expiry (5-15 minutes) reduces the revocation window so even an unrevoked token expires quickly, refresh token revocation invalidates the ability to obtain new access tokens (effective for logout), and maintaining a deny-list in Redis is the most practical approach with minimal performance impact — storing only the `jti` claim (not the full token) with a TTL matching the token's remaining expiry, so Redis automatically cleans up expired entries

20. What is token rotation?
   - **Answer:**
      - Token rotation issues a new refresh token each time a refresh token is used, invalidating the old one
      - When a client calls the `/auth/refresh` endpoint, the current refresh token is validated, revoked in the database, and a new access token and refresh token pair is issued
      - Token rotation limits the damage if a refresh token is stolen — the attacker can use it only once before the legitimate user's next refresh attempt invalidates it, since both tokens cannot be valid simultaneously. Using `@Transactional` on the rotation method ensures the old token revocation and new token issuance are atomic, preventing a race condition where both old and new tokens could be valid

21. Where should JWT be stored?
   - **Answer:**
      - On the client side, JWT should be stored in an httpOnly secure cookie or in-memory variable, not in localStorage
      - A common approach is storing the access token in memory and the refresh token in an httpOnly cookie to mitigate XSS attacks
      - `httpOnly` cookies are inaccessible to JavaScript (preventing XSS token theft) but are sent automatically with every request to the domain (requiring CSRF protection). In-memory storage (`useState` or a module-level variable) provides the strongest XSS protection but the token is lost on page refresh — the refresh token in the httpOnly cookie then obtains a new access token transparently. The trade-off is that in-memory storage requires the refresh flow to be implemented properly, but it eliminates the XSS attack vector entirely

22. Why is storing JWT in localStorage risky?
   - **Answer:**
      - localStorage is accessible by any JavaScript running on the same origin, making it vulnerable to XSS attacks
      - If an attacker injects a script, they can steal the token and impersonate the user
      - httpOnly cookies are the recommended alternative
      - The key distinction is CSRF vs XSS: httpOnly cookies prevent XSS (JavaScript cannot read the cookie) but are vulnerable to CSRF (automatically sent with requests, so a malicious site can trigger requests that include the cookie). Bearer tokens in localStorage prevent CSRF (JavaScript must explicitly attach the header, and a malicious origin cannot read it via CORS) but are vulnerable to XSS. For APIs that use Bearer tokens exclusively, CSRF can be disabled (`.csrf().disable()`) on the backend while the frontend uses httpOnly cookies for the refresh token and keeps the access token in memory only

23. What is CSRF?
   - **Answer:**
      - CSRF (Cross-Site Request Forgery) tricks an authenticated user into executing unwanted actions on a web application
      - In REST API projects using JWT, CSRF protection is typically disabled because JWT-based stateless APIs are not vulnerable to CSRF — there is no session cookie to exploit
      - CSRF works with session cookies because the browser automatically attaches the session cookie to every request to the domain, so a malicious site can create a hidden form that submits to the target app and the browser includes the session cookie without the user's knowledge. Stateless APIs using `Authorization: Bearer` headers are immune because the attacker's site cannot read the JWT from a different origin (CORS blocks it), and JavaScript on the attacker's site cannot set the `Authorization` header on cross-origin requests via the Fetch API

24. When can CSRF be disabled?
   - **Answer:**
      - CSRF can be disabled when using stateless authentication (JWT), when all state-changing endpoints require a custom header, or when the client and API are on different origins with proper CORS
      - CSRF is commonly disabled when using Bearer tokens exclusively
      - Disabling CSRF without understanding the security implications is dangerous — it should always be verified that no session-cookie-based authentication is in use before setting `.csrf().disable()` in the security configuration. CSRF should remain enabled if the app uses session cookies for login (form-based auth), accepts form submissions, or if the frontend sends cookies automatically. For JWT APIs, the `Authorization` header is never auto-attached by browsers, making CSRF protection unnecessary

25. What is CORS?
   - **Answer:**
      - CORS (Cross-Origin Resource Sharing) is a browser security mechanism that controls which origins can access a web resource
      - CORS must be configured in Spring Security to allow a frontend hosted on a different domain to make API calls
      - When a frontend at `https://app.example.com` calls an API at `https://api.example.com`, the browser sends a preflight `OPTIONS` request to check if the target origin allows the request. The server responds with `Access-Control-Allow-Origin` (which origins are allowed), `Access-Control-Allow-Methods` (which HTTP methods are permitted), and `Access-Control-Allow-Headers` (which headers are accepted). Without proper CORS configuration, the browser blocks the response and the request fails client-side

26. How do you configure CORS?
   - **Answer:**
      - CORS can be configured by defining a `CorsConfigurationSource` bean with allowed origins, methods, and headers
      - A typical setup allows a frontend origin and exposes custom headers like `X-Total-Count` for paginated responses
      - Controller-level `@CrossOrigin` applies only to that specific controller and must be repeated on each one, while global CORS configuration via a `CorsConfigurationSource` bean applies to all endpoints consistently. Setting `allowedOrigins("https://app.example.com")` (not `*` in production), `allowedMethods("GET", "POST", "PUT", "DELETE")`, `allowedHeaders("*")`, and `allowCredentials(true)` when the frontend needs to send cookies alongside the JWT in httpOnly cookies provides a complete CORS setup

27. Difference between `hasRole()` and `hasAuthority()`.
   - **Answer:**
      - `hasRole('ADMIN')` automatically prefixes with `ROLE_` to check `ROLE_ADMIN`, while `hasAuthority('REPORT_EXPORT')` checks the exact authority string
      - `hasRole()` is typically used for role-based endpoints and `hasAuthority('REPORT_EXPORT')` for specific permissions that only certain roles possess
      - Spring Security automatically adds the `ROLE_` prefix when evaluating `hasRole()`, so storing the role as `ROLE_ADMIN` in the database and using `hasRole('ADMIN')` both resolve to the same authority. The prefix can be customized or removed with `roleHierarchy()`. Use `hasRole()` for role-based access control (coarse-grained: admin, user, manager) and `hasAuthority()` for permission-based access control (fine-grained: `REPORT_EXPORT`, `USER_DELETE`, `INVENTORY_RESET`) in multi-tenant or complex authorization systems

28. What is method-level security?
   - **Answer:**
      - Method-level security applies access control at the service or controller method level using `@PreAuthorize`, `@PostAuthorize`, `@Secured`, or `@RolesAllowed`
      - For example, `@PreAuthorize("hasRole('ADMIN')")` can be placed on a sensitive service method so only admin users can invoke it
      - Enable it with `@EnableMethodSecurity` on a configuration class. The expression language supports SpEL (Spring Expression Language) — expressions like `@PreAuthorize("hasRole('ADMIN') and #partnerId == authentication.principal.id")` enable fine-grained access control where only an admin managing a specific resource can access it, combining role checks with entity-level ownership validation

29. What is `@PreAuthorize`?
   - **Answer:**
      - `@PreAuthorize` evaluates an access control expression before the method executes
      - For example, `@PreAuthorize("hasRole('ADMIN')")` on a report generation service ensures only admin users can generate and download reports, with the security check happening before any business logic
      - SpEL context variables available include `authentication` (the current `Authentication` object), `principal` (shortcut to `authentication.getPrincipal()`), and method arguments referenced via `#paramName` (e.g., `#userId`, `#orderId`). Multiple conditions can be combined with logical operators for complex authorization rules: `@PreAuthorize("hasRole('ADMIN') or (#orderId != null and @orderService.isOwner(#orderId, authentication.principal.id))")`

30. What is `@PostAuthorize`?
   - **Answer:**
      - `@PostAuthorize` evaluates an access control expression after the method returns, allowing access decisions based on the returned object
      - For example, `@PostAuthorize("returnObject.partnerId == authentication.principal.partnerId")` can enforce that a user can only view their own resource details
      - The `returnObject` SpEL variable refers to the method's return value, enabling authorization based on the actual data being returned. Performance consideration: `@PostAuthorize` still executes the full method even if the eventual access is denied (the result is discarded and an `AccessDeniedException` is thrown), unlike `@PreAuthorize` which blocks execution before any business logic runs — so use `@PostAuthorize` only when the authorization decision requires the method's output

31. What is OAuth2?
   - **Answer:**
      - OAuth2 is an authorization framework where third-party applications get limited access to resources without sharing credentials
      - OAuth2 grant types include authorization code, client credentials, and refresh token
      - Four roles exist: resource owner (the user), client (the app requesting access), authorization server (issues tokens), and resource server (hosts protected resources). The authorization code flow with PKCE is used for public clients (mobile apps, SPAs) — the client gets an authorization code, exchanges it for tokens at the token endpoint, and PKCE prevents authorization code interception attacks. Spring Security 5 supports OAuth2 resource server configuration via `oauth2ResourceServer()` DSL with either JWT validation or opaque token introspection

32. Difference between OAuth2 and JWT.
   - **Answer:**
      - OAuth2 is a protocol framework for authorization; JWT is a token format
      - OAuth2 can use JWT as the token format, but JWT can also be used independently
      - When JWT is used directly without OAuth2, the application both issues and validates tokens for its own APIs
      - OAuth2 describes how tokens are obtained (via grant types like authorization code or client credentials), how they are refreshed (via refresh tokens), and how they expire (via `expires_in`). JWT defines the token structure (header, payload, signature), verification mechanism (signature validation), and self-contained claims. OAuth2 with JWT is common in microservices where a central authorization server issues JWTs and each service validates them independently using the public key, eliminating the need for token introspection calls

33. What is OpenID Connect?
   - **Answer:**
      - OpenID Connect (OIDC) is an identity layer on top of OAuth2 that adds authentication
      - It returns an ID token (JWT) containing user identity information
      - OIDC solves the authentication gap in OAuth2 by standardizing how the client verifies the user's identity
      - OIDC defines three tokens: the ID token (JWT with user identity claims like `name`, `email`, `sub`), the access token (for API access, same as OAuth2), and the refresh token (for obtaining new access tokens). The `spring-security-oauth2-client` module simplifies integration with OIDC providers — a `ClientRegistrationRepository` can be configured with the provider's details (Google, GitHub, Azure AD), and Spring Security handles the redirect flow, code exchange, token parsing, and user principal extraction automatically

34. What is resource server?
   - **Answer:**
      - A resource server hosts protected resources and validates access tokens
      - In a typical Spring Boot setup, API endpoints act as resource servers — they receive JWTs from clients, validate the signature and claims, and serve data only if the token is valid and has sufficient permissions
      - Spring Security's `oauth2ResourceServer()` DSL configures JWT validation with either `jwkSetUri()` (fetches public keys from a remote JWKS endpoint) or `decoder()` (uses a local `JwtDecoder` bean). A resource server can be configured to use a local signing key for token validation instead of a remote JWKS endpoint, avoiding the network call to a JWKS endpoint on every request

35. What is authorization server?
   - **Answer:**
      - An authorization server issues access tokens after successful authentication
      - A single Spring Boot application can act as both authorization server (login endpoint) and resource server (API endpoints) — the login endpoint issues JWTs, and the API endpoints validate them
      - In production microservices, the authorization server should be separated from resource servers for security and scalability — a compromised resource server should not be able to issue new tokens. Dedicated solutions include Spring Authorization Server (lightweight, Spring-native), Keycloak (full-featured with admin UI, user federation, and social login), or AWS Cognito (managed service with user pools). Spring Authorization Server implements the OAuth2 and OIDC specifications and can be configured with `@EnableAuthorizationServer` to handle token issuance, revocation, and client registration

36. How do you secure actuator endpoints?
   - **Answer:**
      - Actuator endpoints are secured by exposing only safe endpoints publicly and applying role-based access to sensitive ones
      - A common setup exposes `/actuator/health` and `/actuator/info` without authentication, while `/actuator/env` and `/actuator/loggers` require ADMIN role
      - A dedicated security filter chain can be configured for the actuator base path (`requestMatcher("/actuator/**")`) with its own authorization rules, separate from the main API filter chain. `management.endpoints.web.exposure.include` in `application.yml` controls which endpoints are exposed over HTTP (typically only `health`, `info`, `metrics`, and `prometheus`). Adding `hasIpAddress("10.0.0.0/8")` restricts sensitive actuator endpoints to internal network IPs, preventing external access even if authorization is bypassed

37. How do you handle unauthorized response?
   - **Answer:**
      - Unauthorized responses are customized using `AuthenticationEntryPoint` and `AccessDeniedHandler`
      - A custom `JwtAuthenticationEntryPoint` can return a JSON response with 401 status and error message instead of the default HTML login page
      - `AuthenticationEntryPoint` handles 401 (unauthenticated) — writing JSON directly using `response.getWriter().write("{\"error\": \"Unauthorized\", \"message\": \"Invalid or missing JWT token\"}")` with `response.setContentType("application/json")` and `response.setStatus(HttpServletResponse.SC_UNAUTHORIZED)`. `AccessDeniedHandler` handles 403 (authenticated but forbidden). A correlation ID (`X-Request-Id` header) can be included in all error responses for debugging — this ID is generated by a `OncePerRequestFilter` at the start of the chain and logged with the full exception details

38. Difference between 401 and 403.
   - **Answer:**
      - 401 Unauthorized means the user is not authenticated (no valid JWT)
      - 403 Forbidden means the user is authenticated but lacks permission
      - A missing or expired JWT returns 401, while an authenticated user trying to access restricted endpoints returns 403
      - In the filter chain, `ExceptionTranslationFilter` catches `AuthenticationException` (thrown when no valid authentication exists) and delegates to `AuthenticationEntryPoint` (401), while `AccessDeniedException` (thrown when an authenticated user lacks required authority) is delegated to `AccessDeniedHandler` (403). The 403 response can be customized to include the required role information so the frontend can display a meaningful "insufficient permissions" message instead of a generic error

39. How do you implement RBAC?
   - **Answer:**
      - RBAC (Role-Based Access Control) assigns permissions to roles, and roles to users
      - Typical roles include `ADMIN`, `PARTNER`, and `VIEWER` stored in a relational database
      - The JWT contains the user's role, and `@PreAuthorize("hasRole('ADMIN')")` on sensitive endpoints enforces access
      - The database schema uses three tables: `users` (id, username, password), `roles` (id, name), and `user_roles` (user_id, role_id) as a many-to-many join table. Roles are loaded in `UserDetailsService.loadUserByUsername()` by joining `users` with `user_roles` and `roles`, then mapping each role to `SimpleGrantedAuthority("ROLE_" + role.getName())`. For endpoints accessible by multiple roles, `hasAnyRole('ADMIN', 'PARTNER')` can be used. A role hierarchy can also be implemented where `ADMIN` implicitly has all `PARTNER` permissions, configured via `RoleHierarchyImpl` with the expression `ROLE_ADMIN > ROLE_PARTNER`

40. How do you implement permission-based access?
   - **Answer:**
      - Permission-based access uses granular permissions like `REPORT_CREATE`, `REPORT_EXPORT`, `USER_DELETE` instead of broad roles
      - Permissions are stored in an `authorities` table and encoded as `SimpleGrantedAuthority` in the JWT, using `hasAuthority('REPORT_EXPORT')` in security checks
      - The database schema adds `permissions` (id, name) and `role_permissions` (role_id, permission_id) tables, so roles are assigned specific permissions rather than checking role names directly. Permissions are loaded by joining `roles` → `role_permissions` → `permissions` in the `UserDetailsService` and mapped to `SimpleGrantedAuthority(permission.getName())`. For complex checks involving entity ownership (e.g., "user can only export reports they own"), a custom `PermissionEvaluator` bean can be created and used with `@PreAuthorize("@permissionEvaluator.hasPermission(authentication, #reportId, 'REPORT', 'EXPORT')")` for SpEL-integrated authorization

41. How do you secure APIs behind a load balancer?
   - **Answer:**
      - When running behind a load balancer or proxy, Spring Security needs to trust forwarded headers to correctly validate requests
      - Configuring `server.forward-headers-strategy=NATIVE` and using `X-Forwarded-For`, `X-Forwarded-Proto`, and `X-Forwarded-Prefix` headers handles this
      - `ForwardedHeaderFilter` (Servlet) or `ForwardedHeaderWebFilter` (Reactive) processes `Forwarded` and `X-Forwarded-*` headers to reconstruct the original request URL, protocol, and client IP. `RemoteIpFilter` is an alternative that specifically handles `X-Forwarded-For` and `X-Forwarded-Proto` for IP-based access control. Trusted proxies should be configured in `application.yml` with `server.forward-headers-filter.trusted-proxies=10.0.0.0/8` to prevent header spoofing — misconfigured forwarded headers could allow an attacker to bypass IP whitelists by injecting a fake `X-Forwarded-For` header from a trusted IP

42. How do you handle `Authorization` header through proxies?
   - **Answer:**
      - Proxies and load balancers should preserve the `Authorization` header without modification
      - The load balancer must be configured to pass through the `Authorization` header by adding it to the whitelist of forwarded headers, ensuring the JWT reaches the Spring Boot application intact
      - Some proxies strip the `Authorization` header for security reasons (e.g., preventing credential forwarding to upstream services). Alternatives when headers are stripped: use client certificates (mutual TLS) for service-to-service auth, pass the token in a custom header like `X-Auth-Token` that the proxy does not strip, or use API keys as a fallback authentication mechanism. `HttpServletRequest` logging at the filter level (`request.getHeader("Authorization")`) can help debug header loss in production, which is useful for identifying when a proxy configuration change accidentally drops the header

43. What is session fixation?
   - **Answer:**
      - Session fixation is an attack where an attacker forces a user to use a session ID known to the attacker
      - In stateless JWT-based APIs, session fixation does not apply since there is no HTTP session
      - But in stateful apps, Spring Security prevents this by creating a new session on authentication
      - The attack works when a user clicks a link containing a pre-set session ID (e.g., `?JSESSIONID=attacker-controlled`), logs in, and the server associates the attacker's known session ID with the authenticated session — the attacker then uses that same session ID to access the user's account. Spring Security prevents this with `sessionManagement().sessionFixation().newSession()` (creates a new session after login) or `.migrateSession()` (copies attributes to a new session). `SessionCreationPolicy.STATELESS` (no session created at all) or `NEVER` (no session created but existing ones are used) are inherently immune to session fixation

44. What is stateless session management?
   - **Answer:**
      - Stateless session management means the server does not store any session state between requests
      - `SessionCreationPolicy.STATELESS` is commonly configured when using JWT tokens — the token itself carries all authentication information, eliminating server-side session storage
      - Four `SessionCreationPolicy` options: `ALWAYS` (creates a session for every request), `IF_REQUIRED` (creates only if needed — the default), `NEVER` (does not create but uses existing), and `STATELESS` (never creates or uses). Benefits of stateless APIs: horizontal scalability without sticky sessions, easier caching (no session affinity needed), simpler load balancer config, and no server-side memory overhead. The trade-off is that tokens cannot be invalidated server-side (requiring a blacklist/deny-list for revocation) and every request must carry the full token since there is no server-side session to reference

45. How do you mix stateful UI and stateless API security?
   - **Answer:**
      - In applications with a web UI and a backend API, the UI may use session-based authentication for web pages while the app uses JWT to call the backend APIs
      - The security configuration uses two filter chains: one with session-based auth for the UI routes and one with JWT for the API routes
      - Multiple `SecurityFilterChain` beans can be configured with `@Order` (lower order = higher priority) and `securityMatcher()` to route requests to the correct chain. The UI chain (`@Order(1)` with `securityMatcher("/ui/**")`) uses `HttpSessionSecurityContextRepository` for stateful session management with CSRF protection. The API chain (`@Order(2)` with `securityMatcher("/api/**")`) uses `BearerTokenServerSecurityContextRepository` for stateless JWT validation with CSRF disabled. Token refresh can be handled by intercepting 401 responses from the API, calling `/auth/refresh` with the httpOnly cookie, and retrying the original request with the new JWT
