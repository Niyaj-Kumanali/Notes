# Jest

## 1. Executive Summary

Jest is a zero-configuration JavaScript testing framework developed by Meta. It provides a complete testing solution including a test runner, assertion library, mocking utilities, code coverage, and snapshot testing. Jest is the most widely used testing framework in the JavaScript ecosystem, particularly in React applications, and is the default testing framework for Create React App. Its key features include a built-in watch mode, parallel test execution, a powerful mocking system, and deep integration with Babel and TypeScript. Jest's philosophy is "delightful testing" -- tests should be fast, reliable, and provide clear error messages.

## 2. Core Theory

### 2.1 Test Structure

Jest uses three main functions to structure tests:

- **`describe(name, fn)`**: Groups related tests into a suite. Can be nested.
- **`test(name, fn)` or `it(name, fn)`**: An individual test case.
- **`expect(value)`**: Creates an assertion. Combined with a **matcher** like `toBe()` or `toEqual()`.

```javascript
describe('UserService', () => {
  describe('getUser', () => {
    it('returns user by ID', () => {
      const user = UserService.getUser(1);
      expect(user).toBeDefined();
      expect(user.id).toBe(1);
    });

    it('throws for non-existent user', () => {
      expect(() => UserService.getUser(999)).toThrow('User not found');
    });
  });
});
```

### 2.2 Lifecycle Hooks

| Hook | Purpose |
|------|---------|
| `beforeAll(fn, timeout)` | Runs once before all tests in the describe block |
| `afterAll(fn, timeout)` | Runs once after all tests in the describe block |
| `beforeEach(fn, timeout)` | Runs before each test in the describe block |
| `afterEach(fn, timeout)` | Runs after each test in the describe block |

### 2.3 Matchers

Jest matchers are categorized as:

- **Equality**: `toBe()`, `toEqual()`, `toStrictEqual()`
- **Truthiness**: `toBeNull()`, `toBeUndefined()`, `toBeDefined()`, `toBeTruthy()`, `toBeFalsy()`
- **Numbers**: `toBeGreaterThan()`, `toBeLessThan()`, `toBeCloseTo()`
- **Strings**: `toMatch(/regex/)`, `toContain(substring)`
- **Arrays/Iterables**: `toContain()`, `toHaveLength()`, `toContainEqual()`
- **Objects**: `toMatchObject()`, `toHaveProperty()`
- **Errors**: `toThrow()`, `toThrowError()`
- **Mocks**: `toHaveBeenCalled()`, `toHaveBeenCalledWith()`, `toHaveReturned()`
- **Snapshots**: `toMatchSnapshot()`, `toMatchInlineSnapshot()`

## 3. Under-the-Hood Deep Dive

### 3.1 How Jest Discovers Tests

Jest uses `jest-config` to determine the test file pattern. By default, it looks for:

- `__tests__/` directories
- Files with `.test.js`, `.test.jsx`, `.test.ts`, `.test.tsx` suffixes
- Files with `.spec.js`, `.spec.jsx`, `.spec.ts`, `.spec.tsx` suffixes

The search is controlled by `testMatch` and `testRegex` in the configuration:

```javascript
// jest.config.js
module.exports = {
  testMatch: ['**/__tests__/**/*.[jt]s?(x)', '**/?(*.)+(spec|test).[tj]s?(x)'],
  // or
  testRegex: '(/__tests__/.*|(\\.|/)(test|spec))\\.[jt]sx?$',
};
```

### 3.2 Test Runner Architecture

```
 jest CLI
    |
    v
 Runtime / Workers (worker_threads or child_process)
    |
    |--- Worker 1: test file A
    |       |--- globalSetup
    |       |--- setupFiles
    |       |--- testEnvironment setup
    |       |--- setupFilesAfterFramework
    |       |--- Run tests
    |       |       |--- beforeAll (describe scope)
    |       |       |--- beforeEach (describe scope)
    |       |       |--- test
    |       |       |--- afterEach (describe scope)
    |       |       |--- afterAll (describe scope)
    |       |--- testEnvironment teardown
    |       |--- globalTeardown
    |
    |--- Worker 2: test file B (parallel)
    |
    v
 Reporter (collects results)
```

Each worker runs in its own process (`child_process.fork`) by default, providing full isolation. This is configurable via `--maxWorkers`.

### 3.3 Fake Timers

Jest can replace the system clock with fake timers, enabling deterministic testing of time-dependent code:

```javascript
jest.useFakeTimers(); // Replace Date.now, setTimeout, setInterval, etc.

// Advance time
jest.advanceTimersByTime(5000); // Advance 5 seconds
jest.advanceTimersToNextTimer(); // Run the next pending timer
jest.runAllTimers(); // Run all pending timers
jest.runOnlyPendingTimers(); // Run only pending (not cleared) timers

// Restore real timers
jest.useRealTimers();
```

### 3.4 Snapshot Serialization

When you call `toMatchSnapshot()`, Jest:

1. Serializes the value using `pretty-format` (a custom serializer)
2. Compares with the stored snapshot file (e.g., `__snapshots__/test.js.snap`)
3. If no snapshot exists, creates one
4. If the snapshot differs, the test fails. Use `--updateSnapshot` to update.

Snapshots are stored as strings in snapshot files and are committed to version control.

### 3.5 Code Coverage

Jest uses Istanbul (via `nyc`) for code coverage. It instruments the code by adding tracking statements at each branch/function/line. When `--coverage` is passed:

1. Before running tests, Jest instruments all source files
2. As tests run, coverage counters are incremented
3. After all tests, coverage is aggregated and reported

```javascript
// jest.config.js
module.exports = {
  collectCoverage: true,
  collectCoverageFrom: ['src/**/*.{js,jsx,ts,tsx}', '!src/**/*.d.ts'],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

## 4. Production Code Examples

### 4.1 Basic Unit Tests

```javascript
// src/math.js
function add(a, b) { return a + b; }
function subtract(a, b) { return a - b; }
function multiply(a, b) { return a * b; }
function divide(a, b) {
  if (b === 0) throw new Error('Division by zero');
  return a / b;
}

module.exports = { add, subtract, multiply, divide };
```

```javascript
// __tests__/math.test.js
const { add, subtract, multiply, divide } = require('../src/math');

