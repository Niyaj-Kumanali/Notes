# Vitest

## 1. Executive Summary

Vitest is a blazing-fast unit testing framework powered by Vite. It is designed as a drop-in replacement for Jest, sharing the same API (`describe`, `it`, `expect`, `jest.mock`-equivalent) while leveraging Vite's transform pipeline and HMR infrastructure. Vitest runs tests in parallel using Worker threads, provides native TypeScript and ESM support without configuration, and includes built-in coverage via c8 or Istanbul. Its key advantage over Jest is speed -- by reusing Vite's transform pipeline with ESBuild/SWC, Vitest can run tests 10-20x faster, especially in large projects. Vitest is the recommended test framework for Vite-based projects (Vue, Svelte, Solid, and increasingly React).

## 2. Core Theory

### 2.1 Test Structure

Vitest uses the same API as Jest, making migration trivial:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';

describe('Calculator', () => {
  it('adds two numbers', () => {
    expect(add(2, 3)).toBe(5);
  });

  it('subtracts two numbers', () => {
    expect(subtract(10, 4)).toBe(6);
  });
});
```

### 2.2 Global vs. Explicit Imports

Vitest supports both Jest-style globals (no import needed) and explicit imports:

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true, // Enables Jest-like global `describe`, `it`, `expect`
  },
});
```

Without globals, import explicitly:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
```

### 2.3 Key Differences from Jest

| Aspect | Jest | Vitest |
|--------|------|--------|
| Test runner | `jest` CLI | `vitest` CLI |
| Mock function | `jest.fn()` | `vi.fn()` |
| Mock module | `jest.mock()` | `vi.mock()` |
| Spy on method | `jest.spyOn()` | `vi.spyOn()` |
| Fake timers | `jest.useFakeTimers()` | `vi.useFakeTimers()` |
| Coverage | Istanbul (built-in) | c8 or Istanbul |
| Transform | Babel/SWC | ESBuild (default) or SWC |
| Config | jest.config.js | vitest.config.ts (or vite.config.ts) |

Vitest's `vi` object replaces Jest's `jest` object. The API is intentionally parallel.

## 3. Under-the-Hood Deep Dive

### 3.1 How Vitest Uses Vite

```
 vitest CLI
    |
    v
 createServer (Vite dev server)
    |
    |--- resolveConfig (Vite + Vitest configs merged)
    |--- createFilter (include/exclude patterns)
    |--- pluginContainer (Vite plugins available in tests)
    |
    v
 runTests
    |
    |--- Pool (threads, forks, or vmThreads)
    |       |--- Each worker:
    |               |--- createViteServer (lightweight)
    |               |--- transform files via ESBuild
    |               |--- execute test file
    |               |--- collect results
    |
    v
 Reporter
```

Vitest starts a Vite dev server internally, which handles module resolution, TypeScript stripping, and path aliases using the same `vite.config.ts`. This means your test environment is identical to your dev environment -- no more mismatched module resolution between tests and runtime.

### 3.2 Pool: Threads vs. Forks vs. VM Threads

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    pool: 'threads', // default - uses worker_threads (fastest, but no process isolation)
    // pool: 'forks',   // uses child_process (better isolation, slightly slower)
    // pool: 'vmThreads', // uses vm.Module in workers (experimental, ESM-focused)
    poolOptions: {
      threads: {
        singleThread: true, // Equivalent to Jest's --runInBand
        useAtomics: true,   // Use Atomics for synchronization
      },
      forks: {
        singleFork: true,
      },
    },
  },
});
```

**Threads (default)**: Uses Node.js `worker_threads`. Fastest because workers share the same process. Better for CPU-bound tests.

**Forks**: Uses `child_process.fork()`. Each worker is a separate process, providing full isolation. Better for tests that modify global state.

**VM Threads**: Experimental. Uses `vm.Module` to run code in a V8 sandbox. Fastest for ESM projects.

### 3.3 How `vi.mock` Works

`vi.mock` is hoisted to the top of the file (like `jest.mock`). Vitest rewrites the AST during transformation to hoist the call. The mock factory receives the original module for partial mocking:

```typescript
import { vi } from 'vitest';

// Full mock
vi.mock('../src/db', () => ({
  query: vi.fn(),
  connect: vi.fn(),
}));

// Partial mock (keep some original exports)
vi.mock('../src/utils', async (importOriginal) => {
  const actual = await importOriginal();
  return {
    ...actual,
    sensitiveFunction: vi.fn(),
  };
});
```

### 3.4 HMR in Tests

Vitest supports Hot Module Replacement during watch mode. When a source file changes, only the tests that depend on that file are re-run. This is powered by Vite's HMR graph, which tracks import dependencies.

```bash
npx vitest --watch  # Enter watch mode with HMR
```

### 3.5 TypeScript and ESM Handling

Vitest processes `.ts` files natively via ESBuild. No additional configuration needed for TypeScript, JSX, or ESM. ESBuild is 10-100x faster than `ts-jest` or `@babel/preset-typescript`.

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    // ESBuild is the default transformer
    // No need for ts-jest or babel-jest
  },
});
```

For custom transforms, you can add Vite plugins:

```typescript
import { defineConfig } from 'vitest/config';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
  plugins: [vue()], // Vue SFC support in tests
  test: {
    environment: 'jsdom',
  },
});
```

## 4. Production Code Examples

### 4.1 Basic Tests

```typescript
// src/math.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function divide(a: number, b: number): number {
  if (b === 0) throw new Error('Division by zero');
  return a / b;
}
```

```typescript
// src/__tests__/math.test.ts
import { describe, it, expect } from 'vitest';
import { add, divide } from '../math';

describe('add', () => {
  it('adds positive numbers', () => {
    expect(add(2, 3)).toBe(5);
  });

  it('adds negative numbers', () => {
    expect(add(-2, -3)).toBe(-5);
  });

  it('adds zero', () => {
    expect(add(0, 5)).toBe(5);
    expect(add(5, 0)).toBe(5);
  });
});

