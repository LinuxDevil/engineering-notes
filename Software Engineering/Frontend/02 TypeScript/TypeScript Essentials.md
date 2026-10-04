---
title: "TypeScript Essentials"
tags: [frontend, frontend/typescript]
aliases: [TypeScript, TS basics, type vs interface, enum vs union, satisfies, unknown vs any]
updated: 2026-09-27
---
# TypeScript Essentials

<Note>
**TL;DR**

TypeScript adds a **static, structural** type system on top of JavaScript; types are erased at compile time and don't exist at runtime. The core skills: choosing between `type`, `interface` and `abstract class`; preferring **union types** over `enum`; **narrowing** unions safely; using `unknown` (not `any`) for untrusted data; `never` for exhaustiveness; and `satisfies` to validate a value without widening it.
</Note>

## Mental model

- **Structural typing** ("duck typing"): compatibility is decided by **shape**, not by name. A value with the right properties fits, regardless of which class/interface it came from.
- **Erasure**: types disappear after compilation. You can't `if (x instanceof MyInterface)`; runtime checks must use JS (`typeof`, `in`, `instanceof` on classes, discriminant fields, schema validators like Zod/Valibot).
- **Inference first**: annotate function boundaries (params, public return types), let TS infer locals.

```ts
interface Point { x: number; y: number }
const p = { x: 1, y: 2, label: 'A' };
const q: Point = p;           // OK: structurally compatible (extra props allowed for non-fresh objects)
const r: Point = { x: 1, y: 2, label: 'A' }; // Error: excess property check on object literals
```

## Basic types cheat sheet

```ts
let s: string; let n: number; let b: boolean; let big: bigint; let sym: symbol;
let u: undefined; let nul: null;
let list: string[];              // or Array<string>
let tuple: [string, number];     // fixed length/positions
let ro: readonly string[];       // no push/pop
let obj: { id: string; note?: string };   // optional prop
let fn: (a: number) => string;
let rec: Record<string, number>;
let lit: 'GET' | 'POST';         // literal union
```

## `type` vs `interface` vs `abstract class`

