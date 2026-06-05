# OAuth 2.0

## 1. Executive Summary

OAuth 2.0 is an authorization framework that enables third-party applications to obtain limited access to services on behalf of a resource owner. It decouples authentication from authorization by introducing an authorization layer and separating the role of the client from the resource owner. OAuth 2.0 is the industry standard for delegated access, used by Google, Facebook, GitHub, and virtually every major platform. It defines multiple grant types for different use cases including web applications, mobile apps, server-to-server communication, and IoT devices.

## 2. Core Theory

### Roles in OAuth 2.0

- **Resource Owner**: The user who authorizes access to their data.
- **Client**: The application requesting access (web app, mobile app, server).
- **Authorization Server**: Issues tokens after successful authentication.
- **Resource Server**: Hosts protected data and validates tokens.

### Authorization Flow

```
+--------+                               +---------------+
|        |--(A)- Authorization Request ->|   Resource    |
|        |                               |     Owner     |
|        |<-(B)-- Authorization Grant ---|               |
|        |                               +---------------+
|        |                               +---------------+
|        |--(C)-- Authorization Grant -->| Authorization |
| Client |                               |     Server    |
|        |<-(D)----- Access Token -------|               |
|        |                               +---------------+
|        |                               +---------------+
|        |--(E)----- Access Token ------>|    Resource   |
|        |                               |     Server    |
|        |<-(F)--- Protected Resource ---|               |
+--------+                               +---------------+
```

### Grant Types

| Grant Type | Use Case | Security |
|------------|----------|----------|
| Authorization Code | Web apps with backend | High (PKCE for mobile) |
| Implicit (deprecated) | Single-page apps | Low (removed in OAuth 2.1) |
| Client Credentials | Server-to-server | High |
| Resource Owner Password Credentials | Legacy/trusted apps | Medium |
| Device Code | Smart TVs, IoT devices | Medium |
| Refresh Token | Obtaining new access tokens | High |

## 3. Under-the-Hood Deep Dive

### Authorization Code Grant (Detailed)

1. Client redirects user to Authorization Server:
   ```
   GET /authorize?response_type=code&client_id=CLIENT_ID
       &redirect_uri=https://client.example.com/callback
       &scope=openid%20profile%20email
       &state=STATE_CSRF_TOKEN
   ```

2. User authenticates and consents.

3. Authorization Server redirects to client's redirect URI:
   ```
   GET /callback?code=AUTHORIZATION_CODE&state=STATE_CSRF_TOKEN
   ```

4. Client exchanges code for tokens (server-side):
   ```
   POST /token
   Content-Type: application/x-www-form-urlencoded
   
   grant_type=authorization_code&code=AUTHORIZATION_CODE
   &redirect_uri=https://client.example.com/callback
   &client_id=CLIENT_ID&client_secret=CLIENT_SECRET
   ```

5. Authorization Server responds:
   ```json
   {
     "access_token": "eyJhbGciOiJSUzI1NiIs...",
     "token_type": "Bearer",
     "expires_in": 3600,
     "refresh_token": "4d6f6f6e...",
     "id_token": "eyJraWQiOiIxZTlnZGs3..."
   }
   ```

### PKCE (Proof Key for Code Exchange)

PKCE protects against authorization code interception attacks:

```java
// Client generates:
String codeVerifier = generateRandomString(128);
String codeChallenge = base64URLEncode(sha256(codeVerifier));

// Authorization request includes code_challenge
GET /authorize?response_type=code&client_id=CLIENT_ID
    &code_challenge_method=S256
    &code_challenge=CODE_CHALLENGE_HASH

// Token request includes code_verifier
POST /token
grant_type=authorization_code&code=CODE
&code_verifier=CODE_VERIFIER_ORIGINAL
&client_id=CLIENT_ID
```

## 4. Production Code Examples