describe('Math utilities', () => {
  test('adds two numbers', () => {
    expect(add(2, 3)).toBe(5);
  });

  test('subtracts two numbers', () => {
    expect(subtract(10, 4)).toBe(6);
  });

  test('multiplies two numbers', () => {
    expect(multiply(3, 4)).toBe(12);
  });

  test('divides two numbers', () => {
    expect(divide(10, 2)).toBe(5);
  });

  test('throws on division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });

  test('handles floating point precision', () => {
    expect(add(0.1, 0.2)).toBeCloseTo(0.3, 5);
  });

  test('handles negative numbers', () => {
    expect(add(-5, 10)).toBe(5);
    expect(subtract(-5, -10)).toBe(5);
  });
});
```

### 4.2 Async Tests

```javascript
// src/userService.js
const api = require('./api');

async function getUser(id) {
  const user = await api.fetchUser(id);
  if (!user) throw new Error('User not found');
  return user;
}

async function getUsers(ids) {
  const results = await Promise.all(ids.map(id => api.fetchUser(id)));
  return results.filter(Boolean);
}

module.exports = { getUser, getUsers };
```

```javascript
// __tests__/userService.test.js
const { getUser, getUsers } = require('../src/userService');
const api = require('../src/api');

jest.mock('../src/api');

describe('UserService', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('getUser returns user when found', async () => {
    api.fetchUser.mockResolvedValue({ id: 1, name: 'Alice' });
    const user = await getUser(1);
    expect(user).toEqual({ id: 1, name: 'Alice' });
    expect(api.fetchUser).toHaveBeenCalledWith(1);
  });

  test('getUser throws when user not found', async () => {
    api.fetchUser.mockResolvedValue(null);
    await expect(getUser(999)).rejects.toThrow('User not found');
  });

  test('getUsers returns only found users', async () => {
    api.fetchUser
      .mockResolvedValueOnce({ id: 1, name: 'Alice' })
      .mockResolvedValueOnce(null)
      .mockResolvedValueOnce({ id: 3, name: 'Bob' });

    const users = await getUsers([1, 2, 3]);
    expect(users).toHaveLength(2);
    expect(users[0].name).toBe('Alice');
    expect(users[1].name).toBe('Bob');
  });

  test('getUsers handles concurrent calls', async () => {
    api.fetchUser.mockResolvedValue({ id: 1, name: 'Alice' });
    const users = await getUsers([1, 2, 3]);
    expect(users).toHaveLength(3);
  });
});
```

### 4.3 Callback Tests

```javascript
// __tests__/callback.test.js
test('callback is called with data', (done) => {
  function fetchData(callback) {
    setTimeout(() => callback('data'), 100);
  }

  fetchData((result) => {
    expect(result).toBe('data');
    done();
  });
});

test('callback is called with error', (done) => {
  function fetchData(callback) {
    setTimeout(() => callback(new Error('fail')), 100);
  }

  fetchData((error) => {
    expect(error).toBeInstanceOf(Error);
    expect(error.message).toBe('fail');
    done();
  });
});
```

### 4.4 Mock Functions

```javascript
// __tests__/mockFunctions.test.js
test('tracks calls and arguments', () => {
  const mockFn = jest.fn((x) => x * 2);

  expect(mockFn(2)).toBe(4);
  expect(mockFn(3)).toBe(6);
  expect(mockFn).toHaveBeenCalledTimes(2);
  expect(mockFn).toHaveBeenCalledWith(2);
  expect(mockFn).toHaveBeenCalledWith(3);
  expect(mockFn).toHaveBeenLastCalledWith(3);
});

test('mock return values', () => {
  const mockFn = jest.fn()
    .mockReturnValueOnce(10)
    .mockReturnValueOnce(20)
    .mockReturnValue(30);

  expect(mockFn()).toBe(10);
  expect(mockFn()).toBe(20);
  expect(mockFn()).toBe(30);
  expect(mockFn()).toBe(30); // Default value
});

test('mock resolved promises', async () => {
  const mockFn = jest.fn()
    .mockResolvedValueOnce('first')
    .mockResolvedValue('default');

  await expect(mockFn()).resolves.toBe('first');
  await expect(mockFn()).resolves.toBe('default');
});

test('mock rejected promises', async () => {
  const mockFn = jest.fn()
    .mockRejectedValue(new Error('fail'));

  await expect(mockFn()).rejects.toThrow('fail');
});

test('mock implementation', () => {
  const mockFn = jest.fn()
    .mockImplementation((a, b) => a + b);

  expect(mockFn(1, 2)).toBe(3);
});

test('mock implementation once', () => {
  const mockFn = jest.fn()
    .mockImplementationOnce(() => 'first')
    .mockImplementationOnce(() => 'second');

  expect(mockFn()).toBe('first');
  expect(mockFn()).toBe('second');
});
```

### 4.5 Module Mocking

```javascript
// __mocks__/fs.js (manual mock)
const path = require('path');
const files = {};

function readFileSync(filePath, encoding) {
  return files[filePath] || '';
}

function writeFileSync(filePath, content) {
  files[filePath] = content;
}

module.exports = { readFileSync, writeFileSync };
```

```javascript
// __tests__/fileService.test.js
jest.mock('fs');

const fs = require('fs');
const { saveFile, readFile } = require('../src/fileService');

test('saveFile writes content', () => {
  saveFile('/tmp/test.txt', 'hello');
  expect(fs.writeFileSync).toHaveBeenCalledWith('/tmp/test.txt', 'hello');
});

test('readFile reads content', () => {
  fs.readFileSync.mockReturnValue('file content');
  const content = readFile('/tmp/test.txt');
  expect(content).toBe('file content');
  expect(fs.readFileSync).toHaveBeenCalledWith('/tmp/test.txt', 'utf-8');
});
```

### 4.6 Snapshot Testing

```javascript
// __tests__/snapshot.test.js
test('toMatchSnapshot creates a snapshot', () => {
  const user = {
    id: 1,
    name: 'Alice',
    email: 'alice@example.com',
    createdAt: new Date('2024-01-01'),
  };
  expect(user).toMatchSnapshot();
});

test('toMatchInlineSnapshot stores inline', () => {
  const config = { theme: 'dark', fontSize: 14 };
  expect(config).toMatchInlineSnapshot(`
    {
      "fontSize": 14,
      "theme": "dark",
    }
  `);
});

test('throws for unexpected shape', () => {
  const data = { type: 'user', attributes: { name: 'Alice' } };
  expect(data).toMatchSnapshot({
    attributes: {
      name: expect.any(String),
    },
  });
});
```

### 4.7 Timer Tests

```javascript
// src/timerService.js
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

function debounce(fn, ms) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), ms);
  };
}

module.exports = { delay, debounce };
```

```javascript
// __tests__/timerService.test.js
const { delay, debounce } = require('../src/timerService');

beforeEach(() => {
  jest.useFakeTimers();
});

afterEach(() => {
  jest.useRealTimers();
});

test('delay resolves after specified time', async () => {
  const promise = delay(5000);

  jest.advanceTimersByTime(5000);
  await expect(promise).resolves.toBeUndefined();
});

test('debounce delays execution', () => {
  const fn = jest.fn();
  const debounced = debounce(fn, 1000);

  debounced();
  debounced();
  debounced();

  expect(fn).not.toHaveBeenCalled();

  jest.advanceTimersByTime(1000);
  expect(fn).toHaveBeenCalledTimes(1);
});
```

### 4.8 Error Testing

```javascript
// __tests__/error.test.js
function validateUser(user) {
  if (!user.name) throw new Error('Name is required');
  if (!user.email) throw new Error('Email is required');
  if (!user.email.includes('@')) throw new Error('Invalid email');
  return { ...user, validated: true };
}

