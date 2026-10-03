---
tags: [frontend, frontend/react, frontend/patterns]
aliases: [React Patterns, HOC, Render Props, Compound Components, Error Boundaries, Portals, Suspense, React component types]
updated: 2026-09-27
---
# Component Patterns

> [!tldr] TL;DR
> React is about **composition**. Components should be **pure** functions of props, state and context. Most reuse today comes from **custom hooks** plus **composition** (`children`, slot props, compound components). HOCs and render props are older patterns you'll still meet in codebases and interviews. React ships a few built-in components for structure (`Fragment`), escape hatches (`createPortal`), async UI (`Suspense`, `lazy`), dev checks (`StrictMode`, `Profiler`) and errors (Error Boundaries, still class-based).

## 1. Pure components (the rule behind everything)
- **React assumes every component is a pure function**: the same inputs give the same JSX.
- A pure component **minds its own business**. It doesn't change any object or variable that existed before it was called.
- **Local mutation is fine.** You can create and change variables and arrays *inside* the render.
- Rendering must always be **pure**. Components only *return* JSX. They never *change* pre-existing objects during render.
- **StrictMode** calls components twice in development to catch impurity.

> [!example] Impure vs pure
> ```jsx
> let guest = 0;
> function Cup() { guest = guest + 1; return <h2>Tea cup for guest #{guest}</h2>; } // ❌ mutates outer variable
>
> function Cup({ guest }) { return <h2>Tea cup for guest #{guest}</h2>; }          // ✅ pure
> function TeaGathering() {
>   const cups = [];                                                                // ✅ local mutation
>   for (let i = 1; i <= 12; i++) cups.push(<Cup key={i} guest={i} />);
>   return cups;
> }
> ```

## 2. Component types

| Type | Definition | Notes |
|---|---|---|
| **Pure** | Same props/state give the same output. `React.PureComponent` (class) or `React.memo` (function) **shallow-compare** props/state and skip the re-render if nothing changed | "Pure" means two things. (1) Every component *should* be pure (no side effects in render). (2) `PureComponent`/`memo` is an *optimisation* that skips re-renders |
| **Static** | No state, no lifecycle/effects. Always renders the same content for the same props | e.g. `<Logo />`, `<Footer />` |
| **Controlled** | **React** controls the value of form elements through state and props | See [[Forms & Actions in React]] |
| **Uncontrolled** | The **DOM** holds the value. React reads it through a `ref` or `FormData` | `defaultValue`, `ref`, form Actions |
| **Stateful** (Container) | Keeps its own state and the logic to change it | |
| **Stateless** (Presentational) | Doesn't manage its own state. Renders from props | Easy to test and reuse |

![[Untitled 46.png]]

## 3. Container / presentational
- **Container** (stateful): fetches data, holds state and handlers.
- **Presentational** (stateless): receives props, renders UI, calls callbacks.
- Today the "container" is usually a **custom hook**, a route **loader**, or a **Server Component**. The idea still holds: separate data from display.

```jsx
function useFlights(query) { return useQuery({ queryKey: ['flights', query], queryFn: () => searchFlights(query) }); }
function FlightResults({ query }) {                 // container (thin)
  const { data, isPending } = useFlights(query);
  return <FlightList flights={data ?? []} loading={isPending} />;
}
function FlightList({ flights, loading }) { /* presentational */ }
```

## 4. Higher-Order Components (HOCs)
- A **wrapper**: a function that **takes a component and returns a new component** with extra behaviour.
- Examples: `connect()` in old Redux, `withRouter` in RR ≤5, `withAuth`.

```tsx
const Enhanced = higherOrderComponent(WrappedComponent);

function withAuth<P extends object>(Wrapped: React.ComponentType<P>) {
  function WithAuth(props: P) {
    const user = useUser();
    if (!user) return <Navigate to="/login" />;
    return <Wrapped {...props} user={user} />;
  }
  WithAuth.displayName = `withAuth(${Wrapped.displayName ?? Wrapped.name})`;
  return WithAuth;
}
```
**Drawbacks:**
- "Wrapper hell" in DevTools.
- Prop-name collisions.
- Unclear data origin.
- You must never create them inside render: it's a new type each time, so it remounts.

Prefer **custom hooks**.

## 5. Render props
The component takes a **function prop** (often `children`) and calls it with its internal state. The caller decides what to render.

```jsx
function MouseTracker({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return <div onPointerMove={e => setPos({ x: e.clientX, y: e.clientY })}>{children(pos)}</div>;
}
<MouseTracker>{({ x, y }) => <p>{x}, {y}</p>}</MouseTracker>
```

