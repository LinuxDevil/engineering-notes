---
title: "Frontend System Design Index"
tags: [frontend, frontend/system-design, interview, index]
aliases: [FE System Design, Frontend System Design]
updated: 2026-09-27
---
# Frontend System Design Index

<Note>
**TL;DR**

Front end system design interviews ask you to design a UI product or a reusable component end to end. They test four things (the owner's four pillars): **API communication, UI data management, UI interactions, and assets management**. Use the [RADIO Framework](/Software%20Engineering/Frontend/15%20System%20Design/RADIO%20Framework) to structure the answer, study the curriculum below to have the building blocks ready, and practice on the question bank.
</Note>

## What FE system design interviews test
The FE system design is about:

| Pillar | What the interviewer wants to hear | Main notes |
|---|---|---|
| **API communication** | REST vs GraphQL, pagination, polling / long-polling / SSE / WebSocket, caching, retries, cancellation, error handling | [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time), [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation), [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST) |
| **UI data management** | Where state lives, normalization, server-state caches, URL state, persistence, offline | [State Management Overview](/Software%20Engineering/Frontend/06%20State%20Management/State%20Management%20Overview), [Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query), [URL as State](/Software%20Engineering/Frontend/06%20State%20Management/URL%20as%20State), [Browser Storage - Cookies, Web Storage & IndexedDB](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Browser%20Storage%20-%20Cookies%2C%20Web%20Storage%20%26%20IndexedDB) |
| **User interface interactions** | Component architecture, events, keyboard + a11y, optimistic UI, animations, forms | [Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns), [DOM, Events & Event Delegation](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/DOM%2C%20Events%20%26%20Event%20Delegation), [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials), [Forms & Actions in React](/Software%20Engineering/Frontend/05%20React/Forms%20%26%20Actions%20in%20React) |
| **Assets management** | Bundles, code splitting, images, fonts, CDN, caching headers, rendering strategy | [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook), [Build Tools & Bundlers](/Software%20Engineering/Frontend/14%20Tooling/Build%20Tools%20%26%20Bundlers), [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands) |

Two kinds of questions:
- **Product design**: a page or app (news feed, flight search, chat, e-commerce PDP). Focus on pages, routing, data flow, rendering strategy.
- **Component design**: a reusable widget (autocomplete, image carousel, data table, poll widget). Focus on the props/config API, states, keyboard + ARIA, and extensibility.

{/* diagram source: /diagrams/diagram-345.mmd */}
![diagram](/diagrams/diagram-345.svg)

## Recommended reading order
1. [RADIO Framework](/Software%20Engineering/Frontend/15%20System%20Design/RADIO%20Framework): the structure and time budget.
2. [Case Study - Autocomplete](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Autocomplete): small component problem; practise the flow in 30 minutes.
3. [Case Study - News Feed](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20News%20Feed): the classic product problem (pagination, virtualization, optimistic updates, real-time).
4. [Case Study - Flight Search & Booking](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Flight%20Search%20%26%20Booking): your home turf; use it to tell real stories from 3 years on a flights product.
5. Fill gaps from the curriculum below.

## The curriculum mapped to topic notes
The original plan had six parts. Each item now lives in a topic note:

### Part 1 | Core Fundamentals
| Item | Where it is covered |
|---|---|
| Box Model | [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts) |
| Positioning System | [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts), [Layout - Flexbox, Grid & Container Queries](/Software%20Engineering/Frontend/04%20CSS/Layout%20-%20Flexbox%2C%20Grid%20%26%20Container%20Queries) |
| Formatting Context | [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts) |
| Stacking Context | [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts) |
| Browser Rendering Cycle and reflow | [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path) |
| Composition Layers | [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path) |
| GPU Acceleration | [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path), [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook) |

### Part 2 | DOM API
| Item | Where it is covered |
|---|---|
| API Refresher | [DOM, Events & Event Delegation](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/DOM%2C%20Events%20%26%20Event%20Delegation) |
| Querying methods | [DOM, Events & Event Delegation](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/DOM%2C%20Events%20%26%20Event%20Delegation) |
| Optimizing query performance | [DOM, Events & Event Delegation](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/DOM%2C%20Events%20%26%20Event%20Delegation), [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook) |

### Part 3 | Web APIs & Complex UI Patterns
| Item | Where it is covered |
|---|---|
| Observer API (pattern) | [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs), [Frontend Design Patterns](/Software%20Engineering/Frontend/09%20Architecture/Frontend%20Design%20Patterns) |
| Intersection Observer | [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs) (used for infinite scroll and lazy images in [Case Study - News Feed](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20News%20Feed)) |
| Mutation Observer | [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs) |
| Resize Observer | [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs) |
| Newer platform APIs (Popover, anchor positioning, View Transitions, Navigation API) | [Modern Web Platform APIs 2025-2026](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Modern%20Web%20Platform%20APIs%202025-2026), [Modern CSS 2025-2026](/Software%20Engineering/Frontend/04%20CSS/Modern%20CSS%202025-2026) |

### Part 4 | Virtualization
| Item | Where it is covered |
|---|---|
| Pattern overview (windowing, overscan, variable heights) | [Virtualization & Large Lists](/Software%20Engineering/Frontend/10%20Performance/Virtualization%20%26%20Large%20Lists) |
| Applied | [Case Study - News Feed](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20News%20Feed), [Case Study - Flight Search & Booking](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Flight%20Search%20%26%20Booking) (results list) |

### Part 5 | Application State & Network Connectivity
| Item | Where it is covered |
|---|---|
| Application state: search / access optimization (normalization, indexes, selectors) | [State Management Overview](/Software%20Engineering/Frontend/06%20State%20Management/State%20Management%20Overview), [Redux & Redux Toolkit](/Software%20Engineering/Frontend/06%20State%20Management/Redux%20%26%20Redux%20Toolkit) |
| Browser storage | [Browser Storage - Cookies, Web Storage & IndexedDB](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Browser%20Storage%20-%20Cookies%2C%20Web%20Storage%20%26%20IndexedDB) |
| Memory offloading (evicting off-screen data, workers, IndexedDB) | [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection), [Virtualization & Large Lists](/Software%20Engineering/Frontend/10%20Performance/Virtualization%20%26%20Large%20Lists), [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs) |
| Introduction to browser networking, protocols (HTTP/1.1, 2, 3) | [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) |
| Talking to the server: long-polling, WebSockets, SSE | [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) |
| REST / GraphQL | [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time), [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST), [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST) |

### Part 6 | Web Application Performance
| Item | Where it is covered |
|---|---|
| JS | [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook), [Build Tools & Bundlers](/Software%20Engineering/Frontend/14%20Tooling/Build%20Tools%20%26%20Bundlers), [React Performance](/Software%20Engineering/Frontend/05%20React/React%20Performance) |
| CSS | [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook), [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path) |
| Images | [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook) |
| Other assets (fonts, video, third-party scripts) | [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook) |
| Metrics | [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals) |

## Practice question bank
| # | Question | Type | Key topics | Write-up |
|---|---|---|---|---|
| 1 | Social media news feed (Facebook/Twitter) | Product | Cursor pagination, infinite scroll, virtualization, optimistic likes, real-time | [Case Study - News Feed](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20News%20Feed) |
| 2 | Autocomplete / typeahead | Component | Debounce, cache, race conditions, ARIA combobox | [Case Study - Autocomplete](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Autocomplete) |
| 3 | Flight search & booking | Product | Search form + URL state, polling multi-supplier results, filters, price changes, checkout, 3DS, RTL | [Case Study - Flight Search & Booking](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Flight%20Search%20%26%20Booking) |
| 4 | Chat app (WhatsApp/Messenger) | Product | WebSocket, message ordering, delivery states, offline queue, read receipts | Practice with [RADIO Framework](/Software%20Engineering/Frontend/15%20System%20Design/RADIO%20Framework) + [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) |
| 5 | Image carousel | Component | Props API, preloading next image, swipe + keyboard, `aria-roledescription="carousel"`, reduced motion, CSS scroll snap | Practice |
| 6 | E-commerce product detail page | Product | SSR/SEO, LCP image, variants in URL, add-to-cart optimistic, reviews lazy-loaded | Practice |
| 7 | Google Docs-like editor | Product | `contenteditable`, document model, OT vs CRDT for collaboration, presence, autosave, undo stack | Practice |
| 8 | Video player | Component | Adaptive streaming (HLS/DASH), buffering, captions, keyboard controls, PiP | Practice |
| 9 | Pagination / data table | Component | Server vs client sort/filter, virtualization, column resize, sticky header, `aria-sort` | Practice with [Virtualization & Large Lists](/Software%20Engineering/Frontend/10%20Performance/Virtualization%20%26%20Large%20Lists) |
| 10 | Poll widget | Component | Embeddable widget, vote once (idempotency), live results, iframe vs Web Component | Practice with [Web Components & Shadow DOM](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Components%20%26%20Shadow%20DOM) |

<Tip>
**For questions without a write-up, do a 45-minute timed run with the [[RADIO Framework]] checklist and write a short note of what you missed.**


</Tip>

## Related
- [Frontend Home](/Software%20Engineering/Frontend/Frontend%20Home) · [Frontend Roadmap & Checklist](/Software%20Engineering/Frontend/Frontend%20Roadmap%20%26%20Checklist)
- [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions) · [React Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/React%20Interview%20Questions) · [Output Questions & Tricky Snippets](/Software%20Engineering/Frontend/17%20Interview%20Prep/Output%20Questions%20%26%20Tricky%20Snippets)
- [Frontend Interview Concepts & Question Bank](/Software%20Engineering/Interviews/Frontend%20Interview%20Concepts%20%26%20Question%20Bank) · [Online Coding Assessment - Bit Count, Consecutive Path & Missing Rolls](/Software%20Engineering/Interviews/Online%20Coding%20Assessment%20-%20Bit%20Count%2C%20Consecutive%20Path%20%26%20Missing%20Rolls)

## Sources
- Owner's notes: `Interviews/Front-End System Design/` (System design, Front-End System design, Core Fundamentals, DOM API, Web APIs Complex UI Patterns, Virtualization, Application State & Network Connectivity, Web Application Performance)
- GreatFrontEnd system design playbook: https://www.greatfrontend.com/front-end-system-design-playbook
- web.dev performance: https://web.dev/learn/performance