| | `type` alias | `interface` | `abstract class` |
|---|---|---|---|
| Exists at runtime | No | No | **Yes** (it's a real class) |
| Describes | Anything: objects, unions, intersections, tuples, primitives, mapped/conditional types | Object shapes (and function/class contracts) | A base class with shared **implementation** + abstract members |
| Extend | Intersection `A & B` | `interface B extends A` (better errors, cached, faster for the checker) | `class B extends A` (single inheritance) |
| Declaration merging | No (duplicate name = error) | **Yes**: same-name interfaces merge (used to augment libraries, `Window`) | No |
| Unions / mapped / conditional | Yes | No | No |
| Implementation code | No | No | Yes (methods, constructor, fields) |
| `implements` | A class can implement an object type | Yes | `extends` |
| Access modifiers | No | No | `private`, `protected`, `public` |

```ts
// type: unions, composition, utility types
type Status = 'idle' | 'loading' | 'success' | 'error';
type WithId<T> = T & { id: string };

// interface: object contracts, extendable, mergeable
interface Flight { number: string; departure: Date }
interface CharterFlight extends Flight { operator: string }
declare global { interface Window { analytics: Analytics } }   // augmentation via merging

// abstract class: shared behaviour + contract, exists at runtime
abstract class PaymentProvider {
  constructor(protected readonly apiKey: string) {}
  abstract charge(amount: number): Promise<string>;           // subclasses must implement
  async pay(amount: number) {                                 // shared implementation
    if (amount <= 0) throw new Error('Invalid amount');
    return this.charge(amount);
  }
}
class StripeProvider extends PaymentProvider {
  async charge(amount: number) { return `stripe-${amount}`; }
}
new PaymentProvider('k'); // Error: cannot create an instance of an abstract class
```

<Tip>
**Practical rule**

Use `interface` for public object contracts that might be extended (props of shared components, API models, library augmentation). Use `type` for unions, tuples, function types and anything computed with generics/utility types. Use an `abstract class` only when you need shared runtime code plus a contract (common in Angular services; rare in React code). Consistency within a codebase matters more than the choice.
</Tip>

<Warning>
**Parameter properties are not "erasable"**

`constructor(protected readonly apiKey: string)` generates runtime code. With `--erasableSyntaxOnly` (for Node's native type stripping) it's an error; write the field and assignment explicitly. See [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0).
</Warning>

## `enum` vs union types

| | `enum` | String literal union | `as const` object |
|---|---|---|---|
| Runtime code | **Yes** (an object; numeric enums get a reverse mapping) | None | The object you wrote |
| Type safety | Numeric enums accept any number (looser before TS 5.0) | Exact | Exact |
| Tree shaking / bundle | Adds code | Zero | Minimal |
| Iterate values | Awkward (reverse mapping pollutes numeric enums) | Needs a separate array | `Object.values(obj)` |
| Works with Node type stripping / `erasableSyntaxOnly` | **No** | Yes | Yes |
| Nominal-ish (can't pass raw string) | String enums: yes | No | No |

```ts
// enum
enum Cabin { Economy = 'ECONOMY', Business = 'BUSINESS' }
function price(c: Cabin) {}
price(Cabin.Economy);
price('ECONOMY');                       // Error: string enums are nominal

// union (preferred default)
type CabinU = 'ECONOMY' | 'BUSINESS';
function priceU(c: CabinU) {}
priceU('ECONOMY');                      // OK, zero runtime cost

// as const object: when you need both runtime values and a type
const CABIN = { Economy: 'ECONOMY', Business: 'BUSINESS' } as const;
type CabinC = typeof CABIN[keyof typeof CABIN];   // 'ECONOMY' | 'BUSINESS'
Object.values(CABIN).map(renderOption);
```

<Note>
**`const enum`**

Inlined at compile time (no object), but it breaks with isolated per-file transpilers (Babel, SWC, esbuild, `isolatedModules`) across files. Avoid in libraries. In 2026 the recommended default is **unions or `as const` objects**.
</Note>

## `any` vs `unknown` vs `never`

| | `any` | `unknown` | `never` |
|---|---|---|---|
| Meaning | Opt out of type checking | "Some value, type not known yet" (top type, safe) | "No value can exist here" (bottom type) |
| Assign anything to it | Yes | Yes | No |
| Use it without checks | Yes (unsafe, spreads silently) | **No**: must narrow first | n/a |
| Typical use | Migration escape hatch (avoid) | `JSON.parse`, `catch (e)`, API responses, generic inputs | Exhaustiveness checks, functions that never return, impossible branches |

```ts
function parse(json: string): unknown { return JSON.parse(json); }
const data = parse(text);
data.name;                              // Error: 'data' is of type 'unknown'
if (typeof data === 'object' && data !== null && 'name' in data) {
  console.log(data.name);               // OK after narrowing
}

try { risky(); } catch (e) {            // e: unknown (with useUnknownInCatchVariables, part of strict)
  if (e instanceof Error) log(e.message);
}

function fail(msg: string): never { throw new Error(msg); }
```

## Narrowing

TypeScript follows control flow and narrows union types inside branches.

| Technique | Example |
|---|---|
| `typeof` | `if (typeof x === 'string')` |
| Truthiness | `if (user)` (careful: `0` and `''` are falsy) |
| Equality | `if (x === null)` |
| `in` operator | `if ('fly' in animal)` |
| `instanceof` | `if (err instanceof HttpError)` |
| Discriminated union | `switch (action.type)` |
| Type predicate | `function isFlight(x): x is Flight` |
| Assertion function | `function assert(c: unknown): asserts c` |

<Info>
**Discriminated union + exhaustive check with `never`**

```ts
type SearchState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; results: Flight[] }
  | { status: 'error'; error: string };

function render(s: SearchState) {
  switch (s.status) {
    case 'idle':    return 'Search for flights';
    case 'loading': return 'Loading...';
    case 'success': return `${s.results.length} flights`;   // s.results available here
    case 'error':   return s.error;
    default: {
      const _exhaustive: never = s;   // compile error if a new status is added and not handled
      return _exhaustive;
    }
  }
}
```
This pattern also makes "impossible states impossible" (no `results` while `loading`), which is great for React state. See [State, Props & Data Flow](/Software%20Engineering/Frontend/05%20React/State%2C%20Props%20%26%20Data%20Flow).
</Info>

<Info>
**Type predicate**

```ts
const isDefined = <T,>(x: T | null | undefined): x is T => x != null;
const ids = [1, null, 3].filter(isDefined);   // number[]
```
Since TS 5.5, simple predicates like `x => x != null` are **inferred** automatically in `filter`.
</Info>

## `satisfies` (TS 4.9)

Checks that a value matches a type **without changing the inferred type** (no widening). Best of both worlds vs a type annotation.

```ts
type Route = { path: string; auth?: boolean };

// Annotation: validated, but keys are widened to string, literal info lost
const routesA: Record<string, Route> = { home: { path: '/' }, admin: { path: '/admin', auth: true } };
routesA.hme;                 // no error (any string key allowed)

// satisfies: validated AND keeps the exact keys/literals
const routes = {
  home: { path: '/' },
  admin: { path: '/admin', auth: true },
} satisfies Record<string, Route>;
routes.hme;                  // Error: property 'hme' does not exist
routes.admin.path;           // type string, known key

const theme = { primary: '#0af', spacing: [4, 8, 16] } satisfies Record<string, string | number[]>;
theme.spacing.map(n => n * 2);   // still number[], not string | number[]
```

`as const satisfies T` is a popular combo for config objects: literal types + validation.

## Other everyday features

```ts
// Optional chaining / nullish coalescing are JS, but TS narrows through them
const city = user?.address?.city ?? 'Unknown';

// Non-null assertion: tells TS "trust me" (avoid unless you really know)
const el = document.getElementById('root')!;

// Type assertion (not a conversion, no runtime effect)
const input = event.target as HTMLInputElement;

// readonly and as const
const AIRPORTS = ['DXB', 'LHR'] as const;       // readonly ['DXB', 'LHR']
type Airport = typeof AIRPORTS[number];          // 'DXB' | 'LHR'

// keyof / typeof / indexed access
type FlightKeys = keyof Flight;                  // 'number' | 'departure'
type Dep = Flight['departure'];                  // Date

// Function overloads
function toDate(v: string): Date;
function toDate(v: number): Date;
function toDate(v: string | number) { return new Date(v); }
```

## Typing React basics

```tsx
type ButtonProps = React.ComponentProps<'button'> & { variant?: 'primary' | 'ghost' };

function Button({ variant = 'primary', ...rest }: ButtonProps) {
  return <button data-variant={variant} {...rest} />;
}

const [flight, setFlight] = useState<Flight | null>(null);
const inputRef = useRef<HTMLInputElement>(null);
const onChange = (e: React.ChangeEvent<HTMLInputElement>) => setQuery(e.target.value);
```

In React 19, `ref` is a normal prop for function components, so you type it via `React.ComponentProps<'input'>` instead of `forwardRef`. See [React 19 to 19.3 & React Compiler](/Software%20Engineering/Frontend/05%20React/React%2019%20to%2019.3%20%26%20React%20Compiler).

## Recommended strictness

Turn on `"strict": true` (TS 6.0 makes it the default) plus `noUncheckedIndexedAccess` (array/record access returns `T | undefined`) and `exactOptionalPropertyTypes` if the team can handle it. See [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0) for a full modern tsconfig.

<Warning>
**Common pitfalls**

- `any` leaking from untyped libraries or `JSON.parse`; prefer `unknown` + validation (Zod/Valibot) at the boundary.
- Using `as` to silence errors instead of narrowing.
- Thinking types validate runtime data: API responses are still `unknown` in reality.
- Numeric enums accepting arbitrary numbers, and the reverse mapping appearing in `Object.values`.
- `{}` type means "any non-nullish value", not "empty object"; use `Record<string, never>` for an empty object.
- Object literal excess property checks only apply to *fresh* literals.
</Warning>

## Interview questions

<Accordion title="What is the difference between `type` and `interface`?">
Both describe object shapes. Interfaces can be extended with `extends` and support declaration merging; type aliases can express unions, intersections, tuples, primitives and mapped/conditional types but can't merge. Use interfaces for extendable object contracts and types for everything computed.
</Accordion>

<Accordion title="When would you use an abstract class instead of an interface?">
When you need shared runtime implementation (methods, constructor logic, protected fields) along with a contract subclasses must fulfil. Interfaces are type-only and disappear at compile time.
</Accordion>

<Accordion title="Enum vs union type: which do you prefer and why?">
Usually unions (or `as const` objects): zero runtime cost, better tree shaking, exact type safety, and compatible with `erasableSyntaxOnly`/Node type stripping. Enums generate runtime objects, numeric ones are loosely typed, and they don't work under type stripping.
</Accordion>

<Accordion title="What's the difference between `any` and `unknown`?">
Both accept any value, but `any` disables checking and propagates silently; `unknown` forces you to narrow before using the value. Use `unknown` for untrusted input and `catch` variables.
</Accordion>

<Accordion title="What is `never` used for?">
A type with no values: return type of functions that throw or loop forever, the result of impossible intersections/narrowing, and exhaustiveness checks in `switch` statements over unions.
</Accordion>

<Accordion title="What does `satisfies` do?">
Validates that an expression matches a type while keeping its own, more specific inferred type, unlike an annotation, which widens the value to the annotated type.
</Accordion>

<Accordion title="What is a discriminated union?">
A union of object types sharing a literal "tag" property (e.g. `status`). Checking the tag narrows to the matching member, which models state machines safely and enables exhaustive checks.
</Accordion>

<Accordion title="What is structural typing?">
Type compatibility is based on shape (properties and their types), not on declared names or inheritance, so any object with the required members is assignable.
</Accordion>

## Related
- [Advanced TypeScript - Generics & Utility Types](/Software%20Engineering/Frontend/02%20TypeScript/Advanced%20TypeScript%20-%20Generics%20%26%20Utility%20Types)
- [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0)
- [Types, Coercion & Equality](/Software%20Engineering/Frontend/01%20JavaScript/Types%2C%20Coercion%20%26%20Equality)
- [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance)
- [State, Props & Data Flow](/Software%20Engineering/Frontend/05%20React/State%2C%20Props%20%26%20Data%20Flow)
- [Angular Fundamentals](/Software%20Engineering/Frontend/08%20Angular/Angular%20Fundamentals)

## Sources
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html): [Everyday types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html), [Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html), [Enums](https://www.typescriptlang.org/docs/handbook/enums.html), [Classes (abstract)](https://www.typescriptlang.org/docs/handbook/2/classes.html)
- [TypeScript 4.9 release notes (satisfies)](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html)
- [TypeScript 5.5 release notes (inferred type predicates)](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
- [Node.js: TypeScript type stripping](https://nodejs.org/api/typescript.html)
