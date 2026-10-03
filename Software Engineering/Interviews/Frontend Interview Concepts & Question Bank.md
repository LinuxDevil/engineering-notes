---
tags: [interviews, interviews/frontend]
aliases: [Interview, Frontend Interview Cheat Sheet, FE Question Bank]
updated: 2026-09-27
---
# Frontend Interview Concepts & Question Bank

> [!tldr] TL;DR
> A one-page revision sheet of the frontend concepts that come up again and again in interviews (JavaScript, React, Next.js, CSS/HTML, rendering, patterns), followed by the question bank with short answers you can say out loud. Each answer links to the full topic note. Corrections to the original notes are marked **(fixed)**.

> [!info] How to use this note
> 1. Read the concept sheet once, fast.
> 2. Open the question bank, try to answer each question aloud **before** expanding it.
> 3. For anything you got wrong, follow the link and read the full note. For deeper answer sets see [[JavaScript Interview Questions]], [[React Interview Questions]] and [[Output Questions & Tricky Snippets]].

## Part 1: Concept sheet

### JavaScript basics
| # | Concept | Short answer |
|---|---|---|
| 1 | `var` vs `let` vs `const` | `var`: function-scoped, can be redeclared, hoisted and initialised to `undefined`. `let`: block-scoped, no redeclaration, hoisted but uninitialised (TDZ). `const`: like `let` but the **binding** cannot be reassigned; the object it points to is still mutable **(fixed: "immutable reference" means the binding, not the value)**. See [[Hoisting, TDZ & var-let-const]]. |
| 2 | Primitive vs non-primitive | Primitives: `string`, `number`, `bigint`, `boolean`, `undefined`, `null`, `symbol` (copied by value, immutable). Everything else is an object (arrays, functions, dates, maps), passed as a reference. See [[Types, Coercion & Equality]]. |
| 3 | Scopes | Global, module, function and block scope. JS is **lexically** scoped: where code is written decides what it can see. See [[Scope & Lexical Environment]]. |
| 4 | Arrow vs regular functions | Arrow: lexical `this`, no own `arguments`, cannot be used with `new`, no `prototype`. Regular: `this` depends on how it is called. See [[this, call, apply & bind]]. |
| 5 | Objects | Mutable collections of key → value properties with a prototype chain. See [[Objects, Arrays & Immutability]]. |
| 6 | `this` | Decided at **call time**: method call → the object; plain call → `undefined` in strict mode (global object in sloppy mode); `new` → the new instance; `call/apply/bind` → explicit; arrow → inherited from the enclosing scope. |
| 7 | Event loop | Call stack runs sync code. When it empties, the event loop drains **all microtasks** (promise callbacks, `queueMicrotask`, `MutationObserver`), then maybe renders, then takes **one macrotask** (timers, I/O, UI events). See [[Event Loop - Microtasks & Macrotasks]]. |
| 8 | Promises | An object for a future value with three states (pending, fulfilled, rejected); `then`/`catch`/`finally` register callbacks that run as microtasks. See [[Promises, Async-Await & Cancellation]]. |
| 9 | Set vs Map | `Set`: unique values. `Map`: key → value with any key type, insertion order kept, `size` is O(1). Union: `new Set([...a, ...b])` or `a.union(b)` (Set methods, ES2025). Merge maps: `new Map([...m1, ...m2])`. See [[Collections - Map, Set, WeakMap & WeakRef]]. |
| 10 | TDZ | The time between entering a scope and the `let`/`const`/`class` declaration line; reading the variable then throws `ReferenceError`. |
| 11 | Hoisting | Declarations are registered during the creation phase of the execution context before code runs. Nothing physically moves **(fixed: "move to the top" is only a metaphor)**. Function declarations are fully available; `var` is `undefined`; `let`/`const`/`class` are in the TDZ. |
| 12 | `async`/`await` | Syntax over promises: `await` pauses the async function and resumes it in a microtask when the promise settles. |
| 13 | Callback hell | Deeply nested callbacks with scattered error handling. Fixed with promises, `async/await` and small named functions. |
| 14 | `null` vs `undefined` | `undefined`: declared but not assigned, missing property, missing argument. `null`: an intentional "no value" set by the programmer. `typeof null === "object"` is a historical bug **(fixed: the original said null "has a memory address"; that is not how the spec defines it)**. |
| 15 | Higher-order functions | Take or return functions (`map`, `filter`, `reduce`, `debounce`). See [[Functions - HOF, Callbacks & Currying]]. |
| 16 | `setTimeout` | Schedules a macrotask **no sooner than** the delay; it runs only when the stack and microtask queue are empty, so the delay is a minimum, not a guarantee. Nested timers are clamped to ≥4 ms. |
| 17 | Promise combinators | `all`: fulfils when all fulfil, rejects on the first rejection. `allSettled`: waits for all, never rejects. `race`: **settles** (fulfils or rejects) like the first to settle **(fixed)**. `any`: fulfils with the first fulfilment, rejects with `AggregateError` if all reject. `finally` is an instance method, not a combinator. |
| 18 | HTML in memory | The parser turns HTML into the **DOM**, a tree of node objects in memory (not a string) **(fixed)**. See [[How Browsers Render - Critical Rendering Path]]. |
| 19 | React Fiber tree | Each fiber links to `child`, `sibling` and `return` (parent), so the tree is walked as a linked structure that React can pause and resume. See [[How React Renders - Reconciliation & Fiber]]. |

