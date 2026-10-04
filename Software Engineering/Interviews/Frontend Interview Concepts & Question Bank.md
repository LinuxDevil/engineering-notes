---
title: "Frontend Interview Concepts & Question Bank"
tags: [interviews, interviews/frontend]
aliases: [Interview, Frontend Interview Cheat Sheet, FE Question Bank]
updated: 2026-09-27
---
# Frontend Interview Concepts & Question Bank

<Note>
**TL;DR**

A one-page revision sheet of the frontend concepts that come up again and again in interviews (JavaScript, React, Next.js, CSS/HTML, rendering, patterns), followed by the question bank with short answers you can say out loud. Each answer links to the full topic note. Corrections to the original notes are marked **(fixed)**.
</Note>

<Info>
**How to use this note**

1. Read the concept sheet once, fast.
2. Open the question bank, try to answer each question aloud **before** expanding it.
3. For anything you got wrong, follow the link and read the full note. For deeper answer sets see [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions), [React Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/React%20Interview%20Questions) and [Output Questions & Tricky Snippets](/Software%20Engineering/Frontend/17%20Interview%20Prep/Output%20Questions%20%26%20Tricky%20Snippets).
</Info>

## Part 1: Concept sheet

### JavaScript basics
| # | Concept | Short answer |
|---|---|---|
| 1 | `var` vs `let` vs `const` | `var`: function-scoped, can be redeclared, hoisted and initialised to `undefined`. `let`: block-scoped, no redeclaration, hoisted but uninitialised (TDZ). `const`: like `let` but the **binding** cannot be reassigned; the object it points to is still mutable **(fixed: "immutable reference" means the binding, not the value)**. See [Hoisting, TDZ & var-let-const](/Software%20Engineering/Frontend/01%20JavaScript/Hoisting%2C%20TDZ%20%26%20var-let-const). |
| 2 | Primitive vs non-primitive | Primitives: `string`, `number`, `bigint`, `boolean`, `undefined`, `null`, `symbol` (copied by value, immutable). Everything else is an object (arrays, functions, dates, maps), passed as a reference. See [Types, Coercion & Equality](/Software%20Engineering/Frontend/01%20JavaScript/Types%2C%20Coercion%20%26%20Equality). |
| 3 | Scopes | Global, module, function and block scope. JS is **lexically** scoped: where code is written decides what it can see. See [Scope & Lexical Environment](/Software%20Engineering/Frontend/01%20JavaScript/Scope%20%26%20Lexical%20Environment). |
| 4 | Arrow vs regular functions | Arrow: lexical `this`, no own `arguments`, cannot be used with `new`, no `prototype`. Regular: `this` depends on how it is called. See [this, call, apply & bind](/Software%20Engineering/Frontend/01%20JavaScript/this%2C%20call%2C%20apply%20%26%20bind). |
| 5 | Objects | Mutable collections of key → value properties with a prototype chain. See [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability). |
| 6 | `this` | Decided at **call time**: method call → the object; plain call → `undefined` in strict mode (global object in sloppy mode); `new` → the new instance; `call/apply/bind` → explicit; arrow → inherited from the enclosing scope. |
| 7 | Event loop | Call stack runs sync code. When it empties, the event loop drains **all microtasks** (promise callbacks, `queueMicrotask`, `MutationObserver`), then maybe renders, then takes **one macrotask** (timers, I/O, UI events). See [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks). |
| 8 | Promises | An object for a future value with three states (pending, fulfilled, rejected); `then`/`catch`/`finally` register callbacks that run as microtasks. See [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation). |
| 9 | Set vs Map | `Set`: unique values. `Map`: key → value with any key type, insertion order kept, `size` is O(1). Union: `new Set([...a, ...b])` or `a.union(b)` (Set methods, ES2025). Merge maps: `new Map([...m1, ...m2])`. See [Collections - Map, Set, WeakMap & WeakRef](/Software%20Engineering/Frontend/01%20JavaScript/Collections%20-%20Map%2C%20Set%2C%20WeakMap%20%26%20WeakRef). |
| 10 | TDZ | The time between entering a scope and the `let`/`const`/`class` declaration line; reading the variable then throws `ReferenceError`. |
| 11 | Hoisting | Declarations are registered during the creation phase of the execution context before code runs. Nothing physically moves **(fixed: "move to the top" is only a metaphor)**. Function declarations are fully available; `var` is `undefined`; `let`/`const`/`class` are in the TDZ. |
| 12 | `async`/`await` | Syntax over promises: `await` pauses the async function and resumes it in a microtask when the promise settles. |
| 13 | Callback hell | Deeply nested callbacks with scattered error handling. Fixed with promises, `async/await` and small named functions. |
| 14 | `null` vs `undefined` | `undefined`: declared but not assigned, missing property, missing argument. `null`: an intentional "no value" set by the programmer. `typeof null === "object"` is a historical bug **(fixed: the original said null "has a memory address"; that is not how the spec defines it)**. |
| 15 | Higher-order functions | Take or return functions (`map`, `filter`, `reduce`, `debounce`). See [Functions - HOF, Callbacks & Currying](/Software%20Engineering/Frontend/01%20JavaScript/Functions%20-%20HOF%2C%20Callbacks%20%26%20Currying). |
| 16 | `setTimeout` | Schedules a macrotask **no sooner than** the delay; it runs only when the stack and microtask queue are empty, so the delay is a minimum, not a guarantee. Nested timers are clamped to ≥4 ms. |
| 17 | Promise combinators | `all`: fulfils when all fulfil, rejects on the first rejection. `allSettled`: waits for all, never rejects. `race`: **settles** (fulfils or rejects) like the first to settle **(fixed)**. `any`: fulfils with the first fulfilment, rejects with `AggregateError` if all reject. `finally` is an instance method, not a combinator. |
| 18 | HTML in memory | The parser turns HTML into the **DOM**, a tree of node objects in memory (not a string) **(fixed)**. See [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path). |
| 19 | React Fiber tree | Each fiber links to `child`, `sibling` and `return` (parent), so the tree is walked as a linked structure that React can pause and resume. See [How React Renders - Reconciliation & Fiber](/Software%20Engineering/Frontend/05%20React/How%20React%20Renders%20-%20Reconciliation%20%26%20Fiber). |

