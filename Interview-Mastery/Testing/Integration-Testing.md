# Integration Testing

---

## Overview

- **Definition:** Integration testing verifies that different components, modules, or services work together correctly. It tests the interactions between units, not the units themselves.
- **Why It Exists:** Unit tests verify individual functions in isolation, but bugs often live at the boundaries between components — wrong data format, missing fields, incorrect protocol, timing issues. Integration testing catches these interface-level defects.
- **Key Concepts:** **Component boundaries** (where one module calls another), **Real dependencies** (database, filesystem, network), **Test doubles** (only for external systems outside your control), **Fixture setup/teardown** (known state), **Contract testing** (API agreements), **Smoke/sanity tests** (critical paths).

---

## Core Concepts

### Testing Pyramid

```
    /\         E2E (slow, brittle, high confidence)
   /  \
  /    \       Integration (medium speed, medium confidence)
 /______\
/________\     Unit (fast, precise, low confidence)
```

Integration tests sit in the middle — more coverage than unit tests but more stable than E2E.

### Types of Integration Tests

| Type | What It Tests | Example |
|------|--------------|---------|
| **Component** | One component with real dependencies | OrderService + real DB |
| **Contract** | API provider-consumer agreement | Consumer Pact tests |
| **Database** | SQL queries, migrations, ORM mapping | Repository tests |
| **File System** | Read/write, permissions, encoding | CSV import/export |
| **Network** | HTTP calls, sockets, message queues | Service-to-service |
| **Third-party** | External API integration | Stripe, SendGrid |

### Database Integration Testing

```javascript
const db = require('../db');
const UserRepository = require('../repositories/userRepository');

describe('UserRepository', () => {
  beforeEach(async () => {
    await db.migrate.latest();
    await db.seed.run();
  });

  afterEach(async () => {
    await db.migrate.rollback();
  });

  it('should find user by email', async () => {
    const user = await UserRepository.findByEmail('alice@test.com');
    expect(user).toBeDefined();
    expect(user.email).toBe('alice@test.com');
  });

  it('should create and persist a user', async () => {
    const newUser = await UserRepository.create({
      name: 'Bob',
      email: 'bob@test.com'
    });
    const fetchedUser = await UserRepository.findById(newUser.id);
    expect(fetchedUser.name).toBe('Bob');
  });
});
```

### Service-to-Service Integration Testing

```javascript
const nock = require('nock');
const PaymentService = require('../services/paymentService');

describe('PaymentService', () => {
  afterEach(() => {
    nock.cleanAll();
  });

  it('should process payment with Stripe', async () => {
    nock('https://api.stripe.com')
      .post('/v1/charges')
      .reply(200, { id: 'ch_123', status: 'succeeded' });

    const result = await PaymentService.charge(1000, 'usd', 'tok_visa');
    expect(result.status).toBe('succeeded');
  });

  it('should handle Stripe 402 error', async () => {
    nock('https://api.stripe.com')
      .post('/v1/charges')
      .reply(402, { error: { message: 'Card declined' } });

    await expect(
      PaymentService.charge(1000, 'usd', 'tok_chargeDeclined')
    ).rejects.toThrow(/card declined/i);
  });
});
```

### Contract Testing with Pact

```javascript
const { Pact } = require('@pact-foundation/pact');

const provider = new Pact({
  consumer: 'OrderService',
  provider: 'PaymentService',
  port: 1234,
});

describe('OrderService -> PaymentService contract', () => {
  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());

  it('should accept valid payment requests', async () => {
    await provider.addInteraction({
      state: 'payment service is available',
      uponReceiving: 'a payment request',
      withRequest: {
        method: 'POST',
        path: '/payments',
        headers: { 'Content-Type': 'application/json' },
        body: { amount: 1000 },
      },
      willRespondWith: {
        status: 201,
        body: { id: 'pay_123', status: 'completed' },
      },
    });

    const res = await fetch('http://localhost:1234/payments', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ amount: 1000 }),
    });

    expect(res.status).toBe(201);
  });
});
```

