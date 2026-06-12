# Vitest

---

## Overview

- **Definition:** Vitest is a Vite-native testing framework that is API-compatible with Jest but leverages Vite's transform pipeline (ESBuild/SWC) for significantly faster execution.
- **Why It Exists:** Jest is slow for large projects — it uses Babel-based transforms and has startup overhead. Vitest reuses Vite's config, module resolution, and HMR infrastructure, making tests 10-20x faster with native TypeScript and ESM support and no configuration.
- **Key Concepts:** **`vi` object** (replaces Jest's `jest` — `vi.fn`, `vi.mock`, `vi.spyOn`), **pools** (threads/forks/vmThreads for parallel execution), **HMR in tests** (only re-run affected tests on source change), **ESBuild** (default transformer, 10-100x faster than ts-jest), **workspace** (multi-package monorepo support).

---

## Core Concepts

### Test Structure

```typescript
import { describe, it, expect } from 'vitest';
import { add, divide } from '../math';

describe('Math utilities', () => {
  it('adds two numbers', () => {
    expect(add(2, 3)).toBe(5);
  });

  it('throws on division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });

  it('handles floating point', () => {
    expect(add(0.1, 0.2)).toBeCloseTo(0.3, 5);
  });
});
```

### Globals Mode

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
export default defineConfig({
  test: { globals: true }, // Enables describe/it/expect globally
});
```

### Mocking with `vi`

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';

vi.mock('../db'); // Hoisted mock

beforeEach(() => { vi.clearAllMocks(); });

it('returns user when found', async () => {
  vi.mocked(db.query).mockResolvedValue({ rows: [{ id: 1, name: 'Alice' }] });
  const user = await getUser(1);
  expect(user.name).toBe('Alice');
  expect(db.query).toHaveBeenCalledWith(
    'SELECT * FROM users WHERE id = $1', [1]
  );
});
```

### Partial Mocking (Keep Original Exports)

```typescript
vi.mock('../utils', async (importOriginal) => {
  const actual = await importOriginal();
  return { ...actual, sensitiveFunction: vi.fn() };
});
```

### Fake Timers

```typescript
import { vi } from 'vitest';

beforeEach(() => { vi.useFakeTimers(); });
afterEach(() => { vi.useRealTimers(); });

it('resolves after delay', async () => {
  const promise = createDelayedGreeting('Alice', 1000);
  vi.advanceTimersByTime(1000);
  await expect(promise).resolves.toBe('Hello, Alice!');
});

it('handles concurrent timers in order', () => {
  const fn = vi.fn();
  setTimeout(() => fn('first'), 100);
  setTimeout(() => fn('second'), 50);
  vi.advanceTimersByTime(100);
  expect(fn.mock.calls[0][0]).toBe('second');
});
```

### Snapshot Testing

```typescript
it('matches snapshot', () => {
  const user = { id: 1, name: 'Alice', email: 'alice@example.com' };
  expect(user).toMatchSnapshot();
});

it('inline snapshot', () => {
  const result = { status: 'success', code: 200 };
  expect(result).toMatchInlineSnapshot(`
    {
      "code": 200,
      "status": "success",
    }
  `);
});
```

### HTTP Mocking with MSW

```typescript
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.get('https://api.example.com/users/1', () =>
    HttpResponse.json({ id: 1, name: 'Alice' })
  ),
);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterAll(() => server.close());
afterEach(() => server.resetHandlers());

it('fetches a user', async () => {
  const res = await fetch('https://api.example.com/users/1');
  expect(await res.json()).toEqual({ id: 1, name: 'Alice' });
});
```

### Pool Configuration

```typescript
import { defineConfig } from 'vitest/config';
export default defineConfig({
  test: {
    pool: 'threads',     // worker_threads (fastest, less isolation)
    // pool: 'forks',    // child_process (better isolation, slightly slower)
    // pool: 'vmThreads' // vm.Module in workers (ESM focused, experimental)
    poolOptions: {
      threads: { singleThread: false, maxThreads: 8, minThreads: 2 },
    },
  },
});
```

---

## Common Mistakes

- **Forgetting to import from 'vitest'** — Using `describe`, `it`, `expect` without imports (unless `globals: true`). This *looks correct* because if the project uses `globals: true` in config, the test works without imports, and the difference between global and imported usage is invisible until the config changes or the test runs in a different project.
- **Confusing `vi.fn()` and `vi.spyOn()`** — `vi.fn()` creates a new mock; `vi.spyOn()` wraps an existing method. This *looks correct* because both produce mock functions that return configured values, and the distinction only matters when you need the real implementation to run for untested calls.
- **Not awaiting async assertions** — `.resolves`/`.rejects` without `await` exits before promise resolves. This *looks correct* because the assertion does not throw synchronously — it returns a promise that Vitest cannot track without `await`, producing a silent false pass.
- **Sharing mutable state between tests** — State leaks across tests; reset in `beforeEach`. This *looks correct* because shared variables reduce boilerplate and seem efficient, and the resulting flakiness appears random rather than structural.
- **Forgetting `vi.mock` is hoisted** — Imports before `vi.mock` still get the real module. This *looks correct* because the code appears to execute top-to-bottom, and the hoisting behavior is invisible — `vi.mock` looks like any other function call.
- **Not clearing mocks** — `vi.clearAllMocks()` in `beforeEach` to prevent call history leaks. This *looks correct* because each test appears isolated during development, and call history leakage only surfaces as unexplained assertion failures in specific test orderings.
- **Over-mocking** — Mocking DB, cache, logger, and queue in one test hides real integration bugs. This *looks correct* because mocked tests are fast, never fail due to infrastructure, and achieve high coverage numbers, creating the illusion of thorough testing.

---

## Key Design Considerations

- **Speed advantage:** Vitest's ESBuild transform is 10-100x faster than `ts-jest` or `babel-jest`. Cold start and HMR re-runs are dramatically faster than Jest.
- **HMR in watch mode:** Only re-runs tests that depend on the changed source file, using Vite's module graph. Faster than Jest's file-pattern-based re-runs.
- **Workspace for monorepos:** `vitest.workspace.ts` defines per-package config (different environments, setups, coverage thresholds) in one file.
- **Browser mode:** Run tests in real browsers via Playwright for E2E and visual testing, sharing the same Vitest API.
- **Sharding in CI:** `vitest run --shard=1/4` splits tests across CI runners for faster pipelines.
- **Coverage:** Supports `v8` (faster) and `istanbul` providers. Thresholds enforce minimum coverage.
- **`vi.hoisted`:** Run code before all imports in a file (useful for setting up variables for `vi.mock` factories).

---

## Real-World Scenarios

### Scenario 1: Migrating a Large Jest Codebase to Vitest
A 50K-line test suite runs in 12 minutes with Jest. The team wants Vitest's speed but fears migration risk. **Migration:** (1) Install Vitest and add a `vitest.config.ts` that mirrors Jest config. (2) Run both frameworks in parallel — keep Jest CI pipeline, add Vitest as a separate job. (3) Fix incompatibilities: Jest's `jest.mock` → `vi.mock`, `jest.fn` → `vi.fn`. (4) After all tests pass in both, switch CI to Vitest only. Result: test time drops from 12 min to 90 seconds.

### Scenario 2: Monorepo with Mixed Test Environments
A monorepo has 10 packages. Package A needs `jsdom` (React components), Package B needs `node` (API tests), Package C needs browser tests (Playwright). **Fix:** Use Vitest workspace — `vitest.workspace.ts` defines per-package configs with different environments. Vitest's pool system (`threads`/`forks`) ensures isolation. Test commands: `vitest run --project web` runs only web package tests. Result: single `vitest` command tests the entire monorepo with appropriate environments per package.

### Scenario 3: Flaky Tests from Module Mocking
A team migrated to Vitest but tests are flaky — sometimes the mock works, sometimes the real module is imported. Developers spend hours debugging mock resolution. **Fix:** The issue is `vi.mock` hoisting — if the import statement uses a dynamic path or the mock factory is incorrect, the real module leaks through. Standardize on `vi.mock()` with factory functions at the top of the file. Use `vi.hoisted()` for mock variables. Add a lint rule enforcing Vitest mock patterns. Result: mock behavior becomes deterministic.

---

## Scenario-Based Questions

1. **Q: You migrate a 3000-test Jest suite to Vitest. 2700 tests pass, 200 fail with "Cannot find module" errors, and 100 pass inconsistently. How do you systematically resolve this?**
   A: Three categories of issues: (1) Module resolution — Vitest uses Vite's resolver, not Jest's. Fix: align `resolve.alias` in `vite.config.ts` with Jest's `moduleNameMapper`. Use `vi.mock` for manual module stubs. (2) Flaky tests — often from `jest.useFakeTimers` not converting cleanly to `vi.useFakeTimers`. Fix: use `vi.useFakeTimers({ toFake: ['setTimeout', 'clearTimeout'] })`. (3) Environment differences — use `vi.stubEnv` and `vi.unstubAllEnvs` for env-dependent tests. Run both frameworks in parallel for 2 weeks before dropping Jest.

2. **Q: A Vitest test uses `vi.mock('../database')` but the real database is still queried. The mock seems to have no effect. What's going on?**
   A: `vi.mock` is hoisted above imports, but if the import path is dynamic, computed, or uses a variable, hoisting may fail. Fix: (1) Ensure the mock factory returns all exports: `vi.mock('../database', () => ({ query: vi.fn(), connect: vi.fn() }))`. (2) Check that `../database` resolves to the same path the module uses (not a different index file). (3) If the module uses named exports, the mock must return matching named exports. (4) Use `vi.hoisted()` to define mock variables that are accessible before imports.

> **Interview follow-up:** How does `vi.mock` hoisting interact with ES module static imports — if the module you are mocking uses named exports, what must your mock factory return to satisfy the import binding?

3. **Q: Your team uses Vitest with `pool: 'threads'`. Tests are fast but occasionally one test's mocks leak into another test's scope. How do you ensure test isolation?**
   A: Threads (worker_threads) share some module state. Fix: (1) Add `beforeEach(() => { vi.clearAllMocks(); vi.unstubAllEnvs(); })` to every test file. (2) Use `pool: 'forks'` instead of `pool: 'threads'` — forks create separate process spaces with better isolation. Trade-off: forks are 10-20% slower. (3) For critical tests, wrap each test in `describe` with its own `beforeEach`. (4) Never mock at the top level of a test file — mock inside `describe` blocks.

4. **Q: A monorepo with 15 packages has Vitest configured. Package A tests need `jsdom`, Package B needs `node`, Package C needs a custom environment. How do you set this up cleanly?**
   A: Use a `vitest.workspace.ts` file:
   ```typescript
   export default defineWorkspace([
     { test: { name: 'web', environment: 'jsdom', include: ['packages/web/**/*.test.ts'], setupFiles: ['web-setup.ts'] } },
     { test: { name: 'api', environment: 'node', include: ['packages/api/**/*.test.ts'] } },
     { test: { name: 'extension', environment: './custom-env.ts', include: ['packages/extension/**/*.test.ts'] } },
   ]);
   ```
   Each workspace entry can have its own config, setup files, and dependencies. Run all: `vitest run`. Run one: `vitest run --project web`.

> **Interview follow-up:** What happens when two workspace entries share a dependency that behaves differently in `jsdom` vs `node` — how do you isolate the module instances between environments?

5. **Q: A developer uses `vi.fn()` to mock a function but also wants to call the real implementation for specific arguments (e.g., real for valid input, mock for invalid). How?**
   A: Use `vi.fn().mockImplementation((arg) => { if (isValid(arg)) return realImpl(arg); else return mockReturn; })`. Or use `vi.spyOn(module, 'method').mockImplementation(...)` to wrap an existing method. For conditional fallthrough: `const spy = vi.spyOn(module, 'method'); spy.mockImplementation((arg) => { if (errorCase) return 'mock'; return spy.getOriginal()(arg); })`.

6. **Q: You're using Vitest to test a function that reads from `import.meta.env`. In CI, the env variables are set by the deployment pipeline, not the `.env` file. Tests pass locally but fail in CI. How do you handle this?**
   A: Vitest supports `import.meta.env` natively. Use `vi.stubEnv('API_URL', 'https://test-api.example.com')` in `beforeEach` and `vi.unstubAllEnvs()` in `afterEach`. For CI, set a Vitest config: `export default defineConfig({ test: { env: { API_URL: 'https://ci-test-api.example.com' } } })`. Never read from `process.env` directly in the function — always use `import.meta.env` which Vite handles consistently.

7. **Q: A Vitest integration test uses `globalSetup` to start a PostgreSQL container. The setup takes 10 seconds and runs once before all tests. But when a test modifies the database, other tests see the changes. How do you isolate database state?**
   A: Use a transaction-per-test pattern. In `globalSetup`, create the database and run migrations. In `beforeEach`, start a database transaction. In `afterEach`, roll back the transaction. This is fast (no container restart) and provides full isolation. For parallel workers, use a connection pool where each worker gets its own connection. Use a unique schema per worker: `CREATE SCHEMA IF NOT EXISTS test_worker_${workerId}`.

> **Interview follow-up:** How do you handle database migrations when each parallel worker needs its own schema — do you run migrations per worker or once globally before the workers start?

8. **Q: You need to test a Vite plugin that transforms `.mdx` files. How do you write a Vitest test for this without building the entire app?**
   A: Use Vite's build API programmatically:
   ```typescript
   import { build } from 'vite';
   import { mdxPlugin } from '../src/mdx-plugin';
   
   it('transforms MDX to JSX', async () => {
     const result = await build({
       plugins: [mdxPlugin()],
       build: { write: false, rollupOptions: { input: 'test/fixtures/test.mdx' } },
     });
     const output = result.output[0].code;
     expect(output).toContain('import { jsx }');
     expect(output).toContain('export default function MDXContent');
   });
   ```
   Use `build.write: false` to avoid writing to disk. Test the plugin in isolation without loading the full Vite config.

9. **Q: A Vitest workspace has a shared `setupFiles` that configures global mocks (e.g., `vi.stubGlobal('fetch', mockFetch)`). Workspace tests that override `fetch` mocks start failing intermittently. Why?**
   A: Global stubs in setup files are shared across all tests in the workspace. When one test overrides the global stub, it leaks to other tests. Fix: (1) Don't setup global mocks in shared setup files. (2) Use `vi.hoisted` to create mocks per file. (3) Use `vi.stubGlobal` inside `beforeEach` + `vi.unstubAllGlobals` in `afterEach` per test, not in setup files. (4) Or make the setup file create a factory function that tests call explicitly.

10. **Q: Your CI runs Vitest with `--shard=1/4` across 4 runners. Test A passes on runners 1, 2, 3 but fails on runner 4. The test doesn't use any shared state. What's happening?**
    A: Sharding splits test files, not tests within files. If the file has state-dependent tests (test B modifies something that test A reads), running only a subset of the file's tests on one shard may expose ordering issues. Fix: (1) Ensure every test is self-contained — no shared variables between tests in the same file. (2) Use `beforeEach` to reset state, not `beforeAll`. (3) If the problem persists, check that the sharding uses a consistent seed: `--seed=1234`. (4) Debug by running only the failing shard locally: `vitest run --shard=4/4`.

---

## Interview Questions

1. **What is Vitest and how is it different from Jest?**
   A: A Vite-native testing framework API-compatible with Jest. Key differences: uses ESBuild (10-100x faster transforms), supports ESM natively, reuses Vite config, has HMR for tests, and uses `vi` instead of `jest` for mocking.

2. **What is the `vi` object in Vitest?**
   A: The replacement for Jest's global `jest` object. Key methods: `vi.fn()` (mock function), `vi.mock()` (module mock), `vi.spyOn()` (spy), `vi.useFakeTimers()` (fake timers), `vi.stubEnv()` (env vars).

3. **What are Vitest pools and which should you use?**
   A: Pools control test execution: `threads` (worker_threads, fastest, less isolation), `forks` (child_process, better isolation, slightly slower), `vmThreads` (experimental, ESM-focused). Use `threads` for most cases, `forks` if mocks leak between tests.

4. **How does Vitest achieve faster test execution than Jest?**
   A: Vite's ESBuild/SWC transforms are 10-100x faster than Babel. Native ESM support avoids CommonJS conversion. HMR re-runs only affected tests. Module graph optimization reduces file watching overhead.

5. **What is a Vitest workspace and when would you use it?**
   A: `vitest.workspace.ts` configures per-package test settings (different environments, setup files, coverage thresholds) in monorepos. Each workspace entry can have its own `test` config.

6. **How do you mock a module in Vitest?**
   A: `vi.mock('../module', () => ({ exportName: vi.fn() }))`. The mock is hoisted above imports. For partial mocking: `vi.mock('../module', async (importOriginal) => { const mod = await importOriginal(); return { ...mod, mockedFn: vi.fn() }; })`.

7. **What is `vi.hoisted` and when do you need it?**
   A: Runs code before all imports in a file — useful for defining variables used inside `vi.mock` factory functions. Without `vi.hoisted`, the factory can't reference file-level variables (they don't exist yet at hoist time).

