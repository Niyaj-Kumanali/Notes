# API Testing

---

## Overview

- **Definition:** API testing validates that an API (typically REST or GraphQL) behaves correctly in terms of functionality, performance, security, and error handling — independent of the UI.
- **Why It Exists:** UIs change frequently; APIs are more stable. Testing at the API layer catches integration issues earlier, is faster than E2E UI tests, and provides the best ROI in the testing pyramid.
- **Key Concepts:** **Endpoint coverage** (all endpoints tested), **HTTP methods** (GET, POST, PUT, PATCH, DELETE), **Status codes** (2xx success, 4xx client error, 5xx server error), **Request/Response validation**, **Auth token handling**, **Rate limiting**, **Idempotency**.

---

## Core Concepts

### Testing Pyramid Layers (API Focus)

- **Unit Tests** (bottom) — Test individual functions/classes in isolation
- **Integration Tests** — Test how components interact (API + DB)
- **Contract Tests** — Verify API provider-consumer agreements
- **E2E Tests** (top) — Full system through the UI

### RESTful API Testing Checklist

- **CRUD Operations:** Create (POST 201), Read (GET 200), Update (PUT/PATCH 200), Delete (DELETE 204)
- **Edge Cases:** Empty body, missing fields, invalid data types, boundary values
- **Error Handling:** 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 500 Internal Server Error
- **Headers:** Content-Type, Accept, Authorization, X-Idempotency-Key, Rate-Limit headers
- **Pagination:** Page size, cursor-based pagination, empty pages, out-of-range pages
- **Filtering & Sorting:** Invalid field names, combined filters, case sensitivity
- **Idempotency:** Repeated POST with same idempotency key returns same result
- **Security:** SQL injection, XSS, JWT expiration, role-based access, OAuth scopes
- **Performance:** Response time thresholds, concurrent users, payload size limits
- **CORS:** Preflight OPTIONS, allowed origins, allowed headers

```javascript
// Example: Testing a REST API with Supertest
const request = require('supertest');
const app = require('../app');
const { createUser, generateToken } = require('./helpers');

describe('POST /api/users', () => {
  it('should create a user and return 201', async () => {
    const res = await request(app)
      .post('/api/users')
      .send({ name: 'Alice', email: 'alice@test.com' })
      .set('Authorization', `Bearer ${adminToken}`);

    expect(res.status).toBe(201);
    expect(res.body).toHaveProperty('id');
    expect(res.body.name).toBe('Alice');
  });

  it('should return 400 for missing email', async () => {
    const res = await request(app)
      .post('/api/users')
      .send({ name: 'Bob' })
      .set('Authorization', `Bearer ${adminToken}`);

    expect(res.status).toBe(400);
    expect(res.body.error).toMatch(/email.*required/i);
  });

  it('should return 401 without auth token', async () => {
    const res = await request(app)
      .post('/api/users')
      .send({ name: 'Charlie', email: 'charlie@test.com' });

    expect(res.status).toBe(401);
  });
});
```

### GraphQL API Testing

```javascript
describe('GraphQL - getUsers', () => {
  it('should return paginated users', async () => {
    const query = `
      query GetUsers($page: Int, $limit: Int) {
        users(page: $page, limit: $limit) {
          items { id name email }
          total
          hasMore
        }
      }
    `;

    const res = await request(app)
      .post('/graphql')
      .send({ query, variables: { page: 1, limit: 10 } })
      .set('Authorization', `Bearer ${token}`);

    expect(res.status).toBe(200);
    expect(res.body.data.users.items).toHaveLength(10);
    expect(res.body.data.users.hasMore).toBe(true);
  });
});
```

---

## Common Mistakes

- **Testing only the happy path** — 90% of bugs are in error handling
- **Hardcoded test data** — Tests fail when test data disappears or conflicts
- **Shared mutable state between tests** — Test ordering dependencies cause flakiness
- **No auth testing** — Security vulnerabilities from missing auth checks
- **Testing through the UI** — Too slow for API-level feedback; test API directly
- **Ignoring idempotency** — Duplicate requests cause duplicate charges/records
- **No response schema validation** — API changes silently break consumers
- **Single-user testing** — Race conditions don't appear until concurrent users
- **No cleanup** — Test data accumulates and slows the system
- **Fake timers for timeout testing** — Real timeout scenarios behave differently