---

## Common Mistakes

- **No database integration tests** — ORM abstractions hide SQL bugs (N+1 queries, wrong joins, missing indexes). This *looks correct* because the ORM handles SQL generation automatically and the application works during development, so writing explicit database tests feels like re-testing the ORM framework itself.
- **Using H2 for testing against PostgreSQL** — H2 doesn't support PostgreSQL-specific features (JSONB, array types, partial indexes); tests pass but prod fails. This *looks correct* because H2 starts instantly, requires no Docker setup, and all tests pass during CI — the PostgreSQL incompatibility only surfaces after deployment.
- **Shared mutable state between tests** — Tests become order-dependent and flaky. This *looks correct* because sharing database rows or fixture data reduces test setup duplication, and the flakiness is intermittent enough to blame on timing rather than design.
- **Testing too many units at once** — Hard to find the root cause when integration fails. This *looks correct* because testing a larger slice of the system feels more realistic and catches more interactions, making the test seem more valuable despite the debugging difficulty.
- **No contract tests for external APIs** — Provider changes silently break consumers. This *looks correct* because the external API works today, and writing contract tests for something you do not control feels like effort with no immediate payoff.
- **Not cleaning up test data** — Tests accumulate data that violates unique constraints. This *looks correct* because each test's small data contribution seems negligible, and cleanup code adds boilerplate that feels unrelated to the test's purpose.
- **Hardcoded ports or URLs** — Tests fail in CI with different environments; use env vars. This *looks correct* because hardcoding works on the developer's machine and one less environment variable to configure feels like simplification.
- **Mocking everything** — If every external call is mocked, you're testing mocks, not integration. This *looks correct* because mocked tests are fast, deterministic, and never fail due to network issues, creating the illusion of reliable coverage.
- **Flaky tests ignored** — "It only fails sometimes" hides real bugs; fix or remove. This *looks correct* because the test passes on the developer's machine and in most CI runs, so the occasional failure gets attributed to environmental noise rather than a genuine defect.
- **No negative tests** — Testing only success paths misses 90% of bugs. This *looks correct* because the happy path proves the feature works under normal conditions, and designing failure scenarios feels speculative compared to verifying expected behavior.

---

## Key Design Considerations

- **Database per test or per suite:** Use transactions that roll back after each test for isolation. Use separate test databases per developer/CI build.
- **Testcontainers** — Run real databases in Docker containers for integration tests. Slower but more accurate than embedded databases.
- **Environment parity** — Production-like setup: same DB version, same OS, same locale. Differences between test and prod environments cause bugs.
- **Fakes vs mocks:** Use fakes (lightweight implementations) for testing; use mocks/spies for verifying interactions. Fakes are more realistic.
- **Test data factory** — Create reusable factories for test data to avoid duplication. Use libraries like Factory Bot (Ruby) or Fishery (JS).
- **Consumer-driven contracts** — Each consumer publishes its expectations; the provider verifies against all consumer contracts. Prevents breaking consumers accidentally.
- **Gatling/k6 for load testing** — Integration tests should also verify performance characteristics under realistic load.

---

## Real-World Scenarios

### Scenario 1: Microservice Dependency Chaos
Service A calls Service B, which calls Service C, which calls Service D. To test Service A, the team starts all 4 services locally. Tests take 30 seconds to start and are flaky because any service can be down. **Fix:** Split into focused component integration tests. Test Service A + a real database + mocked Service B (using WireMock). Test Service B + real database + mocked C. Use contract tests (Pact) to ensure A→B and B→C contracts are correct. Result: tests run in 2 seconds, are deterministic, and catch interface mismatches through contract tests.

