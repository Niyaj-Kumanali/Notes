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
   - **Answer:** Spring Security is a powerful and highly customizable authentication and access-control framework that has been the de facto standard for securing Spring-based applications. It provides declarative security through a filter chain, protecting not only the application's own routes but also addressing common web vulnerabilities such as CSRF, session fixation, clickjacking, and security headers. At its core, it is built around two concepts: authentication, which verifies who a user is, and authorization, which decides what an authenticated user may do. The framework integrates with the Servlet API through a chain of servlet filters, and in Spring Boot 2.x and 3.x it is configured with the modern `SecurityFilterChain` bean DSL, which replaced the older `WebSecurityConfigurerAdapter`. A typical configuration uses `.authorizeHttpRequests()` to define URL-based rules, `.formLogin()` or a custom filter for credential handling, and `.exceptionHandling()` to customize 401/403 responses. Spring Security also provides method-level security via annotations such as `@PreAuthorize` and `@Secured`, allowing fine-grained access control on service and controller methods. Because it is built on interfaces like `AuthenticationManager`, `AuthenticationProvider`, and `UserDetailsService`, nearly every behavior can be extended without rewriting the framework. The framework ships with production-ready support for form login, HTTP Basic, OAuth2, OpenID Connect, SAML, JWT resource servers, and LDAP. A minimal modern configuration looks like this:

     ```java
     @Bean
     SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
         http
             .csrf(csrf -> csrf.disable())
             .authorizeHttpRequests(auth -> auth
                 .requestMatchers("/api/auth/**").permitAll()
                 .anyRequest().authenticated())
             .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
         return http.build();
     }
     ```

     This single bean encapsulates the security rules for a whole application and demonstrates why Spring Security is the standard choice for securing enterprise Spring applications.

2. Difference between authentication and authorization.
   - **Answer:** Authentication is the process of verifying an identity claim — confirming that the presented credentials, such as a username/password pair or a token, actually belong to the user — while authorization is the subsequent process of deciding what that verified identity is allowed to do. Authentication answers "who are you?", whereas authorization answers "what can you do?". In Spring Security the two concerns are deliberately separated: authentication is handled by components such as `AuthenticationManager`, `AuthenticationProvider`, and `UserDetailsService`, while authorization is enforced by `AccessDecisionManager`-style checks against the `GrantedAuthority` objects held on the `Authentication`. After a successful login, an `Authentication` object holding the principal and its authorities is placed in the `SecurityContext`. Authorization checks then read those authorities without re-verifying identity, so a user can be authenticated with a valid token but still denied access to a resource they are not permitted to use. The same distinction appears in HTTP semantics: a request with no or invalid credentials yields `401 Unauthorized`, while an authenticated but underprivileged request yields `403 Forbidden`. Authorization can be enforced at the URL level with `.hasRole("ADMIN")` or at the method level with `@PreAuthorize`. Good design keeps authentication generic (credential validation) and keeps authorization explicit (rule declarations), which makes security rules testable and auditable.

3. What is security filter chain?
   - **Answer:** The security filter chain is an ordered sequence of servlet filters that intercept every HTTP request before it reaches a controller, applying Spring Security's authentication and authorization logic. Spring Security registers a root filter named `DelegatingFilterProxy` in the servlet container, which delegates to a bean called `FilterChainProxy`. `FilterChainProxy` then dispatches the request to the first matching `SecurityFilterChain`, which is itself a list of ordered filters. These filters include `UsernamePasswordAuthenticationFilter` for login form processing, `BasicAuthenticationFilter` for HTTP Basic, `BearerTokenAuthenticationFilter` for JWT, `ExceptionTranslationFilter` to translate security exceptions into HTTP responses, and `AuthorizationFilter` (the modern replacement for `FilterSecurityInterceptor`) that performs the final URL-based authorization decision. Filters are matched to requests through a `SecurityMatcher`, so multiple chains can exist — for example, one chain for `/api/**` with JWT authentication and another for `/actuator/**` with IP-based rules. A custom filter, such as a JWT validation filter, is inserted into the chain by calling `.addFilterBefore(...)` or `.addFilterAfter(...)` relative to a known filter. Each filter in the chain has a single responsibility and either continues the chain with `filterChain.doFilter()` or short-circuits it, for example by writing a 401 response. Because the chain is a normal Spring-managed component, ordering and registration are explicit and deterministic. The chain is defined declaratively via the `SecurityFilterChain` bean, as shown in the configuration for question 1, which keeps security configuration readable and version-controllable.

4. How does Spring Security process a request?
   - **Answer:** An incoming request first hits the servlet container, where `DelegatingFilterProxy` forwards it to `FilterChainProxy`, the entry point of Spring Security's filter infrastructure. `FilterChainProxy` selects the first `SecurityFilterChain` whose matcher matches the request, then runs that chain's filters in order. Authentication filters, such as `UsernamePasswordAuthenticationFilter` for login endpoints or a custom JWT filter, attempt to convert the request into an `Authentication` object. For a JWT request, the custom filter parses the `Authorization: Bearer <token>` header, validates the signature and expiry, and loads the user; if the token is absent or invalid, the request continues unauthenticated and is rejected later with a 401. On successful authentication, the `Authentication` object is stored in `SecurityContextHolder` so downstream code can access the current user. `ExceptionTranslationFilter` wraps the rest of the chain and catches two kinds of exceptions: `AuthenticationException`, which triggers an `AuthenticationEntryPoint` to produce a 401, and `AccessDeniedException`, which triggers an `AccessDeniedHandler` or an `AuthenticationEntryPoint` to produce a 403. Finally, `AuthorizationFilter` makes the definitive URL-level decision using the configured `authorizeHttpRequests` rules — if the current `Authentication` lacks the required authorities, it throws `AccessDeniedException`; otherwise the request proceeds to the controller. The `SecurityContext` is then cleared by `SecurityContextHolderFilter` before the response completes to prevent leaking authentication state across requests. This layered design — filter chain, context, exception translation, and final authorization — is what makes the request flow both robust and extensible.

5. What is `SecurityContextHolder`?
   - **Answer:** `SecurityContextHolder` is the central storage location for the `SecurityContext` of the currently authenticated user. By default it uses a `ThreadLocal` strategy, meaning each request thread holds its own `SecurityContext`, which contains the `Authentication` object for that request. This design gives any component in the same thread — controllers, services, repositories — convenient access to the current user without threading a principal parameter through every method call. The standard pattern is to read the authentication via `SecurityContextHolder.getContext().getAuthentication()`, from which the principal and authorities can be extracted. The holder supports three storage strategies selected via `SecurityContextHolder.setStrategyName()`: `MODE_THREADLOCAL` (the default, one context per thread), `MODE_INHERITABLETHREADLOCAL` (the context is propagated to child threads created within the request), and `MODE_GLOBAL` (a single application-wide context, rarely used). The `MODE_INHERITABLETHREADLOCAL` strategy matters when using `@Async` methods, because a plain `ThreadLocal` context does not propagate to worker threads — this is a common source of "no authentication found" bugs in asynchronous code. Spring Security clears the context after the request completes through `SecurityContextHolderFilter`, preventing one user's authentication from leaking into another request. Because the context is thread-local and request-scoped, care must be taken with `@Async`, reactive programming, and thread pools; in reactive stacks a `ReactiveSecurityContextHolder` is used instead. The holder is also where manual authentication can be set in custom filters:

     ```java
     Authentication auth = new UsernamePasswordAuthenticationToken(user, null, authorities);
     SecurityContextHolder.getContext().setAuthentication(auth);
     ```

     This simple API makes `SecurityContextHolder` the backbone of Spring Security's request-scoped authentication model.

