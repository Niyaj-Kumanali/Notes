# OAuth 2.0

---

## Overview

- **Definition:** OAuth 2.0 is an authorization framework that enables third-party applications to obtain limited access to services on behalf of a resource owner, without sharing credentials.
- **Why It Exists:** Decouples authentication from authorization. Separates the role of the client from the resource owner, allowing delegated access. Industry standard used by Google, Facebook, GitHub, and virtually every major platform.
- **Key Concepts:** **Resource Owner** (user who authorizes access), **Client** (application requesting access), **Authorization Server** (issues tokens), **Resource Server** (validates tokens, serves data), **Grant Types** (Authorization Code, Client Credentials, Device Code, Refresh Token), **PKCE** (Proof Key for Code Exchange — protects public clients), **Scopes** (permissions the client requests).
- **OAuth 2.0 vs OIDC** — OAuth 2.0 is an authorization framework that issues access tokens for resource access. OpenID Connect (OIDC) adds an authentication layer on top: ID Token (JWT with user identity), UserInfo endpoint, and standardized scopes (`openid`, `profile`, `email`). OAuth answers "what can you do?"; OIDC answers "who are you?"
- **Token Types** — Access tokens (short-lived, typically 15-60 minutes, used to access resources), Refresh tokens (long-lived, typically 7-30 days, used to obtain new access tokens), and ID tokens (OIDC, JWT containing user identity claims). Each token type has different security properties and storage requirements.

---

## Core Concepts

- **Authorization Code Grant:** Client redirects user to Auth Server → User authenticates and consents → Auth Server redirects back with `code` → Client exchanges `code` + `client_secret` for tokens → Access + Refresh tokens issued. This is the most secure grant for web apps.
- **PKCE:** Public clients (mobile, SPA) generate a `code_verifier` (random 128-char string) and `code_challenge = SHA256(code_verifier)`. The challenge is sent in the auth request; the verifier is sent in the token request. Prevents interception attacks.
- **Client Credentials Grant:** Server-to-server communication. Client authenticates with its own credentials and receives an access token directly, without user involvement. Used for service-to-service API calls.
- **OAuth 2.1:** Consolidates best practices — PKCE required for all public clients, Implicit and Password grants removed, refresh tokens must be sender-constrained or rotate, redirect URIs use exact matching.
- **Client Authentication Methods** — Confidential clients authenticate using `client_secret_basic` (HTTP Basic Auth), `client_secret_post` (POST body), or `private_key_jwt` (client assertion signed with private key). JWK-based client authentication (`private_key_jwt`) is more secure than shared secrets because the client's private key is never transmitted.
- **Scope Negotiation and Consent** — Clients request specific scopes during authorization. The authorization server may grant a subset based on policy or user consent. The returned token includes only the granted scopes. Resource servers must validate that the token's scope covers the requested operation, typically by checking the `scope` claim in the JWT.

```java
// Spring Security OAuth2 Resource Server
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/**")
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(GET, "/api/public/**").permitAll()
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter())))
        .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
        .build();
}
```

---

## Common Mistakes