### Scenario 2: Database Migration Gone Wrong
A team adds a `NOT NULL` column to a table with 10M rows. The migration passes in CI (which has 100 rows of test data) but fails in production (timeout on backfilling 10M rows, and existing null values violate the constraint). **Fix:** Test migrations against a realistic data volume. Use a subset of production data (anonymized) in a staging environment. Test the migration with `LOCK_TIMEOUT` and verify it completes within the maintenance window. Also test rollback: if the migration fails midway, can you restore?

### Scenario 3: Third-Party API Contract Break
Your e-commerce app integrates with Stripe for payments. Stripe releases a new API version that changes the `charge` response format. Your integration tests mock Stripe and still pass, but production starts failing. **Fix:** Don't mock Stripe in integration tests — use Stripe's test mode (real sandbox). Better: use consumer-driven contract tests where your app (consumer) publishes its expectations. When Stripe changes their API, your contract tests fail before deployment. Additionally, pin your Stripe API version and upgrade on your schedule, not Stripe's.

---

## Scenario-Based Questions

1. **Q: You are building an order processing system. The `OrderService` calls `PaymentService`, which calls `InventoryService`, which calls `ShippingService`. All unit tests pass, but end-to-end orders fail with data corruption. Where is the gap?**
   A: Missing integration tests at each service boundary. Unit tests verify each service in isolation but not the data flow between them. Fix: write component integration tests — test `OrderService` + real database + mocked downstream. Add contract tests between each pair (Order↔Payment, Payment↔Inventory). Test with real serialization (JSON/Protobuf) to catch format mismatches.

2. **Q: Your team uses H2 in-memory database for integration tests because it's fast. Tests pass perfectly. In production with PostgreSQL, unique constraint violations and deadlocks appear. What went wrong?**
   A: H2 is not PostgreSQL-compatible. It doesn't support PostgreSQL-specific features (partial indexes, JSONB operators, `RETURNING`, locking semantics, MVCC behavior). Fix: use Testcontainers to spin up a real PostgreSQL container per test suite. The trade-off: tests are slower (container startup takes 5-10s) but catch real database issues. Alternative: use `testcontainers-java` or `testcontainers-node` with module-level lifecycle (start once per file, not per test).

> **Interview follow-up:** If Testcontainers is too slow for your CI budget, what alternative approaches still catch PostgreSQL-specific bugs without running a real container per test suite?

3. **Q: An integration test for a background job processor is flaky — it sometimes completes before the job starts, sometimes not. The team wants to add a sleep(). How do you fix this properly?**
   A: Never use sleep() — it makes tests slow and still flaky. Use a test-specific fake job queue where job completion is deterministic. Instead of: (1) submit job, (2) wait, (3) assert — use: (1) submit job, (2) invoke the job handler directly with the same data, (3) assert. For timing-sensitive tests, use a `CountDownLatch` or `Promise` that resolves when the job handler completes.

4. **Q: A microservice integration test requires 6 running containers (DB, Redis, 3 other services, message queue). It takes 30 seconds to start and never runs locally. How do you make integration testing practical?**
   A: This test is too broad — it's an E2E test, not an integration test. Split into component integration tests: each service tests with its own database + mocked downstream services using WireMock or similar. Use contract tests (Pact) to verify service-to-service contracts. Keep only 1-2 critical-paths as full E2E tests. Result: 50 fast component tests (seconds) + 2 slow E2E tests (run in CI only).

> **Interview follow-up:** How do contract tests between services differ from what a WireMock stub would verify — and what bug does one catch that the other misses?

