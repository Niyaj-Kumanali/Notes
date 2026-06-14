# JWT (JSON Web Tokens)

---

## Overview

- **Definition:** JWT (RFC 7519) is a compact, URL-safe token format for securely transmitting claims between parties as a JSON object, digitally signed or encrypted.
- **Why It Exists:** Enables stateless authentication — servers validate tokens locally without a session store, making JWTs ideal for distributed systems, microservices, and OAuth 2.0.
- **Key Concepts:** **Header.Payload.Signature** (three dot-separated base64url-encoded parts), **JWS** (signed — ensures integrity), **JWE** (encrypted — ensures confidentiality), **Claims** (sub, iss, aud, exp, iat, nbf, jti), **JWKS** (JSON Web Key Set for public key distribution), **Algorithm Confusion Attack** (attacker changes alg from RS256 to HS256 using public key as secret).
- **Token Binding** — Binds a JWT to a specific client by including a hash of the client's TLS certificate (`cnf` claim) or a DPoP public key, preventing token replay if the token is stolen from a different device or IP.
- **Performance Characteristics** — JWT validation with cached JWKS takes <1ms locally vs 30-50ms for opaque token introspection. The trade-off is larger token size (typically 1-2KB vs 20-30 bytes for opaque) and inability to instantly revoke without server-side checks.

---

## Core Concepts

