---
tags: [frontend, index, checklist]
aliases: [FE Roadmap, Frontend Checklist, JS Concepts Checklist]
updated: 2026-09-27
---
# Frontend Roadmap & Checklist

> [!tldr] TL;DR
> This is your original "JS Concepts" topic list, now linked to the note that covers each item, plus the gaps that were missing. Tick an item when you can explain it out loud and write its example from memory. The original list is kept in `_Archive/JS Concepts`. It came with this reference: https://github.com/sudheerj/reactjs-interview-questions

## JavaScript
- [ ] Variables, `var` / `let` / `const`, strict mode → [[Hoisting, TDZ & var-let-const]]
- [ ] Operators, type coercion, `==` vs `===` → [[Types, Coercion & Equality]]
- [ ] Destructuring, spread, strings, JSON, loops → [[Objects, Arrays & Immutability]], [[Modern JavaScript ES2022-ES2027]]
- [ ] Functions, callbacks, HOF, currying, composition → [[Functions - HOF, Callbacks & Currying]]
- [ ] Classes, prototype chain, prototypal inheritance, defining properties → [[Prototypes, Classes & Inheritance]]
- [ ] Map & WeakMap, Set & WeakSet, WeakRef → [[Collections - Map, Set, WeakMap & WeakRef]]
- [ ] Symbol, Proxy & Reflect → [[Prototypes, Classes & Inheritance]], [[Objects, Arrays & Immutability]]
- [ ] Atomics & shared memory → [[Web Workers, Service Workers & PWAs]]
- [ ] Scope and lexical scope → [[Scope & Lexical Environment]]
- [ ] Closures → [[Closures & Memory Model]]
- [ ] Hoisting and TDZ → [[Hoisting, TDZ & var-let-const]]
- [ ] Execution context and call stack → [[Execution Context & Call Stack]]
- [ ] Context binding (`this`, `call`, `apply`, `bind`) → [[this, call, apply & bind]]
- [ ] Promises and async/await → [[Promises, Async-Await & Cancellation]]
- [ ] Event loop → [[Event Loop - Microtasks & Macrotasks]]
- [ ] Iterators and generators → [[Iterators, Generators & Iterator Helpers]]
- [ ] ES modules and dynamic imports → [[Modules (ESM)]]
- [ ] Error handling → [[Error Handling]]
- [ ] Memory management → [[Memory Management & Garbage Collection]]
- [ ] `performance` object → [[Observer APIs]], [[Core Web Vitals]]
- [ ] 🆕 ES2025 / ES2026 features → [[Modern JavaScript ES2022-ES2027]]

## TypeScript *(gap: you only had headings)*
- [ ] Types vs interfaces vs abstract classes, enums vs unions → [[TypeScript Essentials]]
- [ ] Generics, utility types (`Omit`, `Pick`, `Required`, your `RequireOnly`) → [[Advanced TypeScript - Generics & Utility Types]]
- [ ] 🆕 TS 5.8 → 7.0 → [[TypeScript 5.8 to 7.0]]

## React
- [ ] State and props, component communication → [[State, Props & Data Flow]]
- [ ] Lifecycle and the rules of hooks, custom hooks → [[Hooks Reference]]
- [ ] `useEffect` and `useState`, `useMemo` and `useCallback` → [[Hooks Reference]]
- [ ] Controlled and uncontrolled components → [[Forms & Actions in React]], [[Component Patterns]]
- [ ] Context → [[State Management Overview]], [[Component Patterns]]
- [ ] Reconciliation → [[How React Renders - Reconciliation & Fiber]]
- [ ] HOCs, render props, ref forwarding, portals, error boundaries, Profiler → [[Component Patterns]]
- [ ] Concurrency → [[React Performance]], [[Hooks Reference]]
- [ ] 🆕 Server Components, Actions, React Compiler → [[React Server Components & Server Functions]], [[React 19 to 19.3 & React Compiler]]

## CSS (and Sass, BEM, SMACSS, vertical rhythm)
- [ ] Specificity, cascading, inheritance, selectors, properties, values, units → [[CSS Fundamentals - Cascade, Box Model & Contexts]]
- [ ] Box model, box sizing, position types, display types → [[CSS Fundamentals - Cascade, Box Model & Contexts]]
- [ ] Block formatting context, containing block, margin collapsing, replaced elements → [[CSS Fundamentals - Cascade, Box Model & Contexts]]
- [ ] Stacking context, visual formatting model → [[CSS Fundamentals - Cascade, Box Model & Contexts]]
- [ ] Vertical and horizontal alignment, Flexbox, Grid, float → [[Layout - Flexbox, Grid & Container Queries]]
- [ ] Media queries, viewport, container queries → [[Layout - Flexbox, Grid & Container Queries]]
- [ ] Animations, transitions, transforms, filters → [[CSS Fundamentals - Cascade, Box Model & Contexts]], [[Modern CSS 2025-2026]]
- [ ] BEM, SMACSS, Sass, CSS-in-JS → [[Styling Approaches]]
- [ ] 🆕 `:has`, nesting, `@scope`, anchor positioning, View Transitions → [[Modern CSS 2025-2026]]