6. What is `Authentication` object?
   - **Answer:** `Authentication` is Spring Security's representation of a user's identity and granted permissions, and it is the object stored in the `SecurityContext` after successful authentication. It extends the `Principal` interface and exposes three core methods: `getPrincipal()` returns the identity — typically a `UserDetails` object — `getCredentials()` returns the secret (usually a password or token, which is typically cleared or set to `null` after authentication), and `getAuthorities()` returns a collection of `GrantedAuthority` objects that drive authorization decisions. It also carries the `isAuthenticated()` flag, which indicates whether the presented credentials were verified. Before authentication, an unauthenticated token such as `UsernamePasswordAuthenticationToken` is created from raw credentials; after a provider verifies them, the same token is marked authenticated and enriched with authorities. During a JWT flow, a filter constructs an authenticated token directly from the token's claims, setting the principal to a `UserDetails`-like object and the authorities to the roles or permissions parsed from the JWT. The `AuthenticationManager` is the component that orchestrates the transition from unauthenticated to authenticated, delegating to one or more `AuthenticationProvider` instances. Because everything downstream — `@PreAuthorize`, `hasRole()`, controller code reading the current user — consumes this object, keeping it correct and minimal is important. A typical authenticated token creation looks like:

     ```java
     Authentication authentication = new UsernamePasswordAuthenticationToken(
         userDetails, null, userDetails.getAuthorities());
     ```

     The object is deliberately just a data carrier, so it remains simple while the real logic lives in the authentication providers and filters.

7. What is `GrantedAuthority`?
   - **Answer:** `GrantedAuthority` is an interface representing a single permission or role granted to an authenticated user. In practice the most common implementation is `SimpleGrantedAuthority`, which is little more than an immutable string. These authorities are attached to the `Authentication` object and are what Spring Security uses for all authorization decisions, whether at the URL level (`hasRole`, `hasAuthority`) or the method level (`@PreAuthorize`). Two conventions exist: role-based authorities are stored with the `ROLE_` prefix (for example `ROLE_ADMIN`) and checked with `hasRole("ADMIN")`, which automatically prepends the prefix, while permission-based authorities are stored exactly as their name, such as `REPORT_EXPORT`, and checked with `hasAuthority("REPORT_EXPORT")`. The framework treats both the same way internally — they are just strings — and the prefix is purely a naming convention plus a helper for the expression API. Authorities typically originate from the `UserDetails` implementation's `getAuthorities()` method, which maps roles or permissions from the database into `SimpleGrantedAuthority` instances. A common mapping looks like:

     ```java
     return user.getRoles().stream()
         .map(r -> new SimpleGrantedAuthority("ROLE_" + r.getName()))
         .toList();
     ```

     Because authorities are strings, they are easy to serialize into a JWT payload and parse back on the resource server. Keeping authorities granular (permissions) rather than coarse (roles) gives finer access control, while roles provide simpler administration; many systems use both by mapping roles to sets of permissions.

8. What is `UserDetails`?
   - **Answer:** `UserDetails` is the core interface Spring Security uses to represent a user for authentication and authorization purposes. It provides the security-relevant data that the authentication process needs: `getUsername()`, `getPassword()` (the encoded password stored in the system), `getAuthorities()` (the `GrantedAuthority` collection), and four boolean flags — `isAccountNonExpired()`, `isAccountNonLocked()`, `isCredentialsNonExpired()`, and `isEnabled()` — that together determine whether a user account can authenticate. The framework checks these flags during authentication; for example, if `isEnabled()` returns `false`, the provider throws `DisabledException`, and if `isAccountNonLocked()` returns `false`, it throws `LockedException`. The built-in `org.springframework.security.core.userdetails.User` class is a production-ready immutable implementation that is usually sufficient, but custom applications frequently implement `UserDetails` to wrap a JPA entity and avoid duplicating fields. A custom implementation typically delegates the interface methods to entity fields, mapping database roles to authorities as shown in question 7. Because `UserDetails` couples authentication data (username/password/status) with authorization data (authorities), it is the natural place to enrich an authenticated principal with roles and permissions. It is the return type of `UserDetailsService.loadUserByUsername()`, so it sits at the boundary between the persistence layer and the authentication layer. On the resource side, the `getAuthorities()` collection is transferred into the `Authentication` object's authority list, making `UserDetails` the source of truth for both identity and permissions in a typical username/password application.

9. What is `UserDetailsService`?
   - **Answer:** `UserDetailsService` is a strategy interface whose single method, `UserDetails loadUserByUsername(String username)`, loads user data by username — the bridge between the user storage (database, LDAP, in-memory) and Spring Security's authentication engine. It is consumed primarily by `DaoAuthenticationProvider`, which calls `loadUserByUsername()` to fetch the stored user, then verifies the presented password against the stored encoded password using the configured `PasswordEncoder`. The interface has no built-in caching or validation; a typical implementation uses a repository to query the user table and maps the result to a `UserDetails` instance. A standard custom implementation looks like:

     ```java
     @Service
     public class CustomUserDetailsService implements UserDetailsService {
         private final UserRepository userRepository;

         @Override
         public UserDetails loadUserByUsername(String username) {
             return userRepository.findByUsername(username)
                 .map(this::toUserDetails)
                 .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
         }
     }
     ```

     Throwing `UsernameNotFoundException` when the user does not exist is part of the contract so the provider can signal a failed authentication. The service is intentionally simple and stateless, which makes it easy to test and easy to wrap with caching; for example, loading frequently used users from a Redis cache to reduce database pressure. In a JWT flow, `UserDetailsService` is still useful at login time (to verify credentials) and can also be used by the JWT filter to rehydrate the principal from a token's subject. It is important not to throw `InternalAuthenticationServiceException` for missing users, since that may leak information; `UsernameNotFoundException` is the contract. The service keeps the authentication layer decoupled from the persistence schema, which is the core value of the abstraction.

10. What is `AuthenticationProvider`?
    - **Answer:** `AuthenticationProvider` is the strategy interface that actually performs authentication for a specific type of credential. It declares two methods: `authenticate(Authentication authentication)` and `supports(Class<?> authentication)`. The `supports` method tells the `ProviderManager` which authentication token types this provider can handle, and the `ProviderManager` (the default `AuthenticationManager`) iterates its registered providers, delegating to the first one that declares support. Each provider produces a fully authenticated `Authentication` object or throws `AuthenticationException` subclasses such as `BadCredentialsException`, `LockedException`, or `DisabledException`. The framework ships with `DaoAuthenticationProvider`, which handles `UsernamePasswordAuthenticationToken` by loading the user via `UserDetailsService` and comparing the presented password with the encoded stored password using `PasswordEncoder`. A custom provider can implement alternate authentication mechanisms — OTP codes, client certificates, or JWT validation — and register multiple providers for a single `AuthenticationManager` to support several login styles. A custom provider for a token-based flow might look like:

    ```java
    @Override
    public Authentication authenticate(Authentication authentication) {
        String token = authentication.getCredentials().toString();
        UserDetails user = tokenService.validate(token);
        if (user == null) {
            throw new BadCredentialsException("Invalid token");
        }
        return new UsernamePasswordAuthenticationToken(user, token, user.getAuthorities());
    }
    ```

    Because the provider returns an already-authenticated token, the `AuthenticationManager` accepts it without further checks. The key design point is that authentication logic is pluggable: the manager does not know how authentication works, only which provider supports which token, which keeps the framework extensible.