### Spring Boot OAuth2 Client Configuration

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user, user:email
          custom:
            provider: custom-provider
            client-id: ${CUSTOM_CLIENT_ID}
            client-secret: ${CUSTOM_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: openid, profile, api:read
            client-name: Custom OAuth2 Provider
        provider:
          custom-provider:
            authorization-uri: https://auth.example.com/oauth2/authorize
            token-uri: https://auth.example.com/oauth2/token
            user-info-uri: https://api.example.com/userinfo
            user-name-attribute: sub
            jwk-set-uri: https://auth.example.com/.well-known/jwks.json
```

### OAuth2 Resource Server

```java
@Configuration
@EnableWebSecurity
public class OAuth2ResourceServerConfig {

    @Bean
    public SecurityFilterChain resourceServerFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.GET, "/api/v1/public/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/users").hasAuthority("SCOPE_profile")
                .requestMatchers("/api/v1/admin/**").hasAuthority("SCOPE_admin")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
                .authenticationEntryPoint((request, response, authException) -> {
                    response.setContentType("application/json");
                    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    response.getWriter().write(
                        "{\"error\":\"unauthorized\",\"message\":\"" +
                        authException.getMessage() + "\"}");
                })
            )
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthorities = new JwtGrantedAuthoritiesConverter();
        grantedAuthorities.setAuthorityPrefix("SCOPE_");
        grantedAuthorities.setAuthoritiesClaimName("scope");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthorities);
        return converter;
    }
}
```

### Custom Token Validation with JWK Set

```java
@Configuration
public class JwtConfig {

    @Value("${spring.security.oauth2.resourceserver.jwt.issuer-uri}")
    private String issuerUri;

    @Bean
    public ReactiveJwtDecoder jwtDecoder() {
        return NimbusReactiveJwtDecoder.withJwkSetUri(
                issuerUri + "/.well-known/jwks.json")
            .jwsAlgorithm(SignatureAlgorithm.RS256)
            .build();
    }
}

// Custom JWT validator
@Component
public class CustomJwtValidator implements ReactiveOAuth2TokenValidator<Jwt> {

    private static final String EXPECTED_AUDIENCE = "api.example.com";

    @Override
    public Mono<OAuth2TokenValidatorResult> validate(Jwt jwt) {
        List<OAuth2Error> errors = new ArrayList<>();

        // Validate audience
        if (!jwt.getAudience().contains(EXPECTED_AUDIENCE)) {
            errors.add(new OAuth2Error("invalid_audience",
                "Audience does not match", null));
        }

        // Validate issuer
        if (!"https://auth.example.com".equals(jwt.getIssuer().toString())) {
            errors.add(new OAuth2Error("invalid_issuer",
                "Issuer mismatch", null));
        }

        // Validate token type
        if (!"Bearer".equals(jwt.getClaimAsString("token_type"))) {
            errors.add(new OAuth2Error("invalid_token_type",
                "Token type must be Bearer", null));
        }

        if (errors.isEmpty()) {
            return Mono.just(OAuth2TokenValidatorResult.success());
        }
        return Mono.just(OAuth2TokenValidatorResult.failure(errors));
    }
}
```

### Authorization Code Flow with Spring Security

```java
@Configuration
@EnableWebSecurity
public class OAuth2LoginConfig {

    @Bean
    public SecurityFilterChain loginFilterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/oauth2/authorization/my-oauth2-provider")
                .authorizationEndpoint(auth -> auth
                    .authorizationRequestResolver(
                        customAuthorizationRequestResolver())
                )
                .tokenEndpoint(token -> token
                    .accessTokenResponseClient(
                        customAccessTokenResponseClient())
                )
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService())
                )
                .successHandler((request, response, authentication) -> {
                    OAuth2User oAuth2User = (OAuth2User) authentication.getPrincipal();
                    // Create or update local user
                    userService.syncUser(oAuth2User);
                    response.sendRedirect("/dashboard");
                })
                .failureHandler((request, response, exception) -> {
                    response.sendRedirect("/login?error=" +
                        exception.getMessage());
                })
            )
            .build();
    }

    private OAuth2AuthorizationRequestResolver customAuthorizationRequestResolver() {
        DefaultOAuth2AuthorizationRequestResolver resolver =
            new DefaultOAuth2AuthorizationRequestResolver(
                clientRegistrationRepository,
                OAuth2AuthorizationRequestRedirectFilter
                    .DEFAULT_AUTHORIZATION_REQUEST_BASE_URI);
        resolver.setAuthorizationRequestCustomizer(customizer -> {
            customizer.additionalParameters(params -> {
                params.put("access_type", "offline");
                params.put("prompt", "consent");
            });
        });
        return resolver;
    }
}
```

### OAuth2 Client Credentials Grant

```java
@Service
public class ClientCredentialsService {

