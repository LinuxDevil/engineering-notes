---
title: "Frontend Roadmap & Checklist"
tags: [frontend, index, checklist]
aliases: [FE Roadmap, Frontend Checklist, JS Concepts Checklist]
updated: 2026-09-27
---
# Frontend Roadmap & Checklist

<Note>
**TL;DR**

This is your original "JS Concepts" topic list, now linked to the note that covers each item, plus the gaps that were missing. Tick an item when you can explain it out loud and write its example from memory. The original list is kept in `_Archive/JS Concepts`. It came with this reference: https://github.com/sudheerj/reactjs-interview-questions
</Note>

## JavaScript
- [ ] Variables, `var` / `let` / `const`, strict mode → [Hoisting, TDZ & var-let-const](/Software%20Engineering/Frontend/01%20JavaScript/Hoisting%2C%20TDZ%20%26%20var-let-const)
- [ ] Operators, type coercion, `==` vs `===` → [Types, Coercion & Equality](/Software%20Engineering/Frontend/01%20JavaScript/Types%2C%20Coercion%20%26%20Equality)
- [ ] Destructuring, spread, strings, JSON, loops → [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability), [Modern JavaScript ES2022-ES2027](/Software%20Engineering/Frontend/01%20JavaScript/Modern%20JavaScript%20ES2022-ES2027)
- [ ] Functions, callbacks, HOF, currying, composition → [Functions - HOF, Callbacks & Currying](/Software%20Engineering/Frontend/01%20JavaScript/Functions%20-%20HOF%2C%20Callbacks%20%26%20Currying)
- [ ] Classes, prototype chain, prototypal inheritance, defining properties → [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance)
- [ ] Map & WeakMap, Set & WeakSet, WeakRef → [Collections - Map, Set, WeakMap & WeakRef](/Software%20Engineering/Frontend/01%20JavaScript/Collections%20-%20Map%2C%20Set%2C%20WeakMap%20%26%20WeakRef)
- [ ] Symbol, Proxy & Reflect → [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance), [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability)
- [ ] Atomics & shared memory → [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs)
- [ ] Scope and lexical scope → [Scope & Lexical Environment](/Software%20Engineering/Frontend/01%20JavaScript/Scope%20%26%20Lexical%20Environment)
- [ ] Closures → [Closures & Memory Model](/Software%20Engineering/Frontend/01%20JavaScript/Closures%20%26%20Memory%20Model)
- [ ] Hoisting and TDZ → [Hoisting, TDZ & var-let-const](/Software%20Engineering/Frontend/01%20JavaScript/Hoisting%2C%20TDZ%20%26%20var-let-const)
- [ ] Execution context and call stack → [Execution Context & Call Stack](/Software%20Engineering/Frontend/01%20JavaScript/Execution%20Context%20%26%20Call%20Stack)
- [ ] Context binding (`this`, `call`, `apply`, `bind`) → [this, call, apply & bind](/Software%20Engineering/Frontend/01%20JavaScript/this%2C%20call%2C%20apply%20%26%20bind)
- [ ] Promises and async/await → [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation)
- [ ] Event loop → [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks)
- [ ] Iterators and generators → [Iterators, Generators & Iterator Helpers](/Software%20Engineering/Frontend/01%20JavaScript/Iterators%2C%20Generators%20%26%20Iterator%20Helpers)
- [ ] ES modules and dynamic imports → [Modules (ESM)](/Software%20Engineering/Frontend/01%20JavaScript/Modules%20(ESM))
- [ ] Error handling → [Error Handling](/Software%20Engineering/Frontend/01%20JavaScript/Error%20Handling)
- [ ] Memory management → [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection)
- [ ] `performance` object → [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs), [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals)
- [ ] 🆕 ES2025 / ES2026 features → [Modern JavaScript ES2022-ES2027](/Software%20Engineering/Frontend/01%20JavaScript/Modern%20JavaScript%20ES2022-ES2027)

## TypeScript *(gap: you only had headings)*
- [ ] Types vs interfaces vs abstract classes, enums vs unions → [TypeScript Essentials](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%20Essentials)
- [ ] Generics, utility types (`Omit`, `Pick`, `Required`, your `RequireOnly`) → [Advanced TypeScript - Generics & Utility Types](/Software%20Engineering/Frontend/02%20TypeScript/Advanced%20TypeScript%20-%20Generics%20%26%20Utility%20Types)
- [ ] 🆕 TS 5.8 → 7.0 → [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0)

