# Integration Testing

## 1. Executive Summary

Integration testing validates that multiple software modules or services work together as expected. Unlike unit tests, which isolate individual components, integration tests exercise real interactions between layers such as databases, APIs, file systems, and external services. The primary goal is to catch interface defects, data format mismatches, and contract violations that unit tests cannot detect. In a typical Node.js/TypeScript backend, integration tests spin up a test database, start an HTTP server (or use supertest against the Express/Koa/Fastify app), and exercise entire request-response cycles. The return on investment is high: integration tests reduce regression risk, document system behavior, and give the team confidence to refactor. They sit between unit and end-to-end tests in the test pyramid.

## 2. Core Theory

### 2.1 The Test Pyramid

Integration tests occupy the middle layer of the test pyramid:

```
     /\
    /  \
   /E2E \
  /------\
 / Integ  \
/----------\
/ Unit      \
/------------\
```

- **Unit tests**: Fast, isolated, mock-heavy. Cover individual functions.
- **Integration tests**: Slower, use real dependencies (DB, FS, network). Cover module interactions.
- **E2E tests**: Slowest, simulate full user workflows in a production-like environment.

### 2.2 Key Concepts

- **Test Database**: A dedicated database instance (or in-memory SQLite, or PostgreSQL test container) that is seeded before tests and cleaned after.
- **Test Fixtures**: Known data sets loaded into the test database before each test run.
- **Side Effects**: Integration tests typically write to real storage, so tests must be idempotent and isolated (e.g., wrapping each test in a transaction and rolling back).
- **Contract Testing**: A subtype of integration testing that verifies an API provider meets its consumer's expectations (e.g., Pact or OpenAPI-based).

### 2.3 What Integration Tests Cover

- HTTP route handlers end-to-end (request parsing, validation, business logic, DB query, serialization)
- Middleware stacks (auth, logging, rate limiting, error handling)
- Database queries (ORM mappings, raw SQL, migrations, constraints)
- Message queues (publishing and consuming)
- External API client wrappers (using test doubles or WireMock)

### 2.4 What Integration Tests Do NOT Cover

- Pure unit logic (delegate to unit tests)
- UI rendering (delegate to E2E)
- Third-party service correctness (use test doubles or sandbox environments)

## 3. Under-the-Hood Deep Dive

### 3.1 How Supertest Works

Supertest extends SuperAgent (an HTTP client). When you call `request(app)`, supertest does NOT bind to a real port by default. Instead, it passes the Node.js `http.Server` instance an ephemeral port via `server.listen()` internally, or it uses the app directly if the app is a function `(req, res) => void`. This avoids port conflicts and speeds up execution.

```typescript
// supertest internals (simplified)
function request(app: Express.Application): SuperTest {
  const server = app.listen(0, () => {
    // port 0 = OS assigns a random available port
  });
  const agent = superagent.agent();
  // agent.http() uses the server address
  return new Test(agent, server);
}
```

### 3.2 Transaction Rollback Strategy

To keep tests isolated without manual cleanup, each test runs inside a database transaction that is rolled back at the end:

```typescript
import { Sequelize } from 'sequelize-typescript';

let sequelize: Sequelize;

beforeEach(async () => {
  await sequelize.query('BEGIN');
});

afterEach(async () => {
  await sequelize.query('ROLLBACK');
});
```

This works with PostgreSQL and MySQL. SQLite supports savepoints. For MongoDB, use test containers or drop collections between tests.

### 3.3 Test Container Pattern

For services like PostgreSQL, Redis, or Kafka, test containers spin up lightweight Docker containers:

```typescript
import { PostgreSqlContainer } from '@testcontainers/postgresql';

let container: PostgreSqlContainer;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('testdb')
    .start();
  process.env.DATABASE_URL = container.getConnectionUri();
});
```

### 3.4 How Express Processes a Request

```
 HTTP Request
     |
     v
express.json()       -- parse body
     |
     v
authentication       -- verify JWT/session
     |
     v
authorization        -- check roles
     |
     v
validation           -- joi/zod schema
     |
     v
route handler        -- business logic
     |
     v
database query       -- actual I/O
     |
     v
serialization        -- response formatting
     |
     v
 HTTP Response
```

Integration tests cover the entire pipeline except the transport layer (TCP/TLS).

## 4. Production Code Examples

### 4.1 Basic Express + Supertest Setup

```typescript
// src/app.ts
import express from 'express';
import { router as userRouter } from './routes/users';

const app = express();
app.use(express.json());
app.use('/api/users', userRouter);

export { app };
```

```typescript
// src/server.ts
import { app } from './app';

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Listening on ${PORT}`));
```

```typescript
// src/__tests__/users.integration.test.ts
import request from 'supertest';
import { app } from '../app';
import { db } from '../db';
import { seedUsers } from './seed';

beforeAll(async () => {
  await db.migrate.latest();
  await seedUsers();
});

afterAll(async () => {
  await db.destroy();
});

describe('GET /api/users/:id', () => {
  it('returns a user by ID', async () => {
    const res = await request(app)
      .get('/api/users/42')
      .expect('Content-Type', /json/)
      .expect(200);

    expect(res.body).toMatchObject({
      id: 42,
      name: expect.any(String),
    });
  });

  it('returns 404 for non-existent user', async () => {
    const res = await request(app)
      .get('/api/users/9999')
      .expect(404);

    expect(res.body).toEqual({ error: 'User not found' });
  });
});
```

### 4.2 Testing with Authentication

```typescript
// src/__tests__/auth.integration.test.ts
import request from 'supertest';
import { app } from '../app';
import { createAuthToken } from '../auth';

let token: string;

beforeAll(() => {
  token = createAuthToken({ userId: 1, role: 'admin' });
});

