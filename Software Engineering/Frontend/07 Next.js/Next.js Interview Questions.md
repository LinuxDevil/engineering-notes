---
tags: [frontend, frontend/nextjs, interview]
aliases: [Next.js Interview, Next.js Q&A, Next.js Interview Prep]
updated: 2026-09-27
---
# Next.js Interview Questions

> [!tldr] TL;DR
> 37 Next.js questions ordered from **junior** to **senior**, grouped by topic, written for the **Next.js 16.x / React 19.2+** era (Sept 2026: current release 16.3.6, 16.3 Active LTS). Each answer is short enough to say out loud, with code where it helps, and links to the note that covers the topic in depth. Interviewers in 2026 still ask about the pre-16 caching model, so several answers contrast "before 16" with "Cache Components".

> [!tip] How to use this note
> Level tags: **(J)** junior, **(M)** mid, **(S)** senior. Read the linked note when an answer feels thin. For hands-on practice, rebuild the Acme Shop product page from [[Next.js Performance Optimization]] and write the tests from [[Next.js Testing - Unit, Integration & E2E]].

## 1. Fundamentals and routing

> [!question]- (J) What is Next.js and what does it add on top of React?
> A React framework: file-system routing (App Router), server rendering (static, dynamic, streaming), React Server Components and Server Actions, data caching, image/font/script optimization, bundling (Turbopack), metadata/SEO APIs, and deployment conventions. React is the UI library; Next decides where and when components render. See [[Next.js App Router Fundamentals]].

> [!question]- (J) How does routing work in the App Router?
> Folders are URL segments, `page.tsx` makes a segment routable, `layout.tsx` wraps children and keeps state across navigations. Dynamic segments use `[id]`, catch-all `[...slug]`, optional catch-all `[[...slug]]`; `(group)` folders organize without changing the URL; `_private` folders aren't routable. See [[Next.js App Router Fundamentals]] and [[Next.js Project Structure & Architecture]].

> [!question]- (J) What's the difference between `layout.tsx` and `template.tsx`?
> A layout persists across navigations between its children (state kept, not re-rendered). A template remounts on every navigation, so state resets and effects re-run (useful for enter animations or per-page logging). See [[Next.js App Router Fundamentals]].

> [!question]- (J) What are `loading.tsx`, `error.tsx` and `not-found.tsx`?
> `loading` wraps the page in a Suspense boundary with instant fallback UI; `error` is a Client Component error boundary for the segment (gets `error` and a reset/retry function); `not-found` renders when `notFound()` is called or a URL doesn't match. `global-error` catches root layout errors. Since 16.3, `catchError` builds custom boundaries that don't swallow `redirect()`/`notFound()`. See [[Next.js Error Handling, Loading & Not-Found]].

> [!question]- (M) Why are `params`, `searchParams`, `cookies()` and `headers()` async, and what changed in 16?
> Making them Promises lets Next prerender everything that doesn't depend on the request and only wait where request data is actually read (the basis of PPR/Cache Components). Next 15 introduced async access with a sync fallback; **Next 16 removed synchronous access**.
> ```tsx
> export default async function Page(props: PageProps<"/products/[id]">) {
>   const { id } = await props.params;
>   const currency = (await cookies()).get("currency")?.value ?? "EUR";
> }
> ```
> See [[Next.js 15 to 16.3]].

> [!question]- (M) How do parallel and intercepting routes create a URL-addressable modal?
> A `@modal` slot in the layout renders an intercepting route like `(.)photo/[id]` during client navigation, so the photo opens over the feed. A refresh or shared link renders the real `photo/[id]/page`. The slot's `default.tsx` returns `null` (required in 16); closing uses `router.back()`. See [[Next.js App Router Fundamentals]].

> [!question]- (M) What is `proxy.ts` and why was `middleware.ts` renamed?
> Code that runs before a request completes (redirects, rewrites, headers, coarse auth, i18n negotiation). Next 16 renamed it to `proxy` to stress it's a network boundary in front of the app, and it runs on Node (the Edge runtime stays on deprecated `middleware.ts`). It must not be the only auth check: CVE-2025-29927 (Mar 2025) let attackers skip middleware, and Server Functions are POSTs a matcher can miss. See [[Next.js App Router Fundamentals]], [[Next.js Authentication & Authorization]].

## 2. Server and Client Components

> [!question]- (J) Server Components vs Client Components?
> Server Components (the default) run on the server, can be `async`, read databases and secrets, and ship no JS. Client Components (`'use client'`) are prerendered to HTML and then hydrated to handle state, effects, events and browser APIs; their code and imports ship to the browser. See [[React Server Components & Server Functions]].