describe('validateUser', () => {
  test('throws if name is missing', () => {
    expect(() => validateUser({ email: 'a@b.com' })).toThrow('Name is required');
  });

  test('throws if email is missing', () => {
    expect(() => validateUser({ name: 'Alice' })).toThrow('Email is required');
  });

  test('throws on invalid email format', () => {
    expect(() => validateUser({ name: 'Alice', email: 'invalid' }))
      .toThrow('Invalid email');
  });

  test('returns validated user on success', () => {
    const result = validateUser({ name: 'Alice', email: 'alice@b.com' });
    expect(result).toEqual({
      name: 'Alice',
      email: 'alice@b.com',
      validated: true,
    });
  });
});
```

### 4.9 Custom Matchers

```javascript
// setup/customMatchers.js
expect.extend({
  toBeWithinRange(received, floor, ceiling) {
    const pass = received >= floor && received <= ceiling;
    return {
      pass,
      message: () =>
        `expected ${received} to be within range ${floor} - ${ceiling}`,
    };
  },
  toBeValidDate(received) {
    const pass = received instanceof Date && !isNaN(received.getTime());
    return {
      pass,
      message: () => `expected ${received} to be a valid date`,
    };
  },
});
```

```javascript
// __tests__/customMatchers.test.js
require('../setup/customMatchers');

test('custom matcher toBeWithinRange', () => {
  expect(50).toBeWithinRange(0, 100);
  expect(101).not.toBeWithinRange(0, 100);
});

test('custom matcher toBeValidDate', () => {
  expect(new Date('2024-01-01')).toBeValidDate();
  expect(new Date('invalid')).not.toBeValidDate();
});
```

## 5. Real-World Scenarios

### 5.1 Testing React Components

```javascript
// __tests__/Counter.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import Counter from '../src/components/Counter';

test('renders initial count', () => {
  render(<Counter initialCount={0} />);
  expect(screen.getByText('Count: 0')).toBeInTheDocument();
});

test('increments count on click', () => {
  render(<Counter initialCount={0} />);
  fireEvent.click(screen.getByText('Increment'));
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});

test('decrements count on click', () => {
  render(<Counter initialCount={5} />);
  fireEvent.click(screen.getByText('Decrement'));
  expect(screen.getByText('Count: 4')).toBeInTheDocument();
});

test('does not go below zero', () => {
  render(<Counter initialCount={0} />);
  fireEvent.click(screen.getByText('Decrement'));
  expect(screen.getByText('Count: 0')).toBeInTheDocument();
});
```

### 5.2 Testing Custom Hooks

```javascript
// __tests__/useCounter.test.js
import { renderHook, act } from '@testing-library/react';
import useCounter from '../src/hooks/useCounter';

test('initializes with default value', () => {
  const { result } = renderHook(() => useCounter());
  expect(result.current.count).toBe(0);
});

test('initializes with custom value', () => {
  const { result } = renderHook(() => useCounter(10));
  expect(result.current.count).toBe(10);
});

test('increments count', () => {
  const { result } = renderHook(() => useCounter(0));
  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
});

test('decrements count', () => {
  const { result } = renderHook(() => useCounter(5));
  act(() => result.current.decrement());
  expect(result.current.count).toBe(4);
});

test('resets count', () => {
  const { result } = renderHook(() => useCounter(0));
  act(() => {
    result.current.increment();
    result.current.increment();
  });
  expect(result.current.count).toBe(2);
  act(() => result.current.reset());
  expect(result.current.count).toBe(0);
});
```

### 5.3 Testing Event Emitters

```javascript
// src/eventBus.js
const EventEmitter = require('events');
const bus = new EventEmitter();

module.exports = bus;
```

```javascript
// __tests__/eventBus.test.js
const bus = require('../src/eventBus');

test('emits and receives events', () => {
  const handler = jest.fn();
  bus.on('user:created', handler);
  bus.emit('user:created', { id: 1, name: 'Alice' });
  expect(handler).toHaveBeenCalledWith({ id: 1, name: 'Alice' });
});

test('handles multiple listeners', () => {
  const handler1 = jest.fn();
  const handler2 = jest.fn();
  bus.on('data', handler1);
  bus.on('data', handler2);
  bus.emit('data', 'payload');
  expect(handler1).toHaveBeenCalledWith('payload');
  expect(handler2).toHaveBeenCalledWith('payload');
});

test('removes listeners', () => {
  const handler = jest.fn();
  bus.on('status', handler);
  bus.removeListener('status', handler);
  bus.emit('status', 'ok');
  expect(handler).not.toHaveBeenCalled();
});
```

### 5.4 Testing Express Middleware

```javascript
// __tests__/middleware.test.js
const { validateBody, authenticate } = require('../src/middleware');

function createReq(overrides) {
  return { headers: {}, body: {}, ...overrides };
}
function createRes() {
  const res = {};
  res.status = jest.fn().mockReturnValue(res);
  res.json = jest.fn().mockReturnValue(res);
  return res;
}
function next() { return jest.fn(); }

describe('validateBody', () => {
  const schema = { name: { type: 'string', required: true } };

  test('calls next if body is valid', () => {
    const req = createReq({ body: { name: 'Alice' } });
    const res = createRes();
    const nextFn = next();
    validateBody(schema)(req, res, nextFn);
    expect(nextFn).toHaveBeenCalled();
    expect(res.status).not.toHaveBeenCalled();
  });

  test('returns 422 if body is invalid', () => {
    const req = createReq({ body: {} });
    const res = createRes();
    const nextFn = next();
    validateBody(schema)(req, res, nextFn);
    expect(res.status).toHaveBeenCalledWith(422);
    expect(nextFn).not.toHaveBeenCalled();
  });
});
```

## 6. Performance

### 6.1 Running Tests in Parallel

```bash
# Default: uses number of CPU cores
npx jest --maxWorkers=4

# Run all tests in a single thread (slower but more stable)
npx jest --runInBand

# Run only changed files (watch mode)
npx jest --onlyChanged
```

### 6.2 Test Performance Optimization

```javascript
// jest.config.js
module.exports = {
  // Use 75% of available CPU cores
  maxWorkers: '75%',

  // Cache dependencies
  cacheDirectory: '.jest-cache',

  // Only collect coverage from specific files
  collectCoverageFrom: ['src/**/*.{js,jsx}', '!src/**/*.test.js'],

  // Skip node_modules transformation (much faster)
  transformIgnorePatterns: ['/node_modules/', '\\.pnp\\.[^\\/]+$'],

  // Use watchman for file watching
  watchman: true,
};
```

### 6.3 Benchmarking Tests

```javascript
// __tests__/performance.test.js
function measure(fn, iterations = 1000) {
  const start = process.hrtime.bigint();
  for (let i = 0; i < iterations; i++) fn();
  const end = process.hrtime.bigint();
  return Number(end - start) / 1_000_000 / iterations; // ms per op
}

test('sort function performance', () => {
  const arr = Array.from({ length: 1000 }, () => Math.random());
  const sorted = [...arr].sort((a, b) => a - b);
  const time = measure(() => {
    const copy = [...arr];
    copy.sort((a, b) => a - b);
  });
  console.log(`Sort time: ${time.toFixed(3)}ms`);
  expect(time).toBeLessThan(1); // Should sort 1000 items in < 1ms
});
```

### 6.4 Test Timing and Timeouts

```javascript
// Default timeout: 5000ms
// Increase for slow tests
jest.setTimeout(30000);

test('slow operation completes within timeout', async () => {
  const result = await slowOperation();
  expect(result).toBeDefined();
}, 10000); // Per-test timeout
```

## 7. Security

### 7.1 Testing for Prototype Pollution

```javascript
// __tests__/security.test.js
test('safe object merge prevents prototype pollution', () => {
  const base = { a: 1 };
  const malicious = JSON.parse('{"__proto__":{"admin":true}}');
  const result = safeMerge(base, malicious);
  expect(result.__proto__).not.toHaveProperty('admin');
  expect({}.admin).toBeUndefined();
});

