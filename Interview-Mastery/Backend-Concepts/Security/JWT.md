# JWT (JSON Web Tokens)

## 1. Executive Summary

JSON Web Token (JWT) is an open standard (RFC 7519) for securely transmitting information between parties as a JSON object. JWTs are digitally signed or encrypted, making them verifiable and tamper-proof. They are widely used for authentication (as access tokens in OAuth 2.0), information exchange, and session management. JWTs are stateless, containing all necessary user information in the token itself, which makes them ideal for distributed systems and microservices architectures.

## 2. Core Theory

### JWT Structure

A JWT consists of three parts separated by dots (`.`):

```
Header.Payload.Signature

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIn0.
sJ0yWzW_rF5m2q3OaP5P8KcX7iA
```

### Header

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-1"
}
```

### Payload (Claims)

```json
{
  "sub": "user123",
  "iss": "https://auth.example.com",
  "aud": "api.example.com",
  "exp": 1718200000,
  "iat": 1718196400,
  "nbf": 1718196400,
  "jti": "unique-token-id",
  "name": "John Doe",
  "email": "john@example.com",
  "roles": ["admin", "user"]
}
```

### Signature

The signature is computed as:

```
HMAC-SHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### JWT Types

| Type | Description | Use Case |
|------|-------------|----------|
| JWS (JSON Web Signature) | Signed JWT | Most common, ensures integrity |
| JWE (JSON Web Encryption) | Encrypted JWT | Sensitive data in payload |
| Nested JWT | Signed + Encrypted | Maximum security |

### Signing Algorithms

| Algorithm | Type | Key | Use Case |
|-----------|------|-----|----------|
| HS256 | Symmetric | Shared secret | Single service, internal |
| RS256 | Asymmetric | RSA key pair | Distributed services |
| ES256 | Asymmetric | ECDSA key pair | Better performance than RSA |
| PS256 | Asymmetric | RSA-PSS | Modern RSA alternative |
| EdDSA | Asymmetric | Ed25519 | High security, high performance |

## 3. Under-the-Hood Deep Dive

### JWT Validation Process

1. Parse the token (split by `.`).
2. Decode header (base64url).
3. Verify algorithm (reject `none` algorithm).
4. Fetch signing key (from JWKS endpoint or local store).
5. Verify signature using the key.
6. Validate claims:
   - `exp`: Must be in the future.
   - `nbf`: Must be in the past.
   - `iss`: Must match expected issuer.
   - `aud`: Must include expected audience.
   - `iat`: Reasonable timeframe.
7. Check if token is revoked (optional, for blacklisted tokens).

### JWK (JSON Web Key) Set

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "key-id-1",
      "n": "0vx7agoebGcQSuu...",
      "e": "AQAB",
      "alg": "RS256"
    }
  ]
}
```

### JWT vs Opaque Tokens

| Aspect | JWT | Opaque Token |
|--------|-----|--------------|
| Self-contained | Yes (all info in token) | No (reference to server-side data) |
| Validation | Local (signature check) | Remote (introspection endpoint) |
| Revocation | Hard (until expiry) | Easy (delete server-side) |
| Size | Larger (contains claims) | Small (random string) |
| Debugging | Easy (decode payload) | Hard (need introspection) |
| Performance | Fast (local validation) | Slower (network call) |

## 4. Production Code Examples

### JWT Token Provider

```java
@Component
public class JwtTokenProvider {

    private final PrivateKey privateKey;
    private final PublicKey publicKey;
    private final long accessTokenExpirationMs;
    private final long refreshTokenExpirationMs;

    public JwtTokenProvider(
            @Value("${jwt.private-key}") String privateKeyPem,
            @Value("${jwt.public-key}") String publicKeyPem,
            @Value("${jwt.access-token-expiration:3600000}") long accessExp,
            @Value("${jwt.refresh-token-expiration:604800000}") long refreshExp) {

        this.privateKey = PemUtils.parsePrivateKey(privateKeyPem);
        this.publicKey = PemUtils.parsePublicKey(publicKeyPem);
        this.accessTokenExpirationMs = accessExp;
        this.refreshTokenExpirationMs = refreshExp;
    }