> [!question]- (M) What does `'use client'` actually mark, and where should it go?
> It marks a module boundary: that file and everything it imports become client code. Put it on small interactive leaves and pass server-rendered content as `children` (the donut pattern) so heavy libraries stay on the server.
> ```tsx
> // Server page
> <Tabs>{/* client */}
>   <ProductDescription />{/* server, passed as children */}
> </Tabs>
> ```
> See [[Next.js Performance Optimization]].

> [!question]- (M) What can you pass from a Server Component to a Client Component?
> Serializable props (primitives, plain objects, arrays, Dates, Maps/Sets, Promises that the client `use()`s, JSX elements) and Server Actions. Not functions, class instances or anything with methods. Don't pass secrets: props are embedded in the RSC payload in the HTML. Use `server-only` to make server modules fail if imported on the client. See [[Next.js Project Structure & Architecture]].

> [!question]- (S) How would you structure a large Next.js codebase?
> Route folders stay thin (compose features); a data access layer (`lib/dal`) owns fetching, auth checks and DTOs, marked `server-only`; features/modules own UI + actions + schemas; shared UI in a design-system package; colocate tests; enforce boundaries with lint rules (no client imports from server modules, no cross-feature deep imports). In a monorepo, share config and packages via workspaces. See [[Next.js Project Structure & Architecture]], [[Monorepos]].

## 3. Data fetching, mutations and caching

> [!question]- (J) How do you fetch data in the App Router?
> In an `async` Server Component, call your API or DB directly and render. Start independent requests in parallel (`Promise.all`) and wrap slow parts in `<Suspense>` so they stream. Use client-side fetching (TanStack Query/SWR) for interaction-driven data like infinite scroll or polling. See [[Next.js Data Fetching & Mutations Patterns]].

> [!question]- (M) What are Server Actions and how do you use them safely?
> Async functions marked `'use server'` that the client can call (forms, event handlers); Next exposes them as POST endpoints. Validate input (Zod), authenticate and authorize **inside every action**, return typed results for `useActionState`, and invalidate with `updateTag`/`revalidateTag`/`refresh`.
> ```ts
> "use server";
> export async function renameList(id: string, fd: FormData) {
>   const user = await requireUser();                 // auth inside
>   const name = z.string().min(1).parse(fd.get("name"));
>   await lists.rename({ id, ownerId: user.id, name }); // authz via ownerId
>   updateTag(`list-${id}`);
> }
> ```
> See [[Next.js Data Fetching & Mutations Patterns]], [[Next.js Authentication & Authorization]].

> [!question]- (M) Server Action or Route Handler?
> Server Actions for mutations from your own React UI (progressive enhancement, integrated revalidation). Route Handlers for public/third-party APIs, webhooks, non-React clients, streaming (SSE), file downloads, and a BFF over internal services. See [[Next.js Route Handlers & Backend-for-Frontend]].

> [!question]- (M) Explain static vs dynamic rendering, ISR and streaming.
> Static: rendered at build/revalidation, served from cache. Dynamic: rendered per request because it reads request data. ISR: static pages regenerated in the background after a time or on demand (stale-while-revalidate). Streaming: the server sends the shell and Suspense fallbacks first, then each boundary as it resolves. See [[Next.js Rendering & Caching]].

> [!question]- (M) How did caching change from Next 13/14 to 15 to 16?
> 13/14 cached aggressively by default (`fetch`, full route, router cache). 15 stopped caching `fetch`, GET Route Handlers and client page segments by default. 16 introduced **Cache Components**: everything is dynamic unless you opt in with `'use cache'`, `cacheLife` and `cacheTag`; invalidation uses `updateTag` (read-your-writes), `revalidateTag(tag, profile)` (SWR) and `refresh()`. See [[Next.js Rendering & Caching]].

> [!question]- (S) `updateTag` vs `revalidateTag` vs `refresh`?
> `updateTag(tag)` (Server Actions only) expires the tag and the same request reads fresh data: use after a user's own write. `revalidateTag(tag, 'max')` marks stale with stale-while-revalidate: use for webhooks/background changes. `refresh()` re-renders uncached parts of the current page without touching cached entries (e.g. an unread badge). See [[Next.js Rendering & Caching]].

