# Poku Assertions Reference

Poku exports two assertion modules built on Node's native `node:assert` :

| Module | Comparison style | Import |
|--------|-----------------|--------|
| `assert` | Loose ( `==` ) | `import { assert } from 'poku'` |
| `strict` | Strict ( `===` ) | `import { strict } from 'poku'` |

**Prefer `strict` for all new code.** Use `assert` only when you intentionally need loose equality.

Both modules share the same surface — everything below applies to both.

---

## All Methods

### Truthiness

```ts
assert.ok(value)               // passes if !!value === true
assert.ok(value, 'message')    // all methods accept string | Error as last arg
assert.ok(value, new Error('typed'))
```

### Equality

```ts
assert.equal(actual, expected)            // actual == expected  (loose)
assert.strictEqual(actual, expected)      // actual === expected

assert.notEqual(actual, expected)         // actual != expected
assert.notStrictEqual(actual, expected)   // actual !== expected
```

### Deep Equality

```ts
assert.deepEqual(actual, expected)           // recursive ==
assert.deepStrictEqual(actual, expected)     // recursive === (types must match)

assert.notDeepEqual(actual, expected)
assert.notDeepStrictEqual(actual, expected)
```

```ts
// Deep examples
strict.deepStrictEqual({ a: 1 }, { a: 1 });            // ✅ passes
strict.deepStrictEqual({ a: 1 }, { a: '1' });           // ❌ fails (type mismatch)
assert.deepEqual({ a: 1 }, { a: '1' });                 // ✅ passes (loose)
```

### Error Assertions — Synchronous

```ts
assert.throws(
  () => riskyFn(),
  /expected error message/   // regex tested against error.message
);

assert.throws(
  () => riskyFn(),
  { message: 'exact message', name: 'TypeError' }  // or error-shape object
);

assert.doesNotThrow(() => safeFn());
```

### Error Assertions — Asynchronous

```ts
await assert.rejects(
  async () => await riskyAsync(),
  /message regex/
);

await assert.rejects(
  () => Promise.reject(new TypeError('bad')),
  TypeError   // constructor match
);

await assert.doesNotReject(async () => await safeFn());
```

### Other

```ts
assert.fail('always fails with this message');

assert.ifError(err);  // throws err if err is truthy — ideal for (err, result) callbacks:
fs.readFile('path', (err, data) => {
  assert.ifError(err);  // throws if err is set, passes silently otherwise
  // ...
});
```

---

## `strict` vs `assert` — When to Use Which

```ts
// assert — loose equality, never coerces
assert.equal('1', 1);           // ✅ passes ('1' == 1 is true in JS)
assert.deepEqual([1], ['1']);    // ✅ passes

// strict — type-strict, mirrors ===
strict.equal('1', 1);           // ❌ FAILS
strict.deepStrictEqual([1], ['1']); // ❌ FAILS

// Rule of thumb
// strict → use for all value comparisons
// assert → use only when testing legacy code that mixes types
```

---

## Custom Error Messages

Every assertion method accepts an optional final argument of `string | Error` :

```ts
strict.strictEqual(user.role, 'admin', 'Only admins can access this route');
strict.ok(response.body, new TypeError('Response body must not be empty'));
```

The message replaces the default diff output when the assertion fails. Use it to make failures self-explanatory without reading source code.

---

## Quick Cheatsheet

```
ok(v)                → !!v
equal(a, b)          → a == b
strictEqual(a, b)    → a === b
deepEqual(a, b)      → recursive ==
deepStrictEqual(a,b) → recursive ===
notEqual / notStrictEqual / notDeepEqual / notDeepStrictEqual
throws(fn, matcher)
doesNotThrow(fn)
rejects(fn, matcher)         → async
doesNotReject(fn)            → async
fail(msg)
ifError(err)
```