## React
- [ ] State and props, component communication → [State, Props & Data Flow](/Software%20Engineering/Frontend/05%20React/State%2C%20Props%20%26%20Data%20Flow)
- [ ] Lifecycle and the rules of hooks, custom hooks → [Hooks Reference](/Software%20Engineering/Frontend/05%20React/Hooks%20Reference)
- [ ] `useEffect` and `useState`, `useMemo` and `useCallback` → [Hooks Reference](/Software%20Engineering/Frontend/05%20React/Hooks%20Reference)
- [ ] Controlled and uncontrolled components → [Forms & Actions in React](/Software%20Engineering/Frontend/05%20React/Forms%20%26%20Actions%20in%20React), [Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns)
- [ ] Context → [State Management Overview](/Software%20Engineering/Frontend/06%20State%20Management/State%20Management%20Overview), [Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns)
- [ ] Reconciliation → [How React Renders - Reconciliation & Fiber](/Software%20Engineering/Frontend/05%20React/How%20React%20Renders%20-%20Reconciliation%20%26%20Fiber)
- [ ] HOCs, render props, ref forwarding, portals, error boundaries, Profiler → [Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns)
- [ ] Concurrency → [React Performance](/Software%20Engineering/Frontend/05%20React/React%20Performance), [Hooks Reference](/Software%20Engineering/Frontend/05%20React/Hooks%20Reference)
- [ ] 🆕 Server Components, Actions, React Compiler → [React Server Components & Server Functions](/Software%20Engineering/Frontend/05%20React/React%20Server%20Components%20%26%20Server%20Functions), [React 19 to 19.3 & React Compiler](/Software%20Engineering/Frontend/05%20React/React%2019%20to%2019.3%20%26%20React%20Compiler)

## CSS (and Sass, BEM, SMACSS, vertical rhythm)
- [ ] Specificity, cascading, inheritance, selectors, properties, values, units → [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts)
- [ ] Box model, box sizing, position types, display types → [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts)
- [ ] Block formatting context, containing block, margin collapsing, replaced elements → [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts)
- [ ] Stacking context, visual formatting model → [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts)
- [ ] Vertical and horizontal alignment, Flexbox, Grid, float → [Layout - Flexbox, Grid & Container Queries](/Software%20Engineering/Frontend/04%20CSS/Layout%20-%20Flexbox%2C%20Grid%20%26%20Container%20Queries)
- [ ] Media queries, viewport, container queries → [Layout - Flexbox, Grid & Container Queries](/Software%20Engineering/Frontend/04%20CSS/Layout%20-%20Flexbox%2C%20Grid%20%26%20Container%20Queries)
- [ ] Animations, transitions, transforms, filters → [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts), [Modern CSS 2025-2026](/Software%20Engineering/Frontend/04%20CSS/Modern%20CSS%202025-2026)
- [ ] BEM, SMACSS, Sass, CSS-in-JS → [Styling Approaches](/Software%20Engineering/Frontend/04%20CSS/Styling%20Approaches)
- [ ] 🆕 `:has`, nesting, `@scope`, anchor positioning, View Transitions → [Modern CSS 2025-2026](/Software%20Engineering/Frontend/04%20CSS/Modern%20CSS%202025-2026)

## Web performance
- [ ] Critical rendering path → [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path)
- [ ] Lazy loading, optimizing startup performance → [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook)
- [ ] Navigation and resource timings, Web Performance Timings → [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs), [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals)
- [ ] Performance budgets, RUM vs synthetic monitoring → [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook), [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals)
- [ ] Understanding latency, resource hints, `dns-prefetch` → [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path), [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)

