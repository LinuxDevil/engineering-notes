---
title: "JavaScript Index"
tags: [frontend, frontend/javascript, index]
aliases: [JS Index, JavaScript, JS MOC, JavaScript Map]
updated: 2026-09-27
---
# JavaScript Index

<Note>
**TL;DR**

The map of the JavaScript and TypeScript notes. Read them roughly in the order below: first **how code runs** (execution context → scope → hoisting → closures → `this`), then **values and objects**, then **async**, then **modules, iteration and memory**, then what's new. TypeScript sits on top once the runtime model is solid.
</Note>

## Mind map

{/* diagram source: /diagrams/diagram-218.mmd */}
![diagram](/diagrams/diagram-218.svg)

## Recommended reading order

### Part 1: How code runs (the engine's mental model)

| # | Note | One line |
|---|---|---|
| 1 | [Execution Context & Call Stack](/Software%20Engineering/Frontend/01%20JavaScript/Execution%20Context%20%26%20Call%20Stack) | Thread of execution + memory; global/function contexts, creation vs execution phase, the LIFO call stack, stack overflow. |
| 2 | [Scope & Lexical Environment](/Software%20Engineering/Frontend/01%20JavaScript/Scope%20%26%20Lexical%20Environment) | Global/module/function/block scope, lexical scoping, environment records + `outer` pointer = the scope chain, reading DevTools' Scope panel. |
| 3 | [Hoisting, TDZ & var-let-const](/Software%20Engineering/Frontend/01%20JavaScript/Hoisting%2C%20TDZ%20%26%20var-let-const) | What "hoisting" really is, the Temporal Dead Zone, and the full `var` vs `let` vs `const` comparison. |
| 4 | [Closures & Memory Model](/Software%20Engineering/Frontend/01%20JavaScript/Closures%20%26%20Memory%20Model) | The "backpack" (C.O.V.E.), step-by-step Hard Parts walkthroughs, heap vs stack, V8 contexts, memoization, leaks and stale closures. |
| 5 | [this, call, apply & bind](/Software%20Engineering/Frontend/01%20JavaScript/this%2C%20call%2C%20apply%20%26%20bind) | The four `this` rules + arrow functions, lost `this`, explicit binding, a `bind` polyfill. |
| 6 | [Functions - HOF, Callbacks & Currying](/Software%20Engineering/Frontend/01%20JavaScript/Functions%20-%20HOF%2C%20Callbacks%20%26%20Currying) | First-class functions, HOFs and callbacks (DRY), pure functions, currying, partial application, composition, `debounce`. |

### Part 2: Values, objects and OOP

| # | Note | One line |
|---|---|---|
| 7 | [Types, Coercion & Equality](/Software%20Engineering/Frontend/01%20JavaScript/Types%2C%20Coercion%20%26%20Equality) | 7 primitives vs objects, `typeof` quirks, truthy/falsy, coercion rules, `==` vs `===` vs `Object.is`, `null` vs `undefined`. |
| 8 | [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability) | Property descriptors, copying (shallow/deep, `structuredClone`), non-mutating array methods, immutable updates for React state. |
| 9 | [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance) | Prototype chain, `new`, classes, `extends`/`super`, class fields, `#private`, static blocks, `#x in obj`, composition vs inheritance. |
| 10 | [Collections - Map, Set, WeakMap & WeakRef](/Software%20Engineering/Frontend/01%20JavaScript/Collections%20-%20Map%2C%20Set%2C%20WeakMap%20%26%20WeakRef) | When to use `Map`/`Set` over objects/arrays, weak collections for caches, `WeakRef`/`FinalizationRegistry`, new Set methods and upsert. |

### Part 3: Asynchronous JavaScript

| # | Note | One line |
|---|---|---|
| 11 | [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks) | Call stack + Web APIs + task/microtask queues + rendering steps; ordering puzzles explained. |
| 12 | [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation) | Promise states and combinators, async/await, error handling, `AbortController`, writing a promise from scratch. |

### Part 4: Language and runtime

| # | Note | One line |
|---|---|---|
| 13 | [Modules (ESM)](/Software%20Engineering/Frontend/01%20JavaScript/Modules%20(ESM)) | ESM vs CommonJS, live bindings, dynamic `import()`, import attributes/JSON modules, tree shaking. |
| 14 | [Iterators, Generators & Iterator Helpers](/Software%20Engineering/Frontend/01%20JavaScript/Iterators%2C%20Generators%20%26%20Iterator%20Helpers) | The iteration protocols, generators, async iteration, iterator helpers (ES2025) and `Iterator.concat`. |
| 15 | [Error Handling](/Software%20Engineering/Frontend/01%20JavaScript/Error%20Handling) | `try/catch/finally`, custom errors, `cause`, async errors, global handlers, error boundaries. |
| 16 | [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection) | Stack vs heap, reachability and mark-and-sweep, common FE leaks and how to find them in DevTools. |
| 17 | [Modern JavaScript ES2022-ES2027](/Software%20Engineering/Frontend/01%20JavaScript/Modern%20JavaScript%20ES2022-ES2027) | What landed in each spec year, Baseline status, and what's coming (Temporal, `using`, import defer). |

