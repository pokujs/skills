# Poku Patterns

Runnable, real-world patterns — copy, adapt, ship.

---

## 1. Minimal — Pure Assertions, No Structure

When you just need to verify a module's output without scaffolding:

```ts
// test/unit/math.test.ts
import { strict } from 'poku';
import { add, divide } from '../src/math.ts';

strict.strictEqual(add(1, 2), 3);
strict.strictEqual(add(-1, -1), -2);
strict.throws(() => divide(1, 0), /division by zero/i);
```

Run it directly — no config needed: `node test/unit/math.test.ts` or `npx poku test/unit/` .

---

## 2. Unit Test with Setup/Teardown

Using file-level `beforeEach` / `afterEach` to reset state between each `it` :

```ts
// test/unit/user-service.test.ts
import { describe, it, beforeEach, afterEach, strict } from 'poku';
import { UserService } from '../src/user-service.ts';
import { db } from '../src/db.ts';

const userService = new UserService(db);

beforeEach(async () => await db.migrate());
afterEach(async () => await db.rollback());

await describe('UserService', async () => {
  await it('creates a user', async () => {
    const user = await userService.create({ email: 'alice@example.com' });
    strict.ok(user.id);
    strict.strictEqual(user.email, 'alice@example.com');
  });

  await it('rejects duplicate emails', async () => {
    await userService.create({ email: 'alice@example.com' });
    await strict.rejects(
      () => userService.create({ email: 'alice@example.com' }),
      /duplicate/i
    );
  });

  await it('returns null for unknown ID', async () => {
    const user = await userService.findById('nonexistent-id');
    strict.strictEqual(user, null);
  });
});
```

---

## 3. Integration Test — Multiple Suites in One File

Use `describe` blocks to group related tests, `beforeEach` with `pause()` / `continue()` to control when hooks are active:

```ts
// test/integration/orders.test.ts
import { describe, it, beforeEach, afterEach, strict } from 'poku';
import { createOrder, cancelOrder } from '../src/orders.ts';
import { db } from '../src/db.ts';

const before = beforeEach(async () => await db.begin());
const after  = afterEach(async () => await db.rollback());

await describe('createOrder', async () => {
  await it('creates order with line items', async () => {
    const order = await createOrder({ userId: 'u1', items: [{ sku: 'ABC', qty: 2 }] });
    strict.ok(order.id);
    strict.strictEqual(order.items.length, 1);
    strict.strictEqual(order.status, 'pending');
  });

  await it('validates non-empty items', async () => {
    await strict.rejects(
      () => createOrder({ userId: 'u1', items: [] }),
      /items must not be empty/i
    );
  });
});

await describe('cancelOrder', async () => {
  await it('cancels a pending order', async () => {
    const order = await createOrder({ userId: 'u1', items: [{ sku: 'ABC', qty: 1 }] });
    const cancelled = await cancelOrder(order.id);
    strict.strictEqual(cancelled.status, 'cancelled');
  });

  await it('rejects cancelling a shipped order', async () => {
    // Temporarily disable auto-reset between these nested its
    before.pause();
    after.pause();
    const order = await createOrder({ userId: 'u1', items: [{ sku: 'ABC', qty: 1 }] });
    await db.query("UPDATE orders SET status = 'shipped' WHERE id = $1", [order.id]);
    await strict.rejects(() => cancelOrder(order.id), /cannot cancel/i);
    await db.rollback();
    before.continue();
    after.continue();
  });
});
```

---

## 4. E2E — Background Server + HTTP Tests

Full lifecycle: start server → wait for port → test → teardown.