11. What is `PasswordEncoder`?
    - **Answer:** `PasswordEncoder` is the strategy interface responsible for encoding (hashing) passwords and verifying a raw password against an encoded one. It declares `encode(CharSequence rawPassword)` and `matches(CharSequence rawPassword, String encodedPassword)`. It is used pervasively: `DaoAuthenticationProvider` calls `matches()` during login, registration code calls `encode()` before persisting a new password, and `UserDetailsService` data is always stored in encoded form. Spring Security ships with several implementations: `BCryptPasswordEncoder` (the default recommendation), `SCryptPasswordEncoder`, `Pbkdf2PasswordEncoder`, `Argon2PasswordEncoder`, and the deliberately insecure `NoOpPasswordEncoder` and `DelegatingPasswordEncoder`. The `DelegatingPasswordEncoder` is the default and stores the algorithm id in the encoded string (for example `{bcrypt}$2a$10$...`), which allows the application to upgrade hashing algorithms over time without re-encoding every stored password. Password encoding is a one-way transformation, so an encoded password must never be decoded — verification is done only through `matches()`. A security configuration always defines a single `PasswordEncoder` bean, which is then injected both into the `UserDetailsService` or registration code and into the authentication provider:

    ```java
    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
    ```

    Choosing the right encoder and work factor is a security decision: the algorithm must be slow and salted so that even if the database is leaked, the hashes cannot be cracked by brute force or rainbow tables. Storing plain-text or weakly hashed passwords (MD5, SHA-1 without salt) is one of the most serious security defects an application can have.

12. Why use BCrypt?
    - **Answer:** BCrypt is a password hashing function explicitly designed to resist brute-force, rainbow-table, and GPU-based attacks. Unlike fast general-purpose hashes such as MD5 or SHA-256, BCrypt is deliberately slow, and its cost can be increased with a configurable work factor, so the algorithm scales in difficulty with hardware improvements. BCrypt automatically incorporates a random 128-bit salt into the hash output, which means the same password produces a different hash on every invocation, defeating rainbow-table and precomputed-hash attacks. The salt and cost are embedded in the resulting string (for example `$2a$12$<salt><hash>`), so verification needs no separate salt storage and the work factor can be read back from the hash itself. In Spring Security, `BCryptPasswordEncoder` implements this with a strength parameter — commonly 10 to 13 — where each increment roughly doubles the computation time. Because hashing is one-way, even a full database leak does not reveal plaintext passwords directly; an attacker is forced to test guesses one at a time, and the high cost makes that impractical. Modern alternatives such as Argon2, scrypt, and PBKDF2 are also acceptable, but BCrypt remains the widely used, well-audited default in Spring Security. The trade-off is intentional: a login request is slightly slower, but the security benefit far outweighs the small latency, and the cost can be tuned so that one verification stays comfortably under a hundred milliseconds. That combination — salting, cost factor, one-way design, and library support — is why BCrypt is the standard recommendation for password storage.

13. What is JWT?
    - **Answer:** JWT (JSON Web Token) is a compact, URL-safe token format defined by RFC 7519 that encodes claims as JSON and signs them so they can be verified without server-side session storage. A JWT has three dot-separated parts: a header, a payload, and a signature. The header declares the signing algorithm and token type, typically `{"alg":"HS256","typ":"JWT"}`; the payload contains registered claims such as `sub` (subject), `iat` (issued at), `exp` (expiration), `iss` (issuer), and `aud` (audience), plus any custom claims; and the signature is computed over the base64url-encoded header and payload. For HMAC-signed tokens the signature is an HMAC-SHA256 digest keyed with a shared secret, while for RSA-signed tokens it is produced with a private key and verified with the corresponding public key. The format is self-contained: all the information needed to identify and authorize the user is inside the token, which makes it ideal for distributed systems where each service can validate a token without a central session store. A decoded token looks like `header.payload.signature`, for example `eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIn0.TJVA95OrM7E2c...`. Clients present the token in the `Authorization: Bearer <token>` header on each request. Because the payload is only base64-encoded and not encrypted, JWTs must never carry secrets. Spring Security integrates JWT validation natively through `oauth2ResourceServer()` with a JWT decoder, or through libraries such as jjwt for custom filter implementations.

14. How does JWT authentication work?
    - **Answer:** JWT authentication is a stateless flow that replaces server-side session management with a self-contained signed token. The flow begins at a login endpoint: the client posts credentials, the server authenticates them through `AuthenticationManager` (which delegates to `DaoAuthenticationProvider` and `UserDetailsService`), and upon success generates a signed JWT containing the user's identity, roles, and expiration. The token is returned to the client, which stores it and sends it with every subsequent request in the `Authorization: Bearer <token>` header. On each protected request, a custom JWT filter (typically extending `OncePerRequestFilter`) extracts the header, validates the signature and expiry, parses the claims, and reconstructs an authenticated `Authentication` object which is placed in the `SecurityContextHolder`. Downstream authorization checks then evaluate roles and permissions from that context, just as they would for a session-based login. Because the server keeps no per-user state, the token must carry everything needed — which is why expiry and signature are mandatory. A minimal JWT creation snippet is:

    ```java
    String token = Jwts.builder()
        .subject(username)
        .claim("roles", roles)
        .issuedAt(new Date())
        .expiration(new Date(System.currentTimeMillis() + 3600000))
        .signWith(secretKey)
        .compact();
    ```

    The success criteria are: the server never stores session state, the client always presents the token, and every request is re-validated from scratch. If the token is missing, expired, or has an invalid signature, the request is rejected with a 401 before it reaches any controller logic.

15. What should a JWT contain?
    - **Answer:** A JWT should contain the minimum set of claims needed to authenticate and authorize a request, all of which must be non-sensitive. The standard registered claims are the preferred way to convey structured information: `sub` for the subject (typically the user identifier), `iat` for the issued-at timestamp, `exp` for the expiration timestamp, `iss` for the token issuer, `aud` for the intended audience, and `jti` for a unique token identifier that enables revocation and replay detection. Custom claims should be limited to identity and authorization data such as the user's username, role names, and specific permission strings, because the resource server needs those to build the `GrantedAuthority` collection without hitting a database. Keeping the payload small matters because the token is sent in the `Authorization` header on every request, and large tokens add HTTP overhead and can approach header size limits. A typical claims set looks like:

    ```json
    {
      "sub": "42",
      "username": "john.doe",
      "roles": ["ADMIN"],
      "iat": 1750000000,
      "exp": 1750003600,
      "iss": "order-service",
      "jti": "a3f8-9c21-4bd0"
    }
    ```

    The rule is to include only what is required to serve requests without further lookups, to always set `exp` and `iat`, and to reserve the token for claims rather than for large profile data or anything secret. If a claim would be sensitive if read by a third party, it should not be in the payload.

16. What should a JWT not contain?
    - **Answer:** A JWT must never contain sensitive data because the payload is only base64-encoded, not encrypted — anyone who obtains the token can decode and read every claim without knowing the signing key. Passwords, password hashes, credit card numbers, social security numbers, medical data, API keys, or any personally identifiable information must be excluded. Even a valid, unexpired token can be read by any service or attacker that intercepts it, and unlike a server-side session, the data is replicated with every single request in the HTTP header. Embedding authorization decisions or user permissions in a token is also risky if the resource server trusts them blindly, since a leaked token becomes a permanent credential that can only be invalidated by revocation or expiry. Instead, sensitive data should remain server-side and be looked up when needed; the token should carry only identifiers and coarse authorization hints. If sensitive claims are genuinely unavoidable, the proper solution is JWE (JSON Web Encryption), which encrypts the payload rather than merely signing it — but JWE adds complexity and should be a last resort. The general security principle is defense in depth: treat the JWT as a credential that could be leaked at any time, and ensure that nothing in it harms the system if that happens.

