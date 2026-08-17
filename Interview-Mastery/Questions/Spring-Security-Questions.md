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
      - In my CDMS and inventory projects, I used Spring Security with JWT to secure REST APIs, configured security filter chains, and applied role-based access control for admin and partner users
   - **If asked more:**
      - I would explain the filter chain architecture, how `SecurityFilterChain` replaced the old `WebSecurityConfigurerAdapter`, and walk through my custom `JwtAuthenticationFilter` that extends `OncePerRequestFilter`
2. Difference between authentication and authorization.
   - **Answer:**
      - Authentication verifies who you are (identity), authorization verifies what you can do (permissions)
      - In my CDMS project, users logged in with username/password (authentication), and the JWT contained roles like `ROLE_ADMIN` or `ROLE_PARTNER` (authorization) to control access to different API endpoints
   - **If asked more:**
      - I would explain how authentication creates a `UsernamePasswordAuthenticationToken`, stores it in `SecurityContextHolder`, and how authorization checks `GrantedAuthority` via `@PreAuthorize` or `.hasRole()` in filter chain configuration
3. What is security filter chain?
   - **Answer:**
      - The security filter chain is a sequence of filters that intercept every HTTP request to apply authentication, authorization, CSRF, CORS, and other security logic
      - In my inventory project, I configured a custom filter chain with `JwtAuthenticationFilter` before `UsernamePasswordAuthenticationFilter` to validate JWT tokens on every API call
   - **If asked more:**
      - I would explain how the filter chain is ordered, how `SecurityFilterChain` beans are matched by request matchers, and how I used multiple filter chains for public and private endpoints in the CDMS API
4. How does Spring Security process a request?
   - **Answer:**
      - Each request passes through the security filter chain
      - If a JWT is present, my custom filter extracts the token, validates the signature, loads user details, and creates an `Authentication` object stored in `SecurityContextHolder`
      - If JWT is missing or invalid, the request is rejected with 401 before reaching the controller
   - **If asked more:**
      - I would walk through the full flow: incoming request → `DelegatingFilterProxy` → `FilterChainProxy` → custom filters → `UsernamePasswordAuthenticationFilter` (for login) → `ExceptionTranslationFilter` → `FilterSecurityInterceptor`
5. What is `SecurityContextHolder`?
   - **Answer:**
      - `SecurityContextHolder` stores the `SecurityContext` of the currently authenticated user using a `ThreadLocal`
      - In my CDMS project, I accessed the logged-in user's details in service layers by calling `SecurityContextHolder.getContext().getAuthentication()` to retrieve the username and roles for audit logging
   - **If asked more:**
      - I would explain the three storage modes: MODE_THREADLOCAL (default), MODE_INHERITABLETHREADLOCAL (for async), and MODE_GLOBAL
      - I encountered a context loss issue with `@Async` and had to use `SecurityContextHolder.setStrategyName()` to fix it
6. What is `Authentication` object?
   - **Answer:**
      - `Authentication` represents the authenticated user's identity and authorities
      - After successful JWT validation in my inventory API, I created a `UsernamePasswordAuthenticationToken` containing the user principal, credentials (null for JWT), and granted authorities, and set it in the `SecurityContextHolder`
   - **If asked more:**
      - I would explain the three key methods: `getPrincipal()` (user object), `getCredentials()` (password/token), `getAuthorities()` (roles/permissions), and how `isAuthenticated()` flag is set by the `AuthenticationManager`
7. What is `GrantedAuthority`?
   - **Answer:**
      - `GrantedAuthority` represents a permission granted to the user, like a role or a specific action permission
      - In my CDMS project, I mapped database roles like `ADMIN`, `PARTNER`, and `VIEWER` to `SimpleGrantedAuthority` objects in the `UserDetails` implementation for authorization checks
   - **If asked more:**
      - I would explain the difference between role-based (`ROLE_ADMIN`) and permission-based (`REPORT_EXPORT`) authorities, and how I used `hasRole('ADMIN')` vs `hasAuthority('REPORT_EXPORT')` in method security annotations
