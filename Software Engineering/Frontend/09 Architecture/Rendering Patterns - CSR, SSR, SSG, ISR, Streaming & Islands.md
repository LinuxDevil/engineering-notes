---
title: "Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands"
tags: [frontend, frontend/architecture, frontend/rendering]
aliases: [Rendering Patterns, SSR vs CSR, CSR vs SSR vs SSG, Hydration, Islands Architecture, Progressive Hydration]
updated: 2026-09-27
---
# Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands

<Note>
**TL;DR**

A rendering pattern answers two questions: **where** HTML is produced (browser, server per request, build time, edge) and **how much JavaScript** the browser must run to make it interactive (full hydration, partial, islands, resumable, or none). CSR is simple but slow to first content; SSR/SSG send real HTML fast but pay a hydration cost; streaming, selective/progressive hydration, islands, resumability, React Server Components and Partial Prerendering are all ways to cut that cost. Modern frameworks let you mix patterns **per route or even per component**.
</Note>

## 1. The big picture

{/* diagram source: /diagrams/diagram-310.mmd */}
![diagram](/diagrams/diagram-310.svg)

| Term | One-liner |
|---|---|
| **CSR** | Empty HTML shell + JS bundle; the browser builds the page |
| **SSR** | Server renders HTML per request, then the client hydrates |
| **SSG / prerendering** | HTML generated at build time, served from a CDN |
| **ISR** | SSG pages regenerated in the background after a time or on demand |
| **Streaming SSR** | Server sends HTML in chunks as data resolves (Suspense boundaries) |
| **Hydration** | Client re-runs components over existing HTML to attach listeners and state |
| **Progressive / selective hydration** | Hydrate parts of the page separately, by priority or on demand |
| **Islands** | Mostly static HTML with small, independently hydrated interactive widgets |
| **Resumability** | Serialize app state into HTML so the client resumes without re-executing components (Qwik) |
| **RSC** | Components that run only on the server and ship zero JS; client components hydrate |
| **PPR** | Static shell prerendered at build, dynamic holes streamed at request time |

## 2. Client-side rendering (CSR)

Since Angular, Ember and Backbone, front-end devs have rendered everything client-side. The server redirects every route to a single `index.html` that has no content (or just a loading screen). The browser downloads, parses and runs the JS, fetches data, then renders. Each route is created dynamically in the browser, with no full page reload on navigation.

![image 12](/Attachments/image%2012.png)

{/* diagram source: /diagrams/diagram-311.mmd */}
![diagram](/diagrams/diagram-311.svg)