8. **How do you set environment variables in Vitest?**
   A: Two ways: (1) Config: `test: { env: { API_URL: 'http://test' } }`. (2) In tests: `vi.stubEnv('API_URL', 'http://test')` with cleanup in `afterEach`: `vi.unstubAllEnvs()`.

9. **What is the difference between `vi.fn()` and `vi.spyOn()`?**
   A: `vi.fn()` creates a new mock function. `vi.spyOn()` wraps an existing method, preserving the original implementation. Use `vi.spyOn` when you need the real behavior but want to track calls or selectively mock.

10. **How do you run Vitest in a CI pipeline?**
    A: `vitest run` (single run without watch). For speed: `--shard=1/N` for parallel runners, `--reporter=verbose` for detailed output, `--coverage` with `v8` provider for speed. Ensure `pool: 'forks'` in CI for better isolation.

---

## Developer Recommendations

- **Use `vi.spyOn` over `vi.mock` when possible** — `vi.mock` replaces the entire module; `vi.spyOn` wraps the real method. Spying catches more real behavior and reduces false positives. Trade-off: `vi.spyOn` doesn't work for all ESM patterns. Benefit: tests exercise real code paths, catching more bugs. A team used `vi.mock` for their entire data access layer and maintained 95% coverage, but every deployment to staging uncovered real SQL errors that the mocked tests never exercised — switching to `vi.spyOn` on the real module caught three critical query bugs in the first week.

