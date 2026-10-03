---
tags: [frontend, frontend/tooling]
aliases: [ESLint, Prettier, Biome, Oxlint, pnpm, npm, Yarn, Bun, Deno, Node.js, Package Managers]
updated: 2026-09-27
---
# Linting, Formatting, Package Managers & Runtimes

> [!tldr] TL;DR
> **Linters** (ESLint, Oxlint, Biome) catch bugs and enforce conventions; **formatters** (Prettier, Biome, Oxfmt) end style debates by rewriting code to one style. **Package managers** (npm, pnpm, Yarn, Bun) install and lock dependencies — pnpm is the common choice for monorepos and security defaults. **Runtimes** (Node.js, Bun, Deno) run your tooling, SSR and tests. 2025–2026 brought Rust-based speed (Oxlint, Biome 2, pnpm 12), ESLint 10's flat-config-only world, Node's new yearly release schedule, and VoidZero's all-in-one **Vite+**.

## Linting vs formatting vs type checking

| | Linter | Formatter | Type checker |
|---|---|---|---|
| Finds | Bugs & bad patterns (unused vars, missing hook deps, `==`, floating promises, a11y issues) | Nothing — just rewrites whitespace, quotes, line breaks | Type errors |
| Changes code? | Sometimes (`--fix`) | Always | No |
| Tools | ESLint, Oxlint, Biome | Prettier, Biome, Oxfmt, dprint | `tsc`, TypeScript 7 native |

Rule: **don't use a linter for formatting** — turn off stylistic lint rules (`eslint-config-prettier`) and let the formatter own style.

```mermaid
flowchart LR
  E["Editor: format on save + lint squiggles"] --> P["Pre-commit: lint-staged runs formatter + linter on staged files"]
  P --> CI["CI: format --check, lint, type-check, test"]
  CI --> M["Merge"]
```

## ESLint (10, flat config)
- The most extensible JS/TS linter; plugins for React (`eslint-plugin-react-hooks`, `react-x`), TypeScript (`typescript-eslint`, including type-aware rules), a11y (`jsx-a11y`), imports, Angular (`angular-eslint`), testing.
- **Flat config** (`eslint.config.js`) replaced `.eslintrc`: a plain array of config objects, explicit imports of plugins, no magic string resolution or cascading.
- **ESLint 10** (2026-02-06) is **flat config only** — `.eslintrc*` is gone.

> [!example] `eslint.config.js` for a React + TypeScript app
> ```js
> import js from '@eslint/js';
> import tseslint from 'typescript-eslint';
> import reactHooks from 'eslint-plugin-react-hooks';
> import jsxA11y from 'eslint-plugin-jsx-a11y';
> import prettier from 'eslint-config-prettier';
> import { defineConfig, globalIgnores } from 'eslint/config';
>
> export default defineConfig([
>   globalIgnores(['dist', 'coverage']),
>   js.configs.recommended,
>   tseslint.configs.recommendedTypeChecked,
>   {
>     languageOptions: {
>       parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
>     },
>   },
>   reactHooks.configs.flat.recommended,   // incl. React Compiler rules
>   jsxA11y.flatConfigs.recommended,
>   {
>     files: ['**/*.{ts,tsx}'],
>     rules: {
>       '@typescript-eslint/no-floating-promises': 'error',
>       'no-console': ['warn', { allow: ['warn', 'error'] }],
>     },
>   },
>   prettier, // last: turns off rules that conflict with Prettier
> ]);
> ```
> Plugin export names vary by version — check each plugin's README for its flat-config export.

- `eslint-plugin-react-hooks` v6 uses flat config and (with React Compiler 1.0) ships compiler-powered rules like `set-state-in-render`, `set-state-in-effect`, `refs`.
- Next.js 16 removed `next lint`; run ESLint (or Biome) directly.