- **Pros:** simple hosting (static files), rich app-like navigation, cheap servers, great for apps behind login.
- **Cons:** blank page until JS runs (poor LCP on slow devices), SEO relies on crawlers executing JS (Google does, many social crawlers don't), one `index.html` means one set of OpenGraph metadata for every route, so shared links lose their preview.
- Works well on a good connection; it can be a lot better.

## 3. Server-side rendering (SSR)

### Old-school SSR
Years ago we built pages with e.g. PHP: the server compiled everything, included the data, and delivered a fully populated HTML page. Fast and effective, **but** every navigation repeated the work (full reload, re-download CSS/JS), costing hundreds of ms or seconds.

### Universal / isomorphic apps
What if the **first** load is server-rendered and later navigations are client-side, fetching only the data needed? React popularized this with `renderToString`. The same code runs on server and client. This is called a *universal* or *isomorphic* app (terms used interchangeably).

![Untitled 50](/Attachments/Untitled%2050.png)

![image 13](/Attachments/image%2013.png)

**How SSR works**
1. The server reads the request and determines the content to load.
2. It runs the components (React, Angular...) to generate HTML, fetching user-specific data over its fast, reliable connection (better than a user on [lie-fi](https://developers.google.com/web/fundamentals/performance/poor-connectivity/#what_is_lie-fi)).
3. The server sends the HTML.
4. The browser displays it while downloading the JS bundle.
5. Once JS loads, the app **hydrates** and becomes a fully interactive SPA.

**Why SSR**
1. Faster **first contentful paint / LCP**: the browser can paint HTML without waiting for JS.
2. Better **SEO and social sharing**: content and per-route metadata (OpenGraph) are in the HTML.
3. Better initial load experience: a fully populated page.
4. Access to server resources (DB, secrets, internal APIs) while generating content.

<Note>
**Correction on TTFB**

The original note said SSR "improves Time to First Byte". It's the reverse: rendering on the server usually makes **TTFB slower** than serving a static `index.html`, because the server must fetch data and render first. What SSR improves is **FCP/LCP** (content appears earlier). Streaming and caching (SSG/ISR/PPR) are how you win TTFB back.
</Note>

**Costs:** you must run a server (or serverless/edge functions), handle load and cold starts, and pay for hydration. Browser-only APIs (`window`, `localStorage`) can't run during the server render.

## 4. Hydration

Rendering normally starts from an empty DOM. With **hydration**, the DOM was pre-built by the server; the framework re-runs components on the client, matches their output to the existing DOM and attaches event handlers and state. Rendering components plus attaching handlers to existing HTML is hydration.

{/* diagram source: /diagrams/diagram-312.mmd */}
![diagram](/diagrams/diagram-312.svg)

<Warning>
**Hydration pitfalls**

- **Uncanny valley:** the page looks interactive before it is; clicks during hydration can be lost (Angular's event replay and React's selective hydration mitigate this).
- **Hydration mismatch:** server and client render different output (`Date.now()`, `Math.random()`, `typeof window` checks, locale formatting). Render client-only values in an effect / `afterNextRender`.
- **Double cost:** data fetched on the server must be serialized into the HTML so the client doesn't refetch; you download both HTML and the JS that describes the same UI.
- Heavy hydration hurts **INP** and Total Blocking Time. See [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals).
</Warning>

## 5. Static site generation (SSG) / pre-rendering

The mandatory server for a universal app can be overkill for small apps. **Pre-rendering** compiles pre-selected routes at **build time** into fully populated HTML files served from a static host/CDN, then hydrates. The original note discovered this through the Preact CLI.

- **SSG = "not interactive, just content"** in its purest form (docs, blogs, marketing). Add hydration to make parts interactive.
- **Catch 1:** no user-specific data at build time. The page is generic, so show a well-designed **skeleton** for personalized parts and fetch them on the client.
- **Catch 2:** you need routing that maps each URL to its file (a static host's rewrite rules, a proxy, or a CDN config).
- **Catch 3:** thousands of pages means long builds; content goes stale until the next build. ISR solves this.

The original Q&A framed pre-rendering as a trade-off between CSR and SSR: every pre-rendered page shows a skeleton template while data waits to be rehydrated with AJAX requests, then internal routing is client-side. That describes **SSG + client fetching** for dynamic parts.

## 6. Incremental Static Regeneration (ISR)

ISR (Next.js) = SSG plus a revalidation policy. Pages are served from cache; after the revalidate window, the next request still gets the stale page while a **background regeneration** runs (stale-while-revalidate). You can also revalidate **on demand** from a webhook or Server Action.

{/* diagram source: /diagrams/diagram-313.mmd */}
![diagram](/diagrams/diagram-313.svg)

Trade-off vs SSG: fresh-enough content without full rebuilds, at the price of serving stale pages for one request and needing a server/runtime that supports it. In Next.js 16 caching is opt-in with Cache Components (`"use cache"`, `cacheLife`, `cacheTag`, `revalidateTag`, `updateTag`); Next.js 16.3's instant navigations improve ISR by showing a loading shell first, then prerendering in the background. See [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching).

## 7. Streaming SSR and selective hydration

Classic `renderToString` is all-or-nothing: the slowest data fetch blocks the whole page. **Streaming SSR** (React 18 `renderToPipeableStream` / `renderToReadableStream`) sends the shell immediately and streams each `<Suspense>` boundary's HTML when its data resolves, even out of order, with a tiny inline script that swaps it into place.

```jsx
<Layout>
  <Header />
  <Suspense fallback={<ResultsSkeleton />}>
    <FlightResults query={q} />      {/* slow: streamed later */}
  </Suspense>
  <Suspense fallback={<ReviewsSkeleton />}>
    <Reviews />
  </Suspense>
</Layout>
```

{/* diagram source: /diagrams/diagram-314.mmd */}
![diagram](/diagrams/diagram-314.svg)

**Selective hydration** (React 18): each Suspense boundary hydrates independently; if the user interacts with a not-yet-hydrated boundary, React prioritizes it. React 19.2 also batches Suspense reveals during SSR.

## 8. Progressive, partial hydration and islands

**Progressive hydration** hydrates only certain parts of the app on page load; the rest hydrates progressively as needed (on idle, on visible, on interaction).

**Partial hydration** goes further: parts that are never interactive are never hydrated, so their JS is never shipped.

**Islands architecture** encourages small, focused chunks of interactivity within server-rendered pages. The output is progressively enhanced HTML with explicit rules for how enhancement happens. Instead of one application controlling the full page, there are **multiple entry points**: each island's script is delivered and hydrated independently, and the rest of the page is just static HTML. Astro popularized it.

```astro
---
import Header from '../components/Header.astro';       // static, zero JS
import SearchBox from '../components/SearchBox.jsx';
import PriceChart from '../components/PriceChart.svelte';
---
<Header />
<SearchBox client:load />          <!-- hydrate immediately -->
<PriceChart client:visible />      <!-- hydrate when scrolled into view -->
```

Angular's equivalent is **incremental hydration**: `@defer (hydrate on viewport) { ... }` (stable in v20). See [Modern Angular v17 to v22](/Software%20Engineering/Frontend/08%20Angular/Modern%20Angular%20v17%20to%20v22).

## 9. Resumability (Qwik)

Hydration re-executes components to rebuild state and listeners. **Resumability** serializes state and listener locations into the HTML, so the client **resumes** where the server left off: there's no upfront component execution, only a tiny loader that fetches the handler code for an event when it happens. Startup JS is near zero regardless of app size; the trade-off is fine-grained lazy loading on first interaction, a different programming model (`$` boundaries), and a smaller ecosystem.

## 10. React Server Components (RSC)

From the owner's Intermediate React notes: **SSR** renders the initial page and sends it; then the client does the rest, and the server is no longer involved. **RSC** are components that **only** render on the server. SSR and RSC are different things:

| | SSR | RSC |
|---|---|---|
| Output | HTML for the first load | A serialized component tree (RSC payload) |
| Runs | Every component, on server then again on client (hydration) | Server components only on the server; client components on both |
| JS shipped | All component code | Only client components (`'use client'`) |
| Re-runs on navigation | No (client takes over) | Yes, server components can be re-fetched |

They're **complementary**: frameworks like Next.js App Router render the RSC tree and then SSR it to HTML for the first load. Deep dive: [React Server Components & Server Functions](/Software%20Engineering/Frontend/05%20React/React%20Server%20Components%20%26%20Server%20Functions).

<Warning>
**Security note**

RSC moved code to the server, and with it server attack surface. CVE-2025-55182 (Dec 3, 2025, CVSS 10) was an unauthenticated RCE through Server Function payload deserialization in `react-server-dom-*` 19.0 to 19.2.0; keep React/Next patched. See [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).
</Warning>

## 11. Partial Prerendering (PPR)

**PPR** combines SSG and streaming in one route: a **static shell** is prerendered at build and served instantly from the CDN, while dynamic "holes" (wrapped in Suspense) are rendered per request and streamed into the same response.

{/* diagram source: /diagrams/diagram-315.mmd */}
![diagram](/diagrams/diagram-315.svg)

<Info>
**🆕 2025–2026**

- **React 19.2** (Oct 1, 2025): React DOM partial pre-rendering APIs: `prerender()` returns `{ prelude, postponed }`, then `resume()` or `resumeAndPrerender()`.
- **Next.js 16** (Oct 21, 2025): Cache Components (`cacheComponents: true`, `"use cache"`) complete PPR; the `experimental.ppr` flag was removed/renamed. Everything is dynamic by default and caching is opt-in.
- **React 19.3** (Sept 9, 2026): `browser()` from `react-dom`, used as `use(browser())`, opts a subtree out of SSR.
- **Angular v20** (May 2025): incremental hydration and route-level render mode (`Server`/`Client`/`Prerender` per route) stable.
- **Astro 7** (June 22, 2026): stable route caching; **Astro 6** added live content collections.
</Info>

## 12. Comparison

| | CSR | SSR | Streaming SSR | SSG | ISR | Islands | RSC + PPR |
|---|---|---|---|---|---|---|---|
| TTFB | Fast (static shell) | Slower (render first) | Fast (shell first) | Fastest (CDN) | Fastest (CDN) | Depends (SSG or SSR) | Fast (static shell) |
| FCP / LCP | Slow | Good | Good | Best | Best | Best | Good to best |
| JS to hydrate | All | All | All, but selective | All (if hydrated) | All | Only islands | Only client components |
| SEO | Weak | Good | Good | Good | Good | Good | Good |
| Personalized data | Yes (client) | Yes | Yes | No (client fetch) | No (client fetch) | Yes (in islands / SSR) | Yes (dynamic holes) |
| Freshness | Live | Live | Live | Build time | Revalidate window | Depends | Per component |
| Infra cost | Lowest | Highest | High | Low | Medium | Low to medium | Medium |
| Examples | Admin dashboards, apps behind login | News, search results | E-commerce product pages | Docs, blogs, marketing | Catalogs, listings | Content sites with widgets | Next.js App Router apps |

## 13. When to choose what

{/* diagram source: /diagrams/diagram-316.mmd */}
![diagram](/diagrams/diagram-316.svg)

Rules of thumb:
- **Mix per route.** Marketing pages SSG, search results streaming SSR, account area CSR. Next.js, Angular (`RenderMode` per route), Astro, SvelteKit and Nuxt all support this.
- **Measure field data** (LCP, INP, CLS) before and after; SSR that ships the same JS bundle often improves LCP but worsens INP.
- For a flight-search style product: SSG/ISR landing and destination pages, streaming SSR for results (slow supplier APIs behind Suspense), CSR for the booking funnel behind auth. See [Case Study - Flight Search & Booking](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Flight%20Search%20%26%20Booking).

## Interview questions

<Accordion title="What is the difference between client-side and server-side rendering?">
CSR builds each route in the browser with JS and navigates without full reloads, but shows a blank page (or spinner) until JS runs. SSR sends a fully populated page for any route on the first load, improving FCP/LCP and SEO, and then hydrates so later navigation is client-side.
</Accordion>

<Accordion title="What is a universal (isomorphic) app?">
The same code renders on the server (first load, populated HTML) and on the client (hydration and subsequent navigation). It combines SSR's fast first paint with CSR's app-like navigation.
</Accordion>

<Accordion title="What is hydration, and how does it differ from rendering?">
Rendering creates DOM from scratch. Hydration re-runs components against server-rendered DOM, reusing nodes and attaching event handlers and state. It must produce identical output, otherwise you get mismatch errors.
</Accordion>

<Accordion title="How does SSG differ from SSR, and what does ISR add?">
SSG renders at build time once for everyone (CDN-fast, stale until rebuild); SSR renders per request (fresh, personalized, costs server time). ISR regenerates SSG pages in the background after a revalidate window or on demand, trading one stale response for no full rebuilds.
</Accordion>

<Accordion title="How does CSR impact performance and SEO?">
Larger JS on the critical path, so slower FCP/LCP especially on low-end devices; content invisible to crawlers that don't run JS; one set of meta tags unless you prerender. Fine for authenticated apps where SEO doesn't matter.
</Accordion>

<Accordion title="What is streaming SSR and selective hydration?">
The server flushes the shell immediately and streams each Suspense boundary's HTML as data resolves; each boundary then hydrates independently, prioritizing ones the user interacts with. This fixes all-or-nothing SSR.
</Accordion>

<Accordion title="Islands vs progressive hydration vs resumability?">
Progressive hydration: whole app, hydrated in pieces over time. Islands: only interactive widgets hydrate; static parts never ship JS. Resumability: no hydration at all; state is serialized and handlers load on first interaction (Qwik).
</Accordion>

<Accordion title="Are SSR and React Server Components the same?">
No. SSR produces HTML for the first load and then the client takes over; RSC run only on the server and ship no JS, producing a serialized tree that can be re-fetched. They are complementary: RSC output is typically SSR'd for the first paint.
</Accordion>

<Accordion title="What is Partial Prerendering?">
A static shell prerendered at build time plus dynamic Suspense holes streamed at request time, in one response. In Next.js 16 it is delivered through Cache Components; React 19.2 exposes `prerender`/`resume` primitives.
</Accordion>

## Related
- [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching)
- [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals)
- [React Server Components & Server Functions](/Software%20Engineering/Frontend/05%20React/React%20Server%20Components%20%26%20Server%20Functions)
- [Modern Angular v17 to v22](/Software%20Engineering/Frontend/08%20Angular/Modern%20Angular%20v17%20to%20v22)
- [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path)
- [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals)
- [Other Frameworks - Vue, Svelte, Solid, Astro](/Software%20Engineering/Frontend/09%20Architecture/Other%20Frameworks%20-%20Vue%2C%20Svelte%2C%20Solid%2C%20Astro)
- [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs)
- [Intermediate React V6](/Software%20Engineering/Frontend/Courses%20%26%20Books/FE%20Masters/Intermediate%20React%20V6)

## Sources
- https://blog.logrocket.com/introduction-to-preact-a-smaller-faster-react-alternative-ad5532eb6d79
- https://reactjs.org/docs/react-dom-server.html#rendertostring
- https://developers.google.com/web/fundamentals/performance/poor-connectivity/#what_is_lie-fi
- https://web.dev/articles/rendering-on-the-web
- https://www.patterns.dev/vanilla/islands-architecture
- https://www.patterns.dev/react/progressive-hydration
- https://react.dev/reference/react-dom/server
- https://react.dev/blog/2025/10/01/react-19-2
- https://react.dev/blog/2026/09/09/react-19-3
- https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components
- https://nextjs.org/blog/next-16
- https://nextjs.org/blog/next-16-3
- https://angular.dev/guide/incremental-hydration
- https://docs.astro.build/en/concepts/islands/
- https://qwik.dev/docs/concepts/resumable/
- https://astro.build/blog/astro-7/
