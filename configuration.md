# Poku Configuration Reference

---

## Config File Auto-Discovery

Poku looks for config files in the project root in this order:

1. `.pokurc.jsonc`
2. `.pokurc.json`
3. `poku.config.js` (ESM or CJS)

Override with: `npx poku --config=path/to/config.ts`

Supported custom path extensions: `.ts` , `.mjs` , `.cjs` , `.json` , `.jsonc`

---

## `defineConfig` — Type-Safe Config

```ts
// poku.config.ts
import { defineConfig } from 'poku';

export default defineConfig({
  include: ['./test'],             // directories to search (CLI paths override this)
  filter: /\.(test|spec)\./,       // include files matching regex
  exclude: /fixtures/,             // exclude files matching regex (or array of regex)
  sequential: false,               // run files one-at-a-time
  concurrency: 4,                  // max parallel test files (default: availableParallelism - 1)
  timeout: 10_000,                 // per-file timeout in ms
  failFast: false,                 // stop suite on first failed file
  debug: false,                    // show all console output from test files
  quiet: false,                    // suppress all output
  isolation: 'process',            // 'process' (default) | 'none'
  reporter: 'poku',                // see Reporters section
  envFile: '.env.test',            // path, or true to use '.env'
  kill: {
    port: [3000, 4000],            // kill ports before suite starts
    range: [[3000, 3003]],         // kill port ranges [[start, end], ...]
    pid: [],                       // kill PIDs before suite starts
  },
  deno: {
    allow: ['run', 'env', 'read', 'net'],
    deny: [],
  },
  beforeEach: async () => await setupDB(),    // runs before each TEST FILE
  afterEach:  async () => await teardownDB(), // runs after each TEST FILE
});
```

CommonJS:

```js
const {
    defineConfig
} = require('poku');
module.exports = defineConfig({
    ...
});
```

JSON config with editor autocomplete:

```json
{
  "$schema": "https://poku.io/schemas/configs.json",
  "filter": "\\.(test|spec)\\.",
  "sequential": true,
  "timeout": 5000,
  "reporter": "dot"
}
```

---

## All Config Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `include` | `string[]` | — | Directories to search for test files |
| `filter` | `RegExp` | `/\.(test\|spec)\./i` | Include files matching pattern |
| `exclude` | `RegExp \| RegExp[]` | — | Exclude files matching pattern |
| `sequential` | `boolean` | `false` | Run files one-at-a-time |
| `concurrency` | `number` | `availableParallelism() - 1` | Max concurrent test files |
| `timeout` | `number` | — | Per-file timeout in ms |
| `failFast` | `boolean` | `false` | Stop on first failure |
| `debug` | `boolean` | `false` | Show all process output (overrides `quiet` ) |
| `quiet` | `boolean` | `false` | Suppress all output |
| `isolation` | `'process' \| 'none'` | `'process'` | Per-file process isolation |
| `reporter` | `string \| ReporterPlugin` | `'poku'` | Reporter name or custom reporter |
| `noExit` | `boolean` | `false` | Return code instead of calling `process.exit` |
| `envFile` | `string \| true` | — | Load env file (true = `.env` ) |
| `kill.port` | `number[]` | — | Kill ports before run |
| `kill.range` | `[number, number][]` | — | Kill port ranges before run |
| `kill.pid` | `number[]` | — | Kill PIDs before run |
| `deno.allow` | `string[]` | — | Deno `--allow-*` permissions |
| `deno.deny` | `string[]` | — | Deno `--deny-*` permissions |
| `beforeEach` | `() => unknown` | — | Run before each test **file** |
| `afterEach` | `() => unknown` | — | Run after each test **file** |
| `plugins` | `PokuPlugin[]` | — | Plugin array |
| `testNamePattern` | `RegExp` | — | Run only `it` / `test` with matching title |
| `testSkipPattern` | `RegExp` | — | Skip `it` / `test` with matching title |

---

## Reporters

| Name | Description | Best for |
|------|-------------|----------|
| `poku` | Full output: describe/it titles, real-time errors, summary | Development |
| `dot` | `.` per passing file, `F` per failing file; errors at end | CI with many tests |
| `compact` | `PASS` / `FAIL` badge per file; errors at end | CI |
| `focus` | Shows only failures in real time; silent if all pass | Debugging a subset |
| `classic` | Style from Poku v2 | Legacy preference |

```bash
npx poku --reporter=dot test/
npx poku -r compact test/
```

---

## Full CLI Flags Reference

All flags use camelCase. Use `=` for values: `--concurrency=4` .

| Flag | Short | Description |
|------|-------|-------------|
| `--concurrency=N` | | Max concurrent test files |
| `--config=FILE` | `-c=FILE` | Config file path |
| `--debug` | `-d` | Show all process output |
| `--denoAllow=X,Y` | | Deno allow permissions (comma-separated) |
| `--denoDeny=X,Y` | | Deno deny permissions (comma-separated) |
| `--enforce` | `-x` | Validate CLI options before running (fail fast on typos) |
| `--envFile[=path]` | | Load env file (default: `.env` ) |
| `--exclude=PATTERN` | | Exclude files matching regex |
| `--failFast` | | Stop on first failure |
| `--filter=PATTERN` | | Include only files matching regex |
| `--help` | `-h` | Show help |
| `--isolation=MODE` | | `process` (default) or `none` |
| `--killPid=PID,...` | | Kill PIDs before run |
| `--killPort=PORT,...` | | Kill ports before run |
| `--killRange=S-E,...` | | Kill port ranges (e.g. `3000-3003` ) |
| `--listFiles` | | List discovered test files and exit |
| `--only` | | Enable `.only` modifier globally |
| `--quiet` | `-q` | No console output |
| `--reporter=NAME` | `-r=NAME` | Reporter name |
| `--sequential` | | Run files one-at-a-time |
| `--testNamePattern=P` | `-t=P` | Run only tests whose title matches |
| `--testSkipPattern=P` | | Skip tests whose title matches |
| `--timeout=MS` | | Per-file timeout in ms |
| `--version` | `-v` | Print version |
| `--watch` | `-w` | Watch mode |
| `--watchInterval=MS` | | Watch poll interval (default: `1500` ) |

### `--enforce` / `-x`

Validates all CLI options before the run — catches typos, invalid values, missing files. Useful in CI:

```bash
npx poku -x --sequential --reporter=compact test/
```

### `--listFiles`

Preview exactly which files Poku will discover without running them:

```bash
npx poku --listFiles --filter=\.e2e\. test/
```

---

## `listFiles` — Programmatic File Discovery

```ts
import { listFiles } from 'poku';

const files = await listFiles('test/', {
  filter: /\.test\./,
  exclude: /fixtures/,
});
// → string[] of absolute paths
```
