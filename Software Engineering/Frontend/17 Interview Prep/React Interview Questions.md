---
tags: [frontend, frontend/interview, react, interview]
aliases: [React Q&A, React Questions, Redux Interview Questions]
updated: 2026-09-27
---
# React Interview Questions

> [!tldr] TL;DR
> Model answers for every React, Redux and React Native question from the owner's "JS Questions" list and the React/Next.js sections of [[Frontend Interview Concepts & Question Bank]], plus the React 19 era questions interviewers ask in 2026 (Actions, `use`, Server Components, React Compiler, `Activity`, `useEffectEvent`). Answers are short enough to say out loud; follow the links for depth.

## 1. Fundamentals

> [!question]- What is JSX?
> A syntax extension that lets you write markup in JavaScript. The compiler turns it into function calls (`jsx('div', { className: 'x', children })` with the modern automatic runtime; `React.createElement` in the classic runtime) that return plain objects (React elements) describing the UI. Expressions go in `{}`, attributes are camelCase (`className`, `onClick`), and a component must return a single root (use a Fragment `<>...</>`).

> [!question]- What is the Virtual DOM?
> A lightweight in-memory tree of React elements that describes what the UI should look like. On each render React builds a new tree, diffs it against the previous one (reconciliation) and applies only the minimal changes to the real DOM. It is not "faster than the DOM" by itself; it makes declarative updates efficient enough and batches DOM writes. Node types: host elements (`div`), text, and components (function/class), plus fragments/portals. See [[How React Renders - Reconciliation & Fiber]].

> [!question]- Virtual DOM vs Shadow DOM vs regular DOM?
> - **DOM**: the browser's real document tree.
> - **Virtual DOM**: React's in-memory description used for diffing; a library technique.
> - **Shadow DOM**: a browser feature that attaches an encapsulated DOM subtree (with scoped styles) to an element; used by Web Components. Unrelated to performance. See [[Web Components & Shadow DOM]].

> [!question]- State vs props?
> | Props | State |
> |---|---|
> | Passed from the parent; read-only for the child | Owned by the component; changed with a setter |
> | Changes when the parent re-renders with new values | Changes via `setState` / `useState` setter → re-render |
> | Configure a component | Remember information over time |
> Both are snapshots for a given render. See [[State, Props & Data Flow]].

> [!question]- How do you "update props"?
> A child cannot change its props. The parent owns the data: lift the state to the parent and pass down both the value and a callback (`onChange`) the child calls. The parent updates its state and re-renders the child with new props. For far-away components, use context or a store.
> ```jsx
> function Parent() { const [q, setQ] = useState(''); return <Search value={q} onChange={setQ} />; }
> function Search({ value, onChange }) { return <input value={value} onChange={e => onChange(e.target.value)} />; }
> ```

> [!question]- Stateless vs stateful components? Functional vs class components?
> Stateless (presentational) components render from props only; stateful ones hold state. Historically function components were stateless and class components stateful; since hooks (16.8) function components can have state, effects and context, and are the standard. Class components still work (and are still needed for error boundaries' `componentDidCatch`/`getDerivedStateFromError`), but new code uses functions. See [[Component Patterns]].

> [!question]- What is prop drilling and how do you avoid it?
> Passing props through intermediate components that don't use them just to reach a deep child. Avoid it with **composition** (pass `children` or elements as props so intermediate components don't need to know), **context** for widely used values (theme, user, locale), or a **state library** for shared app state. Don't reach for context first; composition often solves it.

> [!question]- What is Context? How does it work with hooks?
> A way to provide a value to a whole subtree without props. Create with `createContext`, provide with `<ThemeContext value={theme}>` (React 19; `<ThemeContext.Provider>` in older versions), read with `useContext(ThemeContext)` or `use(ThemeContext)` (which can be called conditionally). Every consumer re-renders when the value changes, so memoize the value object and split contexts by update frequency.

