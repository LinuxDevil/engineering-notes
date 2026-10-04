---
title: "Next.js Index"
tags: [frontend, frontend/nextjs, index]
aliases: [Next.js Home, Next.js MOC, NextJS Index, Next.js Notes]
updated: 2026-09-27
---
# Next.js Index

<Note>
**Start here**

The hub for all Next.js notes, ordered from first App Router page to running Next.js 16 in production. Target versions: **Next.js 16.3.x** (16.3 is Active LTS, 15.5 Maintenance LTS as of Sept 2026) and **React 19.2+**. Each note has worked examples, a senior perspective, a 🆕 2025–2026 box and interview questions. Framework-agnostic theory (rendering patterns, auth, HTTP) lives in other Frontend and APIs notes and is linked, not repeated.
</Note>

{/* diagram source: /diagrams/diagram-288.mmd */}
![diagram](/diagrams/diagram-288.svg)

## 🟢 Beginner: build your first real app

| # | Note | What you'll learn |
|---|---|---|
| 1 | [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals) | File conventions, Server vs Client Components, layouts, dynamic routes, parallel and intercepting routes, async request APIs, `proxy.ts` basics |
| 2 | [Next.js Project Structure & Architecture](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Project%20Structure%20%26%20Architecture) | Folder strategies, route groups, private folders, `server-only`/`client-only`, the Data Access Layer, env validation, monorepos |
| 3 | [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found) | `error.tsx`/`global-error.tsx`, `retry()`, `catchError`, `loading.tsx`, `notFound()`, redirects, `onRequestError` |
| 4 | [Next.js Data Fetching & Mutations Patterns](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Data%20Fetching%20%26%20Mutations%20Patterns) | Parallel fetching, `preload`, `React.cache`, Server Actions with zod, `useActionState`, `useOptimistic`, `updateTag`/`refresh`, a full CRUD example |

## 🟡 Intermediate: correct, secure, discoverable

| # | Note | What you'll learn |
|---|---|---|
| 5 | [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching) | Static vs dynamic, streaming, the pre-16 caching layers, **Cache Components** (`'use cache'`, `cacheLife`, `cacheTag`) |
| 6 | [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization) | Sessions, Better Auth / Auth.js / Clerk / custom OIDC, optimistic proxy checks vs DAL checks, RBAC, CVE-2025-29927, CSRF |
| 7 | [Next.js Route Handlers & Backend-for-Frontend](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Route%20Handlers%20%26%20Backend-for-Frontend) | `route.ts`, handler vs action, SSE proxy, webhooks, BFF token handling, rate limiting, CORS, Node vs Edge |
| 8 | [Next.js Internationalization, Metadata & SEO](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Internationalization%2C%20Metadata%20%26%20SEO) | i18n routing, metadata API, sitemaps, OG images, root params |
| 9 | [Next.js Testing - Unit, Integration & E2E](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Testing%20-%20Unit%2C%20Integration%20%26%20E2E) | Testing Server Components, actions and routes; Playwright, including navigation tests |

## 🔴 Senior: production, scale, upgrades

| # | Note | What you'll learn |
|---|---|---|
| 10 | [Next.js Performance Optimization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Performance%20Optimization) | Core Web Vitals in Next, bundles, images/fonts, prefetching, Instant Navigations |
| 11 | [Next.js Self-Hosting & Deployment Options](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Self-Hosting%20%26%20Deployment%20Options) | Vercel vs self-hosting, `standalone`, Docker, cache handlers, Adapters API, version skew |
| 12 | [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes) | Multi-Zones on K8s with Gateway API, Argo CD, shared cache, `deploymentId` (Acme Shop) |
| 13 | [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) | Release timeline, upgrade checklist, security advisories, LTS scheme |
| 14 | [Next.js Interview Questions](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Interview%20Questions) | Question bank across all of the above |

Applied example: [Job Scheduler Dashboard with Next.js](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Scheduler%20Dashboard%20with%20Next.js) builds the `ops-console` (Server Components for reads, Server Actions for job control, an SSE Route Handler proxy, OIDC with role-gated buttons) on top of [Job Control API - Pause, Resume, Run Now & Stop](/Software%20Engineering/Backend/Scheduled%20Jobs/Job%20Control%20API%20-%20Pause%2C%20Resume%2C%20Run%20Now%20%26%20Stop).