17. How do you validate JWT signature?
    - **Answer:** JWT signature validation verifies that a token was issued by the trusted server and has not been tampered with, by recomputing the signature and comparing it to the one attached to the token. For symmetric algorithms such as HS256, the server recomputes the HMAC using the shared secret key; if the result matches, the token is authentic. For asymmetric algorithms such as RS256, the resource server verifies the signature using the public key corresponding to the private key that signed the token — this allows many resource servers to validate tokens without sharing a secret, typically by fetching keys from a JWKS endpoint. In Spring Security, JWT validation is built into `oauth2ResourceServer()` through `jwtDecoder()`, or it can be done explicitly with a library like jjwt:

    ```java
    Claims claims = Jwts.parser()
        .verifyWith(secretKey)
        .build()
        .parseSignedClaims(token)
        .getPayload();
    ```

    Parsing with `parseSignedClaims` automatically validates the signature and throws `JwtException` for tampered, malformed, or incorrectly keyed tokens. Beyond the signature, validation must check the `exp` claim for expiration, the `iat` and `nbf` claims for validity windows, and optionally `aud`, `iss`, and `jti`. The signing key must never be committed to source code; it belongs in environment variables, a secret manager, or a keystore. Failures should be handled centrally — catching `JwtException` in the filter and returning a 401 — rather than letting malformed tokens bubble up as 500 errors. Proper validation is the entire security boundary of a stateless JWT system: if the signature is trusted, the claims are trusted, so every claim used for authorization must be covered by that signature.

18. Difference between access token and refresh token.
    - **Answer:** An access token is a short-lived credential used to authorize API requests — typically valid for 15 to 30 minutes — while a refresh token is a long-lived credential, often valid for days or weeks, used only to obtain new access tokens without forcing the user to re-enter credentials. The two are separated to balance security and usability: a short access-token lifetime means a stolen token is usable only briefly, and the long refresh-token lifetime means users are not constantly re-authenticating. Access tokens are sent on every request in the `Authorization: Bearer` header, so they travel more frequently and are more exposed; refresh tokens are transmitted rarely, only when a new access token is needed, and are stored more securely. In a typical flow the login endpoint returns both, the client uses the access token for requests, and when it expires the client calls a `/auth/refresh` endpoint with the refresh token; the server validates the refresh token, optionally rotates it, and issues a new access/refresh pair. Access tokens are validated statelessly on every request (signature, expiry, claims), whereas refresh tokens are usually validated against a server-side store so they can be revoked. A refresh token's higher value means it needs stronger protection: storage in an httpOnly cookie or a secure server store, and rotation on each use to detect theft. Together they form the standard stateless-but-revocable design that keeps the API fast while retaining control over session lifetime.

19. How do you revoke JWT?
    - **Answer:** A stateless JWT is inherently hard to revoke because the server validates it purely from its signature and claims — nothing server-side is consulted, so "deleting" a token is meaningless. The practical strategies all trade off some statelessness for revocation capability. The most common approach is a denylist: keep a server-side store (Redis works well) of revoked token identifiers (`jti`) or the token itself, and have the authentication filter check the list on every request, rejecting any token found there until its natural expiry. A Redis denylist is efficient because the check is a single `GET` and entries can expire automatically with the token. A complementary strategy is short access-token lifetimes, which shrink the window in which a compromised token is usable. For long-lived sessions, revoking the refresh token is the more powerful lever: if the refresh token is invalidated in its store, the user cannot obtain new access tokens, and the outstanding access token simply dies at expiry. Other approaches include bumping a user's `passwordChangedAt` timestamp and refusing tokens issued before it, or switching to opaque tokens that are checked against the authorization server on every request. A typical denylist check in a filter looks like:

    ```java
    if (jti != null && redisTemplate.hasKey("blacklist:jti:" + jti)) {
        throw new JwtException("Token revoked");
    }
    ```

    The key principle is to design for the worst realistic window — the time between theft detection and natural token expiry — and to combine several controls: short expiry, refresh-token revocation, denylist, and fast password resets.

20. What is token rotation?
    - **Answer:** Token rotation is the practice of issuing a new refresh token every time a refresh token is used, and immediately invalidating the old one, so that each refresh token is single-use. In a rotation flow, when the client calls the refresh endpoint, the server validates the presented refresh token, revokes it in its store, and returns a brand-new access/refresh pair. This turns a stolen refresh token into a bounded risk: an attacker can use it at most once, and if the legitimate user refreshes after the theft, the new token invalidates the attacker's copy — the system can even detect concurrent-use anomalies and revoke the whole session. Rotation is implemented by making refresh-token use stateful: each token is stored server-side with an ID and is marked consumed upon use, and the update that revokes the old token and persists the new one must be atomic, typically inside a `@Transactional` method. A typical refresh handler looks like:

    ```java
    @Transactional
    public TokenPair rotate(String oldToken) {
        RefreshToken stored = refreshTokenRepository.findByToken(oldToken)
            .orElseThrow(() -> new InvalidTokenException());
        if (stored.isRevoked() || stored.isExpired()) throw new InvalidTokenException();
        stored.revoke();
        return issueNewPair(user);
    }
    ```

    Replay detection is an added benefit: if the same refresh token is submitted twice, it is already consumed and the request is rejected, which signals possible theft. The trade-off is more state and more writes per refresh, but that is exactly why rotation is the recommended pattern for high-security applications. Rotation combined with short-lived access tokens gives a revocation window measured in minutes rather than weeks.

21. Where should JWT be stored?
    - **Answer:** On the client side, a JWT should be stored in the most restricted place that still allows the application to function: an httpOnly, Secure, SameSite cookie, or an in-memory variable — never in `localStorage` or `sessionStorage`. An httpOnly cookie is not readable by JavaScript, so even if an attacker achieves XSS, they cannot extract the token; `Secure` restricts transmission to HTTPS, and `SameSite=Strict/Lax` limits cross-site sending. The in-memory approach stores the token in a JavaScript variable, so it is destroyed on page refresh and is never persisted on disk, but a refresh flow is required to restore it after reload. In practice a common pattern is: access token held in memory (short-lived, minimizes exposure), refresh token held in an httpOnly cookie (long-lived, protected from XSS). This combination mitigates XSS for the access token and mitigates CSRF for the refresh token, since the refresh endpoint uses a cookie only when invoked intentionally and can additionally require a custom header. `localStorage` is attractive for simplicity — a single `localStorage.setItem("token", ...)` — but any XSS bug anywhere in the origin immediately surrenders the token, and the token also survives in the browser profile, increasing theft surface. The definitive rule is that anything JavaScript can read, XSS can steal; therefore tokens should live in memory or behind httpOnly cookies, and never in web storage that persists.

22. Why is storing JWT in localStorage risky?
    - **Answer:** Storing a JWT in `localStorage` is risky because `localStorage` is fully readable and writable by any JavaScript running on the same origin, which means any XSS vulnerability immediately becomes a credential-theft vulnerability. If an attacker injects a script — through a comment field, an image URL, a vulnerable dependency, or a malicious link — that script can execute `localStorage.getItem("token")` and exfiltrate the token to the attacker's server without any user interaction. Unlike an httpOnly cookie, which JavaScript cannot read, `localStorage` offers zero protection against XSS; the security of the token depends entirely on there never being a single XSS bug. Additionally, tokens in `localStorage` persist across browser sessions and are copied into backups and sync services, widening the theft surface even after the user logs out. Because `localStorage` is origin-scoped but not request-scoped, the token is also not automatically attached to requests, so the application must read it manually and place it in the `Authorization` header — one more place for bugs. The safer alternatives are httpOnly cookies, which XSS cannot read, and in-memory storage, which is destroyed on reload and limits the blast radius of any single bug. If `localStorage` must be used, the mitigation is defense in depth: strict input sanitization, a Content Security Policy that blocks inline scripts and restricts sources, and short token lifetimes. The fundamental principle is that web storage is not a secret store; secrets in storage readable by scripts are secrets an attacker can obtain.

