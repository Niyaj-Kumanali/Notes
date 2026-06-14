# Spring Security

---

## Overview

- **Definition:** Spring Security is the de-facto security framework for Spring applications. It provides **authentication** (who you are), **authorization** (what you can do), and **protection** against common attacks like CSRF, session fixation, clickjacking, and XSS.
- **Why It Exists:** To offer a comprehensive, configurable security model for Spring applications that handles authentication, authorization, and attack protection out of the box, reducing the risk of security vulnerabilities from custom implementations.
- **Key Concepts:**
  - **Authentication** — Verifies the identity of a user. Supports form login, HTTP Basic, OAuth2, JWT, LDAP, SAML, and Remember-Me across a wide range of authentication mechanisms.
  - **Authorization** — Controls access to resources based on roles, permissions, or custom rules. Supports both URL-based and method-level authorization.
  - **Protection** — Built-in defenses against CSRF, CORS misconfiguration, clickjacking, and header injection. These protections are enabled by default and can be fine-tuned.
  - **Integration** — Seamless integration with OAuth2, JWT, LDAP, SAML, and Spring's method-level security annotations like `@PreAuthorize` and `@PostAuthorize`.
  - **Security Filter Chain:** Spring Security is implemented as a chain of servlet filters. Each filter handles a specific security concern:

    ```
    Request → SecurityContextPersistenceFilter → LogoutFilter →
    UsernamePasswordAuthenticationFilter → BasicAuthenticationFilter →
    ExceptionTranslationFilter → FilterSecurityInterceptor → Controller
    ```

    Each filter in the chain either handles the request or passes it to the next filter. The order of filters matters — placing a filter in the wrong position can bypass security checks.

  - **Authentication Architecture:**

    ```
    AuthenticationProvider ─── UserDetailsService
           │                         │
           │                    ┌────┴────┐
           │                    │  User   │
           │                    │ Details │
           │                    └─────────┘
           ▼
    SecurityContextHolder (ThreadLocal)
           │
           ▼
    SecurityContext
        └─ Authentication (Principal + Credentials + GrantedAuthorities)
    ```

    - **`SecurityContextHolder`** — Stores the current security context in a `ThreadLocal`. Accessible anywhere in the application via `SecurityContextHolder.getContext()`.
    - **`SecurityContext`** — Holds the `Authentication` object representing the currently authenticated user and their granted authorities.
    - **`Authentication`** — Contains the principal (user details), credentials (password/token), and granted authorities (roles/permissions).
    - **`GrantedAuthority`** — Represents a permission or role (e.g., `ROLE_ADMIN`, `ORDER_WRITE`). Used by the authorization mechanism.
    - **`UserDetailsService`** — Loads user-specific data from a database or external system during authentication.
    - **`PasswordEncoder`** — Encodes passwords for storage and validates them during authentication. BCrypt is the recommended default.

---

## Core Concepts

### Security Configuration (JWT + Role-Based)

- Modern Spring Security configuration using the fluent API:

  ```java
  @Configuration
  @EnableWebSecurity
  @EnableMethodSecurity
  public class SecurityConfig {

      @Bean
      public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
          http
              .csrf(AbstractHttpConfigurer::disable)
              .sessionManagement(session -> session
                  .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
              .authorizeHttpRequests(auth -> auth
                  .requestMatchers("/api/v1/auth/**", "/actuator/health").permitAll()
                  .requestMatchers(HttpMethod.GET, "/api/v1/products/**")
                      .hasAnyRole("USER", "ADMIN")
                  .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                  .anyRequest().authenticated()
              )
              .authenticationProvider(authenticationProvider())
              .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

          return http.build();
      }

      @Bean
      public AuthenticationProvider authenticationProvider() {
          DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
          provider.setUserDetailsService(userDetailsService);
          provider.setPasswordEncoder(passwordEncoder());
          return provider;
      }

      @Bean
      public PasswordEncoder passwordEncoder() {
          return new BCryptPasswordEncoder();
      }
  }
  ```

### JWT Authentication Filter