- **Not Using PKCE for Public Clients** — Mobile and SPA apps must use PKCE. Without it, authorization code interception is possible. This *looks correct* because the Authorization Code flow already requires a client secret — the developer doesn't realize that public clients cannot keep secrets and the code itself is the only credential.
- **Storing Tokens Insecurely** — LocalStorage exposes tokens to XSS. Use httpOnly cookies or secure platform storage. This *looks correct* because the token is accessible to JavaScript and the app works — the XSS vector that reads the token is invisible until an attacker finds an injection point.
- **Long-Lived Access Tokens** — Use short expiry (15-60 min) with refresh tokens. This *looks correct* because the user stays logged in without interruption — the security risk of a stolen long-lived token has no observable symptom until misuse.
- **Not Validating Redirect URIs** — Can lead to open redirect vulnerabilities. This *looks correct* because the redirect happens on the client side that the user sees — the developer trusts the browser, not realizing an attacker can craft a custom redirect_uri.
- **Missing State Parameter** — Vulnerable to CSRF attacks on the authorization callback. This *looks correct* because the code exchange succeeds without the state parameter — the CSRF attack is invisible because the tokens are delivered to the attacker, not the legitimate user.
- **Hardcoded Client Secrets** — Use environment variables, vaults, or managed secrets. This *looks correct* because the code compiles and works — the secret is just a string in the repo, invisible to casual inspection.
- **Implicit Grant Usage** — The Implicit Grant (deprecated in OAuth 2.1) exposes tokens in the URL fragment, visible in browser history and server logs. Migrate to Authorization Code + PKCE for all clients. This *looks correct* because the token arrives in the browser and the SPA uses it immediately — the developer doesn't check browser history or server logs after their own login.
- **Missing Token Introspection on Resource Servers** — Resource servers that blindly accept any JWT without validating signature, expiry, issuer, or audience are vulnerable to token forgery attacks. This *looks correct* because the JWT is signed and decodes correctly — the developer trusts the signature alone without verifying that the token was intended for this specific resource server.
- **Overly Broad Scopes** — Requesting `read_all` or `write_all` instead of granular scopes violates the principle of least privilege. Use fine-grained scopes like `orders:read`, `profile:write`. This *looks correct* because requesting broad scopes means fewer re-authorizations — the security cost is hidden in the damage a compromised token can do across the entire API surface.

---

## Key Design Considerations

- **Always Use PKCE** for all public clients (mobile, SPA). For confidential clients (backend), use client_secret + PKCE as defense-in-depth.
- **Short Access Token TTL** (15-60 minutes). Use refresh tokens for long-lived sessions with rotation (each refresh invalidates the previous token).
- **Strict Redirect URI Validation** — Exact match only (not prefix or pattern). Prevents open redirect and code interception.
- **State Parameter** — Random anti-CSRF token in the authorization request. Validated on callback to prevent CSRF.
- **Token Validation:** Resource servers validate JWT locally (signature, exp, iss, aud) or use introspection endpoint for opaque tokens. Cache introspection results.
- **Microservices Architecture:** API Gateway validates tokens, passes them downstream. Downstream services validate locally (JWT) or use token exchange for service-specific tokens.
- **DPoP (Demonstration of Proof-of-Possession):** Binds token to a client's public key. Prevents token replay if the token is stolen.
- **Rich Authorization Requests (RAR)** — An extension that allows clients to request fine-grained authorization with detailed parameters (e.g., `payment:amount=50`, `file:write:path=/docs`). RAR replaces coarse scopes with rich, context-specific authorization data.
- **Token Exchange (RFC 8693)** — Enables a client or service to exchange one token for another with different scopes or audience. Critical for service-to-service delegation in microservices where a token issued for Service A must be exchanged for a token scoped to Service B.
- **FAPI (Financial-grade API)** — A higher-security OAuth 2.0 profile for financial services. Requirements include: JWT-secured authorization requests (JAR), JWT-secured client authentication, PAR (Pushed Authorization Requests), and sender-constrained access tokens (DPoP or mTLS).

---

## Real-World Scenarios

### Scenario 1: Authorization Code Interception on Mobile
**Context:** A mobile banking app uses OAuth 2.0 Authorization Code flow without PKCE. An attacker installs a malicious app on the user's device that registers a custom URL scheme (`mybank://oauth-callback`). When the legitimate app redirects to the authorization server, the malicious app intercepts the callback with the authorization code. The attacker exchanges the code for tokens and gains access to the user's bank account.

**Resolution:** Implement PKCE (Proof Key for Code Exchange). The mobile app generates a random `code_verifier` (128 characters), computes `code_challenge = SHA256(code_verifier)`, and sends the challenge in the authorization request. When exchanging the code for tokens, the app sends the `code_verifier`. The authorization server verifies that `SHA256(code_verifier)` matches the `code_challenge`. Even if the attacker intercepts the authorization code, they cannot exchange it without the `code_verifier`, which never leaves the app.

