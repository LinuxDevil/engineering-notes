---
title: "Collections - Map, Set, WeakMap & WeakRef"
tags: [frontend, frontend/javascript]
aliases: [Map, Set, WeakMap, WeakSet, WeakRef, FinalizationRegistry, Set vs Map]
updated: 2026-09-27
---
# Collections - Map, Set, WeakMap & WeakRef

<Note>
**TL;DR**

**`Map`** is a key→value collection where keys can be *any* value (objects included) and insertion order is kept. **`Set`** is a collection of *unique* values. Their **weak** twins (`WeakMap`, `WeakSet`) hold object keys *weakly*, so they don't prevent garbage collection, which is perfect for caches and private metadata. **`WeakRef`** and **`FinalizationRegistry`** give low-level, non-deterministic access to GC. ES2025 added set algebra (`union`, `intersection`, ...) and ES2026 added `getOrInsert` / `getOrInsertComputed` ("upsert").
</Note>

## Map vs plain object

| | `Map` | Object `{}` |
|---|---|---|
| Key types | **Any** value (objects, functions, `NaN`) | Strings and symbols (numbers become strings) |
| Order | Insertion order, guaranteed | Mostly insertion order, but integer-like keys come first |
| Size | `map.size` | `Object.keys(o).length` |
| Iteration | Directly iterable (`for...of`, `.entries()`) | Needs `Object.entries` |
| Prototype keys | None | Inherited keys like `toString` unless `Object.create(null)` |
| Frequent add/delete | Optimised for it | Slower in some engines |
| JSON | Not serialisable directly (`Object.fromEntries(map)`) | Native |

```js
const seatsByFlight = new Map();
const flight = { id: 'EK202' };
seatsByFlight.set(flight, ['12A', '12B']);   // object as key
seatsByFlight.get(flight);                   // ['12A', '12B']
seatsByFlight.has(flight);                   // true
seatsByFlight.delete(flight);

const m = new Map([['a', 1], ['b', 2]]);     // from entries
Object.fromEntries(m);                       // { a: 1, b: 2 }
new Map(Object.entries({ a: 1 }));           // object -> Map
```

<Tip>
**When to use a Map**

Dynamic keys (lookup tables, caches, counters), non-string keys, frequent insertion/deletion, or when you need reliable order and `.size`. Use plain objects for fixed-shape records and JSON data.
</Tip>

## Set

A collection of **unique** values (compared with SameValueZero, so `NaN` equals `NaN`; objects compare by reference).

```js
const tags = new Set(['beach', 'city', 'beach']);
tags.size;              // 2
tags.add('mountain');
tags.has('city');       // true
[...new Set(array)];    // classic de-duplication
```

### Set methods (ES2025)

Baseline Newly available since 2024-06-11.

| Method | Returns | Venn meaning |
|---|---|---|
| `a.union(b)` | new Set | in A or B |
| `a.intersection(b)` | new Set | in both |
| `a.difference(b)` | new Set | in A, not in B |
| `a.symmetricDifference(b)` | new Set | in exactly one |
| `a.isSubsetOf(b)` | boolean | every A in B |
| `a.isSupersetOf(b)` | boolean | every B in A |
| `a.isDisjointFrom(b)` | boolean | nothing shared |

```js
const selected = new Set(['EK', 'QR']);
const available = new Set(['QR', 'EY', 'FZ']);
selected.intersection(available);   // Set { 'QR' }
selected.union(available);          // Set { 'EK', 'QR', 'EY', 'FZ' }
selected.difference(available);     // Set { 'EK' }
```

The argument must be "set-like" (has `size`, `has`, `keys`), so a `Map` works too (its keys are used).

<Info>
**Combining Set and Map (union), from the owner's interview notes**

```js
// Before ES2025: spread
const union = new Set([...setA, ...setB]);
const mergedMap = new Map([...mapA, ...mapB]);  // later keys win

// Now
const union2 = setA.union(setB);
const commonKeys = new Set(mapA.keys()).intersection(mapB);  // Map is set-like (keys)
```
</Info>

## Upsert: `getOrInsert` / `getOrInsertComputed` (ES2026)

Baseline Newly available since 2026-02-14 (in Node 26 / V8 14.6; TypeScript 6.0 ships the types). Replaces the "check, set, get" dance.

```js
// before
if (!groups.has(key)) groups.set(key, []);
groups.get(key).push(item);

// ES2026
groups.getOrInsert(key, []).push(item);

// Lazily compute the default only when missing
const counts = new Map();
for (const w of words) counts.set(w, counts.getOrInsert(w, 0) + 1);

cache.getOrInsertComputed(url, key => expensiveParse(key));
```

Also available on `WeakMap`.

## WeakMap and WeakSet

- Keys must be **objects** (or non-registered symbols since ES2023), not primitives.
- References to keys are **weak**: if nothing else references the key object, it can be garbage collected, and its entry disappears.
- **Not iterable**, no `size`, no `clear()`: the contents depend on GC timing, so they can't be observed.