## Biome (2)
- Rust **linter + formatter in one binary**, Prettier-compatible formatting, one `biome.json`, very fast.
- **Biome v2** (2025-06-17): **type-aware lint rules without `tsc`**, GritQL plugins, multi-file analysis, and `"extends": "//"` to inherit a root config in monorepos. **v2.5** (2026-06-12) has ~500 rules.
- Trade-off: smaller plugin ecosystem than ESLint (no arbitrary JS plugins).

```json
// biome.json
{
  "formatter": { "indentStyle": "space", "lineWidth": 100 },
  "linter": { "rules": { "recommended": true } },
  "javascript": { "formatter": { "quoteStyle": "single" } }
}
```
```bash
npx @biomejs/biome check --write .   # format + lint + organise imports
```

## Oxlint (and Oxfmt)
- **Oxlint**: Rust linter from the Oxc project (VoidZero), 50–100× faster than ESLint in their benchmarks, ESLint-compatible rule names covering core, TypeScript, React, jsx-a11y, import, unicorn, etc. **1.0 on 2025-06-10**, now at 1.85.
- Common setup: run **Oxlint first** for speed, keep **ESLint** only for rules/plugins Oxlint doesn't cover (`eslint-plugin-oxlint` disables the overlapping ESLint rules).
- **Oxfmt**: Prettier-compatible formatter from Oxc, **in alpha** (whether it has reached beta/1.0 is unverified).

## Prettier
- The opinionated formatter: parse → reprint. Few options by design (`printWidth`, `singleQuote`, `semi`, `trailingComma`…).
- Supports JS/TS/JSX, CSS/SCSS, HTML, Markdown, JSON, YAML, GraphQL, and more via plugins (e.g. `prettier-plugin-tailwindcss` for class sorting).
- Still the default formatter in most teams; Biome and Oxfmt target compatibility with its output.

```json
// .prettierrc
{ "singleQuote": true, "trailingComma": "all", "printWidth": 100 }
```

### Choosing
| Situation | Pick |
|---|---|
| Need specific ESLint plugins (Angular, custom company rules, type-aware typescript-eslint) | ESLint (+ Oxlint in front for speed) + Prettier |
| Want one fast tool, few plugins | Biome |
| Huge monorepo where ESLint is slow | Oxlint + slimmed ESLint |
| All-in on VoidZero stack | Vite+ (Oxlint + Oxfmt) |

## Package managers

| | npm | pnpm | Yarn (Berry, v2+) | Bun |
|---|---|---|---|---|
| Ships with | Node | Separate install | Separate | Bun runtime |
| `node_modules` layout | Flat (hoisted) | **Content-addressable store + symlinks** (strict, no phantom deps) | Plug'n'Play (no `node_modules`) or `node-modules` linker | Flat, very fast installs |
| Disk usage | Duplicated per project | Hard-linked from one global store | Zip cache | Global cache |
| Workspaces | Yes | Yes (best-in-class filtering: `--filter`) | Yes | Yes |
| Lockfile | `package-lock.json` | `pnpm-lock.yaml` | `yarn.lock` | `bun.lock` |
| Dependency scripts | Run by default | **Off by default since v10** (allow-list) | Configurable | Off unless `trustedDependencies` |

**Phantom dependencies**: with npm's hoisting, your code can `import` a package you never declared (it's a dependency of a dependency). It works until that transitive dep changes. pnpm's strict layout makes this an error.

```bash
# Equivalent commands
npm install            | pnpm install            | yarn            | bun install
npm install zod        | pnpm add zod            | yarn add zod    | bun add zod
npm install -D vitest  | pnpm add -D vitest      | yarn add -D ... | bun add -d vitest
npm ci                 | pnpm i --frozen-lockfile| yarn --immutable| bun install --frozen-lockfile
npm run build          | pnpm build              | yarn build      | bun run build
npx vite               | pnpm dlx vite           | yarn dlx vite   | bunx vite
```

