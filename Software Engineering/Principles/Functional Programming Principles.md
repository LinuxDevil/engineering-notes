---
tags: [principles, principles/fp]
aliases: [Functional Programming, FP, Pure Functions, Higher-Order Functions, Function Composition, Side Effects, Functional Core Imperative Shell, Referential Transparency]
updated: 2026-09-27
---
# Functional Programming Principles

> [!tldr] TL;DR
> **Functional programming** builds programs from **pure functions** (same input → same output, no side effects) over **immutable data**, combined with **higher-order functions** and **composition**. You don't need Haskell to benefit: in any language, push side effects (I/O, time, randomness, mutation) to the edges and keep the core as pure transformations (**functional core, imperative shell**). The payoff is code that's easier to test, reason about, parallelise and refactor.

## The idea

OOP organises code around objects that combine state and behaviour. FP organises code around **transformations of data**: values flow through functions. Most modern languages (JavaScript/TypeScript, Java 8+, Kotlin, C#, Python, Swift, Rust, Dart) are multi-paradigm and support both styles.

## 1. Pure functions

A function is **pure** if:
1. Its output depends **only on its inputs** (no reading globals, clocks, random, I/O).
2. It has **no side effects** (no mutation of inputs or external state, no I/O, no exceptions as control flow ideally).

```ts
// ❌ Impure: reads time, mutates input, writes to the network
function applyDiscount(cart: Cart) {
  if (new Date().getDay() === 5) cart.total *= 0.9;   // hidden input + mutation
  analytics.track("discount", cart.id);               // side effect
  return cart;
}

// ✅ Pure: everything it needs is a parameter; returns a new value
function applyDiscount(cart: Cart, today: Date): Cart {
  const isFriday = today.getDay() === 5;
  return isFriday ? { ...cart, total: cart.total * 0.9 } : cart;
}
```

Pure functions are **referentially transparent**: you can replace a call with its result without changing the program. That makes them trivially testable (no mocks), cacheable (memoisation, see [[Closures & Memory Model]]) and safe to run in parallel.

## 2. Immutability

Don't change data; create new versions. Shared immutable data can't be corrupted by another part of the program or another thread. Language tools are covered in [[Immutability Across Languages]].

```ts
const addItem = (cart: Cart, item: Item): Cart => ({ ...cart, items: [...cart.items, item] });
const sorted = prices.toSorted((a, b) => a - b);    // ES2023: non-mutating sort
```

## 3. First-class and higher-order functions

Functions are values: you can store them, pass them and return them. A **higher-order function** takes and/or returns functions.

```ts
// Takes functions: map, filter, reduce
const total = orders
  .filter(o => o.status === "paid")
  .map(o => o.amount)
  .reduce((sum, a) => sum + a, 0);

// Returns a function: a factory / partial application
const multiplier = (factor: number) => (n: number) => n * factor;
const withVat = multiplier(1.21);
withVat(100); // 121
```

```java
// Java streams: the same idea
BigDecimal total = orders.stream()
    .filter(o -> o.status() == Status.PAID)
    .map(Order::amount)
    .reduce(BigDecimal.ZERO, BigDecimal::add);
```

Higher-order functions replace many OO patterns: Strategy, Command, Template Method and Decorator become "pass a function" (see [[Behavioral Patterns]]).

## 4. Composition

Build complex behaviour by chaining small functions: the output of one is the input of the next.

```ts
const pipe = <T>(...fns: Array<(x: T) => T>) => (x: T) => fns.reduce((acc, f) => f(acc), x);

const normalizeEmail = pipe<string>(
  s => s.trim(),
  s => s.toLowerCase(),
  s => s.replace(/\+.*@/, "@"),     // drop "+tag" aliases
);
normalizeEmail("  Ana+news@Example.com "); // "ana@example.com"
```

Currying (turning `f(a, b)` into `f(a)(b)`) and partial application make functions easier to compose; see [[Functions - HOF, Callbacks & Currying]]. Point-free style is fine in moderation; readability wins over cleverness.

## 5. Declarative over imperative

Say **what** you want, not **how** to loop. `filter/map/reduce`, SQL, and React's "UI = f(state)" are declarative.

```ts
// Imperative
const names: string[] = [];
for (let i = 0; i < users.length; i++) if (users[i].active) names.push(users[i].name);
// Declarative
const names = users.filter(u => u.active).map(u => u.name);
```

## 6. Side-effect isolation: functional core, imperative shell

Real programs must do I/O. FP doesn't avoid effects; it **isolates** them. Gary Bernhardt's **functional core, imperative shell**: a thin outer layer does I/O and calls a pure core that makes all decisions.

```mermaid
flowchart LR
  IN["Shell: read request, load data from DB, get current time"] --> CORE["Pure core: decide (validate, compute, choose actions)"]
  CORE --> OUT["Shell: save, send emails, respond"]
```

```ts
// Pure core: decisions as data
type Decision =
  | { kind: "reject"; reason: string }
  | { kind: "approve"; limit: number; notify: string[] };

export function decideLoan(app: Application, history: CreditHistory, now: Date): Decision {
  if (age(app.birthDate, now) < 18) return { kind: "reject", reason: "UNDERAGE" };
  if (history.defaults > 0) return { kind: "reject", reason: "PRIOR_DEFAULT" };
  const limit = Math.min(app.income * 0.3, 50_000);
  return { kind: "approve", limit, notify: [app.email] };
}

// Imperative shell: all I/O at the edges
export async function handleLoanRequest(req: Request) {
  const app = parseApplication(await req.json());
  const history = await creditBureau.fetch(app.nationalIdHash);
  const decision = decideLoan(app, history, new Date());
  await loans.save(app.id, decision);
  if (decision.kind === "approve") await mailer.send(decision.notify, "approved");
  return Response.json(decision);
}
```

`decideLoan` is tested with plain values, no mocks. The shell has almost no logic, so a couple of integration tests cover it. This is the same shape as hexagonal architecture ([[Software Architecture Styles]]).

## 7. Errors as values

Instead of throwing for expected failures, return a value that represents success or failure, so the type system forces callers to handle it.

```ts
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

function parseAge(input: string): Result<number, "NOT_A_NUMBER" | "OUT_OF_RANGE"> {
  const n = Number(input);
  if (Number.isNaN(n)) return { ok: false, error: "NOT_A_NUMBER" };
  if (n < 0 || n > 150) return { ok: false, error: "OUT_OF_RANGE" };
  return { ok: true, value: n };
}
```

Equivalents: `Optional`/`Maybe` for absence, `Either`/`Result` for failure (Rust `Result`, Kotlin `Result`, Scala `Either`, libraries like Effect or neverthrow in TS, Vavr in Java). Exceptions remain right for truly exceptional, unrecoverable situations.

## 8. Algebraic data types and pattern matching

Model data as **sum types** (one of several variants) and **product types** (records), and handle them with exhaustive pattern matching. This "makes illegal states unrepresentable".

```ts
// ❌ Many illegal combinations possible
type Req = { loading: boolean; data?: User; error?: string };
// ✅ Only valid states exist
type Req =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: string };
```

Java 21+ sealed interfaces + records + `switch` patterns bring the same to the JVM (see [[OOP#Objects vs data (data-oriented programming)]]).

## 9. Recursion and laziness

- Recursion replaces loops over recursive structures (trees, JSON). Beware stack depth in languages without tail-call optimisation (JS in most engines, Java, Python).
- **Lazy evaluation** computes values only when needed: generators, Java streams, Kotlin sequences, iterator helpers. Enables infinite sequences and avoids work.

## FP vs OOP: not a war

| | OOP | FP |
|---|---|---|
| Unit of organisation | Objects (state + behaviour) | Functions over data |
| State | Encapsulated, often mutable | Immutable values, explicit state transitions |
| Extensibility | Easy to add types | Easy to add operations |
| Polymorphism | Subtyping, virtual dispatch | Higher-order functions, ADTs + pattern matching, type classes |
| Strength | Modelling entities with lifecycles and invariants | Data transformation pipelines, concurrency, testability |

Pragmatic combination: immutable value objects, pure domain logic, objects/modules at the boundaries for encapsulating I/O and resources.

## Pitfalls

> [!warning] Common mistakes
> - "FP" code that still mutates: `array.sort()` and `reverse()` mutate in JS; `forEach` pushing into outer arrays.
> - Over-abstracted point-free chains nobody can debug. Name intermediate steps.
> - Performance traps: copying large structures in hot loops, `reduce` with spread (`acc => ({...acc, [k]: v})`) is O(n²). Use local mutation inside a function (invisible to callers) or persistent data structures.
> - Hidden impurity: logging, `Date.now()`, `Math.random()`, environment variables inside "pure" functions.
> - Monad-heavy libraries imposed on a team that doesn't know them: readability and hiring costs.

## Senior perspective

- The biggest win is architectural: **isolate side effects**. Most of your business logic can be pure functions, testable in milliseconds.
- Local mutation is fine if it doesn't escape (a `for` loop building a result inside a function is still externally pure).
- Immutability + pure functions make concurrency much easier ([[Concurrency And Parallelism]]): no shared mutable state, no locks.
- Frontend frameworks are FP-shaped: React's `UI = f(state)`, reducers, immutable updates, signals as derived computations.
- In code review ask: "Can this decision be made in a pure function, with I/O moved to the caller?"

## Interview questions

> [!question]- What is a pure function? Why does it matter?
> A function whose result depends only on its arguments and that has no side effects. It's referentially transparent: easy to test without mocks, cacheable, safe to run in parallel, and easy to reason about and refactor.

> [!question]- What is a higher-order function? Give examples.
> A function that takes a function as an argument or returns one: `map`, `filter`, `reduce`, `setTimeout`, middleware factories, `debounce`, `memoize`, Java's `Comparator.comparing`.

> [!question]- What does "functional core, imperative shell" mean?
> Put all decision logic in pure functions (core) that take data and return data (including descriptions of what to do), and keep I/O in a thin outer layer (shell) that loads inputs, calls the core and executes the outputs. The core is unit-testable; the shell is small.

> [!question]- How do you handle errors functionally?
> Represent expected failures as values (`Result`/`Either`, `Option`) in the return type so callers must handle them, and compose them with `map`/`flatMap`. Reserve exceptions for unexpected, unrecoverable conditions.

> [!question]- What does "make illegal states unrepresentable" mean?
> Design types (sum types/discriminated unions, sealed hierarchies) so invalid combinations can't be constructed, e.g. a request is exactly one of idle/loading/success(data)/error(message) rather than an object with optional `loading`, `data` and `error` fields.

> [!question]- Is FP better than OOP?
> Neither universally. FP excels at data transformations, concurrency and testability; OOP at modelling long-lived entities with encapsulated invariants and at open extension with new types. Modern code mixes both: immutable values and pure logic inside, objects/modules for boundaries and resources.

## Related
- [[Principles Index]]
- [[Immutability Across Languages]] · [[OOP]] · [[Behavioral Patterns]] · [[Software Architecture Styles]] · [[Concurrency And Parallelism]]
- JavaScript: [[Functions - HOF, Callbacks & Currying]] · [[Closures & Memory Model]] · [[Objects, Arrays & Immutability]]

## Sources
- [Gary Bernhardt: Boundaries / Functional Core, Imperative Shell (2012)](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)
- John Hughes, [Why Functional Programming Matters (1989)](https://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf)
- [Scott Wlaschin: Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/) · *Domain Modeling Made Functional* (Pragmatic Bookshelf, 2018)
- [Yaron Minsky: Effective ML ("make illegal states unrepresentable")](https://blog.janestreet.com/effective-ml-video/)
- [MDN: Array.prototype.toSorted](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/toSorted)
- [Oracle: java.util.stream package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)
