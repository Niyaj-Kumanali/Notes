# Jest

---

## Overview

- **Definition:** Jest is a zero-configuration JavaScript testing framework developed by Meta. It includes a test runner, assertion library, mocking utilities, code coverage, and snapshot testing.
- **Why It Exists:** Before Jest, testing in JS required assembling multiple tools (Mocha + Chai + Sinon + Istanbul). Jest provides an all-in-one solution with "delightful testing" as its philosophy — fast, reliable, and clear error messages. It's the most widely used JS testing framework, especially in React ecosystems.
- **Key Concepts:** **describe/it/expect** (test structure), **matchers** (toBe, toEqual, toMatchSnapshot), **jest.fn()** (mock functions), **jest.mock()** (module mocking), **lifecycle hooks** (beforeEach, afterAll), **fake timers** (jest.useFakeTimers), **code coverage** (Istanbul via NYC), **snapshot testing**.

---

## Core Concepts

### Test Structure

```javascript
const { add, divide } = require('../src/math');

describe('Math utilities', () => {
  test('adds two numbers', () => {
    expect(add(2, 3)).toBe(5);
  });

  test('throws on division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });

  test('handles floating point', () => {
    expect(add(0.1, 0.2)).toBeCloseTo(0.3, 5);
  });
});
```

### Lifecycle Hooks

```javascript
beforeAll(() => { /* run once before all tests */ });
afterAll(() => { /* run once after all tests */ });
beforeEach(() => { /* run before each test */ });
afterEach(() => { /* run after each test */ });
```

### Async Tests

```javascript
jest.mock('../src/api');
const api = require('../src/api');

beforeEach(() => { jest.clearAllMocks(); });

test('resolves with data', async () => {
  api.fetchUser.mockResolvedValue({ id: 1, name: 'Alice' });
  const user = await getUser(1);
  expect(user.name).toBe('Alice');
  expect(api.fetchUser).toHaveBeenCalledWith(1);
});

test('rejects with error', async () => {
  api.fetchUser.mockResolvedValue(null);
  await expect(getUser(999)).rejects.toThrow('User not found');
});
```

### Mock Functions

```javascript
test('tracks calls and arguments', () => {
  const mock = jest.fn(x => x * 2);
  expect(mock(2)).toBe(4);
  expect(mock).toHaveBeenCalledTimes(1);
  expect(mock).toHaveBeenCalledWith(2);
});

test('mock return values', () => {
  const mock = jest.fn()
    .mockReturnValueOnce(10)
    .mockReturnValue(30);
  expect(mock()).toBe(10);
  expect(mock()).toBe(30);
});
```

### Fake Timers

```javascript
beforeEach(() => { jest.useFakeTimers(); });
afterEach(() => { jest.useRealTimers(); });

test('debounce delays execution', () => {
  const fn = jest.fn();
  const debounced = debounce(fn, 1000);

  debounced();
  debounced();
  expect(fn).not.toHaveBeenCalled();

  jest.advanceTimersByTime(1000);
  expect(fn).toHaveBeenCalledTimes(1);
});
```

### Snapshot Testing

```javascript
test('matches snapshot', () => {
  const user = { id: 1, name: 'Alice', email: 'alice@example.com' };
  expect(user).toMatchSnapshot();
});
```

### Key Matchers

| Category | Matchers |
|----------|----------|
| Equality | `toBe`, `toEqual`, `toStrictEqual` |
| Truthiness | `toBeNull`, `toBeUndefined`, `toBeTruthy`, `toBeFalsy` |
| Numbers | `toBeGreaterThan`, `toBeLessThan`, `toBeCloseTo` |
| Strings | `toMatch(/regex/)`, `toContain` |
| Arrays | `toContain`, `toHaveLength` |
| Objects | `toMatchObject`, `toHaveProperty` |
| Errors | `toThrow` |
| Mocks | `toHaveBeenCalled`, `toHaveBeenCalledWith` |
| Snapshots | `toMatchSnapshot`, `toMatchInlineSnapshot` |

---

## Common Mistakes

- **Using `toBe` for objects** — `toBe` uses `Object.is`, fails for objects. Use `toEqual`.
  - **Why it looks correct:** `toBe` works perfectly for primitives and the distinction between reference and value equality is subtle — a test that passes locally with one object reference may fail in CI where module caching differs.
