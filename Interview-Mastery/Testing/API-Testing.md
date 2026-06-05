# API Testing

## 1. Executive Summary

API testing validates that an application programming interface meets its functional, reliability, performance, and security requirements. It operates at the contract layer between services, making it distinct from UI testing (which validates visual elements) and unit testing (which validates internal logic). API tests send real HTTP requests with controlled payloads and assert on response status, headers, body shape, and timing. This document focuses on REST API testing using JavaScript/TypeScript with supertest for automation and Postman concepts for manual and exploratory testing. Well-designed API tests catch contract violations, auth bypasses, data leaks, and performance regressions before they reach production.

## 2. Core Theory

### 2.1 API Testing Layers

- **Functional testing**: Does the endpoint return the correct response for a given input?
- **Validation testing**: Does the endpoint reject invalid inputs with appropriate errors?
- **Security testing**: Does the endpoint enforce authentication, authorization, and rate limiting?
- **Performance testing**: Does the endpoint respond within acceptable time under load?
- **Contract testing**: Does the endpoint conform to its OpenAPI/Swagger specification?

### 2.2 HTTP Fundamentals

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| GET    | Retrieve resource | Yes | Yes |
| POST   | Create resource | No | No |
| PUT    | Replace resource | Yes | No |
| PATCH  | Partial update | No | No |
| DELETE | Remove resource | Yes | No |

### 2.3 HTTP Status Code Ranges

- **1xx**: Informational (100 Continue, 101 Switching Protocols)
- **2xx**: Success (200 OK, 201 Created, 204 No Content)
- **3xx**: Redirection (301 Moved Permanently, 304 Not Modified)
- **4xx**: Client Error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 422 Unprocessable Entity, 429 Too Many Requests)
- **5xx**: Server Error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout)

### 2.4 Test Case Structure

Every API test case should cover:

- **Request**: Method, URL, headers, query params, body
- **Expected response**: Status code, headers, body structure, body values
- **Side effects**: Database state, external calls, emitted events
- **Edge cases**: Empty body, missing fields, boundary values, malformed input

## 3. Under-the-Hood Deep Dive

### 3.1 HTTP Request/Response Lifecycle

```
 Client                          Server
   |                               |
   |--- TCP Handshake (SYN/SYN-ACK/ACK) -->|
   |                               |
   |--- TLS Handshake (if HTTPS) -------->|
   |                               |
   |--- HTTP Request ------------------->|
   |                               |--- Parse request line & headers
   |                               |--- Parse body (if any)
   |                               |--- Route matching
   |                               |--- Middleware execution
   |                               |--- Handler execution
   |                               |--- Response serialization
   |<-- HTTP Response -------------------|
   |                               |
   |--- Connection close or keep-alive   |
```

### 3.2 Postman Runtime Internals

Postman uses a Chrome V8 runtime (via Electron) to execute scripts. The request flow is:

1. **Prerequest script**: Runs before the request is sent. Used to set variables, compute signatures.
2. **Send request**: Postman sends the HTTP request and waits for the response.
3. **Test script**: Runs after the response is received. Used for assertions (`pm.test`, `pm.expect`).

Postman collections can be exported as JSON and run with Newman (CLI) in CI pipelines.

### 3.3 How supertest Chains Work

Supertest uses a fluent interface. Each method returns the `Test` object, allowing chaining. Internally, `end()` or `expect()` triggers the actual HTTP call:

```typescript
// What happens when you call .expect(200)
request(app)
  .get('/users')
  .expect(200, (err, res) => {
    // callback style
  });

// With async/await, end() is called automatically:
const res = await request(app).get('/users').expect(200);
// .expect(200) returns a Promise when awaited
```

### 3.4 Content Negotiation

Servers use the `Accept` request header to determine response format:

```typescript
// Request JSON
request(app)
  .get('/api/users/1')
  .set('Accept', 'application/json')
  .expect('Content-Type', /json/);

// Request XML (if supported)
request(app)
  .get('/api/users/1')
  .set('Accept', 'application/xml')
  .expect('Content-Type', /xml/);
```

## 4. Production Code Examples

### 4.1 Basic CRUD API Tests with Supertest

```typescript
// src/__tests__/api/users.api.test.ts
import request from 'supertest';
import { app } from '../app';
import { db } from '../db';

describe('Users API', () => {
  let authToken: string;

  beforeAll(async () => {
    await db.migrate.latest();
    const res = await request(app)
      .post('/api/auth/login')
      .send({ username: 'admin', password: 'secret' });
    authToken = res.body.token;
  });

  afterAll(async () => {
    await db.destroy();
  });

  describe('POST /api/users', () => {
    it('creates a user with valid data', async () => {
      const res = await request(app)
        .post('/api/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          name: 'John Doe',
          email: 'john@example.com',
          role: 'user',
        })
        .expect(201)
        .expect('Content-Type', /json/);

      expect(res.body).toMatchObject({
        name: 'John Doe',
        email: 'john@example.com',
      });
      expect(res.body).toHaveProperty('id');
      expect(res.body).not.toHaveProperty('password');
    });

    it('returns 422 for missing required fields', async () => {
      const res = await request(app)
        .post('/api/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({})
        .expect(422);

      expect(res.body.errors).toBeDefined();
    });

    it('returns 409 for duplicate email', async () => {
      await request(app)
        .post('/api/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'A', email: 'dup@example.com' })
        .expect(201);

      await request(app)
        .post('/api/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'B', email: 'dup@example.com' })
        .expect(409);
    });
  });

  describe('GET /api/users/:id', () => {
    it('returns a user by ID', async () => {
      const res = await request(app)
        .get('/api/users/1')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(res.body).toMatchObject({
        id: 1,
        name: expect.any(String),
      });
    });

    it('returns 404 for non-existent user', async () => {
      await request(app)
        .get('/api/users/99999')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(404);
    });

    it('returns 400 for invalid ID format', async () => {
      await request(app)
        .get('/api/users/abc')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(400);
    });
  });

  describe('PUT /api/users/:id', () => {
    it('fully updates a user', async () => {
      const res = await request(app)
        .put('/api/users/1')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Updated Name', email: 'updated@example.com' })
        .expect(200);

      expect(res.body.name).toBe('Updated Name');
    });
  });

  describe('PATCH /api/users/:id', () => {
    it('partially updates a user', async () => {
      const res = await request(app)
        .patch('/api/users/1')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Patched Name' })
        .expect(200);

      expect(res.body.name).toBe('Patched Name');
    });
  });

  describe('DELETE /api/users/:id', () => {
    it('deletes a user', async () => {
      await request(app)
        .delete('/api/users/1')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(204);
    });
  });
});
```

### 4.2 Testing Query Parameters and Pagination

```typescript
describe('GET /api/users', () => {
  it('returns paginated results', async () => {
    const res = await request(app)
      .get('/api/users')
      .query({ page: 1, limit: 10 })
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);

    expect(res.body).toMatchObject({
      data: expect.any(Array),
      meta: {
        page: 1,
        limit: 10,
        total: expect.any(Number),
      },
    });
    expect(res.body.data.length).toBeLessThanOrEqual(10);
  });

  it('supports sorting', async () => {
    const res = await request(app)
      .get('/api/users')
      .query({ sort: 'name:asc' })
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);

    const names = res.body.data.map((u: any) => u.name);
    expect(names).toEqual([...names].sort());
  });

  it('supports filtering by field', async () => {
    const res = await request(app)
      .get('/api/users')
      .query({ filter: 'role:admin' })
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);

    res.body.data.forEach((user: any) => {
      expect(user.role).toBe('admin');
    });
  });
});
```