test('prevents constructor injection', () => {
  const input = { 'constructor.prototype.admin': true };
  expect(() => parse(input)).toThrow();
});
```

### 7.2 Testing for ReDoS

```javascript
// __tests__/redos.test.js
test('regex is not vulnerable to ReDoS', () => {
  const regex = /^(\w+)+$/; // Potentially vulnerable
  const malicious = 'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!';
  const start = Date.now();
  regex.test(malicious);
  const duration = Date.now() - start;
  expect(duration).toBeLessThan(100); // Should complete quickly
});
```

### 7.3 Testing API Key Handling

```javascript
// __tests__/apiKey.test.js
test('API key is not logged', () => {
  const logger = jest.fn();
  processRequest({ headers: { 'x-api-key': 'sk-123456' } }, logger);
  expect(logger).toHaveBeenCalled();
  const logArgs = logger.mock.calls[0];
  const logString = JSON.stringify(logArgs);
  expect(logString).not.toContain('sk-123456');
});
```

## 8. Common Mistakes

### 8.1 Not Clearing Mocks Between Tests

```javascript
// BAD: mock state leaks between tests
jest.mock('../src/api');
const api = require('../src/api');

test('first test', () => {
  api.fetch.mockResolvedValue('data');
  // ...
});

test('second test', () => {
  // api.fetch is still returning 'data' from first test
  api.fetch.mockResolvedValue('other');
});

// GOOD: clear mocks in beforeEach
beforeEach(() => {
  jest.clearAllMocks();
});
```

### 8.2 Testing Implementation Instead of Behavior

```javascript
// BAD: testing internal state
test('counter increments', () => {
  const counter = new Counter();
  counter.increment();
  expect(counter.count).toBe(1); // Internal state
});

// GOOD: testing observable behavior
test('counter increments display', () => {
  render(<Counter />);
  fireEvent.click(screen.getByText('+'));
  expect(screen.getByTestId('count')).toHaveTextContent('1');
});
```

### 8.3 Over-Mocking

```javascript
// BAD: mocking everything
jest.mock('../src/database');
jest.mock('../src/cache');
jest.mock('../src/logger');
jest.mock('../src/queue');

// GOOD: only mock what is necessary
jest.mock('../src/database');
```

### 8.4 Not Using `expect.assertions`

```javascript
// BAD: async test can pass without assertion
test('fetches data', async () => {
  try {
    await fetchData();
  } catch (e) {}
  // Test passes even if fetchData never resolves
});

// GOOD: use expect.assertions
test('fetches data', async () => {
  expect.assertions(1);
  try {
    await fetchData();
  } catch (e) {
    expect(e).toBeDefined();
  }
});
```

### 8.5 Using `toBe` for Objects

```javascript
// BAD: toBe uses Object.is, fails for objects
expect({ a: 1 }).toBe({ a: 1 }); // FAILS

// GOOD: use toEqual for deep equality
expect({ a: 1 }).toEqual({ a: 1 }); // PASSES
```

### 8.6 Not Using `done` with Callbacks

```javascript
// BAD: test ends before callback
test('async callback', () => {
  fetchData((data) => {
    expect(data).toBeDefined();
  });
});

// GOOD: use done
test('async callback', (done) => {
  fetchData((data) => {
    expect(data).toBeDefined();
    done();
  });
});
```

## 9. Senior Engineer Perspective

### 9.1 Test Architecture

Senior engineers organize tests to maximize confidence while minimizing maintenance:

```javascript
// Directory structure
src/
  __tests__/
    unit/       // Pure unit tests (no mocks needed)
    integration // Real dependencies
    fixtures/   // Shared test data
    helpers/    // Test utilities
  __mocks__/    // Manual mocks for node_modules
```

### 9.2 Custom Test Environment

```javascript
// test/customEnvironment.js
const NodeEnvironment = require('jest-environment-node').default;

class CustomEnvironment extends NodeEnvironment {
  constructor(config, context) {
    super(config, context);
    this.global.__TEST_DB_URL__ = process.env.TEST_DB_URL;
  }

  async setup() {
    await super.setup();
    // Set up test database
    await this.global.__setupDB__();
  }

  async teardown() {
    await this.global.__teardownDB__();
    await super.teardown();
  }
}

module.exports = CustomEnvironment;
```

### 9.3 Test Reporting and CI Integration

```javascript
// jest.config.js
module.exports = {
  reporters: [
    'default',
    ['jest-junit', { outputDirectory: 'test-results' }],
    ['jest-html-reporter', { outputPath: 'test-report.html' }],
  ],
};
```

### 9.4 Selective Test Execution

```bash
# Run tests related to changed files
npx jest --onlyChanged

# Run tests matching a pattern
npx jest --testNamePattern="integration"

# Run tests in a specific file
npx jest src/__tests__/user.test.js