```java
// PKCE code challenge generation in mobile app
SecureRandom secureRandom = new SecureRandom();
byte[] codeVerifierBytes = new byte[96];  // 96 bytes → 128 base64 chars
secureRandom.nextBytes(codeVerifierBytes);
String codeVerifier = Base64.getUrlEncoder().withoutPadding()
    .encodeToString(codeVerifierBytes);

MessageDigest md = MessageDigest.getInstance("SHA-256");
byte[] challengeBytes = md.digest(codeVerifier.getBytes(StandardCharsets.US_ASCII));
String codeChallenge = Base64.getUrlEncoder().withoutPadding()
    .encodeToString(challengeBytes);

// Send with authorization request
authRequestUri += "&code_challenge=" + codeChallenge + "&code_challenge_method=S256";
```

### Scenario 2: Microservice Authorization Without Centralized Token Validation
**Context:** A platform with 15 microservices initially validates OAuth tokens by calling the Authorization Server's introspection endpoint on every request. This creates 15× the load on the Auth Server and adds 30-50ms of latency per request. During peak traffic, the Auth Server becomes a bottleneck, causing cascading failures.

**Resolution:** Switch to JWT-based access tokens signed with RS256. Each service fetches the public keys from the Auth Server's JWKS endpoint once (cached for hours) and validates tokens locally. Token introspection is eliminated from the request path. The Auth Server now only handles token issuance and occasional key rotation. This reduces per-request validation time from 50ms to <1ms and removes the Auth Server as a bottleneck. Trade-off: token revocation is no longer immediate — use short token TTLs (15 min) combined with a distributed blacklist for emergency revocations.

### Scenario 3: Cross-Origin Logout with Multiple Clients
**Context:** A user is logged into three apps: a web app (SPA), a mobile app, and a desktop app. All use the same OAuth authorization server. The user logs out from the SPA, but the mobile and desktop apps still have valid sessions. The user expects all sessions to be terminated.

**Resolution:** Implement a centralized logout (Single Logout / SLO). The SPA calls the Auth Server's `/logout` endpoint with the ID token hint. The Auth Server: (1) invalidates all refresh tokens for the user, (2) redirects to each registered client's post-logout URI (OpenID Connect Front-Channel or Back-Channel Logout), and (3) clears the Auth Server session. Each client receives a logout notification and clears its local session. For apps that are offline, the session remains valid until token expiry (trade-off: SLO is best-effort, not guaranteed).

---

## Scenario-Based Questions

1. **Q: Your mobile app's OAuth implementation doesn't use PKCE. A security auditor flags this as critical. The app runs on both iOS and Android, and the authorization server supports PKCE. What changes do you make?**
    - A: (1) Generate a cryptographically random `code_verifier` on the client (128 chars, base64url). (2) Compute `code_challenge = SHA256(code_verifier)`. (3) Add `code_challenge` and `code_challenge_method=S256` to the authorization request. (4) Send `code_verifier` with the token exchange request. (5) The Auth Server verifies the challenge and rejects if it doesn't match. Even if an attacker intercepts the authorization code via a malicious app or man-in-the-middle, they cannot exchange it without the `code_verifier`. PKCE is required by OAuth 2.1 and recommended even for confidential clients as defense-in-depth.
    > **Interview follow-up:** PKCE protects against authorization code interception, but what protects against an attacker who compromises the client application itself and steals the `code_verifier` from memory before it's used? Can the attacker still exchange the code?

2. **Q: Your SPA uses the Implicit Grant (OAuth 2.0). The access token is in the URL fragment. An attacker exploits an XSS vulnerability in the SPA and steals the token from memory. How do you mitigate this, and why should you move away from Implicit Grant?**
   - A: Implicit Grant is deprecated in OAuth 2.1. Problems: (1) Token in URL fragment is exposed in browser history and server logs. (2) No refresh token support (SPAs were considered unable to protect client secrets). (3) No token binding — if stolen, the token is usable from any client. Migration path: switch to Authorization Code with PKCE. The Auth Server redirects with an authorization code (not a token), which is exchanged server-side (or via web worker in the SPA). PKCE protects against code interception. Refresh tokens can be issued as httpOnly cookies. This provides a more secure, standards-compliant solution.

