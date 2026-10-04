---
title: "Node.js 22 to 24+ - What's New"
tags: [backend, backend/node]
aliases: [Node 24, Node 22, Node 25, Node 26, Node 27, What's new in Node, Node release schedule, Node TypeScript type stripping, node:sqlite, require(esm)]
updated: 2026-09-27
---
# Node.js 22 to 24+ - What's New

<Note>
**TL;DR**

As of **September 2026**: **Node 24 "Krypton"** is the Active LTS (LTS since 2025-10-28), **Node 22 "Jod"** is Maintenance LTS, **Node 26** is Current and becomes LTS on **2026-10-28**, and Node 20 and 25 are end-of-life. The big themes of 2024–2026: Node runs **TypeScript natively** (type stripping, stable), a **stable built-in test runner**, `require()` of ES modules, a stable **permission model**, built-in **`fetch`/`WebSocket`/`node:sqlite`/Web Storage/`URLPattern`/Temporal**, and, from Node 27, **one major release per year, every one LTS**.
</Note>

## Release timeline

| Version | First release | LTS | Status (2026-09) | End of life |
|---|---|---|---|---|
| **20 "Iron"** | 2023-04-18 | 2023-10 | **EOL** (last release 2026-03-24) | April 2026 |
| **22 "Jod"** | 2024-04-24 | 2024-10 | Maintenance LTS | April 2027 |
| 23 | 2024-10-16 | never (odd) | EOL | 2025 |
| **24 "Krypton"** | 2025-05-06 | **2025-10-28** (24.11.0) | **Active LTS** | end of April 2028 |
| 25 | 2025-10-15 | never (odd) | EOL | 2026 |
| **26** | 2026-05-05 | **2026-10-28** (scheduled) | Current | April 2029 |
| **27** | alpha from Oct 2026 | Oct 2027 | first release April 2027 | April 2030 |

{/* diagram source: /diagrams/diagram-076.mmd */}
![diagram](/diagrams/diagram-076.svg)

<Info>
**🆕 2026: new release schedule**

Announced **2026-03-10** on nodejs.org. Starting with **Node 27**:
- **One major per year.** A 6-month **Alpha** channel (October–March, semver-major changes allowed, versions like `27.0.0-alpha.1`, for library authors and CI, not production), then the **Current** release in **April**, then **LTS in October**.
- **Every major becomes LTS**; no more odd/even split. LTS lasts **30 months** from first release, 36 months total to EOL.
- Node 26 is the **last** release under the old model.
Production advice is unchanged: run Active or Maintenance LTS only.
</Info>

## Headline features

### Native TypeScript (type stripping)

```bash
node server.ts          # no flag needed (Node 22.18+ / 23.6+)
```

- Node **erases** type annotations and runs the result. No type checking (run `tsc --noEmit` in CI), and **no `tsconfig.json`** (no path aliases, no downlevelling).
- Only *erasable* syntax works. **Not supported**: `enum`, namespaces with runtime code, parameter properties (`constructor(private x)`), `import x = require()` aliases, decorators. TypeScript's `erasableSyntaxOnly` option (TS 5.8) flags these at compile time.
- Use `import type` for type-only imports and include the `.ts` extension in relative imports.

| Milestone | Version |
|---|---|
| `--experimental-strip-types` added | 22.6.0 |
| On by default (no flag) | 23.6.0 / 22.18.0 |
| **Stable** | **25.2.0 (2025-11-11) / 24.12.0 (2025-12-10)** |
| `--experimental-transform-types` removed | 26.0.0 |

See [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0) for the TS side.

### Built-in test runner (`node:test`)

Stable since Node 20. Includes `describe/it`, hooks, mocking (`mock.fn`, `mock.timers`, module mocking), snapshot testing, coverage (`--experimental-test-coverage`), watch mode and reporters. Node 24 made subtests auto-awaited.

```js
// users.test.ts   ->   node --test
import { test, describe, mock } from 'node:test';
import assert from 'node:assert/strict';
import { createUser } from './users.ts';

describe('createUser', () => {
  test('lowercases email', async () => {
    const save = mock.fn(async (u) => u);
    const user = await createUser({ email: 'A@B.COM' }, { save });
    assert.equal(user.email, 'a@b.com');
    assert.equal(save.mock.callCount(), 1);
  });
});
```

Jest/Vitest are still richer for frontend-style testing; `node:test` is great for libraries and services wanting zero dependencies. See [Frontend Testing](/Software%20Engineering/Frontend/13%20Testing/Frontend%20Testing).

### `require()` of ES modules

CommonJS code can `require()` a synchronous ESM module (no top-level `await`, else `ERR_REQUIRE_ASYNC_MODULE`). This ends most of the "dual package" pain.

| Milestone | Version |
|---|---|
| Added behind `--experimental-require-module` | 22.0.0 / 20.17.0 |
| Unflagged | 23.0.0 / 22.12.0 / 20.19.0 |
| No experimental warning | 23.5.0 / 22.13.0 |
| **Stable** | **25.4.0 (2026-01-19)** |

### Permission model

`node --permission --allow-fs-read=... --allow-net ...`. Stable since 23.5.0 / 22.13.0; flag renamed from `--experimental-permission` in 24.0.0; `--allow-net` added in 25.0.0. A seat belt, not a sandbox. See [Node.js Security Best Practices](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Security%20Best%20Practices).

### Web-platform APIs in core

| API | Status |
|---|---|
| `fetch`, `Request`, `Response`, `Headers`, `FormData` (undici) | Stable since Node 21 |
| `WebSocket` **client** (global) | On by default in 22, **stable in 22.4.0** |
| `URLPattern` global | Node 24.0.0 |
| Web Storage (`localStorage`/`sessionStorage`) | On by default in 25.0.0 (use `--localstorage-file` for persistence) |
| **Temporal** | On by default in **26.0.0** |
| `navigator`, `CustomEvent`, `ErrorEvent` (25), Web Streams, Web Crypto | globals |

```js
const res = await fetch('https://api.example.com/users', { signal: AbortSignal.timeout(3000) });
const ws = new WebSocket('wss://echo.example.com'); ws.onmessage = e => console.log(e.data);
const route = new URLPattern({ pathname: '/users/:id' }); route.exec('https://x.dev/users/42').pathname.groups.id; // '42'
Temporal.Now.plainDateISO().add({ days: 30 }).toString();   // Node 26
```

### `node:sqlite`

```js
import { DatabaseSync } from 'node:sqlite';
const db = new DatabaseSync('app.db');
db.exec('CREATE TABLE IF NOT EXISTS kv (k TEXT PRIMARY KEY, v TEXT) STRICT');
db.prepare('INSERT OR REPLACE INTO kv VALUES (?, ?)').run('theme', 'dark');
console.log(db.prepare('SELECT v FROM kv WHERE k = ?').get('theme'));   // { v: 'dark' }
```

Added in 22.5.0 (flagged), unflagged in 23.4.0 / 22.13.0, **release candidate (stability 1.2) since 25.7.0 (2026-02-24)**. Synchronous API: great for CLIs, tests, local tools and small services; for server workloads with concurrency use Postgres.

### Developer experience
- `node --watch` (stable in 22), `node --run <script>` (runs package.json scripts without npm overhead), `--env-file=.env` (20.6+, multiple files supported), `fs.glob` (22), compile cache (22.1+), single-executable apps.
- **npm 11** ships with Node 24. **Corepack is no longer bundled from Node 25**; install pnpm/Yarn explicitly (`npm i -g pnpm` or `corepack` from npm).

### Engine
| Node | V8 | Notable JS features |
|---|---|---|
| 22 | 12.4 | `Array.fromAsync`, Set methods, iterator helpers; **Maglev** on |
| 24 | 13.6 | `Float16Array`, explicit resource management (`using`), `RegExp.escape`, `Error.isError` |
| 25 | 14.1 | faster `JSON.stringify`, `Uint8Array` base64/hex |
| 26 | 14.6 | `Map.prototype.getOrInsert(Computed)`, `Iterator.concat`, Temporal |

See [Modern JavaScript ES2022-ES2027](/Software%20Engineering/Frontend/01%20JavaScript/Modern%20JavaScript%20ES2022-ES2027).

## Breaking changes & removals to know

| Version | Change |
|---|---|
| 24.0.0 | `url.parse()` runtime-deprecated; `tls.createSecurePair` removed; `SlowBuffer` EOL; `fs.F_OK` etc. deprecated; 32-bit s390/ppc dropped |
| 25.0.0 | `SlowBuffer` removed; `fs.F_OK`/`R_OK`/`W_OK`/`X_OK` removed; Corepack unbundled; `assert.CallTracker`, `fs.rmdir(recursive)` EOL |
| 26.0.0 | `http.writeHeader()` removed; legacy `_stream_*` modules removed; `module.register()` runtime-deprecated (use `module.registerHooks()`); `--experimental-transform-types` removed |

## Other runtimes: Deno 2 and Bun

- **Deno 2** (Oct 2024) added strong Node/npm compatibility (`package.json`, `npm:` specifiers), keeps secure-by-default permissions and built-in tooling (fmt, lint, test, compile).
- **Bun** (JavaScriptCore + Zig) is an all-in-one runtime, package manager, bundler and test runner focused on speed, with high Node API compatibility.
- Node's recent features (TS stripping, test runner, sqlite, permissions, watch, `--env-file`) closed much of the gap. Details and comparison: [Linting, Formatting, Package Managers & Runtimes](/Software%20Engineering/Frontend/14%20Tooling/Linting%2C%20Formatting%2C%20Package%20Managers%20%26%20Runtimes).

## Upgrade playbook
1. Read the release's "semver-major" list and deprecation codes; run tests with `--pending-deprecation` / `--throw-deprecation` on the old version first.
2. Rebuild native addons (`NODE_MODULE_VERSION` changes each major: 141 in 25, 147 in 26).
3. Update the Docker base image and `engines` field; test in CI on the target version.
4. Watch for behaviour changes (stream highWaterMark, `AsyncLocalStorage` implementation, OpenSSL defaults).
5. Roll out gradually (canary) and watch error rates and event-loop metrics.

## Senior perspective
- Stay on even/LTS lines until Node 27; then every major is LTS but still wait for the October LTS promotion for production.
- Budget an upgrade every year: EOL dates are fixed, and security patches stop.
- Native TS means you can drop `ts-node`/build steps for services and scripts, but you still need `tsc` for type checking and a bundler/transpiler if you use decorators (NestJS) or enums.
- `require(esm)` lets libraries publish ESM-only; audit your dependency tree before relying on it on Node 20 (EOL anyway).

## Interview questions

<Accordion title="Which Node version should production use in late 2026?">
Node 24 (Active LTS until Oct 2026's handover, supported to April 2028) or Node 22 (Maintenance to April 2027). Node 26 becomes LTS on 2026-10-28. Avoid odd/Current releases and EOL versions like 20.
</Accordion>

<Accordion title="How does Node run TypeScript natively and what are the limits?">
It strips erasable type syntax and runs the JS without type checking or reading tsconfig. Enums, runtime namespaces, parameter properties, import aliases and decorators aren't supported. Stable since 25.2 / 24.12.
</Accordion>

<Accordion title="What is `require(esm)` and why does it matter?">
CommonJS can now `require()` synchronous ES modules (no top-level await). It removes the need for dual CJS/ESM packages. Unflagged in 22.12/20.19, stable in 25.4.
</Accordion>

<Accordion title="What changes with Node's 2026 release schedule?">
From Node 27: one major per year (April), preceded by a 6-month alpha channel, LTS every October, and every release becomes LTS with 30 months of LTS support. Node 26 is the last odd/even-model release.
</Accordion>

<Accordion title="Name built-in features that replace common npm packages.">
`fetch` (node-fetch/axios for simple cases), `WebSocket` client, `node:test` (Jest/Mocha for many cases), `--watch` (nodemon), `--env-file` (dotenv), `fs.glob` (glob), `node:sqlite` (better-sqlite3 for simple uses), `util.parseArgs`, `util.styleText` (chalk), native TS (ts-node/tsx for simple cases).
</Accordion>

## Related
- [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index)
- [Node.js Security Best Practices](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Security%20Best%20Practices) · [Node.js Event Loop & libuv](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Event%20Loop%20%26%20libuv) · [Node.js Performance & Debugging](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Performance%20%26%20Debugging)
- [Linting, Formatting, Package Managers & Runtimes](/Software%20Engineering/Frontend/14%20Tooling/Linting%2C%20Formatting%2C%20Package%20Managers%20%26%20Runtimes) · [TypeScript 5.8 to 7.0](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%205.8%20to%207.0) · [Modern JavaScript ES2022-ES2027](/Software%20Engineering/Frontend/01%20JavaScript/Modern%20JavaScript%20ES2022-ES2027) · [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security)

## Sources
- [Node.js releases (status table)](https://nodejs.org/en/about/previous-releases)
- [Node.js blog: Evolving the Node.js release schedule (2026-03-10)](https://nodejs.org/en/blog/announcements/evolving-the-nodejs-release-schedule)
- [Node.js 22 release announcement](https://nodejs.org/en/blog/announcements/v22-release-announce)
- [Node.js 24.0.0 release notes (2025-05-06)](https://nodejs.org/en/blog/release/v24.0.0)
- [Node.js 24.11.0 LTS "Krypton" (2025-10-28)](https://nodejs.org/en/blog/release/v24.11.0)
- [Node.js 25.0.0 release notes (2025-10-15)](https://nodejs.org/en/blog/release/v25.0.0)
- [Node.js 26.0.0 release notes (2026-05-05)](https://nodejs.org/en/blog/release/v26.0.0)
- [Node.js docs: TypeScript (type stripping history)](https://nodejs.org/api/typescript.html)
- [Node.js docs: require(esm) history](https://nodejs.org/api/modules.html#loading-ecmascript-modules-using-require)
- [Node.js docs: SQLite](https://nodejs.org/api/sqlite.html)
- [Node.js docs: Permissions](https://nodejs.org/api/permissions.html)
- [Node.js docs: Test runner](https://nodejs.org/api/test.html)
- [Node.js Learn: Native WebSocket client](https://nodejs.org/learn/getting-started/websocket)
- [Deno 2 announcement](https://deno.com/blog/v2.0) · [Bun docs](https://bun.sh/docs)