# Run tests with coverage
npx jest --coverage
```

### 9.5 Testing Best Practices

- **F.I.R.S.T. principles**: Fast, Isolated, Repeatable, Self-validating, Timely
- **Arrange-Act-Assert**: Structure each test with clear sections
- **One assertion concept per test**: Test one behavior per `test` block
- **Avoid logic in tests**: No loops, conditionals, or complex computations
- **Prefer `toEqual` over `toBe` for objects**: `toBe` checks reference identity
- **Use `test.todo` for planned tests**: Document what needs to be tested
- **Use `test.skip` for broken tests**: Don't comment them out

## 10. Interview Questions (20: 10 Easy + 10 Medium)

### Easy

**Q1**: What is Jest?
**A**: Jest is a JavaScript testing framework developed by Meta. It includes a test runner, assertion library, mocking framework, code coverage, and snapshot testing.

**Q2**: What is the difference between `toBe` and `toEqual`?
**A**: `toBe` uses `Object.is` for strict reference equality. `toEqual` performs deep recursive comparison for objects and arrays.

**Q3**: How do you test an async function in Jest?
**A**: Return the promise, use `async/await`, or use the `done` callback. Use `expect.assertions(n)` to verify assertions ran.

**Q4**: What is `beforeEach` used for?
**A**: A lifecycle hook that runs before each test in a describe block. Commonly used to reset state, clear mocks, or set up test data.

**Q5**: How do you mock a module in Jest?
**A**: Use `jest.mock('module-name')` at the top of the test file. Jest automatically replaces all exported functions with mocks.

**Q6**: What is a snapshot test?
**A**: A snapshot test captures the output of a component or function and compares it to a stored reference file. If the output changes, the test fails.

**Q7**: How do you run a single test file?
**A**: Use `npx jest path/to/file.test.js` or `npx jest --testPathPattern="filename"`.

**Q8**: What is `jest.fn()` used for?
**A**: Creates a mock function that records calls, arguments, and return values. It can also be configured to return specific values or implementations.

**Q9**: What does `--coverage` do?
**A**: Generates a code coverage report showing which lines, branches, functions, and statements were executed by the tests.

**Q10**: How do you verify a function throws an error?
**A**: Wrap the call in a function: `expect(() => fn()).toThrow('error')`. For async: `await expect(fn()).rejects.toThrow()`.

### Medium

**Q11**: What is the difference between `jest.mock` and `jest.spyOn`?
**A**: `jest.mock` replaces the entire module with mocks. `jest.spyOn` wraps a specific method, tracking calls while keeping the original implementation (unless `mockImplementation` is called).

**Q12**: How do you test code that uses `setTimeout` without waiting?
**A**: Use `jest.useFakeTimers()` to replace timers, then use `jest.advanceTimersByTime()` or `jest.runAllTimers()` to control time.

**Q13**: How do you reset, restore, or clear mocks?
**A**: `jest.clearAllMocks()` clears call history. `jest.resetAllMocks()` clears history and return values. `jest.restoreAllMocks()` restores original implementations.

**Q14**: What is the `done` callback in Jest tests?
**A**: `done` is a callback passed to test functions for testing asynchronous code with callbacks. Call `done()` when the async work completes.

**Q15**: How do you test a function that uses environment variables?
**A**: Use `process.env.KEY = 'value'` in `beforeEach` and delete it in `afterEach`. Or use a setup file that loads `.env.test`.

**Q16**: What is the purpose of `jest.config.js`?
**A**: It configures Jest's behavior: test file patterns, module name mapping, transform rules, coverage settings, and global setup/teardown.

**Q17**: How do you mock a specific function from a module without mocking the entire module?
**A**: Use `jest.spyOn(module, 'functionName')` which creates a mock that wraps the original function.

**Q18**: What is `jest.unmock` used for?
**A**: It restores the original implementation of a previously mocked module. Used in combination with `jest.dontMock` for selective mocking.

**Q19**: How do you handle tests that depend on the current date?
**A**: Use `jest.useFakeTimers()` with a specific date: `jest.setSystemTime(new Date('2024-01-01'))`. Or mock `Date.now()`.

**Q20**: What is the difference between `mockImplementation` and `mockReturnValue`?
**A**: `mockReturnValue` sets the return value for a mock function. `mockImplementation` replaces the entire implementation with a custom function.

## 11. Advanced Interview Questions (20: 10 Hard + 10 System Design)

### Hard

**Q1**: How does Jest achieve parallel test execution? What are the trade-offs?
**A**: Jest uses `child_process.fork()` to run each test file in a separate worker process. This provides full isolation (global state, module cache) and leverages multi-core CPUs. Trade-offs: (1) memory overhead per worker, (2) no shared state between tests, (3) database/socket resources must be partitioned, (4) startup cost per worker. Jest reuses workers across test files to amortize startup cost. The `--runInBand` flag disables parallelism for debugging.

**Q2**: How do you implement a custom Jest transformer?
**A**: Create a module that exports a `process` function:

```javascript
// transform.js
module.exports = {
  process(src, filename) {
    return `module.exports = ${JSON.stringify(src)};`;
  },
};
```

Configure in `jest.config.js`:

```javascript
transform: {
  '\\.template$': './transform.js',
}
```

**Q3**: How do you test memory leaks in Jest?
**A**: Write a test that repeatedly creates and destroys objects, then forces garbage collection. Use `--expose-gc` Node flag and `global.gc()`. Assert that memory usage stays stable. Example:

```javascript
test('no memory leak', () => {
  const heapUsedBefore = process.memoryUsage().heapUsed;
  for (let i = 0; i < 10000; i++) {
    const obj = new HeavyObject();
    obj.dispose();
  }
  global.gc();
  const heapUsedAfter = process.memoryUsage().heapUsed;
  expect(heapUsedAfter - heapUsedBefore).toBeLessThan(1024 * 100); // < 100KB
});
```

**Q4**: How do you handle flaky tests in a large Jest suite?
**A**: (1) Use `jest.retryTimes(3)` for known flaky tests. (2) Implement a `jest-flaky-reporter` that tracks flaky tests over time. (3) Quarantine: move flaky tests to a separate directory and run them in a non-blocking CI stage. (4) Use `--shard` to split tests and isolate flakiness to specific shards. (5) Add deterministic seeds for random data. (6) Use `wait-for-expect` for async assertions instead of arbitrary timeouts.

**Q5**: How does Jest's module system resolve mocks?
**A**: Jest uses its own module resolver (`jest-resolve`) that intercepts `require`/`import`. When `jest.mock('fs')` is called, Jest replaces the module in its registry. The mock is hoisted to the top of the file (via `babel-jest` transform). Manual mocks in `__mocks__/` directories take priority. Virtual mocks can be created with `jest.mock('module', () => ({}), { virtual: true })`.

**Q6**: How do you implement property-based testing in Jest?
**A**: Use a library like `fast-check` with Jest:

```javascript
import fc from 'fast-check';

test('sort is idempotent', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const sorted = [...arr].sort((a, b) => a - b);
      expect([...sorted].sort((a, b) => a - b)).toEqual(sorted);
    })
  );
});
```

**Q7**: How do you write a Jest plugin or custom reporter?
**A**: Create a class that implements the reporter interface:

```javascript
// MyReporter.js
class MyReporter {
  constructor(globalConfig, options) {
    this._globalConfig = globalConfig;
    this._options = options;
  }
  onTestResult(test, testResult, aggregatedResult) {
    console.log(`Test: ${testResult.testFilePath}`);
    testResult.testResults.forEach(r => {
      console.log(`  ${r.status}: ${r.title}`);
    });
  }
}
module.exports = MyReporter;
```

**Q8**: How do you test code that uses `jest.mock` with ES modules?
**A**: Jest has experimental ESM support. Use `--experimental-vm-modules` flag. For mocking, use `jest.unstable_mockModule`:

```javascript
import { jest } from '@jest/globals';

jest.unstable_mockModule('../src/db', () => ({
  query: jest.fn(),
}));

const { query } = await import('../src/db');
```

**Q9**: How do you implement test fixtures with TypeScript generics?
**A**:

```typescript
interface Factory<T> {
  build(overrides?: Partial<T>): T;
  buildMany(count: number, overrides?: Partial<T>): T[];
}

function createFactory<T>(defaults: T): Factory<T> {
  return {
    build: (overrides) => ({ ...defaults, ...overrides }),
    buildMany: (count, overrides) =>
      Array.from({ length: count }, () => ({ ...defaults, ...overrides })),
  };
}
```

**Q10**: How do you test concurrent code without race conditions in tests?
**A**: (1) Use `jest.useFakeTimers()` to control async execution order. (2) Use a deterministic scheduler: replace `Promise` with a custom implementation that queues microtasks. (3) Use structured concurrency: test each concurrent path independently. (4) Use `await Promise.resolve()` to flush microtasks. (5) For Web Workers, use a mock worker that executes synchronously.

### System Design

**Q11**: Design a testing strategy for a monorepo with 20 packages.
**A**: (1) Use Jest's `projects` configuration with a separate config per package. (2) Share a base Jest config. (3) Use `moduleNameMapper` for cross-package imports. (4) Use `jest-junit` for aggregated reports. (5) Use `--selectProjects` to run only changed packages. (6) Use Jest's `--shard` for distributing tests across CI runners. (7) Use Nx or Turborepo for caching and dependency-aware test execution.

**Q12**: How would you design a test migration from Mocha/Chai to Jest?
**A**: (1) Install Jest and configure Jest config to accept `.test.js` files. (2) Map Chai assertions to Jest matchers using a custom transform (replace `expect(x).to.equal(y)` with `expect(x).toBe(y)`). (3) Replace `describe/it` (already same). (4) Replace `sinon` stubs with `jest.fn()` and `jest.spyOn()`. (5) Migrate incrementally, package by package, using Jest's `projects` config to run both frameworks. (6) Remove Mocha/Chai dependencies after full migration.

**Q13**: Design a test data factory system that works across 50 test files.
**A**: Framework: use `fishery` (a factory library). Each entity has a factory file:

```javascript
// factories/user.js
import { Factory } from 'fishery';
import { faker } from '@faker-js/faker';