### 4.3 Testing Headers and CORS

```typescript
describe('CORS headers', () => {
  it('includes CORS headers on cross-origin requests', async () => {
    const res = await request(app)
      .get('/api/public')
      .set('Origin', 'https://example.com')
      .expect(200);

    expect(res.headers['access-control-allow-origin']).toBe('*');
  });

  it('handles preflight OPTIONS request', async () => {
    const res = await request(app)
      .options('/api/users')
      .set('Origin', 'https://example.com')
      .set('Access-Control-Request-Method', 'POST')
      .expect(200);

    expect(res.headers['access-control-allow-methods']).toContain('POST');
  });
});
```

### 4.4 Testing Conditional Requests (ETag/If-None-Match)

```typescript
describe('ETag caching', () => {
  it('returns 304 when resource has not changed', async () => {
    const res1 = await request(app)
      .get('/api/users/1')
      .set('Authorization', `Bearer ${authToken}`);

    const etag = res1.headers.etag;

    const res2 = await request(app)
      .get('/api/users/1')
      .set('Authorization', `Bearer ${authToken}`)
      .set('If-None-Match', etag)
      .expect(304);

    expect(res2.body).toStrictEqual({});
  });
});
```

### 4.5 Postman Collection as Code

```typescript
// postman/collections/users.ts
export const usersCollection = {
  info: {
    name: 'Users API',
    schema: 'https://schema.getpostman.com/json/collection/v2.1.0/collection.json',
  },
  item: [
    {
      name: 'Create User',
      event: [
        {
          listen: 'test',
          script: {
            exec: [
              'pm.test("Status code is 201", () => {',
              '  pm.response.to.have.status(201);',
              '});',
              'pm.test("Response has id", () => {',
              '  pm.expect(pm.response.json()).to.have.property("id");',
              '});',
            ],
          },
        },
      ],
      request: {
        method: 'POST',
        header: [
          { key: 'Content-Type', value: 'application/json' },
          { key: 'Authorization', value: 'Bearer {{authToken}}' },
        ],
        body: {
          mode: 'raw',
          raw: JSON.stringify({
            name: '{{$randomFullName}}',
            email: '{{$randomEmail}}',
          }),
        },
        url: {
          raw: '{{baseUrl}}/api/users',
          host: ['{{baseUrl}}'],
          path: ['api', 'users'],
        },
      },
    },
  ],
};
```

### 4.6 Running Newman in CI

```typescript
// postman/run-newman.ts
import newman from 'newman';

newman.run({
  collection: require('./collections/users.json'),
  environment: require('./environments/test.json'),
  reporters: ['cli', 'junit'],
  iterationCount: 1,
  timeout: 10000,
}, (err) => {
  if (err) { throw err; }
});
```

### 4.7 Testing File Download

```typescript
describe('GET /api/reports/:id/download', () => {
  it('downloads a file as a stream', async () => {
    const res = await request(app)
      .get('/api/reports/1/download')
      .set('Authorization', `Bearer ${authToken}`)
      .buffer(true) // receive full body
      .expect(200)
      .expect('Content-Type', /pdf/)
      .expect('Content-Disposition', /attachment/);

    expect(res.body).toBeInstanceOf(Buffer);
    expect(res.body.length).toBeGreaterThan(0);
  });
});
```

### 4.8 Testing Error Responses

```typescript
describe('Error response format', () => {
  it('returns consistent error shape', async () => {
    const res = await request(app)
      .get('/api/nonexistent')
      .set('Authorization', `Bearer ${authToken}`)
      .expect(404);

    expect(res.body).toMatchObject({
      error: {
        code: 'NOT_FOUND',
        message: expect.any(String),
      },
    });
  });

  it('returns validation errors in standard format', async () => {
    const res = await request(app)
      .post('/api/users')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ email: 'not-an-email' })
      .expect(422);

    expect(res.body.errors).toEqual(
      expect.arrayContaining([
        expect.objectContaining({
          field: 'email',
          message: expect.any(String),
        }),
      ])
    );
  });
});
```

## 5. Real-World Scenarios

### 5.1 Testing OAuth2 Flows

```typescript
describe('OAuth2 authorization_code flow', () => {
  it('exchanges code for token', async () => {
    // Step 1: Get authorization code (simulate user login)
    const authRes = await request(app)
      .post('/api/oauth/authorize')
      .send({
        client_id: 'test-client',
        redirect_uri: 'http://localhost/callback',
        response_type: 'code',
        username: 'user',
        password: 'pass',
      })
      .expect(302);

    const code = extractCode(authRes.headers.location);

    // Step 2: Exchange code for token
    const tokenRes = await request(app)
      .post('/api/oauth/token')
      .send({
        grant_type: 'authorization_code',
        code,
        client_id: 'test-client',
        client_secret: 'test-secret',
      })
      .expect(200);

    expect(tokenRes.body).toHaveProperty('access_token');
    expect(tokenRes.body).toHaveProperty('refresh_token');
    expect(tokenRes.body.token_type).toBe('Bearer');
  });

  it('rejects invalid authorization code', async () => {
    await request(app)
      .post('/api/oauth/token')
      .send({
        grant_type: 'authorization_code',
        code: 'invalid-code',
        client_id: 'test-client',
        client_secret: 'test-secret',
      })
      .expect(400);
  });
});
```

### 5.2 Testing Webhook Delivery

```typescript
describe('Webhook endpoint', () => {
  it('accepts valid webhook payload', async () => {
    const payload = { event: 'order.created', data: { orderId: 123 } };
    const signature = crypto
      .createHmac('sha256', WEBHOOK_SECRET)
      .update(JSON.stringify(payload))
      .digest('hex');

    await request(app)
      .post('/api/webhooks/stripe')
      .set('Content-Type', 'application/json')
      .set('Stripe-Signature', `t=123456,v1=${signature}`)
      .send(payload)
      .expect(200);
  });

  it('rejects webhook with invalid signature', async () => {
    const payload = { event: 'order.created' };
    await request(app)
      .post('/api/webhooks/stripe')
      .set('Stripe-Signature', 't=123456,v1=invalid')
      .send(payload)
      .expect(401);
  });

  it('handles duplicate webhook events idempotently', async () => {
    const payload = { id: 'evt_dup', event: 'order.created' };
    const signature = signWebhook(payload);

    await request(app)
      .post('/api/webhooks/stripe')
      .set('Stripe-Signature', signature)
      .send(payload)
      .expect(200);

    await request(app)
      .post('/api/webhooks/stripe')
      .set('Stripe-Signature', signature)
      .send(payload)
      .expect(200); // Should not create duplicate
  });
});
```

### 5.3 Testing Batch Operations