```ts
// test/e2e/api.test.ts
import { describe, it, startService, waitForPort, strict } from 'poku';

let server: Awaited<ReturnType<typeof startService>>;

try {
  server = await startService('./src/server.ts', {
    startAfter: 'Server ready',   // substring in stdout
    timeout: 30_000,
  });

  await waitForPort(3000, { timeout: 10_000, delay: 100 });

  await describe('GET /health', async () => {
    await it('returns 200', async () => {
      const res = await fetch('http://localhost:3000/health');
      strict.strictEqual(res.status, 200);
    });

    await it('returns JSON body', async () => {
      const res = await fetch('http://localhost:3000/health');
      const body = await res.json();
      strict.deepStrictEqual(body, { status: 'ok' });
    });
  });

  await describe('POST /users', async () => {
    await it('creates a user and returns 201', async () => {
      const res = await fetch('http://localhost:3000/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email: 'test@example.com', name: 'Test' }),
      });
      strict.strictEqual(res.status, 201);
      const body = await res.json();
      strict.ok(body.id);
      strict.strictEqual(body.email, 'test@example.com');
    });

    await it('returns 400 for missing email', async () => {
      const res = await fetch('http://localhost:3000/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ name: 'No Email' }),
      });
      strict.strictEqual(res.status, 400);
    });
  });
} finally {
  await server?.end(3000);
}
```

---

## 5. E2E — Start Server via npm Script

```ts
// test/e2e/api.test.ts
import { startScript, waitForPort, it, strict } from 'poku';

const server = await startScript('start:test', {
  startAfter: 'Listening',
  runner: 'npm',
  timeout: 60_000,
});

await waitForPort(4000, { timeout: 15_000 });

await it('health check', async () => {
  const res = await fetch('http://localhost:4000/health');
  strict.strictEqual(res.status, 200);
});

await server.end(4000);
```

---

## 6. Skip Guards — Platform, Runtime, CI

```ts
import { skip } from 'poku';
import { platform } from 'node:process';

// Platform-specific
if (platform === 'win32') skip('Uses POSIX signals, incompatible with Windows');

// CI only
if (!process.env.CI) skip('Integration tests only run in CI');

// Runtime-specific
const isBun = typeof Bun !== 'undefined';
if (!isBun) skip('Bun-specific API test');

// Build target guard (run against compiled lib only)
const testingLib = process.env.POKU_TEST_TARGET === 'lib';
if (!testingLib) skip('Only run against compiled lib');
```

---

## 7. Title Pattern Filtering

Run a subset of tests without editing code:

```bash
# Run only tests whose title contains "creates user"
npx poku -t "creates user" test/unit/

# Run only e2e tests matching a route
npx poku --testNamePattern="GET /health" test/e2e/

# Skip slow tests (title must contain "slow")
npx poku --testSkipPattern="slow" test/

# Focus with .only from code (requires --only flag)
npx poku --only test/unit/
```

In code — `it.only` without `--only` exits with code 1:

```ts
it.only('I need to focus on this', async () => { ... });
// npx poku --only test/unit/user.test.ts
```

---

## 8. `describe` as Visual Title (No Callback)

Separate suites visually in the reporter without nesting:

```ts
import { describe, it, strict } from 'poku';

describe('Authentication', { background: 'blue' });

await it('login with valid credentials', async () => { ... });
await it('rejects invalid password', async () => { ... });

describe('Authorization', { background: 'blue' });

await it('admin can delete users', async () => { ... });
await it('regular user cannot delete users', async () => { ... });
```

---

## 9. Cross-Runtime Test File

A single file that works on Node.js, Bun, and Deno:

```ts
// test/unit/parser.test.ts
import { describe, it, strict } from 'poku';
import { parseCSV } from '../src/parser.ts';  // relative import, no ext mapping needed

await describe('parseCSV', async () => {
  await it('parses header and rows', async () => {
    const result = parseCSV('name,age\nAlice,30\nBob,25');
    strict.deepStrictEqual(result, [
      { name: 'Alice', age: '30' },
      { name: 'Bob', age: '25' },
    ]);
  });

  await it('handles empty input', async () => {
    strict.deepStrictEqual(parseCSV(''), []);
  });
});
```

Run on each runtime:

```bash
npx poku test/unit/          # Node.js
bun poku test/unit/          # Bun
deno run npm:poku test/unit/ # Deno
```