8. What is `UserDetails`?
   - **Answer:**
      - `UserDetails` is Spring Security's interface representing the user principal with username, password, authorities, and account status flags
      - I implemented a custom `UserDetails` class in my inventory project that wrapped the JPA `User` entity and delegated to it for authentication and authorization
   - **If asked more:**
      - I would explain the methods: `getAuthorities()`, `isAccountNonExpired()`, `isAccountNonLocked()`, `isCredentialsNonExpired()`, and `isEnabled()`, and how I used these for account locking after multiple failed login attempts
9. What is `UserDetailsService`?
   - **Answer:**
      - `UserDetailsService` is a core interface that loads user-specific data by username
      - In my CDMS project, I implemented `CustomUserDetailsService` that queried the MSSQL `users` table via JPA repository, mapped the user entity to `UserDetails`, and returned it for authentication
   - **If asked more:**
      - I would explain the `loadUserByUsername()` method contract, how to handle `UsernameNotFoundException`, and how I cached user details in Redis to avoid querying the database on every request in the inventory project
10. What is `AuthenticationProvider`?
   - **Answer:**
      - `AuthenticationProvider` processes a specific type of authentication request
      - In my JWT-based projects, the `DaoAuthenticationProvider` handled username/password login, and I created a custom `JwtAuthenticationProvider` that validated the JWT token and returned the `Authentication` object
   - **If asked more:**
      - I would explain how to implement a custom `AuthenticationProvider` by overriding `authenticate()` and `supports()`, and how I used `AuthenticationManagerBuilder` to register multiple providers for different authentication types
11. What is `PasswordEncoder`?
   - **Answer:**
      - `PasswordEncoder` handles password hashing and verification
      - In all my projects, I used `BCryptPasswordEncoder` for hashing user passwords before storing them in MSSQL, and Spring Security used the same encoder to verify passwords during login authentication
   - **If asked more:**
      - I would explain the difference between `BCryptPasswordEncoder`, `SCryptPasswordEncoder`, and `Pbkdf2PasswordEncoder`, why BCrypt is preferred for its adaptive salt and work factor, and how I configured the strength parameter in my security config
12. Why use BCrypt?
   - **Answer:**
      - BCrypt is a slow, salted hashing algorithm designed specifically for passwords
      - In my CDMS project, I used `BCryptPasswordEncoder` with strength 12 because it makes brute-force attacks impractical — even if the database is compromised, the attacker cannot reverse the hashes easily
   - **If asked more:**
      - I would explain how BCrypt embeds the salt in the hash output, how the strength factor increases computation time exponentially, and how I chose the strength based on response time requirements (under 1 second per login attempt is acceptable)
13. What is JWT?
   - **Answer:**
      - JWT is a JSON-based token used for stateless authentication consisting of a header, payload, and signature
      - In my CDMS project, after login the server returned a signed JWT containing the user ID, roles, and expiration time, which the client sent in the `Authorization` header for subsequent requests
   - **If asked more:**
      - I would explain the three parts of JWT (header with algorithm, payload with claims, signature), how HMAC-SHA256 signing works, and how I used the `io.jsonwebtoken` (jjwt) library to create and parse tokens in my projects
14. How does JWT authentication work?
   - **Answer:**
      - The client sends credentials to a login endpoint; the server validates them, creates a signed JWT, and returns it
      - In my inventory API, I had a `POST /auth/login` endpoint that accepted username/password, authenticated via `AuthenticationManager`, generated a JWT with 24-hour expiry, and returned it in the response body
   - **If asked more:**
      - I would explain the full request flow: client sends `Authorization: Bearer <token>` header → my `JwtAuthenticationFilter` extracts token → validates signature and expiry → parses claims → creates `Authentication` object → sets in `SecurityContextHolder`
15. What should a JWT contain?
   - **Answer:**
      - A JWT should contain minimal claims: user ID, roles/permissions, issued-at time, and expiration time
      - In my projects, I included `sub` (user ID), `roles` (comma-separated roles), `iat` (issued at), `exp` (expiration), and `iss` (issuer = application name)
   - **If asked more:**
      - I would explain the difference between standard registered claims (`iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`) and custom claims, and why I kept the payload small to reduce HTTP header overhead in the cold-chain project with frequent API calls