## 📚 Course and original notes
- [Ultimate NextJs Course](/Software%20Engineering/Frontend/Courses%20%26%20Books/FE%20Masters/Ultimate%20NextJs%20Course): course notes (in `Courses & Books/FE Masters`): what a framework is, rendering strategies, data fetching basics.
- [NextJS](/Software%20Engineering/Frontend/_Archive/NextJS) (in `Frontend/_Archive`): the original single Next.js note before the restructure, kept unchanged.
- Background from other areas: [React Server Components & Server Functions](/Software%20Engineering/Frontend/05%20React/React%20Server%20Components%20%26%20Server%20Functions) · [Forms & Actions in React](/Software%20Engineering/Frontend/05%20React/Forms%20%26%20Actions%20in%20React) · [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands) · [Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query) · [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) · [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) · [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals) · [Frontend Testing](/Software%20Engineering/Frontend/13%20Testing/Frontend%20Testing) · [Monorepos](/Software%20Engineering/Frontend/14%20Tooling/Monorepos) · [Micro-frontends & Module Federation](/Software%20Engineering/Frontend/09%20Architecture/Micro-frontends%20%26%20Module%20Federation)

## Suggested study paths

<Info>
**🧭 Beginner to senior (about 3–4 weeks part-time)**

1. [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals) → build a small app with 3 routes, a layout and one form.
2. [Next.js Project Structure & Architecture](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Project%20Structure%20%26%20Architecture) → refactor it into `features/` + a `server-only` DAL.
3. [Next.js Data Fetching & Mutations Patterns](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Data%20Fetching%20%26%20Mutations%20Patterns) → implement the Tasks CRUD example end to end.
4. [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found) → add `loading`, `error`, `not-found` and `onRequestError`.
5. [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching) → turn on `cacheComponents`, add `'use cache'` and tags.
6. [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization) → add OIDC login, sessions and a role-gated action.
7. [Next.js Route Handlers & Backend-for-Frontend](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Route%20Handlers%20%26%20Backend-for-Frontend) → add a webhook and an SSE endpoint.
8. [Next.js Internationalization, Metadata & SEO](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Internationalization%2C%20Metadata%20%26%20SEO) → [Next.js Performance Optimization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Performance%20Optimization) → [Next.js Testing - Unit, Integration & E2E](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Testing%20-%20Unit%2C%20Integration%20%26%20E2E).
9. [Next.js Self-Hosting & Deployment Options](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Self-Hosting%20%26%20Deployment%20Options) → [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes).
10. [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) and [Next.js Interview Questions](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Interview%20Questions) to consolidate.
</Info>

<Info>
**🔐 Security-focused path**

[Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization) → [Next.js Data Fetching & Mutations Patterns](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Data%20Fetching%20%26%20Mutations%20Patterns) (section "Server Actions are public HTTP endpoints") → [Next.js Route Handlers & Backend-for-Frontend](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Route%20Handlers%20%26%20Backend-for-Frontend) (webhooks, CORS, rate limits) → [Next.js Project Structure & Architecture](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Project%20Structure%20%26%20Architecture) (DAL, `server-only`) → security advisories in [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) → [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) → [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security)
</Info>

<Info>
**⚡ Upgrading a Next 14/15 codebase**

[Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) (upgrade checklist) → [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching) (old vs new caching model) → [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found) (`retry`, `catchError`) → [Next.js Route Handlers & Backend-for-Frontend](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Route%20Handlers%20%26%20Backend-for-Frontend) (GET caching change, Edge deprecation) → [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization) (`middleware.ts` → `proxy.ts`, Auth.js status)
</Info>

<Info>
**🎯 One week before an interview**

[Next.js Interview Questions](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Interview%20Questions) daily, plus the "Interview questions" sections of [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals), [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching), [Next.js Data Fetching & Mutations Patterns](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Data%20Fetching%20%26%20Mutations%20Patterns) and [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization). Be ready to whiteboard the RSC request lifecycle, a Server Action round trip, and where auth checks go.
</Info>

## 🆕 2025–2026 in this area