- A custom filter that extracts and validates JWT tokens from the `Authorization` header:

  ```java
  @Component
  public class JwtAuthFilter extends OncePerRequestFilter {
      private final JwtService jwtService;
      private final UserDetailsService userDetailsService;

      @Override
      protected void doFilterInternal(HttpServletRequest request,
                                      HttpServletResponse response,
                                      FilterChain chain) throws IOException, ServletException {
          String authHeader = request.getHeader("Authorization");

          if (authHeader == null || !authHeader.startsWith("Bearer ")) {
              chain.doFilter(request, response);
              return;
          }

          try {
              String jwt = authHeader.substring(7);
              String userEmail = jwtService.extractUsername(jwt);

              if (userEmail != null && SecurityContextHolder.getContext()
                      .getAuthentication() == null) {
                  UserDetails user = userDetailsService.loadUserByUsername(userEmail);

                  if (jwtService.isTokenValid(jwt, user)) {
                      UsernamePasswordAuthenticationToken authToken =
                          new UsernamePasswordAuthenticationToken(
                              user, null, user.getAuthorities());
                      authToken.setDetails(
                          new WebAuthenticationDetailsSource().buildDetails(request));
                      SecurityContextHolder.getContext().setAuthentication(authToken);
                  }
              }
              chain.doFilter(request, response);
          } catch (JwtException e) {
              response.setStatus(HttpStatus.UNAUTHORIZED.value());
              response.getWriter().write("Invalid or expired token");
          }
      }
  }
  ```

### Method-Level Security

- Fine-grained authorization at the method level:

  ```java
  @RestController
  @RequestMapping("/api/v1/admin")
  public class AdminController {

      @GetMapping("/users")
      @PreAuthorize("hasRole('ADMIN')")
      public List<User> getAllUsers() { /* ... */ }

      @PostMapping("/promote")
      @PreAuthorize("hasAuthority('admin:write')")
      public void promoteUser(@RequestBody UserId userId) { /* ... */ }

      @DeleteMapping("/users/{id}")
      @PreAuthorize("hasRole('SUPER_ADMIN') or @securityService.canDelete(#id)")
      public void deleteUser(@PathVariable Long id) { /* ... */ }

      @GetMapping("/audit/{id}")
      @PostAuthorize("returnObject.owner == authentication.name")
      public AuditRecord getAudit(@PathVariable Long id) { /* ... */ }
  }
  ```

### UserDetailsService Implementation

- Loading users from a database:

  ```java
  @Service
  public class UserService implements UserDetailsService {
      private final UserRepository userRepository;

      @Override
      public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
          return userRepository.findByEmail(email)
              .map(UserPrincipal::new)
              .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));
      }

      public record UserPrincipal(User user) implements UserDetails {
          @Override
          public Collection<? extends GrantedAuthority> getAuthorities() {
              return user.getRoles().stream()
                  .flatMap(role -> role.getPermissions().stream())
                  .map(perm -> new SimpleGrantedAuthority(perm.getName()))
                  .toList();
          }

          @Override
          public String getPassword() { return user.getPassword(); }
          @Override
          public String getUsername() { return user.getEmail(); }
          @Override
          public boolean isEnabled() { return user.isActive(); }
          @Override public boolean isAccountNonExpired() { return true; }
          @Override public boolean isAccountNonLocked() { return !user.isLocked(); }
          @Override public boolean isCredentialsNonExpired() { return true; }
      }
  }
  ```

---

## Common Mistakes

- **Storing passwords in plain text** — If the database is breached, all passwords are exposed and can be used immediately.
  - Why it looks correct: The application works perfectly — users log in, authentication succeeds, and there are no errors. The security risk is invisible until the database is compromised.
  - Fix: Always use `BCryptPasswordEncoder` or stronger (Argon2, SCrypt) for password hashing with automatic salting.

- **Overly permissive CORS** — Allowing all origins (`*`) opens the door for XSS and data theft from any website.
  - Why it looks correct: During development the frontend on `localhost:3000` connects without errors — the wildcard CORS policy is convenient and the security risk only materializes in production.
  - Fix: Restrict to specific, known origins that are verified and documented.

- **Disabling CSRF for all endpoints without understanding the implications** — CSRF protection should only be disabled for stateless REST APIs using token-based authentication.
  - Why it looks correct: Disabling CSRF eliminates 403 errors during development and is widely documented as the right approach for REST APIs — but if the API also uses cookie-based auth for some endpoints, the application is vulnerable.
  - Fix: Keep it enabled for traditional form-based applications where session cookies are used.