### React
| # | Concept | Short answer |
|---|---|---|
| 20 | Lifecycle | Mount → update → unmount. In function components: render, then effects run after commit; cleanup runs before the next effect and on unmount. |
| 21 | Shadow DOM vs Virtual DOM | Shadow DOM: a browser feature for **encapsulated** DOM subtrees (Web Components). Virtual DOM: React's in-memory description of UI used to compute changes. Unrelated ideas. See [Web Components & Shadow DOM](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Components%20%26%20Shadow%20DOM). |
| 22 | Real DOM vs Virtual DOM | The VDOM is not "faster than the DOM"; it lets React **batch and minimise** DOM writes by diffing. Hand-written targeted DOM updates can be faster **(fixed)**. |
| 23 | Element types | Host elements (`div`), text, components (function/class), fragments, portals, context providers, Suspense boundaries. |
| 24 | Error boundaries | Class components with `getDerivedStateFromError`/`componentDidCatch` that catch **render** errors in their subtree (not event handlers or async code). |
| 25 | State vs props | State: owned by the component, changed only through its setter and treated as **immutable** **(fixed: "mutable" is misleading)**. Props: passed in by the parent, read-only. See [State, Props & Data Flow](/Software%20Engineering/Frontend/05%20React/State%2C%20Props%20%26%20Data%20Flow). |
| 26 | Controlled vs uncontrolled | Controlled: the value lives in React state. Uncontrolled: the DOM keeps the value, read via ref or `FormData` (React 19 form Actions favour this). See [Forms & Actions in React](/Software%20Engineering/Frontend/05%20React/Forms%20%26%20Actions%20in%20React). |
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
| 35 | SSR, SSG, **ISR**, CSR | SSR: HTML per request. SSG: HTML at build. ISR (**Incremental Static Regeneration**, **fixed from "ISG"**): static pages regenerated in the background after a revalidate time or on demand. CSR: HTML shell, JS renders in the browser. See [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands). |
| 36 | Caching | Next.js caches fetched data, rendered routes and client router payloads; Next.js 16 made caching explicit with Cache Components and `"use cache"`. See [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching). |
| 37 | SEO | Pre-rendered HTML, the Metadata API (`generateMetadata`), sitemaps, canonical URLs, structured data, good Core Web Vitals. |
| 38 | Dynamic SEO | `generateMetadata` per dynamic route, fed from the same data as the page. |
| 39 | Ranking | Content relevance and quality first; page experience (Core Web Vitals, HTTPS, mobile) is a secondary signal. |
| 40 | Web Vitals | LCP (≤2.5 s), INP (≤200 ms), CLS (≤0.1) at the 75th percentile. See [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals). |
| 41 | Serverless trade-offs | Functions can have cold starts, execution limits and per-invocation cost; Vercel's Fluid compute (2025) reuses instances to reduce cold starts. |