```typescript
describe('POST /api/batch', () => {
  it('processes multiple operations', async () => {
    const res = await request(app)
      .post('/api/batch')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        requests: [
          { method: 'GET', path: '/api/users/1' },
          { method: 'GET', path: '/api/users/2' },
        ],
      })
      .expect(200);

    expect(res.body.results).toHaveLength(2);
    expect(res.body.results[0].status).toBe(200);
    expect(res.body.results[1].status).toBe(200);
  });

  it('returns partial success for mixed operations', async () => {
    const res = await request(app)
      .post('/api/batch')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        requests: [
          { method: 'GET', path: '/api/users/1' },
          { method: 'GET', path: '/api/users/99999' },
        ],
      })
      .expect(200);

    expect(res.body.results[0].status).toBe(200);
    expect(res.body.results[1].status).toBe(404);
  });
});
```

### 5.4 Testing File Upload with Metadata

```typescript
describe('POST /api/documents/upload', () => {
  it('uploads a file with metadata', async () => {
    const res = await request(app)
      .post('/api/documents/upload')
      .set('Authorization', `Bearer ${authToken}`)
      .field('description', 'Test document')
      .field('tags', 'important,urgent')
      .attach('file', Buffer.from('test content'), 'test.txt')
      .expect(201);

    expect(res.body).toMatchObject({
      filename: 'test.txt',
      description: 'Test document',
      size: 12,
    });
  });

  it('rejects uploads exceeding size limit', async () => {
    const largeBuffer = Buffer.alloc(11 * 1024 * 1024); // 11 MB
    await request(app)
      .post('/api/documents/upload')
      .set('Authorization', `Bearer ${authToken}`)
      .attach('file', largeBuffer, 'large.txt')
      .expect(413);
  });

  it('rejects unsupported file types', async () => {
    await request(app)
      .post('/api/documents/upload')
      .set('Authorization', `Bearer ${authToken}`)
      .attach('file', Buffer.from('<xml/>'), 'malware.exe')
      .expect(415);
  });
});
```

## 6. Performance

### 6.1 Response Time Validation

```typescript
it('responds within 200ms for cached resources', async () => {
  const start = Date.now();
  await request(app)
    .get('/api/users/1')
    .set('Authorization', `Bearer ${authToken}`)
    .expect(200);
  const duration = Date.now() - start;

  expect(duration).toBeLessThan(200);
});

it('responds within 500ms for uncached resources', async () => {
  const start = Date.now();
  await request(app)
    .get('/api/reports/slow')
    .set('Authorization', `Bearer ${authToken}`)
    .expect(200);
  const duration = Date.now() - start;

  expect(duration).toBeLessThan(500);
});
```

### 6.2 Load Testing with k6 (Script Example)

```javascript
// k6-scripts/api-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },  // Ramp-up
    { duration: '1m', target: 20 },   // Steady state
    { duration: '30s', target: 0 },   // Ramp-down
  ],
  thresholds: {
    http_req_duration: ['p(95) < 500'], // 95% of requests under 500ms
    http_req_failed: ['rate < 0.01'],   // Less than 1% failure
  },
};

export default function () {
  const res = http.get('http://localhost:3000/api/users', {
    headers: { Authorization: `Bearer ${__ENV.AUTH_TOKEN}` },
  });

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 300ms': (r) => r.timings.duration < 300,
  });

  sleep(1);
}
```

### 6.3 API Performance Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| Response time (p50) | Median response time | < 200ms |
| Response time (p95) | 95th percentile | < 500ms |
| Response time (p99) | 99th percentile | < 1000ms |
| Error rate | Percentage of 5xx responses | < 0.1% |
| Throughput | Requests per second | Varies by endpoint |
| Payload size | Response body size | < 100KB (list) |

## 7. Security

### 7.1 Authentication Testing

```typescript
describe('Authentication', () => {
  it('rejects request without auth header', async () => {
    await request(app)
      .get('/api/users')
      .expect(401);
  });

  it('rejects malformed auth header', async () => {
    await request(app)
      .get('/api/users')
      .set('Authorization', 'NotABearer token')
      .expect(401);
  });

  it('rejects expired JWT', async () => {
    const expiredToken = jwt.sign(
      { userId: 1 },
      process.env.JWT_SECRET!,
      { expiresIn: '0s' }
    );
    await request(app)
      .get('/api/users')
      .set('Authorization', `Bearer ${expiredToken}`)
      .expect(401);
  });

  it('rejects JWT with wrong secret', async () => {
    const forgedToken = jwt.sign(
      { userId: 1, role: 'admin' },
      'wrong-secret'
    );
    await request(app)
      .delete('/api/users/2')
      .set('Authorization', `Bearer ${forgedToken}`)
      .expect(401);
  });
});
```

### 7.2 Authorization Testing

```typescript
describe('RBAC enforcement', () => {
  let userToken: string;
  let adminToken: string;

  beforeAll(async () => {
    userToken = await loginAs('regular-user', 'pass');
    adminToken = await loginAs('admin', 'pass');
  });

  it('allows admin to delete users', async () => {
    await request(app)
      .delete('/api/users/2')
      .set('Authorization', `Bearer ${adminToken}`)
      .expect(204);
  });

  it('blocks non-admin from deleting users', async () => {
    await request(app)
      .delete('/api/users/2')
      .set('Authorization', `Bearer ${userToken}`)
      .expect(403);
  });

  it('prevents user from accessing another user data', async () => {
    const userAToken = await loginAs('usera', 'pass');
    const userBToken = await loginAs('userb', 'pass');

    // User A should not access User B's data
    await request(app)
      .get('/api/users/2') // Assuming userB has id=2
      .set('Authorization', `Bearer ${userAToken}`)
      .expect(403);
  });
});
```

### 7.3 Input Validation Testing

```typescript
describe('Input validation', () => {
  it('rejects excessively long input', async () => {
    const longName = 'x'.repeat(1000);
    await request(app)
      .post('/api/users')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ name: longName, email: 'test@test.com' })
      .expect(422);
  });

  it('rejects invalid email format', async () => {
    await request(app)
      .post('/api/users')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ name: 'Test', email: 'not-an-email' })
      .expect(422);
  });

  it('rejects prototype pollution attempts', async () => {
    await request(app)
      .post('/api/users')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ __proto__: { admin: true } })
      .expect(422);
  });

  it('rejects SQL injection in query params', async () => {
    await request(app)
      .get('/api/users')
      .query({ id: "1; DROP TABLE users--" })
      .set('Authorization', `Bearer ${authToken}`)
      .expect(400);
  });

  it('rejects XSS in body fields', async () => {
    await request(app)
      .post('/api/users')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ name: '<script>alert("xss")</script>', email: 'xss@test.com' })
      .expect(422);
  });
});
```

### 7.4 Rate Limiting and DoS Protection

```typescript
describe('Rate limiting', () => {
  it('blocks requests exceeding rate limit', async () => {
    const requests = Array.from({ length: 101 }, () =>
      request(app)
        .get('/api/public')
        .set('Authorization', `Bearer ${authToken}`)
    );

    const results = await Promise.all(requests);
    const tooMany = results.filter(r => r.status === 429);
    expect(tooMany.length).toBeGreaterThan(0);
  });

  it('includes rate limit headers', async () => {
    const res = await request(app)
      .get('/api/public')
      .set('Authorization', `Bearer ${authToken}`);

    expect(res.headers['x-ratelimit-limit']).toBeDefined();
    expect(res.headers['x-ratelimit-remaining']).toBeDefined();
    expect(res.headers['x-ratelimit-reset']).toBeDefined();
  });
});
```

