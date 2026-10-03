---
tags: [frontend, frontend/architecture, frontend/patterns]
aliases: [JS design patterns, Design patterns in JavaScript, Observer pattern, Pub-Sub]
updated: 2026-09-27
---
# Frontend Design Patterns

> [!tldr] TL;DR
> Design patterns are named, reusable solutions to recurring design problems. In front-end code a handful show up constantly: **module** and **singleton** (encapsulation and shared instances), **observer / pub-sub** (events, stores, RxJS, signals), **factory**, **proxy** (Vue/MobX reactivity, validation), **decorator** (HOCs, Angular decorators), **strategy**, **command** (undo/redo, Redux actions), **mediator**, **facade** and **adapter** (wrapping APIs). Knowing them lets you name what frameworks already do and pick the right tool in system design interviews. For the general catalogue see [[Design Patterns]]; for principles see [[SOLID]].

## 1. The catalogue at a glance

The owner's checklist (classic GoF plus a few extras), with where each shows up in front-end work:

| Category | Pattern | Front-end example |
|---|---|---|
| **Creational** | Factory Method | `createStore`, `createElement`, component factories |
| | Abstract Factory | Theme factories producing matching button/input/card sets |
| | Builder | Query builders, `new URLSearchParams()` chains, test data builders |
| | Prototype | JS prototypes, `structuredClone` of a template object |
| | Singleton | ES module instances, `providedIn: 'root'` services, an API client |
| | Object Pool | Web Worker pools, reusing DOM nodes in virtualized lists |
| **Structural** | Adapter | Wrapping a 3rd-party SDK or API response into your domain shape |
| | Facade | A `useCheckout()` hook or `BookingService` hiding many calls |
| | Decorator | HOCs, `withLogging(fn)`, TS/Angular decorators |
| | Proxy | `Proxy` objects in Vue 3 / MobX reactivity, caching or validation proxies |
| | Composite | Component trees, menus with nested items |
| | Flyweight | Sharing immutable style/token objects, `React.memo`-style reuse |
| | Bridge | Separating a chart API from its canvas/SVG renderer |
| | Private Class Data | `#private` fields, closures |
| **Behavioral** | Observer | DOM events, `EventTarget`, RxJS, signals, store subscriptions |
| | Command | Redux actions, undo/redo stacks, keyboard shortcut registries |
| | Strategy | Swappable validators, sort/price calculators, payment methods |
| | Mediator | An event bus or store coordinating components |
| | State | Finite state machines (XState) for wizards and booking flows |
| | Iterator | `for...of`, generators, iterator helpers |
| | Chain of Responsibility | Middleware (Redux, Express), HTTP interceptors |
| | Template Method | Base class lifecycles, abstract "fetch-then-render" flows |
| | Memento | Snapshots for undo, persisted form drafts |
| | Null Object | Default no-op logger/analytics clients |
| | Interpreter | Template languages, query/filter DSLs |
| | Visitor | AST transforms (Babel/ESLint plugins, codemods) |

The rest of this note gives a small example for the eleven most useful ones.

## 2. Module pattern

Encapsulate private state and expose a public API. Before ES modules this was an IIFE closure; today every ES module file is a module with private top-level scope.

```js
// IIFE module (classic)
const counter = (() => {
  let count = 0;                          // private
  return { inc: () => ++count, get: () => count };
})();

// ES module (modern): non-exported bindings are private
let count = 0;
export const inc = () => ++count;
export const get = () => count;
```

See [[Modules (ESM)]] and [[Closures & Memory Model]].

## 3. Singleton

Exactly one instance shared app-wide. ES modules are evaluated once and cached, so exporting an instance is already a singleton.

```js
// apiClient.js
class ApiClient {
  constructor(baseUrl) { this.baseUrl = baseUrl; }
  get(path) { return fetch(this.baseUrl + path).then(r => r.json()); }
}
export const api = new ApiClient('/api');   // one instance per module graph
```