describe('divide', () => {
  it('divides two numbers', () => {
    expect(divide(10, 2)).toBe(5);
  });

  it('throws on division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });
});
```

### 4.2 Async Tests

```typescript
// src/__tests__/async.test.ts
import { describe, it, expect } from 'vitest';

function fetchData(): Promise<string> {
  return new Promise((resolve) => {
    setTimeout(() => resolve('data'), 100);
  });
}

function fetchWithError(): Promise<never> {
  return Promise.reject(new Error('Network error'));
}

describe('async tests', () => {
  it('resolves with data', async () => {
    const data = await fetchData();
    expect(data).toBe('data');
  });

  it('rejects with error', async () => {
    await expect(fetchWithError()).rejects.toThrow('Network error');
  });

  it('resolves using resolves matcher', async () => {
    await expect(fetchData()).resolves.toBe('data');
  });
});
```

### 4.3 Mocking with `vi`

```typescript
// src/userService.ts
import { db } from './db';

export async function getUser(id: number) {
  const user = await db.query('SELECT * FROM users WHERE id = $1', [id]);
  return user.rows[0];
}
```

```typescript
// src/__tests__/userService.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { getUser } from '../userService';
import { db } from '../db';

vi.mock('../db');

beforeEach(() => {
  vi.clearAllMocks();
});

describe('getUser', () => {
  it('returns user when found', async () => {
    const mockUser = { id: 1, name: 'Alice', email: 'alice@test.com' };
    vi.mocked(db.query).mockResolvedValue({ rows: [mockUser] });

    const user = await getUser(1);

    expect(user).toEqual(mockUser);
    expect(db.query).toHaveBeenCalledWith(
      'SELECT * FROM users WHERE id = $1',
      [1]
    );
  });

  it('returns undefined when not found', async () => {
    vi.mocked(db.query).mockResolvedValue({ rows: [] });

    const user = await getUser(999);

    expect(user).toBeUndefined();
  });

  it('propagates database errors', async () => {
    vi.mocked(db.query).mockRejectedValue(new Error('Connection refused'));

    await expect(getUser(1)).rejects.toThrow('Connection refused');
  });
});
```

### 4.4 Snapshot Testing

```typescript
// src/__tests__/snapshot.test.ts
import { describe, it, expect } from 'vitest';

function renderUser(user: { id: number; name: string; email: string }) {
  return {
    ...user,
    isAdmin: user.email.includes('admin'),
    gravatarUrl: `https://gravatar.com/${user.email}`,
  };
}

describe('snapshot tests', () => {
  it('matches snapshot', () => {
    const user = renderUser({
      id: 1,
      name: 'Alice',
      email: 'alice@example.com',
    });
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
});
```

### 4.5 Fake Timers

```typescript
// src/__tests__/timers.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

function createDelayedGreeting(name: string, delayMs: number): Promise<string> {
  return new Promise((resolve) => {
    setTimeout(() => resolve(`Hello, ${name}!`), delayMs);
  });
}

beforeEach(() => {
  vi.useFakeTimers();
});

afterEach(() => {
  vi.useRealTimers();
});

describe('timers', () => {
  it('resolves after delay', async () => {
    const promise = createDelayedGreeting('Alice', 1000);

    vi.advanceTimersByTime(1000);

    await expect(promise).resolves.toBe('Hello, Alice!');
  });

  it('does not resolve before delay', () => {
    const promise = createDelayedGreeting('Bob', 5000);

    vi.advanceTimersByTime(4999);

    // The promise should NOT have resolved yet
    // We can check by seeing if it has resolved
    const marker = vi.fn();
    promise.then(marker);
    expect(marker).not.toHaveBeenCalled();
  });

  it('handles multiple concurrent timers', () => {
    const fn = vi.fn();

    setTimeout(() => fn('first'), 100);
    setTimeout(() => fn('second'), 200);
    setTimeout(() => fn('third'), 150);

    vi.advanceTimersByTime(200);

    expect(fn).toHaveBeenCalledTimes(3);
    expect(fn.mock.calls[0][0]).toBe('first');
    expect(fn.mock.calls[1][0]).toBe('third');
    expect(fn.mock.calls[2][0]).toBe('second');
  });
});
```

### 4.6 Testing Classes

```typescript
// src/cache.ts
export class Cache {
  private store = new Map<string, { value: unknown; expiresAt: number }>();

  set(key: string, value: unknown, ttlMs: number): void {
    this.store.set(key, {
      value,
      expiresAt: Date.now() + ttlMs,
    });
  }

  get<T>(key: string): T | undefined {
    const entry = this.store.get(key);
    if (!entry) return undefined;
    if (Date.now() > entry.expiresAt) {
      this.store.delete(key);
      return undefined;
    }
    return entry.value as T;
  }

  delete(key: string): boolean {
    return this.store.delete(key);
  }

  clear(): void {
    this.store.clear();
  }

  get size(): number {
    return this.store.size;
  }
}
```

```typescript
// src/__tests__/cache.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { Cache } from '../cache';

let cache: Cache;

beforeEach(() => {
  vi.useFakeTimers();
  cache = new Cache();
});

afterEach(() => {
  vi.useRealTimers();
});

describe('Cache', () => {
  it('stores and retrieves values', () => {
    cache.set('key1', 'value1', 1000);
    expect(cache.get('key1')).toBe('value1');
  });

  it('returns undefined for missing keys', () => {
    expect(cache.get('nonexistent')).toBeUndefined();
  });

  it('expires entries after TTL', () => {
    cache.set('temp', 'data', 1000);
    vi.advanceTimersByTime(1001);
    expect(cache.get('temp')).toBeUndefined();
  });

  it('deletes entries', () => {
    cache.set('key', 'val', 5000);
    expect(cache.delete('key')).toBe(true);
    expect(cache.get('key')).toBeUndefined();
    expect(cache.delete('key')).toBe(false);
  });

  it('clears all entries', () => {
    cache.set('a', 1, 1000);
    cache.set('b', 2, 1000);
    expect(cache.size).toBe(2);
    cache.clear();
    expect(cache.size).toBe(0);
  });

  it('supports typed retrieval', () => {
    cache.set('user', { id: 1, name: 'Alice' }, 5000);
    const user = cache.get<{ id: number; name: string }>('user');
    expect(user?.name).toBe('Alice');
  });
});
```

### 4.7 Testing with Environment Variables

```typescript
// src/__tests__/env.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

function getDatabaseUrl(): string {
  return process.env.DATABASE_URL || 'postgres://localhost:5432/default';
}

describe('environment variables', () => {
  const originalEnv = process.env;

  beforeEach(() => {
    vi.resetModules();
    process.env = { ...originalEnv };
  });

  afterEach(() => {
    process.env = originalEnv;
  });

  it('uses DATABASE_URL when set', () => {
    process.env.DATABASE_URL = 'postgres://test:5432/mydb';
    expect(getDatabaseUrl()).toBe('postgres://test:5432/mydb');
  });

  it('falls back to default when not set', () => {
    delete process.env.DATABASE_URL;
    expect(getDatabaseUrl()).toBe('postgres://localhost:5432/default');
  });
});
```

### 4.8 Testing Errors and Exceptions

```typescript
// src/validator.ts
export interface UserInput {
  name?: string;
  email?: string;
  age?: number;
}

export class ValidationError extends Error {
  constructor(public field: string, message: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

export function validateUser(input: UserInput): Required<UserInput> {
  if (!input.name || input.name.trim().length === 0) {
    throw new ValidationError('name', 'Name is required');
  }
  if (!input.email || !input.email.includes('@')) {
    throw new ValidationError('email', 'Valid email is required');
  }
  if (input.age !== undefined && (input.age < 0 || input.age > 150)) {
    throw new ValidationError('age', 'Age must be between 0 and 150');
  }
  return {
    name: input.name.trim(),
    email: input.email.toLowerCase(),
    age: input.age ?? 0,
  };
}
```

```typescript
// src/__tests__/validator.test.ts
import { describe, it, expect } from 'vitest';
import { validateUser, ValidationError } from '../validator';

describe('validateUser', () => {
  it('returns validated user for valid input', () => {
    const result = validateUser({
      name: 'Alice',
      email: 'Alice@Example.COM',
      age: 30,
    });
    expect(result).toEqual({
      name: 'Alice',
      email: 'alice@example.com',
      age: 30,
    });
  });

  it('throws ValidationError for missing name', () => {
    expect(() => validateUser({ email: 'a@b.com' })).toThrow(ValidationError);
    expect(() => validateUser({ email: 'a@b.com' })).toThrow('Name is required');
  });

  it('throws ValidationError for invalid email', () => {
    expect(() => validateUser({ name: 'Bob', email: 'invalid' }))
      .toThrow(ValidationError);
    expect(() => validateUser({ name: 'Bob', email: 'invalid' }))
      .toThrow('Valid email is required');
  });

  it('throws for out-of-range age', () => {
    expect(() => validateUser({ name: 'A', email: 'a@b.com', age: -1 }))
      .toThrow('Age must be between 0 and 150');
    expect(() => validateUser({ name: 'A', email: 'a@b.com', age: 200 }))
      .toThrow('Age must be between 0 and 150');
  });
});
```

## 5. Real-World Scenarios

### 5.1 Testing Vue Components (with Vue Test Utils)

```typescript
// src/components/Counter.vue
<script setup lang="ts">
import { ref } from 'vue';

const props = defineProps<{ initialCount?: number }>();
const count = ref(props.initialCount ?? 0);

function increment() { count.value++; }
function decrement() { count.value--; }
</script>

<template>
  <div>
    <p data-testid="count">Count: {{ count }}</p>
    <button @click="increment">+</button>
    <button @click="decrement">-</button>
  </div>
</template>
```

```typescript
// src/components/__tests__/Counter.test.ts
import { describe, it, expect } from 'vitest';
import { mount } from '@vue/test-utils';
import Counter from '../Counter.vue';

describe('Counter', () => {
  it('renders initial count', () => {
    const wrapper = mount(Counter, { props: { initialCount: 5 } });
    expect(wrapper.find('[data-testid="count"]').text()).toBe('Count: 5');
  });

  it('increments on button click', async () => {
    const wrapper = mount(Counter, { props: { initialCount: 0 } });
    await wrapper.find('button').trigger('click');
    expect(wrapper.find('[data-testid="count"]').text()).toBe('Count: 1');
  });

  it('decrements on second button click', async () => {
    const wrapper = mount(Counter, { props: { initialCount: 5 } });
    const buttons = wrapper.findAll('button');
    await buttons[1].trigger('click');
    expect(wrapper.find('[data-testid="count"]').text()).toBe('Count: 4');
  });
});
```

### 5.2 Testing React Components (with Testing Library)

```typescript
// src/components/Counter.tsx
import { useState } from 'react';

interface Props {
  initialCount?: number;
}

export function Counter({ initialCount = 0 }: Props) {
  const [count, setCount] = useState(initialCount);

  return (
    <div>
      <p data-testid="count">Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <button onClick={() => setCount(c => c - 1)}>-</button>
    </div>
  );
}
```

```typescript
// src/components/__tests__/Counter.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen, fireEvent } from '@testing-library/react';
import { Counter } from '../Counter';

describe('Counter', () => {
  it('renders initial count', () => {
    render(<Counter initialCount={5} />);
    expect(screen.getByTestId('count')).toHaveTextContent('Count: 5');
  });

  it('increments count on click', () => {
    render(<Counter initialCount={0} />);
    fireEvent.click(screen.getByText('+'));
    expect(screen.getByTestId('count')).toHaveTextContent('Count: 1');
  });

  it('decrements count on click', () => {
    render(<Counter initialCount={5} />);
    fireEvent.click(screen.getByText('-'));
    expect(screen.getByTestId('count')).toHaveTextContent('Count: 4');
  });
});
```

### 5.3 Testing HTTP Calls with MSW

```typescript
// src/__tests__/api.test.ts
import { describe, it, expect, beforeAll, afterAll, afterEach } from 'vitest';
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const handlers = [
  http.get('https://api.example.com/users/1', () => {
    return HttpResponse.json({ id: 1, name: 'Alice' });
  }),
  http.post('https://api.example.com/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: 2, ...body as object }, { status: 201 });
  }),
];

const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterAll(() => server.close());
afterEach(() => server.resetHandlers());

describe('API client', () => {
  it('fetches a user', async () => {
    const res = await fetch('https://api.example.com/users/1');
    const data = await res.json();
    expect(data).toEqual({ id: 1, name: 'Alice' });
  });

  it('creates a user', async () => {
    const res = await fetch('https://api.example.com/users', {
      method: 'POST',
      body: JSON.stringify({ name: 'Bob' }),
    });
    expect(res.status).toBe(201);
    const data = await res.json();
    expect(data.name).toBe('Bob');
  });

  it('handles errors', async () => {
    server.use(
      http.get('https://api.example.com/users/999', () => {
        return HttpResponse.json({ error: 'Not found' }, { status: 404 });
      })
    );

    const res = await fetch('https://api.example.com/users/999');
    expect(res.status).toBe(404);
    const data = await res.json();
    expect(data.error).toBe('Not found');
  });
});
```

### 5.4 Testing WebSocket Connections

```typescript
// src/__tests__/websocket.test.ts
import { describe, it, expect, vi, afterAll } from 'vitest';
import { WebSocketServer } from 'ws';
import { io as Client } from 'socket.io-client';

function createServer() {
  return new WebSocketServer({ port: 0 }); // random port
}

describe('WebSocket', () => {
  it('sends and receives messages', async () => {
    const wss = createServer();
    const port = (wss.address() as any).port;

    const messagePromise = new Promise<string>((resolve) => {
      wss.on('connection', (ws) => {
        ws.on('message', (data) => {
          resolve(data.toString());
        });
      });
    });

    const ws = new WebSocket(`ws://localhost:${port}`);
    ws.onopen = () => ws.send('hello');

    const received = await messagePromise;
    expect(received).toBe('hello');

    wss.close();
  });
});
```

### 5.5 Testing File System Operations

```typescript
// src/__tests__/file.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { readFile, writeFile, mkdir, rm } from 'fs/promises';
import { join } from 'path';
import { tmpdir } from 'os';
import { randomUUID } from 'crypto';

let testDir: string;

beforeEach(async () => {
  testDir = join(tmpdir(), `vitest-${randomUUID()}`);
  await mkdir(testDir, { recursive: true });
});

afterEach(async () => {
  await rm(testDir, { recursive: true, force: true });
});

describe('file operations', () => {
  it('writes and reads a file', async () => {
    const filePath = join(testDir, 'test.txt');
    await writeFile(filePath, 'hello vitest');
    const content = await readFile(filePath, 'utf-8');
    expect(content).toBe('hello vitest');
  });

  it('overwrites existing files', async () => {
    const filePath = join(testDir, 'data.json');
    await writeFile(filePath, '{"a": 1}');
    await writeFile(filePath, '{"b": 2}');
    const content = await readFile(filePath, 'utf-8');
    expect(JSON.parse(content)).toEqual({ b: 2 });
  });
});
```

## 6. Performance

### 6.1 Benchmarks (Jest vs. Vitest)

| Project Size | Jest (cold) | Vitest (cold) | Vitest (HMR) |
|-------------|-------------|---------------|--------------|
| 100 tests   | 12s         | 3s            | 0.3s         |
| 500 tests   | 45s         | 8s            | 0.8s         |
| 2000 tests  | 3m 20s      | 35s           | 3s           |

### 6.2 Configuration for Speed

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    // Use threads for parallel execution
    pool: 'threads',
    poolOptions: {
      threads: {
        singleThread: false, // Use all available threads
        maxThreads: 8,
        minThreads: 2,
      },
    },

    // Only transform what's needed
    exclude: ['node_modules', 'dist'],

    // Disable coverage in watch mode
    coverage: {
      enabled: false,
    },

    // Set appropriate timeouts
    testTimeout: 10000,
    hookTimeout: 10000,
  },
});
```

### 6.3 Watch Mode with HMR

```bash
# Run in watch mode with HMR
npx vitest --watch

# Run only changed files
npx vitest --changed

# Run specific test file
npx vitest run src/__tests__/math.test.ts
```

### 6.4 Benchmarking Specific Tests

```typescript
// src/__tests__/benchmark.test.ts
import { describe, it, expect } from 'vitest';

function measure(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

describe('performance', () => {
  it('completes within time limit', async () => {
    const start = performance.now();
    await measure(50);
    const elapsed = performance.now() - start;
    expect(elapsed).toBeGreaterThanOrEqual(40);
    expect(elapsed).toBeLessThan(200);
  });
});
```

## 7. Security

### 7.1 Testing for Injection Vulnerabilities

```typescript
// src/__tests__/security.test.ts
import { describe, it, expect } from 'vitest';

function sanitizeHtml(input: string): string {
  return input
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;');
}

function queryBuilder(table: string, id: number): string {
  const allowedTables = ['users', 'posts', 'comments'];
  if (!allowedTables.includes(table)) {
    throw new Error('Invalid table');
  }
  return `SELECT * FROM ${table} WHERE id = ${id}`;
}

describe('security', () => {
  it('prevents XSS via HTML escaping', () => {
    const malicious = '<script>alert("xss")</script>';
    const sanitized = sanitizeHtml(malicious);
    expect(sanitized).not.toContain('<script>');
    expect(sanitized).toContain('&lt;script&gt;');
  });

  it('prevents SQL injection via table whitelist', () => {
    expect(() => queryBuilder('users; DROP TABLE users--', 1)).toThrow();
  });

  it('handles prototype pollution attempts', () => {
    const malicious = { __proto__: { admin: true } };
    const result = Object.assign({}, malicious);
    expect(result.admin).toBeUndefined();
  });
});
```

### 7.2 Testing Secrets Exposure

```typescript
// src/__tests__/secrets.test.ts
import { describe, it, expect } from 'vitest';

function handleRequest(logger: (msg: string) => void, apiKey: string) {
  // Should not log the API key
  logger(`Processing request at ${new Date().toISOString()}`);
}

describe('secrets', () => {
  it('does not log API keys', () => {
    const logs: string[] = [];
    const logger = (msg: string) => logs.push(msg);

    handleRequest(logger, 'sk-1234567890abcdef');

    logs.forEach((log) => {
      expect(log).not.toMatch(/sk-/);
    });
  });
});
```

## 8. Common Mistakes

### 8.1 Using Jest Globals Without Explicit Import

```typescript
// BAD: assumes vitest globals
describe('test', () => {
  it('works', () => {
    expect(1).toBe(1);
  });
});

// GOOD: import explicitly (or enable globals in config)
import { describe, it, expect } from 'vitest';
```

### 8.2 Confusing `vi.fn()` and `vi.spyOn()`

```typescript
// BAD: vi.fn() on an existing method loses original
const original = { method: () => 'real' };
original.method = vi.fn(); // Now method is gone

// GOOD: use vi.spyOn to wrap
const spy = vi.spyOn(original, 'method');
spy.mockImplementation(() => 'mocked');

// BAD: forgetting to restore spy
// ... test ...
// spy.mockRestore() // MISSING
```

### 8.3 Not Awaiting Async Assertions

```typescript
// BAD: missing await
it('returns data', () => {
  expect(fetchData()).resolves.toBe('data');
  // Test ends before promise resolves
});

// GOOD: await the assertion
it('returns data', async () => {
  await expect(fetchData()).resolves.toBe('data');
});
```

### 8.4 Sharing Mutable State Between Tests

```typescript
// BAD: shared state leaks
const state = { count: 0 };

it('increments', () => {
  state.count++;
  expect(state.count).toBe(1);
});

it('also increments', () => {
  state.count++; // Starts at 1, not 0!
  expect(state.count).toBe(1); // FAILS
});

// GOOD: reset in beforeEach
beforeEach(() => {
  state.count = 0;
});
```

### 8.5 Forgetting `vi.mock` Is Hoisted

```typescript
// BAD: imports before mock
import { myFunction } from '../myModule';
vi.mock('../myModule'); // Hoisted above import, but confusing

// GOOD: mock at top, then import
vi.mock('../myModule');
import { myFunction } from '../myModule';
```

### 8.6 Not Using `vi.clearAllMocks` in `beforeEach`

```typescript
// BAD: mock state carries between tests
const mock = vi.fn();
mock('test1');
expect(mock).toHaveBeenCalledTimes(1);

// In the next test, mock still has the call from the first test
expect(mock).toHaveBeenCalledTimes(1); // Actually 2

// GOOD: clear in beforeEach
beforeEach(() => {
  vi.clearAllMocks();
});
```

## 9. Senior Engineer Perspective

### 9.1 Test Architecture with Vitest

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '@test': resolve(__dirname, 'test'),
    },
  },
  test: {
    globals: true,
    environment: 'node',
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
    exclude: ['node_modules', 'dist', 'e2e'],
    setupFiles: ['./test/setup.ts'],
    globalSetup: ['./test/globalSetup.ts'],
    coverage: {
      provider: 'v8', // or 'istanbul'
      reporter: ['text', 'lcov', 'html'],
      include: ['src/**/*.ts'],
      exclude: ['src/**/*.test.ts', 'src/**/*.d.ts'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
      },
    },
    pool: 'threads',
    poolOptions: {
      threads: {
        singleThread: false,
      },
    },
    testTimeout: 10000,
    retry: 0,
  },
});
```

### 9.2 Workspace Configuration (Monorepo)

```typescript
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config';