    private final OAuth2AuthorizedClientManager authorizedClientManager;

    public ClientCredentialsService(
            OAuth2AuthorizedClientManager authorizedClientManager) {
        this.authorizedClientManager = authorizedClientManager;
    }

    public String getAccessToken() {
        OAuth2AuthorizeRequest authorizeRequest = OAuth2AuthorizeRequest
            .withClientRegistrationId("my-client")
            .principal(new AnonymousAuthenticationToken(
                "anonymous", "system",
                List.of(new SimpleGrantedAuthority("ROLE_SYSTEM"))))
            .build();

        OAuth2AuthorizedClient authorizedClient =
            authorizedClientManager.authorize(authorizeRequest);

        return authorizedClient.getAccessToken().getTokenValue();
    }

    public String callApiWithClientCredentials() {
        String token = getAccessToken();

        RestClient restClient = RestClient.builder()
            .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + token)
            .build();

        return restClient.get()
            .uri("https://api.example.com/protected/resource")
            .retrieve()
            .body(String.class);
    }
}
```

### OAuth2 Client with RestClient

```java
@Configuration
public class OAuth2ClientConfig {

    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrationRepository,
            OAuth2AuthorizedClientRepository authorizedClientRepository) {

        OAuth2AuthorizedClientProvider authorizedClientProvider =
            OAuth2AuthorizedClientProviderBuilder.builder()
                .authorizationCode()
                .refreshToken()
                .clientCredentials()
                .password()
                .build();

        DefaultOAuth2AuthorizedClientManager authorizedClientManager =
            new DefaultOAuth2AuthorizedClientManager(
                clientRegistrationRepository, authorizedClientRepository);
        authorizedClientManager.setAuthorizedClientProvider(
            authorizedClientProvider);

        return authorizedClientManager;
    }

    @Bean
    public RestClient restClient(
            OAuth2AuthorizedClientManager authorizedClientManager) {
        return RestClient.builder()
            .requestInterceptor((request, body, execution) -> {
                OAuth2AuthorizeRequest authorizeRequest = OAuth2AuthorizeRequest
                    .withClientRegistrationId("my-client")
                    .principal("system")
                    .build();
                OAuth2AuthorizedClient client =
                    authorizedClientManager.authorize(authorizeRequest);
                request.getHeaders().setBearerAuth(
                    client.getAccessToken().getTokenValue());
                return execution.execute(request, body);
            })
            .build();
    }
}
```

### Refresh Token Handling

```java
@Component
public class RefreshTokenService {

    private final OAuth2AuthorizedClientService authorizedClientService;

    public RefreshTokenService(
            OAuth2AuthorizedClientService authorizedClientService) {
        this.authorizedClientService = authorizedClientService;
    }

    public OAuth2AccessToken refreshAccessToken(
            String clientRegistrationId,
            Principal principal) {

        OAuth2AuthorizedClient client = authorizedClientService
            .loadAuthorizedClient(clientRegistrationId, principal.getName());

        if (client == null || client.getRefreshToken() == null) {
            throw new OAuth2AuthorizationException(
                "No refresh token available");
        }

        // The refresh is handled automatically by
        // OAuth2AuthorizedClientProvider when using
        // OAuth2AuthorizedClientManager
        return client.getAccessToken();
    }
}
```

### Custom OAuth2 User Service

```java
@Component
public class CustomOAuth2UserService
        extends DefaultOAuth2UserService {

    private final UserService userService;

    public CustomOAuth2UserService(UserService userService) {
        this.userService = userService;
    }

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest)
            throws OAuth2AuthenticationException {

        OAuth2User oAuth2User = super.loadUser(userRequest);

        try {
            return processOAuth2User(userRequest, oAuth2User);
        } catch (Exception ex) {
            throw new OAuth2AuthenticationException(
                "Failed to process user: " + ex.getMessage());
        }
    }

    private OAuth2User processOAuth2User(
            OAuth2UserRequest userRequest,
            OAuth2User oAuth2User) {

        String registrationId = userRequest
            .getClientRegistration().getRegistrationId();
        String email = oAuth2User.getAttribute("email");
        String name = oAuth2User.getAttribute("name");

        Optional<User> existingUser = userService.findByEmail(email);

        User user;
        if (existingUser.isPresent()) {
            user = existingUser.get();
            user.setLastLogin(Instant.now());
            user = userService.update(user);
        } else {
            user = userService.createOAuth2User(
                email, name, registrationId);
        }

        return new DefaultOAuth2User(
            List.of(new SimpleGrantedAuthority("ROLE_USER")),
            oAuth2User.getAttributes(),
            "email");
    }
}
```

### Token Revocation

```java
@Component
public class TokenRevocationService {