> [!question]- What are controlled vs uncontrolled components?
> Controlled: React state is the source of truth (`value` + `onChange`); easy validation and dynamic UI. Uncontrolled: the DOM keeps the value; you read it with a ref or `FormData` on submit (`defaultValue`). React 19 form Actions make uncontrolled forms first-class: `<form action={fn}>` receives `FormData`. See [[Forms & Actions in React]].

> [!question]- What is the role of `key`?
> Keys identify elements among siblings so React can match old and new children during reconciliation. Stable, unique keys (ids) preserve state and DOM for moved items. Index keys break when items are inserted, removed or reordered (state attaches to the wrong row). Changing a key on purpose resets a component's state.

## 2. Hooks

> [!question]- Explain `useState` and `useEffect`.
> `useState` stores a value that persists across renders and returns `[value, setValue]`; calling the setter schedules a re-render. `useEffect(fn, deps)` runs `fn` **after the commit** (after paint, usually) to synchronize with external systems (subscriptions, timers, non-React widgets, logging). It re-runs when a dependency changes (compared with `Object.is`), and the returned cleanup runs before the next effect and on unmount.
> ```jsx
> const [count, setCount] = useState(0);
> useEffect(() => {
>   const id = setInterval(() => setCount(c => c + 1), 1000); // updater avoids stale closure
>   return () => clearInterval(id);
> }, []);
> ```
> Don't use effects for derived data (compute during render) or for event-driven logic (put it in the handler). See [[Hooks Reference]].

> [!question]- How does `useState` work internally?
> Each component has a Fiber node holding a linked list of hooks in call order. On each render React walks the list in the same order, so hook N always reads slot N. `setState` enqueues an update on that hook and schedules a render for the fiber; updates are processed (and batched) during the next render. This is why hooks must be called in the same order every render.

> [!question]- Can hooks be used conditionally? Why not?
> No. Hooks are identified by call order, so a hook inside an `if`, loop, or after an early return would shift the order between renders and read the wrong state. Put the condition inside the hook, or split the component. Exception: `use()` can be called conditionally.

> [!question]- How does `useEffect` differ from `componentDidMount` etc.?
> Class lifecycles are about moments (mounted, updated, will unmount); effects are about **synchronizing** with something given current props/state. One effect with deps + cleanup replaces the `didMount`/`didUpdate`/`willUnmount` trio for one concern. Effects capture props/state of their render (closures), and in Strict Mode dev React mounts → unmounts → remounts to expose missing cleanups.

> [!question]- `useEffect` vs `useLayoutEffect`?
> `useEffect` runs after the browser paints (non-blocking). `useLayoutEffect` runs after DOM mutations but **before paint**, synchronously; use it only to measure layout and adjust before the user sees a flicker (tooltips, scroll positions). It blocks painting, so keep it small; it doesn't run on the server.

> [!question]- `useReducer` vs `useState`?
> `useReducer(reducer, initial)` centralizes updates in a pure function `(state, action) => newState`. Prefer it when the next state depends on complex logic, many fields change together, or you want testable transitions (state machines). `useState` is simpler for independent values.

> [!question]- `useRef` vs `forwardRef`? How do you pass a ref to a component?
> `useRef` returns a mutable `{ current }` object that persists across renders without causing re-renders; used for DOM nodes and instance-like values (timer ids). To pass a ref to your own component: in **React 19, `ref` is a regular prop** for function components, so `forwardRef` is no longer needed (it is legacy and will be deprecated).
> ```jsx
> // React 19
> function TextInput({ ref, ...props }) { return <input ref={ref} {...props} />; }
> // React 18 and earlier
> const TextInput18 = forwardRef((props, ref) => <input ref={ref} {...props} />);
> ```
> React 19 also supports ref cleanup functions: `ref={node => { ...; return () => {...} }}`.

> [!question]- What is `useImperativeHandle`?
> Lets a component customize what a parent's ref receives, exposing a small imperative API instead of the raw DOM node.
> ```jsx
> function VideoPlayer({ ref, src }) {
>   const videoRef = useRef(null);
>   useImperativeHandle(ref, () => ({
>     play: () => videoRef.current.play(),
>     pause: () => videoRef.current.pause(),
>   }), []);
>   return <video ref={videoRef} src={src} />;
> }
> // parent: playerRef.current.play()
> ```
> Use sparingly (focus, scroll, media); prefer props for everything declarative.