> [!question]- (S) What can go wrong with `'use cache'`?
> Caching per-user data under a shared key (data leak: arguments and closed-over values are the key, so include user/tenant or don't cache); reading `cookies()`/`headers()` inside the cached scope (not allowed; read outside and pass in); non-serializable return values; forgetting invalidation; per-instance caches in multi-pod deployments (needs `cacheHandlers`). See [[Next.js Rendering & Caching]], [[Next.js Self-Hosting & Deployment Options]].

## 4. Authentication and security

> [!question]- (M) Where should authentication checks live in a Next.js app?
> In layers: `proxy.ts` for cheap optimistic redirects (cookie present?), and **authoritative checks next to the data** (DAL functions, Server Actions, Route Handlers) because proxy can be bypassed or not match. Layouts aren't a security boundary either (they don't re-render on every navigation). See [[Next.js Authentication & Authorization]].

> [!question]- (S) What were the December 2025 React/Next.js vulnerabilities and what's the lesson?
> CVE-2025-55182 (tracked by Next as CVE-2025-66478) was a CVSS 10 unauthenticated RCE via Server Function payload deserialization in React's RSC packages; follow-ups fixed DoS and source exposure. Lessons: RSC/Server Action endpoints are public attack surface, keep React/Next patched quickly (Sept 2026 brought another RCE in `ImageResponse`, fixed in 16.3.6), and authorize inside every action. See [[Next.js 15 to 16.3]].