- **Not validating JWT signature** — Without signature validation, anyone can forge tokens and impersonate any user.
  - Why it looks correct: The JWT lookups succeed and the user is authenticated — the only missing check is the signature, which is invisible unless someone tests with a tampered token.
  - Fix: Use a proper signing algorithm (HS256, RS256) with a secure secret key stored in a secrets manager.

- **Storing JWT in `localStorage`** — Vulnerable to XSS attacks — any injected script can steal the token.
  - Why it looks correct: `localStorage` is the simplest storage mechanism with a well-known API, and during testing with a single user, XSS never occurs.
  - Fix: Use HTTP-only cookies for storing tokens, or keep them in memory only with refresh token rotation.

- **Hard-coded roles in authorization expressions** — Makes permission changes require code deployment and application restart.
  - Why it looks correct: `@PreAuthorize("hasRole('ADMIN')")` is readable, concise, and directly shows the permission rule in the code — the deployment overhead of changing a role only becomes apparent after the 10th time an ops request requires a full release cycle.
  - Fix: Use configurable permissions or a database-backed authorization system that can be updated at runtime.

- **Not securing actuator endpoints** — Actuator endpoints expose sensitive information (heap dumps, env, loggers) that can reveal secrets and internal architecture.
  - Why it looks correct: Actuator endpoints return data in a clean JSON format and the application team uses them during development without issues — the data leakage only matters when an external attacker discovers the actuator URLs.
  - Fix: Always secure them with role-based access or restrict to internal networks.

---

## Real-World Scenarios

### Scenario 1: JWT Authentication for a Stateless REST API

- A mobile banking app needs a stateless authentication mechanism. Users log in once and receive a JWT access token (15 min) and a refresh token (7 days). The access token is sent on every request; the refresh token is used to get new access tokens without re-entering credentials.

  ```java
  @Component
  public class JwtAuthFilter extends OncePerRequestFilter {
      @Override
      protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                      FilterChain chain) throws IOException, ServletException {
          String authHeader = request.getHeader("Authorization");
          if (authHeader == null || !authHeader.startsWith("Bearer ")) {
              chain.doFilter(request, response);
              return;
          }
          try {
              String jwt = authHeader.substring(7);
              String username = jwtService.extractUsername(jwt);
              if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                  UserDetails user = userDetailsService.loadUserByUsername(username);
                  if (jwtService.isTokenValid(jwt, user)) {
                      var authToken = new UsernamePasswordAuthenticationToken(
                          user, null, user.getAuthorities());
                      authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                      SecurityContextHolder.getContext().setAuthentication(authToken);
                  }
              }
              chain.doFilter(request, response);
          } catch (JwtException e) {
              response.setStatus(401);
              response.getWriter().write("Invalid or expired token");
          }
      }
  }
  ```

### Scenario 2: Role-Based Access Control for an Admin Dashboard

- An e-commerce admin dashboard has three roles: `VIEWER` (can see reports), `OPERATOR` (can manage orders), and `ADMIN` (full access). Permissions must be configurable at runtime without redeployment.

  ```java
  @Configuration
  @EnableWebSecurity
  @EnableMethodSecurity
  public class SecurityConfig {
      @Bean
      public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
          http.authorizeHttpRequests(auth -> auth
              .requestMatchers("/api/v1/auth/**", "/actuator/health").permitAll()
              .requestMatchers("/api/v1/admin/reports/**").hasRole("VIEWER")
              .requestMatchers("/api/v1/admin/orders/**").hasAnyRole("OPERATOR", "ADMIN")
              .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
              .anyRequest().authenticated()
          ).sessionManagement(session ->
              session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
          return http.build();
      }
  }

  // Method-level: fine-grained control
  @RestController
  public class OrderController {
      @PreAuthorize("hasAuthority('order:delete')")
      @DeleteMapping("/api/v1/admin/orders/{id}")
      public void deleteOrder(@PathVariable Long id) { ... }
  }
  ```

### Scenario 3: OAuth2 Login for a B2B SaaS Platform

