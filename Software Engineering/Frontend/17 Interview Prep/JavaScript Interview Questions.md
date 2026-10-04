---
title: "JavaScript Interview Questions"
tags: [frontend, frontend/interview, javascript, interview]
aliases: [JS Interview Questions, JS Questions, JavaScript Q&A]
updated: 2026-09-27
---
# JavaScript Interview Questions

<Note>
**TL;DR**

Model answers for every JavaScript, TypeScript and general web question from the owner's "JS Questions" list and the `Interview` notes, grouped by theme. Each answer is short enough to say out loud in 30–60 seconds, with code where it helps and a link to the topic note for depth. React/Redux/React Native questions are in [React Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/React%20Interview%20Questions); "what does this log" puzzles are in [Output Questions & Tricky Snippets](/Software%20Engineering/Frontend/17%20Interview%20Prep/Output%20Questions%20%26%20Tricky%20Snippets).
</Note>

**How to use this note**: cover the answer, say yours out loud, then open the callout. If you are unsure, read the linked topic note.

## 1. Language basics

<Accordion title="What is JavaScript?">
A high-level, dynamically typed, garbage-collected language with first-class functions and prototype-based objects, standardized as ECMAScript (ES2025 published June 2025, ES2026 approved June 2026). It is **single-threaded** with an **event loop** for async work, and runs in browsers (with Web APIs like DOM, fetch, timers) and on servers (Node, Deno, Bun). The engine (V8, SpiderMonkey, JavaScriptCore) parses, JIT-compiles and runs it. See [JavaScript Index](/Software%20Engineering/Frontend/01%20JavaScript/JavaScript%20Index), [Execution Context & Call Stack](/Software%20Engineering/Frontend/01%20JavaScript/Execution%20Context%20%26%20Call%20Stack).
</Accordion>

<Accordion title="What is the difference between `var`, `let` and `const`?">
| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisted | Yes, initialized to `undefined` | Yes, but uninitialized (TDZ) | Yes, but uninitialized (TDZ) |
| Redeclare in same scope | Allowed | Error | Error |
| Reassign | Yes | Yes | No (binding is constant; the object it points to can still be mutated) |
| Global object property | `var` at top level creates `window.x` | No | No |
```js
{
  var a = 1; let b = 2; const c = { n: 3 };
}
a;        // 1 (var leaks out of the block)
// b;     // ReferenceError
const o = { n: 1 }; o.n = 2;  // fine: mutation, not reassignment
```
See [Hoisting, TDZ & var-let-const](/Software%20Engineering/Frontend/01%20JavaScript/Hoisting%2C%20TDZ%20%26%20var-let-const).
</Accordion>

<Accordion title="What is hoisting?">
During the creation phase of an execution context, declarations are registered before any code runs. Function declarations are hoisted with their body (callable before the line). `var` is hoisted and set to `undefined`. `let`/`const`/`class` are hoisted but stay uninitialized until their declaration line: accessing them earlier throws a `ReferenceError`. That window is the **Temporal Dead Zone (TDZ)**. Code is not physically "moved to the top"; it is how the engine sets up memory first.
```js
console.log(x); // undefined
var x = 1;
console.log(y); // ReferenceError: Cannot access 'y' before initialization (TDZ)
let y = 2;
hi();           // works
function hi() {}
```
</Accordion>

<Accordion title="What is the Temporal Dead Zone?">
The time between entering a scope and the `let`/`const`/`class` declaration being evaluated. The binding exists (so it shadows outer variables) but reading or writing it throws. It catches use-before-declare bugs that `var` silently allowed.
</Accordion>

<Accordion title="What are the primitive and non-primitive types?">
Primitives (immutable, compared by value): `string`, `number`, `bigint`, `boolean`, `undefined`, `null`, `symbol`. Everything else is an **object** (compared by reference): plain objects, arrays, functions, dates, maps, sets, etc. `typeof null === 'object'` is a historic bug. See [Types, Coercion & Equality](/Software%20Engineering/Frontend/01%20JavaScript/Types%2C%20Coercion%20%26%20Equality).
</Accordion>

<Accordion title="`null` vs `undefined`?">
`undefined`: a variable was declared but not assigned, a missing property, a missing argument, or a function with no return. `null`: an intentional "no value" set by the programmer. `null == undefined` is `true`, `null === undefined` is `false`. Use `??` to default only on these two (`count ?? 0` keeps `0`, unlike `||`). (The original note said `null` "has a memory address"; that is not a useful distinction: both are primitive values.)
</Accordion>