    public String generateAccessToken(UserDetails userDetails) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + accessTokenExpirationMs);

        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuer("https://api.example.com")
            .setAudience("api.example.com")
            .setIssuedAt(now)
            .setNotBefore(now)
            .setExpiration(expiry)
            .setId(UUID.randomUUID().toString())
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()))
            .claim("scope", extractScopes(userDetails))
            .signWith(privateKey, SignatureAlgorithm.RS256)
            .compact();
    }

    public String generateRefreshToken(UserDetails userDetails) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + refreshTokenExpirationMs);

        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuer("https://api.example.com")
            .setIssuedAt(now)
            .setExpiration(expiry)
            .setId(UUID.randomUUID().toString())
            .claim("type", "refresh")
            .signWith(privateKey, SignatureAlgorithm.RS256)
            .compact();
    }

    public Claims validateToken(String token) {
        try {
            return Jwts.parserBuilder()
                .setSigningKey(publicKey)
                .requireIssuer("https://api.example.com")
                .build()
                .parseClaimsJws(token)
                .getBody();
        } catch (ExpiredJwtException e) {
            throw new TokenExpiredException("Token has expired", e);
        } catch (UnsupportedJwtException e) {
            throw new TokenValidationException("Token format not supported", e);
        } catch (MalformedJwtException e) {
            throw new TokenValidationException("Malformed token", e);
        } catch (SignatureException e) {
            throw new TokenValidationException("Invalid token signature", e);
        } catch (IllegalArgumentException e) {
            throw new TokenValidationException("Token is null or empty", e);
        }
    }

    public String getUsernameFromToken(String token) {
        return validateToken(token).getSubject();
    }

    public boolean isTokenExpired(String token) {
        try {
            Date expiration = Jwts.parserBuilder()
                .setSigningKey(publicKey)
                .build()
                .parseClaimsJws(token)
                .getBody()
                .getExpiration();
            return expiration.before(new Date());
        } catch (ExpiredJwtException e) {
            return true;
        }
    }

    private List<String> extractScopes(UserDetails userDetails) {
        if (userDetails instanceof OAuth2User oauth2User) {
            return oauth2User.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .filter(a -> a.startsWith("SCOPE_"))
                .map(a -> a.substring(6))
                .collect(Collectors.toList());
        }
        return List.of();
    }
}
```

### JWT Authentication Filter

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private static final String AUTHORIZATION_HEADER = "Authorization";
    private static final String BEARER_PREFIX = "Bearer ";

    private final JwtTokenProvider tokenProvider;
    private final UserDetailsService userDetailsService;
    private final TokenBlacklistService blacklistService;

    public JwtAuthenticationFilter(JwtTokenProvider tokenProvider,
                                  UserDetailsService userDetailsService,
                                  TokenBlacklistService blacklistService) {
        this.tokenProvider = tokenProvider;
        this.userDetailsService = userDetailsService;
        this.blacklistService = blacklistService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain filterChain)
            throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null) {
            try {
                // Check if token is blacklisted
                if (blacklistService.isBlacklisted(token)) {
                    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    response.getWriter().write("{\"error\":\"Token has been revoked\"}");
                    return;
                }

                Claims claims = tokenProvider.validateToken(token);
                String username = claims.getSubject();

                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(authentication);
            } catch (TokenValidationException e) {
                log.warn("JWT validation failed: {}", e.getMessage());
                SecurityContextHolder.clearContext();
            } catch (Exception e) {
                log.error("Authentication error", e);
                SecurityContextHolder.clearContext();
            }
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader(AUTHORIZATION_HEADER);
        if (bearerToken != null && bearerToken.startsWith(BEARER_PREFIX)) {
            return bearerToken.substring(BEARER_PREFIX.length());
        }
        return null;
    }
}
```

### Token Blacklist Service

```java
@Component
public class TokenBlacklistService {

    private final RedisTemplate<String, String> redisTemplate;

    public TokenBlacklistService(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void blacklist(String token, long expirationMs) {
        String jti = extractJti(token);
        if (jti != null) {
            redisTemplate.opsForValue()
                .set("blacklist:" + jti, "revoked",
                    Duration.ofMillis(expirationMs));
        }
    }

    public boolean isBlacklisted(String token) {
        String jti = extractJti(token);
        if (jti == null) {
            return false;
        }
        return Boolean.TRUE.equals(
            redisTemplate.hasKey("blacklist:" + jti));
    }

    private String extractJti(String token) {
        try {
            String[] parts = token.split("\\.");
            if (parts.length < 2) return null;
            byte[] payload = Base64.getUrlDecoder().decode(parts[1]);
            ObjectMapper mapper = new ObjectMapper();
            JsonNode claims = mapper.readTree(payload);
            JsonNode jti = claims.get("jti");
            return jti != null ? jti.asText() : null;
        } catch (Exception e) {
            return null;
        }
    }
}
```