### React
| # | Concept | Short answer |
|---|---|---|
| 20 | Lifecycle | Mount → update → unmount. In function components: render, then effects run after commit; cleanup runs before the next effect and on unmount. |
| 21 | Shadow DOM vs Virtual DOM | Shadow DOM: a browser feature for **encapsulated** DOM subtrees (Web Components). Virtual DOM: React's in-memory description of UI used to compute changes. Unrelated ideas. See [[Web Components & Shadow DOM]]. |
| 22 | Real DOM vs Virtual DOM | The VDOM is not "faster than the DOM"; it lets React **batch and minimise** DOM writes by diffing. Hand-written targeted DOM updates can be faster **(fixed)**. |
| 23 | Element types | Host elements (`div`), text, components (function/class), fragments, portals, context providers, Suspense boundaries. |
| 24 | Error boundaries | Class components with `getDerivedStateFromError`/`componentDidCatch` that catch **render** errors in their subtree (not event handlers or async code). |
| 25 | State vs props | State: owned by the component, changed only through its setter and treated as **immutable** **(fixed: "mutable" is misleading)**. Props: passed in by the parent, read-only. See [[State, Props & Data Flow]]. |
| 26 | Controlled vs uncontrolled | Controlled: the value lives in React state. Uncontrolled: the DOM keeps the value, read via ref or `FormData` (React 19 form Actions favour this). See [[Forms & Actions in React]]. |
| 27 | `useEffect` vs `useLayoutEffect` | `useEffect` runs after paint; `useLayoutEffect` runs after DOM mutation but before paint (for measuring layout). |
| 28 | `useRef` vs `forwardRef` | `useRef`: mutable box that survives renders without re-rendering. `forwardRef` passed refs to children; **since React 19 `ref` is a normal prop** for function components and `forwardRef` is no longer needed. |
| 29 | Reconciliation | Diff the new element tree against the previous one: different type → replace subtree; same type → update props; lists matched by `key`. O(n) heuristic. |
| 30 | Fiber object | A unit of work holding type, props, state/hooks list, effects flags, lanes (priority) and pointers to child/sibling/return and the alternate (double buffering). |
| 31 | Prop drilling | Passing props through layers that don't use them. Fix with composition (`children`), context, or a state library. |
| 32 | Context API | Broadcasts a value to a subtree; every consumer re-renders when the value changes, so split contexts and memoise values. |
| 33 | Built-in components | `Fragment`, `StrictMode`, `Profiler`, `Suspense`, and `Activity` (React 19.2). |
| 34 | `useReducer` | State transitions in a pure reducer `(state, action) => newState`; good for complex, related state. |