23. What is CSRF?
    - **Answer:** CSRF (Cross-Site Request Forgery) is an attack that tricks an authenticated victim's browser into sending forged state-changing requests to a site the victim trusts. The classic scenario relies on session cookies: the browser automatically attaches cookies to requests for the target origin, so when the victim visits a malicious page, that page can submit a form or trigger a request to the target application and the victim's session cookie is sent along — the server cannot distinguish the forged request from a legitimate one. CSRF protection works by forcing requests to carry an additional secret that the attacker cannot know: a synchronizer token that the server generates, stores in the session, and requires to be echoed back in a hidden form field or custom header. Spring Security enables CSRF protection by default for state-changing HTTP methods (`POST`, `PUT`, `PATCH`, `DELETE`), and its `CsrfFilter` validates the token for every such request. Because the mitigation relies on the session, applications that are fully stateless — where authentication is carried in a custom header such as `Authorization: Bearer`, not in cookies — are not vulnerable, since a cross-origin attacker has no way to supply the token. CSRF is complementary to XSS: CSRF abuses the browser's cookie behavior, while XSS abuses JavaScript execution. Modern defenses also include `SameSite` cookie attributes and strict CORS, but the synchronizer token remains the core defense in session-based applications. When configuring stateless JWT APIs, CSRF is commonly disabled, but only because the authentication mechanism does not depend on browser-managed cookies.

24. When can CSRF be disabled?
    - **Answer:** CSRF protection can be disabled only when the application's authentication and state-changing requests do not rely on browser-automatically-attached credentials, such as cookies. The clearest case is a stateless REST API that authenticates with a custom header like `Authorization: Bearer <JWT>`: a cross-site attacker cannot read the victim's token, so a forged request carries no credentials and is rejected as unauthenticated. CSRF can also be disabled when every state-changing request is required to carry a custom header (for example `X-Requested-With: XMLHttpRequest` or `Content-Type: application/json`) that a cross-origin form cannot set, or when the client and API are on different origins under a strict CORS policy that only permits the real client. Conversely, CSRF protection must stay on for classic session-cookie-based web applications, where the browser attaches the session cookie automatically and the synchronizer token is the only thing distinguishing a legitimate form from an attack. Before setting `.csrf(csrf -> csrf.disable())`, the configuration must be audited to confirm no cookie-based authentication path exists anywhere, including login endpoints and OAuth2 flows. A safe rule is: keep CSRF enabled by default, and disable it explicitly and deliberately only for documented, stateless, header-authenticated APIs. The decision is recorded in configuration so reviewers can verify the reasoning:

    ```java
    http
        .csrf(csrf -> csrf.disable()) // stateless JWT auth; no cookies to forge
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
    ```

    Disabling CSRF without understanding the authentication mechanism is a real vulnerability, so the justification should always be stated in the code.

25. What is CORS?
    - **Answer:** CORS (Cross-Origin Resource Sharing) is a browser security mechanism that controls whether a web page on one origin may issue requests to resources on another origin. By default, browsers enforce the Same-Origin Policy: a script loaded from origin A cannot read the response of a request to origin B unless B explicitly allows it. CORS works through HTTP headers: the server for origin B responds to a request with `Access-Control-Allow-Origin`, telling the browser which origins may read the response. For "simple" requests the browser sends the actual request and checks the response headers; for requests that use non-simple methods (`PUT`, `DELETE`) or custom headers (`Authorization`, `Content-Type: application/json`), the browser first sends a preflight `OPTIONS` request asking the server which methods, headers, and origins are permitted, and only sends the real request if the preflight passes. Because Spring Security's filter chain intercepts requests before the servlet's CORS handling, CORS must be configured inside Spring Security (or via a filter that runs before it) so that preflight `OPTIONS` requests are answered rather than rejected with 401. A proper setup requires the server to allow the frontend's exact origin, the methods the frontend uses, and the headers it sends, including `Authorization`. CORS is a browser enforcement, not a server-side security boundary: it stops browser-based reads from other origins, but does not stop direct non-browser clients, so authorization rules must remain strict regardless. Configuring CORS is about defining precisely which web frontends may consume the API.

26. How do you configure CORS?
    - **Answer:** In Spring Security the recommended way to configure CORS is to define a `CorsConfigurationSource` bean and enable it on the `HttpSecurity` with `.cors(cors -> cors.configurationSource(source))`, so CORS applies consistently across the whole filter chain rather than being scattered on controllers. A typical configuration specifies the allowed origins, methods, headers, and whether credentials are allowed:

    ```java
    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
    ```

    `setAllowedOrigins` should list exact origins rather than `*` when credentials are involved, because the wildcard is invalid with `setAllowCredentials(true)`. `setAllowCredentials(true)` is required only when cookies are sent, for example when using httpOnly cookie-based authentication; for pure Bearer-token flows it can stay false. `setMaxAge` caches the preflight result in the browser, reducing `OPTIONS` traffic. Alternatively, `@CrossOrigin` can be placed on a controller or method for narrow, local rules, but global configuration via a bean is preferred for consistency and for automatically handling preflight requests inside the security chain. When Spring Security sees an `OPTIONS` preflight, the CORS filter answers it with the configured headers before any authentication logic runs. The configuration should reflect exactly what the frontend needs — nothing more — to keep the allowed cross-origin surface minimal.

27. Difference between `hasRole()` and `hasAuthority()`.
    - **Answer:** `hasRole("ADMIN")` and `hasAuthority("ADMIN")` are both SpEL authorization expressions that check the authorities of the current `Authentication`, but they differ in the role prefix convention. `hasRole("ADMIN")` automatically prepends the `ROLE_` prefix and actually checks for the authority `ROLE_ADMIN`, so it matches authorities created from roles such as `new SimpleGrantedAuthority("ROLE_ADMIN")`. `hasAuthority("REPORT_EXPORT")` checks for the exact authority string as given, with no prefix manipulation, so it matches authorities like `new SimpleGrantedAuthority("REPORT_EXPORT")`. The `ROLE_` prefix is purely a naming convention built into the expression helpers: `hasRole`, `hasAnyRole`, and the `roleHierarchy` mechanism all understand it, while `hasAuthority` and `hasAnyAuthority` treat strings literally. In practice roles are coarse-grained buckets (ADMIN, MANAGER, USER) that are convenient to assign, while authorities are fine-grained permissions (REPORT_CREATE, REPORT_EXPORT) that give precise control. A common design maps both onto the same authority list, storing roles as `ROLE_*` and permissions as bare names, then using:

    ```java
    @PreAuthorize("hasRole('ADMIN') or hasAuthority('REPORT_EXPORT')")
    ```

    The same distinction applies to the URL DSL: `.hasRole("ADMIN")` versus `.hasAuthority("REPORT_EXPORT")`. When using `hasRole`, the prefix can be customized, but the default is almost always kept. Understanding this difference matters because mixing the two incorrectly — checking `hasAuthority("ADMIN")` against `ROLE_ADMIN` — silently fails authorization.

28. What is method-level security?
    - **Answer:** Method-level security applies authorization rules directly on service or controller methods using annotations such as `@PreAuthorize`, `@PostAuthorize`, `@Secured`, and `@RolesAllowed`, enabling fine-grained access control beyond URL-level rules. It is activated with `@EnableMethodSecurity`, which installs AOP interceptors that evaluate the annotations before (and optionally after) the method executes. `@PreAuthorize` is the most powerful because it accepts a full SpEL expression that can reference the authenticated user, method arguments, and returned values. This allows rules that a URL pattern cannot express, such as "a user can only modify their own record":

    ```java
    @PreAuthorize("#id == authentication.principal.userId")
    public Order getOrder(Long id) { ... }
    ```

    Unlike URL-based security, which decides solely on path and HTTP method, method security can make decisions based on data — ownership, status, tenant, or relationships between arguments and the principal. `@Secured` and `@RolesAllowed` are simpler, accepting only a fixed list of roles rather than expressions, while `@PostAuthorize` evaluates after execution, typically against the returned object. The interceptors run through `AuthorizationManager` and integrate with the same authority model, so roles and permissions behave identically to URL checks. Method security is best used for cross-cutting business rules (ownership, status transitions) while URL security covers coarse routing; layering both provides defense in depth. Because annotations execute on method calls, they also protect against callers that bypass the web layer, such as scheduled jobs or internal services — provided the security context has been populated.