    private final RestClient restClient;

    public TokenRevocationService() {
        this.restClient = RestClient.builder()
            .baseUrl("https://auth.example.com")
            .build();
    }

    public void revokeAccessToken(String token, String clientId,
                                 String clientSecret) {
        String body = "token=" + URLEncoder.encode(token, StandardCharsets.UTF_8) +
            "&token_type_hint=access_token" +
            "&client_id=" + URLEncoder.encode(clientId, StandardCharsets.UTF_8) +
            "&client_secret=" + URLEncoder.encode(clientSecret, StandardCharsets.UTF_8);

        restClient.post()
            .uri("/oauth2/revoke")
            .header(HttpHeaders.CONTENT_TYPE,
                MediaType.APPLICATION_FORM_URLENCODED_VALUE)
            .body(body)
            .retrieve()
            .toBodilessEntity();
    }

    public void revokeRefreshToken(String token, String clientId,
                                  String clientSecret) {
        String body = "token=" + URLEncoder.encode(token, StandardCharsets.UTF_8) +
            "&token_type_hint=refresh_token" +
            "&client_id=" + URLEncoder.encode(clientId, StandardCharsets.UTF_8) +
            "&client_secret=" + URLEncoder.encode(clientSecret, StandardCharsets.UTF_8);

        restClient.post()
            .uri("/oauth2/revoke")
            .header(HttpHeaders.CONTENT_TYPE,
                MediaType.APPLICATION_FORM_URLENCODED_VALUE)
            .body(body)
            .retrieve()
            .toBodilessEntity();
    }
}
```

### Custom Access Token Response Client

```java
@Component
public class CustomAccessTokenResponseClient
        implements OAuth2AccessTokenResponseClient<OAuth2AuthorizationCodeGrantRequest> {

    private final RestClient restClient;

    public CustomAccessTokenResponseClient() {
        this.restClient = RestClient.create();
    }

    @Override
    public OAuth2AccessTokenResponse getTokenResponse(
            OAuth2AuthorizationCodeGrantRequest request) {

        ClientRegistration registration = request.getClientRegistration();

        String body = "grant_type=authorization_code" +
            "&code=" + request.getAuthorizationExchange()
                .getAuthorizationResponse().getCode() +
            "&redirect_uri=" + registration.getRedirectUri() +
            "&client_id=" + registration.getClientId() +
            "&client_secret=" + registration.getClientSecret();

        if (request.getAuthorizationExchange()
                .getAuthorizationRequest()
                .getAdditionalParameters()
                .containsKey("code_verifier")) {
            body += "&code_verifier=" + request.getAuthorizationExchange()
                .getAuthorizationRequest()
                .getAdditionalParameters().get("code_verifier");
        }

        TokenResponse tokenResponse = restClient.post()
            .uri(registration.getProviderDetails().getTokenUri())
            .header(HttpHeaders.CONTENT_TYPE,
                MediaType.APPLICATION_FORM_URLENCODED_VALUE)
            .body(body)
            .retrieve()
            .body(TokenResponse.class);

        return OAuth2AccessTokenResponse
            .withToken(tokenResponse.accessToken())
            .tokenType(OAuth2AccessToken.TokenType.BEARER)
            .expiresIn(tokenResponse.expiresIn())
            .refreshToken(tokenResponse.refreshToken())
            .scopes(Set.of(tokenResponse.scope().split(" ")))
            .build();
    }

    private record TokenResponse(
        String accessToken,
        long expiresIn,
        String refreshToken,
        String scope,
        String tokenType
    ) {}
}
```

## 5. Real-World Scenarios

### Scenario 1: Social Login Integration

```
User clicks "Login with Google"
  -> Redirect to Google OAuth2
  -> User consents
  -> Google redirects back with code
  -> Backend exchanges code for tokens
  -> Backend fetches user info from Google
  -> Local user is created/updated
  -> JWT session token is issued