### CSS & HTML
| # | Concept | Short answer |
|---|---|---|
| 42 | Semantic HTML | Elements that describe meaning (`<nav>`, `<main>`, `<article>`, `<button>`): better accessibility, SEO and default behaviour. See [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials). |
| 43 | `defer`, `async`, prefetch | `defer`: download in parallel, run after parsing in order. `async`: run as soon as downloaded, any order. `<link rel="prefetch">`: low-priority fetch for a **future** navigation; `preload`: high-priority fetch for the current page. |
| 44 | HTML parsing | Bytes → tokens → nodes → DOM; `<script>` without `defer/async` blocks parsing; CSS blocks rendering. |
| 45 | Functional (utility) CSS | Single-purpose classes (Tailwind): no naming, small CSS that stops growing, but noisy markup. See [Styling Approaches](/Software%20Engineering/Frontend/04%20CSS/Styling%20Approaches). |
| 46 | CSS performance | Animate `transform` and `opacity` (compositor only, often GPU); avoid animating layout properties; use `will-change` sparingly; `content-visibility: auto` for off-screen sections. |
| 47 | Box model | Content, padding, border, margin; `box-sizing: border-box` makes `width` include padding and border. |
| 48 | CSSOM | The object model built from all CSS; combined with the DOM into the render tree. |
| 49 | Accessibility tree | A **separate** tree the browser derives from the DOM (roles, names, states) and exposes to assistive tech; not simply a subset of the DOM **(fixed)**. |

### Rendering and architecture notes
- **Islands architecture**: small, independent islands of interactivity inside server-rendered static HTML. Each island has its own script and hydrates on its own; the rest of the page stays static (Astro, Fresh). See [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands).
- **Progressive hydration**: hydrate only critical parts on load and the rest later (when visible or idle). React 18+ does **selective hydration** with Suspense boundaries.
- **Observer pattern**: observers subscribe to an observable; when something happens, the observable notifies all of them (DOM events, RxJS, stores). See [Frontend Design Patterns](/Software%20Engineering/Frontend/09%20Architecture/Frontend%20Design%20Patterns).
- **Tree shaking**: the bundler removes exports that are never imported. Needs static ESM and side-effect-free modules. See [Build Tools & Bundlers](/Software%20Engineering/Frontend/14%20Tooling/Build%20Tools%20%26%20Bundlers).
- **Virtualized list / windowing**: render only the rows in (or near) the viewport and recycle them while scrolling, so the DOM stays small for lists with thousands of items. Windowing is the "visible window" technique that virtualization libraries (TanStack Virtual, react-window) implement. See [Virtualization & Large Lists](/Software%20Engineering/Frontend/10%20Performance/Virtualization%20%26%20Large%20Lists).

## Part 2: Question bank

### CSS & HTML
<Accordion title="What is semantic HTML, and why is it important?">
Using elements for their meaning (`<header>`, `<nav>`, `<main>`, `<button>`, `<label>`). Screen readers get roles and landmarks for free, keyboard behaviour comes built in, search engines understand structure, and the code is easier to read.
</Accordion>

<Accordion title="How do `defer`, `async` and prefetch differ?">
`defer` and `async` are script attributes that download without blocking the parser; `defer` runs after parsing, in document order; `async` runs as soon as it arrives. `rel="prefetch"` is a resource hint that fetches something at low priority for a likely **next** navigation. Use `preload` for critical resources of the current page.
</Accordion>

<Accordion title="What tools can analyse the accessibility of a page?">
Lighthouse and the Chrome DevTools accessibility pane (accessibility tree, contrast), axe DevTools / `axe-core` in tests, WAVE, eslint-plugin-jsx-a11y, Playwright's accessibility snapshots, and manual testing with a keyboard and a screen reader (NVDA, VoiceOver). Automated tools catch roughly a third of issues; manual testing is required.
</Accordion>

<Accordion title="How do you push work to the GPU instead of the CPU?">
Animate only `transform` and `opacity` so the browser can do it on the compositor thread, and promote the element to its own layer (`will-change: transform`, only while needed). Too many layers waste memory.
</Accordion>