29. What is `@PreAuthorize`?
    - **Answer:** `@PreAuthorize` is a method-level security annotation that evaluates a SpEL expression before the method executes; if the expression evaluates to `false`, access is denied with an `AccessDeniedException` and the method body never runs. It is the primary tool for expression-based, data-aware authorization in Spring Security and requires `@EnableMethodSecurity` to be active. The expression context exposes useful variables: `authentication` (the current `Authentication`), `principal` (usually the `UserDetails`), and any method parameters referenced by their names with the `#` prefix. Common expressions combine authorities and data:

    ```java
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long userId) { ... }

    @PreAuthorize("#order.customerId == authentication.principal.customerId")
    public Order getOrder(Order order) { ... }
    ```

    The first restricts to administrators; the second enforces ownership, ensuring a user can only act on their own data — a rule impossible to express with URL matching alone. Expressions support logical operators (`and`, `or`, `!`), so complex policies can be stated declaratively. Because the check runs inside the method interceptor, it also protects non-HTTP entry points such as schedulers and message listeners, as long as a security context exists. Failing the expression throws `AccessDeniedException`, which `ExceptionTranslationFilter` converts to a 403 for web requests. The main caveat is that the SpEL expression is evaluated against method arguments, so argument names must be available (the `-parameters` compiler flag) or `@Param` must be used. Using `@PreAuthorize` moves authorization out of controller code and into declarative rules that are easy to review and test.

30. What is `@PostAuthorize`?
    - **Answer:** `@PostAuthorize` evaluates an authorization expression after a method has executed, using the method's returned object to make the access decision. It is useful when the authorization rule depends on data that is only available inside the method's result — for example, a fetch-by-id operation where the ID does not reveal ownership. The returned object is exposed in the expression as `returnObject`:

    ```java
    @PostAuthorize("returnObject == null || returnObject.ownerId == authentication.principal.userId")
    public Document findDocument(Long id) { ... }
    ```

    If the expression is `false`, `AccessDeniedException` is thrown even though the method ran, so the caller never receives the data. A common pattern returns `returnObject` unchanged when allowed, or `null` when the caller must not learn the resource exists. The critical trade-off is performance and information exposure: unlike `@PreAuthorize`, the method body executes even for unauthorized callers, so sensitive work (expensive queries, side effects) should not sit behind `@PostAuthorize` alone. It is best combined with `@PreAuthorize` for coarse pre-checks plus `@PostAuthorize` for the final data-level decision. The `returnObject` variable requires the method to return a single object; collections need a different strategy, such as `@PreAuthorize` with filtering or a service-layer guard. Since the decision is made on the returned data, it must be guaranteed that the method cannot leak the object through other channels (logs, exceptions) before the check runs. Used judiciously, `@PostAuthorize` closes the gap for object-level access control that URL and pre-execution checks cannot cover.

31. What is OAuth2?
    - **Answer:** OAuth2 is an authorization framework (RFC 6749) that lets an application obtain limited, scoped access to a user's resources on another service without ever seeing the user's password. It defines four roles: the resource owner (the user), the client (the application requesting access), the authorization server (which authenticates the user and issues tokens), and the resource server (which hosts the protected resources and validates tokens). Access is granted via tokens rather than credentials, and tokens carry scopes that bound what the client can do. The framework defines several grant types for different situations: the authorization code flow (with PKCE for public clients like SPAs and mobile apps), the client credentials flow (server-to-server), the resource owner password flow (legacy, discouraged), and the refresh token grant. In the authorization code flow, the client redirects the user to the authorization server, the user authenticates and consents, and the server redirects back with a code that the client exchanges for tokens — the user's password is never shared with the client. OAuth2 is about delegation of access, not identity; confirming who the user is requires OpenID Connect on top. Spring Security supports all of this through `spring-security-oauth2-client` for login, and `oauth2ResourceServer()` for validating tokens issued by an external authorization server. OAuth2 solves the credential-sharing problem at scale, which is why it underpins social login, API ecosystems, and microservice auth.

32. Difference between OAuth2 and JWT.
    - **Answer:** OAuth2 is a protocol framework that describes how authorization is delegated and how tokens are obtained, refreshed, and scoped, while JWT is a self-contained token format with a defined structure and signature mechanism. They operate at different layers: OAuth2 defines the flow (who talks to whom, what grants are used, where tokens are issued), and JWT defines the bytes of the token that flows through those steps. An OAuth2 access token may be a JWT, but it may equally be an opaque random string that the resource server validates by calling an introspection endpoint — the token format is an implementation choice inside the framework. Conversely, a JWT can be used completely outside OAuth2, as in a first-party application that issues and validates its own signed tokens with no external authorization server. Practically, this means they answer different questions: OAuth2 answers "how do we authorize a third party?", JWT answers "how do we encode and verify claims in a stateless token?". The two combine naturally in microservices: an OAuth2 authorization server issues JWTs, and resource servers validate the signature and read scopes/claims. The distinction also matters for support: a JWT-based system needs signing keys and claim validation; an OAuth2 system additionally needs grant flows, consent, client registration, and token lifecycle management. Choosing between them is really choosing whether the application needs the full delegation protocol or just a signed token for its own APIs.

33. What is OpenID Connect?
    - **Answer:** OpenID Connect (OIDC) is an identity layer built on top of OAuth2 that adds standardized authentication on top of OAuth2's authorization delegation. OAuth2 alone does not tell the client reliably who the user is — an access token is meant for the resource server, not for proving identity to the client. OIDC solves this by adding an ID token, a JWT that carries verified identity claims about the user (`sub`, `name`, `email`, `email_verified`, and more), which the client can validate locally with the authorization server's public key. The core flow is the authorization code flow with an extra `openid` scope: after the user authenticates, the authorization server returns an authorization code, the client exchanges it for an ID token plus access and refresh tokens, and the ID token proves identity while the access token grants resource access. The authorization server publishes its configuration and signing keys at well-known endpoints (`.well-known/openid-configuration` and a JWKS URI), which lets clients discover everything needed to validate ID tokens automatically. Spring Security integrates OIDC as a client via `spring-security-oauth2-client`, making it straightforward to support login with providers like Google, GitHub, or Azure AD:

    ```yaml
    spring:
      security:
        oauth2:
          client:
            registration:
              google:
                client-id: ...
                client-secret: ...
                scope: openid, profile, email
    ```

    The practical distinction is: OAuth2 is "allow this app to act on my behalf", OIDC is "confirm to this app who I am". Most modern identity providers (Keycloak, Auth0, Entra ID, Cognito) implement OIDC as the standard way to add external login to an application.

34. What is resource server?
    - **Answer:** A resource server is a service that hosts protected resources — data and APIs — and grants access only to requests presenting a valid access token. It does not issue tokens and does not own the user's credentials; instead it validates tokens issued by an authorization server and enforces the scopes or claims those tokens carry. In Spring Security, a resource server is configured with `oauth2ResourceServer()`, and the framework wires up token validation, decoding the JWT with a key fetched from a JWKS endpoint or a configured secret:

    ```java
    http
        .oauth2ResourceServer(rs -> rs
            .jwt(jwt -> jwt.jwkSetUri("https://auth.example.com/.well-known/jwks.json")));
    ```

    Each request that reaches a protected route must present a token, which the framework validates (signature, expiry, issuer, audience) before converting the claims into an `Authentication` populated with the token's authorities and scopes. Access is then enforced with the usual URL rules or `@PreAuthorize` using scope or claim-based expressions. The resource server stays stateless: it validates tokens locally with public keys, so it does not call the authorization server on every request (unless opaque tokens and introspection are used). This allows many resource servers to share one authorization server while each enforces its own authorization rules. Key concerns for a resource server are correct key rotation handling, validating `aud` so a token for another service is rejected, and ensuring no endpoint trusts the token without validation. The resource server is the enforcement point of the OAuth2 model.