export default defineWorkspace([
  {
    test: {
      name: 'packages/core',
      root: './packages/core',
      include: ['src/**/*.test.ts'],
      environment: 'node',
    },
  },
  {
    test: {
      name: 'packages/web',
      root: './packages/web',
      include: ['src/**/*.test.tsx'],
      environment: 'jsdom',
      setupFiles: ['./test/setup.ts'],
    },
  },
  {
    test: {
      name: 'packages/server',
      root: './packages/server',
      include: ['src/**/*.test.ts'],
      environment: 'node',
      globalSetup: ['./test/globalSetup.ts'],
    },
  },
]);
```

### 9.3 Custom Test Environment

```typescript
// test/customEnvironment.ts
import { Environment } from 'vitest/environments';

export default <Environment>{
  name: 'custom',
  async setup() {
    // Set up database, containers, etc.
    process.env.TEST_DATABASE_URL = 'postgres://localhost:5432/test';
    return {
      async teardown() {
        // Clean up
        delete process.env.TEST_DATABASE_URL;
      },
    };
  },
};
```

### 9.4 Retry and Sharding

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    retry: 2, // Retry failed tests up to 2 times
    shard: { index: 1, count: 4 }, // Run 1/4th of tests
  },
});
```

```bash
# CI: split tests across 4 runners
npx vitest run --shard=1/4  # Runner 1
npx vitest run --shard=2/4  # Runner 2
npx vitest run --shard=3/4  # Runner 3
npx vitest run --shard=4/4  # Runner 4
```