<Accordion title="How do scopes work?">
JS uses **lexical scope**: where a function is written decides which variables it can see. Scopes: global, module, function, and block (`let`/`const` inside `{}`). Lookups walk outward through the scope chain until found, else `ReferenceError`. See [Scope & Lexical Environment](/Software%20Engineering/Frontend/01%20JavaScript/Scope%20%26%20Lexical%20Environment).
</Accordion>

<Accordion title="What is mutable vs immutable?">
Primitives are immutable: operations create new values (`str.toUpperCase()` returns a new string). Objects and arrays are mutable: you can change them in place, and every reference sees the change. Immutable updates create new objects instead (`{...obj, a: 1}`, `arr.toSorted()`), which makes change detection cheap (reference comparison) for React/Redux. See [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability).
```js
const a = [3, 1, 2];
const b = a;       b.push(4);         // a is [3,1,2,4] too (same reference)
const c = a.toSorted();               // new array, a untouched (ES2023)
```
</Accordion>

<Accordion title="Is JS pass-by-value or pass-by-reference? How do you &quot;call by reference&quot;?">
Always **pass-by-value**, but for objects the value *is a reference*. So a function can mutate the object you passed, but reassigning the parameter doesn't affect the caller. To get "by reference" behaviour, pass an object/array and mutate it (or return the new value).
```js
function mutate(o) { o.n = 2; }      // caller sees n = 2
function reassign(o) { o = { n: 3 }; } // caller unaffected
const obj = { n: 1 }; mutate(obj); reassign(obj); obj.n; // 2
```
</Accordion>

<Accordion title="What is `Object.freeze`?">
Makes an object's own properties non-writable and non-configurable and prevents adding/removing properties. It is **shallow**: nested objects stay mutable. In strict mode writes throw; in sloppy mode they fail silently. `Object.seal` allows changing existing values but not adding/removing; `Object.preventExtensions` only blocks additions.
```js
const cfg = Object.freeze({ api: { url: '/v1' } });
cfg.x = 1;          // ignored (TypeError in strict mode)
cfg.api.url = '/v2' // works: freeze is shallow
function deepFreeze(o) { Object.values(o).forEach(v => typeof v === 'object' && v !== null && deepFreeze(v)); return Object.freeze(o); }
```
</Accordion>

<Accordion title="How do you loop over an object? What is `for...in`?">
`for...in` iterates **enumerable string keys, including inherited ones** (guard with `Object.hasOwn`). Prefer `Object.keys/values/entries` with `for...of` for own properties. `for...of` iterates **values of iterables** (arrays, strings, Map, Set), not plain objects.
```js
for (const [key, value] of Object.entries(user)) console.log(key, value);
for (const k in user) if (Object.hasOwn(user, k)) console.log(k);
```
</Accordion>

<Accordion title="What is the spread operator?">
`...` expands an iterable (in arrays/calls) or an object's own enumerable properties (in object literals). The opposite, **rest**, collects the remaining items into an array/object. Spread makes **shallow** copies.
```js
const merged = { ...defaults, ...options };  // later keys win
const all = [...a, ...b];
Math.max(...nums);
const { id, ...rest } = user;                 // rest syntax
```
</Accordion>

<Accordion title="What is a shallow clone vs a deep clone?">
A shallow clone copies top-level properties; nested objects are still shared. A deep clone copies all levels. Use `structuredClone(value)` (built in; handles Dates, Maps, Sets, cycles; not functions, DOM nodes or class prototypes). `JSON.parse(JSON.stringify(x))` loses `undefined`, functions, Dates (become strings), Maps and fails on cycles.
```js
const shallow = { ...orig };          // or Object.assign({}, orig)
const deep = structuredClone(orig);
shallow.nested === orig.nested;       // true
deep.nested === orig.nested;          // false
```
</Accordion>