> [!warning] Singleton pitfalls
> Hidden global state makes tests leak into each other and breaks SSR (one instance shared by **all users' requests** on the server). Prefer dependency injection (Angular `providedIn: 'root'`, React context) so tests and requests can get fresh instances.

## 4. Observer and pub/sub

**Observer:** we *subscribe* certain objects, the **observers**, to another object, the **observable**. Whenever an event occurs, the observable notifies all its observers.

**Pub/sub** adds a broker (event bus/channel) between publishers and subscribers, so they don't know each other.

```js
class Observable {
  #observers = new Set();
  subscribe(fn) { this.#observers.add(fn); return () => this.#observers.delete(fn); }
  notify(data) { this.#observers.forEach(fn => fn(data)); }
}

const priceFeed = new Observable();
const unsubscribe = priceFeed.subscribe(p => console.log('new price', p));
priceFeed.notify(420);
unsubscribe();
```

```jsx
// React: subscribe an external store safely (tearing-free with concurrent rendering)
function usePrice() {
  return useSyncExternalStore(priceStore.subscribe, priceStore.getSnapshot);
}
```

Where you already use it: `addEventListener`, `EventTarget`, `MutationObserver`/`IntersectionObserver` ([[Observer APIs]]), RxJS, Redux `subscribe`, Angular/Solid/Vue signals, `BroadcastChannel`, micro-frontend event buses.

## 5. Factory

A function that creates objects without the caller knowing the concrete class.

```js
function createNotifier(channel) {
  switch (channel) {
    case 'email': return { send: msg => sendEmail(msg) };
    case 'sms':   return { send: msg => sendSms(msg) };
    default:      return { send: () => {} };            // Null Object
  }
}
createNotifier(user.pref).send('Your flight is confirmed');
```

```jsx
// Component factory: map CMS/API "type" to a component
const widgets = { hero: Hero, carousel: Carousel, faq: Faq };
function Widget({ type, ...props }) {
  const Component = widgets[type] ?? Fallback;
  return <Component {...props} />;
}
```

## 6. Proxy

Intercept operations on an object (get/set/has...). JS has it built in (`Proxy`); Vue 3 and MobX use it for reactivity.

```js
const validated = target => new Proxy(target, {
  set(obj, key, value) {
    if (key === 'age' && !Number.isInteger(value)) throw new TypeError('age must be an integer');
    obj[key] = value;
    return true;
  },
});
const passenger = validated({});
passenger.age = 30;     // ok
passenger.age = '30';   // TypeError
```

Other front-end proxies: a caching proxy around `fetch`, a lazy-loading proxy for heavy modules, a dev-server API proxy.

## 7. Decorator

Add behaviour to an object or function without changing it: wrap it.

```js
const withTiming = fn => async (...args) => {
  const t = performance.now();
  try { return await fn(...args); }
  finally { console.log(`${fn.name} took ${performance.now() - t}ms`); }
};
const searchFlights = withTiming(api.searchFlights);
```

```jsx
// React HOC = decorator for components
const withAuth = Component => props => {
  const user = useUser();
  return user ? <Component {...props} user={user} /> : <Navigate to="/login" />;
};
```

TypeScript/Angular decorators (`@Component`, `@Input`) are metadata decorators applied by the compiler. In React, hooks have mostly replaced HOCs. See [[Component Patterns]].

## 8. Strategy

Encapsulate interchangeable algorithms behind one interface and pick at runtime. Replaces long `if/else` chains and satisfies the open/closed principle.

```js
const sortStrategies = {
  cheapest: (a, b) => a.price - b.price,
  fastest:  (a, b) => a.durationMin - b.durationMin,
  best:     (a, b) => score(b) - score(a),
};
const sorted = [...flights].sort(sortStrategies[sortBy]);
```

Also: form validation rules, payment providers, pricing/tax rules per market.

## 9. Command

Represent an action as an object so it can be queued, logged, replayed or undone. Redux actions are commands.

```js
const history = [];
function execute(cmd) { cmd.do(); history.push(cmd); }
function undo() { history.pop()?.undo(); }

const addSeat = (booking, seat) => ({
  do:   () => booking.seats.push(seat),
  undo: () => booking.seats.splice(booking.seats.indexOf(seat), 1),
});
execute(addSeat(booking, '12A'));
undo();
```

## 10. Mediator

Components talk through a central mediator instead of to each other, reducing N×N coupling to N×1. A chat room, an event bus, or a store with actions are mediators.

```js
const bus = new EventTarget();                      // mediator
// filters panel
bus.dispatchEvent(new CustomEvent('filters:changed', { detail: { stops: 0 } }));
// results list and map both listen, neither knows about the filters panel
bus.addEventListener('filters:changed', e => refetch(e.detail));
```

Mediator vs observer: observer is one subject to many listeners; a mediator coordinates many-to-many interactions and may contain the coordination logic.

## 11. Facade

A simple interface over a complex subsystem.

```js
// hides pricing API, seat API, analytics and error mapping
export function useCheckout(bookingId) {
  const price = useQuery({ queryKey: ['price', bookingId], queryFn: () => pricingApi.get(bookingId) });
  const pay = useMutation({ mutationFn: payload => paymentsApi.charge(bookingId, payload),
                            onSuccess: () => analytics.track('purchase') });
  return { total: price.data?.total, isLoading: price.isPending, pay: pay.mutate };
}
```

Custom hooks and Angular services are facades most of the time.

## 12. Adapter

Convert one interface into the one your code expects. Essential at API and SDK boundaries so backend changes don't ripple through the UI.

```js
// supplier API shape -> domain model
const toFlight = dto => ({
  id: dto.flight_id,
  from: dto.origin.iata,
  to: dto.destination.iata,
  departAt: new Date(dto.dep_time_utc),
  price: { amount: dto.fare.total / 100, currency: dto.fare.ccy },
});
const flights = (await res.json()).results.map(toFlight);
```

Also: wrapping `localStorage` behind a `Storage` interface so you can swap in IndexedDB, or adapting an old callback API to promises.

## 13. Patterns at the component and application level

Component architecture (owner's checklist; details in [[Component Patterns]]):
- **Container vs presentational** (smart/dumb) components.
- **Layout components** (slots/children for structure).
- **Pure components** (`React.memo`, Angular OnPush): same props, same output.
- **Higher-order components** (decorator pattern) and render props, mostly replaced by hooks.
- **Component factory** (see Factory above).

Application architecture (owner's checklist):

| Topic | Where |
|---|---|
| Monolith vs micro-frontends, Module Federation | [[Micro-frontends & Module Federation]] |
| Monorepo architecture | [[Monorepos]] |
| PWA | [[Web Workers, Service Workers & PWAs]] |
| Serverless / edge rendering | [[Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands]], [[Next.js Rendering & Caching]] |
| Code standards: SOLID, KISS, DRY | [[SOLID]], [[Design Patterns]] |

> [!warning] Common pitfalls
> - Pattern-first design: don't add a factory or mediator until you have the second use case.
> - Singletons and module-level state leak between SSR requests.
> - Observers without unsubscribe cause memory leaks. Always return and call a cleanup function.
> - Over-abstracted facades that hide errors or loading states make debugging hard.

## Interview questions

> [!question]- Explain the observer pattern and where it's used in front-end code.
> A subject keeps a list of observers and notifies them on change. DOM events, RxJS, store subscriptions (Redux, Zustand via `useSyncExternalStore`), signals and the Observer APIs all use it. Pub/sub adds a broker so publishers and subscribers are decoupled.

> [!question]- How is a singleton implemented in JavaScript, and what are the risks?
> Export an instance from an ES module (modules are evaluated once). Risks: hidden global state, test pollution, and shared state across users in SSR. Prefer DI or context-scoped instances.

> [!question]- Decorator vs proxy?
> A decorator wraps an object to **add** behaviour while keeping the interface; a proxy **controls access** to an object (validation, lazy loading, caching, reactivity tracking). JS's `Proxy` object can implement both.

> [!question]- Give a front-end example of the strategy pattern.
> Sorting flights by cheapest/fastest/best via a map of comparator functions, or selecting a payment provider implementation at runtime. It removes conditionals and lets you add strategies without modifying callers.

> [!question]- How would you implement undo/redo?
> Command pattern: each change is an object with `do`/`undo` pushed on a history stack (plus a redo stack), or Memento: snapshot immutable state (easy with Redux/Immer) and move a pointer through snapshots.

> [!question]- Facade vs adapter?
> A facade simplifies a complex subsystem behind a new, smaller interface. An adapter converts an existing interface into another one that clients expect, without simplifying it.

> [!question]- Mediator vs observer?
> Observer: one-to-many notification from a subject. Mediator: a central object coordinating many-to-many communication so components don't reference each other; it often contains the coordination rules.

## Related
- [[Design Patterns]]
- [[SOLID]]
- [[Component Patterns]]
- [[State Management Overview]]
- [[Observer APIs]]
- [[Closures & Memory Model]]
- [[Micro-frontends & Module Federation]]
- [[Angular Fundamentals]]

## Sources
- https://www.patterns.dev/
- https://refactoring.guru/design-patterns
- https://sourcemaking.com/design_patterns
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy
- https://react.dev/reference/react/useSyncExternalStore