### 7.5 HTTPS and Security Headers

```typescript
describe('Security headers', () => {
  it('includes security headers in responses', async () => {
    const res = await request(app)
      .get('/api/public')
      .expect(200);

    expect(res.headers['x-frame-options']).toBe('DENY');
    expect(res.headers['x-content-type-options']).toBe('nosniff');
    expect(res.headers['strict-transport-security']).toBeDefined();
    expect(res.headers['x-xss-protection']).toBe('0');
  });
});
```

## 8. Common Mistakes

### 8.1 Testing Against Production

Never run automated API tests against a production environment. Always use a dedicated test/staging environment with test data.

### 8.2 Hardcoding Test Data

```typescript
// BAD: fragile, assumes database state
it('returns user', async () => {
  const res = await request(app)
    .get('/api/users/1')
    .set('Authorization', 'Bearer test-token');
  expect(res.body.name).toBe('specific-name');
});

// GOOD: create data in the test
it('returns user', async () => {
  const { id } = await createTestUser({ name: 'Alice' });
  const res = await request(app)
    .get(`/api/users/${id}`)
    .set('Authorization', `Bearer ${authToken}`);
  expect(res.body.name).toBe('Alice');
});
```

### 8.3 Not Testing Error Responses

Many test suites only test the happy path. Always test:

- 400 Bad Request (invalid input)
- 401 Unauthorized (no auth)
- 403 Forbidden (insufficient permissions)
- 404 Not Found (non-existent resource)
- 409 Conflict (duplicate)
- 422 Unprocessable Entity (validation failure)
- 429 Too Many Requests (rate limit)
- 500 Internal Server Error (server failure)

### 8.4 Ignoring Response Headers

Headers carry critical information: content type, caching directives, rate limits, CORS. Always assert on them.

### 8.5 Testing Implementation, Not Contract

```typescript
// BAD: testing implementation
it('calls findById', async () => {
  const spy = jest.spyOn(UserModel, 'findById');
  await request(app).get('/api/users/1');
  expect(spy).toHaveBeenCalledWith('1');
});

// GOOD: testing contract
it('returns user by id', async () => {
  const res = await request(app).get('/api/users/1').expect(200);
  expect(res.body.id).toBe(1);
});
```

### 8.6 Flaky Tests from Shared State

Do not share test data between test files. Each test file should set up and tear down its own data.

### 8.7 Over-relying on Postman for Automation

Postman collections are great for manual and exploratory testing, but for CI, use code-based tests (supertest, jest) that are version-controlled, reviewable, and faster.

## 9. Senior Engineer Perspective

### 9.1 API Testing Strategy

A senior engineer designs the API test suite as a layered defense:

- **Unit tests**: Validate individual handlers and services
- **Contract tests**: Verify the API matches its OpenAPI spec
- **Functional integration tests**: Verify request->response behavior
- **Consumer-driven contracts**: Each consumer specifies its expectations
- **End-to-end smoke tests**: Run against staging after deployment

### 9.2 OpenAPI Contract Testing

```typescript
// openapi-contract.test.ts
import OpenAPIClient from 'openapi-client-axios';
import { app } from '../app';

const client = new OpenAPIClient({
  definition: require('./openapi.json'),
  axiosConfigFactory: () => ({
    adapter: require('axios/lib/adapters/http'),
  }),
});

it('response matches OpenAPI spec', async () => {
  const res = await client.get('/users/{id}', { id: 1 });
  // The client validates the response against the spec
  expect(res.status).toBe(200);
});
```

### 9.3 API Versioning Strategy

Test multiple API versions and verify backward compatibility:

```typescript
describe('API v1 vs v2 compatibility', () => {
  it('v2 returns superset of v1', async () => {
    const v1Res = await request(app)
      .get('/api/v1/users/1')
      .set('Authorization', `Bearer ${authToken}`);

    const v2Res = await request(app)
      .get('/api/v2/users/1')
      .set('Authorization', `Bearer ${authToken}`);

    // v2 should have all v1 fields
    Object.keys(v1Res.body).forEach(key => {
      expect(v2Res.body).toHaveProperty(key);
    });
  });
});
```

### 9.4 API Changelog as Tests

Turn breaking changes into automated tests:

```typescript
// Breaking: User response no longer includes 'age' field
it('response does not include deprecated age field', async () => {
  const res = await request(app)
    .get('/api/v2/users/1')
    .set('Authorization', `Bearer ${authToken}`);

  expect(res.body).not.toHaveProperty('age');
});
```

### 9.5 Consumer-Driven Contract Tests

```typescript
// consumer-test.js (run by the consumer team)
describe('Consumer contract: User Service', () => {
  it('provides user data in expected format', async () => {
    const res = await request(userServiceBaseUrl)
      .get('/api/users/me')
      .set('Authorization', `Bearer ${testToken}`)
      .expect(200);

    // Consumer's expectations
    expect(res.body).toHaveProperty('id');
    expect(res.body).toHaveProperty('name');
    expect(res.body).toHaveProperty('email');
    expect(res.body).not.toHaveProperty('password');
  });
});
```

## 10. Interview Questions (20: 10 Easy + 10 Medium)

### Easy

**Q1**: What is an API test?
**A**: An API test sends HTTP requests to an endpoint and validates the response status, headers, and body against expected values.

**Q2**: What is the difference between REST and SOAP API testing?
**A**: REST uses HTTP methods and typically JSON/XML payloads; SOAP uses XML envelopes over HTTP or other protocols. REST is stateless; SOAP supports stateful operations and has built-in error handling.

**Q3**: What is Postman used for?
**A**: Postman is an API client for designing, testing, and documenting APIs. It supports collections, environments, scripts, and automation via Newman.

**Q4**: What is the purpose of the `Authorization` header?
**A**: It carries credentials (bearer token, basic auth, API key) that the server uses to authenticate the client.

**Q5**: What does the HTTP status code 201 mean?
**A**: 201 Created indicates that a new resource was successfully created as a result of a POST or PUT request.

**Q6**: How do you test a GET endpoint that requires query parameters?
**A**: Use `.query({ key: 'value' })` with supertest, or append `?key=value` to the URL.

**Q7**: What is the difference between 401 and 403?
**A**: 401 Unauthorized means the client is not authenticated. 403 Forbidden means the client is authenticated but does not have permission.

**Q8**: What is Newman?
**A**: Newman is Postman's command-line collection runner. It allows running Postman collections in CI/CD pipelines.

**Q9**: What is a pre-request script in Postman?
**A**: A script that runs before a request is sent. Used to set variables, generate timestamps, or compute request signatures.

**Q10**: What are Postman environments?
**A**: Named sets of variables (e.g., base URL, auth token) that allow the same collection to run against different environments (dev, staging, prod).

### Medium

**Q11**: How do you test an endpoint that requires multipart form data?
**A**: Use `.field('key', 'value')` for form fields and `.attach('file', path)` for file uploads in supertest.

**Q12**: How do you test idempotency of PUT endpoints?
**A**: Send the same PUT request twice and assert that the second response has the same status and body as the first, with no additional side effects.