```

### Scenario 2: Service-to-Service Authentication

```
Service A (Payment Service)
  -> Client Credentials Grant -> Auth Server
  <- Access Token
  -> API call with Bearer token -> Service B (Order Service)
  -> Service B validates token -> Resource Server config
  -> Authorized request processed
```

### Scenario 3: Mobile App with PKCE

```
Mobile App
  -> Generates code_verifier (128 chars random)
  -> Computes code_challenge = SHA256(code_verifier)
  -> Opens browser for authorization with code_challenge
  -> Authorization Server redirects with code
  -> App exchanges code + code_verifier for tokens
  -> Stores access_token and refresh_token securely
```

## 6. Performance

### Token Validation Performance

- **Local JWT Validation**: Fastest (no network call), validate signature with JWK.
- **Remote Token Introspection**: Slower (network call), but allows immediate revocation.
- **Hybrid**: Cache introspection results for performance.

### Token Caching

```java
@Component
public class CachedTokenIntrospector {

    private final CacheManager cacheManager;

    public CachedTokenIntrospector(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    public OAuth2AuthenticatedPrincipal introspect(String token) {
        Cache cache = cacheManager.getCache("token-introspection");
        Cache.ValueWrapper cached = cache.get(token);

        if (cached != null) {
            return (OAuth2AuthenticatedPrincipal) cached.get();
        }

        OAuth2AuthenticatedPrincipal principal = doIntrospect(token);
        cache.put(token, principal);
        return principal;
    }

    private OAuth2AuthenticatedPrincipal doIntrospect(String token) {
        // Call introspection endpoint
        // ...
    }
}
```

## 7. Security

### OAuth 2.0 Security Best Practices

- **Always use PKCE** for mobile and public clients.
- **Use short-lived access tokens** (15-60 minutes).
- **Use refresh tokens** for long-lived sessions.
- **Store client secrets securely** (environment variables, vault).
- **Validate redirect URIs** strictly.
- **Use HTTPS** for all endpoints.
- **Implement CSRF protection** for authorization callback.
- **Rotate refresh tokens** on each use (refresh token rotation).
- **Sender-Constrained Tokens**: Use DPoP (Demonstration of Proof-of-Possession) or mTLS.

### Common OAuth 2.0 Attacks

| Attack | Prevention |
|--------|------------|
| Authorization Code Interception | PKCE |
| CSRF on redirect | State parameter (anti-CSRF token) |
| Redirect URI manipulation | Strict redirect URI validation |
| Open Redirector | Validate redirect URIs |
| Token Theft | Short expiry, refresh token rotation |
| Client Impersonation | Client authentication (secret/certificate) |

## 8. Common Mistakes

- **Not using PKCE for public clients**: Mobile and SPA apps must use PKCE.
- **Storing tokens insecurely**: Use httpOnly cookies or secure storage.
- **Long-lived access tokens**: Use short expiry + refresh tokens.
- **Not validating redirect URIs**: Can lead to open redirect vulnerabilities.
- **Missing state parameter**: Vulnerable to CSRF attacks.
- **Hardcoded client secrets**: Use environment variables or vaults.
- **Not refreshing tokens proactively**: Implement token refresh before expiry.
- **Sharing access tokens across services**: Each service should use its own scope.
- **No token revocation**: Implement revoke endpoint for logout and compromise.

## 9. Senior Engineer Perspective

### OAuth 2.1 Improvements

OAuth 2.1 consolidates best practices from OAuth 2.0:
- PKCE is required for all public clients.
- Implicit grant is removed.
- Resource Owner Password Credentials grant is removed.
- Refresh tokens must be sender-constrained or rotate.
- Redirect URIs must use exact matching.

### Token Exchange and Delegation

```java
// Token Exchange (RFC 8693)
// Used for impersonation or delegation
POST /oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=ACCESS_TOKEN
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&requested_token_type=urn:ietf:params:oauth:token-type:access_token
&audience=https://api.downstream.example.com
```

### OAuth 2.0 in Microservices

```
API Gateway
  |-- Validates access token (JWT or introspection)
  |-- Passes token downstream
  |-- Downstream services validate token locally (JWT)
  |-- OR use token exchange for service-specific tokens
```

## 10. Interview Questions (Easy)

1. What is OAuth 2.0 and what problem does it solve?
2. What are the four roles in OAuth 2.0?
3. What is an access token?
4. What is a refresh token?
5. What is the difference between OAuth 2.0 and OpenID Connect?
6. What is the Authorization Code grant?
7. What is the Client Credentials grant?
8. What is the purpose of the redirect URI?
9. What is the state parameter in OAuth 2.0?
10. What is the difference between OAuth and basic authentication?

## Medium

1. What is PKCE and why is it needed?
2. How does the Authorization Code flow work step by step?
3. What is JWT and how does it relate to OAuth 2.0?
4. How do you implement token revocation?
5. What is the difference between opaque tokens and JWT access tokens?
6. How do you handle token refresh in a mobile app?
7. What is the difference between bearer tokens and sender-constrained tokens?
8. How does OAuth 2.0 work with Spring Security?
9. What are OAuth 2.0 scopes and how do they work?
10. What is the difference between authorization and authentication?

## 11. Advanced Interview Questions (Hard)

1. Design an OAuth 2.0 authorization server from scratch.
2. How would you implement refresh token rotation with automatic revocation of old tokens?
3. Design a multi-tenant OAuth 2.0 system where each tenant has its own identity provider.
4. How do you implement DPoP (Demonstration of Proof-of-Possession) for token binding?
5. Design an OAuth 2.0 token exchange system for service-to-service delegation.
6. How would you implement a custom grant type for IoT devices?
7. Design a cross-domain single sign-on (SSO) system using OAuth 2.0.
8. How do you handle OAuth 2.0 in a microservices architecture with an API gateway?
9. Implement a token introspection cache with automatic invalidation.
10. How would you migrate from OAuth 2.0 to OAuth 2.1?

## System Design

1. Design an OAuth 2.0-based authentication system for a SaaS platform.
2. Design a distributed OAuth 2.0 authorization server across multiple regions.
3. Design an OAuth 2.0 API gateway with centralized token validation.
4. Design a multi-provider OAuth 2.0 login system (Google, GitHub, Facebook, Apple).
5. Design an OAuth 2.0-based permission system for a collaborative document platform.
6. Design a token management system with automatic rotation and revocation.
7. Design an OAuth 2.0 authorization flow for a mobile app with biometric authentication.
8. Design a zero-trust architecture using OAuth 2.0 and mutual TLS.
9. Design an OAuth 2.0 system for a B2B API platform with customer-managed identities.
10. Design an OAuth 2.0-based delegated admin system for multi-tenant SaaS.

## 12. Expert-Level Interview Questions (Architect-Level)

1. Design a global OAuth 2.0/OpenID Connect identity platform supporting 100M+ users across 50 regions with sub-second token issuance latency.
2. How would you implement a dynamic client registration system with automatic scope discovery and consent management?
3. Design an OAuth 2.0-based capability-based security model for a distributed system with thousands of services.
4. How would you implement a token exchange protocol that supports both impersonation and delegation across organizational boundaries?
5. Design a continuous authentication system that extends OAuth 2.0 with risk-based step-up authentication.
6. How would you build a federated OAuth 2.0 system that bridges on-premise and cloud identity providers?
7. Design an OAuth 2.0 authorization system for a IoT platform with millions of devices using the device authorization grant.
8. How would you implement a real-time token revocation system that propagates revocations across all services within seconds?
9. Design an OAuth 2.0 audit and compliance system that tracks every authorization decision, token issuance, and API access.
10. How would you design a strategy for gradual OAuth 2.0 adoption across a legacy enterprise with existing session-based authentication?

## 13. Debugging & Troubleshooting

### Common Issues

- **Invalid grant**: Authorization code expired or already used.
- **Invalid redirect URI**: Redirect URI does not match registered URIs.
- **Invalid client**: Client ID or secret is incorrect.
- **Access denied**: User did not consent to requested scopes.
- **Token expired**: Access token has expired, need to refresh.
- **Invalid scope**: Requested scope is not registered for the client.
- **SSL errors**: Certificate validation failures in token endpoint calls.

### Debugging OAuth 2.0 Flows

```java
@Component
public class OAuth2DebugFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;