16. What should a JWT not contain?
   - **Answer:**
      - A JWT should never contain sensitive information like passwords, credit card numbers, or personally identifiable information (PII) because the payload is base64-encoded, not encrypted
      - In my projects, I only stored non-sensitive data like user ID and roles — never the password hash or personal details
   - **If asked more:**
      - I would explain that anyone with the token can decode the payload, and how to use JWE (JWT Encryption) if sensitive claims are needed
      - I would also mention that token size matters when hitting URL length limits or causing HTTP header overhead
17. How do you validate JWT signature?
   - **Answer:**
      - The server validates the signature using the same secret key that signed the token
      - In my CDMS project, I used the jjwt library's `Jwts.parserBuilder().setSigningKey(secretKey).build().parseClaimsJws(token)` which automatically validates the signature, expiry, and throws exceptions for tampered or expired tokens
   - **If asked more:**
      - I would explain HMAC vs RSA signing, how I stored the secret key in environment variables (not in code), and how I handled edge cases like malformed tokens, expired tokens, and tokens with wrong signature by catching `JwtException` and sending 401 responses
18. Difference between access token and refresh token.
   - **Answer:**
      - Access tokens are short-lived (15-30 minutes) and used to authenticate API requests; refresh tokens are long-lived (7-30 days) and used to obtain new access tokens without re-login
      - In my CDMS project, I implemented refresh token rotation where each refresh request invalidated the old refresh token and issued a new pair
   - **If asked more:**
      - I would explain the security trade-off: longer access tokens increase vulnerability if leaked, while refresh tokens reduce login frequency but need secure storage
      - I stored refresh tokens in MSSQL with expiry dates for revocation capability
19. How do you revoke JWT?
   - **Answer:**
      - Stateless JWTs cannot be revoked directly
      - In my inventory project, I maintained a Redis blacklist of revoked JWT IDs (`jti`) until their natural expiry, and added a filter check at the beginning of the security chain to reject blacklisted tokens
   - **If asked more:**
      - I would explain alternative approaches: short token expiry reduces revocation window, refresh token revocation invalidates the ability to get new access tokens, and maintaining a deny-list in Redis is the most practical approach with minimal performance impact
20. What is token rotation?
   - **Answer:**
      - Token rotation issues a new refresh token each time a refresh token is used, invalidating the old one
      - In my CDMS project, when a client called the `/auth/refresh` endpoint, I validated the current refresh token, revoked it in the database, and issued a new access token and refresh token pair
   - **If asked more:**
      - I would explain how token rotation limits the damage if a refresh token is stolen — the attacker can use it only once before the legitimate user's next refresh invalidates it
      - I used `@Transactional` to ensure the rotation was atomic
21. Where should JWT be stored?
   - **Answer:**
      - On the client side, JWT should be stored in an httpOnly secure cookie or in-memory variable, not in localStorage
      - In my CDMS project, the frontend React application stored the access token in memory and the refresh token in an httpOnly cookie to mitigate XSS attacks
   - **If asked more:**
      - I would explain the security implications: localStorage is accessible via JavaScript (XSS vulnerability), while httpOnly cookies are not
      - I would also discuss the trade-off of in-memory storage where page refresh loses the token, requiring refresh token flow
22. Why is storing JWT in localStorage risky?
   - **Answer:**
      - localStorage is accessible by any JavaScript running on the same origin, making it vulnerable to XSS attacks
      - If an attacker injects a script, they can steal the token and impersonate the user
      - In my projects, I recommended the client team use httpOnly cookies instead
   - **If asked more:**
      - I would explain CSRF vs XSS risks, how httpOnly cookies prevent XSS but need CSRF protection, and how I configured the backend with CSRF disabled (since my API used Bearer tokens) but advised the frontend team on proper cookie attributes
23. What is CSRF?
   - **Answer:**
      - CSRF (Cross-Site Request Forgery) tricks an authenticated user into executing unwanted actions on a web application
      - In my REST API projects, I disabled CSRF protection because JWT-based stateless APIs are not vulnerable to CSRF — there is no session cookie to exploit
   - **If asked more:**
      - I would explain how CSRF works with session cookies, why stateful form-based apps need CSRF tokens, and how stateless APIs using `Authorization: Bearer` header are immune to CSRF since the attacker's site cannot read the JWT from a different origin