- A B2B SaaS platform allows enterprise customers to log in via their own SSO providers (Azure AD, Okta, Google Workspace). The application must accept JWTs issued by these external identity providers.

  ```java
  @Configuration
  public class OAuth2Config {
      @Bean
      public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
          http.oauth2Login(oauth -> oauth
              .loginPage("/oauth2/authorization/azure")
              .defaultSuccessUrl("/dashboard", true)
              .userInfoEndpoint(userInfo ->
                  userInfo.oidcUserService(this.oidcUserService()))
          ).oauth2ResourceServer(oauth2 -> oauth2
              .jwt(Customizer.withDefaults()));
          return http.build();
      }

      private OidcUserService oidcUserService() {
          return new OidcUserService();
      }
  }
  ```

---

## Use Cases

- Spring Security handles everything from basic form login to OAuth2 and fine-grained method security. These use cases address common authentication and authorization patterns in production.

- **JWT-based stateless authentication** — A mobile banking app needs a token-based auth mechanism where the server does not store session state.
  - Implement a `OncePerRequestFilter` that extracts the JWT from the `Authorization` header, validates it, and sets the `SecurityContext`. Use short-lived access tokens (15 min) with refresh tokens.
  - **Avoid when:** The client is a server-side web app with server-side rendering — session-based auth is simpler and more secure.

- **Role-based access control (RBAC) for admin UIs** — An admin dashboard has `VIEWER`, `OPERATOR`, and `ADMIN` roles with different endpoint access levels.
  - Configure `SecurityFilterChain` with `.requestMatchers()` for URL-level authorization and `@PreAuthorize` for method-level fine-grained control. Use `@EnableMethodSecurity` to activate annotations.
  - **Avoid when:** Permissions are purely role-based and map cleanly to URL patterns — URL-based authorization in the filter chain is sufficient and testable without loading the full context.

- **OAuth2 / OpenID Connect for enterprise SSO** — A B2B SaaS platform must allow customers to log in via their own identity providers (Azure AD, Okta, Google Workspace).
  - Use Spring Security's OAuth2 client support with `.oauth2Login()`. Configure `application.yml` with client registrations for each provider. The framework handles the redirect and token exchange.
  - **Avoid when:** You control both the client and server — a simpler password-less login flow (magic link, one-time code) may be better for user experience.

- **Method-level security for data-level permissions** — A hospital system must ensure doctors see only their own patients. URL-level authorization cannot express `patient.assignedDoctorId == currentUserId`.
  - Use `@PostAuthorize("returnObject.assignedDoctorId == authentication.principal.id")` on the repository or service method. For collections, use `@PostFilter`.
  - **Avoid when:** The permission check is expensive (database query) — consider a query-level filter (e.g., adding a `WHERE assigned_doctor_id = ?` clause) before applying post-filtering.

- **CSRF protection for state-changing endpoints** — A traditional server-rendered web app must prevent cross-site request forgery attacks on POST/PUT/DELETE endpoints.
  - CSRF protection is enabled by default in Spring Security. Render the CSRF token in forms using Spring's tag library or Thymeleaf's `th:action`. For REST APIs, disable CSRF and use JWT instead.
  - **Avoid when:** Building a stateless REST API (mobile app, SPA with token auth) — disable CSRF via `http.csrf(AbstractHttpConfigurer::disable)` since there is no session cookie to steal.

---

## Scenario-Based Questions

**Q: Your team builds a REST API for a hospital system. Doctors should only see their own patients' data. A `GET /api/patients/{id}` endpoint currently returns any patient if the user is authenticated. How do you enforce that a doctor can only access their own patients?**
- Use `@PostAuthorize` for method-level security after the database query:
  ```java
  @PostAuthorize("returnObject.physicianId == authentication.principal.id")
  public Patient getPatient(Long id) {
      return patientRepository.findById(id).orElseThrow();
  }
  ```
- For list endpoints, filter in the service layer using the authenticated user's ID:
  ```java
  @PreAuthorize("isAuthenticated()")
  public List<Patient> getMyPatients() {
      Long doctorId = ((UserPrincipal) SecurityContextHolder.getContext()
          .getAuthentication().getPrincipal()).getId();
      return patientRepository.findByPhysicianId(doctorId);
  }
  ```
- Never trust the client to send the doctor ID — always derive it from the security context.