| | HOC | Render props | Custom hook |
|---|---|---|---|
| Shares | Logic + wraps the component | Logic + lets the caller render | Logic only |
| Composition | Static (at definition) | Dynamic (in JSX) | In the component body |
| Downsides | Wrapper hell, prop collisions | Callback nesting | Must follow the rules of hooks |
| Today | Rare | Headless libs (Downshift, some table libs) | **Default choice** |

## 6. Composition: children and slot props
- **`children` is just a prop.** Passing JSX as `children` or as other props ("slots") avoids prop drilling.
- It also **prevents re-renders**: elements created by the parent keep their identity when the wrapper's state changes. See [[How React Renders - Reconciliation & Fiber]] (Advanced React, ch. 2).

```jsx
function Layout({ header, sidebar, children }) {
  return (<div className="grid"><header>{header}</header><aside>{sidebar}</aside><main>{children}</main></div>);
}
<Layout header={<SearchBar />} sidebar={<Filters />}><Results /></Layout>
```

## 7. Compound components
A group of components that share **implicit state through context**, with a flexible, declarative API. Examples: `<select>`/`<option>`, and Radix/React Aria style components.

```tsx
const TabsCtx = createContext<{ active: string; setActive: (v: string) => void } | null>(null);
const useTabs = () => { const c = use(TabsCtx); if (!c) throw new Error('Use inside <Tabs>'); return c; };

export function Tabs({ defaultValue, children }) {
  const [active, setActive] = useState(defaultValue);
  return <TabsCtx value={{ active, setActive }}><div role="tablist">{children}</div></TabsCtx>;
}
Tabs.Tab = function Tab({ value, children }) {
  const { active, setActive } = useTabs();
  return <button role="tab" aria-selected={active === value} onClick={() => setActive(value)}>{children}</button>;
};
Tabs.Panel = function Panel({ value, children }) {
  const { active } = useTabs();
  return active === value ? <div role="tabpanel">{children}</div> : null;
};

<Tabs defaultValue="flights">
  <Tabs.Tab value="flights">Flights</Tabs.Tab><Tabs.Tab value="hotels">Hotels</Tabs.Tab>
  <Tabs.Panel value="flights"><FlightSearch /></Tabs.Panel>
  <Tabs.Panel value="hotels"><HotelSearch /></Tabs.Panel>
</Tabs>
```

## 8. Polymorphic components (the `as` prop)
One component that can render as different elements (`h1`, `a`, `button`, a router `Link`, and so on) while keeping its styles and behaviour, with type-safe props.

```tsx
type PolymorphicProps<E extends React.ElementType, P = {}> =
  P & { as?: E } & Omit<React.ComponentPropsWithRef<E>, keyof P | 'as'>;

function Text<E extends React.ElementType = 'span'>({ as, size = 'md', ...rest }: PolymorphicProps<E, { size?: 'sm' | 'md' | 'lg' }>) {
  const Component = as ?? 'span';
  return <Component className={`text-${size}`} {...rest} />;
}

<Text as="h1" size="lg">Title</Text>
<Text as="a" href="/deals">Deals</Text>          // href is type-checked because as="a"
<Text as={Link} to="/trips">My trips</Text>
```
The "`asChild`" (Slot) pattern from Radix is an alternative: it merges props onto the single child element.

## 9. Built-in components

### `Fragment`
- Groups children **without leaving a trace in the DOM**. The DOM is the same as if the elements were not grouped.
- Use `<Fragment key={id}>` when you need a key in a list. **React 19.3:** a `ref` on `<Fragment>` gives a `FragmentInstance` (`addEventListener`, `focus`, `observeUsing`, …).

```jsx
<>
  <td>A</td><td>B</td>
</>
// or, when you need a key:
{items.map(i => <Fragment key={i.id}><dt>{i.term}</dt><dd>{i.desc}</dd></Fragment>)}
```

### Portals (`createPortal`)
- Render children into **a different DOM node** (for example `document.body`) while they **stay in the same React tree**.
- Context, state, and **event bubbling follow the React tree**, not the DOM tree.
- Use for modals, tooltips, toasts, and escaping `overflow: hidden` or `z-index` stacking contexts.