## Web APIs
- [ ] Event phases and delegation, HTML attributes vs DOM properties, DOM and BOM API → [DOM, Events & Event Delegation](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/DOM%2C%20Events%20%26%20Event%20Delegation)
- [ ] Web Components → [Web Components & Shadow DOM](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Components%20%26%20Shadow%20DOM)
- [ ] Web Storage, IndexedDB, Storage API → [Browser Storage - Cookies, Web Storage & IndexedDB](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Browser%20Storage%20-%20Cookies%2C%20Web%20Storage%20%26%20IndexedDB)
- [ ] Service Worker, Web Workers, Background Sync → [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs)
- [ ] Fetch, XHR, WebSocket, Server-Sent Events, Beacon → [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)
- [ ] Intersection, Resize and Mutation Observers, Long Tasks → [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs)
- [ ] History API → Navigation API → [Modern Web Platform APIs 2025-2026](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Modern%20Web%20Platform%20APIs%202025-2026)
- [ ] Geolocation, Notifications, Canvas, WebGL, WebRTC, Fullscreen, PiP, Wake Lock, Broadcast Channel, Channel Messaging, Page Visibility, Credential Management, Web Animations, and the rest → quick-reference table in [Modern Web Platform APIs 2025-2026](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Modern%20Web%20Platform%20APIs%202025-2026)

## Application architecture
- [ ] PWA → [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs)
- [ ] Micro-frontends, Module Federation → [Micro-frontends & Module Federation](/Software%20Engineering/Frontend/09%20Architecture/Micro-frontends%20%26%20Module%20Federation)
- [ ] Monolith vs monorepo → [Monorepos](/Software%20Engineering/Frontend/14%20Tooling/Monorepos)
- [ ] Serverless and edge → [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands), [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching)

## Rendering patterns
- [ ] CSR and routing, SSR, SSG, rehydration, pre-rendering, Server Components → [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands)
- [ ] Streaming, islands, partial pre-rendering → [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands), [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching)

## Component architecture
- [ ] Container, layout, presentational, pure, HOC, component factory → [Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns)

## Code standards
- [ ] SOLID, KISS, DRY → [SOLID](/Software%20Engineering/Principles/SOLID), [DRY, KISS & YAGNI](/Software%20Engineering/Principles/DRY%2C%20KISS%20%26%20YAGNI)

## Design patterns
- [ ] Creational, structural, behavioral patterns (as used in front end) → [Frontend Design Patterns](/Software%20Engineering/Frontend/09%20Architecture/Frontend%20Design%20Patterns), [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns)

## Design systems
- [ ] Tokens, component libraries, Storybook → [Design Systems](/Software%20Engineering/Frontend/09%20Architecture/Design%20Systems)

## Tooling (packaging and bundling)
- [ ] Vite, Webpack, Turbopack (🆕 Rolldown, Rspack) → [Build Tools & Bundlers](/Software%20Engineering/Frontend/14%20Tooling/Build%20Tools%20%26%20Bundlers)
- [ ] Linters, formatters, package managers → [Linting, Formatting, Package Managers & Runtimes](/Software%20Engineering/Frontend/14%20Tooling/Linting%2C%20Formatting%2C%20Package%20Managers%20%26%20Runtimes)

## Gaps you didn't have before (now covered)
- [ ] Accessibility → [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials)
- [ ] Security (XSS, CSRF, CSP) → [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)
- [ ] Auth and token storage → [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage)
- [ ] Supply chain security → [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security)
- [ ] Testing strategy → [Frontend Testing](/Software%20Engineering/Frontend/13%20Testing/Frontend%20Testing)
- [ ] Server state (TanStack Query) → [Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query)
- [ ] Virtualization → [Virtualization & Large Lists](/Software%20Engineering/Frontend/10%20Performance/Virtualization%20%26%20Large%20Lists)
- [ ] AI in front end → [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI)

## System design
- [ ] Framework → [RADIO Framework](/Software%20Engineering/Frontend/15%20System%20Design/RADIO%20Framework)
- [ ] Practice → [Case Study - News Feed](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20News%20Feed), [Case Study - Autocomplete](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Autocomplete), [Case Study - Flight Search & Booking](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Flight%20Search%20%26%20Booking)

## Outside front end (kept from your original list)
- **OOP** (classes, inheritance, polymorphism, abstraction, encapsulation, interfaces, abstract classes, functional interfaces) → [OOP](/Software%20Engineering/Principles/OOP), [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface)
- **Functional programming** (first-class and pure functions, currying, composing, recursion, functors, monads, applicative functors, referential transparency, type and category theory, lambda calculus) → start with [Functions - HOF, Callbacks & Currying](/Software%20Engineering/Frontend/01%20JavaScript/Functions%20-%20HOF%2C%20Callbacks%20%26%20Currying)
- **Data structures and algorithms** (arrays, hashing, linked lists, stacks, queues, heaps, trees, searching, sorting, DP, backtracking, greedy, KMP, Z-algorithm, bit manipulation) → your `DSA` folder