---

## Key Design Considerations

- **Use real HTTP, not in-process calls** — Supertest or similar that makes real HTTP requests catches middleware, serialization, and network layer issues.
- **Database state management** — Use transactions that roll back after each test, or dedicated test DBs. Never use production data.
- **Schema validation** — Use JSON Schema or OpenAPI validation in tests to ensure API contract compliance on every change.
- **Idempotency key pattern** — Clients generate a UUID per request; server deduplicates. Tests should verify duplicate requests return the same result without side effects.
- **Rate limiting tests** — Verify 429 Too Many Requests after exceeding limits. Reset counters or use test-specific rate limit config.
- **Floating point precision** — JSON numbers are 64-bit; big integers lose precision. Send as strings or use libraries that serialize BigInt correctly.
- **CORS preflight** — Automate OPTIONS request testing for all endpoints; browsers behave differently than CLI tools.
- **Health check endpoints** — `/health`, `/ready`, `/live` endpoints should not require auth and should be tested separately.

---

## Real-World Scenarios

### Scenario 1: Payment API Idempotency
A mobile payment app allows users to pay for orders. Due to network issues, the payment request is sent twice. The user is charged twice. **Fix:** Implement idempotency keys. The client generates a UUID (`Idempotency-Key` header) per request. The server checks if it has processed this key; if yes, returns the cached response. Test: send the same idempotency key twice, verify only one charge and both return 200. Also test: expired key cleanup, key reuse after success/failure.

### Scenario 2: Rate-Limited Public API
A weather API serves 10K developers. One developer's bug causes 1000 req/sec against the free tier, starving other developers. **Fix:** Implement token bucket rate limiting per API key. Return `429 Too Many Requests` with `Retry-After` header. In tests, create dedicated test API keys with low rate limits, verify 429 responses, verify the counter resets, and test concurrent requests from different keys.

### Scenario 3: API Version Migration
You're deprecating v1 of your REST API (removing a deprecated field). Consumers have 30 days to migrate to v2. You need to ensure the v2 contract is correct and v1 deprecation is communicated properly. **Fix:** Maintain both versions with different URL prefixes. Use consumer-driven contract tests (Pact) where each consumer publishes their expectations. The provider verifies against all consumer contracts before deploying v1 deprecation. Test: v2 backward compatibility, correct `Deprecation` and `Sunset` headers on v1, proper 410 Gone after sunset date.

---

## Scenario-Based Questions

1. **Q: You are building a payment API where users can retry failed payments. A network glitch causes the same payment request to arrive twice. How do you ensure users aren't charged twice?**
   A: Implement idempotency keys. Clients send a unique UUID in the `Idempotency-Key` header. The server checks if the key was processed: if yes, return cached response; if no, process and cache. Test: send same key twice — verify 200 both times but only one charge. Edge cases: expired keys, invalid UUID, key reuse after 24h, concurrent requests with same key.

2. **Q: Your public API serves 10K developers. One developer's bug sends 1000 req/sec, starving all other developers. How do you design a rate-limiting solution and test it?**
   A: Use token bucket or sliding window counter per API key. Return `429 Too Many Requests` with `Retry-After` header. For testing: create test keys with low limits (e.g., 5 req/min), send 6 rapid requests — verify 5 succeed, 1 gets 429. Test that the counter resets after the window. Test concurrent requests from different keys to ensure isolation.

3. **Q: You're migrating from REST to GraphQL. Existing mobile clients use REST; new clients use GraphQL. How do you test both without duplicating all tests?**
   A: Build a test factory that creates equivalent test cases for both protocols. The test logic (assertions, edge cases) is shared; only the request/response layer differs. Use a schema-first approach — derive REST endpoints and GraphQL resolvers from the same schema. Validate that both produce identical results for the same operation.

