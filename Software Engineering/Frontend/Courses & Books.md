---
tags: [frontend, index, courses]
aliases: [FE Courses, Frontend Courses, Books]
updated: 2026-09-27
---
# Courses & Books

> [!tldr] TL;DR
> These are your source notes from courses and books, kept as they were and moved into `Courses & Books/`. The topic notes contain the cleaned-up, merged version of this material, so use these notes when you want the course's own flow. This page also keeps the course and article lists from the original "Front End" note.

## Course notes
### Frontend Masters: [[FE Masters]]
- [[Intermediate React V6]]: CSR vs SSR vs SSG, RSC vs SSR → merged into [[Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands]] and [[React Server Components & Server Functions]]
- [[Ultimate NextJs Course]]: rendering strategies and auth → merged into [[Next.js Rendering & Caching]] and [[Authentication & Token Storage]]
- [[Reach Hook Form]]: React Hook Form cheatsheet → merged into [[Forms & Actions in React]]
- Front-End System Design (course) → [[Frontend System Design Index]]
- MCP → [[AI in Frontend - MCP, WebMCP & Built-in AI]]

### JavaScript: The Hard Parts (Will Sentance)
- v2: [[JavaScript the hard parts v2]], with [[Principles]], [[Functions & Callbacks]], [[Closures]] and [[Asynchronous JavaScript]] → merged into [[Execution Context & Call Stack]], [[Functions - HOF, Callbacks & Currying]], [[Closures & Memory Model]] and [[Event Loop - Microtasks & Macrotasks]]
- v3 (2026): [[JS the hard parts v3]] (slides PDF in `Resources`). Its new topics are immutable array methods and promise aborts → [[Objects, Arrays & Immutability]], [[Promises, Async-Await & Cancellation]]

## Book notes
- [[Advance React Book]] (Nadia Makarevich, *Advanced React*), chapters 1–2 → [[How React Renders - Reconciliation & Fiber]], [[React Performance]]
- [[JS Book - Secrets of JS Ninja]] → [[Iterators, Generators & Iterator Helpers]], [[Build Tools & Bundlers]]
- [[Building Micro-frontends with React]] (Packt, Nx + React 18): [[Building Micro frontends with React 18]], [[Chapter 1  Introducing Microfrontends]], [[Chapter 2  Key Principles and Components of Microfrontends]], [[Chapter 3  Monorepo versus Polyrepos for Microfrontends]], [[Chapter 4  Implementing The Multi-SPA Pattern for Microfrontends]] → summarized in [[Micro-frontends & Module Federation]] and [[Monorepos]]

## Courses list (from the original note)
### React
| Priority | Course | Status |
|---|---|---|
| Maybe | [Complete Intro to React v8](https://frontendmasters.com/courses/complete-react-v8/) | ✅ Done |
| Must | [Redux Fundamentals](https://frontendmasters.com/courses/redux-fundamentals/) | ✅ Done |
| Must | [Intermediate React v5](https://frontendmasters.com/courses/intermediate-react-v5/) | ✅ Done |
| Must | [Advanced Redux](https://frontendmasters.com/courses/advanced-redux/) | ✅ Done |
| Must | [React Performance](https://frontendmasters.com/courses/react-performance/) | |
| Must | [The Hard Parts of UI Development](https://frontendmasters.com/courses/hard-parts-ui-dev/) | |
| A big must | [YouTube playlist](https://www.youtube.com/watch?v=OcB3rTln-fI&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3) | |
| | [A Tour of JavaScript & React Patterns](https://frontendmasters.com/courses/tour-js-patterns/) | |

### Node ✅
| Priority | Course | Status |
|---|---|---|
| Maybe | [Introduction to Node.js v2](https://frontendmasters.com/courses/node-js-v2/) | |
| Must | [API Design in Node.js v4](https://frontendmasters.com/courses/api-design-nodejs-v4/) | ✅ Done |
| Must | [Advanced Node for Developers (Udemy)](https://www.udemy.com/course/advanced-node-for-developers/) | |

## Saved articles
**JavaScript**
- [ui.dev: execution contexts, hoisting, scopes and closures](https://ui.dev/ultimate-guide-to-execution-contexts-hoisting-scopes-and-closures-in-javascript)
- [javascript.info: event loop](https://javascript.info/event-loop#event-loop)
- [Unraveling the JavaScript event loop](https://www.linkedin.com/pulse/unraveling-javascript-event-loop-comprehensive-guide-michael-baker/)
- [freeCodeCamp: var, let and const](https://www.freecodecamp.org/news/var-let-and-const-whats-the-difference/)
- [ui.dev JavaScript visualizer](https://ui.dev/javascript-visualizer)

**Browser**
- [DOM vs BOM](https://www.geeksforgeeks.org/what-is-the-difference-between-dom-and-bom/)
- [101 JavaScript critical rendering path](https://indepth.dev/posts/1498/101-javascript-critical-rendering-path)

**React internals**
- [Virtual DOM vs real DOM](https://www.geeksforgeeks.org/difference-between-virtual-dom-and-real-dom/)
- [ReactJS virtual DOM](https://www.geeksforgeeks.org/reactjs-virtual-dom/)
- [How the Virtual DOM is implemented in React](https://indepth.dev/posts/1501/exploring-how-virtual-dom-is-implemented-in-react)
- [LogRocket: virtual DOM in React](https://blog.logrocket.com/virtual-dom-react/)
- [Inside Fiber](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)
- [Change detection in Angular and React](https://indepth.dev/posts/1064/what-every-front-end-developer-should-know-about-change-detection-in-angular-and-react)
- [React Profiler](https://indepth.dev/posts/1497/react-profiler)
- [jser.dev: overall of React internals](https://jser.dev/2023-07-11-overall-of-react-internals/)
- [DOM diffing](https://www.geeksforgeeks.org/explain-dom-diffing/)
- [Diffing algorithm](https://www.geeksforgeeks.org/what-is-diffing-algorithm/)
- [The initial render cycle](https://www.geeksforgeeks.org/explain-the-initial-cycle-in-react-which-happens-during-the-first-render/)
- [thoughtbot: React rendering misconception](https://thoughtbot.com/blog/react-rendering-misconception)

**React hooks**
- [ui.dev: React context](https://ui.dev/react-context)
- [ui.dev: why React hooks](https://ui.dev/why-react-hooks)
- [All React hooks in one short](https://medium.com/@AbidKazmi/all-react-hooks-in-one-short-4b0ed4b5a6e4)
- [React hooks lifecycle diagram](https://wavez.github.io/react-hooks-lifecycle/)
- [Retool: lifecycle methods and hooks](https://retool.com/blog/the-react-lifecycle-methods-and-hooks-explained/)
- [Debounce/throttle a callback with hooks](https://medium.com/programming-essentials/how-to-debounce-throttle-a-callback-with-react-hooks-deb26b2fd7c)

**Other**
- [Prisma introspection](https://medium.com/dooboolab/start-using-prisma-in-your-existing-database-with-prisma-introspection-334a96701bab)

## Related
- [[Frontend Home]] · [[Frontend Roadmap & Checklist]]