> [!question]- What is memoization in React? `useMemo` vs `useCallback` vs `React.memo`?
> Memoization caches a result and reuses it while inputs are unchanged.
> - `useMemo(() => compute(a, b), [a, b])` caches a **value**.
> - `useCallback(fn, [deps])` caches a **function reference** (same as `useMemo(() => fn, deps)`).
> - `React.memo(Component)` skips re-rendering a component when its props are shallowly equal.
> `useCallback`/`useMemo` mostly matter when passing values to memoized children or effect deps. With **React Compiler 1.0** (Oct 2025) this memoization is inserted automatically at build time, so manual memo is rarely needed in compiled code. See [[React Performance]], [[React 19 to 19.3 & React Compiler]].

> [!question]- How do you optimize a React app?
> 1. Measure first (React DevTools Profiler, Chrome Performance panel with React Performance Tracks in 19.2).
> 2. Fix the state structure: keep state close to where it is used, avoid storing derived data, split contexts.
> 3. Avoid unnecessary renders: React Compiler or `memo`/`useMemo`/`useCallback` where the profiler shows it matters; stable keys.
> 4. Keep interactions responsive: `useTransition`/`useDeferredValue` for heavy updates (INP).
> 5. Render less: virtualize long lists, lazy-load routes and heavy components (`lazy` + `Suspense`).
> 6. Ship less JS: code splitting, Server Components, tree shaking, smaller dependencies.
> 7. Optimize assets: images, fonts; SSR/streaming for faster first paint.

> [!question]- What are custom hooks?
> Functions starting with `use` that call other hooks to reuse **stateful logic** (not state itself: each call gets its own state). Examples: `useDebounce`, `useFetch`, `useMediaQuery`. They replaced most HOCs and render props.

## 3. Rendering, reconciliation and Fiber

> [!question]- What is reconciliation? What is the diffing algorithm?
> Reconciliation is how React turns a new element tree into DOM updates. Comparing two arbitrary trees is O(n³), so React uses heuristics for O(n): (1) elements of **different types** produce different trees (unmount old subtree, mount new); (2) same type → keep the DOM node/instance and update changed props; (3) children lists are matched by **key**. The result is a list of effects applied in the commit phase.

> [!question]- What happens during rendering? What are React's phases?
> 1. **Trigger**: initial `root.render()` or a state update.
> 2. **Render phase**: React calls components to compute the new tree and diffs it. Pure, can be paused, restarted or thrown away (concurrent rendering).
> 3. **Commit phase**: React applies DOM mutations synchronously, attaches refs, runs `useLayoutEffect`; then the browser paints; then `useEffect`s run.
> "Rendering" in React means calling your components, not painting pixels.

> [!question]- What is React Fiber? What is in a Fiber object?
> The reconciler architecture since React 16. Each component instance/element has a **fiber**: a plain object with `type`, `key`, `props`, `stateNode` (DOM node or instance), the hooks linked list (`memoizedState`), pending updates, effect flags, and pointers `child`, `sibling`, `return` (parent), forming a linked-list tree that can be walked iteratively. Two trees exist (current and work-in-progress). Because work is split into units per fiber, React can pause, prioritize (lanes) and resume rendering, which enables concurrent features like transitions and Suspense.

> [!question]- Explain the component lifecycle.
> **Mounting** (created and inserted), **updating** (re-render from new props/state/context), **unmounting** (removed). Class methods: `constructor` → `render` → `componentDidMount`; `shouldComponentUpdate` → `render` → `getSnapshotBeforeUpdate` → `componentDidUpdate`; `componentWillUnmount`. Function equivalents: render = the function body; `useLayoutEffect`/`useEffect` with deps for mount/update; cleanup functions for unmount.
> ```jsx
> useEffect(() => {
>   const sub = source.subscribe(onData);   // mount + whenever `source` changes
>   return () => sub.unsubscribe();         // before re-run + unmount
> }, [source]);
> ```