- **Not clearing mocks between tests** — Mock state leaks: call `jest.clearAllMocks()` in `beforeEach`.
  - **Why it looks correct:** Each test appears to work in isolation during development, and mock state leakage only manifests as hard-to-reproduce failures in specific test orderings.
- **Not awaiting async assertions** — `expect(fn()).resolves.toBe('x')` without `await` exits before promise resolves.
  - **Why it looks correct:** The assertion does not throw synchronously — it returns a promise, and without `await` the test completes before the promise settles, producing a quiet false pass.
- **Missing `done` with callbacks** — Async callback test exits before callback runs.
  - **Why it looks correct:** The test function returns immediately and Jest reports it as passing — the assertion inside the callback either never runs or fires after the test already finished.
- **Not using `expect.assertions`** — Async test with try/catch can pass without reaching the assertion.
  - **Why it looks correct:** The test looks complete with a try/catch block, and the assertion inside the try seems guaranteed to execute unless you consciously consider the catch path silently swallowing failures.
- **Over-mocking** — Mocking everything (DB, cache, logger, queue) means you're testing mocks, not code.
  - **Why it looks correct:** Mocked tests are fast, deterministic, and never fail due to infrastructure issues, creating the illusion of thorough coverage.
- **Testing implementation, not behavior** — Checking internal state instead of observable output.
  - **Why it looks correct:** Internal state is easier to access and assert on than figuring out what observable output the behavior produces, and it feels like a more thorough verification.
- **Shared mutable test data** — Tests become order-dependent and flaky.
  - **Why it looks correct:** Sharing setup reduces boilerplate and seems efficient, and the flakiness appears random rather than structural until a specific ordering consistently breaks the suite.

---

## Key Design Considerations

- **Parallel execution:** Jest uses `child_process.fork()` per test file. `--maxWorkers` controls concurrency. `--runInBand` disables it (for debugging).
- **F.I.R.S.T. principles:** Fast, Isolated, Repeatable, Self-validating, Timely.
- **Arrange-Act-Assert:** Structure each test with three clear sections.
- **Manual mocks:** Use `__mocks__/` directory for sharing common mocks across test files.
- **Custom matchers:** `expect.extend({ toBeWithinRange(received, floor, ceiling) { ... } })`.
- **Module resolution:** `moduleNameMapper` in config for path aliases and asset mocks.
- **Custom environments:** Extend `jest-environment-node` or `jest-environment-jsdom` for per-test setup.
- **`jest.spyOn` vs `jest.mock`:** `spyOn` wraps existing methods (preserving original), `mock` replaces entire module.

---

## Real-World Scenarios

### Scenario 1: Large Monorepo Jest Migration
- A monorepo with 200 packages and 15K tests takes 45 minutes to run Jest. Developers avoid running tests before pushing, leading to CI failures. **Fix:** Configure Jest with `projects` to run tests in parallel per package. Use `--changedSince=main` to only test changed packages and their dependents. Set up Jest's `maxWorkers` to match CI CPU cores. Implement test sharding: `--shard=1/4` splits tests across 4 CI runners. Result: CI test time drops from 45 to 5 minutes.

### Scenario 2: Flaky Snapshot Tests in CI
- A team uses Jest snapshot testing for React components. Tests pass locally but fail in CI. The snapshot diff shows "Expected - 1, Received + 2" for seemingly identical outputs. **Fix:** The issue is non-deterministic data — timestamps, auto-generated IDs, or random values in component output. Use `toMatchSnapshot({ id: expect.any(String), createdAt: expect.any(String) })` to ignore specific fields. Also check that CI and local Node.js versions match. Mock all non-deterministic values in `beforeEach`.

### Scenario 3: Migrating Mocha to Jest
- A legacy project uses Mocha + Chai + Sinon + Istanbul. The team wants Jest for better DX, but has 3000 existing tests. **Migration:** Create a Jest config that uses the same `test` directory. Run both frameworks in parallel initially. Use `jest-codemods` for automated migration of `expect` syntax. Run the full suite with both runners for 2 weeks to catch regressions. Remove Mocha config only after all tests pass identically for 2 consecutive weeks.

## Use Cases

- **Unit testing React components** — testing component rendering, state changes, and user interactions
  - React Testing Library with Jest tests components from the user's perspective. `screen.getByText()`, `fireEvent.click()`, and `waitFor()` for async assertions.
  - **Avoid when:** the component is tightly coupled to the DOM — snapshot testing may be a better fit for complex rendering logic.