**Q13**: What is content negotiation and how do you test it?
**A**: Content negotiation is the process where the server selects the best representation based on the `Accept` request header. Test by setting different `Accept` headers and asserting the `Content-Type` of the response.

**Q14**: How do you test API pagination?
**A**: Send requests with `page` and `limit` parameters. Assert that: (a) each page returns the correct number of items, (b) the metadata (total, page count) is correct, (c) pages have no duplicates, (d) requesting a page beyond the last returns an empty array.

**Q15**: How do you handle test data cleanup for API tests?
**A**: (a) Create data in `beforeAll` and delete in `afterAll`, (b) use a dedicated test database that is reset between suites, (c) wrap each test in a database transaction and roll back.

**Q16**: What is the difference between supertest's `.expect(200)` and `.expect(statusCode, body)`?
**A**: `.expect(200)` only checks the status code. `.expect(200, body)` checks both the status code and that the response body deep-equals the provided body.

**Q17**: How do you test WebSocket endpoints?
**A**: Use a WebSocket client library (e.g., `ws`) to connect to the server, send messages, and assert on received messages. Ensure the server is listening on a real port.

**Q18**: How do you test API backward compatibility during a version upgrade?
**A**: Run the old test suite against the new API version. Assert that all existing tests pass. Additionally, run new tests for the new features separately.

**Q19**: What is HATEOAS and how do you test it?
**A**: HATEOAS (Hypermedia as the Engine of Application State) means API responses include links to related resources. Test by asserting that the response contains a `links` array with expected `rel` and `href` values.

**Q20**: How do you test a rate-limited endpoint without waiting for the rate limit window?
**A**: (a) Use a test-specific rate limit configuration with a very low limit and short window, (b) or mock the rate limiter store (e.g., Redis) and inject a state where the limit is exceeded.

## 11. Advanced Interview Questions (20: 10 Hard + 10 System Design)

### Hard

**Q1**: How do you test an API that uses conditional GET (ETag/If-None-Match)?
**A**: First, send a GET and capture the `ETag` header. Then send the same GET with `If-None-Match: <etag>`. Assert 304 with empty body. Then modify the resource. Send GET with the old ETag. Assert 200 with the updated resource and a new ETag.

**Q2**: How do you test an API that implements server-sent events (SSE)?
**A**: Use a streaming HTTP client. Connect to the SSE endpoint, read the event stream line by line, and assert on `event:` and `data:` fields. Test: (a) initial connection receives current state, (b) subsequent events are pushed, (c) client reconnection works with `Last-Event-ID`.

**Q3**: How do you test a GraphQL API for N+1 query performance?
**A**: Write a test that sends a query that would trigger N+1 (e.g., fetch a list of posts with authors). Assert that the total number of SQL queries is bounded (e.g., by enabling query logging and counting). Use `knex` or TypeORM query logging to capture and assert on query count.

**Q4**: How do you test an API that uses cursor-based pagination?
**A**: (a) Create more items than the page size. (b) Send a request without cursor and capture the `nextCursor` from the response. (c) Send a request with that cursor and assert that no items from the previous page are returned. (d) Verify the cursor is opaque (base64-encoded or hashed). (e) Test that an invalid cursor returns 400.

**Q5**: How do you test an API that implements bulk operations with transactional semantics?
**A**: (a) Send a bulk request where all operations succeed. Assert all succeed. (b) Send a bulk request where one operation fails. Assert that the entire request rolls back (no partial writes). (c) Verify the rollback by reading the state after the failed request.

**Q6**: How do you test an API that uses request collapsing (coalescing) for cache?
**A**: Send multiple concurrent identical requests. Assert that the upstream service is called only once (verify via a counter or spy on the downstream). Assert that all concurrent requests receive the same response.

**Q7**: How do you test an API that uses signed URLs (e.g., S3 presigned URLs)?
**A**: (a) Request a signed URL from the API. (b) Use the signed URL to upload or download directly from S3. (c) Assert the operation succeeds. (d) Modify the URL slightly and assert it fails with 403. (e) Wait for the URL to expire and assert it fails.

**Q8**: How do you test an API with multiple region deployment and geo-routing?
**A**: Use a test framework that can set the `X-Forwarded-For` header or source IP. Assert that requests from different regions are routed to the correct region. Verify that data is consistent across regions (eventually) and that cross-region latency is acceptable.

**Q9**: How do you test an API that uses server-side encryption with customer-provided keys (SSE-C)?
**A**: (a) Upload a resource providing the encryption key in a header. (b) Download the resource with the same key. Assert success. (c) Download the resource with a wrong key. Assert 403. (d) Download the resource without a key. Assert 400. (e) Verify that the raw storage backend shows encrypted data.

**Q10**: How do you test API schema evolution with protobuf or Avro?
**A**: (a) Create a producer that sends messages with an older schema version. (b) Create a consumer that expects the new schema with a default value for a new field. Assert that deserialization succeeds with the default. (c) Create a consumer that expects the new schema with no default for a new field. Assert that deserialization fails or throws.

### System Design

**Q11**: Design an API testing framework for a system with 50+ microservices.
**A**: Use a layered approach: (1) Contract tests per service using OpenAPI + Dredd or Stoplight. (2) Consumer-driven contract tests with Pact. (3) Integration tests using Docker Compose with the service under test and its immediate dependencies. (4) End-to-end smoke tests that deploy all services and run critical user journeys. (5) A CI pipeline that parallelizes tests by service, using test containers for databases. Use a centralized test data factory library shared across services. Each service has a `test-manifest.json` that declares its dependencies, test data requirements, and contract location.

**Q12**: Design an API test strategy for a real-time bidding system with sub-100ms latency requirements.
**A**: (a) Functional tests: verify bid logic, budget checks, frequency capping. (b) Performance tests: run with k6 or vegeta at 10x expected QPS, assert p99 < 100ms. (c) Chaos tests: randomly fail downstream services (budget service, user profile service) and verify fallback behavior. (d) Integration tests: use WireMock for all downstream services with realistic latency (add 5-20ms delay). (e) Use a deterministic seed for the bidding algorithm so tests are reproducible. (f) Test concurrent bid requests for the same impression to verify dedup.

**Q13**: Design a testing approach for an API gateway that handles 100+ routes, rate limiting, auth, and request transformation.
**A**: (a) Route testing: generate a route table from the gateway config and automatically generate a test for each route (200 for success, 401 without auth, 404 for non-existent). (b) Rate limiting: test per-route limits, global limits, burst behavior. (c) Auth: test each auth method (JWT, API key, OAuth2). (d) Transformation: test header injection, body rewriting, query param mapping. (e) Use property-based testing: generate random valid requests and verify the gateway passes them through correctly. (f) Load test with realistic traffic patterns.

**Q14**: How would you design API tests for a multi-tenant SaaS platform where each tenant can have custom extensions/plugins?
**A**: (a) Core API tests: run against a standard tenant configuration. (b) Extension tests: each extension provides its own test suite that is loaded dynamically. (c) Tenant isolation tests: create two tenants with overlapping data, assert that tenant A cannot access tenant B's data. (d) Configuration tests: change a tenant's config (e.g., feature flag, custom field) and assert the API behavior changes accordingly. (e) Use a test matrix where each tenant configuration is a dimension.