| Date | Change | Note |
|---|---|---|
| 2025-02-26 | 15.2: streaming metadata; Node.js middleware (experimental); `global-error` shown in dev | [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) · [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found) |
| 2025-03-21 | **CVE-2025-29927** middleware auth bypass made public (patched 15.2.3 / 14.2.25 / 13.5.9 / 12.3.5) | [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization) |
| 2025-04-09 | 15.3: `instrumentation-client.ts`, `onNavigate`, `useLinkStatus` | [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found) · [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals) |
| 2025-07-14 | 15.4: experimental `global-not-found.js` | [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found) |
| 2025-08-18 | 15.5: Node.js middleware stable; `typedRoutes` stable; `PageProps`/`LayoutProps`/`RouteContext` helpers; `next lint` deprecated | [Next.js Project Structure & Architecture](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Project%20Structure%20%26%20Architecture) · [Next.js Route Handlers & Backend-for-Frontend](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Route%20Handlers%20%26%20Backend-for-Frontend) |
| 2025-09-22 | **Auth.js joins Better Auth**; Auth.js in maintenance mode | [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization) |
| 2025-10-21 | **16.0**: Turbopack default; Cache Components; `proxy.ts` replaces `middleware.ts` (Node only); `updateTag`/`refresh`; `revalidateTag(tag, profile)`; `next lint` removed | [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) · [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching) · [Next.js Data Fetching & Mutations Patterns](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Data%20Fetching%20%26%20Mutations%20Patterns) |
| 2025-12-03 | Critical RSC Server Function RCE (CVE-2025-55182); follow-ups Dec 11 | [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) · [Next.js Data Fetching & Mutations Patterns](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Data%20Fetching%20%26%20Mutations%20Patterns) |
| 2025-12-18 | 16.1: dev filesystem cache on by default; `next upgrade` | [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) |
| 2026-03-18 | 16.2: Adapters API stable; `AGENTS.md` in `create-next-app`; experimental `unstable_catchError`/`unstable_retry` | [Next.js Self-Hosting & Deployment Options](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Self-Hosting%20%26%20Deployment%20Options) · [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found) |
| 2026-08-03 | **16.3**: Instant Navigations (opt-in), stable `catchError`/`retry`, root params, native Node streams, `@next/playwright` `instant()`, Active LTS | [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) · [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found) · [Next.js Performance Optimization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Performance%20Optimization) · [Next.js Testing - Unit, Integration & E2E](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Testing%20-%20Unit%2C%20Integration%20%26%20E2E) · [Next.js Internationalization, Metadata & SEO](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Internationalization%2C%20Metadata%20%26%20SEO) |
| 2026 (16.3.x docs) | `export const runtime = 'edge'` deprecated; Node.js is the only recommended runtime | [Next.js Route Handlers & Backend-for-Frontend](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Route%20Handlers%20%26%20Backend-for-Frontend) |
| 2026-09-22 | 16.3.6 security fix for the `next/og` `ImageResponse` RCE | [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) |

Still experimental as of Sept 2026: `forbidden()`/`unauthorized()` (`authInterrupts`) and `global-not-found`. Vault-wide changes: [What's New in Frontend 2025-2026](/Software%20Engineering/Frontend/What's%20New%20in%20Frontend%202025-2026).

## Related
- [Frontend Home](/Software%20Engineering/Frontend/Frontend%20Home) · [Software Engineering](/Software%20Engineering/Software%20Engineering)
- [React Index](/Software%20Engineering/Frontend/05%20React/React%20Index) · [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index) · [Frontend Roadmap & Checklist](/Software%20Engineering/Frontend/Frontend%20Roadmap%20%26%20Checklist)
- [Scheduled Jobs Index](/Software%20Engineering/Backend/Scheduled%20Jobs/Scheduled%20Jobs%20Index) (the `ops-console` example)

## Sources
- [Next.js documentation](https://nextjs.org/docs) (v16.3.6)
- [Next.js blog](https://nextjs.org/blog): [16](https://nextjs.org/blog/next-16), [16.1](https://nextjs.org/blog/next-16-1), [16.2](https://nextjs.org/blog/next-16-2), [16.3](https://nextjs.org/blog/next-16-3)
- [Next.js support policy / LTS](https://nextjs.org/support-policy)
- [Vercel: Postmortem on the middleware bypass](https://vercel.com/blog/postmortem-on-next-js-middleware-bypass)
- [Better Auth: Auth.js is now part of Better Auth](https://better-auth.com/blog/authjs-joins-better-auth)
