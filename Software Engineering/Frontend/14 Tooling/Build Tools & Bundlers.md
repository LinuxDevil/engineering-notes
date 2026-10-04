---
title: "Build Tools & Bundlers"
tags: [frontend, frontend/tooling]
aliases: [Bundlers, Build Tools, Vite, Webpack, Tree Shaking, Turbopack, Rspack, Rolldown]
updated: 2026-09-27
---
# Build Tools & Bundlers

<Note>
**TL;DR**

A **bundler** takes your source (TS, JSX, CSS, images, npm packages) and produces optimised files browsers can load fast: it resolves the module graph, transforms code, **tree-shakes** unused exports, **splits** code into chunks, minifies and fingerprints files. In development, modern tools (Vite) serve native ES modules with **HMR** instead of bundling everything. In 2026 the landscape is **Vite 8 (Rolldown + Oxc)**, **Turbopack** (Next.js default), **Rspack 2** (webpack-compatible, Rust), **esbuild**, and legacy **webpack 5**. Create React App is sunset.
</Note>

## What a bundler does

{/* diagram source: /diagrams/diagram-332.mmd */}
![diagram](/diagrams/diagram-332.svg)

Why bundle at all when browsers support ES modules?
- Thousands of small module requests (a deep `node_modules` waterfall) are slow even over HTTP/2.
- Bare imports (`import React from 'react'`) need resolving.
- TS/JSX/CSS modules need transforming; production needs minification, tree shaking and content-hashed filenames for long-term caching.

### Transpiler vs bundler vs minifier
| Job | Tools |
|---|---|
| **Transpile** (TS/JSX/new syntax → browser JS) | Babel, SWC, esbuild, Oxc, `tsc` (also type-checks) |
| **Bundle** (module graph → chunks) | webpack, Rollup, Rolldown, esbuild, Rspack, Turbopack, Parcel |
| **Minify** | Terser, esbuild, SWC, Oxc minifier, Lightning CSS (CSS) |
| **Type-check** | `tsc` / TypeScript 7 native (bundlers usually *strip* types without checking) |

## Dev server vs production build

| | Dev server | Production build |
|---|---|---|
| Goal | Instant startup, fast feedback | Smallest, fastest output |
| Vite approach | Serves **native ESM** on demand; only transforms files the browser requests; pre-bundles dependencies once | Full bundle (Rolldown in Vite 8; Rollup before), tree shaking, splitting, minification |
| webpack approach | Bundles everything in memory, then serves | Same pipeline with optimisations on |
| Source maps | Fast, inline | Separate files (upload to error tracker) |
| HMR | Yes | No |

<Warning>
**Dev/prod parity**

Because dev and build used different pipelines (esbuild + native ESM vs Rollup), Vite apps occasionally behaved differently between the two. Vite 8's single Rolldown-based pipeline is partly about closing this gap. Always test the production build (`vite build && vite preview`).
</Warning>

### HMR (Hot Module Replacement)
When a file changes, the dev server sends the updated module over a WebSocket; the runtime swaps it in **without a full reload**, keeping app state. React Fast Refresh preserves component state across edits; editing a non-component module usually bubbles up to the nearest HMR boundary or causes a full reload.

{/* diagram source: /diagrams/diagram-333.mmd */}
![diagram](/diagrams/diagram-333.svg)

## Tree shaking & dead code elimination

From the owner's Vite note ("Webpack, React Eject, Vite"): when bundling your app, Vite uses
- **Tree shaking** ("live code inclusion") — only includes code that you are actively using.
- **Dead code elimination** — finds code that can never run or whose result is unused, and deletes it.

And from the owner's interview note: tree shaking aims to remove code that will never be used from the final JavaScript bundle. Done right, it reduces bundle size and lowers download, parse and (in some cases) execution time. Your module bundler (webpack, Rollup/Rolldown…) is what you'd expect to remove dead code automatically.

**How they differ**

| | Tree shaking | Dead code elimination (DCE) |
|---|---|---|
| Level | **Module graph**: which *exports* are imported anywhere | **Inside code**: unreachable branches, unused variables, `if (false)` |
| Done by | Bundler | Mostly the minifier (Terser/esbuild/Oxc) |
| Requires | **Static ESM** `import`/`export` | Constant folding (e.g. `process.env.NODE_ENV === 'production'`) |

<Info>
**What tree shaking can and can't remove**

