---
title: "Modern JavaScript ES2022-ES2027"
tags: [frontend, frontend/javascript]
aliases: [Modern JavaScript, ES2022, ES2023, ES2024, ES2025, ES2026, ES2027, What's new in JavaScript]
updated: 2026-09-27
---
# Modern JavaScript ES2022-ES2027

<Note>
**TL;DR**

A year-by-year catalogue of what landed in ECMAScript since 2022, with a one-line snippet each and a link to the note that explains it in depth. The yearly spec is published every June: **ES2025** (16th edition, June 2025) and **ES2026** (17th edition, approved **June 30, 2026**). Explicit Resource Management (`using`) and Temporal are finished (Stage 4) but target **ES2027**.
</Note>

## How features get in (TC39 stages)

{/* diagram source: /diagrams/diagram-221.mmd */}
![diagram](/diagrams/diagram-221.svg)

- "Stage 4" ≠ "works everywhere". Check **Baseline** status: *Newly available* (all core browsers support it as of a date), *Widely available* (30+ months in all core browsers), *Limited* (at least one core browser is missing it).
- Browsers often ship features before the spec year (e.g. `RegExp.escape` was Baseline in 2025 but is in ES2026).

## ES2022

| Feature | Snippet | Deep dive |
|---|---|---|
| Class fields, private `#fields`/`#methods`, static fields | `class A { #count = 0; static max = 5; #inc() {} }` | [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance) |
| Class static blocks | `class C { static { C.init(); } }` | [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance) |
| Ergonomic brand checks | `#count in obj` | [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance) |
| Top-level `await` | `const cfg = await loadConfig();` (in modules) | [Modules (ESM)](/Software%20Engineering/Frontend/01%20JavaScript/Modules%20(ESM)) |
| `.at()` (arrays, strings, typed arrays) | `arr.at(-1)` | [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability) |
| `Object.hasOwn` | `Object.hasOwn(obj, 'id')` | [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability) |
| Error cause | `new Error('Load failed', { cause: err })` | [Error Handling](/Software%20Engineering/Frontend/01%20JavaScript/Error%20Handling) |
| RegExp match indices `/d` | `/(?<y>\d{4})/d.exec(s).indices.groups.y` | |

## ES2023

| Feature | Snippet | Deep dive |
|---|---|---|
| Change array by copy | `arr.toSorted()`, `arr.toReversed()`, `arr.toSpliced(1, 1)`, `arr.with(0, x)` | [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability) |
| `findLast` / `findLastIndex` | `bookings.findLast(b => b.paid)` | [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability) |
| Symbols as WeakMap keys | `new WeakMap([[Symbol('k'), v]])` (non-registered symbols) | [Collections - Map, Set, WeakMap & WeakRef](/Software%20Engineering/Frontend/01%20JavaScript/Collections%20-%20Map%2C%20Set%2C%20WeakMap%20%26%20WeakRef) |
| Hashbang grammar | `#!/usr/bin/env node` at the top of a script | |

## ES2024

| Feature | Snippet | Deep dive |
|---|---|---|
| `Object.groupBy` / `Map.groupBy` | `Object.groupBy(flights, f => f.carrier)` | [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability) |
| `Promise.withResolvers` | `const { promise, resolve, reject } = Promise.withResolvers()` | [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation) |
| RegExp `v` flag (set notation, string properties) | `/[\p{L}--[a-z]]/v` | |
| Well-formed Unicode strings | `str.isWellFormed()`, `str.toWellFormed()` | |
| Resizable / transferable `ArrayBuffer` | `new ArrayBuffer(8, { maxByteLength: 64 }).resize(16)`; `buf.transfer()` | |
| `Atomics.waitAsync` | `await Atomics.waitAsync(int32, 0, 0).value` | [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs) |

## ES2025 (published June 2025)

| Feature | Baseline | Snippet | Deep dive |
|---|---|---|---|
| Sync iterator helpers | Newly, 2025-03-31 | `naturals().filter(n => n % 2 === 0).map(n => n * n).take(3).toArray()` | [Iterators, Generators & Iterator Helpers](/Software%20Engineering/Frontend/01%20JavaScript/Iterators%2C%20Generators%20%26%20Iterator%20Helpers) |
| Set methods | Newly, 2024-06-11 | `a.union(b); a.intersection(b); a.difference(b); a.isSubsetOf(b)` | [Collections - Map, Set, WeakMap & WeakRef](/Software%20Engineering/Frontend/01%20JavaScript/Collections%20-%20Map%2C%20Set%2C%20WeakMap%20%26%20WeakRef) |
| `Promise.try` | Newly, 2025-01-07 | `Promise.try(() => mayThrowSync(x)).catch(handle)` | [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation) |
| JSON modules + import attributes | Newly, 2025-04-29 | `import cfg from './cfg.json' with { type: 'json' }` | [Modules (ESM)](/Software%20Engineering/Frontend/01%20JavaScript/Modules%20(ESM)) |
| `Float16Array`, `Math.f16round` | Newly, 2025-04-04 | `new Float16Array([1.5, 2.25])` | |
| RegExp modifiers | | `/^(?i:hello) World$/` | |
| Duplicate named capture groups | | `/(?<y>\d{4})-\d\d\|\d\d-(?<y>\d{4})/` | |
| Redeclarable global eval-vars | | (minor spec cleanup) | |

## ES2026 (approved June 30, 2026)

| Feature | Baseline | Snippet | Deep dive |
|---|---|---|---|
| `RegExp.escape` | Newly, 2025-05-01 | `new RegExp(RegExp.escape(userInput), 'g')` | [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) |
| `Array.fromAsync` | Widely, 2026-07-25 | `const rows = await Array.fromAsync(asyncIterable)` | [Iterators, Generators & Iterator Helpers](/Software%20Engineering/Frontend/01%20JavaScript/Iterators%2C%20Generators%20%26%20Iterator%20Helpers) |
| `Error.isError` | **Limited** (no Safari) | `Error.isError(e)` (works across realms) | [Error Handling](/Software%20Engineering/Frontend/01%20JavaScript/Error%20Handling) |
| `Math.sumPrecise` | Newly, 2026-04-10 | `Math.sumPrecise([1e20, 0.1, -1e20]) // 0.1` | |
| `Uint8Array` base64/hex | Newly, 2025-09-05 | `Uint8Array.fromBase64('SGVsbG8='); bytes.toBase64(); bytes.toHex()` | |
| Upsert (`getOrInsert`, `getOrInsertComputed`) | Newly, 2026-02-14 | `counts.getOrInsert(k, 0)` | [Collections - Map, Set, WeakMap & WeakRef](/Software%20Engineering/Frontend/01%20JavaScript/Collections%20-%20Map%2C%20Set%2C%20WeakMap%20%26%20WeakRef) |
| `Iterator.concat` | Newly, 2026-03-24 | `Iterator.concat(a, b, c)` | [Iterators, Generators & Iterator Helpers](/Software%20Engineering/Frontend/01%20JavaScript/Iterators%2C%20Generators%20%26%20Iterator%20Helpers) |
| JSON.parse source text access | Newly, 2025-03-31 | `JSON.parse(t, (k, v, { source }) => k === 'id' ? BigInt(source) : v)` | |

<Info>
**Why `RegExp.escape` matters**

```js
// User searches for "C++ (beginner)": special characters break or inject into the regex
const unsafe = new RegExp(query, 'i');                  // SyntaxError or wrong matches
const safe = new RegExp(RegExp.escape(query), 'i');     // matches the literal text
```
</Info>

<Info>
**Why JSON source text access matters**

```js
JSON.parse('{"id": 12345678901234567890}').id;           // 12345678901234567000 (precision lost)
JSON.parse('{"id": 12345678901234567890}', (k, v, { source }) =>
  k === 'id' ? BigInt(source) : v).id;                   // 12345678901234567890n
```
</Info>

## ES2027 (finished, Stage 4, expected in the 2027 edition)

| Feature | Status | Snippet | Deep dive |
|---|---|---|---|
| Explicit Resource Management | Limited: Chrome 134, Firefox 141, no Safari | `using file = openFile(); await using conn = await db.connect();` | [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection) |
| Temporal | Limited: Firefox 139, Chrome/Edge 144, Node 26; **not in Safari**; types in TypeScript 6.0 | `Temporal.Now.plainDateISO().add({ days: 1 }).toString()` | below |
| `Atomics.pause` | | spin-wait hint for shared-memory code | |
| Joint Iteration | | iterate several iterators in lockstep | [Iterators, Generators & Iterator Helpers](/Software%20Engineering/Frontend/01%20JavaScript/Iterators%2C%20Generators%20%26%20Iterator%20Helpers) |

### Temporal in one minute

A modern date/time API replacing `Date`: immutable objects, explicit time zones and calendars, no 0-based months, separate types for different concepts.

| Type | Represents | Example use |
|---|---|---|
| `Temporal.Instant` | An exact moment (UTC nanoseconds) | Timestamps, logs |
| `Temporal.ZonedDateTime` | Moment + time zone | Flight departure in the airport's zone |
| `Temporal.PlainDate` | Calendar date, no time/zone | Birthdays, check-in date |
| `Temporal.PlainTime` / `PlainDateTime` | Wall-clock time without zone | Opening hours |
| `Temporal.Duration` | Length of time | "2h 35m" layovers |

```js
Temporal.Now.plainDateISO().add({ days: 1 }).toString();       // tomorrow, e.g. '2026-09-28'
Temporal.Now.instant().toZonedDateTimeISO('Asia/Dubai');         // now in Dubai

// Flight duration across time zones
const dep = Temporal.ZonedDateTime.from('2026-10-01T09:30[Asia/Dubai]');
const arr = Temporal.ZonedDateTime.from('2026-10-01T14:05[Europe/London]');
dep.until(arr).toString();   // 'PT7H35M' (Dubai UTC+4, London on BST UTC+1)
```

Because Safari doesn't ship it yet, use a polyfill (e.g. `@js-temporal/polyfill` or `temporal-polyfill`) in browser apps.

## Stage 3 to watch (not finished)

- **Import defer**: `import defer * as heavy from './heavy.js'` (TypeScript 5.9, Rspack 2.0). See [Modules (ESM)](/Software%20Engineering/Frontend/01%20JavaScript/Modules%20(ESM)).
- Source phase imports, import text, iterator chunking / includes / join, error stack accessor, await dictionary, RegExp buffer boundaries.
- Decorators: TypeScript has supported standard (2022-style) decorators since TS 5.0, but check the TC39 repo for the proposal's current stage before quoting it; it isn't in any published ES edition yet.

<Info>
**🆕 2025–2026**

- **ES2025** published June 2025; **ES2026** approved June 30, 2026.
- `using` and **Temporal** slipped to **ES2027**; both are Stage 4 but Safari support is missing.
- **Node 26** (2026-05-05) ships Temporal on by default and V8 14.6 (Upsert, `Iterator.concat`).
- **TypeScript 6.0** added the `es2025` lib, Temporal types and Map upsert types. See [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0).
</Info>

## Interview questions

<Accordion title="What are some features added to JavaScript in the last few years that you use?">
`?.`/`??` (ES2020), private class fields and top-level await (ES2022), `toSorted`/`with` (ES2023), `Object.groupBy` and `Promise.withResolvers` (ES2024), iterator helpers, Set methods and `Promise.try` (ES2025), `RegExp.escape`, `Array.fromAsync`, Map upsert (ES2026).
</Accordion>

<Accordion title="What is Temporal and why do we need it?">
A new built-in date/time API (Stage 4, ES2027) with immutable types, first-class time zones and calendars, and separate types for instants, zoned and plain dates. It fixes `Date`'s mutability, 0-based months, poor zone support and parsing quirks.
</Accordion>

<Accordion title="What does &quot;Stage 4&quot; mean vs &quot;Baseline&quot;?">
Stage 4 means TC39 finished the proposal and it will be in the next yearly spec. Baseline is about browser support: Newly available once all core browsers ship it, Widely available after 30 months.
</Accordion>

<Accordion title="How do you use a new feature that isn't in all browsers yet?">
Feature-detect, use a polyfill (core-js, Temporal polyfill) or transpile (TypeScript/Babel/SWC for syntax like `using`), and check Baseline/browserslist targets.
</Accordion>

## Related
- [JavaScript Index](/Software%20Engineering/Frontend/01%20JavaScript/JavaScript%20Index)
- [What's New in Frontend 2025-2026](/Software%20Engineering/Frontend/What's%20New%20in%20Frontend%202025-2026)
- [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0)
- [Modern Web Platform APIs 2025-2026](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Modern%20Web%20Platform%20APIs%202025-2026)
- [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability)
- [Iterators, Generators & Iterator Helpers](/Software%20Engineering/Frontend/01%20JavaScript/Iterators%2C%20Generators%20%26%20Iterator%20Helpers)
- [Collections - Map, Set, WeakMap & WeakRef](/Software%20Engineering/Frontend/01%20JavaScript/Collections%20-%20Map%2C%20Set%2C%20WeakMap%20%26%20WeakRef)
- [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection)

## Sources
- [tc39/proposals finished proposals](https://github.com/tc39/proposals/blob/main/finished-proposals.md), [tc39/proposals](https://github.com/tc39/proposals)
- [ECMAScript 2026 spec](https://tc39.es/ecma262/2026/), [InfoWorld: ECMAScript 2026 approved](https://www.infoworld.com/article/4193461/ecmascript-2026-specification-approved.html)
- [web-features explorer: explicit resource management](https://web-platform-dx.github.io/web-features-explorer/features/explicit-resource-management/), [Temporal](https://web-platform-dx.github.io/web-features-explorer/features/temporal/)
- [Node.js 26.0.0](https://nodejs.org/en/blog/release/v26.0.0)
- [MDN: Temporal](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal)
- [webstatus.dev](https://webstatus.dev/)