        if (httpRequest.getRequestURI().contains("oauth2")) {
            log.debug("OAuth2 request: {} {}",
                httpRequest.getMethod(), httpRequest.getRequestURI());
            Collections.list(httpRequest.getParameterNames())
                .forEach(name -> {
                    if (!name.contains("secret") && !name.contains("token")) {
                        log.debug("  param: {} = {}", name,
                            httpRequest.getParameter(name));
                    }
                });
        }

        chain.doFilter(request, response);
    }
}
```

## 14. Comparison Section

### OAuth 2.0 vs SAML 2.0

| Aspect | OAuth 2.0 | SAML 2.0 |
|--------|-----------|----------|
| Protocol | JSON/REST | XML/SOAP |
| Token Format | JWT or opaque | SAML Assertion (XML) |
| Use Case | Authorization & delegated access | Enterprise SSO |
| Mobile Support | Excellent | Poor |
| Modern Web | Yes | No |
| Complexity | Low | High |
| Standard | IETF RFCs | OASIS |

### OAuth 2.0 vs OpenID Connect (OIDC)

| Aspect | OAuth 2.0 | OIDC |
|--------|-----------|------|
| Purpose | Authorization | Authentication |
| Token | Access Token | ID Token (JWT) |
| User Info | Not defined | UserInfo endpoint |
| Profile | Authorization framework | Authentication layer on OAuth 2.0 |
| Standard | RFC 6749 | OpenID Foundation |

## 15. Revision Notes

- OAuth 2.0: delegated authorization framework
- 4 roles: Resource Owner, Client, Authorization Server, Resource Server
- Grant types: Authorization Code, Client Credentials, Device Code, Refresh Token
- PKCE: code_verifier + code_challenge for public clients
- Access tokens: short-lived, Bearer typically JWT
- Refresh tokens: long-lived, used to get new access tokens
- Scopes define what the client can access
- State parameter prevents CSRF attacks
- Spring Security: `@EnableWebSecurity`, `oauth2Login()`, `oauth2ResourceServer()`
- OAuth 2.1: PKCE required, Implicit removed, Password removed

## 16. Cheat Sheet

```
+------------------------------------------------------------------+
| OAUTH 2.0 CHEAT SHEET                                            |
+------------------------------------------------------------------+
| ROLES                                                            |
|   Resource Owner  -> User who owns the data                      |
|   Client          -> App requesting access                       |
|   Authorization   -> Issues tokens                               |
|   Server                                                         |
|   Resource Server -> Hosts protected data                        |
+------------------------------------------------------------------+
| GRANT TYPES                                                      |
|   Authorization Code  -> Web apps (with PKCE for mobile)         |
|   Client Credentials  -> Server-to-server                        |
|   Device Code         -> Smart TVs, IoT                          |
|   Refresh Token       -> Get new access tokens                   |
+------------------------------------------------------------------+
| TOKEN ENDPOINT PARAMETERS                                        |
|   grant_type: authorization_code | client_credentials | ...      |
|   code:         The authorization code                           |
|   redirect_uri: Must match authorization request                 |
|   client_id:    Client identifier                                |
|   client_secret: Client secret (confidential clients)            |
|   code_verifier: PKCE verifier (public clients)                  |
+------------------------------------------------------------------+
| SPRING BOOT CONFIGURATION KEYS                                   |
|   spring.security.oauth2.client.registration.*                  |
|   spring.security.oauth2.client.provider.*                      |
|   spring.security.oauth2.resourceserver.jwt.*                   |
+------------------------------------------------------------------+
| COMMON ENDPOINTS                                                 |
|   /oauth2/authorize    -- Authorization endpoint                 |
|   /oauth2/token        -- Token endpoint                         |
|   /oauth2/revoke       -- Revocation endpoint                    |
|   /oauth2/introspect   -- Introspection endpoint                 |
|   /.well-known/jwks.json -- JWK Set endpoint                    |
|   /.well-known/openid-configuration -- OIDC discovery           |
+------------------------------------------------------------------+
| SECURITY CHECKLIST                                               |
|   [ ] Use HTTPS everywhere                                       |
|   [ ] PKCE for all public clients                                |
|   [ ] Validate redirect URIs strictly                            |
|   [ ] Use state parameter for CSRF protection                    |
|   [ ] Short access token TTL (15-60 min)                        |
|   [ ] Rotate refresh tokens                                      |
|   [ ] Validate all token claims (iss, aud, exp, iat)            |
|   [ ] Store secrets in vault/env vars                           |
|   [ ] Implement token revocation                                |
+------------------------------------------------------------------+
```