### 9.5 Testing Utilities

```typescript
// test/utils.ts
import { vi } from 'vitest';

export function createMockLogger() {
  return {
    info: vi.fn(),
    error: vi.fn(),
    warn: vi.fn(),
    debug: vi.fn(),
  };
}

export function createMockResponse(overrides = {}) {
  return {
    status: vi.fn().mockReturnThis(),
    json: vi.fn().mockReturnThis(),
    send: vi.fn().mockReturnThis(),
    end: vi.fn().mockReturnThis(),
    ...overrides,
  };
}

export function createMockRequest(overrides = {}) {
  return {
    body: {},
    params: {},
    query: {},
    headers: {},
    ...overrides,
  };
}
```

## 10. Interview Questions (20: 10 Easy + 10 Medium)

### Easy

**Q1**: What is Vitest?
**A**: Vitest is a Vite-native unit testing framework that is API-compatible with Jest but uses Vite's transform pipeline (ESBuild) for faster execution.

**Q2**: How does Vitest differ from Jest?
**A**: Vitest uses Vite for module transformation (ESBuild/SWC), has native TypeScript and ESM support, and is significantly faster than Jest.

**Q3**: What is `vi.fn()` used for?
**A**: Creates a mock function (equivalent to `jest.fn()`). It records calls, arguments, and return values.

**Q4**: How do you mock a module in Vitest?
**A**: Use `vi.mock('module-path', factory)` at the top of the test file. The factory returns an object with mock exports.

**Q5**: What is the `vi.spyOn` function?
**A**: Wraps an existing method on an object, allowing you to track calls and optionally change the implementation.

**Q6**: How do you run a single test file with Vitest?
**A**: Use `npx vitest run path/to/file.test.ts` or `npx vitest run --reporter=verbose filename`.

**Q7**: What is the purpose of `vitest.config.ts`?
**A**: It configures Vitest's behavior: test environment, file patterns, coverage settings, aliases, and plugins.

**Q8**: How do you test async code in Vitest?
**A**: Use `async/await` with the test function, or use `.resolves`/`.rejects` matchers.

**Q9**: What does `vi.useFakeTimers()` do?
**A**: Replaces `setTimeout`, `setInterval`, `Date.now`, and other time functions with controllable fake versions.

**Q10**: How do you enable globals in Vitest?
**A**: Set `globals: true` in `vitest.config.ts` under `test`. Or import explicitly: `import { describe, it, expect } from 'vitest'`.

### Medium

**Q11**: What is the difference between Vitest's `pool: 'threads'` and `pool: 'forks'`?
**A**: Threads use `worker_threads` (shared process, faster, less isolation). Forks use `child_process` (separate process, better isolation, slightly slower).

**Q12**: How do you set up Vitest for a React project?
**A**: Install `vitest`, `@testing-library/react`, `jsdom`. Configure `environment: 'jsdom'` in `vitest.config.ts`. Use `@vitejs/plugin-react`.

**Q13**: How do you test code that uses environment variables in Vitest?
**A**: Set `process.env.VAR = 'value'` in `beforeEach` and restore in `afterEach`. Or use `vi.stubEnv('VAR', 'value')`.