35. What is authorization server?
    - **Answer:** An authorization server is the component that authenticates users and issues tokens (access tokens, refresh tokens, and optionally ID tokens) after successful authentication and consent. It is the trusted issuer in the OAuth2 architecture: resource servers validate tokens against the keys it publishes, and clients obtain tokens through its grant flows. In the OAuth2 model the authorization server is deliberately separated from resource servers so that issuance and validation can scale and evolve independently. Dedicated products include Spring Authorization Server (the successor to the removed legacy authorization server support in Spring Security), Keycloak, and cloud identity services like AWS Cognito and Entra ID. A common architecture for first-party applications collapses the roles: the same Spring Boot application issues JWTs at a `/auth/login` endpoint and validates them for its own APIs, acting as both issuer and resource server — which is simpler but couples issuance with the business domain. A production microservice setup instead centralizes issuance in a dedicated authorization server, and each resource server only fetches public keys from its JWKS endpoint to validate tokens. The authorization server owns identity data, token issuance policies, token lifetimes, scopes, client registrations, and consent screens, making it the highest-value target in the system — it must be hardened, monitored, and separated from untrusted network zones. Its core contract is simple: authenticate the user once, issue a signed token the rest of the system trusts.

36. How do you secure actuator endpoints?
    - **Answer:** Actuator endpoints expose operational data — health, metrics, environment variables, loggers, thread dumps — and several of them leak sensitive internals, so they must be protected like any other resource. The first line of defense is limiting exposure at the management layer: expose only the endpoints that are actually needed, for example health and info publicly, and everything else behind authentication:

    ```yaml
    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus
    ```

    The second line is a dedicated `SecurityFilterChain` for the actuator base path, so rules are explicit and independent of the application's API security. A common pattern grants anonymous access only to `health` and `info`, and requires an ADMIN role for sensitive endpoints such as `env`, `loggers`, `heapdump`, and `threaddump`:

    ```java
    @Bean
    SecurityFilterChain actuatorChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .anyRequest().hasRole("ADMIN"))
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
    ```

    Because this chain matches only `/actuator/**`, it does not interfere with the main API chain, and `@Order` on the beans controls which chain wins for overlapping patterns. Additional hardening includes binding management endpoints to a separate port or localhost in production, restricting by source IP with `hasIpAddress()`, and disabling exposure entirely if the deployment does not need remote monitoring. `health` can stay public (it usually reveals no secrets), but anything that can change behavior or dump state must be restricted.

37. How do you handle unauthorized response?
    - **Answer:** By default, Spring Security's responses for missing or invalid authentication are not API-friendly — an unauthenticated request to a protected endpoint may get a redirect to a login page or an empty 403/401 — so API applications customize them with an `AuthenticationEntryPoint` (for unauthenticated requests, producing 401) and an `AccessDeniedHandler` (for authenticated but forbidden requests, producing 403). A typical entry point writes a JSON error body with a consistent structure instead of an HTML page:

    ```java
    @Component
    public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {
        @Override
        public void commence(HttpServletRequest req, HttpServletResponse res,
                             AuthenticationException ex) throws IOException {
            res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            res.setContentType(MediaType.APPLICATION_JSON_VALUE);
            res.getWriter().write("{\"error\":\"Unauthorized\",\"message\":\"Missing or invalid credentials\"}");
        }
    }
    ```

    The handlers are registered in the security chain via `.exceptionHandling(eh -> eh.authenticationEntryPoint(entryPoint).accessDeniedHandler(deniedHandler))`. In a JWT architecture, the entry point is what turns a missing or expired token into a clean 401 with the right `WWW-Authenticate` header, and the denied handler turns an insufficient-authority request into a 403. The error contract should be consistent across the API — the same shape for validation errors and security errors — so clients can handle failures uniformly. It is also good practice to include a correlation or request ID in the response to support debugging without leaking internals. `ExceptionTranslationFilter` is the component that catches the underlying `AuthenticationException` and `AccessDeniedException` and invokes these handlers, so customizing them centralizes all security error responses in one place.

38. Difference between 401 and 403.
    - **Answer:** `401 Unauthorized` means the request lacks valid authentication — there is no token, the token is expired, or the token has an invalid signature — while `403 Forbidden` means the request was successfully authenticated but the user does not have permission to access the resource. The two are decided at different points in the filter chain: a missing or invalid token is an authentication problem surfaced by the `AuthenticationEntryPoint`, and an authenticated user hitting a route they are not allowed to hit is an authorization problem surfaced by the `AccessDeniedHandler`. The mapping is driven by which exception is thrown: `AuthenticationException` yields 401, and `AccessDeniedException` yields 403. In practice, an expired JWT produces 401, and a partner user calling an admin-only endpoint produces 403. The distinction matters for clients: on a 401 the client should re-authenticate or refresh its token, while on a 403 re-authenticating will not help because the account lacks the required role or permission. A subtlety is that when an unauthenticated user requests a resource requiring a specific role, the framework sends 401 (authentication is the first gate) rather than 403. In Spring Security, `ExceptionTranslationFilter` implements this logic: it catches `AccessDeniedException`, and if the user is anonymous or not authenticated, it delegates to the `AuthenticationEntryPoint` (401), otherwise to the `AccessDeniedHandler` (403). Custom handlers (question 37) make these responses explicit JSON rather than framework defaults.

39. How do you implement RBAC?
    - **Answer:** RBAC (Role-Based Access Control) restricts system access by assigning users to roles and roles to permissions. The database model is typically three tables — `users`, `roles`, and a join table `user_roles` (or `users_roles`) — plus optionally a `permissions` table linked to roles. During authentication, `UserDetailsService` loads the user and its roles, mapping each role name to a `GrantedAuthority` with the `ROLE_` prefix. Spring Security then enforces role checks both in the filter chain and with method security:

    ```java
    @PreAuthorize("hasRole('ADMIN')")
    public void deletePartner(Long id) { ... }

    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    public List<Report> listReports() { ... }
    ```

    Roles are coarse-grained and stable, which makes them easy to administer: an admin assigns a role, and every endpoint protected by that role reacts immediately. Role checks appear in the URL DSL too, for example `.requestMatchers("/api/admin/**").hasRole("ADMIN")`. Role hierarchy can express inheritance, where a MANAGER effectively has ADMIN's role for rule purposes, configured with `RoleHierarchy`. Care is needed to map roles from the database exactly once and to keep the role-to-authority mapping in one place, usually in the `UserDetails` construction. RBAC is simple to reason about and audit, which is why it is the default model for most line-of-business applications, but it becomes coarse when fine-grained object-level rules are needed — at that point permission-based access (question 40) or ownership expressions are layered on top.