**Q: Your JWT-based authentication works perfectly in development. In production behind a load balancer, every request returns 401. The JWT is valid. What changed?**
- The load balancer likely terminates TLS and forwards HTTP to the application. If your JWT filter checks the scheme (`request.isSecure()`) or the `X-Forwarded-Proto` header is not configured, the forward may break the token validation. Fix by: (a) configuring the proxy: `server.forward-headers-strategy=framework`, (b) ensuring `X-Forwarded-Proto`, `X-Forwarded-Host`, and `X-Forwarded-For` headers are set by the load balancer, (c) using `ForwardedHeaderFilter` if needed. Also check if the load balancer strips the `Authorization` header.
- **Interview follow-up:** The candidate configured `server.forward-headers-strategy=framework`. The fix works initially, but two weeks later the production deployment pipeline adds a CDN (CloudFront) in front of the load balancer. The CDN terminates TLS, sets `X-Forwarded-Proto: https`, but also injects its own `Authorization` header for CDN authentication, overwriting the client's original JWT. The application now receives the CDN's auth header instead of the client's token, and all requests return 401 again. How would you design the authentication architecture (considering filter ordering, header propagation, and token forwarding) so the application receives the client's original JWT despite CDN and load balancer intermediaries?

**Q: You add CSRF protection to your REST API, and now all POST/PUT/DELETE requests return 403. Your API is consumed by a React SPA. How do you fix this?**
- For stateless REST APIs using token-based auth (JWT, OAuth2), CSRF protection is unnecessary — there is no session cookie to exploit. Disable CSRF:
  ```java
  http.csrf(AbstractHttpConfigurer::disable);
  ```
- If you must keep CSRF (e.g., cookie-based auth), configure CSRF token repository to return tokens via a cookie that the SPA can read:
  ```java
  http.csrf(csrf -> csrf
      .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()));
  ```
- Then the SPA reads the XSRF-TOKEN cookie and sends it as the X-XSRF-TOKEN header.

**Q: You have a public endpoint `GET /api/products` that should be accessible without authentication. But users report getting redirected to a login page. What's wrong?**
- Your security configuration likely has `.anyRequest().authenticated()` without explicitly permitting the products endpoint:
  ```java
  http.authorizeHttpRequests(auth -> auth
      .requestMatchers("/api/products").permitAll()   // Must be BEFORE anyRequest
      .anyRequest().authenticated()
  );
  ```
- The order matters — `permitAll()` must come before `authenticated()`. Also check that the URL pattern matches exactly: `/api/products` does NOT match `/api/products/123`.

**Q: Your user management UI allows admins to change user roles. The changes should take effect immediately without requiring the user to log out or the admin to redeploy. How do you design this?**
- Store roles and permissions in the database (not in code). The `UserDetailsService` loads them on every authentication. For already-authenticated users, use one of these strategies: (a) Short JWT access token TTL (5-15 min) so permissions refresh quickly, (b) Implement a token revocation check — validate a token version stored in Redis/DB on every request, (c) Use session-based auth with a permission cache that has a short TTL:
  ```java
  @Service
  public class DynamicUserService implements UserDetailsService {
      @Override
      public UserDetails loadUserByUsername(String email) {
          User user = userRepository.findByEmail(email).orElseThrow();
          return new UserPrincipal(user); // Loads current roles from DB
      }
  }
  // In JwtAuthFilter, validate token version against DB on every request
  ```

**Q: Your `@PreAuthorize("hasRole('ADMIN')")` on a service method always returns false even though the authenticated user has the `ROLE_ADMIN` authority. You verified it's in the database. What's wrong?**
- Check that the granted authority is stored exactly as `ROLE_ADMIN` (with the prefix). `hasRole('ADMIN')` checks for `ROLE_ADMIN`. If your database stores just `ADMIN` and your `UserDetailsService` returns `new SimpleGrantedAuthority("ADMIN")`, the prefix is missing. Either: (a) prepend `ROLE_` in the `UserDetailsService`:
  ```java
  return user.getRoles().stream()
      .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
      .toList();
  ```
- Or (b) use `hasAuthority('ADMIN')` instead of `hasRole('ADMIN')`. Be consistent across the codebase.