export const userFactory = Factory.define(() => ({
  id: faker.number.int(),
  name: faker.person.fullName(),
  email: faker.internet.email(),
  role: 'user',
  createdAt: faker.date.past(),
}));
```

Shared trait support:

```javascript
userFactory.traits({
  admin: { role: 'admin' },
  inactive: { active: false },
});
```

**Q14**: How do you design a test suite for a CI/CD pipeline that must complete in under 5 minutes?
**A**: (1) Parallelize across 10+ CI containers using `--shard`. (2) Run unit tests only on every commit (fast). (3) Run integration tests in parallel with database-per-container. (4) Use test impact analysis: only run tests for changed files. (5) Cache dependencies (node_modules, Jest cache). (6) Use `--onlyChanged` for PR commits. (7) Run slow E2E tests nightly. (8) Use ESBuild/SWC for faster transforms.

**Q15**: Design a test strategy for a real-time collaborative editing application.
**A**: (1) Unit tests for CRDT/OT algorithms with property-based testing (fast-check). (2) Integration tests for WebSocket connections and state sync. (3) Concurrency tests: simulate multiple users editing simultaneously. (4) Conflict resolution tests: verify convergence. (5) Reconnection tests: verify state recovery after disconnect. (6) Performance tests: measure latency and throughput at scale.

**Q16**: How do you design a screenshot comparison test suite for a web application?
**A**: (1) Use Jest with `jest-image-snapshot` custom matcher. (2) Render components in a headless browser (Puppeteer/Playwright). (3) Take screenshots at various viewports. (4) Compare against baseline screenshots using pixel diff. (5) Set a diff threshold (e.g., 0.1% pixel difference allowed). (6) Store baselines in version control. (7) Use `--updateSnapshot` to update baselines. (8) Run visual tests on a separate CI step.

**Q17**: Design a test suite for an internationalization (i18n) library.
**A**: (1) Test key resolution: verify that `t('greeting')` returns the correct translation for each locale. (2) Test interpolation: `t('hello', {name: 'Alice'})`. (3) Test pluralization: `t('items', {count: 0|1|5})`. (4) Test fallback chain: en-US -> en -> default. (5) Test missing keys: should return key name or throw. (6) Test RTL support: verify layout changes. (7) Generate tests from locale files: for each locale, run all tests.

**Q18**: How do you design a mutation testing pipeline with Jest?
**A**: (1) Use `stryker-mutator` with Jest runner. (2) Configure mutation operators (e.g., arithmetic, logical, conditional). (3) Run mutation tests on the changed files in PRs. (4) Set a mutation score threshold (e.g., 80%). (5) Report uncovered mutations as code review comments. (6) Track mutation score over time. (7) Integrate with CI but do not block the build (use as a quality metric).

**Q19**: Design a testing approach for a CLI tool built with Node.js.
**A**: (1) Test argument parsing with various flag combinations. (2) Test help output with `--help`. (3) Test exit codes (0 success, 1 error). (4) Test stdout/stderr output. (5) Test file I/O side effects (create temp files, run CLI, assert output files). (6) Test error handling (invalid arguments, missing files, permission errors). (7) Use `execa` or `child_process.execFileSync` to run the CLI in tests. (8) Use `os.tmpdir()` for temp directories.

**Q20**: How do you design a test harneess for a plugin system?
**A**: (1) Define a plugin interface (TypeScript interface). (2) Create mock plugins for each hook point. (3) Test plugin lifecycle: load, initialize, execute, destroy. (4) Test error handling: a crashing plugin should not crash the host. (5) Test sandboxing: plugin should not access host internals. (6) Test resource limits: plugin should not exceed memory/CPU limits. (7) Test hot-reload: replace a plugin at runtime and verify the new plugin takes effect. (8) Test third-party plugins: run a matrix of known plugin versions against the current host version.

## 12. Expert-Level Interview Questions (10: Architect-Level)

**Q1**: Design a zero-flake testing strategy for a 2000-test Jest suite running in CI.
**A**: (1) Identify flake sources: async timing, database state, random data, network calls, file system. (2) Eliminate: use fake timers, transaction rollback, deterministic seeds, mock all network calls, use in-memory FS. (3) Detect: run each test 3 times in CI; if it fails intermittently, tag it as flaky and quarantine. (4) Report: track flake rate per test over time; alert if a test flakes more than 1% of runs. (5) Prevent: require all new tests to pass 5 consecutive runs before being accepted. (6) Automatically disable tests that flake more than 10 times and notify the owning team.

**Q2**: How would you implement a "test gap analysis" tool that identifies untested code paths?
**A**: (1) Use Jest's `--coverage` to get line/branch coverage. (2) Parse the coverage JSON output. (3) Use the AST of the source code to identify all execution paths (control flow graph). (4) Cross-reference coverage data with paths. (5) Report uncovered paths with code snippets. (6) For each uncovered path, suggest a test case. (7) Integrate as a Jest plugin that runs after coverage collection. (8) Prioritize uncovered paths by risk (public API > internal helper, complex logic > simple getter).

**Q3**: You need to migrate a 5000-test Jest suite from JavaScript to TypeScript. Design the migration strategy.
**A**: (1) Enable `allowJs: true` in tsconfig so JS tests still run. (2) Rename `.js` to `.ts` incrementally, file by file. (3) Add TypeScript types to test files gradually. (4) Use `@ts-check` in JSDoc comments for gradual typing without renaming. (5) Configure Jest with `ts-jest` or `@swc/jest` for TypeScript transformation. (6) Set up a `jest.config.ts` (Jest supports TypeScript config natively). (7) Use `expect-TypeOf` from `vitest` or `jest-extended` for type-level assertions. (8) Run both JS and TS tests in parallel during migration.

**Q4**: Design a system that auto-generates Jest tests from OpenAPI specs.
**A**: (1) Parse the OpenAPI 3.0 spec (JSON/YAML). (2) For each path + method, generate: (a) a success test with valid example data from the spec, (b) a 400 test with invalid data (violating schema constraints), (c) a 401 test (no auth), (d) a 404 test (non-existent resource). (3) Use spec examples for request bodies. (4) Use `faker` to generate random valid data for fields with `example` or `format`. (5) Output Jest test files with supertest calls. (6) Run generated tests against the API server. (7) Track pass/fail rate and report spec compliance.

**Q5**: How do you ensure Jest tests are deterministic across different operating systems and Node versions?
**A**: (1) Pin Node version in `.nvmrc` and `engines` field. (2) Use OS-agnostic path separators (`path.sep` or always use `/`). (3) Avoid filesystem ordering (use `sort()` on directory listings). (4) Use `toMatchSnapshot` with a custom serializer that normalizes OS-specific values. (5) Use Docker containers for CI to ensure identical environments. (6) Mock `os.platform()` and `os.type()` if code branches on OS. (7) Use `jest.useFakeTimers()` for time-dependent code. (8) Test on Windows, macOS, and Linux in CI using matrix builds.

**Q6**: Design a test performance regression detection system for Jest.
**A**: (1) Track per-test execution time in CI using `--reporters=jest-performance-reporter`. (2) Store historical timing data in a database (e.g., SQLite in CI artifact). (3) For each CI run, compare current timing to the rolling average of the last 10 runs. (4) Flag any test that is 2x slower than its historical average. (5) Group by test file and flag files that are 1.5x slower overall. (6) Alert the team with a GitHub comment listing slow tests. (7) Use `--testTimeout` to fail tests that exceed the max expected time. (8) Visualize timing trends in a dashboard.

**Q7**: How would you implement Jest-compatible property-based testing with shrinking?
**A**: (1) Define a `fc.assert` wrapper that uses Jest's `expect`. (2) Implement generators for common types (integer, string, array, object). (3) Implement shrinking: when a failing input is found, try smaller inputs to find the minimal failing case. (4) Integrate with Jest's test lifecycle: `beforeEach`/`afterEach` still work. (5) Support custom generators with `fc.custom`. (6) Report the minimal failing input in the test failure message. (7) Use `seed` for reproducibility. (8) Example: `test('sort property', fc.property(fc.array(fc.int()), (arr) => { ... }))`.

**Q8**: Design a system to measure and enforce test quality beyond code coverage.
**A**: (1) Mutation testing score: use Stryker to measure how many mutants survive. (2) Test-to-code ratio: measure lines of test code vs. source code. (3) Assertion density: average number of assertions per test. (4) Test isolation: measure how many tests fail when run in a different order (`--order=random`). (5) Flake rate: percentage of non-deterministic tests. (6) Time-to-feedback: average time to run the full suite. (7) Combine these into a single "test health score" (0-100). (8) Enforce minimum score in CI; allow temporary exceptions with ADRs.

**Q9**: How do you design a Jest test suite that supports "time travel" debugging?
**A**: (1) Override `Date`, `setTimeout`, `setInterval`, `requestAnimationFrame` with a virtual clock. (2) Record all timer calls in a log. (3) After a test failure, replay the log to recreate the exact sequence of events. (4) Allow stepping forward/backward in virtual time. (5) Integrate with Jest's `--verbose` to print the event log on failure. (6) Support snapshot comparison at different time points. (7) Implement as a custom Jest environment that wraps the real timers.

**Q10**: Design a testing infrastructure for a federated GraphQL gateway that composes schemas from 10+ services.
**A**: (1) Unit tests for each service's resolvers. (2) Integration tests for the gateway: start the gateway with mock services using `@graphql-tools/mock`. (3) Supergraph composition tests: verify the gateway correctly composes all service schemas. (4) Query planning tests: verify that queries are split and routed to the correct services. (5) Error propagation tests: a service error should be wrapped but not crash the gateway. (6) Performance tests: measure query planning and execution time with 10+ services. (7) Contract tests: each service publishes its subgraph schema; CI detects breaking changes. (8) Use `jest` with `graphql-js` for schema-level assertions.

## 13. Debugging & Troubleshooting

### 13.1 Common Errors

**Error: `expect(received).toBe(expected) // Object.is equality`**
- Cause: Using `toBe` for objects or arrays.
- Fix: Use `toEqual` for deep comparison.