3. **Q: Your OAuth authorization server issues access tokens with a symmetric key (HS256). You have 20 resource servers that all need to validate tokens. A developer accidentally commits the HS256 secret to a public repository. What do you do?**
    - A: (1) Rotate the HS256 secret immediately — generate a new one. (2) Invalidate all existing tokens (force re-authentication or re-issue). (3) All 20 resource servers need the new secret — coordinated deployment required. (4) Migrate to RS256 (asymmetric): the Auth Server uses a private key to sign, resource servers use a public key from JWKS. No shared secret to leak. Key rotation is simpler — update JWKS, no coordinated deployments to 20 services. This is the standard recommendation for distributed systems with multiple resource servers.
    > **Interview follow-up:** Migrating from HS256 to RS256 means all 20 services must be updated to fetch JWKS and validate asymmetrically. If you deploy the JWKS endpoint before any service is updated, old services using HS256 will reject valid tokens signed with the new RS256 key. How do you sequence this migration without downtime?

4. **Q: A user reports that when they click "Login with Google" on your app, the redirect brings them back to an error page saying "Invalid state parameter." Users who don't see this error can log in fine. What is happening, and how do you fix it?**
   - A: The `state` parameter is a CSRF token sent in the authorization request and validated on callback. The error means the state in the callback doesn't match the state stored in the session. Causes: (1) Browser privacy settings (Safari ITP, Firefox Enhanced Tracking Protection) may clear session storage between the redirect and callback. (2) The user opened the authorization URL in a new tab/window. (3) Multiple tabs: state from one tab is overwritten by another. Fix: store state in sessionStorage (not localStorage) and validate. For browsers that clear session on redirect, use a stateless approach: `state = HMAC(secureRandom, session_id)` — validate without server-side storage.

5. **Q: Your API allows both first-party (your own app) and third-party (external developer) clients. Both use OAuth 2.0. A third-party app has a bug that sends 1000 token requests per second, overloading the authorization server. How do you protect the Auth Server without blocking legitimate traffic?**
    - A: (1) Rate limit per client ID: max 10 token requests per second per client. (2) Separate queues for first-party and third-party token requests — first-party always has priority. (3) Implement client authentication with stronger measures for third-party apps: require `client_assertion` (JWT signed by the client) instead of simple `client_secret`. (4) Monitor and auto-throttle clients showing anomalous behavior. (5) Charge by API usage (or enforce tiers) to disincentivize abuse. (6) Use a CDN/WAF to absorb DDoS-level traffic before it reaches the Auth Server.
    > **Interview follow-up:** With separate queues for first-party and third-party tokens, what happens when a first-party client has a bug and generates 10x its normal traffic? Does the priority queue protect you from your own clients?

6. **Q: Your OAuth implementation has a single redirect URI for all environments (localhost, staging, production). A developer debugging locally uses a different port and the authorization code never arrives. How do you handle multiple redirect URIs securely?**
   - A: (1) Register separate client IDs for each environment (dev, staging, prod) — each with its own redirect URIs. (2) Use exact URI matching (OAuth 2.1 requires this): register `http://localhost:3000/callback`, `https://staging.example.com/callback`, `https://example.com/callback`. (3) Never use wildcard or pattern matching — an attacker could register `https://evil.com` with a redirect_uri starting with `https://example.com`. (4) For local development, use tools like `ngrok` with a stable URL instead of localhost. (5) Validate redirect URIs on the server side with a strict allowlist that rejects unexpected URIs.

7. **Q: A user grants your app access to their Google Drive via OAuth 2.0 with scope `drive.readonly`. Three months later, your app is hacked. The attacker uses the stored refresh token to access the user's Google Drive. The user didn't revoke access. How do you design your system to minimize damage from this scenario?**
   - A: (1) Never store refresh tokens indefinitely: invalidate them if not used for 30 days. (2) Use refresh token rotation: each use issues a new token, old one is invalidated. Theft is detected when the attacker's use invalidates the user's token. (3) Bind the refresh token to the client: use DPoP (Demonstration of Proof-of-Possession) to tie the token to a specific client key pair. A stolen token is useless without the corresponding private key. (4) Request minimal scopes: `drive.metadata.readonly` instead of `drive.readonly` — less data exposed if compromised. (5) Monitor for anomalous usage patterns (new IP, new device) and revoke on suspicion.