describe('POST /api/articles', () => {
  it('creates an article when authenticated', async () => {
    const res = await request(app)
      .post('/api/articles')
      .set('Authorization', `Bearer ${token}`)
      .send({ title: 'Test', body: 'Content' })
      .expect(201);

    expect(res.body.title).toBe('Test');
  });

  it('returns 401 without token', async () => {
    await request(app)
      .post('/api/articles')
      .send({ title: 'Test', body: 'Content' })
      .expect(401);
  });

  it('returns 403 for insufficient role', async () => {
    const userToken = createAuthToken({ userId: 2, role: 'user' });
    await request(app)
      .post('/api/articles')
      .set('Authorization', `Bearer ${userToken}`)
      .send({ title: 'Test', body: 'Content' })
      .expect(403);
  });
});
```

### 4.3 Testing Database Interactions

```typescript
// src/__tests__/db.integration.test.ts
import { db } from '../db';

describe('User repository', () => {
  beforeEach(async () => {
    await db('users').truncate();
  });

  it('creates a user and returns the record', async () => {
    const [id] = await db('users').insert({
      name: 'Alice',
      email: 'alice@example.com',
    });

    const user = await db('users').where({ id }).first();
    expect(user.name).toBe('Alice');
    expect(user.email).toBe('alice@example.com');
  });

  it('enforces unique email constraint', async () => {
    await db('users').insert({ name: 'A', email: 'dup@example.com' });
    await expect(
      db('users').insert({ name: 'B', email: 'dup@example.com' })
    ).rejects.toThrow();
  });
});
```

### 4.4 Testing a Full REST End-to-End Flow

```typescript
// src/__tests__/articles.flow.integration.test.ts
import request from 'supertest';
import { app } from '../app';

let token: string;
let articleId: number;

describe('Article lifecycle', () => {
  beforeAll(async () => {
    const res = await request(app)
      .post('/api/auth/login')
      .send({ username: 'admin', password: 'secret' });
    token = res.body.token;
  });

  it('creates an article', async () => {
    const res = await request(app)
      .post('/api/articles')
      .set('Authorization', `Bearer ${token}`)
      .send({ title: 'Integration Testing', body: 'Deep dive' });
    articleId = res.body.id;
    expect(res.status).toBe(201);
  });

  it('retrieves the article', async () => {
    const res = await request(app)
      .get(`/api/articles/${articleId}`)
      .expect(200);
    expect(res.body.title).toBe('Integration Testing');
  });

  it('updates the article', async () => {
    await request(app)
      .put(`/api/articles/${articleId}`)
      .set('Authorization', `Bearer ${token}`)
      .send({ title: 'Updated Title' })
      .expect(200);
  });

  it('deletes the article', async () => {
    await request(app)
      .delete(`/api/articles/${articleId}`)
      .set('Authorization', `Bearer ${token}`)
      .expect(204);

    await request(app)
      .get(`/api/articles/${articleId}`)
      .expect(404);
  });
});
```

### 4.5 Testing External API Calls with WireMock

```typescript
// src/__tests__/payment.integration.test.ts
import { WireMock } from 'wiremock-captain';
import request from 'supertest';
import { app } from '../app';

const wiremock = new WireMock('http://localhost:8089');

beforeAll(async () => {
  await wiremock.registerStub({
    request: {
      method: 'POST',
      urlPath: '/charge',
    },
    response: {
      status: 200,
      body: { id: 'ch_123', status: 'succeeded' },
    },
  });
});

afterAll(async () => {
  await wiremock.reset();
});

it('processes payment successfully', async () => {
  const res = await request(app)
    .post('/api/payments')
    .send({ amount: 5000, currency: 'usd' })
    .expect(201);

  expect(res.body.status).toBe('succeeded');
});
```

### 4.6 Testing GraphQL Integration

```typescript
// src/__tests__/graphql.integration.test.ts
import request from 'supertest';
import { app } from '../app';

describe('GraphQL API', () => {
  it('fetches a user by ID', async () => {
    const query = `
      query GetUser($id: ID!) {
        user(id: $id) {
          name
          email
        }
      }
    `;

    const res = await request(app)
      .post('/graphql')
      .send({ query, variables: { id: '1' } })
      .expect(200);

    expect(res.body.data.user.name).toBeDefined();
  });

  it('rejects invalid queries', async () => {
    const res = await request(app)
      .post('/graphql')
      .send({ query: '{ invalidField }' })
      .expect(400);
  });
});
```

### 4.7 Testing with TypeORM

```typescript
// src/__tests__/typeorm.integration.test.ts
import { createConnection, getConnection } from 'typeorm';
import request from 'supertest';
import { app } from '../app';
import { User } from '../entities/User';

beforeAll(async () => {
  await createConnection({
    type: 'sqlite',
    database: ':memory:',
    entities: [User],
    synchronize: true,
  });
});

afterAll(async () => {
  await getConnection().close();
});

it('persists and retrieves a user', async () => {
  const res = await request(app)
    .post('/api/users')
    .send({ name: 'Test', email: 'test@test.com' })
    .expect(201);

  const repo = getConnection().getRepository(User);
  const user = await repo.findOne(res.body.id);
  expect(user.email).toBe('test@test.com');
});
```

## 5. Real-World Scenarios

### 5.1 Microservices Integration Testing

In a microservices architecture, integration tests verify that service A correctly calls service B. Use test doubles (WireMock, MockServer) to simulate downstream services:

```typescript
// service-a/__tests__/order.integration.test.ts
import { MockServer } from 'mockserver-client';

const mockServer = new MockServer('localhost', 1080);

beforeAll(async () => {
  await mockServer.mockAnyResponse({
    httpRequest: { method: 'GET', path: '/inventory/check' },
    httpResponse: {
      statusCode: 200,
      body: JSON.stringify({ available: true }),
    },
  });
});
```

### 5.2 Event-Driven Integration (Kafka/RabbitMQ)

```typescript
// __tests__/event.integration.test.ts
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ brokers: ['localhost:9093'] });
const producer = kafka.producer();
const consumer = kafka.consumer({ groupId: 'test-group' });

