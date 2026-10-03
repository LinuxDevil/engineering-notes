---
tags: [frontend, frontend/whats-new, index]
aliases: [What's New 2025, What's New 2026, Frontend 2025-2026, FE Changelog]
updated: 2026-09-27
---
# What's New in Frontend 2025-2026

> [!tldr] TL;DR
> A digest of everything that changed from January 2025 to September 2026, with dates, status and a link to the note that explains each change. Each item was checked against official sources (vendor blogs, TC39, webstatus.dev) in September 2026. The five biggest shifts:
> 1. **React Compiler 1.0** makes manual memoization mostly unnecessary.
> 2. **Next.js 16** makes caching opt-in (`"use cache"`), makes Turbopack the default, and renames middleware to `proxy.ts`.
> 3. **Angular went signals-first**: zoneless, Signal Forms, `OnPush` by default.
> 4. **TypeScript 7** is a native (Go) compiler, about 10× faster.
> 5. **Vite 8** runs on Rolldown and Oxc, and supply-chain attacks changed npm defaults.

> [!info] How to read Baseline labels
> - **Newly available**: works in all core browsers (Chrome, Edge, Firefox, Safari) as of that date.
> - **Widely available**: has worked in all of them for 30+ months.
> - **Limited**: at least one core browser is missing it. Use it as progressive enhancement only.

## Timeline

| When | What | Note |
|---|---|---|
| 2025-02-14 | Create React App sunset | [[React 19 to 19.3 & React Compiler]] |
| 2025-02-28 | TypeScript 5.8 (`--erasableSyntaxOnly`) | [[TypeScript 5.8 to 7.0]] |
| 2025-03-28 | React 19.1 | [[React 19 to 19.3 & React Compiler]] |
| 2025-05-28 | Angular 20 (stable signals APIs, incremental hydration) | [[Modern Angular v17 to v22]] |
| 2025-06 | ES2025 published; Vite 7 (June 24); Biome 2; Oxlint 1.0 | [[Modern JavaScript ES2022-ES2027]], [[Build Tools & Bundlers]] |
| 2025-06-28 | European Accessibility Act applies | [[Accessibility Essentials]] |
| 2025-07-31 | npm trusted publishing (OIDC) GA | [[Supply Chain Security]] |
| 2025-08-01 | TypeScript 5.9 (`import defer`) | [[TypeScript 5.8 to 7.0]] |
| 2025-08-18 | Next.js 15.5 (Node middleware stable, typed routes) | [[Next.js 15 to 16.3]] |
| 2025-09-08 / 09-14 | chalk/debug npm compromise, then the Shai-Hulud worm | [[Supply Chain Security]] |
| 2025-10-01 | React 19.2 (`<Activity>`, `useEffectEvent`, partial pre-rendering) | [[React 19 to 19.3 & React Compiler]] |
| 2025-10-07 | React Compiler 1.0; React Foundation announced | [[React Performance]] |
| 2025-10-14 | Same-document View Transitions become Baseline | [[Modern Web Platform APIs 2025-2026]] |
| 2025-10-21 | **Next.js 16** (Turbopack default, Cache Components, `proxy.ts`) | [[Next.js Rendering & Caching]] |
| 2025-10-22 | Vitest 4 (Browser Mode stable) | [[Frontend Testing]] |
| 2025-10-28 | Node 24 becomes LTS | [[Linting, Formatting, Package Managers & Runtimes]] |
| 2025-11-19 | Angular 21 (zoneless by default, Signal Forms experimental, Vitest default) | [[Modern Angular v17 to v22]] |
| 2025-12-03 | **CVE-2025-55182**: critical RCE in React Server Components | [[React Server Components & Server Functions]] |
| 2025-12-09 | MCP moves to the Agentic AI Foundation (Linux Foundation) | [[AI in Frontend - MCP, WebMCP & Built-in AI]] |
| 2025-12-12 | Invoker commands (`command`/`commandfor`) become Baseline | [[Modern Web Platform APIs 2025-2026]] |
| 2026-01-13 | Navigation API becomes Baseline | [[Modern Web Platform APIs 2025-2026]] |
| 2026-02-12 | Interop 2026 announced | [[Modern CSS 2025-2026]] |
| 2026-02-24 | Trusted Types become Baseline; React Foundation launched | [[Frontend Security - XSS, CSRF, CSP & Trusted Types]] |
| 2026-03-12 | **Vite 8** (Rolldown + Oxc) | [[Build Tools & Bundlers]] |
| 2026-03-23 | TypeScript 6.0 (new defaults, last JS-based release) | [[TypeScript 5.8 to 7.0]] |
| 2026-03-24 | `@scope` becomes Baseline | [[Modern CSS 2025-2026]] |
| 2026-04-22 | Rspack 2.0 | [[Build Tools & Bundlers]] |
| 2026-04-28 | pnpm 11 (`minimumReleaseAge` defaults to 1 day) | [[Supply Chain Security]] |
| 2026-05 | Chrome 148: Prompt API stable on the web | [[AI in Frontend - MCP, WebMCP & Built-in AI]] |
| 2026-05-07 | Rolldown 1.0 | [[Build Tools & Bundlers]] |
| 2026-05-19 | Container style queries become Baseline | [[Layout - Flexbox, Grid & Container Queries]] |
| 2026-06-03 | **Angular 22** (Signal Forms stable, `OnPush` default, `@Service()`) | [[Modern Angular v17 to v22]] |
| 2026-06-09 | WebMCP origin trial (Chrome 149) | [[AI in Frontend - MCP, WebMCP & Built-in AI]] |
| 2026-06 | `:has()` and CSS nesting become *Widely* available; `field-sizing` becomes Baseline | [[Modern CSS 2025-2026]] |
| 2026-06-17 | React Router v8 | [[React Router]] |
| 2026-06-30 | **ES2026** approved | [[Modern JavaScript ES2022-ES2027]] |
| 2026-07-08 | **TypeScript 7.0** (native Go port) | [[TypeScript 5.8 to 7.0]] |
| 2026-07-28 | Chrome 151 ships the Soft Navigations API | [[Core Web Vitals]] |
| 2026-08-03 | Next.js 16.3 (Instant Navigations, TS 7 type checking) | [[Next.js 15 to 16.3]] |
| 2026-09-03 | Vitest 5 | [[Frontend Testing]] |
| 2026-09-09 | **React 19.3** (`<ViewTransition>` and Fragment refs stable) | [[React 19 to 19.3 & React Compiler]] |
| 2026-09-22 | Next.js security fix 16.3.6 (`next/og` RCE) | [[Next.js 15 to 16.3]] |