- **API mocking in tests** — testing components that depend on API calls without making real network requests
  - `jest.mock()` or `msw` (Mock Service Worker) intercepts HTTP requests. Return controlled responses. Test loading, error, and success states.
  - **Avoid when:** the API contract is unstable — contract tests should catch API changes; API mock tests complement, not replace, integration tests.

- **Snapshot testing for UI consistency** — detecting unintended UI changes in React or Vue components
  - `toMatchSnapshot()` captures rendered output. Snapshot diff on CI shows what changed. Review and update snapshots when changes are intentional.
  - **Avoid when:** snapshots are large or change frequently — 300 snapshot failures from one shared component change defeats the purpose; use targeted assertions.

- **Timer and async testing** — testing code that uses `setTimeout`, `setInterval`, or promises
  - `jest.useFakeTimers()` simulates time passage without waiting real time. `jest.advanceTimersByTime()` fast-forwards for timer-dependent code.
  - **Avoid when:** the async code has real I/O dependencies — use real async/await with `expect.assertions()` for promise-based tests.

- **Code coverage enforcement** — ensuring new code is adequately tested before merging
  - Collect coverage with `--coverage`. Enforce minimum thresholds per file or overall. Branch coverage catches untested conditional paths.
  - **Avoid when:** coverage is gamed — 100% coverage doesn't guarantee quality. Focus on meaningful assertions, not coverage percentages.

---

## Scenario-Based Questions

- **Q: You're debugging a test that passes when run alone but fails when run with all other tests. The test reads from a JSON file on disk. What's likely happening and how do you fix it?**
   - **A:** Shared mutable filesystem state — another test modifies the same JSON file. Fix: (1) Use `beforeEach` to copy a fresh fixture file. (2) Use `jest.mock` to mock the file-reading function instead. (3) Better: use `jest.createMockFromModule` to automatically mock the module. For integration-level tests, use `os.tmpdir()` with unique temp directories per test that are cleaned up in `afterEach`.

- **Q: Your team has 1500 Jest snapshot tests for React components. Every time someone changes a shared component, 300 snapshots break, causing massive PR diffs. Developers start accepting snapshots without reviewing them. How do you fix this?**
   - **A:** Snapshots on shared components create coupling. Fix: (1) Use `toMatchSnapshot({ prop: expect.any(String) })` to ignore non-deterministic fields. (2) For shared components, use inline snapshots or explicit assertions instead of file snapshots. (3) Better: use testing-library queries (`getByText`, `getByRole`) instead of snapshot testing — test behavior, not markup. (4) If you keep snapshot tests, run them in CI but don't make them blocking for shared component changes — trust code review.
   - **Interview follow-up:** If you switch to Testing Library queries over snapshots, how do you prevent the same brittleness from reappearing through over-specific `getByRole` or `getByTestId` assertions?

- **Q: You're using `jest.useFakeTimers()` to test a debounced search input. The test calls the debounced function, advances time, but the function never fires. What's wrong?**
   - **A:** Common mistake: the debounce implementation uses `setTimeout`, but Jest's fake timers need to be configured correctly. Fix: ensure `jest.useFakeTimers()` is called before importing the debounced function (fake timers must be active when the module is loaded). Use `jest.advanceTimersByTime(debounceDelay)` not `jest.runAllTimers()` — `runAllTimers` may fire all pending timers including infinite loops. Also ensure the debounce function is the one using `setTimeout` (some libraries use `requestAnimationFrame` which fake timers don't support).

- **Q: A legacy test suite uses `done` callbacks everywhere. Half the tests time out because `done` is called twice or never called. How do you migrate these tests to modern async/await?**
   - **A:** Systematic migration: (1) Run `jest --detectOpenHandles` to find leaking tests. (2) Replace `done` with `async/await` pattern: `test('name', async () => { const result = await asyncFunction(); expect(result).toBe('x'); })`. (3) For tests that use `done` in `.then/.catch`, replace with `await` and `expect.rejects`. (4) Use `jest-codemods` for automated transformation. (5) After migration, add an ESLint rule banning `done` in new tests.