4. **Q: A GET endpoint that returns user data suddenly starts returning 500s. The only change was adding a new column to the database table. What testing gap exists?**
   A: Missing API schema validation tests. The new column changed the response shape (e.g., a non-nullable field returned null). Fix: use OpenAPI/Swagger schema validation in tests — `expect(res.body).toMatchSchema(userSchema)`. Add contract tests that verify the response structure. Also add integration tests that verify the endpoint with the actual database migration.

5. **Q: You're testing a file upload API. A user uploads a 2GB video. Your test only uses 1KB files. The endpoint works in tests but times out in production. What's missing?**
   A: Tests don't cover realistic payload sizes. Add tests for: (1) large files (near the limit) — verify correct handling, (2) files exceeding the limit — expect 413 Payload Too Large, (3) streaming — verify the server doesn't buffer the entire file in memory, (4) concurrent uploads. Monitor memory usage during large upload tests.

6. **Q: A webhook endpoint receives events from Stripe. How do you test that it correctly handles retries, duplicate events, and invalid signatures?**
   A: Use Stripe's test mode to generate real webhook events. For retries: send the same event twice — verify idempotent handling (second call returns same result). For duplicate events: verify the handler uses event IDs for deduplication. For signatures: send a request without signature header (expect 401), with wrong signature (expect 401), with expired timestamp (expect 403).

7. **Q: Your API has 50 endpoints, each with 10+ validation rules (required fields, type checks, length limits, format validation). Manually testing every combination is tedious and error-prone. How do you automate this?**
   A: Use property-based testing (fast-check, hypothesis). Define valid input generators for each field. Write properties: "For any valid input, the endpoint returns 2xx" and "For any invalid input violating exactly one rule, the endpoint returns 4xx with the correct error code." This generates 1000s of test cases automatically. Supplement with boundary value tests for edge cases.

8. **Q: Your API returns cached data for 5 minutes. A test writes new data and immediately reads it, but the read returns stale data. How do you test caching behavior?**
   A: Test caching explicitly: (1) Write data, read it — verify fresh data returned (first request may bypass cache). (2) Write new data, read again within the cache window — verify stale data if `Cache-Control` headers indicate cache hit. (3) Use `Cache-Control: no-cache` header — verify fresh data returned. (4) Test cache invalidation: after a write, the cache should be purged or the resource should be re-fetched.

9. **Q: You run 500 API tests in CI. 10 of them fail randomly about 20% of the time. Developers start ignoring failures. How do you stabilize the test suite?**
   A: Flaky tests destroy trust. Track flakiness with `jest --repeatEach=5` to detect flaky tests. Common causes: (1) Shared database state — use transactions that rollback. (2) Race conditions — add proper awaits. (3) Time-dependent tests — freeze time. (4) Test ordering dependencies — randomize test order. Quarantine flaky tests immediately (skip them, tag as `@flaky`), fix them, then re-enable. Do not tolerate flaky tests.

10. **Q: Your mobile app uses a REST API. A new version of the API changes the response format (field renamed). Old app versions break. How do you test backward compatibility?**
    A: Version your API (`/v1/`, `/v2/`). Keep old versions running until sunset. Write version-specific tests that run against each active version. Use API versioning in tests: `test('v1 user format', () => { ... })`, `test('v2 user format', () => { ... })`. Add a "compatibility suite" that runs old version tests against the current version to ensure no breaking changes slip through.

---

## Interview Questions

1. **What is the difference between REST and GraphQL API testing?**
   A: REST tests fixed endpoints with specific HTTP methods; GraphQL tests a single endpoint with varying queries. REST tests focus on HTTP status codes; GraphQL tests focus on response data structure (200 with errors in body).

2. **What HTTP status codes should you test?**
   A: 200 (success), 201 (created), 204 (no content), 400 (bad request), 401 (unauthorized), 403 (forbidden), 404 (not found), 409 (conflict), 422 (unprocessable), 429 (rate limited), 500 (server error), 503 (service unavailable).