### Refresh Token Rotation

```java
@Service
public class RefreshTokenService {

    private final JwtTokenProvider tokenProvider;
    private final RefreshTokenRepository refreshTokenRepository;

    public RefreshTokenService(JwtTokenProvider tokenProvider,
                              RefreshTokenRepository refreshTokenRepository) {
        this.tokenProvider = tokenProvider;
        this.refreshTokenRepository = refreshTokenRepository;
    }

    @Transactional
    public TokenPair refreshAccessToken(String refreshToken) {
        // Validate the refresh token
        Claims claims = tokenProvider.validateToken(refreshToken);

        // Check refresh token type claim
        if (!"refresh".equals(claims.get("type"))) {
            throw new TokenValidationException("Invalid token type");
        }

        String username = claims.getSubject();

        // Find the stored refresh token
        RefreshTokenEntity stored = refreshTokenRepository
            .findByTokenId(claims.getId())
            .orElseThrow(() -> new TokenValidationException(
                "Refresh token not found"));

        // Check if already used (rotation detects theft)
        if (stored.isUsed()) {
            // Token reuse detected! Revoke all tokens for this user
            refreshTokenRepository.revokeAllForUser(username);
            throw new TokenReuseException(
                "Refresh token reuse detected. All tokens revoked.");
        }

        // Mark as used
        stored.setUsed(true);
        refreshTokenRepository.save(stored);

        // Generate new token pair
        UserDetails userDetails = userDetailsService.loadUserByUsername(username);
        String newAccessToken = tokenProvider.generateAccessToken(userDetails);
        String newRefreshToken = tokenProvider.generateRefreshToken(userDetails);

        // Store new refresh token
        String newJti = tokenProvider.getJtiFromToken(newRefreshToken);
        refreshTokenRepository.save(new RefreshTokenEntity(
            newJti, username, false,
            tokenProvider.getExpirationDate(newRefreshToken)));

        return new TokenPair(newAccessToken, newRefreshToken);
    }

    public record TokenPair(String accessToken, String refreshToken) {}
}
```

### Multi-Key JWT (Key Rotation)

```java
@Component
public class RotatingKeyProvider {

    private final Map<String, KeyPair> keyPairs = new ConcurrentHashMap<>();
    private String activeKeyId;

    public RotatingKeyProvider() {
        // Generate initial key
        rotateKeys();
    }

    public void rotateKeys() {
        KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
        generator.initialize(2048);
        KeyPair newPair = generator.generateKeyPair();
        String newKeyId = UUID.randomUUID().toString();

        // Keep previous key for validation of existing tokens
        if (activeKeyId != null) {
            // Keep up to 2 old keys for grace period
            if (keyPairs.size() > 2) {
                String oldestKey = keyPairs.keySet().iterator().next();
                keyPairs.remove(oldestKey);
            }
        }

        keyPairs.put(newKeyId, newPair);
        this.activeKeyId = newKeyId;
    }

    public PrivateKey getActivePrivateKey() {
        return keyPairs.get(activeKeyId).getPrivate();
    }

    public String getActiveKeyId() {
        return activeKeyId;
    }

    public PublicKey getPublicKey(String kid) {
        KeyPair pair = keyPairs.get(kid);
        return pair != null ? pair.getPublic() : null;
    }

    public List<JWK> getJWKSet() {
        return keyPairs.entrySet().stream()
            .map(entry -> {
                RSAPublicKey pubKey = (RSAPublicKey)
                    entry.getValue().getPublic();
                return new RSAKey.Builder(pubKey)
                    .keyID(entry.getKey())
                    .algorithm(JWSAlgorithm.RS256)
                    .build();
            })
            .collect(Collectors.toList());
    }
}
```

### JWT with Refresh Token Rotation and Device Tracking

```java
@Entity
@Table(name = "refresh_tokens")
public class RefreshTokenEntity {

    @Id
    private String tokenId;

    @Column(nullable = false)
    private String username;

    @Column(nullable = false)
    private boolean used;

    @Column(nullable = false)
    private Instant expiresAt;

    @Column
    private String deviceId;

    @Column
    private String ipAddress;

    @Column
    private String userAgent;

    @Column
    private Instant createdAt;

    @Column
    private Instant lastUsedAt;

    @Column
    private int reuseCount;

    @PrePersist
    public void prePersist() {
        this.createdAt = Instant.now();
    }
}
```