- **Structure:** Header contains `alg` (signing algorithm) and `kid` (key ID). Payload contains registered claims (`sub`, `iss`, `aud`, `exp`, `iat`, `nbf`, `jti`) and custom claims. Signature is computed over `base64UrlEncode(header) + "." + base64UrlEncode(payload)`.
- **Signing Algorithms:** **HS256** (HMAC symmetric — shared secret, simple but harder to manage across services). **RS256** (RSA asymmetric — public/private key pair, standard for distributed systems). **ES256** (ECDSA — faster than RSA). **EdDSA** (Ed25519 — high security and performance).
- **Validation Process:** Parse token → Decode header → Verify algorithm (reject `none`) → Fetch signing key (JWKS or local) → Verify signature → Validate claims: `exp` (future), `nbf` (past), `iss` (matches), `aud` (includes this service), `iat` (reasonable) → Check blacklist by `jti`.
- **JWT vs Opaque Tokens:** JWT is self-contained (local validation, larger size, hard to revoke). Opaque tokens are references (requires introspection endpoint, small, instant revocation).
- **Token Binding with DPoP** — DPoP (Demonstration of Proof-of-Possession) binds a JWT to a specific client key pair. The client proves possession of the private key on every request. A stolen JWT is useless without the corresponding private key. DPoP is recommended by OAuth 2.1 for high-security scenarios.
- **Claim Validation Best Practices** — Always validate `exp` (reject expired), `nbf` (reject early), `iss` (must match expected issuer), `aud` (must include this service's identifier), and `iat` (reject if too far in past or future). Use a configurable clock skew window of 30-60 seconds. Validate `jti` uniqueness for replay detection in high-security systems.

```java
// JWT generation with RS256
String token = Jwts.builder()
    .setSubject(userId).setIssuer("https://api.example.com")
    .setIssuedAt(now).setExpiration(expiry).setId(UUID.randomUUID().toString())
    .claim("roles", userDetails.getAuthorities())
    .signWith(privateKey, SignatureAlgorithm.RS256)
    .compact();
```

---

## Common Mistakes

- **Storing JWT in localStorage** — Vulnerable to XSS. Use httpOnly cookies.
  - **Why it looks correct:** The app works perfectly in development and the token is available to JavaScript for API calls — the XSS vulnerability is invisible until an attacker injects a script tag.
- **No Token Revocation Mechanism** — Compromised tokens valid until expiry. Maintain a blacklist by `jti` in Redis.
  - **Why it looks correct:** JWT is "stateless" — the entire point is not needing server-side storage. The inability to revoke is a feature until a token is stolen.
- **Using Symmetric Keys Across Services** — Shared secret is hard to manage securely. Prefer asymmetric (RS256/ES256).
  - **Why it looks correct:** The symmetric approach works in a monolith and keeps configuration simple — the risk of secret leakage grows with each additional consumer.
- **Not Validating Algorithm** — Vulnerable to algorithm confusion attacks. Whitelist expected algorithms.
  - **Why it looks correct:** The JWT library processes `alg` automatically — the developer never considers that an attacker can change the algorithm header.
- **Including Sensitive Data in Payload** — Payload is base64 encoded, not encrypted. Never include passwords/PII.
  - **Why it looks correct:** Base64 is unreadable at a glance — the developer mistakes encoding for encryption.
- **Long Expiry Without Refresh Rotation** — Increased theft window. Use 15-min access tokens with rotating refresh tokens.
  - **Why it looks correct:** A 24-hour token means fewer logins — the theft window grows silently, with no symptom until the token is compromised.
- **Not Setting an Appropriate `aud` Claim** — Without audience validation, a token issued for one service can be used against any other service. Always include and validate the `aud` claim to restrict token usage to the intended recipient.
  - **Why it looks correct:** The token validates against the issuer — the developer doesn't realize a token meant for Service A can authenticate against Service B.
- **Hardcoding Secrets in Source Code** — Committing JWT signing secrets or private keys to version control exposes the entire authentication system. Use environment variables, secrets managers (Vault, AWS Secrets Manager), or JWKS endpoints for key distribution.
  - **Why it looks correct:** The repository is private — "nobody outside the team will see it" — until a contractor leaves with access or the repo is accidentally made public.

---

## Key Design Considerations

- **Token Strategy:** Access Token (JWT, 15 min, in-memory client). Refresh Token (opaque or JWT, 7 days, rotation, httpOnly cookie). ID Token (JWT for OIDC user profile).
- **Refresh Token Rotation:** Each refresh invalidates the previous token. If a stolen refresh token is used, the legitimate user's next refresh fails — detecting theft. Rotate on every use.
- **Key Rotation:** Keep multiple keys in JWKS. Use `kid` header to identify signing key. Keep old keys for grace period to validate existing tokens.
- **Token Blacklist:** Store `jti` in Redis with TTL matching token expiry. On logout or compromise, add to blacklist. Check before each request.
- **Algorithm Whitelist:** Accept only expected algorithms (e.g., RS256, ES256). Reject `none`, HS256 if you use asymmetric keys.
- **Clock Skew:** Allow configurable clock skew (typically 30-60 seconds) for `exp` and `nbf` validation to handle time differences across servers.
- **Token Exchange (RFC 8693)** — Enables a service to exchange one JWT for another with different claims or audience. Useful for service-to-service delegation where Service A has a token for its audience but needs to call Service B with a token scoped to B's audience.
- **Key Rotation Automation** — Automate key rotation by publishing new keys to JWKS before the old keys expire. Use a grace period where both old and new keys are valid for verification. Monitor JWKS fetch rates to ensure all clients have picked up the new key before deactivating the old one.

---

## Real-World Scenarios

### Scenario 1: Algorithm Confusion Attack Mitigation
**Context:** Your authentication service uses RS256 (RSA asymmetric) to sign JWTs. A developer accidentally exposes the public key via a log file. An attacker discovers the public key and crafts a JWT with `alg: "HS256"`, signing it with the public key as the HMAC secret. The server's JWT library, when reading `alg: "HS256"`, treats the public key as an HMAC shared secret and accepts the forged token. The attacker gains admin access.

**Resolution:** (1) Whitelist accepted algorithms in the JWT library — reject `HS256` if your service only uses `RS256`. (2) Never use the same variable for symmetric and asymmetric keys — maintain separate code paths. (3) Use `jose.requireKey` or equivalent in Java's Nimbus JOSE to enforce key type checking. (4) Implement a `typ` header check to distinguish JWT (access tokens) from JWK (key material). (5) Monitor for unexpected `alg` values in logs and alert on anomalies.

### Scenario 2: Refresh Token Theft Detection
**Context:** A refresh token (7-day expiry, no rotation) is stolen from a mobile app's secure storage. The attacker uses it to generate new access tokens. The legitimate user does not notice until the token expires in 7 days. The system has no mechanism to detect or respond to the theft.

**Resolution:** Implement refresh token rotation with theft detection. On each refresh, issue a new refresh token and invalidate the old one. If a stolen token is used after the legitimate token has been rotated, the legitimate user's next refresh fails — they get an error and must re-authenticate. The system logs this as a theft indicator. Additionally:
- Bind refresh tokens to device fingerprints (device ID, IP range, user-agent)
- Use refresh token family tracking (store parent-child relationships, detect forks)
- Implement automatic revocation of the entire token family on suspected theft

### Scenario 3: Microservice JWT Validation Without Latency
**Context:** Your system has 20 microservices. Each service needs to validate every incoming request's JWT. The first implementation calls the Auth Server's introspection endpoint on every request. This creates 20× the request volume to the Auth Server, adds 50ms per hop, and the Auth Server becomes a bottleneck and single point of failure.

**Resolution:** Switch from opaque tokens to JWTs signed with RS256. Each service fetches the public key from the JWKS endpoint once (caching for 24 hours). JWT validation is local and takes <1ms. The Auth Server is no longer in the request path — it only handles token issuance. Benefits: no network call for validation, no central bottleneck, each service can validate independently. Trade-off: token revocation is not immediate — valid until JWT's `exp`. For immediate revocation, use a Redis-based blacklist that each service checks in <5ms.

## Use Cases

- **Stateless authentication for microservices** — API gateways validating tokens across 20+ services
  - Each service validates JWT locally using cached JWKS public keys (<1ms validation). No centralized session store or auth server bottleneck.
  - **Avoid when:** instant token revocation is required — JWTs are valid until `exp`. Combine with short TTLs (15 min) and a distributed blacklist.

- **OAuth 2.0 access tokens** — authorization for third-party applications or mobile apps
  - JWT as the access token format allows resource servers to validate without calling the authorization server each time. Self-contained user claims and scopes.
  - **Avoid when:** tokens are large (>8KB) and bandwidth is constrained — opaque tokens are more compact.

- **Single sign-on (SSO) tokens** — cross-domain authentication across multiple applications in an organization
  - ID tokens (OpenID Connect) carry user identity claims (name, email, roles). One login authenticates the user across all apps in the federation.
  - **Avoid when:** applications have dramatically different security requirements — use separate issuers or audience restrictions per domain.

- **Service-to-service authentication** — internal API calls between backend services
  - JWT signed with a service account's private key. Each service validates the token's signature, issuer, and audience before accepting the request.
  - **Avoid when:** services are in the same trust boundary (e.g., same Kubernetes cluster) — mutual TLS or network policies may be simpler.

- **Information exchange with verifiable integrity** — password reset tokens, email verification links, or invitation links
  - Signed JWT carries the user ID and purpose. The signature prevents tampering. Expiration limits the validity window.
  - **Avoid when:** the payload contains sensitive data — encrypt the JWT (JWE) or use a server-side session reference instead.

---

## Scenario-Based Questions

1. **Q: Your JWT library is vulnerable to algorithm confusion. A security auditor reports that setting `alg: "none"` bypasses validation entirely on some endpoints. How do you fix this across the entire codebase?**
    - A: (1) Configure the JWT library to require a signature — reject tokens with `alg: "none"`, `alg: "None"`, `alg: "NONE"`, `alg: "nOnE"` (case-insensitive check). (2) Whitelist expected algorithms — only `RS256` or `ES256`. (3) Enforce algorithm validation at the library level, not in application code. In Java with Nimbus JOSE: `JWSAlgorithm.parse(tokenHeader.getAlg()).require(expectedAlgs)`. (4) Write a centralized JWT validation utility used by all services — don't duplicate parsing logic. (5) Add a response filter that detects and logs any `alg` values outside the whitelist.
    - **Interview follow-up:** The centralized JWT utility is a shared library that all services import — what happens when a security patch requires updating the library? How do you ensure every service picks up the fix without manual coordination?

2. **Q: Your refresh tokens use rotation. A legitimate user reports they are randomly logged out during normal usage. Investigation shows their refresh token was rotated but the new token was not received by the client (network issue during refresh response). How do you handle this without compromising security?**
   - A: The user is stuck — old token is invalidated, new token wasn't received. Options: (1) Maintain a grace period: keep the old refresh token valid for 30 seconds after rotation. During this window, either token can be used, but using the old one triggers a new rotation. This handles network failures gracefully. (2) Store refresh tokens in a sliding window: each user has 2-3 valid refresh tokens at any time (rotation keeps a small window of the previous token). (3) Implement re-authentication with limited scope: if refresh fails, allow the user to re-authenticate with their password and issue a new session without losing data.

3. **Q: You need to revoke a specific user's access immediately because their account was compromised. JWTs have 15-minute expiry. How do you achieve instant revocation without switching to opaque tokens?**
    - A: (1) JWT blacklist in Redis: store the `jti` (JWT ID) of the user's current access token with TTL matching remaining expiry. Each service checks the blacklist before processing. (2) Blacklist all tokens by `sub` (user ID): store `user_<id>` in Redis with TTL of 15 minutes. Any JWT with that user ID is rejected. (3) Invalidate all refresh tokens for the user server-side (database/REDIS record). (4) Increment a "token version" claim in the user's DB record — issue new tokens with the incremented version; reject any token with an older version. This is more efficient than individual blacklists.
    - **Interview follow-up:** The token version approach requires a database read on every request to check the version — doesn't this defeat the stateless validation advantage of JWTs? How do you cache the version check without reintroducing the revocation delay problem?

4. **Q: Your mobile app uses JWT stored in secure device storage. A user uninstalls and reinstalls the app, losing the stored token. The app needs to maintain the session without forcing the user to log in again. How do you handle this?**
   - A: This is a core limitation of client-side token storage — uninstall destroys all local data. Options: (1) Use device-backed encryption (Android Keystore / iOS Keychain) which sometimes survives reinstall if tied to device identity. (2) Use biometric-based key derivation — derive the token encryption key from device biometrics, so it's not stored in app data. (3) Use a server-side session that survives reinstall — store a persistent session identifier server-side, tied to device fingerprint, that allows reissuing tokens after reinstall. (4) Accept the trade-off: require re-authentication on reinstall. This is actually the most secure option — a compromised device is the one edge case where login is desirable.

5. **Q: Your backend issues JWTs signed with RS256. A developer accidentally commits the private key to GitHub. You discover this 6 hours later. What steps do you take to contain the breach?**
   - A: (1) Immediately revoke the compromised key pair — rotate to a new key in JWKS. (2) Invalidate all tokens signed with the compromised key: blacklist all token `jti`s issued in the last 24 hours. (3) Generate a new key pair and add to JWKS with a new `kid`. (4) Force re-authentication for all users (clear sessions). (5) Audit the commit history: check if any other secrets were exposed. (6) Implement pre-commit hooks (git-secrets) and secret scanning to prevent future leaks. (7) Shorten token expiry to minimize blast radius of future leaks.

6. **Q: You have a single-page application (SPA) that stores JWTs in memory. On page refresh, the token is lost and the user sees a flash of unauthenticated UI before being redirected to login. How do you fix the user experience without sacrificing security?**
   - A: (1) Use a refresh token in an httpOnly cookie: SPA loads, checks if it can get a new access token silently. If the refresh cookie exists, the server issues a new access token. The user never sees the refresh token — it's httpOnly and not accessible to JavaScript. (2) Use a `token` cookie (not localStorage) with `SameSite=Strict` — the SPA reads this on page load to restore the session. (3) Implement token persistence in a web worker: the web worker can store tokens and survive page refreshes (the main thread can request tokens from the worker on load). (4) Use OAuth 2.0's `prompt=none` with iframes: silently re-authenticate the user in a hidden iframe. The auth server recognizes the session cookie and issues new tokens without user interaction.

7. **Q: Your JWT claims include `roles: ["ADMIN", "USER"]`. A developer creates a new admin-only endpoint and checks for the "ADMIN" role. A user authenticates and their JWT has `roles: ["USER"]`. The developer's role check uses `Optional.isPresent()` but has a bug — it checks for the existence of the `roles` claim rather than the value. All authenticated users pass this check. How do you prevent these logic errors systematically?**
   - A: (1) Use an enum for roles, not strings — `Role.valueOf(claimValue)` throws on invalid values. (2) Implement a centralized `AuthorizationService.isAdmin(token)` method that parses roles and validates. (3) Use Spring Security method-level security (`@PreAuthorize("hasRole('ADMIN')")`) instead of manual claims parsing. (4) Add integration tests that verify authorization logic with both valid and invalid tokens. (5) Use property-based testing to test all edge cases of role parsing. (6) Add linter rules that flag manual JWT claim parsing.

8. **Q: Your JWKS endpoint returns two keys with `kid: "key-2024-v1"` and `kid: "key-2025-v2"`. Tokens are signed with the v2 key. On key rotation day, you add a v3 key and start signing with it. Clients that cached the JWKS 24 hours ago only know about v1 and v2. They reject tokens signed with v3. How do you handle this transition?**
   - A: (1) Add the new key to JWKS 24 hours before signing: publish v3 in the JWKS but continue signing with v2. This gives clients time to pick up the new key. (2) Set a transition period: sign with both v2 and v3 simultaneously for 24-48 hours (publish both in JWKS). (3) Use a grace period for key validation: when verifying tokens, allow keys that were valid within the last 48 hours, not just the current key. (4) Monitor JWKS fetch rates to know when all clients have updated. (5) Implement client-side key caching with `Cache-Control: max-age=3600,must-revalidate` to limit stale caches to 1 hour.

9. **Q: Your microservice A receives a JWT from the API gateway. Service A needs to call service B, but the JWT was issued for `aud: "service-a"`. Service B rejects the token because the audience doesn't match. How do you propagate the authentication context without re-authentication?**
   - A: Solutions: (1) Token Exchange: the Auth Server supports token exchange. Service A exchanges the original token for a new token with `aud: "service-b"`. This is the OAuth 2.0 standard approach. (2) Include all expected audiences: issue tokens with `aud: ["service-a", "service-b"]` — each service validates its own audience. (3) Use a service-specific token: the gateway authenticates the user and passes a user-context token (JWT with user claims) to all services. Internal service-to-service calls use their own Client Credentials grant token. (4) Remove audience validation for internal services (risky — any service can impersonate any other). Recommended: token exchange for security.

10. **Q: A third-party API you integrate with requires a JWT in the Authorization header. The JWT must include a specific `roles` claim that your services use. The third-party's JWT has the same `roles` claim but with different values. When you pass the incoming third-party JWT to your own services, your authorization logic uses the `roles` claim and grants elevated privileges. How do you prevent this cross-system claim conflict?**
    - A: (1) Translation layer: create a gateway/adapter that validates the third-party JWT, maps roles to your system's role format, and issues a new internal JWT with your claims format. Never pass external tokens to internal services. (2) Claim namespacing: use `https://your-domain.com/roles` instead of just `roles`. Third-party tokens won't have this namespaced claim. (3) Always validate `iss` (issuer) and reject tokens with unexpected issuers. (4) Use a dedicated JWT for internal communication with strictly controlled claims. External tokens are consumed and discarded at the boundary.
    - **Interview follow-up:** The translation layer issues a new JWT signed with your own key — if the translation layer itself is compromised, an attacker can mint tokens with arbitrary roles. How do you protect the translation layer from becoming a privileged escalation point?

---

## Interview Questions

1. **What is the structure of a JWT?**
   - A: Three base64url-encoded segments separated by dots: `Header.Payload.Signature`. Header contains algorithm (`alg`) and key ID (`kid`). Payload contains claims (sub, iss, aud, exp, iat, jti). Signature validates integrity.

2. **What is the difference between JWS and JWE?**
   - A: JWS (JSON Web Signature) signs the payload for integrity — the content is readable. JWE (JSON Web Encryption) encrypts the payload for confidentiality — the content is hidden. JWS is more common for access tokens; JWE is used when claims contain sensitive data.

3. **How does JWT signature verification work?**
   - A: The receiver decodes the header to get `alg` and `kid`, fetches the corresponding public key from JWKS, and verifies that `HMAC/RSA/ECDSA(base64UrlEncode(header) + "." + base64UrlEncode(payload))` matches the signature. This is done locally without network calls if the JWKS is cached.

4. **What is the algorithm confusion attack and how do you prevent it?**
   - A: An attacker changes `alg` from `RS256` to `HS256` and signs the token using the server's public key (which is public) as the HMAC secret. The server, using the same public key as an HMAC key, accepts the forged token. Prevention: whitelist expected algorithms in the JWT library and reject all others.

5. **How do you implement token revocation for JWTs?**
   - A: Store the `jti` (JWT ID) in a Redis blacklist with TTL matching the remaining token expiry. Check the blacklist on every request. On logout, add the access token's `jti` and invalidate the refresh token server-side. For user-level revocation, blacklist the `sub` claim across all services.

6. **What is refresh token rotation and why is it important?**
   - A: Each refresh operation issues a new refresh token and invalidates the old one. This limits the window for stolen tokens. If a stolen refresh token is used, the legitimate user's next refresh fails — revealing the theft. Rotation is required by OAuth 2.1.

7. **What is a JWKS endpoint?**
   - A: A `/.well-known/jwks.json` endpoint that publishes the authorization server's public keys in JWK (JSON Web Key) format. Clients fetch this to verify JWT signatures without per-provider static configuration. Supports key rotation by adding new keys and removing old ones.

8. **Where should you store JWTs in a browser?**
   - A: Short access tokens (15 min) in memory (JavaScript variable) — lost on refresh but most secure. Refresh tokens in httpOnly, Secure, SameSite=Strict cookies — immune to XSS but protected against CSRF by SameSite. Never use localStorage for JWTs — any XSS vulnerability exposes all tokens.

9. **What claims must you always validate?**
   - A: `exp` (not expired), `nbf` (not before — must be in the past), `iss` (matches expected issuer), `aud` (includes this service), `iat` (reasonable timeframe — not in the future). Also validate `alg` from a whitelist. Optionally validate `jti` for uniqueness check.

10. **Design a JWT-based authentication system with instant revocation.**
    - A: Short-lived access tokens (15 min, RS256, stored in memory). Rotating refresh tokens (7 days, httpOnly cookie). Token blacklist in Redis by `jti` with TTL = remaining token expiry. On logout: blacklist access token, invalidate refresh token server-side. On compromise: blacklist all tokens for that user by `sub`. Grace period of 30 seconds for refresh token network failures. Each service validates JWT locally against cached JWKS, then checks blacklist in Redis.

---

## Developer Recommendations

- **Always whitelist JWT algorithms** — The algorithm confusion attack is one of the most common JWT vulnerabilities. Libraries like `jjwt` and Nimbus JOSE allow configuring expected algorithms. Never rely on the `alg` header from the token itself to determine verification strategy. Set `parser.setExpectedAlgorithm(RS256)` or equivalent in every JWT validation path. This single defense prevents a whole class of attacks.
  - **Production story:** A well-known incident at a major auth provider involved an attacker changing `alg` from `RS256` to `HS256` and signing with the exposed public key — the token was accepted because the library used the key variable regardless of algorithm type.

- **Use asymmetric algorithms (RS256/ES256) for distributed systems** — Symmetric HS256 requires all services to share the same secret. If one service is compromised, the secret is leaked. With RS256, only the authorization server has the private key. Services only need the public key from JWKS, which is not sensitive. Key rotation is also easier — just publish new keys in JWKS without coordinating secret distribution across 20+ services.

- **Implement refresh token rotation with a grace window** — Pure rotation is fragile: a network failure during the rotation response leaves the user with no valid token. Keep the old refresh token valid for 30 seconds after rotation. On successful response receipt, the client marks the rotation as complete. If the client receives an error, it can retry with the old token. This trade-off (30-second security window) dramatically improves reliability without meaningful security risk.

- **Separate external and internal JWT domains** — Never pass tokens from third-party systems to your internal services. Third-party JWTs have different claims, formats, and trust boundaries. Always validate and translate at the API gateway. Issue a fresh internal JWT with your own claims structure. This prevents claim collision attacks (e.g., a third-party token that happens to have an `admin: true` claim) and ensures consistent validation across your system.

- **Cache JWKS aggressively but honor cache headers** — JWKS validation is the critical path for every authenticated request. Cache JWKS responses for at least 1 hour (or respect `Cache-Control` and `Expires` headers). This eliminates the JWKS fetch latency from the critical path. The trade-off: key rotation takes up to 1 hour to propagate. Mitigation: add new keys to JWKS 24 hours before activating them (publish, wait, then sign). Monitor JWKS fetch rates to detect stale caches.

- **Validate JWT claims in a centralized library, not per-service** — Each microservice reimplementing JWT validation introduces inconsistency and bugs. Create a shared JWT validation library (internal Maven/Gradle dependency) that all services use. It handles: algorithm whitelist, JWKS fetching and caching, claim validation (exp, nbf, iss, aud), blacklist checking, and role extraction. Changes to token strategy require changing only one library version, not 20 services.
- **Use short access token TTLs (15 minutes or less) as a defense-in-depth measure** — Even with perfect implementation, a stolen JWT gives the attacker full access until expiry. Short TTLs minimize the damage window. Combine with rotating refresh tokens for seamless user experience. For high-security systems, use 5-minute access tokens with token binding (DPoP) so a stolen token is unusable from a different device.
- **Never pass external JWTs to internal services without validation and translation** — A third-party JWT may have claims that overlap with your internal claim names (e.g., `roles`, `admin`). Always validate external tokens at the API gateway, translate claims to your internal format, and issue a fresh internal JWT. This prevents claim collision attacks where an attacker crafts a third-party token with malicious claim values.