beforeAll(async () => {
  await producer.connect();
  await consumer.connect();
  await consumer.subscribe({ topic: 'order-placed' });
});

it('processes order event', (done) => {
  consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      expect(event.type).toBe('order.placed');
      done();
    },
  });

  producer.send({
    topic: 'order-placed',
    messages: [{ value: JSON.stringify({ orderId: 1, amount: 100 }) }],
  });
});
```

### 5.3 Database Migration Testing

```typescript
// __tests__/migration.integration.test.ts
import { db } from '../db';

it('applies migrations correctly', async () => {
  const [hasTable] = await db.raw(
    `SELECT EXISTS (SELECT FROM information_schema.tables WHERE table_name = 'users')`
  );
  expect(hasTable.exists).toBe(true);

  const columns = await db('users').columnInfo();
  expect(columns).toHaveProperty('id');
  expect(columns).toHaveProperty('email');
  expect(columns.email.type).toBe('character varying');
});
```

### 5.4 File Upload Integration

```typescript
import path from 'path';
import request from 'supertest';
import { app } from '../app';

it('uploads a file', async () => {
  const res = await request(app)
    .post('/api/upload')
    .attach('file', path.resolve(__dirname, 'fixtures/sample.pdf'))
    .expect(201);

  expect(res.body.filename).toMatch(/\.pdf$/);
});
```

## 6. Performance

### 6.1 Execution Time Benchmarks

| Test Type        | Avg Duration | Parallelism |
|------------------|-------------|-------------|
| Unit test        | 1-10 ms     | Unlimited   |
| Integration (in-memory DB) | 50-200 ms | Moderate    |
| Integration (Docker DB)    | 200-500 ms | Low         |
| E2E (full stack) | 1-10 s      | Serial      |

### 6.2 Optimization Strategies

- **Use in-memory databases** (SQLite `:memory:`, `mongodb-memory-server`) for tests that don't need production-specific SQL features.
- **Run tests in parallel** with Jest's `--maxWorkers` or Vitest's `threads`. Ensure each worker gets its own database (e.g., schema-per-worker or separate test file).
- **Pre-seed data** in `beforeAll` instead of `beforeEach` when tests share read-only fixtures.
- **Disable logging** during tests to reduce I/O.
- **Connection pooling**: Reuse database connections across tests in the same file.

```typescript
// jest.config.ts
export default {
  maxWorkers: 4,
  setupFilesAfterSetup: ['./jest.integration.setup.ts'],
  testMatch: ['**/*.integration.test.ts'],
  testTimeout: 30000,
};
```

### 6.3 Database Per Worker Pattern

```typescript
// globalSetup.ts
import { PostgreSqlContainer } from '@testcontainers/postgresql';

export default async function () {
  const container = await new PostgreSqlContainer()
    .withDatabase('test')
    .start();
  process.env.DATABASE_URL = container.getConnectionUri();
}
```

## 7. Security

### 7.1 SQL Injection Testing

Integration tests should verify that endpoints are not vulnerable to injection:

```typescript
it('prevents SQL injection', async () => {
  const res = await request(app)
    .get('/api/users')
    .query({ id: "1; DROP TABLE users; --" })
    .expect(400);

  // Verify the table still exists
  const { rows } = await db.raw('SELECT COUNT(*) FROM users');
  expect(rows[0].count).toBeDefined();
});
```

### 7.2 NoSQL Injection (MongoDB)

```typescript
it('prevents NoSQL injection', async () => {
  const res = await request(app)
    .post('/api/auth/login')
    .send({
      username: { $gt: '' },
      password: { $gt: '' },
    })
    .expect(401);
});
```

### 7.3 Auth Bypass Testing

```typescript
describe('Auth bypass attempts', () => {
  it('rejects request with tampered JWT', async () => {
    const tamperedToken = 'eyJhbGciOiJub25lIn0.eyJzdWIiOiIxIn0.'; // alg:none
    await request(app)
      .delete('/api/users/1')
      .set('Authorization', `Bearer ${tamperedToken}`)
      .expect(401);
  });

  it('rejects expired token', async () => {
    const expired = jwt.sign({ sub: 1 }, SECRET, { expiresIn: '0s' });
    await request(app)
      .get('/api/users/me')
      .set('Authorization', `Bearer ${expired}`)
      .expect(401);
  });
});
```

### 7.4 Rate Limiting

```typescript
it('enforces rate limit', async () => {
  const requests = Array.from({ length: 110 }, () =>
    request(app).get('/api/public')
  );
  const results = await Promise.all(requests);
  const tooMany = results.filter(r => r.status === 429);
  expect(tooMany.length).toBeGreaterThan(0);
});
```

## 8. Common Mistakes

### 8.1 Testing Implementation Details

Don't assert on internal calls that are not part of the contract:

```typescript
// BAD: asserting on implementation
expect(userService.sendEmail).toHaveBeenCalled();

// GOOD: assert on observable behavior
const res = await request(app).post('/api/users').send(...);
expect(res.status).toBe(201);
// Verify side effect: check user exists in DB
const user = await db('users').where({ email }).first();
expect(user).toBeDefined();
```

### 8.2 Sharing State Between Tests

```typescript
// BAD: tests depend on each other
let userId: number;
it('creates user', async () => {
  const res = await request(app).post('/api/users');
  userId = res.body.id; // leaked state
});
it('deletes user', async () => {
  await request(app).delete(`/api/users/${userId}`); // fragile
});

// GOOD: each test is self-contained
it('completes full lifecycle', async () => {
  const createRes = await request(app).post('/api/users');
  const id = createRes.body.id;
  await request(app).delete(`/api/users/${id}`).expect(204);
});
```

### 8.3 Not Cleaning Test Data

Always clean up between runs to avoid test pollution:

```typescript
afterEach(async () => {
  await db('users').truncate();
  // Or use transaction rollback
});
```

### 8.4 Using Mocks Instead of Real Instances

```typescript
// BAD: defeats purpose of integration test
jest.mock('../db');
(db.query as jest.Mock).mockResolvedValue([{ id: 1 }]);