**Q14**: What is `vi.stubGlobal` used for?
**A**: Stubs a global variable (like `window`, `globalThis`, `process`) for the duration of a test.

**Q15**: How do you run Vitest in watch mode?
**A**: Use `npx vitest` (without `run`). By default, Vitest starts in watch mode. Use `--watch` explicitly.

**Q16**: What is the Vitest workspace feature?
**A**: Allows running tests across multiple packages or projects with different configurations using a `vitest.workspace.ts` file.

**Q17**: How do you import the original module while using `vi.mock`?
**A**: Use `vi.mock('module', async (importOriginal) => { const actual = await importOriginal(); return { ...actual, mockedFn: vi.fn() }; })`.

**Q18**: What is `vi.hoisted` used for?
**A**: `vi.hoisted()` runs code at the top of the file, before any imports. Useful for setting up variables used in `vi.mock` factories.

**Q19**: How do you generate coverage reports in Vitest?
**A**: Use `--coverage` flag or set `coverage.enabled: true` in config. Supports `v8` (faster) and `istanbul` providers.

**Q20**: How does Vitest handle TypeScript paths aliases?
**A**: Vitest reads the `resolve.alias` from `vite.config.ts` or `vitest.config.ts`. Path aliases work automatically in tests.

## 11. Advanced Interview Questions (20: 10 Hard + 10 System Design)

### Hard

**Q1**: How does Vitest's HMR for tests work differently from Jest's watch mode?
**A**: Vitest uses Vite's HMR graph: when a source file changes, only the dependent test files are re-executed. Jest re-runs all test files that match the changed file's pattern. Vitest's approach is faster because it precisely tracks the dependency graph. Under the hood, Vite's module graph maps each module to its dependents. When a file changes, Vite sends an HMR update to the test runner with the list of affected test modules.

**Q2**: How do you implement a custom Vitest reporter?
**A**: Create a class that implements the Reporter interface:

```typescript
import { Reporter } from 'vitest/reporters';

export default class CustomReporter implements Reporter {
  onInit(ctx: Vitest) {
    console.log('Test suite started');
  }
  onTestRunEnd(testRun: TestRun) {
    const results = testRun.result;
    console.log(`Passed: ${results.passes.length}`);
    console.log(`Failed: ${results.failures.length}`);
  }
  onTestRunTimeout(tasks: TaskCustom[]) {
    console.warn('Test run timed out');
  }
}
```

Register in config: `reporters: ['default', './CustomReporter.ts']`.

**Q3**: How does Vitest handle ES module mocking compared to CommonJS?
**A**: For ESM, Vitest uses Vite's module graph to intercept imports. The mock is injected at the Vite plugin level, replacing module resolution. For CJS, Vitest uses Node.js's `require` cache manipulation (like Jest). Vitest's ESM support is more robust because it operates at the transform level rather than the require cache level. However, ESM mocking requires that mocked modules use static imports (not dynamic `import()`).

**Q4**: How do you implement property-based testing with Vitest and fast-check?
**A**: 

```typescript
import { describe, it, expect } from 'vitest';
import fc from 'fast-check';

describe('sort', () => {
  it('is idempotent', () => {
    fc.assert(
      fc.property(fc.array(fc.integer()), (arr) => {
        const sorted = [...arr].sort((a, b) => a - b);
        const sortedAgain = [...sorted].sort((a, b) => a - b);
        expect(sorted).toEqual(sortedAgain);
      })
    );
  });

  it('preserves length', () => {
    fc.assert(
      fc.property(fc.array(fc.string()), (arr) => {
        const sorted = [...arr].sort();
        expect(sorted).toHaveLength(arr.length);
      })
    );
  });
});
```

**Q5**: How does Vitest achieve test isolation between worker threads?
**A**: Each worker thread runs in its own V8 isolate within the same Node.js process. Module caches are per-worker. However, global state (e.g., `process.env`) is shared. For full isolation, use `pool: 'forks'` which spawns separate processes. The trade-off: threads are faster (shared memory) but forks provide true isolation. Vitest also supports `pool: 'vmThreads'` which uses `vm.Module` for per-module isolation.

**Q6**: How do you test a Vite plugin using Vitest?
**A**: 

```typescript
import { describe, it, expect } from 'vitest';
import { build, resolveConfig } from 'vite';

describe('My Vite plugin', () => {
  it('transforms files correctly', async () => {
    const config = await resolveConfig({
      plugins: [myPlugin()],
      build: { write: false },
    }, 'build');

    const result = await build(config);
    // Assert on build output
    const output = result.output[0];
    expect(output.code).toContain('expected transformation');
  });
});
```

**Q7**: How do you implement a custom Vitest matcher?
**A**: 

```typescript
import { expect } from 'vitest';

function toBeWithinRange(received: number, floor: number, ceiling: number) {
  const pass = received >= floor && received <= ceiling;
  return {
    pass,
    message: () =>
      `expected ${received} to be within range ${floor} - ${ceiling}`,
  };
}

expect.extend({ toBeWithinRange });

// Usage
expect(10).toBeWithinRange(5, 15);
```

**Q8**: How does `vi.mock` handle hoisting differently from Jest?
**A**: Both hoist mock calls to the top of the file. Vitest implements hoisting via a Vite plugin that rewrites the AST during transformation. Jest uses Babel plugin `babel-jest-hoist`. Vitest's implementation is more reliable for ESM and TypeScript files because it operates after ESBuild transforms TypeScript to JavaScript, so it handles all syntax variants uniformly.

**Q9**: How do you test race conditions with Vitest?
**A**: Use controlled concurrency patterns:

```typescript
it('handles concurrent updates', async () => {
  const results: string[] = [];
  const fn = vi.fn().mockImplementation(async (id: string) => {
    results.push(id);
  });

  await Promise.all([
    fn('a'),
    fn('b'),
    fn('c'),
  ]);

  expect(results.sort()).toEqual(['a', 'b', 'c']);
});

it('tests async ordering', async () => {
  vi.useFakeTimers();
  const order: number[] = [];

  setTimeout(() => order.push(1), 100);
  setTimeout(() => order.push(2), 50);
  setTimeout(() => order.push(3), 150);

  vi.advanceTimersByTime(150);
  expect(order).toEqual([2, 1, 3]);
});
```

**Q10**: How do you configure Vitest for a library that needs to run tests in both Node and browser environments?
**A**: Use the workspace feature with two project configurations:

```typescript
// vitest.workspace.ts
export default defineWorkspace([
  { test: { name: 'node', environment: 'node', include: ['test/node/**/*.test.ts'] } },
  { test: { name: 'browser', environment: 'jsdom', include: ['test/browser/**/*.test.ts'] } },
]);
```

Or use Vitest Browser Mode:

```typescript
export default defineConfig({
  test: {
    browser: {
      enabled: true,
      provider: 'playwright',
      instances: [{ browser: 'chromium' }],
    },
  },
});
```

### System Design

**Q11**: Design a testing strategy for a large monorepo with 50 packages using Vitest.
**A**: (1) Use Vitest workspace to define per-package configurations. (2) Share a base config via a shared package. (3) Use `resolve.alias` for cross-package imports. (4) Use `--changed` flag to run only changed packages in CI. (5) Use sharding to distribute tests across CI runners. (6) Use `coverage.thresholds` per package. (7) Use `poolOptions.threads.maxThreads` to limit resource usage. (8) Use `retry` for flaky network-dependent tests.

**Q12**: How would you migrate a 5000-test Jest suite to Vitest?
**A**: (1) Install Vitest and create `vitest.config.ts` with `globals: true` for compatibility. (2) Map `jest.fn` -> `vi.fn`, `jest.mock` -> `vi.mock`. (3) Install `@vitest/expect` for Jest matcher polyfills. (4) Run both frameworks simultaneously using separate CI jobs. (5) Migrate package by package. (6) Fix TypeScript paths (Vite alias vs. Jest moduleNameMapper). (7) Remove Jest config after full migration. (8) Performance benchmark: compare cold start, warm start, and HMR times.