```js
// utils.js
export function formatPrice(n) { return n.toFixed(2); }
export function heavyChart() { /* 50 KB */ }

// app.js
import { formatPrice } from './utils.js';  // ✅ heavyChart is dropped

// ❌ Defeats tree shaking
import * as utils from './utils.js'; utils[name]();   // dynamic property access
const utils = require('./utils');                      // CommonJS is not statically analysable
import _ from 'lodash';                                // CJS package → whole library

// ✅ Tree-shakeable alternatives
import { debounce } from 'lodash-es';
import debounce from 'lodash/debounce';
```
</Info>

### `sideEffects`
A module has **side effects** if importing it does something (polyfills, CSS imports, `window.x = …`, registering custom elements). A bundler can only drop an unused import if it knows the module is side-effect free.

```json
// package.json of a library
{
  "name": "@acme/ui",
  "type": "module",
  "sideEffects": ["**/*.css", "./src/polyfills.js"],
  "exports": { ".": "./dist/index.js" }
}
```
- `"sideEffects": false` = every file is pure → whole unused files can be skipped.
- `/*#__PURE__*/` annotations mark a single call (e.g. `React.forwardRef(...)`, `styled.div(...)`) as removable if its result is unused.
- Barrel files (`index.ts` re-exporting everything) are a common reason tree shaking underperforms, especially in dev and in tests; import from the specific path or rely on `sideEffects: false`.

## Code splitting
Split the bundle so users download only what the current page needs.
- **Entry/route splitting**: each route is a chunk (framework routers do this automatically).
- **Dynamic `import()`**: lazy-load features (`React.lazy`, Angular `loadComponent`, `@defer`).
- **Vendor/shared chunks**: shared deps extracted so they're cached across routes and deploys.
- **Content hashing** (`app.3f9a.js`) + `Cache-Control: immutable` = long-term caching; only changed chunks are re-downloaded.

```js
// Vite ≤ 7 (Rollup) manual chunking example — Vite 8 uses Rolldown, check its chunking options
export default defineConfig({
  build: {
    rollupOptions: {
      output: { manualChunks: { react: ['react', 'react-dom'], charts: ['echarts'] } },
    },
  },
});
```
Too many tiny chunks = request overhead; too few = over-downloading. Let the bundler's defaults do most of the work. See [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook).

Related bundler features: **Module Federation** (webpack 5, Rspack, Vite plugin) shares code between separately deployed apps — see [Micro-frontends & Module Federation](/Software%20Engineering/Frontend/09%20Architecture/Micro-frontends%20%26%20Module%20Federation).

## The tools

### webpack (5)
- The long-time standard: everything is a module, **loaders** transform files, **plugins** hook into the compilation. Enormously flexible, huge ecosystem, Module Federation originated here.
- Downsides: JS-based and slow on large apps, complex config.
- Still everywhere in legacy apps (CRA, older Angular CLI builders, older Next.js).

```js
// webpack.config.js (minimal)
module.exports = {
  entry: './src/index.tsx',
  output: { filename: '[name].[contenthash].js', clean: true },
  module: { rules: [{ test: /\.tsx?$/, use: 'swc-loader', exclude: /node_modules/ }] },
  resolve: { extensions: ['.tsx', '.ts', '.js'] },
  optimization: { splitChunks: { chunks: 'all' } },
};
```