// GOOD: use real database
const users = await db('users');
```

### 8.5 Slow Tests Without Isolation

Running all integration tests against a single shared database state leads to flaky tests. Use schema-per-test-file or transaction rollback.

### 8.6 Forgetting to Await Promises

```typescript
// BAD: unhandled promise
it('creates user', () => {
  request(app).post('/api/users').send({ name: 'test' });
  // test ends before request completes
});

// GOOD
it('creates user', async () => {
  await request(app).post('/api/users').send({ name: 'test' });
});
```

## 9. Senior Engineer Perspective

### 9.1 Integration Test Strategy

A senior engineer designs the integration test suite around risk:

- **Critical paths** (auth, payments, user data) get thorough integration coverage.
- **Read-only paths** (GET endpoints) get lighter coverage (a few smoke tests).
- **Error paths** are tested as thoroughly as happy paths.
- **Contract tests** are maintained for each external dependency.

### 9.2 Flaky Test Management

Flaky integration tests erode trust. Tactics to handle them:

- **Retry mechanism**: Use `jest.retryTimes(2)` for tests that interact with network services.
- **Test quarantining**: Move flaky tests to a separate CI pipeline step that does not block deployment.
- **Deterministic seeds**: Use libraries like `faker.seed(42)` for reproducible test data.
- **Polling with timeout**: Instead of arbitrary `setTimeout`, use `wait-for-expect`:

```typescript
import waitForExpect from 'wait-for-expect';