## By area

### JavaScript and TypeScript
- **ES2025**:
  - Iterator helpers (`.map().filter().take()` on any iterator)
  - New `Set` methods (`union`, `intersection`, …)
  - `Promise.try`
  - JSON modules with import attributes
  - `Float16Array`
- **ES2026**:
  - `RegExp.escape`
  - `Array.fromAsync`
  - `Error.isError`
  - `Math.sumPrecise`
  - `Uint8Array` base64/hex
  - `Map.prototype.getOrInsert`
  - `Iterator.concat`
- **Coming in ES2027** (Stage 4 now):
  - `using` / `await using` (explicit resource management)
  - **Temporal**, which ships in Chrome and Firefox but not Safari yet
- **TypeScript**: 6.0 turns on `strict` by default and removes old options. 7.0 is the native compiler. Its programmatic API arrives in 7.1, so framework template tooling still uses TypeScript 6 for now.
- **Node strips types natively** (stable in 24.12 / 25.2). Only erasable syntax works: no enums, no namespaces.

→ [[Modern JavaScript ES2022-ES2027]], [[TypeScript 5.8 to 7.0]]

### React
- **19.x adds**:
  - Actions and `useActionState`
  - `useOptimistic`
  - `use()`
  - `ref` as a prop
  - `<Activity>`
  - `useEffectEvent`
  - `cacheSignal`
  - partial pre-rendering
  - Performance Tracks in Chrome DevTools
  - `<ViewTransition>` and Fragment refs (stable in 19.3)
- **React Compiler 1.0** does automatic memoization at build time. Its lint rules live in `eslint-plugin-react-hooks`.
- **Server Components security**: the December 2025 advisories are a reminder that RSC endpoints are server attack surface. Patch quickly.

→ [[React 19 to 19.3 & React Compiler]], [[React Server Components & Server Functions]]

### Next.js
- **Next 16 reverses the caching default**:
  - Everything is dynamic unless you opt in with `"use cache"`, `cacheLife` and `cacheTag`.
  - `updateTag` gives read-your-writes updates inside Server Actions.
  - Request APIs (`params`, `cookies()`, …) must be awaited.
  - `middleware.ts` becomes `proxy.ts`.
  - Turbopack is the default everywhere.
- **Next 16.3**:
  - Instant Navigations
  - LTS channels (16.3 Active LTS, 15.5 Maintenance LTS)

→ [[Next.js Rendering & Caching]], [[Next.js 15 to 16.3]]

### Angular
- **v20**: signals APIs stable (`effect`, `linkedSignal`, `toSignal`).
- **v20.2**: zoneless stable.
- **v21**: zoneless by default, and Vitest is the default test runner.
- **v22**:
  - Signal Forms and `resource`/`httpResource` stable
  - `OnPush` becomes the default change detection
  - `@Service()`
  - HttpClient uses fetch by default
  - Angular Aria stable