### Next.js
| # | Concept | Short answer |
|---|---|---|
| 35 | SSR, SSG, **ISR**, CSR | SSR: HTML per request. SSG: HTML at build. ISR (**Incremental Static Regeneration**, **fixed from "ISG"**): static pages regenerated in the background after a revalidate time or on demand. CSR: HTML shell, JS renders in the browser. See [[Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands]]. |
| 36 | Caching | Next.js caches fetched data, rendered routes and client router payloads; Next.js 16 made caching explicit with Cache Components and `"use cache"`. See [[Next.js Rendering & Caching]]. |
| 37 | SEO | Pre-rendered HTML, the Metadata API (`generateMetadata`), sitemaps, canonical URLs, structured data, good Core Web Vitals. |
| 38 | Dynamic SEO | `generateMetadata` per dynamic route, fed from the same data as the page. |
| 39 | Ranking | Content relevance and quality first; page experience (Core Web Vitals, HTTPS, mobile) is a secondary signal. |
| 40 | Web Vitals | LCP (≤2.5 s), INP (≤200 ms), CLS (≤0.1) at the 75th percentile. See [[Core Web Vitals]]. |
| 41 | Serverless trade-offs | Functions can have cold starts, execution limits and per-invocation cost; Vercel's Fluid compute (2025) reuses instances to reduce cold starts. |

### CSS & HTML
| # | Concept | Short answer |
|---|---|---|
| 42 | Semantic HTML | Elements that describe meaning (`<nav>`, `<main>`, `<article>`, `<button>`): better accessibility, SEO and default behaviour. See [[Accessibility Essentials]]. |
| 43 | `defer`, `async`, prefetch | `defer`: download in parallel, run after parsing in order. `async`: run as soon as downloaded, any order. `<link rel="prefetch">`: low-priority fetch for a **future** navigation; `preload`: high-priority fetch for the current page. |
| 44 | HTML parsing | Bytes → tokens → nodes → DOM; `<script>` without `defer/async` blocks parsing; CSS blocks rendering. |
| 45 | Functional (utility) CSS | Single-purpose classes (Tailwind): no naming, small CSS that stops growing, but noisy markup. See [[Styling Approaches]]. |
| 46 | CSS performance | Animate `transform` and `opacity` (compositor only, often GPU); avoid animating layout properties; use `will-change` sparingly; `content-visibility: auto` for off-screen sections. |
| 47 | Box model | Content, padding, border, margin; `box-sizing: border-box` makes `width` include padding and border. |
| 48 | CSSOM | The object model built from all CSS; combined with the DOM into the render tree. |
| 49 | Accessibility tree | A **separate** tree the browser derives from the DOM (roles, names, states) and exposes to assistive tech; not simply a subset of the DOM **(fixed)**. |

### Rendering and architecture notes
- **Islands architecture**: small, independent islands of interactivity inside server-rendered static HTML. Each island has its own script and hydrates on its own; the rest of the page stays static (Astro, Fresh). See [[Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands]].
- **Progressive hydration**: hydrate only critical parts on load and the rest later (when visible or idle). React 18+ does **selective hydration** with Suspense boundaries.
- **Observer pattern**: observers subscribe to an observable; when something happens, the observable notifies all of them (DOM events, RxJS, stores). See [[Frontend Design Patterns]].
- **Tree shaking**: the bundler removes exports that are never imported. Needs static ESM and side-effect-free modules. See [[Build Tools & Bundlers]].
- **Virtualized list / windowing**: render only the rows in (or near) the viewport and recycle them while scrolling, so the DOM stays small for lists with thousands of items. Windowing is the "visible window" technique that virtualization libraries (TanStack Virtual, react-window) implement. See [[Virtualization & Large Lists]].

## Part 2: Question bank

### CSS & HTML
> [!question]- What is semantic HTML, and why is it important?
> Using elements for their meaning (`<header>`, `<nav>`, `<main>`, `<button>`, `<label>`). Screen readers get roles and landmarks for free, keyboard behaviour comes built in, search engines understand structure, and the code is easier to read.

> [!question]- How do `defer`, `async` and prefetch differ?
> `defer` and `async` are script attributes that download without blocking the parser; `defer` runs after parsing, in document order; `async` runs as soon as it arrives. `rel="prefetch"` is a resource hint that fetches something at low priority for a likely **next** navigation. Use `preload` for critical resources of the current page.