**Q: A security audit reveals that your application stores passwords using `NoOpPasswordEncoder` in the test configuration. The test config is accidentally deployed to production in the same JAR. How do you prevent this?**
- Never use `NoOpPasswordEncoder` — even in tests. Use `BCryptPasswordEncoder` everywhere and test with pre-encoded passwords. Prevent misconfiguration by: (a) Removing the test config class from the production classpath (use separate test source roots), (b) Adding an integration test that fails if any `PasswordEncoder` bean is not an instance of `BCryptPasswordEncoder` or `Argon2PasswordEncoder`, (c) Using ArchUnit to enforce that `NoOpPasswordEncoder` is not referenced outside test packages.
- **Interview follow-up:** The candidate proposed using `BCryptPasswordEncoder` everywhere. In a legacy system with 100,000 existing users whose passwords are hashed with SHA-256, switching to BCrypt for new users means the application must support two password encoders simultaneously — BCrypt for new passwords and SHA-256 for legacy passwords. How would you implement dual password encoder support in Spring Security so that logging in with an old SHA-256 password triggers an automatic upgrade to BCrypt on successful authentication?

**Q: Your application uses both `@PreAuthorize` on service methods and `requestMatchers` in the security filter chain. Some endpoints are checked twice, some not at all. How do you design a clear authorization strategy?**
- Define a clear layering: Filter chain handles coarse-grained access (authenticated vs unauthenticated, role-based URL patterns). Method-level security handles fine-grained, resource-specific rules:
  - **Filter chain**: Check authentication status, block unauthenticated requests, enforce role-based URL patterns (e.g., `/api/admin/**` requires `ADMIN`).
  - **`@PreAuthorize`**: Check resource ownership, data-level permissions, complex SpEL conditions.
- Never check the same rule in both places. Document the strategy and enforce it with code reviews.

**Q: Your OAuth2 login works but the user info endpoint fails with 401 when the SPA tries to call it with the access token obtained from the OAuth2 flow. The same token works in Postman. What's different?**
- The SPA likely sends the token in a different way than Postman. Common issues: (a) The SPA sends the token in a cookie (from the OAuth2 redirect), but the API expects it in the `Authorization: Bearer` header. (b) CORS preflight (OPTIONS) request does not include the token, and the OPTIONS request fails. (c) The SPA uses a different token (ID token vs access token). Fix by explicitly setting the authorization header in the SPA and ensuring CORS allows the Authorization header.

**Q: Your security configuration sets `.sessionCreationPolicy(SessionCreationPolicy.STATELESS)` but some admin endpoints still need session-based auth for the admin UI (Thymeleaf pages). How do you mix stateless API and stateful UI?**
- Separate the configurations by request matcher:
  ```java
  @Configuration
  @Order(1)
  public static class ApiSecurityConfig {
      @Bean
      public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
          http.securityMatcher("/api/**")
              .sessionManagement(s ->
                  s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
              .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
          return http.build();
      }
  }

  @Configuration
  @Order(2)
  public static class WebSecurityConfig {
      @Bean
      public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
          http.securityMatcher("/admin/**")
              .formLogin(Customizer.withDefaults())
              .sessionManagement(s ->
                  s.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED));
          return http.build();
      }
  }
  ```
- Use `@Order` to ensure the API config is evaluated first (more specific matchers first).

---

## Interview Questions

- **What is Spring Security and what does it provide?**
  - Spring Security is the de-facto security framework for Spring applications. It provides authentication (who you are), authorization (what you can do), and protection against common attacks (CSRF, session fixation, clickjacking, XSS). It is implemented as a chain of servlet filters.

- **What is the Security Filter Chain?**
  - The Security Filter Chain is a series of servlet filters that each handle a specific security concern. Filters include authentication (e.g., `UsernamePasswordAuthenticationFilter`), authorization (`FilterSecurityInterceptor`), and exception handling (`ExceptionTranslationFilter`). The order of filters matters — each filter either handles the request or passes it to the next.

- **What is the difference between authentication and authorization?**
  - Authentication verifies identity — "who are you?" (e.g., validating username/password, JWT token). Authorization determines access — "what can you do?" (e.g., checking if a user has the `ADMIN` role). Authentication comes first, followed by authorization.

- **What is `SecurityContextHolder` and how does it work?**
  - `SecurityContextHolder` stores the security context in a `ThreadLocal`. It holds the `Authentication` object which contains the principal (user details), credentials, and granted authorities. It is accessible anywhere in the application via `SecurityContextHolder.getContext().getAuthentication()`. For async operations, the context must be explicitly propagated.