### JWKS Endpoint

```java
@RestController
public class JwksController {

    private final RotatingKeyProvider keyProvider;

    public JwksController(RotatingKeyProvider keyProvider) {
        this.keyProvider = keyProvider;
    }

    @GetMapping("/.well-known/jwks.json")
    public ResponseEntity<JWKSet> getJwks() {
        JWKSet jwkSet = new JWKSet(keyProvider.getJWKSet());
        return ResponseEntity.ok(jwkSet);
    }
}
```

### Token Claims Enricher

```java
@Component
public class TokenClaimsEnricher {

    private final UserPreferencesService preferencesService;
    private final PermissionService permissionService;

    public TokenClaimsEnricher(UserPreferencesService preferencesService,
                              PermissionService permissionService) {
        this.preferencesService = preferencesService;
        this.permissionService = permissionService;
    }

    public Map<String, Object> enrichClaims(String username) {
        Map<String, Object> claims = new HashMap<>();

        // Add user permissions
        claims.put("permissions", permissionService
            .getPermissions(username));

        // Add user preferences
        UserPreferences preferences = preferencesService
            .getPreferences(username);
        claims.put("locale", preferences.getLocale());
        claims.put("timezone", preferences.getTimezone());

        // Add feature flags
        claims.put("features", featureFlagService
            .getEnabledFeatures(username));

        // Add metadata
        claims.put("auth_time", Instant.now().getEpochSecond());
        claims.put("auth_method", "password");

        return claims;
    }
}
```

### HMAC JWT (Symmetric)

```java
@Component
public class HmacJwtTokenProvider {

    private final SecretKey secretKey;
    private final long expirationMs;

    public HmacJwtTokenProvider(
            @Value("${jwt.secret}") String secret,
            @Value("${jwt.expiration:3600000}") long expirationMs) {

        byte[] keyBytes = secret.getBytes(StandardCharsets.UTF_8);
        this.secretKey = new SecretKeySpec(keyBytes, "HmacSHA256");
        this.expirationMs = expirationMs;
    }

    public String generateToken(String subject, Map<String, Object> claims) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + expirationMs);

        JwtBuilder builder = Jwts.builder()
            .setSubject(subject)
            .setIssuedAt(now)
            .setExpiration(expiry)
            .signWith(secretKey, SignatureAlgorithm.HS256);

        claims.forEach(builder::claim);

        return builder.compact();
    }

    public Claims validateToken(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(secretKey)
            .build()
            .parseClaimsJws(token)
            .getBody();
    }
}
```

### Logout Handler

```java
@Component
public class JwtLogoutHandler implements LogoutHandler {

    private final TokenBlacklistService blacklistService;
    private final JwtTokenProvider tokenProvider;

    @Override
    public void logout(HttpServletRequest request,
                      HttpServletResponse response,
                      Authentication authentication) {
        String token = extractToken(request);
        if (token != null) {
            try {
                Claims claims = tokenProvider.validateToken(token);
                long ttl = claims.getExpiration().getTime() -
                    System.currentTimeMillis();
                if (ttl > 0) {
                    blacklistService.blacklist(token, ttl);
                }
            } catch (Exception e) {
                log.warn("Could not blacklist token on logout", e);
            }
        }

        SecurityContextHolder.clearContext();
    }

    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### JWT Cookie Strategy

```java
@Component
public class JwtCookieManager {

    private final String accessTokenCookieName = "access_token";
    private final String refreshTokenCookieName = "refresh_token";

    public ResponseCookie createAccessTokenCookie(String token, long maxAge) {
        return ResponseCookie.from(accessTokenCookieName, token)
            .httpOnly(true)
            .secure(true)
            .sameSite("Strict")
            .path("/api")
            .maxAge(Duration.ofMillis(maxAge))
            .domain("example.com")
            .build();
    }

    public ResponseCookie createRefreshTokenCookie(String token, long maxAge) {
        return ResponseCookie.from(refreshTokenCookieName, token)
            .httpOnly(true)
            .secure(true)
            .sameSite("Strict")
            .path("/auth")
            .maxAge(Duration.ofMillis(maxAge))
            .domain("example.com")
            .build();
    }