8. **Q: Your API gateway validates OAuth tokens and passes them downstream. Service A needs to call Service B on behalf of the user. Service B checks the token's `aud` claim and rejects it because it was issued for Service A. How do you handle this without user re-authentication?**
   - A: (1) Use the OAuth Token Exchange extension (RFC 8693): Service A calls the Auth Server with the user's token and requests a new token for Service B. The Auth Server validates the delegation and issues a new token with `aud: "service-b"`. (2) Use a token exchange API in your Auth Server that accepts the current token and returns a service-specific token. (3) Alternatively, use a "user context" token: a signed JWT containing user identity and claims that all internal services accept without audience validation (trade-off: less secure, but simpler). Recommended approach: token exchange for clear audit trail and scoped permissions.

9. **Q: Your OAuth server issues only opaque tokens (reference tokens). Every resource server must call the introspection endpoint to validate. During peak load, the introspection endpoint times out, causing all authenticated requests to fail. How do you solve this without migrating to JWTs immediately?**
   - A: (1) Cache introspection results: cache the token's validity for 5 minutes (or whatever the token's remaining TTL is). This is safe because opaque tokens are valid until expiry. (2) Use a distributed cache (Redis) shared across all resource servers. (3) Fallback behavior: if introspection times out, allow the request with degraded access (cache the decision for 30 seconds). (4) Circuit breaker on the introspection client: if error rate > 50%, serve from cache for 60 seconds. (5) Parallel migration to JWTs: run both opaque and JWT tokens in parallel during a transition period.

10. **Q: Your app uses OpenID Connect for authentication. A user logs in, and you receive an ID token with `nonce` claim. What is the `nonce` for, and what happens if you don't validate it?**
    - A: The `nonce` parameter prevents replay attacks on ID tokens. Your app generates a random `nonce` and sends it with the authentication request. The ID token includes this `nonce`. You validate that the ID token's `nonce` matches what you sent. Without `nonce` validation, an attacker could intercept an ID token (from a previous session) and replay it to impersonate the user. This is especially important for implicit/hybrid flows where the ID token is returned directly in the redirect. Store the `nonce` in session and validate before creating a session.

---

## Interview Questions

1. **What are the four roles in OAuth 2.0?**
   - A: Resource Owner (the user who authorizes access), Client (the application requesting access), Authorization Server (issues tokens after authentication), Resource Server (validates tokens, serves protected data).

2. **What is the difference between OAuth 2.0 and OpenID Connect?**
   - A: OAuth 2.0 is an authorization framework — it issues access tokens for resource access. OpenID Connect (OIDC) adds an authentication layer on top: ID Token (JWT with user identity), UserInfo endpoint, and standardized scopes (openid, profile, email). OAuth = what you can do; OIDC = who you are.

3. **What is PKCE and why is it needed?**
   - A: PKCE (Proof Key for Code Exchange) protects public clients (mobile apps, SPAs) against authorization code interception. The client generates a random `code_verifier`, sends its hash as `code_challenge` in the auth request, then proves possession by sending the `code_verifier` in the token request. Even if an attacker intercepts the authorization code, they can't exchange it without the verifier.

4. **How does the Authorization Code flow work?**
   - A: 1) Client redirects user to Auth Server with client_id, redirect_uri, scope, state. 2) User authenticates and consents. 3) Auth Server redirects back with authorization code. 4) Client exchanges code + client_secret (and PKCE verifier for public clients) server-side. 5) Auth Server returns access token + refresh token.

5. **What is the Client Credentials grant used for?**
   - A: Server-to-server communication where no user is involved. The client authenticates with its own credentials (client_id + client_secret or client assertion) and receives an access token directly. Used for backend services calling APIs, cron jobs, and inter-service communication.

6. **What is the state parameter and why is it important?**
   - A: A random value sent in the authorization request and validated on callback. Prevents CSRF attacks on the OAuth redirect flow — an attacker cannot inject a malicious authorization code because they don't know the user's state value. Implemented using session storage or HMAC-based stateless tokens.

7. **How do you implement token revocation?**
   - A: Call the Auth Server's `/oauth2/revoke` endpoint with the token and client credentials. For JWTs, also maintain a server-side blacklist by `jti`. On logout: revoke access token (or let it expire), invalidate refresh token server-side. OAuth 2.0 requires the revocation endpoint (RFC 7009).