pnpm timeline:
- **pnpm 10** (Jan 2025): dependency lifecycle scripts not run by default.
- **pnpm 11** (2026-04-28): **`minimumReleaseAge` defaults to 1 day**, `blockExoticSubdeps` on by default, SQLite store, native publish, requires Node 22+.
- **pnpm 12** (2026-08-26): **rewritten in Rust**, compatible with 11.

Security implications (cooldowns, install scripts, lockfiles): [[Supply Chain Security]]. Workspaces: [[Monorepos]].

## Runtimes

### Node.js
- The default runtime for tooling, SSR and servers.
- **Release lines**: **Node 24** became LTS ("Krypton") on 2025-10-28; **Node 26.0** (2026-05-05) has **Temporal on by default**, V8 14.6 (Map upsert, `Iterator.concat`), removed `--experimental-transform-types`, and becomes LTS in Oct 2026.
- **New release schedule** (announced 2026-03-10): **one major per year** starting with **Node 27** (alpha Oct 2026, current Apr 2027), and **every release becomes LTS** — no more odd/even split.
- **Native TypeScript type stripping**: on by default since v23.6 / v22.18, **Stable** in v25.2 / v24.12. Only erasable syntax works (no enums, runtime namespaces, parameter properties, `import =`) — pair with TS `erasableSyntaxOnly`.
- `require(esm)` works unflagged (Node 22.12+ / 20.19+).

```bash
node --watch src/server.ts   # runs TS directly (types stripped), restarts on change
```

Tip: pin the Node version per repo (`.nvmrc`, `engines`, `packageManager` field, or Volta/mise/fnm) so CI and developers match.

### Bun
- All-in-one runtime written in Zig on **JavaScriptCore**: runtime + package manager + bundler + test runner, Node-compatible APIs, native TS/JSX, very fast startup and installs.
- Versions: 1.2 (Jan 2025), 1.3 (Oct 2025), **1.4 (2026-08-20)**.
- Popular as a fast package manager/test runner even in Node projects; check compatibility for production servers.

### Deno
- Runtime by Node's creator on V8 + Rust: **secure by default** (explicit `--allow-net`, `--allow-read`…), native TS, built-in formatter/linter/test runner, web-standard APIs.
- **Deno 2** added strong Node/npm compatibility (`npm:` specifiers, `package.json`); now at 2.9.

| | Node | Bun | Deno |
|---|---|---|---|
| Engine | V8 | JavaScriptCore | V8 |
| TS | Type stripping (erasable only) | Native | Native |
| Package manager | npm (plus pnpm/Yarn) | Built-in | Built-in / npm compat |
| Permissions | Permission model (opt-in) | None | Secure by default |
| Ecosystem maturity | Highest | Growing | Growing |

## Vite+ (VoidZero)
- **Vite+ beta** (2026-07-02), **MIT open source**: a unified **`vp`** CLI covering **dev, check, test, build, pack and run**, bundling **Vite 8, Vitest, Rolldown, tsdown, Oxlint and Oxfmt** — one toolchain instead of wiring six tools.
- VoidZero **joined Cloudflare** on 2026-06-04; the projects remain MIT.
- Still beta — evaluate, but don't bet a large production monorepo on it yet.

> [!warning] Common pitfalls
> - Mixing package managers in one repo (two lockfiles) → non-reproducible installs. Enforce one via the `packageManager` field and CI.
> - Running ESLint stylistic rules *and* Prettier → endless conflicts.
> - Type-aware lint rules on every keystroke in huge repos — slow; scope them or use Biome/Oxlint for the fast path.
> - Relying on phantom dependencies (works with npm, breaks with pnpm).
> - Using Node's type stripping with `enum`s — use `as const` objects instead.