it('eventually processes the event', async () => {
  await waitForExpect(async () => {
    const user = await db('users').where({ email: 'test@test.com' }).first();
    expect(user.status).toBe('active');
  }, 10000, 500);
});
```

### 9.3 Testing in CI

```yaml
# .github/workflows/test.yml
jobs:
  integration:
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test
          POSTGRES_PASSWORD: test
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:integration
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/test
```

### 9.4 Test Architecture Decision Records

Senior engineers document why certain testing decisions were made:

- Why we chose transaction rollback vs. truncation
- Why we use test containers vs. dedicated test DB
- Which endpoints get integration vs. unit tests

## 10. Interview Questions (20: 10 Easy + 10 Medium)

### Easy

**Q1**: What is the difference between unit testing and integration testing?
**A**: Unit testing verifies individual components in isolation with mocked dependencies. Integration testing verifies that multiple components work together using real dependencies (database, file system, network).

**Q2**: What is supertest used for?
**A**: Supertest is a Node.js library for testing HTTP servers. It provides a high-level API for making requests and asserting on responses without binding to a real network port.

**Q3**: How do you set up a test database for integration tests?
**A**: Options include: (a) in-memory SQLite, (b) Docker test containers, (c) a dedicated test database instance that is migrated before the suite runs.

**Q4**: What is the purpose of `beforeAll` and `afterAll` in Jest?
**A**: `beforeAll` runs once before all tests in a describe block, typically used for setup (connecting to DB, seeding data). `afterAll` runs once after all tests for teardown.

**Q5**: What does `request(app).get('/path').expect(200)` do?
**A**: It sends an HTTP GET request to the Express app and asserts that the response status code is 200.

**Q6**: What is a test fixture?
**A**: A fixed data set loaded into the database before tests run, ensuring consistent starting state.

**Q7**: How do you handle authentication tokens in integration tests?
**A**: Generate a real token using the same auth function used in production (e.g., `jwt.sign()` or `createAuthToken()`) and pass it in the `Authorization` header.

**Q8**: What is the test pyramid?
**A**: A visual metaphor showing that you should have many unit tests, fewer integration tests, and even fewer end-to-end tests.

**Q9**: Why should integration tests avoid mocks?
**A**: Because the purpose of integration tests is to verify real interactions. Using mocks defeats the purpose and can hide interface defects.

**Q10**: What is the difference between `toEqual` and `toMatchObject` in Jest?
**A**: `toEqual` checks deep equality of all properties. `toMatchObject` checks that the expected object is a subset of the actual object.

### Medium

**Q11**: How do you test a file upload endpoint with supertest?
**A**: Use the `.attach('fieldName', filePath)` method to attach a file to the request, then assert on the response.

**Q12**: How do you prevent test pollution in integration tests?
**A**: Use transaction rollback (BEGIN/ROLLBACK per test), truncate tables between tests, or use a separate schema/database per test worker.

**Q13**: What is the difference between supertest with a real port vs. an in-process app?
**A**: With an in-process app (passing the Express app directly), supertest does not bind to a real port, making tests faster and avoiding port conflicts.

**Q14**: How do you test WebSocket connections in integration tests?
**A**: Use `ws` or `socket.io-client` to connect to the running server, emit events, and assert on responses. This typically requires the server to be listening on a real port.

**Q15**: How do you test database migrations?
**A**: Run migrations in `beforeAll`, then query the information schema to verify that tables, columns, indexes, and constraints exist as expected.

**Q16**: What is WireMock and when should you use it?
**A**: WireMock is a HTTP mock server. Use it when your application calls external APIs and you want to simulate responses without hitting real services.

**Q17**: How do you test a middleware that logs requests?
**A**: Use a custom stream or spy on `console.log`/logger, make a request with supertest, and assert that the log output contains the expected fields.

**Q18**: What is the role of `testTimeout` in Jest configuration?
**A**: It sets the maximum time (in ms) a test can run before Jest aborts it. Integration tests often need a higher timeout (e.g., 30000).

**Q19**: Can integration tests replace unit tests?
**A**: No. Integration tests are slower and harder to debug. Unit tests provide fast feedback for individual logic. Both are needed.

**Q20**: How do you test error handling middleware in Express?
**A**: Make a request that triggers an error (e.g., invalid body, bad ID), then assert that the response has the expected status code and error shape.

## 11. Advanced Interview Questions (20: 10 Hard + 10 System Design)

### Hard

**Q1**: How do you test a multi-tenant database where each tenant has an isolated schema?
**A**: Create a test helper that creates a new schema before each test, runs migrations, seeds tenant-specific data, and drops the schema after. Use a connection pool that accepts a schema parameter.

**Q2**: How do you implement integration tests for a saga pattern (distributed transaction) across multiple services?
**A**: Use test containers for all dependent services (DB, message queue, downstream APIs). Write tests that simulate the entire saga: send the initial command, verify compensating transactions on failure, and assert final state consistency.

**Q3**: How do you test a service that uses a read replica and a primary database?
**A**: Set up two database connections. Route writes to the primary, reads to the replica. After a write, test both immediate consistency (primary read) and eventual consistency (replica read with a retry loop).

**Q4**: What strategies do you use to test time-dependent logic (cron jobs, TTLs)?
**A**: (a) Inject a fake clock using `sinon.useFakeTimers` or `jest.useFakeTimers`. (b) For database-level TTLs, set a very short TTL in tests. (c) For cron jobs, test the job function directly by invoking it and checking side effects.

**Q5**: How do you test optimistic concurrency control (e.g., version fields)?
**A**: Create two separate requests that read the same entity, both modify it, and assert that the second write fails with a version conflict (HTTP 409). Test the retry logic.

**Q6**: How do you test database connection pool exhaustion?
**A**: Configure a pool with a very small max (e.g., 2). Make concurrent requests that hold connections open (e.g., slow queries). Assert that additional requests receive a 503 or timeout error.

**Q7**: How do you test a GraphQL resolver that aggregates data from multiple REST APIs?
**A**: Use WireMock or MockServer to simulate each REST endpoint with realistic responses. The GraphQL resolver integration test sends a query and asserts the aggregated result.

**Q8**: How do you handle testing with feature flags?
**A**: Set feature flag values in test setup (e.g., environment variables or a feature flag service mock). Write parameterized tests that run with flag on and off, asserting different behavior in each case.

**Q9**: How do you test a custom Express error handler that sends different responses in dev vs. prod?
**A**: Set `NODE_ENV=development` and `NODE_ENV=production` in separate test files or `describe` blocks. Assert that dev responses include stack traces while prod responses do not.

**Q10**: How do you integration test a rate limiter that uses Redis?
**A**: Use `@testcontainers/redis` to spin up a Redis container. Flush Redis between tests. Send requests up to the limit and assert 200, then send one more and assert 429.

### System Design

**Q11**: Design an integration test suite for an event-driven microservices system with 10 services.
**A**: Use a shared test library with test container factories. Each service has contract tests (pact) for its API. Integration tests run against Docker Compose with the service under test and its immediate dependencies. Use message queue listeners to verify async flows. CI runs integration tests in parallel using matrix builds, each with its own compose file.

**Q12**: How would you test a payment system that integrates with Stripe, PayPal, and a custom ledger?
**A**: Use test containers for the ledger database. Use Stripe test keys and PayPal sandbox. Write tests for: (a) successful payment flow, (b) declined card, (c) expired card, (d) insufficient funds, (e) refund, (f) idempotency (same idempotency key returns same result), (g) webhook handling (signature verification, event dedup).

**Q13**: Design a testing strategy for a real-time collaboration app (like Google Docs) using WebSockets and CRDTs.
**A**: Integration tests: (a) connect multiple WebSocket clients, (b) simulate concurrent edits, (c) assert that all clients converge to the same document state, (d) test reconnection and state sync after disconnect, (e) test presence (who is online). Use `ws` library with multiple connections. Use a test timeout of 30s for convergence detection.

**Q14**: How would you test a recommendation engine that uses a mix of SQL, Redis cache, and a machine learning service?
**A**: Use test containers for SQL and Redis. Mock the ML service with WireMock (return fixed recommendations). Test: (a) cache hit returns quickly, (b) cache miss fetches from ML and populates cache, (c) fallback when ML service is down, (d) personalized vs. generic recommendations. Assert on response time and content.

**Q15**: Design integration tests for a data pipeline that ingests CSV files, transforms them, and loads into a warehouse.
**A**: Use test containers for the warehouse (e.g., ClickHouse). Place test CSV files in a monitored directory. Assert that: (a) the file is consumed and moved to processed/, (b) the transformed data appears in the warehouse, (c) malformed CSVs are moved to error/ and logged, (d) duplicate files are deduped, (e) schema changes are handled. Use `chokidar` or `fs.watch` to trigger assertions.

**Q16**: How do you test a caching layer with stale-while-revalidate semantics?
**A**: (a) Seed cache with stale data. (b) Request the endpoint and assert that stale data is returned immediately. (c) Wait for the revalidation to complete. (d) Request again and assert fresh data. (e) Test that concurrent requests coalesce (only one revalidation request). Use a fake upstream that introduces a configurable delay.

**Q17**: Design tests for a service mesh (e.g., Istio) integration including retries, circuit breaking, and timeouts.
**A**: Deploy the service and a test harness in a Kubernetes cluster (or use kind/k3s). Integration tests: (a) inject HTTP 503 from downstream and verify retries, (b) trip the circuit breaker by sending many failing requests, then verify subsequent requests fail fast, (c) set a very short timeout and verify the client receives a 504. Use `fortio` or `vegeta` for load generation.

**Q18**: How would you test a search service that uses Elasticsearch with custom analyzers?
**A**: Use `@testcontainers/elasticsearch` to start Elasticsearch. Create the index with the custom analyzer in `beforeAll`. Test: (a) exact match, (b) fuzzy match, (c) stemming (e.g., "running" matches "run"), (d) stop words are ignored, (e) pagination, (f) highlighting, (g) aggregation queries. Assert on `hits.total` and `hits.hits`.

**Q19**: Design an integration test framework for a serverless application (AWS Lambda + API Gateway + DynamoDB).
**A**: Use SST (Serverless Stack) or AWS CDK to deploy a test stack. Use `dynamodb-local` (test container). Write tests that invoke the Lambda function directly (via AWS SDK) or through a local API Gateway. Test: (a) CRUD operations, (b) IAM authorization (cognito), (c) DynamoDB transactions, (d) stream processing (DynamoDB Streams + Lambda), (e) idempotent retries.

**Q20**: How do you test a multi-region database replication setup?
**A**: This is difficult to test locally. Strategies: (a) use two test containers with logical replication configured, (b) write to the primary, poll the replica until data appears (eventual consistency test), (c) failover test: stop the primary, verify the replica promotes, (d) conflict resolution: write to both regions simultaneously and verify the conflict resolution policy (last-writer-wins, CRDT, etc.).

## 12. Expert-Level Interview Questions (10: Architect-Level)

**Q1**: Design a test strategy for a legacy monolith being decomposed into microservices. How do you ensure integration tests catch regressions during the migration?
**A**: Use a strangler fig pattern. Maintain a parallel test suite that runs against both the monolith and the new services. Implement contract tests (consumer-driven) for each bounded context. Add a "compatibility" CI stage that runs the old integration tests against the new service endpoints. Measure coverage of the seam between monolith and extracted service. Use feature flags to toggle between old and new implementations in production, and write integration tests that run with both flag states.

**Q2**: How do you achieve deterministic integration testing in a system that uses real time (not wall-clock time) for scheduling?
**A**: Replace the system clock with a virtual clock at the application boundary. Use dependency injection so that all components that need the current time use a `Clock` interface. In tests, inject a `FakeClock` that you can advance manually. For database-level time (e.g., `NOW()`), use a database that supports time override (e.g., PostgreSQL with `pg_timezone` extension) or wrap the DB driver to inject timestamps. For distributed scheduling (e.g., cron across services), use a centralized scheduler service that can be paused and advanced in tests.

**Q3**: You have a system that uses eventual consistency with a 5-minute reconciliation window. How do you test this without waiting 5 minutes in every test?
**A**: (a) Make the reconciliation interval configurable and set it to 100ms in tests. (b) Use a deterministic event scheduler that processes reconciliation immediately when triggered. (c) Use a test hook that manually calls the reconciliation function and waits for completion. (d) For end-to-end correctness, run a small number of tests with the real interval in a nightly pipeline. The key design principle: make time a pluggable dependency at every layer.

**Q4**: Propose an integration test architecture for a system that must comply with PCI-DSS, SOC2, and GDPR simultaneously.
**A**: Compliance requirements add test dimensions: (a) PCI: test that credit card data is never logged, always encrypted at rest and in transit, and access is audited. Write integration tests that intercept log output and assert no PAN data. Use test containers with encryption-enabled databases. (b) SOC2: test access controls (RBAC), audit trails, and change management. Every integration test should exercise auth and produce audit log entries. (c) GDPR: test data deletion (right to erasure), data export, consent management. Write tests that create a user, exercise data, delete the user, and verify all related records are anonymized or removed. Use a compliance test suite that runs as a separate CI stage with enhanced assertions.

**Q5**: How would you design a "pact-like" contract testing framework without using Pact, to run in a CI environment that cannot install Ruby or Docker?
**A**: Build a lightweight contract testing library: each provider publishes an OpenAPI 3.0 spec. Each consumer generates expectations from the spec using a TypeScript type-to-schema compiler. The consumer tests record actual requests/responses as contract files (JSON). The provider tests replay the consumer contracts: for each consumer, the provider starts its app, sends the recorded request, and asserts the response matches the recorded response. The matcher supports regex-based pattern matching (like Pact's `like()` and `term()`). The framework runs as a Jest reporter that compares contracts across builds and warns on breaking changes.

**Q6**: How do you test a system that uses CRDTs for offline-first collaboration with millions of expected concurrent users?
**A**: Integration tests should focus on convergence and conflict resolution: (a) create N replicas, each with the same initial state, (b) apply different operations on each replica in random order, (c) sync all replicas and assert convergent state. Use property-based testing (fast-check) to generate random operation sequences. Test edge cases: concurrent moves of the same element, concurrent insert/delete at the same position, network partitioning and merging. Use a test cluster with configurable latency and partition simulation (e.g., using `toxiproxy`). Measure convergence time (number of sync rounds) and message size. For scale testing, simulate millions of operations in CI using a benchmark harness, not the full integration suite.

**Q7**: You are building a platform that must integrate with 50+ third-party APIs, each with different authentication, rate limits, and SLAs. Design the integration test approach.
**A**: Categorize APIs into tiers: Tier 1 (critical, e.g., payment): use sandbox environments with real test credentials; run full integration suite against them nightly. Tier 2 (important, e.g., shipping): use record-and-replay (VCR-like) with `nock` or `polly-js`; periodically re-record cassettes to detect API changes. Tier 3 (non-critical, e.g., analytics): use WireMock with manually defined stubs. Create an API Health Check integration test that runs every minute in staging and alerts on failures. Use a circuit breaker pattern and test that the system degrades gracefully when each API is unavailable. Maintain contract tests (OpenAPI) for each API's most critical endpoints.

**Q8**: How do you integration test a database sharding layer where data is distributed across 100+ physical shards based on a hash of the tenant ID?
**A**: Use a test shard router that routes to in-memory databases. Write tests that: (a) verify that data for tenant A goes to shard 1 and data for tenant B goes to shard 2 (hash consistency), (b) verify cross-shard queries (if supported) or their absence, (c) test rebalancing (add a new shard, trigger migration, assert data is redistributed without loss), (d) test shard failure (bring one shard down, verify reads fall back to replicas or return partial results). Use a configurable shard count (e.g., 4 test shards). Assert that queries are routed to the correct shard by inspecting query logs or using a wrapper proxy that records shard assignments.

**Q9**: How do you integration test a Feature Store (as used in ML pipelines) that must serve features at low latency while supporting point-in-time correctness for training?
**A**: Use test containers for the online store (e.g., Redis) and offline store (e.g., S3-compatible MinIO). Test: (a) feature ingestion (batch and streaming), (b) point-in-time lookup (query features as of a historical timestamp; assert correct values), (c) serving consistency (same feature key returns same value from online and offline stores), (d) staleness handling (feature TTL expiry), (e) backfill (recompute features for a historical time range and assert training data reproducibility). Use a deterministic feature generator (seeded random) for reproducible tests.

**Q10**: You need to integrate a testing strategy across 20 teams, each using different languages (Node, Python, Go, Java). How do you enforce integration testing standards?
**A**: Mandate a contract-first approach: every service must publish an OpenAPI or protobuf spec. Use a centralized contract registry with breaking change detection (e.g., `openapi-diff`). Each team's CI runs a compliance check: does the service pass its own contract's examples? Does it pass consumer-driven contract tests? Centralize test infrastructure: provide a shared test container library (Docker images for common dependencies), a "test harness" CLI that sets up databases, WireMock, and message queues with one command. Use a polyglot integration test runner (e.g., `Testcontainers` with language bindings for each stack). Enforce test coverage gates via a shared CI pipeline that aggregates coverage reports. Run a "global integration test night" that deploys all services to a staging environment and runs cross-service scenarios.

## 13. Debugging & Troubleshooting

### 13.1 Common Error: ECONNREFUSED

```text
Error: connect ECONNREFUSED 127.0.0.1:5432
```

**Root cause**: The database container or service is not running.

**Fix**: Ensure test containers or a local DB is started in your Jest global setup. Check that the `DATABASE_URL` environment variable is set.

```typescript
// jest.globalSetup.ts
import { PostgreSqlContainer } from '@testcontainers/postgresql';