<Accordion title="What is BEM?">
Block, Element, Modifier: a class naming convention (`card`, `card__title`, `card--featured`) that keeps specificity flat and makes relationships visible. Less needed with CSS Modules, scoped styles or `@scope`.
</Accordion>

<Accordion title="What is the CSSOM and how is it evaluated?">
The browser parses all stylesheets into the CSS Object Model. CSS is render-blocking: the browser needs the full CSSOM before building the render tree (DOM + computed styles), then layout, paint and composite.
</Accordion>

<Accordion title="What is functional CSS and what are its benefits?">
Utility-first CSS: small single-purpose classes composed in markup. Benefits: no naming, no dead CSS, consistent design tokens, CSS size plateaus. Costs: long class lists, need for components to avoid repetition.
</Accordion>

<Accordion title="What is CSS specificity?">
The weight used to pick between conflicting rules: inline styles > IDs > classes/attributes/pseudo-classes > elements/pseudo-elements, compared as a tuple. `!important`, cascade layers (`@layer`) and source order also matter. `:where()` has zero specificity. See [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts).
</Accordion>

### React hooks
<Accordion title="What is `useState` for and how does it work internally?">
It stores state for a component between renders. React keeps a linked list of hooks on the component's fiber; each `useState` call reads its slot by call order. `setState` queues an update and schedules a re-render; updates are batched.
</Accordion>

<Accordion title="How does `useEffect` differ from `componentDidMount`?">
Effects synchronise with external systems after commit and run after every render whose dependencies changed, not only on mount. The cleanup handles unmount and re-runs. Think "sync with X", not lifecycle events.
</Accordion>

<Accordion title="What is `useReducer` and how does it differ from `useState`?">
`useReducer` centralises transitions in a pure reducer, good when next state depends on several fields or many actions. `useState` is simpler for independent values. `useState` is implemented on top of the same mechanism.
</Accordion>

<Accordion title="How does the Context API work with hooks?">
`createContext` + a provider, read with `useContext(Ctx)` (or `use(Ctx)` in React 19, which can be called conditionally). Consumers re-render when the provider value changes identity.
</Accordion>

<Accordion title="Can hooks be called conditionally? Why not?">
No. React identifies each hook by call order on every render; a condition would shift the order and give a hook another hook's state. The exception is `use()`, which can be called inside conditions.
</Accordion>

### React rendering and lifecycle
<Accordion title="How does React apply Virtual DOM changes to the real DOM?">
Render phase: call components, build the new fiber tree, diff against the current one and collect effects (interruptible). Commit phase: apply the DOM mutations synchronously, then run layout effects, then passive effects.
</Accordion>

<Accordion title="What happens during a React render?">
A state update schedules work with a priority lane. React re-runs the affected components, reconciles children, and if something changed, commits DOM updates. Rendering is pure and may be restarted or discarded.
</Accordion>

<Accordion title="Explain the component lifecycle with examples.">
Mount: first render, DOM inserted, effects run (fetch data, subscribe). Update: props/state change → re-render → effects with changed deps re-run after their cleanup. Unmount: cleanups run (unsubscribe, clear timers).
</Accordion>

<Accordion title="What are React's phases when updating?">
Trigger (setState), render (compute and diff), commit (mutate DOM, layout effects), then passive effects (`useEffect`) after paint.
</Accordion>

<Accordion title="How do function components handle lifecycle compared to classes?">
Classes use `componentDidMount/DidUpdate/WillUnmount`. Function components use `useEffect` with dependencies and a cleanup; `useLayoutEffect` for synchronous DOM reads.
</Accordion>

### Types of React components
<Accordion title="Function vs class components?">
Function components are plain functions using hooks; classes use `this.state` and lifecycle methods. All new React features (hooks, Server Components, compiler) target function components. Error boundaries still require a class.
</Accordion>

<Accordion title="What are Pure Components?">
`React.PureComponent` (class) or `React.memo` (function) skip re-rendering when props are shallowly equal. With the React Compiler (1.0, Oct 2025) most manual memoisation becomes unnecessary. See [React Performance](/Software%20Engineering/Frontend/05%20React/React%20Performance).
</Accordion>

<Accordion title="HOCs vs render props?">
Both share logic. A HOC wraps a component and returns a new one; a render prop passes a function child that renders. Custom hooks replaced most uses of both. See [Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns).
</Accordion>

<Accordion title="Controlled vs uncontrolled components?">
Controlled: `value` + `onChange` in state, React is the source of truth (instant validation). Uncontrolled: `defaultValue`, read via ref or `FormData` on submit (less re-rendering).
</Accordion>