3. **What is idempotency in API testing?**
   A: An operation is idempotent if repeated requests produce the same result. GET, PUT, DELETE are idempotent; POST is not. Test by sending the same request multiple times and asserting identical results.

4. **What is an idempotency key and how do you test it?**
   A: A client-generated UUID sent in a header to prevent duplicate processing. Test: send the same key twice, assert both return 200 but only one side effect (e.g., one charge).

5. **What is the difference between contract testing and integration testing for APIs?**
   A: Contract testing verifies the API provider meets consumer expectations (schema, behavior). Integration testing verifies the provider works correctly with real dependencies (DB, other services).

6. **How do you test API pagination?**
   A: Test: first page returns expected items, cursor/offset advances correctly, last page has `hasMore: false`, out-of-range page returns empty, concurrent inserts don't skip/duplicate items.

7. **What is schema validation in API testing?**
   A: Asserting the response body matches a predefined schema (OpenAPI, JSON Schema). Catches: missing fields, wrong types, extra unexpected fields. Tools: `jest-json-schema`, `swagger-validator`.

8. **How do you test API authentication and authorization?**
   A: Test: endpoint without token (expect 401), with invalid token (expect 401), with expired token (expect 401), with insufficient role (expect 403), with valid token (expect 200).

9. **What is the testing pyramid for APIs?**
   A: Unit tests (bottom, many) → Integration tests (middle) → Contract tests → E2E tests (top, few). API tests sit at the integration level — test endpoints with real HTTP but controlled dependencies.

10. **How do you test error responses?**
    A: For each validation rule: test violating it alone, violating multiple rules, boundary values, and missing optional fields. Assert: correct HTTP status, structured error body (code, message, field), and appropriate headers.

---

## Developer Recommendations

- **Test error paths as thoroughly as happy paths** — 90% of API bugs are in error handling. For every endpoint, test: missing required fields, invalid types, out-of-range values, unauthorized access, and internal server errors. Trade-off: more test code than happy-path tests. Benefit: the most common production failures are caught before deployment.

- **Use real HTTP for API tests, not in-process calls** — `supertest` makes real HTTP requests through your Express/Koa app, catching middleware, serialization, and header issues that in-process calls miss. Trade-off: slightly slower (sub-millisecond HTTP overhead). Benefit: tests match real client behavior exactly.

- **Implement and test idempotency keys for all mutating endpoints** — Duplicate requests happen (network retries, user double-click). Idempotency prevents double charges, duplicate records, and inconsistent state. Trade-off: clients must generate and track UUIDs. Benefit: safe retry semantics without side effects.

- **Use consumer-driven contract tests to prevent breaking changes** — When your API changes, consumer contracts catch downstream breakage before deploy. Pact or OpenAPI-based contract testing integrates into CI. Trade-off: setup overhead and contract maintenance. Benefit: zero integration surprises during deployment.

- **Test rate limiting with dedicated test keys** — Use API keys with artificially low limits in tests. Verify 429 responses, `Retry-After` headers, and counter reset after the window. Trade-off: test-specific rate limit config adds complexity. Benefit: rate limiting code is verified under controlled conditions.

- **Use structured error responses, not string messages** — Return `{ code: "VALIDATION_ERROR", field: "email", message: "..." }` instead of just `"Email is invalid"`. Test against error codes, not error strings. Trade-off: larger response payloads. Benefit: consumers don't break when error messages are improved or localized.

- **Freeze time in tests** — API responses often include timestamps. Use `vi.useFakeTimers()` (Vitest) or `jest.useFakeTimers()` (Jest) to freeze time at a known value. Assert with `expect.any(String)` for timestamps or exact values after freezing. Trade-off: frozen time may skip edge cases around DST, leap years. Benefit: deterministic, reproducible tests.

- **Version your API from day one** — Even before you need it, prefix endpoints with `/v1/`. Adding a version later requires migrating all existing consumers. Tests should version-stamp their assertions. Trade-off: URLs are slightly longer. Benefit: you can evolve the API without breaking existing clients.