**Q13**: Design a test infrastructure for a real-time multiplayer game server using Vitest.
**A**: (1) Unit tests for game logic (pure functions with property-based testing). (2) Integration tests for WebSocket message handling. (3) Concurrency tests: simulate 100 concurrent players using `Promise.all` with fake timers. (4) State synchronization tests: verify all clients receive the same state after updates. (5) Deterministic lockstep tests: verify all clients reach the same game state given the same inputs. (6) Use `vi.useFakeTimers()` for deterministic tick-based game loop testing. (7) Performance tests: measure update throughput and latency.

**Q14**: How do you design a Vitest-based test suite for a database migration system?
**A**: (1) Test migration up/down functions. (2) Test that migrations are idempotent (running twice produces same schema). (3) Test migration ordering: apply migrations in sequence and verify schema state. (4) Test rollback: apply up, then down, then up again. (5) Test data migration: insert data with old schema, apply migration, verify data is transformed correctly. (6) Use test containers (PostgreSQL) per test file. (7) Use `globalSetup` for container lifecycle. (8) Use `beforeEach` to create a fresh database schema.

**Q15**: Design a testing approach for a distributed task queue system.
**A**: (1) Unit tests for task serialization/deserialization. (2) Integration tests for queue produce/consume with in-memory broker. (3) Tests for task retry logic: simulate failure, verify retry with backoff. (4) Tests for task deduplication: submit same task twice, assert executed once. (5) Tests for priority queue: submit tasks with different priorities, assert higher priority runs first. (6) Tests for worker failure: simulate worker crash, verify task is re-queued. (7) Use `vi.useFakeTimers()` for backoff and timeout testing.

**Q16**: Design a test suite for an internationalization (i18n) library using Vitest.
**A**: (1) Test key resolution with parameterized tests across all locales. (2) Test pluralization rules (one, few, many, other) for each locale. (3) Test interpolation: `t('hello', { name: 'World' })`. (4) Test fallback chains: `en-US -> en -> root`. (5) Test missing key handling: return key name or throw. (6) Test nested key access: `t('nav.menu.home')`. (7) Generate locale coverage reports: percentage of translated keys per locale. (8) Use `describe.each` for cross-locale testing.

**Q17**: Design a Vitest plugin that automatically generates test cases from TypeScript types.
**A**: (1) Parse TypeScript types using the TypeScript compiler API. (2) For each exported function, generate a test file with: (a) a happy path test using the parameter types, (b) edge case tests for nullable/optional fields, (c) boundary tests for numeric ranges. (3) Use `zod` or `io-ts` schemas to generate valid/invalid test data. (4) Output `.test.ts` files alongside the source. (5) Support custom generators via decorators or JSDoc annotations. (6) Run as a Vite plugin that watches for type changes.

**Q18**: How do you design a test suite for a CLI tool built with Node.js using Vitest?
**A**: (1) Test argument parsing: use `process.argv` mocking. (2) Test stdout/stderr output: capture writes using `vi.spyOn(process.stdout, 'write')`. (3) Test exit codes: wrap the CLI entry point and assert on return code. (4) Test file I/O: use temp directories with `fs/promises`. (5) Test interactive prompts: mock `readline` or `inquirer`. (6) Test error output: invalid args, missing files, permissions. (7) Use `execa` to run the built CLI in a subprocess. (8) Set up and tear down temp directories in `beforeEach`/`afterEach`.

**Q19**: Design a test system for a plugin-based architecture where plugins are loaded dynamically.
**A**: (1) Define a plugin interface (TypeScript interface). (2) Create test plugins: valid plugin, plugin that throws on load, plugin that times out. (3) Test plugin lifecycle: load, initialize, execute, destroy. (4) Test isolation: one crashing plugin should not affect others. (5) Test sandboxing: plugin should not access host process internals. (6) Test resource limits: memory, CPU, file descriptors per plugin. (7) Test hot-reload: replace plugin at runtime without restart. (8) Use `vi.mock` to simulate different plugin environments.

**Q20**: Design a Vitest integration for end-to-end testing with Playwright.
**A**: (1) Use `@vitest/web` or Vitest's browser mode with Playwright provider. (2) Define a custom test environment that starts the dev server before tests. (3) Use Playwright's `page` object to navigate and assert. (4) Write tests as:

```typescript
import { test } from 'vitest';
import { expect } from '@playwright/test';

test('homepage loads', async ({ page }) => {
  await page.goto('http://localhost:5173');
  await expect(page.locator('h1')).toHaveText('Welcome');
});

test('form submission', async ({ page }) => {
  await page.goto('http://localhost:5173/login');
  await page.fill('[name="email"]', 'user@test.com');
  await page.fill('[name="password"]', 'password');
  await page.click('button[type="submit"]');
  await expect(page).toHaveURL(/dashboard/);
});
```

(5) Use Vitest's `globalSetup` to start the server and `globalTeardown` to stop it. (6) Run E2E tests in a separate CI job with `pool: 'forks'` for isolation.

## 12. Expert-Level Interview Questions (10: Architect-Level)

**Q1**: Design a Vitest-based test framework for a federated GraphQL gateway that composes schemas from 20 services.
**A**: The framework must: (1) Spin up mock services that serve valid GraphQL schemas (using `@graphql-tools/mock`). (2) Compose the supergraph using `@apollo/gateway` or `@graphql-hive/gateway`. (3) Run queries against the gateway and verify correct routing. (4) Test schema changes: update one service's schema and verify the gateway detects the breaking change. (5) Test error propagation: make a mock service return errors and verify the gateway formats them correctly. (6) Test performance: measure query planning time with all 20 services. Implementation: use Vitest workspace with per-service configs. Use `globalSetup` to start mock services via test containers. Use `vi.mock` to replace real service endpoints with mocks.

**Q2**: How would you implement "time-travel" debugging for Vitest tests?
**A**: (1) Override all asynchronous APIs (timers, promises, events) with a virtual scheduler. (2) Record every async operation in a timeline log (timestamp, type, payload). (3) On test failure, serialize the timeline to a JSON file. (4) Provide a REPL tool that can replay the timeline step by step, allowing the developer to inspect state at each async operation. (5) Integrate with Vitest's error reporting to show the relevant timeline entries near the failure point. (6) Implement as a custom Vitest environment wrapping `vi.useFakeTimers()` with enhanced logging.

**Q3**: You need to ensure that a 10000-test Vitest suite completes in under 2 minutes in CI. Design the system.
**A**: (1) Use `pool: 'threads'` with `maxThreads` set to number of CPU cores (e.g., 16 on a CI runner). (2) Use sharding across 4 CI runners: `--shard=1/4` through `--shard=4/4`. (3) Use test impact analysis: `--changed` to only run tests for changed files. (4) Cache Vitest's transform cache: use a CI cache key based on lockfile hash. (5) Use ESBuild (default) and avoid custom Babel transforms. (6) Disable coverage in fast CI; run coverage separately nightly. (7) Use `poolOptions.threads.useAtomics: true` for faster inter-thread communication. (8) Profile with `--reporter=hanging-process` to find slow tests. (9) Move slow integration tests to a separate CI job. (10) Target: 10000 tests / 4 shards = 2500 tests per shard. At 100 tests/second per shard, that is 25 seconds. Add overhead: ~60 seconds per shard.

**Q4**: Design a mutation testing pipeline using Vitest and Stryker.
**A**: (1) Install `@stryker-mutator/core` with `@stryker-mutator/vitest-runner`. (2) Configure Stryker to mutate only business logic files (exclude types, configs, mocks). (3) Run mutation testing on changed files in PRs. (4) Set a mutation score threshold (e.g., 75% for core logic). (5) Report uncovered mutants as in-line GitHub PR comments. (6) Track mutation score over time in a dashboard. (7) Use Vitest's `--changed` to limit mutation testing to affected files. (8) Run full mutation suite nightly on a schedule. (9) Exclude mutation operators that produce equivalent mutants (e.g., no-op changes).

**Q5**: How would you implement a Vitest-compatible API for property-based testing with automatic shrinking?
**A**: (1) Create a `fastCheck` wrapper module that exports Vitest test builders:

```typescript
import { fc, test } from 'vitest-fast-check';

test.prop([fc.integer(), fc.integer()], 'addition is commutative', (a, b) => {
  expect(a + b).toBe(b + a);
});
```

(2) On failure, use fast-check's built-in shrinker to find the minimal failing input. (3) Report the minimal input in the Vitest error message. (4) Support all fast-check arbitraries. (5) Integrate with Vitest's retry: on failure, re-run with the failing seed. (6) Implement as a Vitest plugin that registers the custom test builder.

**Q6**: Design a test generation system that uses LLMs to generate Vitest tests from source code.
**A**: (1) For each source file, parse the AST (using TypeScript compiler). (2) Extract exported functions, their parameter types, and return types. (3) Build a prompt including: the function signature, JSDoc comments, and the function body (or a summary). (4) Send to an LLM API to generate test cases. (5) Post-process: validate that generated tests compile (use TypeScript compiler API). (6) Run generated tests and measure coverage. (7) Only keep tests that increase line/branch coverage. (8) Output `.test.ts` files alongside source. (9) Human review via PR.

**Q7**: How do you design a zero-flake test infrastructure for Vitest across 100 microservices?
**A**: (1) Eliminate sources of flakiness: use fake timers, mock all network calls (MSW), use in-memory databases, use deterministic random seeds. (2) Detect flakiness: run each test 3 times in CI; if any run fails, mark as flaky. (3) Quarantine: automatically move flaky tests to a `flaky/` directory and run them in a non-blocking CI stage. (4) Alert: notify the owning team via Slack when a test enters quarantine. (5) Require 5 consecutive green runs to un-quarantine. (6) Track flake rate per service over time in a dashboard. (7) Use `retry: 2` as a safety net, but aim to eliminate the need for retries.

**Q8**: How would you extend Vitest to support snapshot testing for API responses with automatic masking of dynamic fields?
**A**: (1) Create a custom matcher `toMatchApiSnapshot` that accepts field mask config:

```typescript
expect(response).toMatchApiSnapshot({
  masks: ['id', 'createdAt', 'updatedAt'],
  patterns: [/^token_.*/],
  dateFormat: 'ISO',
});
```

(2) The matcher serializes the response, replaces masked fields with placeholders (`[UUID]`, `[DATE]`, `[ID]`), and compares against the stored snapshot. (3) On first run, creates the snapshot with placeholders. (4) Support nested masking paths: `user.address.id`. (5) Implement as a Vitest plugin using `expect.extend`.

**Q9**: Design a test architecture for an event-sourced CQRS system using Vitest.
**A**: (1) Unit tests for individual event handlers (pure functions). (2) Unit tests for command handlers (input validation, event emission). (3) Integration tests: send a command, verify the correct events are stored in the event store. (4) Projection tests: given a sequence of events, verify the read model is built correctly. (5) Consistency tests: send a command, then query the read model, verify eventual consistency (using polling). (6) Snapshot tests: verify that event store snapshots are created at the correct intervals and can be used for replay. (7) Concurrency tests: send concurrent commands for the same aggregate, verify optimistic concurrency control (version conflicts). (8) Use an in-memory event store for test speed.

**Q10**: How would you implement a visual regression testing system integrated with Vitest?
**A**: (1) Use Vitest's browser mode with Playwright. (2) Render components in headless Chrome, take screenshots. (3) Use pixel-diff comparison (pixelmatch or jest-image-snapshot). (4) Store baseline images in version control (with LFS). (5) On diff, generate a diff image highlighting changed pixels. (6) Report: attach diff images to Vitest's error output. (7) Update baselines by running with `--update` flag. (8) Integration:

```typescript
import { test, expect } from 'vitest';
import { screenshot } from '../visual-test-utils';

test('homepage matches design', async () => {
  const image = await screenshot('/home');
  expect(image).toMatchImageSnapshot({
    threshold: 0.001, // 0.1% pixel diff allowed
    blur: 2, // Anti-aliasing tolerance
  });
});
```

## 13. Debugging & Troubleshooting

### 13.1 Common Errors

**Error: `Cannot use import statement outside a module`**
- Cause: Vitest is not transforming the file (ESM vs CJS issue).
- Fix: Ensure `vitest.config.ts` has the correct transform settings. For ESM, use `"type": "module"` in `package.json`.

**Error: `vi.mock is not defined`**
- Cause: `vi.mock` called outside a test file or without importing `vi`.
- Fix: Import `vi` at the top: `import { vi } from 'vitest'`.

**Error: `TypeError: vi.fn is not a function`**
- Cause: Using Jest globals (`jest.fn`) instead of Vitest (`vi.fn`).
- Fix: Replace `jest.fn` with `vi.fn`.

**Error: `The module factory of vi.mock is not allowed to reference any out-of-scope variables`**
- Cause: The mock factory references variables that are not in scope at the hoisted location.
- Fix: Use `vi.hoisted()` to define variables used in the factory:

```typescript
const mockData = vi.hoisted(() => ({ id: 1, name: 'test' }));
vi.mock('../service', () => ({
  fetch: vi.fn().mockResolvedValue(mockData),
}));
```

**Error: `Vitest was started with --watch but no test files matched`**
- Cause: Test file pattern does not match any files.
- Fix: Check `include` and `exclude` patterns in config.

### 13.2 Debugging Tips

```typescript
// Log during tests
it('debug', () => {
  const result = someFunction();
  console.log('result:', result);
  expect(result).toBeDefined();
});

// Use the debugger
it('debug with inspector', () => {
  debugger; // Requires --inspect-brk flag
  const result = someFunction();
  expect(result).toBeDefined();
});
```

```bash
# Run with Node inspector
npx vitest --inspect-brk

# Then open chrome://inspect
```

### 13.3 Inspecting Mock State

```typescript
it('inspect mock calls', () => {
  const mock = vi.fn();
  mock('a', 1);
  mock('b', 2);

  console.log('Calls:', mock.mock.calls);
  // [['a', 1], ['b', 2]]
  console.log('Results:', mock.mock.results);
  // [{ type: 'return', value: undefined }, ...]
  console.log('Instances:', mock.mock.instances);
});
```

### 13.4 Verbose Output

```bash
# Show individual test results
npx vitest run --reporter=verbose

# Show hanging processes (find tests that don't end)
npx vitest run --reporter=hanging-process

# Show full config
npx vitest run --showConfig
```

## 14. Comparison Section

### 14.1 Vitest vs. Jest

| Aspect | Vitest | Jest |
|--------|--------|------|
| Transforms | ESBuild (default) or SWC | Babel (default) or SWC |
| Speed (cold) | 2-5x faster | Baseline |
| Speed (HMR) | Instant, dependency-aware | File-pattern-based |
| TypeScript | Native ESBuild | ts-jest or babel |
| ESM | First-class | Experimental |
| Config | Vite-based | Standalone |
| Browser mode | Built-in (Playwright) | Separate (Jest-Puppeteer) |
| Coverage | v8 or Istanbul | Istanbul only |
| Mock API | `vi.fn()`, `vi.mock()` | `jest.fn()`, `jest.mock()` |
| Workspaces | Built-in workspace support | Projects config |
| Ecosystem | Growing | Very mature |

### 14.2 Vitest vs. Mocha

| Aspect | Vitest | Mocha |
|--------|--------|-------|
| Setup | Zero config | Requires plugins |
| Assertions | Built-in, Jest-compatible | External (Chai) |
| Mocking | Built-in (`vi.*`) | External (Sinon) |
| Coverage | Built-in | External (nyc) |
| Speed | Fast (ESBuild) | Moderate |
| TypeScript | Native | External (ts-node) |
| Parallelism | Built-in (threads/forks) | External |

### 14.3 Pool Types

| Pool | Isolation | Speed | Use Case |
|------|-----------|-------|----------|
| `threads` | Moderate (shared process) | Fastest | Most projects |
| `forks` | High (separate processes) | Moderate | Global state, native modules |
| `vmThreads` | High (VM sandbox) | Very fast | ESM projects (experimental) |