<Accordion title="What is `forwardRef` and when is it used?">
It let a function component receive a `ref` and pass it to a DOM node. Since React 19, `ref` is a regular prop for function components, so `forwardRef` is only needed for older React versions.
</Accordion>

### Reconciliation
<Accordion title="What is reconciliation and how does React find the minimal updates?">
It diffs the new tree against the old using two heuristics: elements of different types produce different trees, and `key`s identify children across renders. This makes the diff O(n) instead of O(n³).
</Accordion>

<Accordion title="What is the role of keys?">
Keys give list items a stable identity so React can reorder, insert and remove without destroying state. Use stable ids, not array indexes when the list can reorder.
</Accordion>

<Accordion title="What is React Fiber?">
The reconciler rewrite (React 16) where each element becomes a fiber unit of work. It lets React split rendering into chunks, pause, prioritise (lanes) and resume, which powers concurrent features and Suspense.
</Accordion>

### Rendering strategies
<Accordion title="What is SSR and when would you use it?">
HTML rendered on the server per request: fast first paint and SEO for personalised or frequently changing pages. Cost: server work per request and a hydration step.
</Accordion>

<Accordion title="SSG vs SSR?">
SSG renders at build time and serves from a CDN (fastest, cheapest) but data is as old as the last build. SSR renders on every request (fresh, slower, more server cost).
</Accordion>

<Accordion title="What is ISR and how does it work?">
Static pages with a revalidation rule: after the revalidate time, the next request gets the cached page while Next.js regenerates it in the background (stale-while-revalidate); `revalidatePath`/`revalidateTag` trigger it on demand.
</Accordion>

<Accordion title="How does CSR affect performance and SEO?">
Empty HTML shell until JS downloads and runs: slower LCP on slow devices and weaker SEO for crawlers that don't run JS well. Fine for dashboards behind a login.
</Accordion>

<Accordion title="ISR vs SSG trade-offs?">
ISR avoids full rebuilds for large sites and keeps data fresher, but users may see stale content for one revalidation window and you need a server/runtime that supports it.
</Accordion>

### Hydration and SEO
<Accordion title="What is hydration?">
Attaching React's event handlers and state to server-rendered HTML by rendering the same tree on the client and adopting the existing DOM. Mismatches between server and client output cause warnings or re-rendering.
</Accordion>

<Accordion title="How does hydration differ from rendering?">
Rendering produces the UI (HTML on the server or DOM on the client). Hydration reuses existing HTML and only makes it interactive.
</Accordion>

<Accordion title="What affects SEO in React/Next.js apps?">
Crawlable HTML (SSR/SSG), metadata and canonical tags, clean URLs, sitemaps, structured data, internal links, performance (Core Web Vitals), mobile friendliness, no content hidden behind client-only rendering.
</Accordion>

<Accordion title="How do you optimise Web Vitals in Next.js?">
`next/image` with sizes and priority for the LCP image, `next/font`, streaming with Suspense, fewer client components and less JS, avoiding layout shifts with reserved sizes, and breaking long tasks for INP.
</Accordion>

<Accordion title="Which tools find SEO issues?">
Lighthouse, Google Search Console (coverage, Core Web Vitals report), PageSpeed Insights (CrUX field data), the Rich Results Test, and crawlers like Screaming Frog.
</Accordion>

### Critical rendering path
<Accordion title="What is the critical rendering path?">
The sequence of steps a browser takes to turn HTML, CSS and JS into pixels: build DOM, build CSSOM, run blocking scripts, build the render tree, layout, paint, composite. See [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path).
</Accordion>

<Accordion title="How do you optimise it?">
Fewer critical resources, bytes and round trips: inline critical CSS, `defer` scripts, preload the LCP resource, compress, use HTTP/2+ and a CDN, avoid render-blocking third parties.
</Accordion>

<Accordion title="What role do CSS and JS play?">
CSS blocks rendering (the CSSOM is required). Classic scripts block parsing and wait for preceding CSS, because they may read styles.
</Accordion>

<Accordion title="How does the browser prioritise work?">
The preload scanner fetches resources early; fetch priority depends on type and position (`fetchpriority` can adjust it); the main thread runs tasks, and rendering happens between tasks at frame boundaries.
</Accordion>

<Accordion title="Which tools analyse the critical rendering path?">
Chrome DevTools Performance and Network panels, Lighthouse, WebPageTest waterfalls, and the Performance Insights for LCP phases.
</Accordion>