```jsx
import { createPortal } from 'react-dom';
function Modal({ open, onClose, children }) {
  if (!open) return null;
  return createPortal(
    <div className="backdrop" onClick={onClose}><div role="dialog" aria-modal="true" onClick={e => e.stopPropagation()}>{children}</div></div>,
    document.body,
  );
}
```
Also consider native `<dialog>` and the Popover API. See [[Modern Web Platform APIs 2025-2026]] and [[Accessibility Essentials]].

### `Suspense` and `lazy` (code splitting)
- `<Suspense>` shows a **fallback until its children have finished loading**. That includes `lazy` components, `use(promise)`, and framework data.
- **Code splitting:** load only the route or component the user needs. Bundlers such as Vite see the dynamic `import()` and **leave it out of the first page load**, putting it in a separate chunk.

```tsx
import { Suspense, lazy } from 'react';
const Details = lazy(() => import('./Details'));   // module must default-export a component

export default function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Details />
    </Suspense>
  );
}
```
- Put boundaries where a loading state makes sense to the user.
- Nested boundaries reveal progressively.
- A transition (`startTransition`) keeps the old UI on screen instead of flashing back to a fallback that was already shown.
- More in [[React Performance]] and [[React Server Components & Server Functions]] (streaming).

### `StrictMode`
Development-only checks:
- renders components **an extra time**,
- runs effects **an extra time** (mount → cleanup → mount),
- runs ref callbacks an extra time (React 19),
- **warns about deprecated APIs**.

It has no effect in production.

```jsx
<StrictMode>
  <App />
</StrictMode>
```

### `Profiler`
Measures the rendering performance of a React tree **programmatically**. `onRender` receives:
- `id`: the `id` prop of the Profiler tree that just committed.
- `phase`: `"mount"`, `"update"` or `"nested-update"`.
- `actualDuration`: ms spent rendering the Profiler and its descendants **for this update**. It shows how well the subtree uses memoization.
- `baseDuration`: an estimate in ms of re-rendering the **whole subtree without any optimisation**.
- `startTime`: when React began rendering this update.
- `commitTime`: when React committed this update. It's shared by every Profiler in the commit.

```jsx
function onRender(id, phase, actualDuration, baseDuration, startTime, commitTime) {
  // aggregate or log render timings...
}
<Profiler id="App" onRender={onRender}>
  <App />
</Profiler>
```
Production profiling needs a profiling build. For interactive work, use the React DevTools Profiler or **Performance Tracks (19.2)**. See [[React Performance]].

## 10. Error Boundaries
- Components that **catch errors in their child tree**, log them, and show a **fallback UI** instead of the crashed tree.
- They catch errors **during rendering, in lifecycle methods/effects and in constructors** anywhere below them.
- They **don't catch**:
  - errors in **event handlers** (use try/catch),
  - **async code** (`setTimeout`, un-awaited promises) unless it goes back through state or `use()`,
  - **SSR** errors,
  - errors thrown **in the boundary itself**.
- **You still need a class** that implements `static getDerivedStateFromError` (to render the fallback) and/or `componentDidCatch` (to log). There is no hook equivalent.

```tsx
class ErrorBoundary extends React.Component<{ fallback: React.ReactNode; children: React.ReactNode }, { hasError: boolean }> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }          // render phase: switch to fallback
  componentDidCatch(error: Error, info: React.ErrorInfo) {                  // commit phase: side effects
    logToService(error, info.componentStack);
  }
  render() { return this.state.hasError ? this.props.fallback : this.props.children; }
}
```