> [!question]- What tools can analyse the accessibility of a page?
> Lighthouse and the Chrome DevTools accessibility pane (accessibility tree, contrast), axe DevTools / `axe-core` in tests, WAVE, eslint-plugin-jsx-a11y, Playwright's accessibility snapshots, and manual testing with a keyboard and a screen reader (NVDA, VoiceOver). Automated tools catch roughly a third of issues; manual testing is required.

> [!question]- How do you push work to the GPU instead of the CPU?
> Animate only `transform` and `opacity` so the browser can do it on the compositor thread, and promote the element to its own layer (`will-change: transform`, only while needed). Too many layers waste memory.

> [!question]- What is BEM?
> Block, Element, Modifier: a class naming convention (`card`, `card__title`, `card--featured`) that keeps specificity flat and makes relationships visible. Less needed with CSS Modules, scoped styles or `@scope`.

> [!question]- What is the CSSOM and how is it evaluated?
> The browser parses all stylesheets into the CSS Object Model. CSS is render-blocking: the browser needs the full CSSOM before building the render tree (DOM + computed styles), then layout, paint and composite.

> [!question]- What is functional CSS and what are its benefits?
> Utility-first CSS: small single-purpose classes composed in markup. Benefits: no naming, no dead CSS, consistent design tokens, CSS size plateaus. Costs: long class lists, need for components to avoid repetition.

> [!question]- What is CSS specificity?
> The weight used to pick between conflicting rules: inline styles > IDs > classes/attributes/pseudo-classes > elements/pseudo-elements, compared as a tuple. `!important`, cascade layers (`@layer`) and source order also matter. `:where()` has zero specificity. See [[CSS Fundamentals - Cascade, Box Model & Contexts]].

### React hooks
> [!question]- What is `useState` for and how does it work internally?
> It stores state for a component between renders. React keeps a linked list of hooks on the component's fiber; each `useState` call reads its slot by call order. `setState` queues an update and schedules a re-render; updates are batched.

> [!question]- How does `useEffect` differ from `componentDidMount`?
> Effects synchronise with external systems after commit and run after every render whose dependencies changed, not only on mount. The cleanup handles unmount and re-runs. Think "sync with X", not lifecycle events.

> [!question]- What is `useReducer` and how does it differ from `useState`?
> `useReducer` centralises transitions in a pure reducer, good when next state depends on several fields or many actions. `useState` is simpler for independent values. `useState` is implemented on top of the same mechanism.

> [!question]- How does the Context API work with hooks?
> `createContext` + a provider, read with `useContext(Ctx)` (or `use(Ctx)` in React 19, which can be called conditionally). Consumers re-render when the provider value changes identity.

> [!question]- Can hooks be called conditionally? Why not?
> No. React identifies each hook by call order on every render; a condition would shift the order and give a hook another hook's state. The exception is `use()`, which can be called inside conditions.

### React rendering and lifecycle
> [!question]- How does React apply Virtual DOM changes to the real DOM?
> Render phase: call components, build the new fiber tree, diff against the current one and collect effects (interruptible). Commit phase: apply the DOM mutations synchronously, then run layout effects, then passive effects.

> [!question]- What happens during a React render?
> A state update schedules work with a priority lane. React re-runs the affected components, reconciles children, and if something changed, commits DOM updates. Rendering is pure and may be restarted or discarded.

> [!question]- Explain the component lifecycle with examples.
> Mount: first render, DOM inserted, effects run (fetch data, subscribe). Update: props/state change → re-render → effects with changed deps re-run after their cleanup. Unmount: cleanups run (unsubscribe, clear timers).

> [!question]- What are React's phases when updating?
> Trigger (setState), render (compute and diff), commit (mutate DOM, layout effects), then passive effects (`useEffect`) after paint.

> [!question]- How do function components handle lifecycle compared to classes?
> Classes use `componentDidMount/DidUpdate/WillUnmount`. Function components use `useEffect` with dependencies and a cleanup; `useLayoutEffect` for synchronous DOM reads.

### Types of React components
> [!question]- Function vs class components?
> Function components are plain functions using hooks; classes use `this.state` and lifecycle methods. All new React features (hooks, Server Components, compiler) target function components. Error boundaries still require a class.

