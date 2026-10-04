---
title: "Advanced TypeScript - Generics & Utility Types"
tags: [frontend, frontend/typescript]
aliases: [Generics, Utility Types, Conditional Types, Mapped Types, infer, Template Literal Types, RequireOnly]
updated: 2026-09-27
---
# Advanced TypeScript - Generics & Utility Types

<Note>
**TL;DR**

**Generics** are type parameters: they let functions, components and types work over many types while keeping the relationship between input and output. With **constraints** (`extends`), **`keyof`**, **indexed access**, **conditional types** (with `infer`), **mapped types** and **template literal types**, you can compute new types from existing ones. The built-in **utility types** (`Partial`, `Pick`, `Omit`, `ReturnType`...) are just mapped/conditional types you can read and write yourself.
</Note>

## Generics

```ts
// Without generics: lose the type or duplicate code
function firstAny(arr: any[]): any { return arr[0]; }

// With generics: the output type follows the input type
function first<T>(arr: T[]): T | undefined { return arr[0]; }
const n = first([1, 2, 3]);        // number | undefined (T inferred as number)
const s = first<string>(['a']);    // explicit type argument
```

{/* diagram source: /diagrams/diagram-234.mmd */}
![diagram](/diagrams/diagram-234.svg)

### Generic interfaces, types, classes

```ts
interface ApiResponse<T> { data: T; error?: string; fetchedAt: Date }
type Paginated<T> = { items: T[]; page: number; total: number };

class TypedCache<K, V> {
  #map = new Map<K, V>();
  get(k: K) { return this.#map.get(k); }
  set(k: K, v: V) { this.#map.set(k, v); return this; }
}

async function getJson<T>(url: string): Promise<T> {
  const res = await fetch(url);
  return res.json() as Promise<T>;     // note: a cast, not validation
}
const flights = await getJson<Paginated<Flight>>('/api/flights');
```

### Constraints (`extends`) and `keyof`

```ts
// T must have an id
function byId<T extends { id: string }>(items: T[]): Record<string, T> {
  return Object.fromEntries(items.map(i => [i.id, i]));
}

// K must be a key of T; return type is the property type
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const f = { number: 'EK202', seats: 300 };
getProp(f, 'seats');      // number
getProp(f, 'gate');       // Error: '"gate"' is not assignable to '"number" | "seats"'
```

### Defaults and `const` type parameters

```ts
type Store<S = Record<string, unknown>> = { state: S };

// TS 5.0: infer literal types as if `as const` was written
function routes<const T extends readonly string[]>(paths: T) { return paths; }
const r = routes(['/home', '/admin']);   // readonly ['/home', '/admin']
```

### Generic React components

```tsx
type ListProps<T> = {
  items: T[];
  getKey: (item: T) => string;
  renderItem: (item: T) => React.ReactNode;
};

function List<T>({ items, getKey, renderItem }: ListProps<T>) {
  return <ul>{items.map(i => <li key={getKey(i)}>{renderItem(i)}</li>)}</ul>;
}

<List items={flights} getKey={f => f.number} renderItem={f => f.number} />;  // T = Flight
```

