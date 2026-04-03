---
name: poku
description: Use when writing, running, or debugging tests with Poku — a cross-runtime test runner for Node.js, Bun, and Deno. Covers assertions, describe/it blocks, background services, watch mode, configuration, CLI flags, and patterns for integration and e2e tests.
---

# Poku Testing Framework

## Overview

Poku is a **cross-runtime, file-based test runner** targeting Node.js (≥16), Bun (≥1), and Deno (≥2) with native TypeScript support via `tsx`. Its core insight: **test files are plain JavaScript/TypeScript** — they execute top-to-bottom with ordinary `await`, not framework magic. Every test file runs in its own child process by default, giving complete isolation between test files.

```bash
npm i -D poku          # Node.js
npm i -D poku tsx      # Node.js + TypeScript
bun add -d poku        # Bun
deno add npm:poku      # Deno (optional)
```

**Supporting reference files in this skill:**
- [assertions.md](assertions.md) — full `assert` / `strict` API with all methods
- [background-services.md](background-services.md) — `startService`, `startScript`, `waitForPort`, `kill`, and more
- [configuration.md](configuration.md) — config file options, all CLI flags, reporters
- [patterns.md](patterns.md) — runnable unit, integration, and e2e examples

---

## Core Concepts

### Plain Files, Natural Flow

Unlike Jest/Vitest, Poku has no hidden execution model. An `async describe` callback runs top-to-bottom; `await` controls sequencing.

```ts
import { describe, it, strict } from 'poku';

// Simplest possible test — no describe/it required
strict.strictEqual(add(1, 2), 3);

// With structure
await describe('Calculator', async () => {
  await it('adds numbers', async () => {
    strict.strictEqual(add(1, 2), 3);
  });
  await it('throws on NaN', async () => {
    strict.throws(() => add(NaN, 1), /NaN/);
  });
});
```

### Process Isolation

Each test **file** runs in its own child process (`isolation: 'process'` default). Module caches, global state, and `process.exitCode` are fully independent per file. Use `isolation: 'none'` only when you intentionally share state across files.

---

## Running Tests

```bash
npx poku test/                    # run all tests in directory
npx poku test/unit test/e2e       # multiple paths
npx poku --filter=\.e2e\.         # files matching regex
npx poku --sequential             # run files one at a time
npx poku --failFast               # stop on first failure
npx poku --watch                  # watch mode (re-runs changed file)
npx poku --debug                  # show all console output
npx poku --listFiles              # preview which files will run
npx poku -t "user login"          # run only tests matching title
npx poku --reporter=dot           # minimal CI output
```

See [configuration.md](configuration.md) for the full flag list.

```ts
import { poku } from 'poku';

await poku(['test/unit', 'test/integration']);          // calls process.exit()
const code = await poku('test/', { noExit: true });    // returns 0 | 1
```

---

## `describe` / `it` / `test`

`test` is an alias for `it`.

```ts
describe(title, cb)           // group with title
describe(title, { background: 'blue' })  // title-only (visual separator, no cb)
it(title, cb)                 // named test
it(cb)                        // anonymous test

it.skip('reason', cb)         // skip — counts as passed
it.todo('planned test')       // todo — cb ignored, hooks skipped
it.only('critical', cb)       // run only this — requires --only flag
// same modifiers on describe
```

**Critical: always `await` async `it`/`describe` when order matters**

```ts
// ❌ WRONG — it blocks run concurrently, order not guaranteed
describe('Suite', async () => {
  it(async () => { await doA(); });
  it(async () => { await doB(); });   // may run before doA
});

// ✅ RIGHT — sequential execution
await describe('Suite', async () => {
  await it(async () => { await doA(); });
  await it(async () => { await doB(); });
});
```

---

## Hooks

### File-level — `beforeEach` / `afterEach`

Run before/after every `it`/`test` block in the same file. `todo`-modified tests do **not** trigger hooks.

```ts
import { beforeEach, afterEach } from 'poku';

const before = beforeEach(async () => await db.reset());
const after  = afterEach(async () => await db.cleanup());

// Control the lifecycle
before.pause();    before.continue();    before.reset();
```

`beforeEach` also accepts `{ immediate: true }` to fire once on registration.

### Suite-level — via `poku()` API

Run before/after each **file** (not each `it`). Cannot be set from CLI.

```ts
await poku('test/', {
  beforeEach: async () => await seed(),
  afterEach:  async () => await truncate(),
});
```

---

## Skip / Conditional Skip

```ts
import { skip } from 'poku';

skip();                                       // skip file, default message
skip('reason');                               // skip with message

if (platform === 'win32') skip('POSIX only'); // platform guard
if (!process.env.CI) skip('CI only');         // env guard
```

Skipped files count as **passed** — never increment failure count.

---

## Watch Mode

```bash
npx poku --watch                   # polls every 1500ms
npx poku --watch --watchInterval=500
```

- Re-runs the specific **changed file** on save
- Type `rs` + Enter to reset all watchers and re-run everything
- Dynamic imports with variable paths are **not** tracked

---

## Utilities

```ts
import { envFile, log, sleep } from 'poku';

await envFile();             // load .env (no ../ traversal allowed)
await envFile('.env.test');  // custom path

log('always visible');       // unlike console.log, visible without --debug

await sleep(500);            // integer ms required
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `await` on async `it`/`describe` | Always await when order or teardown matters |
| `console.log` invisible | Use `log()` from poku, or run with `--debug` |
| `it.only` exits with code 1 | Pass `--only` CLI flag alongside, or remove `.only` |
| `beforeEach` not firing | Check if test uses `.todo` — todo skips all hooks |
| Service still running after test | Always call `server.end()`, ideally in a `finally` block |
| `isolation: 'none'` + parallel files | Module state collides — use `'process'` (default) |

---

## Quick Reference

```
poku(paths, configs?)              Run test suite
assert / strict                    Assertions (see assertions.md)
describe(title, cb)                Group tests
it(title, cb) / test(title, cb)    Single test block
it.skip / it.todo / it.only        Test modifiers
beforeEach(cb, opts?)              Hook before each it
afterEach(cb)                      Hook after each it
startService / startScript         Background processes (see background-services.md)
waitForPort / waitForExpectedResult  Readiness polling (see background-services.md)
sleep(ms)                          Promise delay
kill.port / kill.range / kill.pid  Terminate processes (see background-services.md)
envFile(path?)                     Load .env file
skip(reason?)                      Skip current file
log(msg)                           Always-visible log
listFiles(dir, opts?)              List matching test files
defineConfig(opts)                 Type-safe config (see configuration.md)
version                            Installed version string
```