→ [[Modern Angular v17 to v22]], [[Angular Change Detection, Zones & Signals]]

### CSS and HTML
- **Now Baseline**:
  - `@scope`
  - container style queries
  - `:has()` and nesting (both Widely available)
  - `field-sizing`
  - `sibling-index()`
  - `contrast-color()`
  - `@starting-style`
  - same-document View Transitions
  - invoker commands
  - popover
- **Still Limited, use as enhancement only**:
  - anchor positioning (shipped in all three engines, but Baseline data still says Limited)
  - customizable `<select>` (no Firefox)
  - scroll-driven animations (no Firefox)
  - CSS `if()` and `@function` (Chrome only)
  - cross-document View Transitions

→ [[Modern CSS 2025-2026]], [[Modern Web Platform APIs 2025-2026]]

### Performance
- **Core Web Vitals are unchanged**: LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1.
- **LCP and INP now work in Safari** (Baseline Dec 2025), so field data finally covers all browsers.
- **Soft Navigations API** shipped in Chrome 151. It is supported by web-vitals v6. How CrUX will use SPA navigations hasn't been announced yet.

→ [[Core Web Vitals]], [[Performance Optimization Playbook]]

### Tooling
- **Vite 8**: Rolldown + Oxc.
- **Vite+ beta**: a unified `vp` CLI.
- **VoidZero** joined Cloudflare.
- **Rspack 2**, **Biome 2**, **Oxlint**, **ESLint 10** (flat config only).
- **pnpm 12** is rewritten in Rust.
- **Node** moves to one major release a year, and every release becomes LTS.

→ [[Build Tools & Bundlers]], [[Linting, Formatting, Package Managers & Runtimes]]

### Security
- **npm attacks and responses**:
  - The chalk/debug phishing compromise and the self-replicating Shai-Hulud worm.
  - npm responded with trusted publishing, short-lived tokens, and 2FA-only publishing.
  - pnpm now delays fresh releases by default.
- **Framework CVEs**: RSC (Dec 2025) and `next/og` (Sep 2026).
- **Platform defenses**:
  - Trusted Types are Baseline.
  - `setHTML()` (Sanitizer API) ships in Chrome and Firefox.

→ [[Supply Chain Security]], [[Frontend Security - XSS, CSRF, CSP & Trusted Types]]

### Testing and accessibility
- **Testing tools**:
  - Vitest Browser Mode is stable.
  - Playwright Test Agents (planner, generator, healer) and aria snapshots are available.
  - Angular defaults to Vitest.
- **Rules and standards**:
  - The EAA is enforced (WCAG 2.1 AA via EN 301 549).
  - WCAG 3 is still a Working Draft.
  - `ariaNotify()` is Baseline (Sep 2026).

→ [[Frontend Testing]], [[Accessibility Essentials]]

### AI in frontend
- **MCP** is now a Linux Foundation project.
- **Chrome DevTools for agents** reached 1.0.
- **Next.js and Angular** ship MCP servers and `AGENTS.md`.
- **WebMCP** lets pages expose tools to agents (origin trial).
- **Chrome built-in AI** (Translator, Summarizer, Language Detector, Prompt API) is stable in Chrome.

→ [[AI in Frontend - MCP, WebMCP & Built-in AI]]

### Other frameworks
- **Vue 3.6** (Vapor Mode) is still in RC.
- **SvelteKit 3** and **Solid 2.0** are also RCs.
- **Astro 6 and 7** have shipped.
- **Remix 3** is a separate, non-React framework.

→ [[Other Frameworks - Vue, Svelte, Solid, Astro]]

> [!warning] Still unconfirmed as of Sep 2026
> - The TC39 stage of decorators.
> - Whether `import defer` ships unflagged in any browser.
> - The exact date TanStack Start became stable.
> - Oxfmt's release status.
> - The start date of Shai-Hulud 2.0.
> - How CrUX will use soft-navigation data.
>
> Re-check these before quoting them in an interview.

## Sources
- React blog: https://react.dev/blog
- Next.js blog: https://nextjs.org/blog
- Angular blog: https://blog.angular.dev
- TypeScript blog: https://devblogs.microsoft.com/typescript/
- TC39 finished proposals: https://github.com/tc39/proposals/blob/main/finished-proposals.md
- Baseline data: https://webstatus.dev and https://web-platform-dx.github.io/web-features-explorer/
- Interop 2026: https://web.dev/blog/interop-2026
- Vite 8: https://vite.dev/blog/announcing-vite8
- npm supply chain plan: https://github.blog/security/supply-chain-security/our-plan-for-a-more-secure-npm-supply-chain/