> [!question]- What are Pure Components?
> `React.PureComponent` (class) or `React.memo` (function) skip re-rendering when props are shallowly equal. With the React Compiler (1.0, Oct 2025) most manual memoisation becomes unnecessary. See [[React Performance]].

> [!question]- HOCs vs render props?
> Both share logic. A HOC wraps a component and returns a new one; a render prop passes a function child that renders. Custom hooks replaced most uses of both. See [[Component Patterns]].

> [!question]- Controlled vs uncontrolled components?
> Controlled: `value` + `onChange` in state, React is the source of truth (instant validation). Uncontrolled: `defaultValue`, read via ref or `FormData` on submit (less re-rendering).

> [!question]- What is `forwardRef` and when is it used?
> It let a function component receive a `ref` and pass it to a DOM node. Since React 19, `ref` is a regular prop for function components, so `forwardRef` is only needed for older React versions.

### Reconciliation
> [!question]- What is reconciliation and how does React find the minimal updates?
> It diffs the new tree against the old using two heuristics: elements of different types produce different trees, and `key`s identify children across renders. This makes the diff O(n) instead of O(n³).

> [!question]- What is the role of keys?
> Keys give list items a stable identity so React can reorder, insert and remove without destroying state. Use stable ids, not array indexes when the list can reorder.

> [!question]- What is React Fiber?
> The reconciler rewrite (React 16) where each element becomes a fiber unit of work. It lets React split rendering into chunks, pause, prioritise (lanes) and resume, which powers concurrent features and Suspense.

### Rendering strategies
> [!question]- What is SSR and when would you use it?
> HTML rendered on the server per request: fast first paint and SEO for personalised or frequently changing pages. Cost: server work per request and a hydration step.

> [!question]- SSG vs SSR?
> SSG renders at build time and serves from a CDN (fastest, cheapest) but data is as old as the last build. SSR renders on every request (fresh, slower, more server cost).

> [!question]- What is ISR and how does it work?
> Static pages with a revalidation rule: after the revalidate time, the next request gets the cached page while Next.js regenerates it in the background (stale-while-revalidate); `revalidatePath`/`revalidateTag` trigger it on demand.

> [!question]- How does CSR affect performance and SEO?
> Empty HTML shell until JS downloads and runs: slower LCP on slow devices and weaker SEO for crawlers that don't run JS well. Fine for dashboards behind a login.

> [!question]- ISR vs SSG trade-offs?
> ISR avoids full rebuilds for large sites and keeps data fresher, but users may see stale content for one revalidation window and you need a server/runtime that supports it.

### Hydration and SEO
> [!question]- What is hydration?
> Attaching React's event handlers and state to server-rendered HTML by rendering the same tree on the client and adopting the existing DOM. Mismatches between server and client output cause warnings or re-rendering.

> [!question]- How does hydration differ from rendering?
> Rendering produces the UI (HTML on the server or DOM on the client). Hydration reuses existing HTML and only makes it interactive.

> [!question]- What affects SEO in React/Next.js apps?
> Crawlable HTML (SSR/SSG), metadata and canonical tags, clean URLs, sitemaps, structured data, internal links, performance (Core Web Vitals), mobile friendliness, no content hidden behind client-only rendering.

> [!question]- How do you optimise Web Vitals in Next.js?
> `next/image` with sizes and priority for the LCP image, `next/font`, streaming with Suspense, fewer client components and less JS, avoiding layout shifts with reserved sizes, and breaking long tasks for INP.

> [!question]- Which tools find SEO issues?
> Lighthouse, Google Search Console (coverage, Core Web Vitals report), PageSpeed Insights (CrUX field data), the Rich Results Test, and crawlers like Screaming Frog.

### Critical rendering path
> [!question]- What is the critical rendering path?
> The sequence of steps a browser takes to turn HTML, CSS and JS into pixels: build DOM, build CSSOM, run blocking scripts, build the render tree, layout, paint, composite. See [[How Browsers Render - Critical Rendering Path]].

> [!question]- How do you optimise it?
> Fewer critical resources, bytes and round trips: inline critical CSS, `defer` scripts, preload the LCP resource, compress, use HTTP/2+ and a CDN, avoid render-blocking third parties.

> [!question]- What role do CSS and JS play?
> CSS blocks rendering (the CSSOM is required). Classic scripts block parsing and wait for preceding CSS, because they may read styles.