export default async function () {
  const container = await new PostgreSqlContainer()
    .withDatabase('test')
    .start();
  process.env.__DATABASE_URL__ = container.getConnectionUri();
}
```

### 13.2 Test Timeout

```text
Timeout - Async callback was not invoked within the 5000 ms timeout specified by jest.setTimeout.Timeout
```

**Root cause**: The test takes longer than the default 5s timeout.

**Fix**: Increase the timeout in your test config:

```typescript
// jest.config.ts
export default {
  testTimeout: 30000,
};
```

Or per-test:

```typescript
jest.setTimeout(60000);
```

### 13.3 Database Lock Errors

```text
deadlock detected
ERROR:  duplicate key value violates unique constraint
```

**Root cause**: Tests running in parallel are operating on the same database rows or attempting to insert conflicting data.

**Fix**: Ensure each test worker has its own database or schema. Use `isolatedModules: true` or `--runInBand` for serial execution as a last resort.

```typescript
// jest.config.ts
export default {
  maxWorkers: 1, // Run integration tests serially
};
```

### 13.4 Flaky Tests Due to Async Side Effects

```text
Received: undefined
    expect(received).resolves.toEqual(expected)
```

**Root cause**: The test resolved before the side effect completed.

**Fix**: Use `wait-for-expect` or poll the database until the expected state appears.

```typescript
await waitForExpect(async () => {
  const user = await db('users').where({ email: 'test@test.com' }).first();
  expect(user).toBeDefined();
});
```

### 13.5 Debugging Supertest Responses

```typescript
// Print the full response for debugging
const res = await request(app).get('/api/users/1');
console.log({ status: res.status, body: res.body, headers: res.headers });
```

Or use the `.expect()` chaining to inspect intermediate results:

```typescript
const res = await request(app)
  .get('/api/users/1')
  .expect((res) => {
    // Custom assertion: log and check
    console.log(res.body);
    if (!res.body.id) throw new Error('Missing id');
  });