    public ResponseCookie clearAccessTokenCookie() {
        return ResponseCookie.from(accessTokenCookieName, "")
            .httpOnly(true)
            .secure(true)
            .sameSite("Strict")
            .path("/api")
            .maxAge(0)
            .build();
    }

    public ResponseCookie clearRefreshTokenCookie() {
        return ResponseCookie.from(refreshTokenCookieName, "")
            .httpOnly(true)
            .secure(true)
            .sameSite("Strict")
            .path("/auth")
            .maxAge(0)
            .build();
    }
}
```

## 5. Real-World Scenarios

### Scenario 1: Session Management with JWT

```
Login:
  User -> POST /auth/login -> Server validates credentials
  Server -> Generates access_token (15min) + refresh_token (7 days)
  Server -> Returns tokens + sets httpOnly cookies

API Request:
  Client -> GET /api/resource (with Bearer token in cookie/header)
  Server -> Validates JWT signature, checks expiry
  Server -> Checks blacklist (Redis)
  Server -> Extracts user info from claims
  Server -> Authorizes and processes request

Token Refresh:
  Client -> POST /auth/refresh (with refresh_token)
  Server -> Validates refresh token
  Server -> Checks if refresh token was already used (rotation)
  Server -> Issues new access_token + refresh_token pair
  Client -> Continues with new tokens

Logout:
  Client -> POST /auth/logout
  Server -> Adds access_token JTI to blacklist
  Server -> Invalidates refresh_token in database
```

### Scenario 2: Service-to-Service with JWT

```
Service A wants to call Service B:
  Service A -> Creates a JWT signed with its private key
  Service A -> Sets claims: iss=service-a, aud=service-b, sub=order-service
  Service A -> Calls Service B with Bearer token
  Service B -> Fetches Service A's public key from JWKS endpoint
  Service B -> Validates signature, audience, issuer
  Service B -> Processes request if valid
```

### Scenario 3: Stateless Microservices

```
API Gateway
  |-- Validates JWT
  |-- Extracts user ID, roles, permissions
  |-- Passes JWT to downstream services
  |-- Downstream validates signature locally (no DB/cache needed)
  |-- Each service checks claims independently
```

## 6. Performance

### JWT Performance Optimization

- **Local Validation**: Validate JWT locally without remote calls.
- **Cached JWKs**: Cache public keys to avoid repeated downloads.
- **Token Claim Caching**: Cache parsed claims for repeat tokens within a request.
- **Compact Claims**: Minimize payload size for faster transmission.
- **Algorithm Choice**: ES256 is faster than RS256 for verification.

### Token Size Optimization

```java
// Minimize claims to reduce token size
public String generateOptimizedToken(String userId, String role) {
    return Jwts.builder()
        .setSubject(userId)
        .claim("r", role)  // Short claim names
        .signWith(privateKey)
        .compact();
}
```

### JWT Validation Benchmarks

```
Algorithm   Sign (ops/s)   Verify (ops/s)
HS256       2,000,000      1,500,000
RS256 2048   20,000          100,000
ES256        50,000          30,000
EdDSA       100,000          80,000
```

## 7. Security

### JWT Security Best Practices

- **Use asymmetric signing (RS256/ES256)** for distributed systems.
- **Never accept the `none` algorithm**: Always validate algorithm.
- **Set short expiry**: 15 minutes for access tokens.
- **Use refresh token rotation**: Detect token theft.
- **Validate all claims**: `exp`, `nbf`, `iss`, `aud`, `iat`.
- **Use `jti` claim**: Unique token ID for revocation.
- **Store tokens securely**: httpOnly cookies or secure storage.
- **Implement token blacklist**: For immediate revocation needs.
- **Use key rotation**: Rotate signing keys periodically.
- **Validate algorithm against expected list**: Prevent algorithm confusion attacks.

### Algorithm Confusion Attack

Attack: Attacker changes `alg` from `RS256` to `HS256` and signs token with public key.

```java
// Prevention: Validate algorithm against expected list
public Claims validateToken(String token) {
    String header = new String(Base64.getUrlDecoder().decode(
        token.split("\\.")[0]));
    JsonNode headerJson = new ObjectMapper().readTree(header);
    String alg = headerJson.get("alg").asText();

    // Only allow specific algorithms
    if (!Set.of("RS256", "ES256").contains(alg)) {
        throw new TokenValidationException(
            "Algorithm not allowed: " + alg);
    }

    return Jwts.parserBuilder()
        .setSigningKeyResolver(signingKeyResolver)
        .build()
        .parseClaimsJws(token)
        .getBody();
}
```

## 8. Common Mistakes

- **Storing JWT in localStorage**: Vulnerable to XSS; use httpOnly cookies.
- **Long expiry without refresh rotation**: Increased theft window.
- **No token revocation mechanism**: Cannot invalidate compromised tokens.
- **Using symmetric keys across services**: Shared secret is hard to manage.
- **Including sensitive data in payload**: JWT payload is base64 encoded, not encrypted.
- **Not validating algorithm**: Vulnerable to algorithm confusion attacks.
- **Ignoring token replay**: Use `jti` and nonce for important operations.
- **No audience validation**: Token may be used at wrong resource server.
- **Leaking tokens in logs**: Mask tokens in logs.
- **Null secret/weak key**: Use strong keys (256-bit+ for HMAC, 2048-bit+ for RSA).

## 9. Senior Engineer Perspective

### When to Use JWT

**Good fit:**
- Stateless authentication in distributed systems.
- OAuth 2.0 access tokens.
- Microservices with local token validation.
- Single sign-on (SSO) systems.
- APIs with high throughput requirements.

**Bad fit:**
- Server-side sessions (use session cookies).
- Applications requiring instant revocation.
- Small applications (overhead not justified).
- Systems with strict size constraints (e.g., HTTP headers limit).

### JWT vs Session Architecture

```
JWT:
  + Stateless (no DB needed)
  + Scalable (any server can validate)
  - Hard to revoke
  - Larger payload