> [!question]- What is a Pure Component? When should you use it?
> `React.PureComponent` is a class component that implements `shouldComponentUpdate` with a shallow comparison of props and state; `React.memo` is the function-component equivalent. Use for components that render often with the same props. Pitfall: new object/array/function props every render defeat it.

> [!question]- What are Higher-Order Components and how do they differ from render props?
> HOC: `const Enhanced = withX(Component)` wraps a component to inject props/behaviour. Render props: a component takes a function prop and calls it to render (`<Mouse render={pos => ...} />`). Both share logic; HOCs can cause wrapper hell and prop name collisions; render props nest deeply. Custom hooks replaced both for most cases.

> [!question]- What are Error Boundaries?
> Class components with `static getDerivedStateFromError` (render a fallback) and/or `componentDidCatch` (log) that catch errors thrown **during rendering, in lifecycles and constructors** of their children. They don't catch errors in event handlers, async code, or SSR. React 19 adds root options `onCaughtError`/`onUncaughtError`/`onRecoverableError` for reporting. Most apps use `react-error-boundary`.

> [!question]- What do Fragments, Strict Mode and Profiler do?
> `<Fragment>`/`<>` groups children without extra DOM (React 19.3 adds stable Fragment refs). `<StrictMode>` (dev only) double-invokes renders and effects to surface impure code and missing cleanups, and warns about deprecated APIs. `<Profiler id onRender>` measures render timing programmatically (the DevTools Profiler is the UI version).

## 4. React 19 era questions (2025–2026)

> [!question]- What are Actions, `useActionState` and `useOptimistic`?
> An **Action** is an async function run inside a transition; React tracks pending state, errors and optimistic updates for you. `<form action={fn}>` calls it with `FormData` and resets the form on success. `useActionState(fn, initial)` returns `[state, formAction, isPending]`. `useFormStatus` reads the parent form's pending state. `useOptimistic` shows a temporary value while the action runs, reverting when it finishes.
> ```jsx
> const [error, submit, isPending] = useActionState(async (_prev, fd) => {
>   return (await updateName(fd.get('name'))) ?? null;
> }, null);
> return <form action={submit}><input name="name" /><button disabled={isPending}>Save</button>{error && <p>{error}</p>}</form>;
> ```

> [!question]- What does `use()` do?
> Reads a **promise** (suspending until it resolves, integrating with Suspense and error boundaries) or a **context**. Unlike other hooks it can be called inside conditions and loops. Don't create the promise inside the same client component render (it would be new each time); pass it from a Server Component or a cache.

> [!question]- What are React Server Components and Server Functions?
> Server Components run only on the server (or at build time): they can access data directly, send no JS to the client, and stream their rendered output (RSC payload). Client Components (`'use client'`) are the interactive parts and hydrate in the browser. Server Functions (`'use server'`) are async functions the client can call like RPC (used as Actions). Security matters: treat Server Functions as public endpoints; the Dec 2025 CVE-2025-55182 (CVSS 10) RCE in RSC payload handling required upgrades to 19.0.1/19.1.2/19.2.1+. See [[React Server Components & Server Functions]].

> [!question]- What is the React Compiler?
> A build-time tool (1.0 in Oct 2025, `babel-plugin-react-compiler`) that automatically memoizes components and hooks, so you rarely need `useMemo`, `useCallback` or `memo`. It relies on the Rules of React (pure render, no mutation of props/state); lint rules ship in `eslint-plugin-react-hooks`. Works with React 17+ (with `react-compiler-runtime` for 17/18). Next.js 16 supports it (`reactCompiler: true`).

> [!question]- What are `<Activity>` and `useEffectEvent` (React 19.2)?
> `<Activity mode="hidden">` hides a subtree while keeping its state (effects are unmounted, rendering is low priority): great for tabs and back navigation. `useEffectEvent` extracts non-reactive logic from an effect so it can read the latest props/state without being a dependency (fixes the "stale closure vs re-subscribe" dilemma).