**Error: `jest.mock is not defined`**
- Cause: `jest.mock` called outside of test file scope or without Jest environment.
- Fix: Ensure the file has a `.test.js` extension and is run by Jest.

**Error: `Timeout - Async callback was not invoked`**
- Cause: Async test did not complete within the timeout (5s default).
- Fix: Increase timeout with `jest.setTimeout(30000)` or return the promise.

**Error: `Cannot find module`**
- Cause: Module path is incorrect or not transformed.
- Fix: Check `moduleNameMapper` in config. Ensure the module is installed.

**Error: `Received: serializes to the same string`**
- Cause: Snapshot was updated but the test still expects the old value.
- Fix: Run with `--updateSnapshot` to update snapshots.

### 13.2 Debugging Techniques

```javascript
// Log inside a test
test('debug', () => {
  const result = someFunction();
  console.log('Debug:', JSON.stringify(result, null, 2));
  expect(result).toBeDefined();
});

// Use Jest's verbose output
// npx jest --verbose

// Run a single test
// npx jest --testNamePattern="should handle edge case"

// Debug in Node inspector
// npx jest --inspect-brk
// Then open chrome://inspect
```

### 13.3 Debugging Mock Interactions

```javascript
test('debug mock calls', () => {
  const mock = jest.fn();
  mock('arg1', 'arg2');

  console.log('Calls:', mock.mock.calls);
  console.log('Instances:', mock.mock.instances);
  console.log('Results:', mock.mock.results);
  console.log('Contexts:', mock.mock.contexts);
});
```

## 14. Comparison Section

### 14.1 Jest vs. Vitest

| Aspect | Jest | Vitest |
|--------|------|--------|
| Performance | Slower (transforms with Babel) | Faster (ESBuild/SWC native) |
| TypeScript | Requires `ts-jest` or `babel` | Native (ESBuild) |
| ESM support | Experimental | First-class |
| Configuration | File-based (jest.config.js) | Vite config integration |
| Mocking | `jest.mock`, hoisted | `vi.mock`, hoisted |
| HMR | No | Yes (Vite HMR) |
| Resource usage | Higher | Lower |
| Maturity | Very mature (10+ years) | Growing (3+ years) |

### 14.2 Jest vs. Mocha

| Aspect | Jest | Mocha |
|--------|------|-------|
| Setup | Zero config | Requires Chai, Sinon, etc. |
| Assertions | Built-in | External (Chai) |
| Mocking | Built-in | External (Sinon) |
| Coverage | Built-in | External (Istanbul/nyc) |
| Snapshots | Built-in | External |
| Watch mode | Built-in | External (nodemon) |
| Speed | Good | Good |
| Popularity | Very high | High |

### 14.3 `toBe` vs. `toEqual` vs. `toStrictEqual`

| Matcher | Use case | Notes |
|---------|----------|-------|
| `toBe` | Primitives (strings, numbers, booleans) | Uses `Object.is` |
| `toEqual` | Objects, arrays | Deep comparison, ignores undefined |
| `toStrictEqual` | Strict deep equality | Also checks types, undefined, extra keys |

### 14.4 `jest.fn()` vs. `jest.spyOn()`

| Aspect | `jest.fn()` | `jest.spyOn()` |
|--------|-------------|----------------|
| Purpose | Create new mock function | Wrap existing method |
| Original call | Not preserved | Preserved by default |
| Use case | Standalone callbacks | Wrapping module methods |
| Restoration | Not needed | `mockRestore()` needed |

## 15. Revision Notes

### Jest API Reference