```

### 13.6 Nock Not Intercepting

```typescript
// If nock is not intercepting, ensure it is loaded before your app module
import nock from 'nock';
beforeAll(() => {
  nock.disableNetConnect(); // Blocks all outgoing HTTP
  nock.enableNetConnect(/127\.0\.0\.1/); // Allow localhost
});
```

## 14. Comparison Section

### 14.1 Integration Testing vs. Unit Testing

| Aspect | Unit Test | Integration Test |
|--------|-----------|-----------------|
| Scope | Single function/module | Multiple modules/services |
| Dependencies | All mocked | Real (DB, FS, network) |
| Speed | Milliseconds | Milliseconds to seconds |
| Flakiness | Low | Medium |
| Debugging | Easy | Harder (real I/O) |
| Coverage target | Logic paths | Interface contracts |
| CI frequency | Every commit | Every commit (subset) |

### 14.2 Supertest vs. node-fetch vs. axios for Testing

| Library | Pros | Cons |
|---------|------|------|
| Supertest | Built-in assertions, no port binding, Express integration | HTTP client only, no gRPC/WebSocket |
| node-fetch | Standard API, lightweight | No assertion helpers |
| axios | Interceptors, wide adoption | Heavier, no built-in test assertions |

### 14.3 Transaction Rollback vs. Truncate

| Strategy | Pros | Cons |
|----------|------|------|
| Transaction rollback | Fastest, no cleanup needed | Requires DB support, nested transactions complex |
| Truncate between tests | Simple, works everywhere | Slower (DDL), auto-increment resets |
| Savepoints | Balance of both | More complex setup |

### 14.4 Test Container vs. Dedicated Test DB

| Approach | Pros | Cons |
|----------|------|------|
| Test containers | Isolated, version-controlled, CI-friendly | Slower (Docker startup) |
| Dedicated test DB | Faster startup, shared resources | State pollution across suites, CI complexity |

### 14.5 In-Memory DB vs. Production-Like DB

| Aspect | In-Memory (SQLite) | Production-Like (PostgreSQL) |
|--------|-------------------|------------------------------|
| Speed | Very fast | Moderate |
| SQL compatibility | Subset (no window functions, partial index) | Full |
| Migration testing | Limited | Full |
| CI setup | None | Docker container needed |

## 15. Revision Notes

### Key Points to Remember

- Integration tests verify that real components work together.
- Use supertest for HTTP integration testing (no port binding).
- Use transaction rollback or truncation for test isolation.
- Use test containers for databases and external services.
- Integration tests are slower than unit tests but faster than E2E.
- Always test error paths, auth bypass, injection, and rate limiting.
- Flaky tests destroy trust: use retries, quarantines, and polling.
- Contract testing prevents breaking changes between services.
- Each test worker needs its own database to avoid state pollution.
- In-memory databases trade SQL compatibility for speed.

### Common Acronyms

- **IoC**: Inversion of Control
- **SUT**: System Under Test
- **ADR**: Architecture Decision Record
- **CDC**: Consumer-Driven Contract
- **SLA**: Service Level Agreement
- **CRDT**: Conflict-Free Replicated Data Type

## 16. Cheat Sheet

```text
+==============================================================================+
|                    INTEGRATION TESTING CHEAT SHEET                           |
+==============================================================================+