> [!question]- (S) How do you protect a Next.js BFF from CSRF and SSRF?
> Server Actions only accept POST and compare the `Origin` host with `x-forwarded-host`/`host`, rejecting mismatches (add extra hosts with `experimental.serverActions.allowedOrigins` when a proxy doesn't forward the public host; note that requests with no `Origin` header are let through with a warning, so SameSite cookies still matter); Route Handlers that mutate need SameSite cookies plus origin checks or tokens. For SSRF: never fetch user-supplied URLs without an allow-list, keep `images.remotePatterns` strict (16 blocks local IPs in the image optimizer). See [[Next.js Route Handlers & Backend-for-Frontend]], [[Frontend Security - XSS, CSRF, CSP & Trusted Types]].

## 5. Performance

> [!question]- (J) How do `next/image` and `next/font` help performance?
> `next/image` generates responsive `srcset`, converts to AVIF/WebP, lazy-loads by default and reserves space (no CLS); mark the LCP image `preload` (16) or `fetchPriority="high"`. `next/font` self-hosts fonts at build time (no request to Google at runtime) with metric-adjusted fallbacks to avoid layout shift. See [[Next.js Performance Optimization]].

> [!question]- (M) What changed in `next/image` in Next 16?
> `qualities` defaults to `[75]`, `minimumCacheTTL` 60 s → 4 h, `16` removed from `imageSizes`, local IP upstreams blocked, max 3 redirects, query strings on local images need `localPatterns.search`, and `priority` is deprecated in favor of `preload`. See [[Next.js Performance Optimization]].

> [!question]- (M) How do you find and fix a large client bundle in Next 16?
> Run `next experimental-analyze` (Turbopack Bundle Analyzer, 16.1+) to see modules per route and their import chains; fix by moving rendering libraries to Server Components, pushing `'use client'` down, `next/dynamic` for heavy widgets, `optimizePackageImports` for barrel libraries. `@next/bundle-analyzer` is webpack-only. See [[Next.js Performance Optimization]].

> [!question]- (S) A product page has good Lighthouse scores but poor field LCP and INP. How do you investigate?
> Collect RUM with `useReportWebVitals`/`web-vitals` attribution by route pattern and device. For LCP: check TTFB (dynamic render? buffered stream?), whether the hero is preloaded and correctly sized, font swap. For INP: Long Animation Frames show which scripts block (third parties, hydration of big Client Components); reduce client JS, defer scripts, consider the React Compiler. Validate fixes against field p75, not the lab score. See [[Next.js Performance Optimization]], [[Core Web Vitals]].

> [!question]- (S) How does prefetching work in Next 16.x and when would you turn it off?
> `<Link>` prefetches in production when links enter the viewport: static routes fully, dynamic routes down to the nearest `loading.js`. 16.0 added layout deduplication and incremental prefetching; 16.3 bundles small prefetches and adds opt-in Partial Prefetching (one App Shell per route). Turn it off (`prefetch={false}`) or use hover-intent for huge link lists, and never put side effects in layouts because prefetch renders them. See [[Next.js Performance Optimization]].

## 6. Testing

> [!question]- (M) How do you test a Next.js App Router app?
> Vitest + RTL for Client Components and sync components; Server Actions, Route Handlers and `proxy.ts` as plain functions (mock `next/headers`, `next/cache`, auth; MSW for network); Playwright E2E against `next build && next start` for async Server Components, streaming, caching and auth flows. See [[Next.js Testing - Unit, Integration & E2E]].

> [!question]- (M) Can you unit test async Server Components?
> The official docs say Vitest/Jest don't support them and recommend E2E. Keep them thin and unit-test the data layer and the synchronous presentational components they render. See [[Next.js Testing - Unit, Integration & E2E]].

> [!question]- (S) How do you keep an E2E suite fast and reliable in CI?
> Build once and start the production server via Playwright `webServer`; reuse auth with `storageState` from a setup project; shard across CI jobs; seed data through APIs; mock backends the Next server calls with a mock server (not `page.route`); trace on first retry; run screenshot tests in the Playwright Docker image; quarantine flaky tests. Same suite on a schedule becomes synthetic monitoring ([[Playwright Automation as a Cron Job]]). See [[Next.js Testing - Unit, Integration & E2E]].

## 7. i18n, metadata and SEO

> [!question]- (M) How do you add internationalized routing?
> A `[locale]` root segment with `generateStaticParams` for all locales and `<html lang>`, locale negotiation in `proxy.ts` (cookie > `Accept-Language` > default), dictionaries loaded server-side; or next-intl (`defineRouting`, its middleware in `proxy.ts`, `getTranslations`). Next 16.3's `next/root-params` reads the locale anywhere. See [[Next.js Internationalization, Metadata & SEO]].

> [!question]- (M) How does the Metadata API work?
> Export `metadata` or `generateMetadata` from layouts/pages (Server Components). Values merge from root to leaf (`title.template` in the root layout). File conventions add `opengraph-image`, `icon`, `sitemap.ts`, `robots.ts`. Since 15.2 metadata streams for browsers and JS-capable bots but blocks for HTML-limited bots (`htmlLimitedBots`). See [[Next.js Internationalization, Metadata & SEO]].

> [!question]- (S) How do you get hreflang and canonical right for a multi-locale shop?
> Self-referencing canonical per page, reciprocal `alternates.languages` for every locale plus `x-default`, the same URLs in `sitemap.ts` alternates and internal links, no redirects on alternate URLs, never canonicalize translations to the default locale. See [[Next.js Internationalization, Metadata & SEO]].

## 8. Deployment and operations

> [!question]- (J) What are the ways to deploy a Next.js app?
> Vercel (managed), any Node server (`next start`), Docker (`output: 'standalone'`), static export (`output: 'export'`, limited features), or a platform adapter (Netlify, Cloudflare, AWS via OpenNext, Bun). See [[Next.js Self-Hosting & Deployment Options]].

> [!question]- (S) What must you handle when running multiple self-hosted instances?
> Shared `cacheHandler` + `cacheHandlers` (with `refreshTags` for tag coordination) and `cacheMaxMemorySize: 0`; the same `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` for all instances; `deploymentId` for version skew and keeping old static assets; runtime env instead of `NEXT_PUBLIC_*` for per-environment values; a CDN that forwards `rsc` and keys on `_rsc`; no response buffering; graceful SIGTERM drain. See [[Next.js Self-Hosting & Deployment Options]], [[Deploying Next.js Micro-frontends to Kubernetes]].

> [!question]- (S) What is the Deployment Adapters API and why does it matter?
> A public build-time API (alpha in 16.0, stable in 16.2 via `adapterPath`/`NEXT_ADAPTER_PATH`) that lets platforms modify config and consume the full build output. Verified adapters are open source and pass the Next.js compatibility suite (Vercel and Bun as of 16.3.6 docs; Cloudflare and Netlify working toward it; OpenNext rebuilding AWS/Cloudflare/Netlify adapters on it). It reduces reverse-engineering and lock-in. See [[Next.js Self-Hosting & Deployment Options]].

> [!question]- (S) How would you migrate a large app from Next 15 to 16?
> Upgrade Node (20.9+)/TS, run `npx @next/codemod@canary upgrade latest` plus the async request API codemod, rename middleware to proxy, add `default.tsx` to parallel slots, give `revalidateTag` a profile, replace `next lint` and runtime config, check `next/image` defaults and Turbopack builds (or `--webpack` temporarily), then adopt Cache Components route by route behind E2E tests and RUM. See [[Next.js 15 to 16.3]].

## 9. Architecture and judgment

> [!question]- (S) When would you not choose Next.js?
> Pure client apps behind a login where SEO and first load don't matter and a static SPA (Vite + React Router/TanStack Router) is simpler to host; content sites where Astro's islands ship less JS; teams that can't operate the server and caching model and can't use a managed platform; heavy real-time apps whose core is a separate backend anyway. See [[Other Frameworks - Vue, Svelte, Solid, Astro]], [[Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands]].

> [!question]- (S) Design a dashboard in Next.js that controls backend jobs (pause, resume, run now) with live status.
> Server Components render the job list from the backend (short or no caching); Server Actions perform mutations with auth, role checks and idempotency keys, then `refresh()`/`updateTag`; a Route Handler proxies the backend's SSE stream to the browser (BFF, auth header added server-side); a small Client Component subscribes with `EventSource` and patches rows; role-based buttons are hidden in UI **and** enforced server-side. See [[Next.js Route Handlers & Backend-for-Frontend]] and [[Job Scheduler Dashboard with Next.js]].

## Senior perspective

What senior-level interviewers listen for in Next.js answers:
- **Trade-offs, not features**: every rendering/caching choice trades freshness, cost, complexity and latency. Say which you'd pick for a given page and why.
- **Security reflexes**: Server Actions and Route Handlers are public; auth lives next to the data; patch cadence matters (2025–2026 had multiple critical advisories).
- **Operations awareness**: multi-instance caching, skew, CDN behaviour, streaming through proxies, observability by release.
- **Migration pragmatism**: most real codebases mix Pages Router, pre-16 caching and Cache Components; incremental adoption with tests and RUM beats big-bang rewrites.
- **Knowing what's new and what's still experimental**: e.g. Partial Prefetching and the Rust React Compiler are opt-in/experimental in 16.3; the Bundle Analyzer is experimental; Adapters are stable.

> [!info]+ 🆕 2025–2026 topics interviewers now expect
> - **16.0 (2025-10-21)**: Turbopack default, Cache Components (`'use cache'`, `cacheLife`, `cacheTag`, `updateTag`, `refresh`), `proxy.ts`, async-only request APIs, React Compiler support stable, `next/image` defaults, AMP and `next lint` removed.
> - **Dec 2025**: React RSC RCE (CVE-2025-55182 / Next CVE-2025-66478) and follow-up advisories.
> - **16.1 (2025-12-18)**: Bundle Analyzer (experimental), dev filesystem cache stable.
> - **16.2 (2026-03-18)**: Adapters API stable, faster rendering, `catchError`/`retry` experimental.
> - **16.3 (2026-08-03)**: Instant Navigations (opt-in), root params, stable `catchError`/`retry`, `@next/playwright` `instant()`; 16.3 Active LTS.
> - **16.3.6 (2026-09-22)**: `ImageResponse` RCE fix.

## Related
- [[Next.js Index]]
- [[Next.js App Router Fundamentals]]
- [[Next.js Rendering & Caching]]
- [[Next.js 15 to 16.3]]
- [[Next.js Project Structure & Architecture]]
- [[Next.js Data Fetching & Mutations Patterns]]
- [[Next.js Authentication & Authorization]]
- [[Next.js Error Handling, Loading & Not-Found]]
- [[Next.js Route Handlers & Backend-for-Frontend]]
- [[Next.js Performance Optimization]]
- [[Next.js Testing - Unit, Integration & E2E]]
- [[Next.js Self-Hosting & Deployment Options]]
- [[Next.js Internationalization, Metadata & SEO]]
- [[React Interview Questions]]
- [[Frontend Interview Concepts & Question Bank]]
- [[Frontend Home]]

## Sources
- [Next.js docs](https://nextjs.org/docs) (App Router: routing, rendering, caching, Server Actions, metadata, deploying, testing)
- Next.js blog: [15](https://nextjs.org/blog/next-15), [15.2](https://nextjs.org/blog/next-15-2), [16](https://nextjs.org/blog/next-16), [16.1](https://nextjs.org/blog/next-16-1), [16.2](https://nextjs.org/blog/next-16-2), [16.3](https://nextjs.org/blog/next-16-3), [CVE-2025-29927](https://nextjs.org/blog/cve-2025-29927), [Security update Sept 22, 2026](https://nextjs.org/blog/nextjs-security-update-september-22-2026)
- [React: Critical security vulnerability in RSC (2025-12-03)](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)
- [Next.js upgrade guide to 16](https://nextjs.org/docs/app/guides/upgrading/version-16), [Data security guide](https://nextjs.org/docs/app/guides/data-security)