**Q15**: Design a testing strategy for a financial API that must handle idempotency, exactly-once processing, and strict ordering.
**A**: (a) Idempotency tests: send the same idempotency key twice, assert the second response is identical to the first and no duplicate side effects. (b) Ordering tests: send requests with out-of-order timestamps and assert the server rejects or reorders them. (c) Exactly-once: simulate a network failure after the server processes the request but before the client receives the response. Re-send with the same idempotency key. Assert no double processing. (d) Concurrency: send concurrent requests with overlapping state and assert consistent behavior (optimistic locking, version conflicts). (e) Audit trail: assert that every state change is recorded in the audit log with the correct sequence number.

**Q16**: Design an API testing framework for a streaming data platform (Kafka Connect + Schema Registry + REST proxy).
**A**: (a) Schema evolution tests: register a new schema version, produce messages with old and new schemas, consume and validate both can be deserialized. (b) Connector tests: create a source connector, produce data to the source system, assert data appears in Kafka with the correct schema. (c) REST proxy tests: produce and consume messages through the REST proxy, verify binary and Avro formats. (d) Failure tests: stop a connector, restart it, verify it resumes from the correct offset. (e) Performance tests: produce at max expected throughput and verify the REST proxy does not drop or reorder messages.

**Q17**: Design tests for a multi-step approval API (e.g., expense report that goes through submit -> approve -> pay).
**A**: (a) Happy path: submit -> manager approve -> finance approve -> pay. Assert status transitions. (b) Rejection: submit -> manager reject. Assert status is "rejected" and no further actions allowed. (c) Escalation: if approval is pending beyond a timeout, assert the request escalates to the next level. (d) Parallel approval: some steps require multiple approvers. Assert that the workflow waits for all. (e) Recall: after submission but before approval, the submitter can recall. Assert the status changes and no further approvals are processed.

**Q18**: How would you test an API that uses a CQRS/Event Sourcing architecture?
**A**: (a) Command tests: send write commands, assert they produce the correct events in the event store. (b) Query tests: build read models, query them, assert the state matches the expected projection. (c) Consistency tests: send a command, then query. Depending on the consistency model (eventual vs. strong), assert either immediate or eventual consistency using polling. (d) Replay tests: delete the read model, replay all events, assert the read model is rebuilt correctly. (e) Snapshot tests: verify that snapshots are created at the expected intervals and that replay from a snapshot produces the correct state.

**Q19**: Design an API test suite for a healthcare API that must comply with HIPAA.
**A**: (a) Auth tests: verify MFA is enforced for certain endpoints. (b) Audit tests: every PHI access must be logged. Assert audit log entries contain user ID, timestamp, action, resource ID. (c) Data minimization: assert that API responses never include sensitive fields (SSN, full DOB) unless explicitly requested with elevated privileges. (d) Encryption: assert that all responses use HTTPS (test by attempting HTTP and verifying redirect or rejection). (e) Breach notification: test that access patterns that match breach criteria (e.g., bulk download of patient records) trigger an alert. (f) Consent: test that a patient's data is not accessible after they revoke consent.

**Q20**: Design a testing strategy for a serverless API (API Gateway + Lambda + DynamoDB) with 1000+ endpoints.
**A**: (a) Use AWS SAM or SST to deploy a test stack with ephemeral resources. (b) Generate test cases from the OpenAPI spec: for each path + method, generate a test with valid input, invalid input, and auth failure. (c) Use DynamoDB single-table design tests: verify that queries use the correct index, that GSIs are populated, and that no full table scans occur. (d) Lambda cold start tests: measure response time for concurrent first requests. (e) Test API Gateway throttling, WAF rules, and request validation. (f) Use CloudWatch Logs integration to assert on log output (no secrets in logs). (g) Run tests in parallel using separate API Gateway stages and DynamoDB tables per test worker.

## 12. Expert-Level Interview Questions (10: Architect-Level)

**Q1**: Design an API testing maturity model for an organization with 200+ engineers.
**A**: Level 1 (Ad-hoc): Manual testing via Postman, no automation. Level 2 (Basic): Automated smoke tests for critical paths, run on every commit. Level 3 (Contract): OpenAPI specs are source of truth, contract tests enforce spec compliance. Level 4 (Consumer-Driven): Each team publishes consumer tests; breaking changes are caught before deployment. Level 5 (Continuous): API tests are part of the deployment pipeline; canary deployments run API tests against the new version before full rollout. Level 6 (Self-Healing): Tests automatically update when the spec changes (within backward-compatible limits); flaky tests auto-quarantine. Measurement: time to detect API regression (Level 1: hours, Level 6: seconds).

**Q2**: How would you design an API testing strategy for a system that must maintain backward compatibility for 5 years?
**A**: (a) Versioning strategy: URL versioning (`/v1/`, `/v2/`) with documented sunset policy. (b) Test suite per version: each version has its own contract tests that never change. (c) Sunset tests: when a version is deprecated, tests assert that a `Sunset` header and `Deprecation` header are included. (d) Breaking change detection: run a diff between the new OpenAPI spec and the old one; any breaking change (removed field, tightened constraint) fails CI. (e) Migration tests: write tests that verify clients can migrate from v1 to v2 without data loss. (f) Long-running tests: a nightly CI job runs the oldest supported version's test suite against the current codebase.

**Q3**: You are consulting for a company whose API test suite takes 6 hours to run. How do you reduce it to under 30 minutes?
**A**: (a) Analyze test distribution: most tests are likely integration or E2E tests hitting real databases. (b) Parallelize: split tests by service/module and run in parallel CI containers. (c) Optimize database setup: use in-memory databases where possible, transaction rollback instead of truncation. (d) Remove redundancy: if a full CRUD test exists for each endpoint, consolidate into a parameterized test that runs the same logic for all endpoints. (e) Categorize: run fast unit+integration tests on every commit; run slow E2E tests nightly. (f) Use test impact analysis: only run tests that cover changed code. (g) Increase hardware: more CI runners, bigger instances. (h) Profile: find the slowest 10% of tests and optimize them (add indexes, reduce test data size). (i) Target: 6 hours -> 30 minutes is an 12x improvement; achievable through parallelization (8x) and test optimization (1.5x).

**Q4**: Design a chaos engineering approach for API testing. How do you verify that APIs degrade gracefully?
**A**: (a) Inject failures: use a service mesh (e.g., Istio) or a proxy (e.g., Toxiproxy) to inject latency, errors, and network partitions. (b) Test scenarios: (1) database connection pool exhaustion, (2) downstream service returns 500, (3) downstream service is unreachable (TCP timeout), (4) TLS certificate expires, (5) disk I/O slowdown, (6) memory pressure triggers GC thrash. (c) Assertions: the API returns a 5xx error (not a crash/hang), the error response has a standard format, the request is logged with the failure reason, the circuit breaker opens, the fallback path is used (e.g., cached data, degraded response). (d) Steady-state verification: run a baseline API test suite under normal conditions. Then run the same suite under each fault condition. Compare results.