### Part 5: TypeScript

| # | Note | One line |
|---|---|---|
| 18 | [TypeScript Essentials](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%20Essentials) | Types vs interfaces, narrowing, unions, `unknown` vs `any`, tsconfig essentials, typing React. |
| 19 | [Advanced TypeScript - Generics & Utility Types](/Software%20Engineering/Frontend/02%20TypeScript/Advanced%20TypeScript%20-%20Generics%20%26%20Utility%20Types) | Generics and constraints, conditional/mapped/template-literal types, `infer`, built-in utility types. |
| 20 | [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0) | Recent releases: erasable syntax, `import defer`, TS 6.0 new defaults, the native TS 7.0 compiler, Node type stripping. |

Then practise with [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions) and [Output Questions & Tricky Snippets](/Software%20Engineering/Frontend/17%20Interview%20Prep/Output%20Questions%20%26%20Tricky%20Snippets).

## Study tips

- Draw it. For any snippet, sketch **Global Memory**, the **Call Stack**, and each function's **Local Memory**, like the Hard Parts walkthroughs. Most "tricky output" questions become mechanical.
- Visualise hoisting, scope and closures step by step with the [JavaScript Visualizer (ui.dev)](https://ui.dev/javascript-visualizer).
- Put a `debugger` in a closure and read DevTools' **Scope** (Local / Closure / Script / Global) and **Call Stack** panels.

## Course and book notes

- [JavaScript the hard parts v2](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/JavaScript%20the%20hard%20parts%20v2): [Principles](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/Principles) → [Functions & Callbacks](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/Functions%20%26%20Callbacks) → [Closures](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/Closures) → [Asynchronous JavaScript](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/Asynchronous%20JavaScript) (Will Sentance, Frontend Masters). Their walkthroughs are migrated into notes 1, 4 and 6 above.
- [JS the hard parts v3](/Software%20Engineering/Frontend/Courses%20%26%20Books/JS%20the%20hard%20parts%20v3): 2026 version of the course (HOFs, closure, async, OOP, type coercion, immutable array methods, promise aborts).
- [JS Book - Secrets of JS Ninja](/Software%20Engineering/Frontend/Courses%20%26%20Books/JS%20Book%20-%20Secrets%20of%20JS%20Ninja): the topics it introduces live here:
  - **Generators** (functions that produce values on request and can suspend) → [Iterators, Generators & Iterator Helpers](/Software%20Engineering/Frontend/01%20JavaScript/Iterators%2C%20Generators%20%26%20Iterator%20Helpers)
  - **Proxies** (control access to objects) → [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability)
  - **Transpilers** (Babel, Traceur: "tomorrow's JS today") → [Build Tools & Bundlers](/Software%20Engineering/Frontend/14%20Tooling/Build%20Tools%20%26%20Bundlers)
  - **DOM** (structured representation of the UI built from HTML) → [DOM, Events & Event Delegation](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/DOM%2C%20Events%20%26%20Event%20Delegation)
- Topic checklist ("JS Concepts": variables, strict mode, coercion, classes, JSON, Map/Set, Symbol, Atomics, Proxy/Reflect, performance object, ...) → [Frontend Roadmap & Checklist](/Software%20Engineering/Frontend/Frontend%20Roadmap%20%26%20Checklist).

## Related
- [Frontend Home](/Software%20Engineering/Frontend/Frontend%20Home) · [Frontend Roadmap & Checklist](/Software%20Engineering/Frontend/Frontend%20Roadmap%20%26%20Checklist) · [What's New in Frontend 2025-2026](/Software%20Engineering/Frontend/What's%20New%20in%20Frontend%202025-2026) · [Courses & Books](/Software%20Engineering/Frontend/Courses%20%26%20Books)
- [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path) · [DOM, Events & Event Delegation](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/DOM%2C%20Events%20%26%20Event%20Delegation) · [React Index](/Software%20Engineering/Frontend/05%20React/React%20Index)
- [Frontend Interview Concepts & Question Bank](/Software%20Engineering/Interviews/Frontend%20Interview%20Concepts%20%26%20Question%20Bank) · [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions)

## Sources
- [MDN JavaScript Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide)
- [MDN JavaScript Reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference)
- [ECMAScript specification](https://tc39.es/ecma262/) · [TC39 proposals](https://github.com/tc39/proposals)
- [JavaScript Visualizer (ui.dev)](https://ui.dev/javascript-visualizer)