8. **What is the difference between scopes and roles?**
   - A: Scopes define what a client can do on behalf of the user (e.g., `drive:read`, `email`). Roles define user permissions within the application (e.g., `ADMIN`, `MEMBER`). Scopes are OAuth concepts at the API level; roles are application-level authorization. Scopes = delegated permissions; roles = user privileges.

9. **How do you handle OAuth 2.0 in a microservices architecture?**
   - A: API Gateway validates tokens and passes them downstream. Services validate JWTs locally using cached JWKS. Internal service-to-service calls use Client Credentials grant or token exchange. Stateless session management with short-lived tokens. Avoid introspection calls in the request path.

10. **What changes does OAuth 2.1 introduce?**
    - A: PKCE required for all public clients. Implicit Grant removed (Authorization Code + PKCE replaces it). Resource Owner Password Credentials Grant removed. Refresh tokens must be sender-constrained (DPoP) or use rotation. Redirect URIs use exact matching (no patterns). These changes consolidate security best practices into the spec.

---

## Developer Recommendations

- **Always use PKCE — even for confidential clients** — PKCE was designed for public clients, but it adds defense-in-depth for all clients. If a server's `client_secret` is leaked, PKCE still protects against authorization code interception. The implementation cost is minimal: a few extra random bytes and a hash. OAuth 2.1 requires PKCE for public clients. Make it a default for all OAuth flows, regardless of client type. A real-world breach at a major tech company involved an attacker who obtained a client secret from a compromised backend service — PKCE would have prevented the resulting authorization code interception even with the leaked secret.

- **Use JWT access tokens with asymmetric signatures for distributed systems** — Opaque tokens require introspection on every request, creating a central bottleneck and adding 30-50ms latency per hop. JWT with RS256/ES256 enables local validation in <1ms using cached public keys from JWKS. The trade-off: token revocation is no longer immediate (valid until `exp`). Mitigate with short token lifetimes (15 min) and an optional blacklist for emergency revocations. The performance gain outweighs the revocation delay for most systems.

- **Implement refresh token rotation with theft detection** — Refresh tokens are high-value targets (they issue new access tokens indefinitely). Rotation ensures each refresh token is single-use. If a stolen token is used, the legitimate user's next refresh fails, providing immediate theft detection. Combine rotation with device fingerprinting (IP, user-agent) for additional security. The trade-off: network failures during rotation can leave users logged out — mitigate with a 30-second grace period where the old token remains valid.

- **Use the state parameter for CSRF protection** — Without state, an attacker can craft a malicious authorization request, intercept the callback, and inject the resulting authorization code into the user's session. This would grant the attacker access to the user's account. The state parameter links the authorization request to the callback using a cryptographically random value stored in the user's session. Validate state on every callback and reject mismatches.

- **Centralize token validation in an API gateway** — In a microservices architecture, each service should not independently implement OAuth token validation. The API gateway validates tokens, extracts claims, and passes a standardized user context (by claims or a dedicated internal JWT) to downstream services. This ensures consistent policy enforcement, simplifies auditing, and reduces the attack surface. Internal services can trust the gateway without implementing their own validation logic.

- **Plan for key rotation from day one** — Signing keys expire, are compromised, or need algorithm upgrades. From launch, support multiple keys in JWKS. Use the `kid` header to identify which key signed the token. When rotating: (1) add the new key to JWKS, (2) wait for all clients to fetch the updated JWKS (monitor cache duration), (3) start signing with the new key, (4) keep old keys in JWKS for token validation until all tokens signed with them expire. This phased approach prevents validation failures during the transition.
- **Use the Authorization Code grant with PKCE for all client types, even confidential ones** — PKCE adds a second factor of protection beyond the client secret. If a confidential client's secret is leaked, PKCE still prevents authorization code interception. The overhead is negligible — a single SHA-256 hash of a random string. Make PKCE the default for every OAuth flow in your authorization server configuration.
- **Implement token exchange for service-to-service delegation** — When Service A has a user token and needs to call Service B, never pass the original token to Service B (wrong audience). Use token exchange (RFC 8693) to obtain a token scoped to Service B. The authorization server validates the original token's identity and issues a new token with the correct audience. This maintains the security boundary between services.