5. **Q: Your CI pipeline adds a new database column. An unrelated microservice's API starts returning 500s in production. How could contract testing have prevented this?**
   A: The new column likely changed the API response shape (e.g., serializing the new column in a response that the consumer doesn't expect). Consumer-driven contract tests (Pact) would catch this: the consumer publishes its expectations (response schema), and the provider verifies against all consumer contracts before deploying. Fix: implement Pact tests at each service boundary. When the API changes, contract tests fail before deployment.

6. **Q: A team mocks the database in all integration tests. Tests run fast but miss N+1 queries, wrong JOINs, and missing indexes. The team argues mocks are "good enough." How do you convince them otherwise?**
   A: Run a comparison: the mocked tests pass 100% of the time, but the team spends 20% of each sprint debugging database issues in production. Implement one real database integration test for the critical path (e.g., order checkout) and measure the bugs caught. Show the data: "3 production incidents this quarter were SQL-related bugs that the mocked tests would never catch." Propose a hybrid: 80% mocked for speed, 20% real for correctness.

7. **Q: An integration test for a file upload feature uses a real filesystem but doesn't clean up between tests. Tests pass locally but fail in CI because of accumulated test files. How do you fix this?**
   A: Use a temporary directory per test. Create a temp folder in `beforeEach`, delete it in `afterEach`. Use `fs.mkdtempSync()` (Node.js) or `java.nio.file.Files.createTempDirectory()`. For tests that must use a specific path, use dependency injection — inject a `FileStorage` interface, and in tests, inject a `TempFileStorage` implementation that cleans up automatically.

8. **Q: A database migration test passes in CI with 100 rows of test data but fails in production with 10M rows. The column backfill takes 45 minutes in production. What testing approach would catch this?**
   A: Test migrations against realistic data volumes. Create a "performance migration test" suite that runs against a database with production-scale data (anonymized subset). Assert that migrations complete within the maintenance window. Test both forward and rollback migrations. Use `EXPLAIN ANALYZE` in tests to verify query plans use indexes. Also test with concurrent reads/writes to verify locking doesn't block production traffic.

> **Interview follow-up:** How would you write an automated test that verifies a migration completes within a specific time budget, rather than just checking that it does not error?

9. **Q: You have two services communicating through a message queue. Service A publishes an event, Service B consumes it. A schema change in the event payload causes Service B to deserialize incorrectly. How do you test this?**
   A: Use schema registry (Avro/Protobuf) with compatibility validation. In integration tests: (1) Publish an event with the new schema, (2) Start a consumer with the old schema — verify it still works (backward compatible), (3) Publish an event with the old schema, start a consumer with the new schema — verify it still works (forward compatible). Use contract tests where the consumer defines the expected schema.

10. **Q: You need to test that a service correctly handles a downstream service returning 503 Service Unavailable. Your current tests mock this, but the mock doesn't simulate real network behavior (connection reset, slow response, partial data). How do you test resilience properly?**
    A: Use a fault-injection proxy like Toxiproxy or Chaos Monkey. Set up a real test scenario: (1) Downstream service is healthy — test normal flow. (2) Inject network latency (2s delay) — verify timeout handling. (3) Inject connection reset — verify retry logic. (4) Inject 503 — verify circuit breaker opens. (5) Restore service — verify circuit breaker closes after health check succeeds. Don't mock network behavior; test with real network conditions.

---

## Interview Questions

1. **What is integration testing?**
   A: Verifying that different components, modules, or services work together correctly. Tests the interactions at boundaries, not the units themselves.

2. **What is the difference between unit testing and integration testing?**
   A: Unit tests test a single unit in isolation (mocked dependencies). Integration tests test the interaction between real components (real DB, real filesystem, real network).

3. **What is Testcontainers?**
   A: A library that spins up real Docker containers (PostgreSQL, Redis, etc.) for integration tests. Provides more realistic testing than embedded/in-memory databases.

4. **What is contract testing?**
   A: Testing that an API provider meets the expectations of its consumers. Each consumer defines its expected contract; the provider must satisfy all contracts before deploying. Tools: Pact, Spring Cloud Contract.

5. **What is the N+1 query problem and how do you catch it in integration tests?**
   A: When an ORM executes N additional queries for N items after the initial query. Catch it with a query counter in integration tests — assert query count is below a threshold.

6. **What is a test double? Give examples.**
   A: A replacement for a real dependency. Types: dummy (passes data), fake (working but simplified), stub (returns canned answers), spy (records calls), mock (expects specific calls).

7. **Why is H2 a bad choice for testing PostgreSQL-specific features?**
   A: H2 lacks PostgreSQL features: JSONB operators, partial indexes, `RETURNING`, `ON CONFLICT`, MVCC semantics, and locking behavior. Tests pass on H2 but fail on PostgreSQL.

8. **What is the difference between a mock and a fake?**
   A: A mock verifies interactions (was `save()` called with the right argument?). A fake is a lightweight working implementation (in-memory database). Use mocks for behavior verification, fakes for state verification.

9. **What is the purpose of a smoke test in integration testing?**
   A: A quick check that the critical path of the system works. Runs after deployment to verify the system is alive before running the full test suite. Examples: health check endpoint, database ping.

10. **How do you test database migrations?**
    A: (1) Apply migration A, verify schema matches expected. (2) Seed test data. (3) Apply migration B (the one under test). (4) Assert data integrity, no null constraints violated. (5) Roll back migration B. (6) Assert the schema reverts correctly. Test against realistic data volumes.

---

## Developer Recommendations

- **Use Testcontainers for real database testing, not H2** — H2 is incompatible with PostgreSQL-specific features. Testcontainers spins up a real PostgreSQL container that behaves exactly like production. Trade-off: 5-10s slower startup per test suite. Benefit: zero "works on H2, fails on Postgres" bugs. A team used H2 for two years with all tests passing, then discovered their entire audit logging system had silently stored hundreds of gigabytes of duplicate rows in production because H2's `ON CONFLICT DO UPDATE` semantics differed from PostgreSQL's — the data cleanup ran for 14 hours and incurred significant compute costs.

- **Split broad integration tests into focused component tests** — A test that requires 5 running services is an E2E test, not an integration test. Test each service with its own database + mocked downstream services. Use contract tests for service-to-service boundaries. Trade-off: more test files. Benefit: faster, more deterministic tests that pinpoint failures.

- **Never mock the database** — Mocking the database means you're not testing SQL queries, constraints, or database-specific behavior. Use real databases (Testcontainers) for integration tests. Keep mocks for unit tests only. Trade-off: slower tests. Benefit: catches N+1 queries, constraint violations, and wrong JOINs. A team mocked every database call and maintained 95% test pass rates, but spent 30% of each sprint debugging SQL issues in production — the first real database test they added immediately caught a missing index that caused a 30-second query in their most critical endpoint.

- **Use transactions for test isolation** — In `beforeEach`, start a transaction. In `afterEach`, roll it back. This is faster than truncating tables between tests and provides full isolation. Trade-off: doesn't test commit behavior (use a separate "commit" test suite). Benefit: each test starts with a clean, known database state.

- **Test migrations against realistic data volumes** — A migration that works with 10 rows of test data may fail with 10M rows (timeout, locking, index rebuild). Create a "performance migration test" with a production-scale anonymized dataset. Trade-off: test data management overhead. Benefit: no migration surprises during deployment.

- **Use fault-injection proxies for resilience testing** — Mocks don't simulate real network failures (connection resets, slow responses, partial data). Toxiproxy lets you inject real network conditions. Trade-off: additional infrastructure in test environment. Benefit: verifies circuit breakers, retry logic, and timeout handling under realistic conditions.

- **Measure query counts in every integration test** — Assert `expect(queries.count).toBeLessThan(5)` for operations that should be efficient. This catches N+1 queries and missing eager loading. Trade-off: query threshold tuning per test. Benefit: performance regressions are caught immediately.

- **Use consumer-driven contracts for service boundaries** — Each consumer publishes its expectations. The provider must pass all consumer contract tests before deploying. Trade-off: Pact/contract setup and maintenance. Benefit: breaking changes are caught in CI, not during production incidents.