> [!question]- What changed in React 19.3?
> (Sept 9, 2026) `<ViewTransition>` became stable (animate enter/exit/update/shared elements with the View Transitions API), Fragment refs are stable, `browser()` from `react-dom` opts a subtree out of SSR, React DOM supports Trusted Types, and transitions render independently. See [[React 19 to 19.3 & React Compiler]].

> [!question]- `useTransition` vs `useDeferredValue`?
> `useTransition` marks a state update you trigger as non-urgent (`startTransition(() => setTab(t))`) and gives `isPending`. `useDeferredValue(value)` gives a lagging copy of a value you receive (e.g., from props or a fast-changing input) so expensive children re-render with lower priority. (The owner's note had the typo `useDifferedValue`.)

## 5. Rendering strategies, hydration and SEO (React / Next.js)

> [!question]- CSR vs SSR vs SSG vs ISR?
> | | When HTML is built | Good for | Trade-off |
> |---|---|---|---|
> | **CSR** | In the browser | Private dashboards, highly interactive apps | Slow first paint, weak SEO, big JS |
> | **SSR** | On each request | Personalized/fresh pages that need SEO | Server cost, TTFB |
> | **SSG** | At build time | Docs, marketing, blogs | Stale until rebuild; build time grows |
> | **ISR** | Statically, regenerated in the background after a revalidate time or on demand | Large catalogs (product/route pages) | Briefly stale content |
> Modern React adds **streaming SSR** with Suspense and **Server Components**; Next.js 16 adds Cache Components/PPR (static shell + dynamic holes). The owner's note used "ISG" for Incremental Static Generation; Next.js calls it ISR (Incremental Static Regeneration). See [[Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands]].

> [!question]- What is hydration? How does it differ from rendering?
> Rendering produces the HTML (on the server) or DOM (on the client). **Hydration** is React attaching event handlers and state to server-rendered HTML by rendering the same tree on the client and adopting the existing DOM instead of recreating it. The output must match; mismatches (dates, `Math.random`, `window` checks) cause warnings or client re-rendering. React 18+ supports **selective hydration** (hydrate Suspense boundaries independently, prioritizing what the user interacts with).

> [!question]- What are progressive hydration and islands architecture?
> **Progressive hydration** hydrates parts of the page over time (on idle, visibility or interaction) instead of all at once. **Islands** go further: the page is static HTML with small independent interactive "islands" that load and hydrate separately (Astro, Fresh); the rest never ships JS. Server Components get a similar result in React by keeping non-interactive parts on the server.

> [!question]- What factors impact SEO in a React/Next.js app, and how do you optimize?
> Crawlable HTML (SSR/SSG instead of CSR-only), unique titles/meta per page (Next.js Metadata API; React 19 hoists `<title>`/`<meta>`), canonical URLs, `hreflang`, sitemaps, structured data, semantic HTML, internal links, and performance (Core Web Vitals are a ranking signal). Dynamic SEO: generate metadata from data per route (`generateMetadata`) and pre-render dynamic routes. Tools: Search Console, Lighthouse, Rich Results Test, PageSpeed Insights/CrUX. See [[Core Web Vitals]].

> [!question]- How does Next.js implement dynamic routing and caching?
> App Router: folders map to routes; `[id]` dynamic segments, `[...slug]` catch-all, `(group)` route groups; `params` is a promise (must be awaited since 16). Caching in Next.js 16: everything is dynamic by default; opt in with `'use cache'` + `cacheLife`/`cacheTag`, invalidate with `revalidateTag(tag, profile)`/`updateTag()`. See [[Next.js App Router Fundamentals]], [[Next.js Rendering & Caching]].

> [!question]- Next.js SSR vs "traditional" SSR? Challenges of deploying on Vercel?
> Traditional SSR renders the whole page per request in one pass; Next.js adds streaming, Server Components, per-segment static/dynamic rendering, and a static shell with dynamic holes (PPR). On serverless platforms: cold starts of functions, execution time limits, cost at scale, region placement far from the database, and vendor-specific features. Mitigate with caching, edge/regional choices, or self-hosting (Adapters API is stable since 16.2).

> [!question]- What is a virtualized list? What is windowing?
> A **virtualized list** renders only the items visible in the viewport (plus a small overscan) instead of all items, computing which to show from the scroll position; this keeps DOM size, memory and render time small for thousands of rows. **Windowing** is that technique: the "window" is the visible slice; items outside it are removed from the DOM. Libraries: `@tanstack/react-virtual`, `react-window`. See [[Virtualization & Large Lists]].

## 6. Redux and state management

> [!question]- What is Redux and how does it work?
> A predictable state container: one **store** holds the state tree; the UI **dispatches actions** (`{ type, payload }`); **reducers** (pure functions) compute the next state; subscribers (via `useSelector`) re-render when their selected slice changes. Middleware sits between dispatch and reducer for async/logging.
> ```mermaid
> flowchart LR
>   UI -->|"dispatch(action)"| MW["Middleware (thunk, listener)"]
>   MW --> R["Reducers (pure)"]
>   R --> S["Store (state)"]
>   S -->|"useSelector"| UI
> ```
> See [[Redux & Redux Toolkit]].

> [!question]- What is Redux Toolkit (RTK)?
> The official, recommended way to write Redux. `configureStore` (good defaults, DevTools, thunk, dev checks), `createSlice` (reducers + action creators, Immer lets you write "mutating" code safely), `createAsyncThunk`, `createEntityAdapter` (normalized collections), and **RTK Query** for data fetching and caching. The legacy `createStore` is deprecated in favour of `configureStore`.

> [!question]- How do you return a function from a Redux action? (thunks)
> With the thunk middleware (included by default in `configureStore`), you can dispatch a function instead of an object; it receives `dispatch` and `getState`, so it can do async work and dispatch real actions.
> ```js
> const fetchUser = id => async (dispatch, getState) => {
>   dispatch(userLoading());
>   try { dispatch(userLoaded(await api.getUser(id))); }
>   catch (e) { dispatch(userFailed(e.message)); }
> };
> // RTK
> export const fetchUser2 = createAsyncThunk('user/fetch', id => api.getUser(id));
> ```

> [!question]- Redux Saga vs Redux Thunk?
> Thunk: simple functions with async/await; easy to learn; fine for most request/response flows. Saga: generator-based middleware that listens for actions and runs "sagas" with declarative effects (`takeLatest`, `race`, `fork`, `cancel`); powerful for complex flows (debounced search, cancellation, websockets, orchestration) and easy to test, but more boilerplate and a learning curve. Today RTK's **listener middleware** and **RTK Query** cover most saga use cases.

> [!question]- Why does a reducer need a default case?
> Every action goes through every reducer. For actions a reducer doesn't handle, it must **return the existing state unchanged** (otherwise it returns `undefined` and wipes that slice). Also, Redux calls reducers with an init action and `state === undefined` to get initial state, so a default parameter is needed. `createSlice` handles both automatically.
> ```js
> function counter(state = 0, action) {
>   switch (action.type) { case 'inc': return state + 1; default: return state; }
> }
> ```

> [!question]- Is Redux single store or multi store?
> Single store by design (one source of truth), split into slices with `combineReducers`/slices. Multiple stores are possible but discouraged (breaks DevTools, middleware and cross-slice logic).

> [!question]- What is Reselect?
> A library (built into RTK as `createSelector`) for **memoized selectors**: derived data is recomputed only when inputs change, and returns the same reference otherwise, so `useSelector` doesn't trigger re-renders for unchanged derived arrays/objects.
> ```js
> const selectVisibleTodos = createSelector(
>   [state => state.todos, state => state.filter],
>   (todos, filter) => todos.filter(t => filter === 'all' || t.status === filter)
> );
> ```

> [!question]- Redux vs Context?
> Context is a **dependency-injection mechanism**, not a state manager: every consumer re-renders when the value changes, with no selectors, middleware or DevTools. Great for low-frequency global values (theme, locale, auth user). Redux (or Zustand, etc.) is for frequently changing shared state with selectors, predictable updates, middleware and tooling. Server data belongs in RTK Query/TanStack Query rather than either. See [[State Management Overview]].

## 7. React Native

> [!question]- What is React Native and how does it work?
> A framework (now owned by the React Foundation with React and JSX, since Feb 2026) for building native iOS/Android apps with React. Your JS runs in a JS engine (Hermes by default); React renders to **native views** (`View` → `UIView`/`android.view.View`), not to a WebView. Layout uses Yoga (flexbox). The **New Architecture** (default in recent versions) replaces the old bridge with **JSI** (JavaScript Interface: direct C++ bindings between JS and native), **Fabric** (the new concurrent renderer) and **TurboModules** (lazily loaded native modules).

> [!question]- What is "bridging"?
> In the old architecture, JS and native communicated over an asynchronous **bridge** that serialized messages as JSON in batches; heavy traffic (animations, lists) caused lag. "Bridging" also means writing a **native module** to expose platform APIs to JS (Swift/Kotlin code callable from JS). With the New Architecture, JSI lets JS hold references to C++ objects and call them synchronously, and TurboModules/codegen generate typed bindings.

> [!question]- How do you handle dates in React Native?
> Same as the web: store UTC ISO strings, format with `Intl` (Hermes supports `Intl` in modern versions) or `date-fns`/Day.js; use `@react-native-community/datetimepicker` for native pickers; be careful with device time zone and locale settings. See the date answer in [[JavaScript Interview Questions]].

> [!question]- Background services, Realm DB, UI libraries? (experience questions)
> Answer from experience, but know the landscape:
> - **Background work**: headless JS (Android), background fetch/tasks (`expo-background-task`/`react-native-background-fetch`), push notifications to wake the app; OS limits are strict on iOS.
> - **Local DB**: Realm (object DB, reactive queries; note that Atlas Device Sync was deprecated by MongoDB in 2024, so the local DB continues community-driven), SQLite (`expo-sqlite`, op-sqlite), WatermelonDB, MMKV for key-value.
> - **UI libraries**: React Native Paper (Material), Tamagui, NativeWind (Tailwind), gluestack; or a custom design system on top of primitives. Choose by design language, performance, theming and maintenance.

> [!question]- Flexbox differences in React Native?
> Everything is flexbox by default, `flexDirection` defaults to `column` (web: `row`), `flex` takes a single number, no CSS cascade (styles are per component objects), and units are density-independent pixels.

## Related
- [[React Index]] · [[JavaScript Interview Questions]] · [[Output Questions & Tricky Snippets]]
- [[Hooks Reference]] · [[How React Renders - Reconciliation & Fiber]] · [[React Performance]] · [[React 19 to 19.3 & React Compiler]]
- [[Redux & Redux Toolkit]] · [[State Management Overview]] · [[Next.js Rendering & Caching]]
- [[Frontend Interview Concepts & Question Bank]] · [[Advance React Book]]

## Sources
- Owner's notes: `Frontend/Front End.md` (JS Questions list), `Interviews/Interview.md` (Concepts, Questions, Islands, Progressive Hydration, Virtualized List)
- react.dev reference: https://react.dev/reference/react
- React 19: https://react.dev/blog/2024/12/05/react-19 · 19.2: https://react.dev/blog/2025/10/01/react-19-2 · 19.3: https://react.dev/blog/2026/09/09/react-19-3
- React Compiler 1.0: https://react.dev/blog/2025/10/07/react-compiler-1
- RSC security advisory: https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components
- React Foundation: https://react.dev/blog/2026/02/24/the-react-foundation
- Redux Toolkit: https://redux-toolkit.js.org · Redux style guide: https://redux.js.org/style-guide/
- React Native architecture: https://reactnative.dev/architecture/landing-page
- patterns.dev (islands, progressive hydration): https://www.patterns.dev/
- Next.js 16: https://nextjs.org/blog/next-16