> [!question]- How does the browser prioritise work?
> The preload scanner fetches resources early; fetch priority depends on type and position (`fetchpriority` can adjust it); the main thread runs tasks, and rendering happens between tasks at frame boundaries.

> [!question]- Which tools analyse the critical rendering path?
> Chrome DevTools Performance and Network panels, Lighthouse, WebPageTest waterfalls, and the Performance Insights for LCP phases.

### JavaScript
> [!question]- `var`, `let` and `const`?
> See concept 1: scope (function vs block), redeclaration, TDZ, and reassignment of the binding.

> [!question]- How does `this` behave in arrow vs regular functions?
> Arrow functions take `this` from the surrounding scope at definition time; regular functions get it from the call site. That's why arrow functions are good for callbacks inside methods and bad as object methods.

> [!question]- Explain the event loop and async tasks.
> Sync code runs on the stack; async APIs hand work to the host; when done they queue callbacks. After each macrotask, all microtasks run, then the browser may render. Example: `console.log(1); setTimeout(()=>log(2)); Promise.resolve().then(()=>log(3)); log(4)` prints 1 4 3 2.

> [!question]- How does `Promise.all` work and what are the alternatives?
> It takes an iterable and returns a promise of all results in input order, rejecting on the first rejection (the other promises keep running). Use `allSettled` when you need every outcome, `any` for the first success, `race` for timeouts.

### React advanced
> [!question]- How do React Profiler and Strict Mode help?
> The Profiler (DevTools or `<Profiler>`) measures what rendered and why. Strict Mode double-invokes renders and effects in development to reveal impure renders and missing cleanups.

> [!question]- How do you avoid prop drilling?
> Composition (pass components as children), context for widely shared values, or a store (Zustand, Redux Toolkit) for app state; server state in TanStack Query.

### Next.js
> [!question]- How does Next.js do dynamic routing?
> File-system routes with segments like `app/blog/[slug]/page.tsx`, catch-all `[...slug]`, and `generateStaticParams` to pre-render known params.

> [!question]- How do you configure caching in Next.js?
> In Next.js 16: enable Cache Components and mark cacheable functions/components with `"use cache"`, control lifetime with `cacheLife` and invalidate with `cacheTag`/`revalidateTag`. Older App Router: `fetch` options (`cache`, `next.revalidate`) and route segment config.

> [!question]- Next.js SSR vs "traditional" SSR?
> Traditional SSR renders a full page template per request. Next.js adds React Server Components (zero client JS for server parts), streaming with Suspense, partial pre-rendering, and per-segment caching.

> [!question]- How does Next.js improve SEO compared to plain React (CRA/Vite SPA)?
> It sends pre-rendered HTML with metadata, so crawlers and link previews see content immediately, and it improves LCP.

> [!question]- Challenges of deploying Next.js on serverless platforms?
> Cold starts, function duration and size limits, cache storage across instances (needs a shared cache for self-hosting), cost per invocation, and vendor-specific features. Self-hosting needs a cache handler for multiple instances.

### Other
> [!question]- How do you approach A/B testing with multiple variants?
> Hypothesis and primary metric first, sticky assignment per user, enough sample size per variant (correct for multiple comparisons), server/edge assignment to avoid flicker, log exposure when seen, fixed stopping rule. Full answer in [[JavaScript Interview Questions]].

## Related
- [[Interviews Index]]
- [[JavaScript Interview Questions]] · [[React Interview Questions]] · [[Output Questions & Tricky Snippets]] · [[Angular Interview Q&A]]
- [[Frontend Home]] · [[Frontend System Design Index]]
- [[Online Coding Assessment - Bit Count, Consecutive Path & Missing Rolls]]

## Sources
- [MDN Web Docs](https://developer.mozilla.org/)
- [react.dev: Preserving and resetting state](https://react.dev/learn/preserving-and-resetting-state) and [React 19 release post](https://react.dev/blog/2024/12/05/react-19)
- [Next.js docs](https://nextjs.org/docs)
- [web.dev: Critical rendering path](https://web.dev/learn/performance/understanding-the-critical-path)
- Owner's original interview notes (restructured here)