| Use case | Why weak |
|---|---|
| Caching results per object (memoization) | Cache entries die with the object, no leak |
| Private data per instance (pre-`#private`) | Data is released with the instance |
| Metadata on DOM nodes | Removed nodes don't stay alive because of your map |
| `WeakSet` to mark "already visited/processed" objects | E.g. cycle detection in deep clone |

<Info>
**Per-object memoization without leaks**

```js
const cache = new WeakMap();
function layoutFor(node) {
  if (cache.has(node)) return cache.get(node);
  const result = computeExpensiveLayout(node);
  cache.set(node, result);
  return result;
}
// When `node` is removed from the DOM and dropped, its cache entry can be collected.
```
</Info>

<Warning>
**Map-as-cache leak**

A regular `Map` keyed by DOM nodes or component instances keeps them alive forever (a classic SPA memory leak). Use `WeakMap`, or delete entries explicitly. See [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection).
</Warning>

## WeakRef and FinalizationRegistry (ES2021)

- **`WeakRef(obj)`** holds a weak reference; `ref.deref()` returns the object or `undefined` if it was collected.
- **`FinalizationRegistry(cb)`** lets you register a callback that *may* run some time after an object is collected (e.g. to clean up a cache entry keyed by a string).

```js
const cache = new Map();                          // string key -> WeakRef(value)
const registry = new FinalizationRegistry(key => {
  if (!cache.get(key)?.deref()) cache.delete(key);  // clean up the dead entry
});

function getImage(url) {
  const cached = cache.get(url)?.deref();
  if (cached) return cached;
  const img = decodeImage(url);
  cache.set(url, new WeakRef(img));
  registry.register(img, url);
  return img;
}
```

<Warning>
**Use sparingly**

GC is non-deterministic: an object may be collected late or never, and finalizers may never run (e.g. when the page closes). Never use them for essential logic (saving data, releasing locks). For deterministic cleanup use `try/finally` or `using` (see [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection)).
</Warning>

## Quick decision table

| Need | Use |
|---|---|
| Unique values, membership checks, set algebra | `Set` |
| Dynamic keys, any key type, counters, lookup tables | `Map` |
| Attach data to objects you don't own, without leaks | `WeakMap` |
| Tag objects (visited, initialised) | `WeakSet` |
| Cache that may be dropped under memory pressure | `WeakRef` (+ `FinalizationRegistry`) |
| Fixed-shape record / JSON | plain object |

<Info>
**🆕 2025–2026**

- **Set methods**: part of **ES2025** (Baseline Newly available 2024-06-11).
- **Upsert (`getOrInsert`, `getOrInsertComputed`)** on `Map`/`WeakMap`: **ES2026**, Baseline Newly available 2026-02-14; types in TypeScript 6.0.
- Grouping into a Map: `Map.groupBy` (ES2024), see [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability).
</Info>

## Interview questions

<Accordion title="What is the difference between a Set and a Map?">
A Set stores unique values; a Map stores key→value pairs. Both keep insertion order, accept any value type, have `size` and are iterable.
</Accordion>

<Accordion title="When would you use a Map instead of an object?">
When keys are dynamic or non-strings (e.g. objects), when you add/remove often, or need reliable order and size. Objects suit fixed-shape records and JSON.
</Accordion>

<Accordion title="What is a WeakMap and why isn't it iterable?">
A map with object keys held weakly, so entries vanish when the key is garbage-collected. Because GC timing is unobservable by design, it has no iteration, `size` or `clear`.
</Accordion>

<Accordion title="Give a real use case for WeakMap.">
Caching computed data per DOM node or object instance, or storing private per-instance data, without preventing garbage collection.
</Accordion>

<Accordion title="How do you get the union or intersection of two sets?">
ES2025: `a.union(b)`, `a.intersection(b)` (also `difference`, `symmetricDifference`, `isSubsetOf`...). Before: `new Set([...a, ...b])` and `new Set([...a].filter(x => b.has(x)))`.
</Accordion>

<Accordion title="What are WeakRef and FinalizationRegistry? Should you use them?">
`WeakRef` holds an object without keeping it alive; `deref()` may return undefined. `FinalizationRegistry` runs a callback maybe-after collection. Only for caches/optimisations; never for program correctness, because GC timing is non-deterministic.
</Accordion>

## Related
- [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection)
- [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability)
- [Iterators, Generators & Iterator Helpers](/Software%20Engineering/Frontend/01%20JavaScript/Iterators%2C%20Generators%20%26%20Iterator%20Helpers)
- [Modern JavaScript ES2022-ES2027](/Software%20Engineering/Frontend/01%20JavaScript/Modern%20JavaScript%20ES2022-ES2027)
- [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions)

## Sources
- MDN: [Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map), [Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set), [WeakMap](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap), [WeakRef](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef), [FinalizationRegistry](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry)
- [tc39/proposals finished proposals](https://github.com/tc39/proposals/blob/main/finished-proposals.md)
- [TypeScript 6.0 announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-6-0/)
- [Node.js 26.0.0](https://nodejs.org/en/blog/release/v26.0.0)
