# Poku Background Services Reference

Use these utilities for integration and e2e tests that need external processes (servers, databases, workers).

---

## `startService` — Run a File in Background

Spawns a file in a detached child process and waits for it to signal readiness.

```ts
import { startService } from 'poku';

const server = await startService('./src/server.ts', {
  startAfter: 'Listening on',  // wait for this substring in stdout
  timeout: 30_000,             // ms before rejecting (default: none)
  verbose: false,              // forward service stdout/stderr to parent (default: false)
  cwd: './',                   // working directory (default: process.cwd())
});
```

### `startAfter` Values

| Value | Behaviour |
|-------|-----------|
| `string` | Resolves when stdout contains this substring (case-sensitive) |
| `number` | Waits exactly N milliseconds, then resolves |
| `undefined` | Resolves on the very first stdout output |

### `end(port?)` Method

```ts
await server.end();       // kill the process
await server.end(3000);   // kill process + force-close port (needed for Bun/Deno)
```

Always call `end()` — otherwise the background process keeps running after the test file exits.

**Recommended pattern — always clean up:**

```ts
let server: Awaited<ReturnType<typeof startService>>;
try {
  server = await startService('./server.ts', { startAfter: 'ready' });
  // ... tests ...
} finally {
  await server?.end(3000);
}
```

---

## `startScript` — Run a `package.json` Script

Same semantics as `startService` but executes a script by name from `package.json` / `deno.json` .

```ts
import { startScript } from 'poku';

const server = await startScript('start:test', {
  startAfter: 'Listening',
  runner: 'npm',    // 'npm' | 'bun' | 'deno' | 'yarn' | 'pnpm'
  timeout: 60_000,
  verbose: false,
  cwd: './',
});

await server.end(3000);
```

---

## `waitForPort` — Poll Until TCP Port is Open

Use **after** starting a service when you need to confirm the port is actually accepting connections (not just that the process started).

```ts
import { waitForPort } from 'poku';

await waitForPort(3000, {
  host: 'localhost',  // default
  interval: 100,      // retry every 100ms (default)
  timeout: 15_000,    // give up after 15s and throw
  delay: 200,         // extra wait after port opens (for slow bootstraps)
});
```

**Combine with `startService` :**

```ts
const server = await startService('./server.ts', { startAfter: 'ready' });
await waitForPort(3000, { timeout: 10_000, delay: 100 });
// guaranteed: port is open before tests run
```

---

## `waitForExpectedResult` — Poll Any Condition

Generic polling for when a port check isn't enough — e.g., waiting for an endpoint to return 200, a DB to have data, or a queue to drain.

```ts
import { waitForExpectedResult } from 'poku';

// Wait for HTTP health check
await waitForExpectedResult(
  () => fetch('http://localhost:3000/health').then(r => r.status),
  200,
  { interval: 200, timeout: 30_000 }
);

// Wait for DB row to exist
await waitForExpectedResult(
  () => db.query('SELECT COUNT(*) FROM jobs WHERE status = $1', ['done']).then(r => r.rows[0].count),
  '3',
  { interval: 500, timeout: 10_000 }
);
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `interval` | `number` | `100` | Retry delay in ms |
| `timeout` | `number` | `undefined` | Reject after this many ms |
| `strict` | `boolean` | `false` | Use `deepStrictEqual` instead of `deepEqual` |

---

## `sleep` — Simple Delay

```ts
import { sleep } from 'poku';

await sleep(500);   // integer ms — throws if non-integer passed
```

Prefer `waitForPort` or `waitForExpectedResult` over `sleep` when possible. `sleep` is a last resort when there's no observable signal.

---

## `kill` — Terminate Ports / Processes

```ts
import { kill } from 'poku';

await kill.port(3000);              // kill process listening on port 3000
await kill.port([3000, 4000]);      // kill multiple ports
await kill.range(3000, 3010);       // kill all ports in range (inclusive)
await kill.pid(1234);               // kill by PID
await kill.pid([1234, 5678]);       // kill multiple PIDs
```

**Requirements:** Uses `lsof` on Linux/macOS, `netstat` on Windows. Ensure these are available in your environment.

**From CLI — kill before the suite starts:**

```bash
npx poku --killPort=3000 test/
npx poku --killPort=3000 --killPort=4001 test/
npx poku --killRange=3000-3003 test/
npx poku --killPid=1234 test/
```

---

## `getPIDs` — Inspect Processes on Ports

```ts
import { getPIDs } from 'poku';

const pids = await getPIDs(3000);               // → number[] for port
const pids = await getPIDs([3000, 4000]);        // → number[] for multiple ports
const pids = await getPIDs.range(3000, 3010);   // → number[] for range
```

Useful for debugging lingering processes before deciding whether to kill them.

---

## Full E2E Lifecycle Pattern

```ts
import { describe, it, startService, waitForPort, strict } from 'poku';

let server: Awaited<ReturnType<typeof startService>>;

try {
  await describe('HTTP API', async () => {
    // 1. Start the server process
    server = await startService('./src/app.ts', {
      startAfter: 'Server ready',
      timeout: 30_000,
    });

    // 2. Confirm the port is truly open
    await waitForPort(3000, { timeout: 10_000, delay: 100 });

    // 3. Run tests
    await it('GET /health', async () => {
      const res = await fetch('http://localhost:3000/health');
      strict.strictEqual(res.status, 200);
    });

    await it('POST /users', async () => {
      const res = await fetch('http://localhost:3000/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email: 'test@example.com' }),
      });
      strict.strictEqual(res.status, 201);
      const body = await res.json();
      strict.ok(body.id);
    });
  });
} finally {
  // 4. Always clean up
  await server?.end(3000);
}
```