**Q5**: How do you ensure API tests are not flaky in a microservices environment with async communication?
**A**: (a) Use deterministic testing: replace message queues with in-memory buses in test mode. (b) Use polling with timeout: `wait-for-expect` for async side effects. (c) Use test transactions: if the test involves multiple services, use a distributed transaction ID that allows tracking the flow end-to-end. (d) Implement retry logic in the test framework: automatically retry failed tests up to 3 times with exponential backoff. (e) Quarantine flaky tests: CI automatically moves flaky tests to a separate suite and notifies the team. (f) Use log-based assertions: instead of asserting on external state, assert on structured logs that capture the async flow. (g) Version-lock external dependencies: use Docker images with specific tags, not `latest`.

**Q6**: How would you design an API test suite that validates both REST and GraphQL endpoints from the same service?
**A**: (a) Define the domain model once in TypeScript types. (b) Create a test data factory that generates entities. (c) For each test scenario, write two tests: one REST (supertest endpoint call) and one GraphQL (supertest POST to `/graphql`). (d) Assert the same shape of data. (e) Use a test helper that takes a GraphQL query and a REST route and compares the responses for equivalent requests. (f) Test consistency: make a change via REST, then query via GraphQL, and verify the change is reflected. (g) Run the shared data factory tests against both interfaces to ensure they remain in sync.

**Q7**: You are building an API that must serve both mobile and web clients. Mobile clients are sensitive to payload size. How do you test this?
**A**: (a) Add a test that measures the response payload size for each endpoint and asserts it does not exceed a threshold (e.g., 50KB for list endpoints, 10KB for detail). (b) Test sparse field sets: if the API supports `?fields=id,name`, assert the response excludes unrequested fields and the payload is smaller. (c) Test pagination defaults: assert the default page size is appropriate for mobile (e.g., 20 items). (d) Test compression: assert the server supports gzip/brotli and measure the compressed size. (e) Test ETags: assert the mobile client can use conditional requests to avoid re-downloading unchanged data. (f) Simulate slow networks: use a network throttle (e.g., `--net-throttle` in Chrome or a proxy) and assert the API is usable at 3G speeds.

**Q8**: Design a test strategy for an API that uses feature flags to gradually roll out new functionality.
**A**: (a) For each feature flag, write tests with the flag on and off. (b) In CI, run the full test suite with all flags off (baseline) and with the new flag on. (c) Use a test matrix: each flag combination is a dimension. For N flags, run 2^N combinations, but use pairwise testing to reduce combinations. (d) Test gradual rollout: enable the flag for 10% of requests (using a deterministic user ID hash). Assert that the user experience is correct based on their hash value. (e) Test kill switch: enable the flag, then disable it, and verify the system falls back to the old behavior. (f) Test flag persistence: enable a flag, make a request that uses the flag, disable the flag, make another request, and assert the system is in the expected state.

**Q9**: How do you test API compliance with RFC 723x (HTTP/1.1) specifications?
**A**: (a) Write a compliance test suite that covers: (1) status codes (all standard codes), (2) headers (Cache-Control, ETag, Content-Length, Transfer-Encoding), (3) methods (GET, HEAD, POST, PUT, DELETE, OPTIONS, PATCH), (4) conditional requests (If-Match, If-None-Match, If-Modified-Since, If-Unmodified-Since), (5) content negotiation (Accept, Content-Type), (6) connection management (keep-alive, close). (b) Use a dedicated HTTP compliance testing library (e.g., `http-suite`). (c) Test edge cases: (1) request without Host header (must return 400), (2) request with malformed header (must return 400), (3) HEAD request (must return same headers as GET but no body), (4) OPTIONS request (must return Allow header). (d) Run the compliance suite as part of CI and track compliance percentage over time.

**Q10**: An API you own is failing in production with sporadic 504 Gateway Timeout errors. How do you write a test to reproduce and fix this?
**A**: (a) Check the ALB/nginx logs to identify the endpoint and time of timeout. (b) Analyze the slow path: it could be a database query, an external API call, or a computation. (c) Write a test that reproduces the slow condition: (1) if it's a slow DB query, create enough data in the test DB to trigger a full table scan, (2) if it's an external API, use WireMock with a delay of 30s, (3) if it's a computation, generate worst-case input data. (d) Set the test timeout to match the ALB idle timeout (e.g., 60s). (e) Assert that the API returns a 502 or 503 (gateway timeout) within the timeout, or fix the issue and assert a 200 under the slow condition. (f) Add a performance regression test that alerts if the endpoint's p95 response time exceeds the ALB timeout.

## 13. Debugging & Troubleshooting

### 13.1 Common Errors and Fixes

**Error: ECONNREFUSED**
- Cause: Server not running or wrong port.
- Fix: Ensure the app is listening. In supertest, pass the Express app directly.

**Error: socket hang up**
- Cause: Server closed connection unexpectedly, often due to a crash or timeout.
- Fix: Check server error logs. Increase timeout. Remove `app.listen()` call when using supertest.