### JavaScript
<Accordion title="`var`, `let` and `const`?">
See concept 1: scope (function vs block), redeclaration, TDZ, and reassignment of the binding.
</Accordion>

<Accordion title="How does `this` behave in arrow vs regular functions?">
Arrow functions take `this` from the surrounding scope at definition time; regular functions get it from the call site. That's why arrow functions are good for callbacks inside methods and bad as object methods.
</Accordion>

<Accordion title="Explain the event loop and async tasks.">
Sync code runs on the stack; async APIs hand work to the host; when done they queue callbacks. After each macrotask, all microtasks run, then the browser may render. Example: `console.log(1); setTimeout(()=>log(2)); Promise.resolve().then(()=>log(3)); log(4)` prints 1 4 3 2.
</Accordion>

<Accordion title="How does `Promise.all` work and what are the alternatives?">
It takes an iterable and returns a promise of all results in input order, rejecting on the first rejection (the other promises keep running). Use `allSettled` when you need every outcome, `any` for the first success, `race` for timeouts.
</Accordion>

### React advanced
<Accordion title="How do React Profiler and Strict Mode help?">
The Profiler (DevTools or `<Profiler>`) measures what rendered and why. Strict Mode double-invokes renders and effects in development to reveal impure renders and missing cleanups.
</Accordion>

<Accordion title="How do you avoid prop drilling?">
Composition (pass components as children), context for widely shared values, or a store (Zustand, Redux Toolkit) for app state; server state in TanStack Query.
</Accordion>

### Next.js
<Accordion title="How does Next.js do dynamic routing?">
File-system routes with segments like `app/blog/[slug]/page.tsx`, catch-all `[...slug]`, and `generateStaticParams` to pre-render known params.
</Accordion>

<Accordion title="How do you configure caching in Next.js?">
In Next.js 16: enable Cache Components and mark cacheable functions/components with `"use cache"`, control lifetime with `cacheLife` and invalidate with `cacheTag`/`revalidateTag`. Older App Router: `fetch` options (`cache`, `next.revalidate`) and route segment config.
</Accordion>

<Accordion title="Next.js SSR vs &quot;traditional&quot; SSR?">
Traditional SSR renders a full page template per request. Next.js adds React Server Components (zero client JS for server parts), streaming with Suspense, partial pre-rendering, and per-segment caching.
</Accordion>

<Accordion title="How does Next.js improve SEO compared to plain React (CRA/Vite SPA)?">
It sends pre-rendered HTML with metadata, so crawlers and link previews see content immediately, and it improves LCP.
</Accordion>

<Accordion title="Challenges of deploying Next.js on serverless platforms?">
Cold starts, function duration and size limits, cache storage across instances (needs a shared cache for self-hosting), cost per invocation, and vendor-specific features. Self-hosting needs a cache handler for multiple instances.
</Accordion>

### Other
<Accordion title="How do you approach A/B testing with multiple variants?">
Hypothesis and primary metric first, sticky assignment per user, enough sample size per variant (correct for multiple comparisons), server/edge assignment to avoid flicker, log exposure when seen, fixed stopping rule. Full answer in [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions).
</Accordion>

## Related
- [Interviews Index](/Software%20Engineering/Interviews/Interviews%20Index)
- [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions) · [React Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/React%20Interview%20Questions) · [Output Questions & Tricky Snippets](/Software%20Engineering/Frontend/17%20Interview%20Prep/Output%20Questions%20%26%20Tricky%20Snippets) · [Angular Interview Q&A](/Software%20Engineering/Frontend/08%20Angular/Angular%20Interview%20Q%26A)
- [Frontend Home](/Software%20Engineering/Frontend/Frontend%20Home) · [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index)
- [Online Coding Assessment - Bit Count, Consecutive Path & Missing Rolls](/Software%20Engineering/Interviews/Online%20Coding%20Assessment%20-%20Bit%20Count%2C%20Consecutive%20Path%20%26%20Missing%20Rolls)

## Sources
- [MDN Web Docs](https://developer.mozilla.org/)
- [react.dev: Preserving and resetting state](https://react.dev/learn/preserving-and-resetting-state) and [React 19 release post](https://react.dev/blog/2024/12/05/react-19)
- [Next.js docs](https://nextjs.org/docs)
- [web.dev: Critical rendering path](https://web.dev/learn/performance/understanding-the-critical-path)
- Owner's original interview notes (restructured here)