## Web performance
- [ ] Critical rendering path → [[How Browsers Render - Critical Rendering Path]]
- [ ] Lazy loading, optimizing startup performance → [[Performance Optimization Playbook]]
- [ ] Navigation and resource timings, Web Performance Timings → [[Observer APIs]], [[Core Web Vitals]]
- [ ] Performance budgets, RUM vs synthetic monitoring → [[Performance Optimization Playbook]], [[Core Web Vitals]]
- [ ] Understanding latency, resource hints, `dns-prefetch` → [[How Browsers Render - Critical Rendering Path]], [[Networking - HTTP, Caching & Real-time]]

## Web APIs
- [ ] Event phases and delegation, HTML attributes vs DOM properties, DOM and BOM API → [[DOM, Events & Event Delegation]]
- [ ] Web Components → [[Web Components & Shadow DOM]]
- [ ] Web Storage, IndexedDB, Storage API → [[Browser Storage - Cookies, Web Storage & IndexedDB]]
- [ ] Service Worker, Web Workers, Background Sync → [[Web Workers, Service Workers & PWAs]]
- [ ] Fetch, XHR, WebSocket, Server-Sent Events, Beacon → [[Networking - HTTP, Caching & Real-time]]
- [ ] Intersection, Resize and Mutation Observers, Long Tasks → [[Observer APIs]]
- [ ] History API → Navigation API → [[Modern Web Platform APIs 2025-2026]]
- [ ] Geolocation, Notifications, Canvas, WebGL, WebRTC, Fullscreen, PiP, Wake Lock, Broadcast Channel, Channel Messaging, Page Visibility, Credential Management, Web Animations, and the rest → quick-reference table in [[Modern Web Platform APIs 2025-2026]]

## Application architecture
- [ ] PWA → [[Web Workers, Service Workers & PWAs]]
- [ ] Micro-frontends, Module Federation → [[Micro-frontends & Module Federation]]
- [ ] Monolith vs monorepo → [[Monorepos]]
- [ ] Serverless and edge → [[Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands]], [[Next.js Rendering & Caching]]

## Rendering patterns
- [ ] CSR and routing, SSR, SSG, rehydration, pre-rendering, Server Components → [[Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands]]
- [ ] Streaming, islands, partial pre-rendering → [[Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands]], [[Next.js Rendering & Caching]]

## Component architecture
- [ ] Container, layout, presentational, pure, HOC, component factory → [[Component Patterns]]

## Code standards
- [ ] SOLID, KISS, DRY → [[SOLID]], [[DRY, KISS & YAGNI]]

## Design patterns
- [ ] Creational, structural, behavioral patterns (as used in front end) → [[Frontend Design Patterns]], [[Design Patterns]]

## Design systems
- [ ] Tokens, component libraries, Storybook → [[Design Systems]]

## Tooling (packaging and bundling)
- [ ] Vite, Webpack, Turbopack (🆕 Rolldown, Rspack) → [[Build Tools & Bundlers]]
- [ ] Linters, formatters, package managers → [[Linting, Formatting, Package Managers & Runtimes]]

## Gaps you didn't have before (now covered)
- [ ] Accessibility → [[Accessibility Essentials]]
- [ ] Security (XSS, CSRF, CSP) → [[Frontend Security - XSS, CSRF, CSP & Trusted Types]]
- [ ] Auth and token storage → [[Authentication & Token Storage]]
- [ ] Supply chain security → [[Supply Chain Security]]
- [ ] Testing strategy → [[Frontend Testing]]
- [ ] Server state (TanStack Query) → [[Server State - TanStack Query & RTK Query]]
- [ ] Virtualization → [[Virtualization & Large Lists]]
- [ ] AI in front end → [[AI in Frontend - MCP, WebMCP & Built-in AI]]

## System design
- [ ] Framework → [[RADIO Framework]]
- [ ] Practice → [[Case Study - News Feed]], [[Case Study - Autocomplete]], [[Case Study - Flight Search & Booking]]

## Outside front end (kept from your original list)
- **OOP** (classes, inheritance, polymorphism, abstraction, encapsulation, interfaces, abstract classes, functional interfaces) → [[OOP]], [[Abstract Class and Interface]]
- **Functional programming** (first-class and pure functions, currying, composing, recursion, functors, monads, applicative functors, referential transparency, type and category theory, lambda calculus) → start with [[Functions - HOF, Callbacks & Currying]]
- **Data structures and algorithms** (arrays, hashing, linked lists, stacks, queues, heaps, trees, searching, sorting, DP, backtracking, greedy, KMP, Z-algorithm, bit manipulation) → your `DSA` folder