40. How do you implement permission-based access?
    - **Answer:** Permission-based (or attribute-based) access control grants access on fine-grained permissions such as `REPORT_EXPORT`, `USER_DELETE`, or `ORDER_APPROVE` rather than on coarse roles. The data model adds a `permissions` table and a role-to-permission mapping, so a role is effectively a bundle of permissions; the security-relevant authorities loaded into `UserDetails` then include both the role names (as `ROLE_*`) and the permissions (as bare strings). Enforcement uses `hasAuthority`:

    ```java
    @PreAuthorize("hasAuthority('REPORT_EXPORT')")
    public byte[] exportReport() { ... }

    @PreAuthorize("hasAuthority('USER_DELETE')")
    public void deleteUser(Long id) { ... }
    ```

    The advantage is precision: two users with the same role can have different effective capabilities, and new capabilities can be granted without creating new roles. A common enrichment is object-level rules that combine permissions with data, such as ownership — `@PreAuthorize("hasAuthority('REPORT_VIEW') and #report.ownerId == authentication.principal.userId")`. Loading happens through the `UserDetails` implementation, which queries the role/permission tables and flattens them into a single authority list. The JWT can carry the permission strings so resource-server-side checks stay stateless. Permission-based access is more granular and easier to reason about for feature flags and capabilities, but it increases administrative complexity because permissions must be mapped to roles carefully. Many production systems use both models: roles for coarse assignment and administration, permissions for fine-grained enforcement, with `hasRole`/`hasAuthority` used accordingly.

41. How do you secure APIs behind a load balancer?
    - **Answer:** When an application runs behind a load balancer, reverse proxy, or TLS terminator, the requests Spring Security sees arrive from the proxy rather than the client, and scheme and address information arrive via standard forwarded headers. The application must be configured to trust those headers, or features that depend on the client's IP or the original scheme (HTTPS redirects, IP-based access rules, `server.forward-headers-strategy`) silently break. Spring Boot handles this with `server.forward-headers-strategy=NATIVE`, which enables the Servlet container's native forwarded-header handling (`ForwardedHeaderFilter` in Spring, or equivalent in Tomcat), so `X-Forwarded-Proto`, `X-Forwarded-For`, and `X-Forwarded-Host` are honored:

    ```yaml
    server:
      forward-headers-strategy: native
    ```

    Only the trusted proxy addresses should be allowed to set forwarded headers; otherwise a client could spoof `X-Forwarded-For` to bypass IP-whitelist rules. Spring Security additionally offers `RemoteIpFilter` / `RemoteIpValve` to normalize forwarded addresses against a trusted-proxy list. Authorization rules then behave correctly behind the proxy: `hasIpAddress()` matches the real client IP, HTTPS enforcement works, and generated URLs use the public scheme and host. It is also important that the proxy forwards the original `Host` or the configured `X-Forwarded-Host`, so redirects and CORS processing see the public domain rather than an internal one. Load balancer health checks should hit endpoints that do not trigger authentication failures that pollute logs, and timeouts must be consistent with token validation times. Correct forwarded-header handling is a prerequisite for reliable security decisions in any proxied deployment.

42. How do you handle `Authorization` header through proxies?
    - **Answer:** Proxies and load balancers must transparently forward the `Authorization` header so the JWT or bearer token reaches the application unchanged. Most proxies pass arbitrary headers by default, but some — especially caching layers, WAFs, or custom gateways — strip `Authorization` for security reasons, which silently breaks authentication with a confusing 401. The solution is explicit configuration: the proxy must be told to whitelist the `Authorization` header and must not log or rewrite it. On the application side, Spring Security reads the header via `BearerTokenAuthenticationFilter` (or a custom JWT filter), so no code change is needed once the header arrives. Debugging header loss is straightforward with request logging, for example registering a filter that logs which headers actually arrived, or using `curl -v` against the public URL to compare before and after the proxy. The `Authorization` header must also be forwarded with the original case — HTTP headers are case-insensitive, but proxies must not normalize the value or strip `Bearer `. For high-security setups, alternative credential transport such as mTLS client certificates can replace header-based auth entirely, at the cost of much more operational complexity. API keys and custom headers like `X-API-Key` suffer the same stripping problem, so they need the same whitelisting. The golden rule is to verify the header end-to-end with real requests through the proxy, since this class of bug is invisible when testing locally.

43. What is session fixation?
    - **Answer:** Session fixation is an attack in which an attacker pre-selects a session ID, forces the victim to use it (for example by sending a link that plants a cookie), and then waits for the victim to authenticate — after which the attacker, knowing the session ID, can hijack the authenticated session. The root cause is a server that keeps the same session ID across the login transition: the attacker's known ID becomes an authenticated ID. Spring Security prevents this by default: upon successful authentication, it invalidates the existing session and issues a fresh session ID, a behavior controlled by `sessionFixation()` in the session-management configuration — strategies include `newSession()` (default), `migrateSession()` (keeps attributes, new ID), and `changeSessionId()` (keeps attributes, changes ID only). For a stateful application the configuration is:

    ```java
    http.sessionManagement(sm -> sm
        .sessionFixation(SessionFixationConfigurer::newSession)
        .maximumSessions(1));
    ```

    The `migrateSession` and `changeSessionId` variants preserve session attributes while rotating the ID, which is what applications need when data is stored in the session. JWT-based stateless applications are inherently immune to session fixation because there is no server-side session and no cookie to hijack — each request is authenticated from the bearer token. Additional mitigations include logging in over HTTPS, setting the `Secure` attribute on cookies, and expiring sessions on logout. The defensive principle is that the session ID must change whenever privilege changes, so that any ID an attacker knows becomes worthless after login.

44. What is stateless session management?
    - **Answer:** Stateless session management means the server stores no per-user session state between requests; every request is authenticated independently from the information it carries, typically a JWT in the `Authorization` header. In Spring Security it is configured with `SessionCreationPolicy.STATELESS`, which tells the framework never to create or use an HTTP session for the request — even if a session cookie is presented, it is ignored for security purposes:

    ```java
    http.sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
    ```

    The `SessionCreationPolicy` options are `ALWAYS` (create a session if none exists), `IF_REQUIRED` (create only when needed, the default), `NEVER` (never create, but use one if present), and `STATELESS` (never create or use). Statelessness yields major operational benefits: any instance of a horizontally scaled service can serve any request, so there is no need for sticky sessions; servers can be restarted and scaled freely without losing user sessions; and the memory footprint is lower because nothing is stored. The trade-off is control: there is no server-side record to query, so tokens cannot be immediately invalidated, session state cannot be queried, and features like per-user session lists or concurrent-login limits are harder to implement. Revocation therefore relies on the techniques from questions 19 and 20 — short expiry, refresh-token revocation, and denylists. Stateless authentication is the standard for modern REST APIs and microservices because it scales horizontally and keeps each service independently verifiable.

45. How do you mix stateful UI and stateless API security?
    - **Answer:** Applications that serve both browser-rendered pages and a JSON API typically run two security models side by side: session-cookie authentication for the UI, and stateless JWT or token authentication for the API. Spring Security supports this cleanly with multiple `SecurityFilterChain` beans, each matched to a different path and carrying its own authentication strategy. The UI chain uses form login and session-based authentication (with CSRF enabled and session fixation protection), while the API chain uses bearer-token authentication and `SessionCreationPolicy.STATELESS` with CSRF disabled:

    ```java
    @Bean @Order(1)
    SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/api/**")
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .build();
    }

    @Bean @Order(2)
    SecurityFilterChain uiChain(HttpSecurity http) throws Exception {
        return http
            .formLogin(Customizer.withDefaults())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login").permitAll()
                .anyRequest().authenticated())
            .build();
    }
    ```

    The `@Order` annotation and `securityMatcher` decide which chain handles which request; the first matching chain wins, so more specific paths (the API) come first. On the frontend, a common pattern is a SPA that authenticates through the API with JWT, storing the access token in memory and the refresh token in an httpOnly cookie, so the browser UI and API are governed by the same stateless token flow while classic server-rendered pages keep their sessions. Stateful and stateless parts can even share the same user store: the UI session is established after a login that also issues the API token. The key is keeping the two chains' configuration independent and explicit, and documenting which paths use which model so future security changes cannot silently break one side.