> [!info]+ 🆕 2025–2026
> - **ESLint 10** (2026-02-06): flat config only.
> - **Biome v2** (2025-06-17): type-aware linting without `tsc`, GritQL plugins; v2.5 (2026-06-12) ~500 rules.
> - **Oxlint 1.0** (2025-06-10), now 1.85; **Oxfmt** in alpha (later status unverified).
> - **Vite+ beta** (2026-07-02, MIT); **VoidZero joined Cloudflare** (2026-06-04).
> - **pnpm 11** (2026-04-28) 1-day `minimumReleaseAge` default; **pnpm 12** (2026-08-26) Rust rewrite.
> - **Node 24 LTS** (2025-10-28); **Node 26** (2026-05-05); yearly majors with every release LTS starting Node 27.
> - **Bun 1.4** (2026-08-20); **Deno 2.9**.
> - Next.js 16 removed `next lint`; `eslint-plugin-react-hooks` v6 uses flat config and includes React Compiler rules.

## Interview questions

> [!question]- Linter vs formatter?
> A linter analyses code for bugs and bad patterns (and can auto-fix some); a formatter only reprints code in a consistent style. Use both, and disable stylistic lint rules so they don't fight.

> [!question]- What changed with ESLint flat config?
> Config is a JS array of objects in `eslint.config.js`, plugins are imported as modules, there's no cascading `.eslintrc` lookup or string-based `extends`. ESLint 10 supports only flat config.

> [!question]- Why pnpm over npm?
> A content-addressable store with hard links saves disk and time; a strict symlinked `node_modules` prevents phantom dependencies; excellent workspace filtering for monorepos; safer defaults (no dependency install scripts since v10, 1-day release-age cooldown since v11).

> [!question]- What is a phantom dependency?
> Importing a package you didn't declare in `package.json` that happens to be hoisted into `node_modules` by another dependency. It can break whenever that dependency changes; pnpm's strict layout surfaces it.

> [!question]- Node vs Bun vs Deno?
> Node (V8) is the mature default with the largest ecosystem and now strips TS types natively. Bun (JavaScriptCore) is an all-in-one fast runtime/package manager/test runner with Node compatibility. Deno (V8 + Rust) is secure-by-default with built-in tooling and good npm compatibility.

> [!question]- How does Node's release schedule work now?
> Previously: two majors a year, even-numbered ones became LTS. From Node 27 (alpha Oct 2026, current Apr 2027): one major per year, and every release becomes LTS.

## Related
- [[Build Tools & Bundlers]]
- [[Monorepos]]
- [[Supply Chain Security]]
- [[Frontend Testing]]
- [[TypeScript 5.8 to 7.0]]
- [[TypeScript Essentials]]
- [[What's New in Frontend 2025-2026]]

## Sources
- [ESLint – Configuration files (flat config)](https://eslint.org/docs/latest/use/configure/configuration-files)
- [typescript-eslint – Getting started](https://typescript-eslint.io/getting-started/)
- [Biome v2 announcement](https://biomejs.dev/blog/biome-v2/)
- [Announcing Oxlint 1.0](https://voidzero.dev/posts/announcing-oxlint-1-stable)
- [Announcing Vite+ beta](https://voidzero.dev/posts/announcing-vite-plus-beta)
- [VoidZero joins Cloudflare](https://voidzero.dev/posts/voidzero-cloudflare)
- [Prettier – Rationale](https://prettier.io/docs/rationale)
- [pnpm – Motivation](https://pnpm.io/motivation), [pnpm 11](https://pnpm.io/blog/releases/11.0), [pnpm 12](https://pnpm.io/blog/releases/12.0)
- [Node.js – TypeScript](https://nodejs.org/api/typescript.html)
- [Node.js – Evolving the release schedule](https://nodejs.org/en/blog/announcements/evolving-the-nodejs-release-schedule)
- [Node.js 26.0.0](https://nodejs.org/en/blog/release/v26.0.0)
- [Bun docs](https://bun.sh/docs), [Deno docs](https://docs.deno.com/)