(In `.tsx` arrow functions write `<T,>` so the parser doesn't read `<T>` as JSX.)

## Type operators

| Operator | Example | Result |
|---|---|---|
| `keyof T` | `keyof { a: 1; b: 2 }` | `'a' \| 'b'` |
| `typeof value` | `typeof config` | The type of a runtime value |
| Indexed access `T[K]` | `Flight['departure']` | `Date` |
| `T[number]` | `(typeof AIRPORTS)[number]` | Union of array element types |
| Union `A \| B` | `string \| null` | Either |
| Intersection `A & B` | `Base & { id: string }` | Both |

## Conditional types

`T extends U ? X : Y`: a type-level `if`.

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<'hi'>;   // true
type B = IsString<42>;     // false

// Distributive over unions (when T is a naked type parameter)
type NonNull<T> = T extends null | undefined ? never : T;
type C = NonNull<string | null | undefined>;  // string

// Prevent distribution by wrapping in a tuple
type IsUnion<T> = [T] extends [string] ? 'all strings' : 'not all';
```

### `infer`: extract a type from inside another

```ts
type ElementOf<T> = T extends readonly (infer E)[] ? E : never;
type E1 = ElementOf<Flight[]>;                     // Flight

type Awaited2<T> = T extends Promise<infer V> ? Awaited2<V> : T;   // recursive unwrap
type R = Awaited2<Promise<Promise<number>>>;       // number

type FirstArg<F> = F extends (first: infer A, ...rest: any[]) => any ? A : never;
type Props<C> = C extends React.ComponentType<infer P> ? P : never;
```

## Mapped types

Iterate over keys to build a new object type.

```ts
type MyPartial<T>  = { [K in keyof T]?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type Mutable<T>    = { -readonly [K in keyof T]: T[K] };   // remove modifier
type Concrete<T>   = { [K in keyof T]-?: T[K] };           // remove optional

// Key remapping with `as` (TS 4.1)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type FlightGetters = Getters<{ number: string; seats: number }>;
// { getNumber: () => string; getSeats: () => number }

// Filter keys by value type
type KeysOfType<T, V> = { [K in keyof T]: T[K] extends V ? K : never }[keyof T];
type DateKeys = KeysOfType<Flight, Date>;   // 'departure'
```

## Template literal types

```ts
type HttpMethod = 'get' | 'post';
type Endpoint = `/api/${'flights' | 'hotels'}`;
type Handler = `${HttpMethod}:${Endpoint}`;   // 'get:/api/flights' | 'get:/api/hotels' | 'post:...' ...

type CssVar = `--${string}`;
const v: CssVar = '--brand-color';            // OK

type EventName<T extends string> = `on${Capitalize<T>}`;   // 'click' -> 'onClick'

// Parse a route's params at the type level
type Params<S extends string> =
  S extends `${string}:${infer P}/${infer Rest}` ? P | Params<Rest>
  : S extends `${string}:${infer P}` ? P : never;
type P = Params<'/flights/:from/:to'>;   // 'from' | 'to'
```

Built-in string helpers: `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize`.

## Built-in utility types

| Utility | What it does | Example |
|---|---|---|
| `Partial<T>` | All props optional | Update payloads: `Partial<User>` |
| `Required<T>` | All props required (removes `?`) | Normalised config after defaults |
| `Readonly<T>` | All props readonly | Immutable state |
| `Pick<T, K>` | Keep only keys `K` | `Pick<User, 'id' \| 'name'>` |
| `Omit<T, K>` | Remove keys `K` | `Omit<User, 'password'>` |
| `Record<K, V>` | Object with keys `K` and values `V` | `Record<Airport, number>` |
| `Exclude<U, E>` | Remove members from a union | `Exclude<Status, 'idle'>` |
| `Extract<U, E>` | Keep matching union members | `Extract<Event, { type: 'click' }>` |
| `NonNullable<T>` | Remove `null`/`undefined` | |
| `ReturnType<F>` | Function's return type | `ReturnType<typeof store.getState>` |
| `Parameters<F>` | Tuple of params | `Parameters<typeof fetch>[1]` |
| `Awaited<T>` | Unwrap promises recursively | `Awaited<ReturnType<typeof load>>` |
| `InstanceType<C>` / `ConstructorParameters<C>` | For classes | |
| `NoInfer<T>` (TS 5.4) | Block inference from this position | `fn<T>(values: T[], fallback: NoInfer<T>)` |

<Note>
**`Omit` is not strictly typed**

`Omit<T, K>` accepts any `K extends string | number | symbol`, so typos in `K` aren't errors. A strict version: `type StrictOmit<T, K extends keyof T> = Omit<T, K>;`. Also, `Omit` on a **union** doesn't distribute (it collapses to common keys); use `type DistributiveOmit<T, K extends PropertyKey> = T extends any ? Omit<T, K> : never;`.
</Note>

### `Required` vs "required only some"

`Required<T>` makes **all** properties required. Often you want a mix: some required, the rest optional. That's the owner's favourite helper:

<Info>
**`RequireOnly<T, P>`: "a great type"**

```ts
type RequireOnly<T, P extends keyof T> = Pick<T, P> & Partial<Omit<T, P>>;

interface User {
  id: string;
  realName: string;
  alterEgo: string;
  email: string;
  avatarUrl: string;
}

// Then you can use it
type DraftUser = RequireOnly<User, 'realName' | 'alterEgo'>;
// { realName: string; alterEgo: string } & { id?: string; email?: string; avatarUrl?: string }

const draft: DraftUser = { realName: 'Bruce Wayne', alterEgo: 'Batman' };   // OK
const bad: DraftUser = { realName: 'Clark Kent' };                          // Error: alterEgo missing
```
How it works: `Pick<T, P>` keeps the chosen keys as-is (required), `Omit<T, P>` takes the rest and `Partial` makes them optional, and `&` combines them.
</Info>

Related helpers worth knowing:

```ts
// The opposite: make only some keys optional
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
type NewUser = PartialBy<User, 'id' | 'avatarUrl'>;

// Flatten intersections for nicer hovers
type Prettify<T> = { [K in keyof T]: T[K] } & {};

// Deep partial
type DeepPartial<T> = { [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K] };

// Exactly one of several keys
type OneOf<T, K extends keyof T = keyof T> =
  K extends keyof T ? { [P in K]: T[P] } & Partial<Record<Exclude<keyof T, K>, never>> : never;
```

## Real-world example: typing Redux Toolkit store hooks

From the owner's Redux notes. Derive the state and dispatch types **from the store itself** so they never drift.

`store.ts`

```ts
import { configureStore } from '@reduxjs/toolkit';
import { tasksReducer } from '../features/tasks-slice';

const store = configureStore({
  reducer: {
    tasks: tasksReducer,
  },
});

export type ApplicationState = ReturnType<typeof store.getState>;   // utility type + typeof
export type ApplicationDispatch = typeof store.dispatch;            // includes thunk types

export default store;
```

`hooks.ts` (the owner's original version, using `TypedUseSelectorHook`, an interface for creating a selector hook typed for your store's root state):

```ts
import { useSelector, TypedUseSelectorHook, useDispatch } from 'react-redux';
import type { ApplicationDispatch, ApplicationState } from './store';

export const useAppSelector: TypedUseSelectorHook<ApplicationState> = useSelector;
export const useAppDispatch: () => ApplicationDispatch = useDispatch;
```

Modern equivalent (React Redux 9+ / RTK 2), using the `.withTypes<>()` helpers:

```ts
import { useDispatch, useSelector } from 'react-redux';
import type { ApplicationDispatch, ApplicationState } from './store';

export const useAppDispatch = useDispatch.withTypes<ApplicationDispatch>();
export const useAppSelector = useSelector.withTypes<ApplicationState>();
```

Then you can just use them like:

```ts
const tasks = useAppSelector(state => state.tasks.entities);   // fully typed
const dispatch = useAppDispatch();                              // knows about thunks
```

Why it works: `ReturnType<typeof store.getState>` uses `typeof` to get the function's type and the `ReturnType` conditional type (built with `infer`) to extract the state shape. See [Redux & Redux Toolkit](/Software%20Engineering/Frontend/06%20State%20Management/Redux%20%26%20Redux%20Toolkit).

## Variance, briefly

- Function **parameters** are checked contravariantly under `strictFunctionTypes` (for function-typed properties, not methods).
- Arrays are covariant (unsound but practical): `Dog[]` is assignable to `Animal[]`.
- You can annotate variance on type parameters (`in`, `out`, TS 4.7) for performance/clarity in libraries.

<Warning>
**Common pitfalls**

- Over-engineering types that nobody can read; prefer simple, named helper types and add comments.
- `getJson<T>()` returning `T` is a **cast**, not validation. Validate external data (Zod/Valibot) and infer the type from the schema: `type Flight = z.infer<typeof FlightSchema>`.
- Distributive conditional types surprising you on unions (wrap in `[T]` to stop it).
- Deep recursive types slowing down the checker ("Type instantiation is excessively deep").
- `Omit` on union types losing discriminants.
</Warning>

## Interview questions

<Accordion title="What are generics and why use them?">
Type parameters that let code work with many types while preserving the relationship between inputs and outputs (e.g. `first<T>(arr: T[]): T`). They avoid both duplication and `any`.
</Accordion>

<Accordion title="What does `K extends keyof T` mean?">
A constraint: `K` must be one of `T`'s keys. Combined with `T[K]` it lets you type property access generically (`getProp(obj, key)` returns the property's exact type).
</Accordion>

<Accordion title="Explain conditional types and `infer`.">
`T extends U ? X : Y` chooses a type based on assignability, distributing over unions. `infer` declares a type variable inside the `extends` clause to capture part of a type, e.g. `T extends Promise<infer V> ? V : T`.
</Accordion>

<Accordion title="How are `Partial`, `Pick` and `Omit` implemented?">
`Partial<T> = { [K in keyof T]?: T[K] }`, `Pick<T, K extends keyof T> = { [P in K]: T[P] }`, `Omit<T, K> = Pick<T, Exclude<keyof T, K>>`.
</Accordion>

<Accordion title="How would you make only some properties required?">
`type RequireOnly<T, P extends keyof T> = Pick<T, P> & Partial<Omit<T, P>>`: pick the required keys, make the rest optional, intersect them.
</Accordion>

<Accordion title="What are template literal types used for?">
Building string literal unions from other types: event names (`on${Capitalize<E>}`), CSS variables, typed route strings and parsing route params, key remapping in mapped types.
</Accordion>

<Accordion title="How do you type Redux hooks?">
Export `RootState = ReturnType<typeof store.getState>` and `AppDispatch = typeof store.dispatch`, then create `useAppSelector`/`useAppDispatch` with `useSelector.withTypes<RootState>()` and `useDispatch.withTypes<AppDispatch>()` (or the older `TypedUseSelectorHook`).
</Accordion>

<Accordion title="What's the difference between `Exclude` and `Omit`?">
`Exclude` removes members from a **union** type; `Omit` removes **keys** from an object type (implemented with `Pick` + `Exclude<keyof T, K>`).
</Accordion>

## Related
- [TypeScript Essentials](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%20Essentials)
- [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0)
- [Redux & Redux Toolkit](/Software%20Engineering/Frontend/06%20State%20Management/Redux%20%26%20Redux%20Toolkit)
- [Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns)
- [Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query)

## Sources
- TypeScript Handbook: [Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html), [Keyof](https://www.typescriptlang.org/docs/handbook/2/keyof-types.html), [Conditional types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html), [Mapped types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html), [Template literal types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html), [Utility types](https://www.typescriptlang.org/docs/handbook/utility-types.html)
- [Redux: Usage with TypeScript](https://redux.js.org/usage/usage-with-typescript), [React Redux: withTypes](https://react-redux.js.org/using-react-redux/usage-with-typescript#withtypes)
- [Type Challenges](https://github.com/type-challenges/type-challenges)