- **Q: A React component test uses `fireEvent.change(input, { target: { value: 'new' } })` but the component's state doesn't update. The test fails but the feature works in the browser. What's happening?**
   - **A:** Likely the component uses a controlled input and the test doesn't wrap the event in `act()`. Fix: wrap the event in `act()`: `await act(async () => { fireEvent.change(input, { target: { value: 'new' } }); })`. Better: use `@testing-library/user-event` instead of `fireEvent` — it automatically wraps in `act` and simulates more realistic interactions (keypress events, focus/blur). User-event also handles edge cases like clearing input before typing.
   - **Interview follow-up:** How would you write a test that verifies a controlled input correctly handles state updates triggered by asynchronous validation running in parallel with user keystrokes?

- **Q: Your test suite runs in 10 minutes. A colleague adds 1000 parameterized tests using `test.each`, increasing runtime to 30 minutes. How do you optimize?**
   - **A:** Parameterized tests are useful but can explode test count. Fix: (1) Use `test.each` with `describe.each` to group related parameters. (2) Run parameterized tests in parallel within a file — Jest parallelizes at the file level, not the test level. (3) Split parameterized tests into separate files for better parallelization. (4) Use `--maxWorkers=50%` in CI to leverage all CPUs. (5) Profile with `jest --verbose --showSeed` to find slow test cases and reduce parameter count.

- **Q: You need to test a function that uses `crypto.randomUUID()` to generate IDs. Every test run has different IDs, making snapshot tests fail. How do you handle non-deterministic values?**
   - **A:** Mock the non-deterministic function: `jest.spyOn(globalThis.crypto, 'randomUUID').mockReturnValue('fixed-uuid-123')`. For snapshot tests, use property matchers: `expect(result).toMatchSnapshot({ id: expect.any(String) })`. Better: pass a test-specific ID generator via dependency injection — in tests, inject a deterministic generator (counter-based). This makes assertions explicit without snapshot coupling.
   - **Interview follow-up:** What happens to your deterministic ID strategy when the same test runs across multiple parallel workers — do you get ID collisions, and how do you prevent them?

- **Q: You're testing a Node.js server that uses `process.env` for configuration. Different tests need different environment variables, but they share the same process. Tests start failing when run together. How do you isolate env-dependent tests?**
   - **A:** Never modify `process.env` directly — it leaks between tests. Fix: (1) Use `jest.resetModules()` in `beforeEach` to clear the module cache. (2) Set env vars before importing the module being tested: `beforeEach(() => { process.env.NODE_ENV = 'test'; delete require.cache[require.resolve('../src/config')]; })`. (3) Better: use a config module that reads from a dependency-injected source — in tests, inject a test config object. (4) Use `jest.spyOn` to mock specific config values.

- **Q: A team uses Jest to test a Python-based data pipeline (via child_process). Tests take 5 seconds each because spawning Python is slow. There are 200 tests — total 17 minutes. How do you speed this up?**
   - **A:** The architecture creates coupling between JS tests and Python runtime. Fix: (1) Run Python tests in Python (pytest) and test the JS layers separately. (2) If cross-language testing is essential, split into smoke tests (run once, not per test) and unit tests. (3) Mock the Python process: `jest.spyOn(child_process, 'execFile').mockResolvedValue({ stdout: 'result' })`. (4) Use `beforeAll` to start a Python server process, `afterAll` to stop it, and communicate via HTTP instead of spawning per test.