In practice, use **[`react-error-boundary`](https://github.com/bvaughn/react-error-boundary)**. It gives you `FallbackComponent`, `resetKeys` and `onReset`, and a `useErrorBoundary()` hook to push async or event-handler errors into the boundary.

```tsx
import { ErrorBoundary } from 'react-error-boundary';
<ErrorBoundary FallbackComponent={({ error, resetErrorBoundary }) => (
    <div role="alert"><p>{error.message}</p><button onClick={resetErrorBoundary}>Retry</button></div>)}
  resetKeys={[searchParams.toString()]}>
  <Results />
</ErrorBoundary>
```
- **React 19** added the root options `onCaughtError` and `onUncaughtError` (on `createRoot`/`hydrateRoot`) for central error reporting.
- **React 19.3** transports `Error.cause` and `AggregateError` from server to client.
- Frameworks wrap all of this: Next.js `error.tsx`, React Router `ErrorBoundary` route exports.

## 11. Context patterns
- **Provider + custom hook** (shown above in `useTabs`). Throw if it's used outside the provider.
- **Split contexts:**
  - by **update frequency** (theme vs cursor position),
  - **state vs dispatch**: `dispatch` is stable, so components that only dispatch don't re-render.
- **Memoize the value object.** `value={{ user, login }}` is a new object every render, so every consumer re-renders. Use `useMemo`, or rely on the Compiler.
- **Don't use context as a global store for fast-changing data.** Every consumer re-renders on every change, and there are no selectors. Use Zustand/Redux (built on `useSyncExternalStore`) or TanStack Query. See [[State Management Overview]].
- **Compound components** use a private context (section 7).

```tsx
const StateCtx = createContext<State | null>(null);
const DispatchCtx = createContext<React.Dispatch<Action> | null>(null);

export function CartProvider({ children }) {
  const [state, dispatch] = useReducer(cartReducer, initialCart);
  return (
    <DispatchCtx value={dispatch}>          {/* stable → button-only consumers never re-render */}
      <StateCtx value={state}>{children}</StateCtx>
    </DispatchCtx>
  );
}
```

## Interview questions
> [!question]- What makes a component "pure", and what is `PureComponent`/`memo`?
> A pure component has no side effects during render and returns the same JSX for the same props, state and context. React relies on that for every component. `PureComponent`/`memo` is a separate *optimisation*: it shallow-compares props (and state) and skips re-rendering when they're equal.

> [!question]- HOC vs render props vs custom hooks?
> All three share logic. HOCs wrap a component and inject props. Render props pass state to a function the caller renders. Hooks share stateful logic directly inside the component, with no extra tree nodes, and they're the modern default. HOCs are still used for cross-cutting wrappers (auth, feature flags).

> [!question]- How do Error Boundaries work, and what don't they catch?
> A class component with `getDerivedStateFromError` (renders a fallback) and/or `componentDidCatch` (logs) catches errors thrown while rendering its descendants. It doesn't catch event-handler errors, async errors outside React's render, SSR errors, or its own errors. `react-error-boundary` adds reset support and a hook for async errors.

> [!question]- Why use a Portal? Do events bubble through it?
> To render outside the parent's DOM (modals, tooltips) and escape overflow or z-index. Events bubble through the **React** tree, so a click inside a portal still reaches handlers on its React ancestors, and context still works.

> [!question]- What does StrictMode do?
> In development only: it double-invokes render functions, reducers and effects (mount → unmount → mount), and ref callbacks, to find impure code and missing cleanups. It also warns about deprecated APIs. It does nothing in production.

> [!question]- How does code splitting with `lazy` work?
> `lazy(() => import('./X'))` creates a component that loads its module on first render and suspends until then. The bundler puts `X` in its own chunk. A `<Suspense>` boundary shows a fallback while it loads.

> [!question]- What are compound components? Give an example.
> A set of components that share implicit state through context and are used together declaratively, like `<Tabs><Tabs.Tab/><Tabs.Panel/></Tabs>` or `<select>`/`<option>`. They give flexible markup without prop explosion.

> [!question]- When is context the wrong tool?
> For high-frequency or large global state. Every consumer re-renders on every change and there are no selectors. Use composition first, split contexts, or use an external store with selectors.

## Related
- [[How React Renders - Reconciliation & Fiber]] · [[Hooks Reference]] · [[State, Props & Data Flow]] · [[Forms & Actions in React]] · [[React Performance]]
- [[Frontend Design Patterns]] · [[Design Systems]] · [[Design Patterns]] · [[SOLID]]
- [[Accessibility Essentials]] (dialogs, tabs roles)

## Sources
- react.dev: [Keeping Components Pure](https://react.dev/learn/keeping-components-pure) · [Fragment](https://react.dev/reference/react/Fragment) · [createPortal](https://react.dev/reference/react-dom/createPortal) · [Suspense](https://react.dev/reference/react/Suspense) · [lazy](https://react.dev/reference/react/lazy) · [StrictMode](https://react.dev/reference/react/StrictMode) · [Profiler](https://react.dev/reference/react/Profiler) · [Catching rendering errors with an error boundary](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary) · [memo](https://react.dev/reference/react/memo)
- [react-error-boundary](https://github.com/bvaughn/react-error-boundary)
- [React Profiler (indepth.dev)](https://indepth.dev/posts/1497/react-profiler)
- [A Tour of JavaScript & React Patterns (Frontend Masters)](https://frontendmasters.com/courses/tour-js-patterns/)
- [React 19](https://react.dev/blog/2024/12/05/react-19) · [React 19.3](https://react.dev/blog/2026/09/09/react-19-3)