<Accordion title="How do you remove all references to an object with infinite levels of nesting?">
Two readings, both worth mentioning:
1. **Break shared references** (so edits don't leak back): deep clone with `structuredClone`, which also handles circular references.
2. **Let it be garbage-collected**: drop every reference you hold (set variables to `null`, remove from arrays/maps/caches, remove event listeners, clear timers). The GC's mark-and-sweep will reclaim an object graph, cycles included, once nothing reachable points to it. Use `WeakMap`/`WeakRef` for caches so you don't keep objects alive. See [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection), [Collections - Map, Set, WeakMap & WeakRef](/Software%20Engineering/Frontend/01%20JavaScript/Collections%20-%20Map%2C%20Set%2C%20WeakMap%20%26%20WeakRef).
</Accordion>

<Accordion title="`==` vs `===`?">
`===` (strict) compares type and value with no conversion. `==` (loose) converts types first using coercion rules (`'1' == 1`, `null == undefined`, `0 == ''` all `true`). Use `===` except the idiom `x == null` to check both `null` and `undefined`. `Object.is` is like `===` but `Object.is(NaN, NaN)` is `true` and `Object.is(0, -0)` is `false`. See [Types, Coercion & Equality](/Software%20Engineering/Frontend/01%20JavaScript/Types%2C%20Coercion%20%26%20Equality).
</Accordion>

<Accordion title="Why is a JS array &quot;not good enough&quot; (for some jobs)?">
JS arrays are objects with integer-like keys and a `length`, not fixed-type contiguous memory. Consequences: they can be sparse (holes), hold mixed types, `includes`/`indexOf` are O(n), `shift`/`unshift` are O(n), and numeric workloads are slower than typed memory. Better tools: `Set`/`Map` for O(1) lookups and uniqueness, **typed arrays** (`Float64Array`, `Uint8Array`, `Float16Array` from ES2025) for binary/numeric data, and a real queue/deque for heavy queueing. Engines optimize "packed" arrays of one element kind, so avoid holes and mixed types in hot code.
</Accordion>

## 2. Functions, `this` and closures

<Accordion title="Classic (function declaration/expression) vs arrow functions vs IIFE?">
| | Regular function | Arrow function |
|---|---|---|
| `this` | Dynamic: decided by how it is called | Lexical: taken from the surrounding scope |
| `arguments` | Yes | No (use rest `...args`) |
| `new` / `prototype` | Can be a constructor | Cannot |
| Hoisting | Declarations are hoisted | Behaves like the variable it is assigned to |
| Good for | Methods, constructors, generators | Callbacks, keeping outer `this` |
An **IIFE** (Immediately Invoked Function Expression) runs as soon as it is defined; it was used to create a private scope before modules and block scope. "Inline function" usually means a function defined where it is used (e.g., `onClick={() => ...}`).
```js
(function () { const secret = 1; })();   // IIFE
const counter = { n: 0, inc() { setTimeout(() => this.n++, 0); } }; // arrow keeps `this`
```
See [Functions - HOF, Callbacks & Currying](/Software%20Engineering/Frontend/01%20JavaScript/Functions%20-%20HOF%2C%20Callbacks%20%26%20Currying), [this, call, apply & bind](/Software%20Engineering/Frontend/01%20JavaScript/this%2C%20call%2C%20apply%20%26%20bind).
</Accordion>

<Accordion title="How does `this` work?">
Decided at call time (except arrows). Rules in priority order: `new` binding (new object) → explicit (`call`/`apply`/`bind`) → implicit (`obj.method()` → `obj`) → default (`undefined` in strict mode/modules, global object in sloppy mode). Arrow functions ignore all of these and use the enclosing `this`. Class bodies are strict, so a detached method gets `undefined`.
</Accordion>

<Accordion title="When should you use `bind()`?">
When you pass a method as a callback and need `this` fixed to its object (event handlers, `setTimeout`, class component handlers), or to partially apply arguments. `call`/`apply` invoke immediately (apply takes an array); `bind` returns a new function.
```js
const btn = { label: 'Pay', click() { console.log(this.label); } };
setTimeout(btn.click, 0);             // undefined (this lost)
setTimeout(btn.click.bind(btn), 0);   // 'Pay'
const add = (a, b) => a + b;
const add10 = add.bind(null, 10);     // partial application
```
</Accordion>

<Accordion title="What is a closure?">
A function plus the lexical environment it was created in. The function keeps access to outer variables even after the outer function returned (the owner's "backpack" analogy: the function carries its surrounding data with it). Used for data privacy, factories, memoization, debounce, and React hooks. See [Closures & Memory Model](/Software%20Engineering/Frontend/01%20JavaScript/Closures%20%26%20Memory%20Model), [Closures](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/Closures).
```js
function makeCounter() { let count = 0; return () => ++count; }
const next = makeCounter(); next(); next(); // 2
```
</Accordion>

<Accordion title="What is a higher-order function (and a higher-order component)?">
A higher-order function takes a function as an argument and/or returns one: `map`, `filter`, `reduce`, `debounce`, `memoize`. A **higher-order component** (React) is the same idea for components: a function that takes a component and returns an enhanced component (`withAuth(Page)`). Today hooks usually replace HOCs. See [Functions - HOF, Callbacks & Currying](/Software%20Engineering/Frontend/01%20JavaScript/Functions%20-%20HOF%2C%20Callbacks%20%26%20Currying), [Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns).
</Accordion>

<Accordion title="What is a pure function?">
Same inputs → same output, and no side effects (no mutation of arguments/outer state, no I/O, no randomness/time). Easy to test, cache and reason about. Reducers and React render functions must be pure.
```js
const add = (a, b) => a + b;                 // pure
const push = (arr, x) => { arr.push(x); };    // impure: mutates input
const append = (arr, x) => [...arr, x];       // pure version
```
</Accordion>

<Accordion title="What is a &quot;void function&quot; and what are &quot;function references&quot;?">
A void function returns nothing, so calling it yields `undefined` (the `void` operator also evaluates an expression and returns `undefined`, e.g. `void 0`). A function reference is the function value itself (`handleClick`) as opposed to calling it (`handleClick()`). Classic bug: `onClick={handleClick()}` calls it during render; `onClick={handleClick}` passes the reference. In TypeScript, a callback typed `() => void` may return a value that is simply ignored.
</Accordion>

<Accordion title="`map` vs `forEach`?">
`map` returns a **new array** of the callback results (same length) and is chainable; use it to transform. `forEach` returns `undefined` and is for side effects. Neither can `break` (use `for...of`, `some`, or `find`). Neither awaits async callbacks: use `for...of` with `await` or `Promise.all(arr.map(async ...))`.
</Accordion>

<Accordion title="What is currying?">
Transforming `f(a, b, c)` into `f(a)(b)(c)`. Useful for partial application and building specialized functions (`const withTax = rate => price => price * (1 + rate)`).
</Accordion>

## 3. Asynchronous JavaScript and the event loop

<Accordion title="Is JavaScript synchronous or asynchronous?">
The language runs **synchronously on a single thread** (one call stack). Asynchrony comes from the **host environment**: timers, network, and I/O are handled by Web APIs/Node's libuv, and their callbacks are queued and run later by the **event loop** when the stack is empty. So: single-threaded, non-blocking via the event loop. See [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks).
</Accordion>

<Accordion title="What is the message queue / event loop? Micro vs macro tasks?">
- **Call stack**: where functions execute (LIFO).
- **Web APIs**: timers, fetch, DOM events run outside the engine.
- **Macrotask (task) queue**: `setTimeout`, `setInterval`, I/O, UI events, `MessageChannel`.
- **Microtask queue**: promise reactions (`.then`, `await` continuations), `queueMicrotask`, `MutationObserver`.
- **Event loop**: take **one** macrotask → run it to completion → **drain all microtasks** → the browser may render (style/layout/paint, with `requestAnimationFrame` callbacks just before) → repeat.
```js
console.log('Script Start');
setTimeout(() => console.log('setTimeout callback'), 0);
Promise.resolve().then(() => console.log('Promise resolved'));
console.log('Script End');
// Script Start, Script End, Promise resolved, setTimeout callback
```
Correction to a common claim: rendering does not happen "in the microtask step"; it happens between tasks, after microtasks are drained. A never-ending microtask chain blocks rendering. Full walkthrough in [Output Questions & Tricky Snippets](/Software%20Engineering/Frontend/17%20Interview%20Prep/Output%20Questions%20%26%20Tricky%20Snippets).
</Accordion>

<Accordion title="How do you run code after some tasks are done?">
- One async task: `await task()` or `.then()`.
- Several in parallel, need all: `await Promise.all([...])` (or `allSettled` if some may fail).
- Always run cleanup: `.finally()` or `try/finally`.
- After the current sync code: `queueMicrotask(fn)`; after rendering: `requestAnimationFrame` then `setTimeout`; when idle: `requestIdleCallback`/`scheduler.postTask`.
- Events from other code: callbacks, `EventTarget`/custom events, or the observer pattern.
</Accordion>

<Accordion title="What is a Promise? What are its states?">
An object representing the eventual result of an async operation. States: **pending** → **fulfilled** (with a value) or **rejected** (with a reason). Once settled it never changes. `.then/.catch/.finally` register reactions that run as microtasks and return new promises, which enables chaining. See [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation).
</Accordion>

<Accordion title="What is callback hell and how do you avoid it?">
Deeply nested callbacks for sequential async steps ("pyramid of doom"): hard to read, error handling is repeated at each level, and control flow (loops, early return) is painful. Fix with promises and `async/await`, named functions, and small composable steps.
```js
// hell
getUser(id, u => getOrders(u, os => getItems(os[0], items => render(items), onErr), onErr), onErr);
// flat
try { const u = await getUser(id); const os = await getOrders(u); render(await getItems(os[0])); }
catch (e) { onErr(e); }
```
</Accordion>

<Accordion title="How do you deal with a promise? Give an example.">
```js
async function loadUser(id) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), 5000);   // or AbortSignal.timeout(5000)
  try {
    const res = await fetch(`/api/users/${id}`, { signal: controller.signal });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);        // fetch only rejects on network errors
    return await res.json();
  } catch (err) {
    if (err.name === 'AbortError') return null;
    throw err;
  } finally {
    clearTimeout(timer);
  }
}
loadUser(1).then(render).catch(showError);
```
Key points: always handle rejection, check `res.ok`, support cancellation, clean up in `finally`.
</Accordion>

<Accordion title="Write a Promise from scratch (and return a value from a callback).">
```js
class MyPromise {
  #state = 'pending'; #value; #handlers = [];
  constructor(executor) {
    const settle = (state, value) => {
      if (this.#state !== 'pending') return;
      if (state === 'fulfilled' && value && typeof value.then === 'function') {
        return value.then(v => settle('fulfilled', v), e => settle('rejected', e)); // adopt thenables
      }
      this.#state = state; this.#value = value;
      this.#handlers.forEach(h => h());
    };
    try { executor(v => settle('fulfilled', v), e => settle('rejected', e)); }
    catch (e) { settle('rejected', e); }
  }
  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const run = () => queueMicrotask(() => {
        const cb = this.#state === 'fulfilled' ? onFulfilled : onRejected;
        if (typeof cb !== 'function') {
          return this.#state === 'fulfilled' ? resolve(this.#value) : reject(this.#value);
        }
        try { resolve(cb(this.#value)); } catch (e) { reject(e); }
      });
      this.#state === 'pending' ? this.#handlers.push(run) : run();
    });
  }
  catch(onRejected) { return this.then(undefined, onRejected); }
  finally(fn) { return this.then(v => { fn(); return v; }, e => { fn(); throw e; }); }
  static resolve(v) { return new MyPromise(r => r(v)); }
}

// Promisifying a callback API so the callback's value is returned:
const readFileAsync = path => new Promise((resolve, reject) =>
  fs.readFile(path, 'utf8', (err, data) => (err ? reject(err) : resolve(data))));
const text = await readFileAsync('a.txt');
```
</Accordion>

<Accordion title="`Promise.all` vs `allSettled` vs `race` vs `any`?">
| Method | Resolves when | Rejects when |
|---|---|---|
| `all` | All fulfill → array of values (in input order) | **Any** rejects (fail fast) |
| `allSettled` | All settle → `[{status, value/reason}]` | Never |
| `race` | First to **settle** (fulfill or reject) | If that first one rejects |
| `any` | First to **fulfill** | All reject → `AggregateError` |
Use `all` when every result is required, `allSettled` for independent widgets/suppliers, `race` for timeouts, `any` for "fastest mirror". Also new: `Promise.try` (ES2025) and `Promise.withResolvers` (ES2024).
</Accordion>

<Accordion title="What does `async/await` do?">
`async` functions always return a promise. `await` pauses that function (not the thread) until the promise settles; the rest of the function continues as a microtask. Errors become exceptions you catch with `try/catch`. Run independent awaits in parallel with `Promise.all`, not one after another.
</Accordion>

<Accordion title="What is `setTimeout(fn, 0)` really?">
It schedules `fn` as a macrotask after at least 0 ms (browsers clamp nested timers to ≥4 ms after 5 levels). It runs after the current script and all microtasks, and possibly after a render.
</Accordion>

## 4. Objects, collections and data

<Accordion title="`Set` vs `Map` (and how to union them)?">
`Set`: unique values, O(1) `has`. `Map`: key → value with **any type of key**, insertion order kept, `size`, and better performance for frequent add/delete than plain objects. ES2025 added Set methods:
```js
const a = new Set([1, 2]), b = new Set([2, 3]);
a.union(b);          // Set {1,2,3}   (older: new Set([...a, ...b]))
a.intersection(b);   // Set {2}
a.difference(b);     // Set {1}
const m1 = new Map([['x', 1]]), m2 = new Map([['y', 2]]);
const merged = new Map([...m1, ...m2]); // union of maps, later wins
```
See [Collections - Map, Set, WeakMap & WeakRef](/Software%20Engineering/Frontend/01%20JavaScript/Collections%20-%20Map%2C%20Set%2C%20WeakMap%20%26%20WeakRef).
</Accordion>

<Accordion title="How are objects and prototypes related?">
Every object has an internal `[[Prototype]]` link. Property lookup walks the chain until found or `null`. `class` is syntax over constructor functions + prototypes. See [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance).
</Accordion>

<Accordion title="How do you handle dates (in JS, React or React Native)?">
Store and send **ISO 8601 UTC** strings or epoch ms; convert to local time only for display. Format with `Intl.DateTimeFormat` (locale + `timeZone`), relative times with `Intl.RelativeTimeFormat`. For arithmetic and time zones use a library (`date-fns` + `date-fns-tz`, Day.js, Luxon). **Temporal** (Stage 4, targeting ES2027) fixes the `Date` API and has shipped in Firefox 139 and Chrome 144 but not Safari, so use a polyfill or library for now. In React Native, Hermes supports `Intl` on modern versions; check the specific `Intl` features you use.
</Accordion>

## 5. Networking and APIs (general)

<Accordion title="Axios or fetch?">
`fetch` is built into browsers and Node 18+, promise-based and streaming-capable, but it only rejects on network errors (check `res.ok`), needs manual JSON and timeouts (`AbortSignal.timeout`), and has no interceptors. Axios adds interceptors, automatic JSON, timeouts, upload progress (XHR in browsers) and consistent errors, at the cost of bundle size. Many teams write a thin `fetch` wrapper; RTK Query/TanStack Query sit on top of either.
</Accordion>

<Accordion title="What are interceptors? How do you send a token with every API call?">
Interceptors are hooks that run on every request/response: attach auth headers, log, transform errors, refresh expired tokens and retry.
```js
// axios
api.interceptors.request.use(cfg => { cfg.headers.Authorization = `Bearer ${getToken()}`; return cfg; });
api.interceptors.response.use(r => r, async err => {
  if (err.response?.status === 401 && !err.config._retry) {
    err.config._retry = true; await refreshToken(); return api(err.config);
  }
  throw err;
});
// fetch wrapper
export const http = (url, opts = {}) =>
  fetch(url, { ...opts, headers: { ...opts.headers, Authorization: `Bearer ${getToken()}` } });
```
Prefer **httpOnly cookies** for session tokens where possible (not readable by JS, so XSS can't steal them). See [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage), [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies).
</Accordion>

<Accordion title="What is an HTTP handler?">
Server side: the function that receives a request and produces a response (Express `(req, res) => {}`, a Next.js Route Handler `export async function GET(request) {}`, a service worker `fetch` event handler). Client side, people sometimes mean the central module that performs HTTP calls and handles errors (the fetch wrapper/interceptors above).
</Accordion>

<Accordion title="How do you monitor API traffic?">
Browser DevTools Network panel (filter, throttling, HAR export, initiator), React Native via Flipper successors/React Native DevTools or a proxy (Charles, Proxyman, mitmproxy). In code: interceptors that log timing and errors, `PerformanceObserver` for `resource` entries, and in production RUM/APM (Sentry, Datadog) with tracing headers. MSW (Mock Service Worker) to inspect and mock in dev/tests.
</Accordion>

## 6. TypeScript

<Accordion title="What is TypeScript?">
A typed superset of JavaScript by Microsoft. Types are checked at compile time and erased at runtime. Benefits: catch bugs early, better autocomplete/refactoring, self-documenting APIs. TS 7.0 (July 2026) is the native Go port, about 8–12x faster builds. Node can now run erasable TS directly (type stripping). See [TypeScript Essentials](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%20Essentials), [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0).
</Accordion>

<Accordion title="What are generics?">
Type parameters that let a function/type work with many types while keeping the relationship between them.
```ts
function first<T>(arr: T[]): T | undefined { return arr[0]; }
const n = first([1, 2]);            // number | undefined
type ApiResponse<T> = { data: T; error?: string };
function pluck<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }
```
See [Advanced TypeScript - Generics & Utility Types](/Software%20Engineering/Frontend/02%20TypeScript/Advanced%20TypeScript%20-%20Generics%20%26%20Utility%20Types).
</Accordion>

<Accordion title="`interface` vs `type`?">
Both describe object shapes. Interfaces can be merged (declaration merging) and `extends`; types can express unions, intersections, mapped and conditional types. Common rule: `interface` for public object contracts, `type` for everything else.
</Accordion>

## 7. Browser, HTML and CSS (general web)

<Accordion title="What is the Critical Rendering Path and how do you optimize it?">
The sequence the browser follows to turn HTML, CSS and JS into pixels: parse HTML → DOM; parse CSS → CSSOM; combine → render tree; layout; paint; composite. CSS is render-blocking and synchronous scripts are parser-blocking. Optimize by inlining critical CSS, deferring non-critical CSS/JS (`defer`, `async`, `type="module"`), preloading key resources (LCP image, fonts), reducing bytes (compression, minification, code splitting) and round trips. Tools: Lighthouse, DevTools Performance panel, WebPageTest. See [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path).
</Accordion>

<Accordion title="`defer` vs `async` vs prefetch/preload?">
- `<script defer>`: download in parallel, execute **after parsing, in order**, before `DOMContentLoaded`. Default choice for app scripts (module scripts are deferred by default).
- `<script async>`: download in parallel, execute **as soon as downloaded**, order not guaranteed. For independent scripts (analytics).
- `<link rel="preload">`: fetch a resource needed **for this page** early, with high priority.
- `<link rel="prefetch">`: fetch a resource likely needed **for the next navigation**, low priority. (Speculation Rules can prefetch/prerender whole pages in Chromium.)
</Accordion>

<Accordion title="What is semantic HTML and why is it important?">
Using elements for their meaning (`<header>`, `<nav>`, `<main>`, `<article>`, `<button>`, `<label>`) rather than generic `<div>`s. It gives accessibility for free (roles, keyboard support, the accessibility tree), better SEO, and more maintainable code. See [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials).
</Accordion>

<Accordion title="What is the accessibility tree? Which tools analyze accessibility?">
A tree derived from the DOM that browsers expose to assistive technologies, with each node's role, name, state and value. Tools: DevTools Accessibility pane and Lighthouse, axe DevTools, WAVE, screen readers (VoiceOver, NVDA), keyboard-only testing, and Playwright/axe in CI.
</Accordion>

<Accordion title="How is HTML stored in memory?">
Not as a string: the parser builds the **DOM**, a tree of node objects (elements, text, comments) with parent/child/sibling links, exposed to JS through the DOM API. See [DOM, Events & Event Delegation](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/DOM%2C%20Events%20%26%20Event%20Delegation).
</Accordion>

<Accordion title="What is the box model? What is flexbox?">
Box model: content + padding + border + margin; `box-sizing: border-box` makes `width` include padding and border. Flexbox is a one-dimensional layout model (row or column) for distributing space and aligning items (`justify-content` on the main axis, `align-items` on the cross axis, `flex: grow shrink basis`). Grid is the two-dimensional counterpart. See [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts), [Layout - Flexbox, Grid & Container Queries](/Software%20Engineering/Frontend/04%20CSS/Layout%20-%20Flexbox%2C%20Grid%20%26%20Container%20Queries).
</Accordion>

<Accordion title="What is CSS specificity?">
The weight used to pick between conflicting rules: (inline styles) > (ids) > (classes, attributes, pseudo-classes) > (elements, pseudo-elements). Compared left to right; ties go to the later rule. `!important` and cascade layers (`@layer`) sit above specificity in the cascade.
</Accordion>

<Accordion title="What is the CSSOM and how is it evaluated?">
The CSS Object Model: the browser parses all stylesheets into a tree of rules, then computes styles per element (cascade, inheritance, specificity). Because layout needs final styles, CSS blocks rendering until the CSSOM is ready.
</Accordion>

<Accordion title="How do you use the GPU instead of the CPU (in CSS)?">
Animate only `transform` and `opacity`, which can run on the compositor thread without layout/paint. Hint with `will-change: transform` sparingly (each layer costs memory). Avoid animating `top/left/width/height`. See [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook).
</Accordion>

<Accordion title="What is BEM? What is functional CSS?">
BEM (Block__Element--Modifier) is a naming convention that keeps specificity flat and makes relationships clear (`.card__title--large`). Functional/utility-first CSS (Tailwind) composes small single-purpose classes in markup (`flex gap-4 text-sm`): no naming, tiny CSS that doesn't grow with features, consistent design tokens; the cost is verbose markup. See [Styling Approaches](/Software%20Engineering/Frontend/04%20CSS/Styling%20Approaches).
</Accordion>

<Accordion title="What is the Observer pattern?">
A subject (observable) keeps a list of observers and notifies them when something happens. `addEventListener`, `IntersectionObserver`, RxJS and store subscriptions are all examples. See [Frontend Design Patterns](/Software%20Engineering/Frontend/09%20Architecture/Frontend%20Design%20Patterns), [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs).
</Accordion>

<Accordion title="What is tree shaking?">
Removing unused exports from the final bundle. It relies on static ESM `import`/`export` so the bundler can see what is used; side effects (`"sideEffects": false` in `package.json`) and CommonJS limit it. It reduces download, parse and execution time. See [Build Tools & Bundlers](/Software%20Engineering/Frontend/14%20Tooling/Build%20Tools%20%26%20Bundlers), [Modules (ESM)](/Software%20Engineering/Frontend/01%20JavaScript/Modules%20(ESM)).
</Accordion>

<Accordion title="How do you approach A/B testing with multiple variants?">
Define the hypothesis, primary metric and guardrails first; split traffic evenly and sticky per user (hash of user id); make sure each variant has enough sample size (more variants need more traffic or longer runs; correct for multiple comparisons); assign on the server/edge to avoid flicker and CLS; log exposures only when the variant is actually seen; decide the stopping rule upfront. See [Case Study - Flight Search & Booking](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Flight%20Search%20%26%20Booking) (analytics section).
</Accordion>

## 8. Coding challenges from past interviews (from [Online Coding Assessment - Bit Count, Consecutive Path & Missing Rolls](/Software%20Engineering/Interviews/Online%20Coding%20Assessment%20-%20Bit%20Count%2C%20Consecutive%20Path%20%26%20Missing%20Rolls))
The original solutions were in Java; here are JavaScript versions.

<Info>
**Count set bits of A × B (A, B ≤ 100,000,000)**

The product can reach 10^16, above `Number.MAX_SAFE_INTEGER` (≈9×10^15), so use `BigInt`.
```js
function countBits(A, B) {
  let p = BigInt(A) * BigInt(B), count = 0;
  while (p > 0n) { count += Number(p & 1n); p >>= 1n; }
  return count;
}
countBits(3, 7); // 21 = 10101 → 3
```
(Correction: the original text wrote 21 as `101010`; that is 42. 21 in binary is `10101`.)
</Info>

<Info>
**Path 1 → N visiting every vertex in increasing order**

A path 1→2→…→N exists only if every edge (i, i+1) exists. Store edges in a `Set` and check each consecutive pair: O(N + M).
```js
function hasPath(N, A, B) {
  const edges = new Set();
  for (let i = 0; i < A.length; i++) {
    const [lo, hi] = A[i] < B[i] ? [A[i], B[i]] : [B[i], A[i]];
    if (hi === lo + 1) edges.add(lo);
  }
  for (let v = 1; v < N; v++) if (!edges.has(v)) return false;
  return true;
}
hasPath(4, [1,2,4,4,3], [2,3,1,3,1]); // true
hasPath(4, [1,2,1,3], [2,4,3,4]);     // false
hasPath(6, [2,4,5,3], [3,5,6,4]);     // false
```
</Info>

<Info>
**Missing dice rolls**

Needed sum = M × (N + F) − sum(A). It must be between F and 6F. Start all missing rolls at 1 and greedily add up to 5 to each.
```js
function missingRolls(A, F, M) {
  let need = M * (A.length + F) - A.reduce((s, x) => s + x, 0);
  if (need < F || need > 6 * F) return [0];
  const rolls = Array(F).fill(1);
  need -= F;
  for (let i = 0; i < F && need > 0; i++) { const add = Math.min(5, need); rolls[i] += add; need -= add; }
  return rolls;
}
missingRolls([3,2,4,3], 2, 4); // [6, 6]
missingRolls([6,1], 1, 1);     // [0]
```
</Info>

<Info>
**Git and shell multiple choice (answers)**

- Which change does **not** change a commit's SHA: author, date, or message? **None**: all of them are part of the hashed commit object, so changing any produces a new SHA.
- Who last modified a line? **`git blame <file>`** (`git log -L` for a line's history).
- Which prints "hello"? **`echo hello`**.
- Run all `.py` files in the current directory tree: **`for i in $(find . -name '*.py'); do python "$i"; done`** (option B; backticks work too, and `find ... -exec python {} \;` avoids problems with spaces in names).
</Info>

## Related
- [JavaScript Index](/Software%20Engineering/Frontend/01%20JavaScript/JavaScript%20Index) · [React Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/React%20Interview%20Questions) · [Output Questions & Tricky Snippets](/Software%20Engineering/Frontend/17%20Interview%20Prep/Output%20Questions%20%26%20Tricky%20Snippets)
- [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks) · [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation) · [Closures & Memory Model](/Software%20Engineering/Frontend/01%20JavaScript/Closures%20%26%20Memory%20Model)
- [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index) · [Frontend Interview Concepts & Question Bank](/Software%20Engineering/Interviews/Frontend%20Interview%20Concepts%20%26%20Question%20Bank) · [Online Coding Assessment - Bit Count, Consecutive Path & Missing Rolls](/Software%20Engineering/Interviews/Online%20Coding%20Assessment%20-%20Bit%20Count%2C%20Consecutive%20Path%20%26%20Missing%20Rolls)

## Sources
- Owner's notes: `Frontend/Front End.md` (JS Questions list, event loop example), `Interviews/Interview.md`, `Interviews/Interview 1.md`
- freeCodeCamp, var/let/const: https://www.freecodecamp.org/news/var-let-and-const-whats-the-difference/
- MDN JavaScript reference: https://developer.mozilla.org/en-US/docs/Web/JavaScript
- MDN `structuredClone`: https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone
- HTML spec, event loop processing model: https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model
- TC39 finished proposals: https://github.com/tc39/proposals/blob/main/finished-proposals.md
- Temporal support: https://web-platform-dx.github.io/web-features-explorer/features/temporal/