Session:
  + Easy to revoke (delete session)
  + Small client-side footprint
  - Stateful (requires session store)
  - Session affinity or shared store needed
```

### Token Strategy for Enterprise

```
Access Token: JWT, 15 min expiry, in-memory client
Refresh Token: Opaque or JWT, 7 day expiry, rotation, httpOnly cookie
ID Token: JWT (OIDC), contains user profile info
Service Token: JWT signed with service key, short expiry
```

## 10. Interview Questions (Easy)

1. What does JWT stand for?
2. What are the three parts of a JWT?
3. What is the purpose of JWT signature?
4. What is the difference between JWT and JWS?
5. What claims are commonly found in a JWT payload?
6. What is the `exp` claim?
7. What is the difference between symmetric and asymmetric JWT signing?
8. What is base64url encoding?
9. What is the purpose of the `sub` claim?
10. How do you send a JWT in an HTTP request?

## Medium

1. How does JWT signature verification work?
2. What is the algorithm confusion attack and how do you prevent it?
3. How do you implement token revocation for JWTs?
4. What is refresh token rotation and why is it important?
5. How do you handle JWT expiry on the client side?
6. What is the difference between JWE and JWS?
7. How do you implement key rotation for JWT signing?
8. What is a JWKS endpoint and how does it work?
9. How do you validate JWT audience and issuer?
10. What are the security implications of storing JWTs in localStorage vs cookies?

## 11. Advanced Interview Questions (Hard)

1. Design a JWT-based authentication system with instant revocation capability.
2. How would you implement a JWT token exchange protocol between services?
3. Design a multi-tenancy JWT system where tenants have different signing keys.
4. How do you implement JWT-based delegation (impersonation) with audit trail?
5. Design a JWT token format that supports both backward and forward compatibility.
6. How would you implement a distributed JWT blacklist across multiple data centers?
7. Design a JWT-based API key system for third-party developers.
8. How do you handle JWT clock skew across distributed systems?
9. Implement a JWT step-up authentication system for sensitive operations.
10. How would you migrate from opaque tokens to JWTs in a production system?

## System Design

1. Design a JWT-based authentication system for a microservices platform.
2. Design a JWT-based SSO system for multiple applications.
3. Design a JWT-based API gateway with centralized token validation.
4. Design a JWT-based permission system for a multi-tenant SaaS.
5. Design a JWT token management system with automatic rotation and revocation.
6. Design a JWT-based BFF (Backend for Frontend) authentication flow.
7. Design a JWT-based webhook verification system.
8. Design a JWT-based service mesh authentication system.
9. Design a JWT-based mobile authentication with biometric integration.
10. Design a JWT-based federated identity system across organizations.

## 12. Expert-Level Interview Questions (Architect-Level)

1. Design a global JWT-based authentication infrastructure for 100M+ users across 50 regions with sub-millisecond validation latency and instant revocation capability.
2. How would you implement a JWT-based capability system where tokens grant fine-grained, composable permissions that can be delegated and attenuated?
3. Design a hybrid token strategy that combines JWT's stateless benefits with session-like revocation using distributed Bloom filters.
4. How would you build a JWT-based zero-trust architecture where every request (including inter-service) requires a valid, short-lived token with proof-of-possession?
5. Design a JWT token chain system for distributed workflows where each service adds claims and signs, creating an immutable audit trail.
6. How would you implement a JWT-based consent management system for GDPR compliance where user consent decisions are encoded in tokens?
7. Design a system that automatically detects JWT token theft by analyzing usage patterns (geography, device, timing) and triggers revocation.
8. How would you implement a JWT-based capability URLs system where tokens encode access rights to specific resources with time-limited, one-time use semantics?
9. Design a JWT key management infrastructure that supports automatic rotation, multi-region distribution, and disaster recovery with zero-downtime key transitions.
10. How would you build a JWT-based cross-organization federation system where tokens can be issued and validated across independent security domains?

## 13. Debugging & Troubleshooting

### Common Issues

- **SignatureException**: Signing key mismatch or corrupted token.
- **ExpiredJwtException**: Token has passed its expiry time.
- **MalformedJwtException**: Token is not properly formatted (wrong number of segments).
- **UnsupportedJwtException**: Token uses an unexpected format or algorithm.
- **IllegalArgumentException**: Token is null, empty, or whitespace.
- **JWT validation fails after key rotation**: Client cached old public key.

### Debugging JWT

```java
@Component
public class JwtDebugger {