24. When can CSRF be disabled?
   - **Answer:**
      - CSRF can be disabled when using stateless authentication (JWT), when all state-changing endpoints require a custom header, or when the client and API are on different origins with proper CORS
      - In my CDMS and inventory projects, I disabled CSRF since I used Bearer tokens exclusively
   - **If asked more:**
      - I would explain that disabling CSRF without understanding the security implications is dangerous
      - I would always verify that no session-cookie-based authentication is in use before setting `.csrf().disable()` in the security configuration
25. What is CORS?
   - **Answer:**
      - CORS (Cross-Origin Resource Sharing) is a browser security mechanism that controls which origins can access a web resource
      - In my CDMS project, I configured CORS in Spring Security to allow the React frontend hosted on a different domain to make API calls
   - **If asked more:**
      - I would explain the preflight OPTIONS request, how the `Access-Control-Allow-Origin` header works, and how I configured `allowedOrigins`, `allowedMethods`, and `allowedHeaders` in a `@Bean` CorsConfigurationSource for the inventory project
26. How do you configure CORS?
   - **Answer:**
      - I configure CORS by defining a `CorsConfigurationSource` bean with allowed origins, methods, and headers
      - In my cold-chain project, I allowed the React dashboard origin and exposed custom headers like `X-Total-Count` for paginated responses
   - **If asked more:**
      - I would explain the difference between controller-level `@CrossOrigin` and global CORS configuration, why global config is better for consistency, and how I used `setAllowCredentials(true)` when the frontend needed to send cookies along with JWT in httpOnly cookies
27. Difference between `hasRole()` and `hasAuthority()`.
   - **Answer:**
      - `hasRole('ADMIN')` automatically prefixes with `ROLE_` to check `ROLE_ADMIN`, while `hasAuthority('REPORT_EXPORT')` checks the exact authority string
      - In my CDMS project, I used `hasRole('PARTNER')` for partner endpoints and `hasAuthority('REPORT_EXPORT')` for a specific permission that only admins with export permission had
   - **If asked more:**
      - I would explain how Spring Security adds the `ROLE_` prefix with `roleHierarchy()`, how to customize the prefix, and when to use role-based vs permission-based access control in a multi-tenant inventory system
28. What is method-level security?
   - **Answer:**
      - Method-level security applies access control at the service or controller method level using `@PreAuthorize`, `@PostAuthorize`, `@Secured`, or `@RolesAllowed`
      - In my inventory project, I used `@PreAuthorize("hasRole('ADMIN')")` on the partner data reset method so only admin users could trigger it
   - **If asked more:**
      - I would explain `@EnableMethodSecurity` and the attribute-based expression language — I used SpEL expressions like `@PreAuthorize("hasRole('ADMIN') and #partnerId == authentication.principal.id")` for fine-grained access control
29. What is `@PreAuthorize`?
   - **Answer:**
      - `@PreAuthorize` evaluates an access control expression before the method executes
      - In my CDMS project, I used `@PreAuthorize("hasRole('ADMIN')")` on the report generation service to ensure only admin users could generate and download reports, with the security check happening before any business logic
   - **If asked more:**
      - I would explain the SpEL context variables available — `authentication`, `principal`, and method arguments using `#paramName` — and how I combined multiple conditions with logical operators for complex authorization rules in the inventory service
30. What is `@PostAuthorize`?
   - **Answer:**
      - `@PostAuthorize` evaluates an access control expression after the method returns, allowing access decisions based on the returned object
      - In my inventory project, I used `@PostAuthorize("returnObject.partnerId == authentication.principal.partnerId")` to enforce that a user can only view their own partner details
   - **If asked more:**
      - I would explain the performance considerations — `@PostAuthorize` still executes the method even if access is denied, unlike `@PreAuthorize` which blocks before execution
      - I would also explain the `returnObject` SpEL variable