+--- SETUP -------------------------------------------------------------------+
|                                                                              |
|  import request from 'supertest';                                           |
|  import { app } from '../app';                                              |
|  import { db } from '../db';                                                |
|                                                                              |
|  beforeAll(async () => {  /* migrate DB */ } )                              |
|  afterAll(async () => {   /* close DB */  } )                               |
|  beforeEach(async () => { /* begin txn */ } )                               |
|  afterEach(async () => {  /* rollback */  } )                               |
|                                                                              |
+--- REQUEST METHODS ---------------------------------------------------------+
|                                                                              |
|  .get(url)               .post(url)          .put(url)                      |
|  .patch(url)             .delete(url)        .options(url)                  |
|                                                                              |
|  .send(body)             .set(key, value)    .query(params)                 |
|  .attach(field, path)    .field(name, val)   .redirects(n)                  |
|  .auth(user, pass)       .buffer()           .responseType('blob')          |
|                                                                              |
+--- ASSERTIONS --------------------------------------------------------------+
|                                                                              |
|  .expect(status)                    .expect('Content-Type', /json/)         |
|  .expect('Content-Length', '100')   .expect(hasHeader)                      |
|  .expect((res) => { /* custom */ })                                         |
|                                                                              |
|  expect(res.body).toEqual(expected)                                         |
|  expect(res.body).toMatchObject(partial)                                    |
|  expect(res.body).toHaveProperty('key')                                     |
|  expect(res.body).toEqual(expect.arrayContaining([...]))                    |
|                                                                              |
+--- DATABASE HELPERS --------------------------------------------------------+
|                                                                              |
|  // Transaction rollback                                                    |
|  beforeEach(() => db.query('BEGIN'));                                       |
|  afterEach(() => db.query('ROLLBACK'));                                     |
|                                                                              |
|  // Truncate all tables                                                     |
|  afterEach(async () => {                                                    |
|    const tables = await db('information_schema.tables')                     |
|      .where('table_schema', 'public')                                       |
|      .select('table_name');                                                 |
|    for (const { table_name } of tables) {                                   |
|      await db.raw(`TRUNCATE TABLE ${table_name} CASCADE`);                  |
|    }                                                                        |
|  });                                                                        |
|                                                                              |
+--- TEST CONTAINERS ---------------------------------------------------------+
|                                                                              |
|  import { PostgreSqlContainer } from '@testcontainers/postgresql';          |
|                                                                              |
|  const container = await new PostgreSqlContainer()                          |
|    .withDatabase('testdb')                                                  |
|    .start();                                                                |
|  process.env.DATABASE_URL = container.getConnectionUri();                   |
|                                                                              |
+--- EXTERNAL API MOCKING ----------------------------------------------------+
|                                                                              |
|  import nock from 'nock';                                                   |
|                                                                              |
|  nock('https://api.stripe.com')                                             |
|    .post('/v1/charges')                                                     |
|    .reply(200, { id: 'ch_123' });                                           |
|                                                                              |
|  import { WireMock } from 'wiremock-captain';                               |
|  const wiremock = new WireMock('http://localhost:8089');                    |
|  await wiremock.registerStub({ request: {...}, response: {...} });          |
|                                                                              |
+--- WAITING FOR ASYNC -------------------------------------------------------+
|                                                                              |
|  import waitForExpect from 'wait-for-expect';                               |
|                                                                              |
|  await waitForExpect(async () => {                                          |
|    const result = await checkSideEffect();                                  |
|    expect(result).toBe(expected);                                           |
|  }, 10000, 500);                                                            |
|                                                                              |
+--- COMMON PATTERNS ---------------------------------------------------------+
|                                                                              |
|  // Auth header                                                             |
|  .set('Authorization', `Bearer ${token}`)                                   |
|                                                                              |
|  // JSON body                                                               |
|  .send({ key: 'value' })                                                    |
|                                                                              |
|  // Query params                                                            |
|  .query({ page: 1, limit: 20 })                                             |
|                                                                              |
|  // File upload                                                             |
|  .attach('avatar', '/path/to/photo.jpg')                                    |
|                                                                              |
|  // Cookies                                                                 |
|  .set('Cookie', 'sessionId=abc123')                                         |
|                                                                              |
+--- CI CONFIGURATION --------------------------------------------------------+
|                                                                              |
|  # .github/workflows/test.yml                                               |
|  services:                                                                  |
|    postgres:                                                                |
|      image: postgres:16-alpine                                              |
|      env: { POSTGRES_DB: test, POSTGRES_PASSWORD: test }                   |
|  steps:                                                                     |
|    - run: npm run test:integration                                          |
|                                                                              |
+==============================================================================+
```