- **Set up `vi.clearAllMocks` in `beforeEach`** — Mock state leaks between tests in threaded pools. Always reset: `beforeEach(() => { vi.clearAllMocks(); vi.unstubAllEnvs(); vi.unstubAllGlobals(); })`. Trade-off: boilerplate. Benefit: no order-dependent test failures. In a threaded pool, a team spent two weeks debugging a flaky test that only failed on the third CI run — the root cause was a mock return value from a previous test leaking through worker thread memory, which `vi.clearAllMocks()` would have eliminated.

- **Use `pool: 'forks'` in CI for better isolation** — Threads share process state, which can cause mock leaks. Forks create separate processes. Trade-off: 10-20% slower. Benefit: eliminates an entire category of flaky tests.

- **Leverage Vitest's workspace for monorepos** — Single `vitest` command runs all packages with appropriate environments. Configure per-package: `environment`, `setupFiles`, `coverage.thresholds`. Trade-off: workspace config complexity. Benefit: consistent test experience across all packages.

- **Use `vi.useFakeTimers` for time-dependent code** — Functions that use `setTimeout`, `setInterval`, or `Date.now()` are non-deterministic. Freeze time with `vi.useFakeTimers()` and advance with `vi.advanceTimersByTime()`. Trade-off: fake timers may skip real timing edge cases. Benefit: deterministic, fast tests.

- **Migrate from Jest incrementally, not in a big bang** — Run Vitest and Jest in parallel. Use codemods for automated migration. Fix failures gradually over 2-4 weeks. Trade-off: dual maintenance during migration. Benefit: no massive PR that blocks development.

- **Use `vi.stubEnv` for environment-dependent tests** — Functions that read `import.meta.env` or `process.env` should be tested with different env values. `vi.stubEnv('NODE_ENV', 'production')` in `beforeEach`, restore in `afterEach`. Trade-off: must remember to unstub. Benefit: each test specifies exactly the env it needs.

- **Benchmark before optimizing test speed** — Vitest is already fast. If tests are slow, profile first: `vitest --pool=threads --reporter=verbose`. Common culprits: synchronous filesystem reads, unoptimized database fixtures, large snapshot files. Fix the bottleneck, not the framework.