```javascript
// Test structure
describe('suite', () => { test('case', () => {}); });
it('case', () => {});
test.each([1, 2, 3])('parametrized %i', (n) => {});
describe.each([['a'], ['b']])('suite %s', (param) => {});

// Lifecycle
beforeAll(() => {}); afterAll(() => {});
beforeEach(() => {}); afterEach(() => {});

// Matchers
expect(value).toBe(primitive);
expect(value).toEqual(object);
expect(value).toStrictEqual(object);
expect(value).toBeTruthy(); expect(value).toBeFalsy();
expect(value).toBeNull(); expect(value).toBeUndefined();
expect(value).toBeDefined();
expect(value).toBeGreaterThan(n);
expect(value).toBeLessThan(n);
expect(value).toBeCloseTo(n, decimals);
expect(value).toMatch(/regex/);
expect(value).toContain(item);
expect(array).toHaveLength(n);
expect(object).toHaveProperty('key', value);
expect(object).toMatchObject(partial);
expect(fn).toThrow(error);
expect(fn).toHaveBeenCalled();
expect(fn).toHaveBeenCalledWith(...args);
expect(fn).toHaveBeenCalledTimes(n);
expect(fn).toHaveReturned();
expect(promise).resolves.toBe(value);
expect(promise).rejects.toThrow(error);

// Mock functions
jest.fn(); jest.fn(implementation);
jest.fn().mockReturnValue(value);
jest.fn().mockReturnValueOnce(value);
jest.fn().mockResolvedValue(value);
jest.fn().mockRejectedValue(error);
jest.fn().mockImplementation(fn);
jest.spyOn(object, 'method');

// Mock modules
jest.mock('module-name');
jest.mock('module-name', () => ({ mockExport: jest.fn() }));
jest.unmock('module-name');
jest.clearAllMocks();
jest.resetAllMocks();
jest.restoreAllMocks();

// Timers
jest.useFakeTimers();
jest.useRealTimers();
jest.advanceTimersByTime(ms);
jest.runAllTimers();
jest.runOnlyPendingTimers();
jest.setSystemTime(date);

// Hooks
jest.retryTimes(3);
jest.setTimeout(30000);

// Only/Skip
test.only('only', () => {});
test.skip('skip', () => {});
test.todo('planned');
```

### Best Practices Checklist

- [ ] Tests follow Arrange-Act-Assert pattern
- [ ] One behavior per test
- [ ] No logic in tests (no if/for/while)
- [ ] Use `toEqual` for objects, `toBe` for primitives
- [ ] Clear mocks in `beforeEach`
- [ ] Test error paths and edge cases
- [ ] Use `expect.assertions` for async callback tests
- [ ] Snapshot files are reviewed in PRs
- [ ] Coverage threshold is enforced
- [ ] Tests are fast (unit < 10ms, integration < 500ms)

## 16. Cheat Sheet

```text
+==============================================================================+
|                           JEST CHEAT SHEET                                    |
+==============================================================================+

+--- TEST STRUCTURE ----------------------------------------------------------+
|                                                                              |
|  describe('Module', () => {                                                 |
|    beforeAll(() => { /* setup */ })                                         |
|    afterAll(() => { /* teardown */ })                                       |
|    beforeEach(() => { /* reset */ })                                        |
|    afterEach(() => { /* cleanup */ })                                       |
|                                                                              |
|    it('does something', () => {                                             |
|      // Arrange                                                              |
|      const input = 'test';                                                  |
|      // Act                                                                 |
|      const result = fn(input);                                              |
|      // Assert                                                              |
|      expect(result).toBe('expected');                                       |
|    });                                                                       |
|  });                                                                         |
|                                                                              |
+--- ASSERTION MATCHERS ------------------------------------------------------+
|                                                                              |
|  .toBe(val)              .toEqual(val)         .toStrictEqual(val)           |
|  .toBeNull()             .toBeUndefined()      .toBeDefined()                |
|  .toBeTruthy()           .toBeFalsy()          .toBeNaN()                    |
|  .toBeGreaterThan(n)     .toBeLessThan(n)      .toBeCloseTo(n, d)            |
|  .toMatch(/re/)          .toContain(item)      .toHaveLength(n)              |
|  .toHaveProperty(k, v)   .toMatchObject(o)     .toThrow(err)                 |
|  .resolves.toBe(v)       .rejects.toThrow(e)                                |
|                                                                              |
+--- MOCKING -----------------------------------------------------------------+
|                                                                              |
|  jest.fn()                            // Create mock function               |
|  jest.fn(() => 'impl')                // With implementation                 |
|  jest.fn().mockReturnValue(v)         // Return value                        |
|  jest.fn().mockResolvedValue(v)       // Resolve promise                     |
|  jest.fn().mockRejectedValue(e)       // Reject promise                      |
|  jest.spyOn(obj, 'method')            // Spy on method                       |
|  jest.mock('module')                  // Mock entire module                  |
|  jest.mock('module', () => ({...}))   // Manual mock factory                 |
|                                                                              |
|  // Assertions on mocks                                                      |
|  expect(fn).toHaveBeenCalled()                                              |
|  expect(fn).toHaveBeenCalledTimes(n)                                        |
|  expect(fn).toHaveBeenCalledWith(a, b)                                      |
|  expect(fn).toHaveBeenLastCalledWith(a)                                     |
|  expect(fn).toHaveReturnedWith(val)                                         |
|                                                                              |
+--- TIMERS ------------------------------------------------------------------+
|                                                                              |
|  jest.useFakeTimers()                 // Replace timers                     |
|  jest.advanceTimersByTime(1000)       // Fast-forward 1s                     |
|  jest.runAllTimers()                  // Run all pending timers              |
|  jest.runOnlyPendingTimers()          // Don't run newly created ones        |
|  jest.setSystemTime(new Date('2024-01-01')) // Set current date              |
|  jest.useRealTimers()                 // Restore real timers                 |
|                                                                              |
+--- CLI COMMANDS ------------------------------------------------------------+
|                                                                              |
|  npx jest                            // Run all tests                        |
|  npx jest --watch                    // Watch mode                            |
|  npx jest --coverage                 // With coverage                        |
|  npx jest --verbose                  // Verbose output                       |
|  npx jest --runInBand                // Serial (no workers)                 |
|  npx jest --maxWorkers=4             // 4 parallel workers                  |
|  npx jest --onlyChanged              // Changed files only                  |
|  npx jest fileName                   // Specific file                        |
|  npx jest -t "pattern"               // Test name pattern                    |
|  npx jest --updateSnapshot           // Update snapshots                     |
|  npx jest --clearCache               // Clear Jest cache                    |
|  npx jest --showConfig               // Show resolved config                |
|                                                                              |
+--- CONFIGURATION -----------------------------------------------------------+
|                                                                              |
|  // jest.config.js                                                           |
|  module.exports = {                                                         |
|    testEnvironment: 'node',                  // node, jsdom, custom          |
|    roots: ['<rootDir>/src'],                 // Source roots                  |
|    testMatch: ['**/__tests__/**/*.test.js'], // Test file pattern             |
|    moduleNameMapper: {                        // Module aliases               |
|      '^@/(.*)$': '<rootDir>/src/$1',                                        |
|    },                                                                        |
|    transform: {                               // Transformers                 |
|      '^.+\\.ts$': 'ts-jest',                                                |
|    },                                                                        |
|    coverageThreshold: {                       // Coverage gates              |
|      global: { lines: 80, branches: 80 },                                   |
|    },                                                                        |
|    setupFilesAfterSetup: ['./jest.setup.js'],  // Setup files               |
|    globalSetup: './globalSetup.js',            // Before all workers         |
|    globalTeardown: './globalTeardown.js',      // After all workers          |
|  };                                                                          |
|                                                                              |
+==============================================================================+
```