31. What is OAuth2?
   - **Answer:**
      - OAuth2 is an authorization framework where third-party applications get limited access to resources without sharing credentials
      - In my projects, I didn't use OAuth2 directly — we used JWT with a custom authorization server
      - But I understand OAuth2 grant types: authorization code, client credentials, and refresh token
   - **If asked more:**
      - I would explain the roles (resource owner, client, authorization server, resource server), the authorization code flow with PKCE for public clients, and how Spring Security 5 supports OAuth2 resource server configuration with JWT or opaque tokens
32. Difference between OAuth2 and JWT.
   - **Answer:**
      - OAuth2 is a protocol framework for authorization; JWT is a token format
      - OAuth2 can use JWT as the token format, but JWT can also be used independently
      - In my projects, I used JWT directly without OAuth2 — the application both issued and validated tokens for its own APIs
   - **If asked more:**
      - I would explain that OAuth2 describes how tokens are obtained and refreshed, while JWT defines the token structure and verification mechanism
      - OAuth2 with JWT introspection is common in microservices, where a gateway validates tokens centrally
33. What is OpenID Connect?
   - **Answer:**
      - OpenID Connect (OIDC) is an identity layer on top of OAuth2 that adds authentication
      - It returns an ID token (JWT) containing user identity information
      - I didn't implement OIDC in my projects, but I understand it solves the authentication gap in OAuth2 by standardizing how the client verifies the user's identity
   - **If asked more:**
      - I would explain the ID token, access token, and refresh token roles in OIDC, and how `spring-security-oauth2-client` simplifies integration with providers like Google, GitHub, or Azure AD for login
34. What is resource server?
   - **Answer:**
      - A resource server hosts protected resources and validates access tokens
      - In my projects, all my Spring Boot APIs acted as resource servers — they received JWTs from clients, validated the signature and claims, and served data only if the token was valid and had sufficient permissions
   - **If asked more:**
      - I would explain how Spring Security's `oauth2ResourceServer()` DSL configures JWT validation with `jwkSetUri()` or `decoder()`, and how I configured my resource server to use a local signing key for token validation instead of a remote JWKS endpoint
35. What is authorization server?
   - **Answer:**
      - An authorization server issues access tokens after successful authentication
      - In my CDMS project, the same Spring Boot application acted as both authorization server (login endpoint) and resource server (API endpoints) — the login endpoint issued JWTs, and the API endpoints validated them
   - **If asked more:**
      - I would explain that in production microservices, the authorization server should be separate from resource servers
      - I would discuss Spring Authorization Server, Keycloak, or AWS Cognito as dedicated authorization server solutions
36. How do you secure actuator endpoints?
   - **Answer:**
      - I secure actuator endpoints by exposing only safe endpoints publicly and applying role-based access
      - In my inventory project, I exposed `/actuator/health` and `/actuator/info` without authentication, while `/actuator/env` and `/actuator/loggers` required ADMIN role
   - **If asked more:**
      - I would explain using a dedicated security filter chain for the actuator base path, configuring `management.endpoints.web.exposure.include` carefully, and how I used IP whitelist by adding `hasIpAddress()` constraint in the security rules
37. How do you handle unauthorized response?
   - **Answer:**
      - I customize the unauthorized response using `AuthenticationEntryPoint` and `AccessDeniedHandler`
      - In my CDMS project, I implemented a `JwtAuthenticationEntryPoint` that returned a JSON response with 401 status and error message instead of the default HTML login page
   - **If asked more:**
      - I would show my custom entry point implementation: `response.sendError()` vs writing JSON directly using `response.getWriter().write()`, and how I included a correlation ID in the error response for debugging
38. Difference between 401 and 403.
   - **Answer:**
      - 401 Unauthorized means the user is not authenticated (no valid JWT)
      - 403 Forbidden means the user is authenticated but lacks permission
      - In my inventory project, a missing or expired JWT returned 401, while a partner user trying to access admin endpoints returned 403
   - **If asked more:**
      - I would explain the filter chain flow: `ExceptionTranslationFilter` handles `AuthenticationException` (→401) and `AccessDeniedException` (→403 when authenticated)
      - I used `AccessDeniedHandler` to customize the 403 JSON response with the required role information