- **Q: You have a Jest custom environment that sets up jsdom with specific browser polyfills. A new team member adds a test that imports a library incompatible with jsdom (uses `WebSocket`). The whole test file crashes. How do you isolate incompatible tests?**
   - **A:** Jest supports per-file environments via docblock: `/** @jest-environment node */` at the top of the file. Fix: add the docblock to tests that need Node.js APIs. For mixed environments: split into separate test files with different environment pragmas. Configure Jest's `projects` to use different environments per subdirectory. For a monorepo, use a root-level `jest.config.js` with `projects: [ '<rootDir>/packages/*' ]` — each package can have its own environment.

---

## Interview Questions

- **What is Jest and why is it popular?**
   - **A:** A zero-config JS testing framework by Meta. Popular because it includes test runner, assertions, mocking, coverage, and snapshots in one package — no need for Mocha+Chai+Sinon+Istanbul.

- **What is the difference between `toBe` and `toEqual`?**
   - **A:** `toBe` uses `Object.is` (reference equality for objects). `toEqual` deep-compares values. Use `toEqual` for objects and arrays; use `toBe` for primitives.

- **What is the purpose of `jest.mock`?**
   - **A:** Automatically replaces a module's exports with mock implementations. Hoisted to the top of the file by Jest, ensuring the mock is in place before any imports resolve.

- **How do you test async code in Jest?**
   - **A:** Return a promise, use `async/await`, or use the `done` callback. `await expect(fn()).resolves.toBe(value)` for resolved promises. `await expect(fn()).rejects.toThrow()` for rejected promises.

- **What are Jest lifecycle hooks and their order?**
   - **A:** `beforeAll` (once before all), `beforeEach` (before each test), `afterEach` (after each test), `afterAll` (once after all). Execution: beforeAll → beforeEach → test → afterEach → (repeat) → afterAll.

- **What is snapshot testing and when should you use it?**
   - **A:** Captures the output of a test and compares it to a stored snapshot. Good for: UI components, serialization, config files. Bad for: large snapshots that nobody reviews, non-deterministic output.

- **How does Jest run tests in parallel?**
   - **A:** Each test file runs in its own child process (`child_process.fork`). `--maxWorkers` controls concurrency. `--runInBand` runs sequentially (for debugging).

- **What is `jest.spyOn` and how is it different from `jest.mock`?**
   - **A:** `spyOn` wraps an existing method (preserving original), tracks calls, and can mock selectively. `mock` replaces the entire module. Use `spyOn` when you need the real implementation most of the time.

- **What are the F.I.R.S.T. principles of testing?**
   - **A:** Fast (run quickly), Isolated (no shared state), Repeatable (same result every time), Self-validating (pass/fail, no manual checking), Timely (written before or alongside code).

- **How do you debug a failing Jest test?**
   - **A:** (1) `--verbose` for detailed output. (2) `--runInBand` to disable parallel execution. (3) `--detectOpenHandles` for async leaks. (4) `test.only` to isolate a single test. (5) `console.log` or debugger statement with `node --inspect-brk`.

---

## Developer Recommendations

- **Prefer `toEqual` over `toBe` for objects** — `toBe` uses reference equality; two objects with identical content will fail. `toEqual` deep-compares. The trade-off: `toEqual` is slower for large objects. Benefit: tests actually verify the data, not the memory reference.
  - **Production story:** In a production incident, a team compared two identical user objects with `toBe` — the test passed locally (same reference from module cache) but failed in CI (different references from fresh loads), wasting an entire day of debugging before someone noticed the matcher was wrong.

- **Always clear mocks in `beforeEach`** — Mock state leaks between tests if not reset. Use `jest.clearAllMocks()` or `jest.resetAllMocks()` in `beforeEach`. Trade-off: one line of boilerplate per test file. Benefit: tests are truly isolated — no order-dependent failures.

- **Never use `toBe` with floating-point numbers** — Floating-point arithmetic is imprecise (0.1 + 0.2 !== 0.3). Use `toBeCloseTo(expected, precision)`. Trade-off: you must specify precision. Benefit: tests pass even with tiny representation errors.

- **Use `expect.assertions` for async tests** — An async test with try/catch can pass without running any assertion if the promise rejects unexpectedly. `expect.assertions(1)` ensures at least one assertion ran. Trade-off: you must count expected assertions. Benefit: false positives are eliminated.
  - **Production story:** A team had an async test with try/catch that silently passed for six months because the API endpoint returned an unexpected 500 error, the catch block logged nothing, and no assertion ever ran — the false sense of security delayed discovery of a critical regression by two release cycles.

- **Use `jest --changedSince=main` in CI** — Running all tests on every commit is wasteful. `--changedSince` runs only tests related to changed files. Trade-off: may miss integration failures across unchanged files. Benefit: CI time drops from minutes to seconds for most commits.

- **Prefer Testing Library queries over snapshot tests for components** — `getByText`, `getByRole` test behavior (what the user sees and interacts with), not implementation (the exact DOM structure). Snapshots break on every refactoring. Trade-off: more verbose assertions. Benefit: tests survive refactoring and catch real regressions.

- **Write tests in arrange-act-assert (AAA) format** — Three clear sections separated by blank lines. Arrange: set up test data. Act: invoke the function. Assert: verify the output. Trade-off: more lines per test. Benefit: tests are readable, maintainable, and self-documenting.

- **Use `jest.spyOn` over `jest.mock` when possible** — `spyOn` wraps the real implementation; tests can selectively mock. `jest.mock` replaces the entire module, often hiding real behavior. Trade-off: `spyOn` doesn't work for all scenarios (ESM modules). Benefit: tests exercise more real code, catching more bugs.