### Vite
- Dev server on native ESM + dependency pre-bundling; framework-agnostic (React, Vue, Svelte, Solid, plus meta-frameworks: SvelteKit, Astro, React Router, TanStack Start, Nuxt).
- Rich plugin API (Rollup-compatible).
- **Vite 7** (2025-06-24): ESM-only, Node 20.19+ / 22.12+.
- **Vite 8** (2026-03-12): **Rolldown replaces both esbuild and Rollup** as the single bundler, with **Oxc** for parsing, transforms and minification; claims 10–30× faster builds; built-in tsconfig paths, Vite DevTools, browser log forwarding. Current: 8.3.1.

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: { port: 5173, proxy: { '/api': 'http://localhost:3000' } },
  build: { sourcemap: true, target: 'es2022' },
});
```

### Rolldown & Oxc (VoidZero)
- **Rolldown**: Rust bundler with a Rollup-compatible API; **1.0 stable on 2026-05-07**.
- **Oxc**: Rust parser/transformer/minifier/resolver, plus Oxlint and Oxfmt.
- Both come from **VoidZero** (Evan You's company), which **joined Cloudflare on 2026-06-04**; projects stay MIT.

### Turbopack
- Rust, incremental bundler built by Vercel for Next.js (function-level caching).
- **Default for `next dev` and `next build` since Next.js 16** (opt out with `--webpack`); filesystem cache for dev stable/on by default in 16.1, build filesystem cache on by default in 16.3.
- Tied to Next.js — not a general-purpose bundler you'd pick for another framework.

### Rspack
- Rust port of webpack with a **webpack-compatible API and loader/plugin ecosystem** (by ByteDance). The migration path for big webpack apps (including Module Federation).
- **Rspack 2.0** (2026-04-22): pure ESM core, `import defer` support, experimental RSC, ~90% smaller install. Now 2.2. **Rsbuild** is its batteries-included layer.

### esbuild
- Go-based, extremely fast bundler/transpiler/minifier by Evan Wallace. Great for libraries, scripts and as a building block (it powered Vite ≤ 7's dependency pre-bundling and transforms, and Angular's `application` builder).
- Limited plugin API and less aggressive chunking/HMR — rarely used alone for big apps.

### Others
- **Rollup**: ESM-first bundler, excellent tree shaking, the classic choice for libraries (being succeeded by Rolldown).
- **Parcel**: zero-config bundler.
- **tsdown / tsup / unbuild**: library bundlers (tsdown is Rolldown-based and part of Vite+).
- **Angular CLI**: `application` builder based on esbuild + Vite dev server; the webpack builder is deprecated in Angular v22.

## Comparison

| Tool | Language | Dev model | Best for | Status (Sept 2026) |
|---|---|---|---|---|
| webpack 5 | JS | Bundle then serve | Legacy apps, maximum plugin flexibility | Maintained; being migrated away from |
| Vite 8 | JS + Rust core (Rolldown/Oxc) | Native ESM + HMR | New SPAs and most meta-frameworks | Current 8.3.x |
| Turbopack | Rust | Incremental bundling | Next.js apps | Default in Next 16 |
| Rspack 2 | Rust | Bundle (fast), webpack API | Migrating large webpack apps | 2.2 |
| esbuild | Go | Bundle (very fast) | Libraries, tooling, scripts | Stable, widely embedded |
| Rolldown 1.0 | Rust | Bundler engine | Under Vite 8; libraries via tsdown | 1.0 stable (2026-05-07) |
| Parcel | JS + Rust | Zero-config | Small projects | Maintained |

## Historical note: Create React App and "eject"
- **CRA** (2016) gave React apps a zero-config webpack + Babel + Jest setup via `react-scripts`.
- `npm run eject` copied the hidden webpack/Babel/Jest config into your project so you could customise it — a **one-way** operation: afterwards you own and maintain all that config and can't take `react-scripts` upgrades. Alternatives were CRACO or `react-app-rewired`.
- CRA fell behind (slow webpack dev server, no SSR, unmaintained deps). React's docs stopped recommending it, and React officially **sunset Create React App on 2025-02-14**, recommending a framework (Next.js, React Router, Expo…) or a build tool like **Vite** or Parcel for SPAs.
- Migration: CRA → Vite is usually straightforward (`index.html` at the root, `REACT_APP_*` → `VITE_*` env vars, Jest → Vitest).

<Warning>
**Common pitfalls**

- Assuming the bundler type-checks — Vite/esbuild/SWC strip types; run `tsc --noEmit` (or TS 7) in CI.
- CommonJS dependencies and `import *` defeating tree shaking.
- Missing `sideEffects` in your own libraries → consumers can't drop unused code (or, if set wrongly to `false`, CSS imports disappear).
- Leaking secrets through `import.meta.env.VITE_*` / `process.env.NEXT_PUBLIC_*` — see [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).
- Only testing the dev server.
</Warning>

<Info>
**🆕 2025–2026**

- **CRA sunset**: 2025-02-14.
- **Vite 7** (2025-06-24) ESM-only; **Vite 8** (2026-03-12) Rolldown + Oxc single pipeline; now 8.3.1.
- **Rolldown 1.0** stable 2026-05-07.
- **Next.js 16** (2025-10-21): Turbopack stable and default for dev and build; 16.3 turns on the build filesystem cache and adds `import.meta.glob` in Turbopack.
- **Rspack 2.0** (2026-04-22): pure ESM core, `import defer`, experimental RSC.
- **Vite+ beta** (2026-07-02, MIT): one `vp` CLI for dev/check/test/build/pack/run bundling Vite 8, Vitest, Rolldown, tsdown, Oxlint and Oxfmt — see [Linting, Formatting, Package Managers & Runtimes](/Software%20Engineering/Frontend/14%20Tooling/Linting%2C%20Formatting%2C%20Package%20Managers%20%26%20Runtimes).
- **Angular v22** deprecated the webpack builder.
- **TypeScript 7.0** (2026-07-08), the native Go port, makes type-checking ~8–12× faster — see [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0).
</Info>

## Interview questions

<Accordion title="What does a bundler do and why do we still need one with native ES modules?">
Resolves the module graph, transforms TS/JSX/CSS, tree-shakes, splits code, minifies and hashes files. Native ESM alone means thousands of requests and no transforms or optimisations, so production still bundles; dev servers like Vite use native ESM for speed.
</Accordion>

<Accordion title="Tree shaking vs dead code elimination?">
Tree shaking is at the module-graph level: unused exports of ES modules are excluded. DCE removes unreachable/unused code inside modules, mostly done by the minifier. Tree shaking needs static `import`/`export` and side-effect information.
</Accordion>

<Accordion title="What is the `sideEffects` field?">
A package.json hint telling bundlers which files have side effects when imported. `false` (or a list of exceptions like CSS files) lets the bundler drop whole modules that aren't used.
</Accordion>

<Accordion title="Why is Vite's dev server fast?">
It doesn't bundle the app in dev: it serves source files as native ESM on demand, transforms only what the browser requests, pre-bundles dependencies once, and HMR updates only the changed module.
</Accordion>

<Accordion title="How does HMR work?">
The dev server watches files; on change it re-transforms the module and notifies the browser over a WebSocket; the client runtime fetches the new module and swaps it at the nearest HMR boundary, keeping app state (React Fast Refresh preserves component state).
</Accordion>

<Accordion title="webpack vs Vite vs Turbopack vs Rspack — which would you pick?">
New SPA or most frameworks: Vite. Next.js: Turbopack (default). Large existing webpack app needing speed with minimal rewrite: Rspack. Library: Rolldown/tsdown or esbuild. Keep webpack only where migration cost is too high.
</Accordion>

<Accordion title="What did &quot;eject&quot; do in Create React App?">
It copied the hidden webpack/Babel/Jest configuration into the project so it could be customised, permanently detaching from `react-scripts` upgrades. CRA itself was sunset in February 2025.
</Accordion>

## Related
- [Linting, Formatting, Package Managers & Runtimes](/Software%20Engineering/Frontend/14%20Tooling/Linting%2C%20Formatting%2C%20Package%20Managers%20%26%20Runtimes)
- [Monorepos](/Software%20Engineering/Frontend/14%20Tooling/Monorepos)
- [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook)
- [Modules (ESM)](/Software%20Engineering/Frontend/01%20JavaScript/Modules%20(ESM))
- [Micro-frontends & Module Federation](/Software%20Engineering/Frontend/09%20Architecture/Micro-frontends%20%26%20Module%20Federation)
- [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3)
- [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0)
- [Frontend Interview Concepts & Question Bank](/Software%20Engineering/Interviews/Frontend%20Interview%20Concepts%20%26%20Question%20Bank)

## Sources
- Owner's notes: `Frontend/Front End.md` ("Webpack, React Eject, Vite" callout), `Interviews/Interview.md` (Tree Shaking), `Frontend/JS Concepts.md` (Packaging & Bundling: Vite, Webpack, Turbopack)
- [web.dev – Reduce JavaScript payloads with tree shaking](https://web.dev/articles/reduce-javascript-payloads-with-tree-shaking)
- [webpack – Tree Shaking](https://webpack.js.org/guides/tree-shaking/)
- [Vite – Why Vite](https://vite.dev/guide/why)
- [Announcing Vite 8](https://vite.dev/blog/announcing-vite8)
- [Announcing Rolldown 1.0](https://voidzero.dev/posts/announcing-rolldown-1-0)
- [Rspack 2.0](https://rspack.rs/blog/announcing-2-0)
- [Next.js 16](https://nextjs.org/blog/next-16), [Next.js 16.3](https://nextjs.org/blog/next-16-3)
- [React – Sunsetting Create React App](https://react.dev/blog/2025/02/14/sunsetting-create-react-app)
- [esbuild](https://esbuild.github.io/)
- [VoidZero joins Cloudflare](https://voidzero.dev/posts/voidzero-cloudflare)