**Error: Parse Error: Expected HTTP/**
- Cause: Server responded with non-HTTP (e.g., HTML error page instead of JSON).
- Fix: Assert on response status code first, then inspect `res.text` to see the raw response.

**Error: Timeout of 5000ms exceeded**
- Cause: Test takes longer than default timeout.
- Fix: Increase timeout in Jest config or use `jest.setTimeout()`.

**Error: Nock: Disallowed net connect**
- Cause: Nock is blocking real HTTP connections that your app needs.
- Fix: Use `nock.enableNetConnect()` to allow connections to specific hosts.

### 13.2 Debugging Tips

```typescript
// Log the full request and response
const res = await request(app)
  .post('/api/users')
  .send({ name: 'test' });

console.log('Status:', res.status);
console.log('Headers:', JSON.stringify(res.headers, null, 2));
console.log('Body:', JSON.stringify(res.body, null, 2));
```

### 13.3 Using Postman Console for Debugging

```javascript
// Postman test script debugging
console.log('Response status:', pm.response.code);
console.log('Response body:', JSON.stringify(pm.response.json(), null, 2));
console.log('Environment variable:', pm.environment.get('baseUrl'));

pm.test('Debug', () => {
  console.log('Running debug assertion');
  pm.expect(pm.response.code).to.be.oneOf([200, 201]);
});
```

### 13.4 Network Inspection

Use `curl` or `httpie` to manually test the same request:

```bash
curl -v -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name": "test", "email": "test@test.com"}'
```

### 13.5 Checking Database State

```typescript
const user = await db('users').where({ email: 'test@test.com' }).first();
console.log('User after API call:', user);
```

## 14. Comparison Section

### 14.1 supertest vs. Postman vs. curl

| Aspect | supertest | Postman | curl |
|--------|-----------|---------|------|
| Type | Library (npm) | GUI + CLI | CLI tool |
| CI integration | Native (Jest/Vitest) | Newman | Shell scripts |
| Assertions | Jest/Vitest expect | pm.test/javascript | Manual |
| Version control | Yes (code) | JSON collections | Commands |
| Speed | Fast (in-process) | Slow (network) | Medium |
| Learning curve | Medium | Low | Low |
| Collaboration | Code review | Postman workspaces | Scripts |

### 14.2 REST vs. GraphQL API Testing

| Aspect | REST | GraphQL |
|--------|------|---------|
| Endpoints | Multiple (one per resource) | Single (/graphql) |
| Testing focus | Status codes, headers, body | Query validation, resolver errors |
| Over-fetching | Common | Client-controlled |
| Tooling | supertest, Postman | Apollo client, graphql-request |
| Schema validation | OpenAPI/Swagger | GraphQL schema |
| Versioning | URL or header-based | Schema evolution (deprecation) |

### 14.3 SOAP vs. REST Testing

| Aspect | SOAP | REST |
|--------|------|------|
| Protocol | XML over HTTP/SMTP/etc. | JSON/XML over HTTP |
| Testing tool | SoapUI, Postman | supertest, Postman |
| State | Stateful possible | Stateless |
| Security | WS-Security | HTTPS, JWT, OAuth2 |
| Contract | WSDL | OpenAPI/Swagger |

### 14.4 Synchronous vs. Asynchronous API Testing

| Aspect | Synchronous | Asynchronous |
|--------|------------|--------------|
| Response | Immediate | Delayed (callback, poll, webhook) |
| Test pattern | Request-assert | Request-poll-assert or request-listen-assert |
| Timeout | Short (5s) | Long (30s+) |
| Flakiness | Low | Higher |
| Examples | REST, GraphQL | WebSocket, SSE, message queues |

## 15. Revision Notes

### Key Concepts

- API testing verifies the contract between client and server.
- Always test happy path AND error paths (4xx, 5xx).
- Test security: auth, authorization, injection, rate limiting.
- Use supertest for in-process HTTP testing (no port needed).
- Use Postman for manual/exploratory testing and Newman for CI.
- OpenAPI specs should be the single source of truth.
- Payload size, response time, and idempotency are critical.
- Test data isolation prevents flaky tests.
- Consumer-driven contracts catch breaking changes early.
- Version your API and test each version independently.

### Testing Checklist

- [ ] All HTTP methods (GET, POST, PUT, PATCH, DELETE) tested
- [ ] All status codes (2xx, 4xx, 5xx) asserted
- [ ] Authentication required (401 without credentials)
- [ ] Authorization enforced (403 for wrong role)
- [ ] Input validation (422 for invalid input)
- [ ] Rate limiting applied (429 when exceeded)
- [ ] Content type correct
- [ ] Response headers present (ETag, Cache-Control, security headers)
- [ ] Pagination works (limit, offset, cursor)
- [ ] Idempotency for PUT/DELETE
- [ ] File upload (size limit, type validation)
- [ ] CORS headers present
- [ ] Payload size within limits
- [ ] Response time within SLA
- [ ] No sensitive data leakage (passwords, tokens, PII)

## 16. Cheat Sheet

```text
+==============================================================================+
|                         API TESTING CHEAT SHEET                              |
+==============================================================================+

+--- HTTP METHODS & STATUS CODES ---------------------------------------------+
|                                                                              |
|  GET    /users       200 OK              returns list                        |
|  GET    /users/:id   200 OK              404 Not Found                       |
|  POST   /users       201 Created         409 Conflict (duplicate)            |
|  PUT    /users/:id   200 OK              404 Not Found                       |
|  PATCH  /users/:id   200 OK              422 Unprocessable                   |
|  DELETE /users/:id   204 No Content      404 Not Found                       |
|                                                                              |
+--- SUPERTEST PATTERNS ------------------------------------------------------+
|                                                                              |
|  import request from 'supertest';                                           |
|  import { app } from '../app';                                              |
|                                                                              |
|  // Basic GET                                                               |
|  const res = await request(app)                                             |
|    .get('/api/users/1')                                                     |
|    .set('Authorization', 'Bearer ' + token)                                 |
|    .expect(200)                                                             |
|    .expect('Content-Type', /json/);                                         |
|                                                                              |
|  // POST with body                                                          |
|  await request(app)                                                         |
|    .post('/api/users')                                                      |
|    .send({ name: 'Alice', email: 'a@b.com' })                               |
|    .expect(201)                                                             |
|    .expect((res) => {                                                       |
|      expect(res.body.id).toBeDefined();                                     |
|    });                                                                      |
|                                                                              |
|  // Query params                                                            |
|  await request(app)                                                         |
|    .get('/api/users')                                                       |
|    .query({ page: 1, limit: 20, sort: 'name:asc' })                        |
|    .expect(200);                                                            |
|                                                                              |
|  // File upload                                                             |
|  await request(app)                                                         |
|    .post('/api/upload')                                                     |
|    .field('description', 'My file')                                         |
|    .attach('file', '/path/to/file.pdf')                                     |
|    .expect(201);                                                            |
|                                                                              |
+--- POSTMAN TEST SCRIPTS ----------------------------------------------------+
|                                                                              |
|  // Status code check                                                       |
|  pm.test('Status is 200', () => pm.response.to.have.status(200));           |
|                                                                              |
|  // Body check                                                              |
|  pm.test('Has id', () => {                                                  |
|    pm.expect(pm.response.json()).to.have.property('id');                    |
|  });                                                                        |
|                                                                              |
|  // Header check                                                            |
|  pm.test('Content-Type is JSON', () => {                                    |
|    pm.expect(pm.response.headers.get('Content-Type'))                       |
|      .to.include('application/json');                                       |
|  });                                                                        |
|                                                                              |
|  // Response time                                                           |
|  pm.test('Response time < 500ms', () => {                                   |
|    pm.expect(pm.response.responseTime).to.be.below(500);                    |
|  });                                                                        |
|                                                                              |
|  // Set env variable from response                                          |
|  pm.environment.set('userId', pm.response.json().id);                       |
|                                                                              |
+--- COMMON ASSERTIONS -------------------------------------------------------+
|                                                                              |
|  expect(res.body).toEqual(expected)        // deep equality                  |
|  expect(res.body).toMatchObject(partial)   // subset match                   |
|  expect(res.body).toHaveProperty('key')    // key exists                     |
|  expect(res.headers['etag']).toBeDefined() // header exists                  |
|  expect(res.status).toBe(200)              // status code                    |
|  expect(res.body.data.length).toBe(10)     // array length                   |
|                                                                              |
+--- SECURITY TESTS ----------------------------------------------------------+
|                                                                              |
|  // No auth -> 401                                                          |
|  request(app).get('/api/users').expect(401)                                 |
|                                                                              |
|  // Wrong role -> 403                                                        |
|  request(app).delete('/api/users/1').set('Authorization', token).expect(403)|
|                                                                              |
|  // SQL injection -> 400                                                     |
|  request(app).get('/api/users').query({id:"1; DROP TABLE users--"}).expect(400)|
|                                                                              |
|  // XSS -> 422                                                              |
|  request(app).post('/api/users').send({name:"<script>alert(1)</script>"})   |
|    .expect(422)                                                             |
|                                                                              |
+--- CURL REFERENCE ----------------------------------------------------------+
|                                                                              |
|  curl -X GET http://localhost:3000/api/users                                |
|  curl -X POST -H "Content-Type: application/json"                           |
|       -d '{"name":"Alice"}' http://localhost:3000/api/users                 |
|  curl -X PUT -H "Authorization: Bearer $TOKEN"                               |
|       http://localhost:3000/api/users/1                                     |
|  curl -X DELETE http://localhost:3000/api/users/1                           |
|  curl -v -X OPTIONS http://localhost:3000/api/users                         |
|                                                                              |
+==============================================================================+
```