### 14.4 Coverage Providers

| Provider | Speed | Accuracy | Features |
|----------|-------|----------|----------|
| `v8` | Fast | Native V8 | Line, function, block |
| `istanbul` | Moderate | Instrumented | Line, branch, function, statement |

## 15. Revision Notes

### Vitest API Quick Reference

| Jest API | Vitest API |
|----------|-----------|
| `jest.fn()` | `vi.fn()` |
| `jest.mock()` | `vi.mock()` |
| `jest.spyOn()` | `vi.spyOn()` |
| `jest.useFakeTimers()` | `vi.useFakeTimers()` |
| `jest.clearAllMocks()` | `vi.clearAllMocks()` |
| `jest.resetAllMocks()` | `vi.resetAllMocks()` |
| `jest.restoreAllMocks()` | `vi.restoreAllMocks()` |
| `jest.setTimeout()` | `vi.setConfig({ testTimeout: n })` |
| `jest.retryTimes()` | `test.retry(n)` or config `retry: n` |

### Key Differences from Jest

- Use `vi.fn()` instead of `jest.fn()`
- Use `vi.mock()` instead of `jest.mock()`
- ESBuild is the default transformer (no Babel needed)
- Native TypeScript support (no ts-jest)
- Config uses Vite format (aliases in `resolve.alias`)
- ESM is fully supported
- HMR during watch mode is dependency-aware

### Best Practices

- Import explicitly: `import { describe, it, expect, vi } from 'vitest'`
- Use `vi.hoisted()` for variables in mock factories
- Use `vi.clearAllMocks()` in `beforeEach`
- Prefer `pool: 'threads'` for speed, `pool: 'forks'` for isolation
- Use `--changed` in CI to run only affected tests
- Use workspace config for monorepos
- Prefer `v8` coverage provider for speed

## 16. Cheat Sheet

```text
+==============================================================================+
|                          VITEST CHEAT SHEET                                   |
+==============================================================================+

+--- IMPORTS -----------------------------------------------------------------+
|                                                                              |
|  import { describe, it, expect, vi, beforeEach } from 'vitest'              |
|  import { describe, it, expect } from 'vitest'  // if globals: true         |
|                                                                              |
+--- TEST STRUCTURE ----------------------------------------------------------+
|                                                                              |
|  describe('Module', () => {                                                 |
|    beforeAll(() => { /* setup */ })                                         |
|    afterAll(() => { /* teardown */ })                                       |
|    beforeEach(() => { /* reset */ })                                        |
|    afterEach(() => { /* cleanup */ })                                       |
|                                                                              |
|    it('does something', () => {                                             |
|      expect(actual).toBe(expected);                                         |
|    });                                                                       |
|  });                                                                         |
|                                                                              |
+--- VI MOCKING --------------------------------------------------------------+
|                                                                              |
|  vi.fn()                           // Create mock function                  |
|  vi.fn(() => 'impl')               // With implementation                    |
|  vi.fn().mockReturnValue(v)        // Sync return value                      |
|  vi.fn().mockResolvedValue(v)      // Async resolve                          |
|  vi.fn().mockRejectedValue(e)      // Async reject                           |
|  vi.spyOn(obj, 'method')           // Wrap existing method                   |
|  vi.mock('module')                 // Mock entire module                     |
|  vi.mock('module', () => ({fn}))   // Manual mock factory                    |
|  vi.unmock('module')               // Restore original                       |
|  vi.hoisted(() => { ... })         // Hoisted code before setup              |
|                                                                              |
|  // Mock assertions                                                          |
|  expect(fn).toHaveBeenCalled()                                               |
|  expect(fn).toHaveBeenCalledTimes(n)                                         |
|  expect(fn).toHaveBeenCalledWith(a, b)                                       |
|  expect(fn).toHaveBeenLastCalledWith(a)                                      |
|  expect(fn).toHaveReturnedWith(val)                                          |
|                                                                              |
+--- CLEANUP -----------------------------------------------------------------+
|                                                                              |
|  vi.clearAllMocks()     // Clear call history                                |
|  vi.resetAllMocks()     // Clear + reset return values                       |
|  vi.restoreAllMocks()   // Restore original implementations                  |
|                                                                              |
+--- TIMERS ------------------------------------------------------------------+
|                                                                              |
|  vi.useFakeTimers()            // Replace timers with fakes                  |
|  vi.advanceTimersByTime(ms)    // Fast-forward time                          |
|  vi.runAllTimers()             // Execute all pending timers                  |
|  vi.runOnlyPendingTimers()     // Execute without scheduling new ones         |
|  vi.setSystemTime(date)        // Mock current date                          |
|  vi.useRealTimers()            // Restore real timers                        |
|                                                                              |
+--- ENVIRONMENT & GLOBALS ---------------------------------------------------+
|                                                                              |
|  vi.stubEnv('KEY', 'value')    // Mock process.env.KEY                       |
|  vi.unstubAllEnvs()            // Restore all env vars                       |
|  vi.stubGlobal('name', val)    // Mock global (window, globalThis)            |
|  vi.unstubAllGlobals()         // Restore all globals                        |
|                                                                              |
+--- ASSERTION MATCHERS ------------------------------------------------------+
|                                                                              |
|  .toBe(v)              .toEqual(v)         .toStrictEqual(v)                  |
|  .toBeNull()           .toBeUndefined()    .toBeDefined()                     |
|  .toBeTruthy()         .toBeFalsy()        .toBeNaN()                         |
|  .toBeGreaterThan(n)   .toBeLessThan(n)    .toBeCloseTo(n, d)                 |
|  .toMatch(/re/)        .toContain(item)    .toHaveLength(n)                   |
|  .toHaveProperty(k,v)  .toMatchObject(o)   .toThrow(err)                      |
|  .resolves.toBe(v)     .rejects.toThrow(e)                                    |
|                                                                              |
+--- CLI COMMANDS ------------------------------------------------------------+
|                                                                              |
|  npx vitest                    // Watch mode (default)                       |
|  npx vitest run                // Run once                                   |
|  npx vitest run --reporter=verbose  // Verbose output                        |
|  npx vitest run --coverage     // With coverage                              |
|  npx vitest run --changed      // Only changed files                         |
|  npx vitest run file.test.ts   // Specific file                              |
|  npx vitest run -t "pattern"   // Test name pattern                          |
|  npx vitest run --shard=1/4    // Run 1/4th of tests                         |
|  npx vitest run --retry=2      // Retry failed tests                         |
|  npx vitest run --update       // Update snapshots                           |
|  npx vitest --inspect-brk      // Debug mode                                 |
|                                                                              |
+--- CONFIGURATION -----------------------------------------------------------+
|                                                                              |
|  // vitest.config.ts                                                         |
|  import { defineConfig } from 'vitest/config'                               |
|                                                                              |
|  export default defineConfig({                                               |
|    test: {                                                                   |
|      globals: true,             // Enable describe/it/expect globally        |
|      environment: 'node',       // node, jsdom, happy-dom, custom            |
|      pool: 'threads',           // threads, forks, vmThreads                 |
|      include: ['src/**/*.test.ts'],  // Test file patterns                   |
|      exclude: ['node_modules'],       // Excluded patterns                   |
|      setupFiles: ['./setup.ts'],       // Setup before tests                 |
|      globalSetup: './globalSetup.ts',  // Once per worker pool               |
|      globalTeardown: './globalTeardown.ts', // Cleanup after all tests       |
|      coverage: {                                                             |
|        provider: 'v8',           // v8 or istanbul                           |
|        reporter: ['text', 'lcov'],                                          |
|        thresholds: { lines: 80, branches: 75 },                             |
|      },                                                                      |
|      retry: 0,                    // Retry failed tests                       |
|      testTimeout: 10000,          // Per-test timeout                         |
|      poolOptions: { threads: { singleThread: false } },                     |
|    },                                                                        |
|    resolve: {                                                                |
|      alias: { '@': '/src' },     // Path aliases                             |
|    },                                                                        |
|  });                                                                         |
|                                                                              |
+==============================================================================+
```