    public void debugToken(String token) {
        try {
            String[] parts = token.split("\\.");
            if (parts.length != 3) {
                log.error("Token has {} parts (expected 3)", parts.length);
                return;
            }

            // Decode header
            String header = new String(Base64.getUrlDecoder().decode(parts[0]));
            log.info("Header: {}", header);

            // Decode payload
            String payload = new String(Base64.getUrlDecoder().decode(parts[1]));
            log.info("Payload: {}", payload);

            // Log expiry status
            ObjectMapper mapper = new ObjectMapper();
            JsonNode claims = mapper.readTree(payload);
            long exp = claims.get("exp").asLong();
            boolean expired = exp * 1000 < System.currentTimeMillis();
            log.info("Token expired: {}", expired);
            if (expired) {
                long expiredMs = System.currentTimeMillis() - (exp * 1000);
                log.info("Expired {} ms ago", expiredMs);
            }

        } catch (Exception e) {
            log.error("Failed to debug token", e);
        }
    }
}
```

### Token Inspection Endpoint

```java
@RestController
public class TokenDebugController {

    @PostMapping("/debug/token")
    public ResponseEntity<Map<String, Object>> debugToken(
            @RequestHeader("Authorization") String authHeader) {

        String token = authHeader.replace("Bearer ", "");
        Map<String, Object> info = new HashMap<>();

        try {
            Claims claims = jwtTokenProvider.validateToken(token);
            info.put("valid", true);
            info.put("subject", claims.getSubject());
            info.put("issuer", claims.getIssuer());
            info.put("audience", claims.getAudience());
            info.put("issuedAt", claims.getIssuedAt());
            info.put("expiration", claims.getExpiration());
            info.put("remaining", claims.getExpiration().getTime()
                - System.currentTimeMillis());
            info.put("claims", claims);
        } catch (Exception e) {
            info.put("valid", false);
            info.put("error", e.getClass().getSimpleName());
            info.put("message", e.getMessage());
        }

        return ResponseEntity.ok(info);
    }
}
```

## 14. Comparison Section

### JWT vs Session Tokens

| Aspect | JWT | Session |
|--------|-----|---------|
| Storage | Client-side (stateless) | Server-side (stateful) |
| Scalability | Excellent (no shared state) | Requires shared session store |
| Revocation | Hard (until expiry) | Instant (delete session) |
| Payload | Contains user data | Just session ID |
| Size | Larger (KB) | Small (bytes) |
| Cross-domain | Easy | Hard |
| Security | Token theft = full access | Session theft = single session |

### JWT vs PASETO

| Aspect | JWT | PASETO |
|--------|-----|--------|
| Standard | IETF RFC 7519 | IETF draft |
| Algorithm | Developer chooses | Versioned (v1: RSA, v2: EdDSA) |
| Security | Algorithm confusion risks | Algorithm fixed per version |
| Ecosystem | Very mature | Growing |
| Complexity | High (many options) | Low (opinionated) |
| Adoption | Widely used | Niche |

## 15. Revision Notes

- JWT = Header.Payload.Signature (base64url encoded, dot-separated)
- Header: algorithm (alg), type (typ), key ID (kid)
- Payload: claims (sub, iss, aud, exp, iat, nbf, jti)
- Signature: HMAC (symmetric) or RSA/ECDSA/EdDSA (asymmetric)
- Validate: signature, expiry, issuer, audience, not-before, algorithm
- Security: never accept `none` algorithm, use algorithm whitelist, short expiry
- Revocation: blacklist by jti (Redis), refresh token rotation
- Spring Boot: JwtTokenProvider, OncePerRequestFilter, JWKS endpoint
- Prefer asymmetric signing (RS256/ES256) for distributed systems
- Use httpOnly cookies for web apps, handle refresh rotation

## 16. Cheat Sheet

```
+------------------------------------------------------------------+
| JWT CHEAT SHEET                                                  |
+------------------------------------------------------------------+
| STRUCTURE                                                        |
|   Header.Payload.Signature                                       |
|   eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.                        |
|   eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIn0.        |
|   sJ0yWzW_rF5m2q3OaP5P8KcX7iA                                   |
+------------------------------------------------------------------+
| STANDARD CLAIMS                                                  |
|   sub  (Subject)     -> User identifier                          |
|   iss  (Issuer)      -> Token issuer                             |
|   aud  (Audience)    -> Intended recipient                       |
|   exp  (Expiration)  -> Expiry timestamp                         |
|   nbf  (Not Before)  -> Not valid before timestamp               |
|   iat  (Issued At)   -> Token issuance timestamp                 |
|   jti  (JWT ID)      -> Unique token identifier                  |
+------------------------------------------------------------------+
| SIGNING ALGORITHMS                                               |
|   HS256:  HMAC-SHA256   (symmetric, shared secret)               |
|   RS256:  RSA-SHA256    (asymmetric, RSA key pair)               |
|   ES256:  ECDSA-SHA256  (asymmetric, ECDSA key pair)             |
|   EdDSA:  Ed25519       (asymmetric, modern)                     |
|   WARNING: Never accept "none" algorithm!                        |
+------------------------------------------------------------------+
| VALIDATION CHECKLIST                                             |
|   [ ] 1. Parse token (3 parts)                                   |
|   [ ] 2. Decode header, check algorithm whitelist                |
|   [ ] 3. Fetch signing key (from JWKS or local)                  |
|   [ ] 4. Verify signature                                        |
|   [ ] 5. Check exp is in future                                  |
|   [ ] 6. Check nbf is in past                                    |
|   [ ] 7. Validate issuer matches expected                        |
|   [ ] 8. Validate audience includes this service                 |
|   [ ] 9. Check if token is blacklisted (jti)                     |
+------------------------------------------------------------------+
| SPRING BOOT EXAMPLE                                              |
|   Jwts.builder()                                                 |
|     .setSubject(userId)                                          |
|     .setIssuer("https://api.example.com")                        |
|     .setIssuedAt(new Date())                                     |
|     .setExpiration(new Date(System.currentTimeMillis() + 3600000))|
|     .claim("roles", List.of("ADMIN"))                            |
|     .signWith(privateKey, SignatureAlgorithm.RS256)              |
|     .compact();                                                  |
+------------------------------------------------------------------+
| SECURITY BEST PRACTICES                                          |
|   1. Use asymmetric signing (RS256/ES256)                        |
|   2. Short access token TTL (15-60 min)                         |
|   3. Refresh token rotation                                      |
|   4. Store tokens in httpOnly cookies                            |
|   5. Validate algorithm whitelist                                |
|   6. JWKS endpoint for public key distribution                   |
|   7. Blacklist compromised tokens by jti                         |
|   8. Rotate signing keys regularly                               |
|   9. Never include sensitive data in payload                     |
|  10. Validate all claims (iss, aud, exp, nbf)                   |
+------------------------------------------------------------------+
```