39. How do you implement RBAC?
   - **Answer:**
      - RBAC (Role-Based Access Control) assigns permissions to roles, and roles to users
      - In my CDMS project, I had roles like `ADMIN`, `PARTNER`, and `VIEWER` stored in MSSQL
      - The JWT contained the user's role, and `@PreAuthorize("hasRole('ADMIN')")` on sensitive endpoints enforced access
   - **If asked more:**
      - I would explain the database schema: `users`, `roles`, and `user_roles` tables, how I loaded roles via `UserDetailsService`, and how I used `hasAnyRole()` for endpoints accessible by multiple roles
      - I would also discuss role hierarchy for inheritance patterns
40. How do you implement permission-based access?
   - **Answer:**
      - Permission-based access uses granular permissions like `REPORT_CREATE`, `REPORT_EXPORT`, `USER_DELETE` instead of broad roles
      - In my inventory project, I stored permissions in the `authorities` table and encoded them as `SimpleGrantedAuthority` in the JWT, using `hasAuthority('REPORT_EXPORT')` in security checks
   - **If asked more:**
      - I would explain the difference between role-based and permission-based access, how to load permissions from the database, and how I created a custom `PermissionEvaluator` for complex permission checks involving entity ownership
41. How do you secure APIs behind a load balancer?
   - **Answer:**
      - When running behind a load balancer or proxy, Spring Security needs to trust forwarded headers to correctly validate requests
      - In my EC2-deployed projects behind an ALB, I configured `server.forward-headers-strategy=NATIVE` and used `X-Forwarded-For`, `X-Forwarded-Proto`, and `X-Forwarded-Prefix` headers
   - **If asked more:**
      - I would explain the `ForwardedHeaderFilter` and `RemoteIpFilter`, how to configure trusted proxies in `application.yml`, and the security implications of misconfigured forwarded headers that could bypass IP-based access controls
42. How do you handle `Authorization` header through proxies?
   - **Answer:**
      - Proxies and load balancers should preserve the `Authorization` header without modification
      - In my deployment, we configured the ALB to pass through the `Authorization` header by adding it to the whitelist of forwarded headers, ensuring the JWT reached my Spring Boot application intact
   - **If asked more:**
      - I would explain that some proxies strip the `Authorization` header for security reasons, and how to use client certificates or API keys as alternatives
      - I would also discuss configuring `HttpServletRequest` logging to debug header loss in production
43. What is session fixation?
   - **Answer:**
      - Session fixation is an attack where an attacker forces a user to use a session ID known to the attacker
      - In my stateless JWT-based APIs, session fixation did not apply since there was no HTTP session
      - But in stateful apps, Spring Security prevents this by creating a new session on authentication
   - **If asked more:**
      - I would explain `SessionCreationPolicy.IF_REQUIRED` vs `STATELESS`, how to configure session fixation protection with `sessionManagement().sessionFixation().newSession()`, and why stateless authentication is inherently immune to session fixation
44. What is stateless session management?
   - **Answer:**
      - Stateless session management means the server does not store any session state between requests
      - In my CDMS and inventory projects, I configured `SessionCreationPolicy.STATELESS` because I used JWT tokens — the token itself carried all authentication information, eliminating server-side session storage
   - **If asked more:**
      - I would explain the `SessionCreationPolicy` options (ALWAYS, IF_REQUIRED, NEVER, STATELESS), the benefits of stateless APIs (scalability, no sticky sessions, easier caching), and the trade-off of not being able to invalidate tokens server-side
45. How do you mix stateful UI and stateless API security?
   - **Answer:**
      - In projects like my cold-chain dashboard, the React UI used session-based authentication for the web pages, and the React app stored a JWT to call the backend APIs
      - The security configuration used two filter chains: one with session-based auth for the UI routes and one with JWT for the API routes
   - **If asked more:**
      - I would explain configuring multiple `SecurityFilterChain` beans with `@Order` and `securityMatcher`, how to use `HttpSessionSecurityContextRepository` for stateful parts, and how I managed token refresh in the React app when the JWT expired