- **What is the difference between `hasRole()` and `hasAuthority()`?**
  - `hasRole('ADMIN')` automatically prepends `ROLE_` — it checks for `ROLE_ADMIN`. `hasAuthority('ROLE_ADMIN')` checks the exact string. Use `hasRole` for role-based checks and `hasAuthority` for granular permissions. This allows differentiating between role-based and permission-based security.

- **How does JWT authentication work in Spring Security?**
  - A custom filter (extending `OncePerRequestFilter`) extracts the JWT from the `Authorization: Bearer` header, validates the signature using a secret or public key, extracts the username, loads `UserDetails`, creates an `Authentication` token, and sets it in the `SecurityContextHolder`. The filter runs before `UsernamePasswordAuthenticationFilter`.

- **What is `@PreAuthorize` and `@PostAuthorize`?**
  - These annotations enable method-level security. `@PreAuthorize` checks authorization before method execution (e.g., `@PreAuthorize("hasRole('ADMIN')")`). `@PostAuthorize` checks after execution (e.g., `@PostAuthorize("returnObject.owner == authentication.name")`). Requires `@EnableMethodSecurity`.

- **What is CSRF and when should you disable it?**
  - Cross-Site Request Forgery (CSRF) tricks a user into making unwanted requests while authenticated. Spring Security enables CSRF by default. Disable it only for stateless REST APIs using token-based auth (JWT) where there is no session cookie to exploit. Keep it enabled for traditional form-based applications.

- **How do you store passwords securely?**
  - Use `BCryptPasswordEncoder` (default, good balance of security and performance), `Argon2PasswordEncoder` (stronger, memory-hard), or `SCryptPasswordEncoder`. Never use `NoOpPasswordEncoder` (plain text) or `StandardPasswordEncoder` (SHA-256, too fast for brute force). Always salt and hash passwords — never encrypt them.

- **What is `UserDetailsService` and how do you customize it?**
  - `UserDetailsService` is an interface with a single method `loadUserByUsername(String username)`. Implement it to load user data from a database, LDAP, or external API. Return a `UserDetails` object containing username, password, and granted authorities. Spring Security calls it during authentication.

---

## Developer Recommendations

- **Use `BCryptPasswordEncoder` over `NoOpPasswordEncoder` or custom password hashing** — BCrypt is intentionally slow (configurable strength), includes a random salt, and resists rainbow table attacks. Rolling your own password hashing or using plain text is a critical security vulnerability. Never use `NoOpPasswordEncoder` anywhere, even in tests.
- **Use method-level security (`@PreAuthorize`) for data-level access control** — Filter-chain security (`requestMatchers`) handles coarse access (URL patterns). `@PreAuthorize` with SpEL handles fine-grained rules like "user can only edit their own orders". Combining both gives defense in depth.
- **Disable CSRF for stateless REST APIs** — CSRF protection is essential for cookie-based session auth but unnecessary and harmful for token-based APIs. CSRF tokens require server-side state, which contradicts stateless design. Always disable CSRF when using JWT or OAuth2.
- **Use short-lived access tokens with refresh tokens** — Access tokens (5-15 min TTL) limit the damage if stolen. Refresh tokens (7-30 day TTL) stored in HTTP-only cookies provide a secure way to obtain new access tokens without re-authentication. Never store access tokens in `localStorage` — use HTTP-only cookies or in-memory storage.
- **Store roles/permissions in the database, not in code** — Hard-coded roles in `@PreAuthorize` strings require code deployment to change. Store role-permission mappings in the database and load them dynamically in `UserDetails.getAuthorities()`. Use a cache with short TTL so changes propagate quickly.
- **Use `@EnableMethodSecurity` for method-level authorization** — The older `@EnableGlobalMethodSecurity` is deprecated. `@EnableMethodSecurity` integrates with the modern authorization manager pattern and supports `@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, and `@PostFilter`.
- **Always use HTTPS in production** — Without HTTPS, credentials and tokens are transmitted in plain text and can be intercepted via MITM attacks. Enforce HTTPS with `http.requiresChannel(channel -> channel.anyRequest().requiresSecure())`. Use HSTS headers.
- **Secure Actuator endpoints separately** — Actuator endpoints expose sensitive data (heap dumps, env, loggers). Never expose them on the same port as the API without authentication. Either use a separate management port (`management.server.port=8081`) or restrict access with `.requestMatchers("/actuator/**").hasRole("ADMIN")`.
