---
title: "Node.js Security Best Practices"
tags: [backend, backend/node, security]
aliases: [Node security, Node.js security, Node permission model, Express security]
updated: 2026-09-27
---
# Node.js Security Best Practices

<Note>
**TL;DR**

Most Node breaches come from **untrusted input** (injection, path traversal, prototype pollution, ReDoS), **dependencies** (malicious or vulnerable npm packages), and **misconfiguration** (secrets in code, missing headers, no rate limits, verbose errors). Validate every input with a schema, use parameterised queries, never pass user input to a shell, keep dependencies pinned and audited, run on a supported LTS, and use the **permission model** as a seat belt, not a sandbox.
</Note>

## Threat map

{/* diagram source: /diagrams/diagram-083.mmd */}
![diagram](/diagrams/diagram-083.svg)

## Input handling

**Validate at the boundary** with a schema (zod, valibot, JSON Schema/Ajv, Fastify schemas). Reject unknown fields; cap sizes.

### Injection

```js
// SQL: never concatenate
await db.query(`SELECT * FROM users WHERE email = '${email}'`);        // ❌
await db.query('SELECT * FROM users WHERE email = $1', [email]);      // ✅ parameterised

// NoSQL (MongoDB) operator injection: {"password": {"$ne": null}}
const { email, password } = LoginSchema.parse(req.body);              // ✅ ensures strings

// Command injection
exec(`convert ${req.query.file} out.png`);                           // ❌ shell
execFile('convert', [safeFile, 'out.png']);                          // ✅ args array, no shell
```

### Path traversal

```js
import path from 'node:path';
const ROOT = path.resolve('uploads');
function safeJoin(userPath) {
  const full = path.resolve(ROOT, userPath);
  if (!full.startsWith(ROOT + path.sep)) throw new Error('Invalid path');   // blocks ../../etc/passwd
  return full;
}
```

### Prototype pollution

Merging user JSON into objects can set `__proto__`/`constructor.prototype` and change behaviour app-wide.

```js
// ❌ naive deep merge of req.body into defaults
// ✅ validate with a schema; use Object.create(null) or Map for dictionaries;
//    node --disable-proto=delete removes the __proto__ accessor
```

### ReDoS
Regexes with nested quantifiers (`/(a+)+$/`) can take exponential time on crafted input and block the event loop. Use simple regexes, length limits, or a linear-time engine (e.g. `re2`).

### Other input risks
- **SSRF**: fetching user-supplied URLs lets attackers reach internal services (cloud metadata `169.254.169.254`). Allow-list hosts, resolve and block private IP ranges, disable redirects.
- **Deserialization**: never `eval`, `new Function`, or `vm` on untrusted input (`vm` is **not** a security boundary).
- **File uploads**: limit size/count, check type by content, store outside the web root with random names.

## HTTP hardening

```js
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import cors from 'cors';

app.disable('x-powered-by');
app.use(helmet());                                              // CSP, HSTS, noSniff, frameguard...
app.use(cors({ origin: ['https://app.example.com'], credentials: true }));
app.use(express.json({ limit: '100kb' }));
app.use('/auth', rateLimit({ windowMs: 15 * 60_000, limit: 20 }));
```

- Set timeouts (`server.headersTimeout`, `requestTimeout`) against Slowloris.
- Return generic error messages; log details server-side (no stack traces in responses, `NODE_ENV=production`).
- Cookies: `HttpOnly`, `Secure`, `SameSite=Lax/Strict`. Tokens and sessions: see [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) and [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies).
- Browser-side defences (XSS, CSRF, CSP): [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).

## Authentication & crypto
- Hash passwords with **argon2id**, **scrypt** or **bcrypt** (async variants; they use the thread pool). Never plain SHA-256.
- Compare secrets with `crypto.timingSafeEqual`.
- Use `crypto.randomUUID()` / `randomBytes` for tokens, never `Math.random()`.
- Verify JWTs with an explicit algorithm allow-list and check `exp`, `aud`, `iss`.

## Secrets & config
- Never commit `.env`; load secrets from the environment or a secret manager. Node 20.6+ supports `node --env-file=.env` natively (no dotenv needed).
- Don't log secrets or full request bodies; redact (pino `redact`).
- Least-privilege DB users and cloud IAM roles.

## Dependencies (supply chain)
- Commit the lockfile and install with `npm ci` (or `pnpm install --frozen-lockfile`).
- `npm audit`, Dependabot/Renovate, Socket or Snyk for known CVEs and suspicious packages.
- Disable or review install scripts (`npm ci --ignore-scripts`, pnpm's default of not running dependency build scripts unless allowed).
- Prefer packages with provenance (`npm publish --provenance`) and pin versions; watch for typosquats.
- Full background: [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security).

## Runtime
- Run a **supported LTS** (24 or 22 as of Sept 2026; Node 20 reached EOL in 2026). EOL versions get no security fixes.
- Don't run as root in containers (`USER node` in the official image); read-only filesystem where possible.
- Subscribe to Node security releases; they ship coordinated patches for all supported lines.

### The permission model

```bash
node --permission --allow-fs-read=/app --allow-fs-write=/app/tmp server.js
node --permission --allow-net --allow-worker server.js
```

```js
if (!process.permission.has('fs.write', '/etc')) console.log('cannot write /etc');
```

- Without flags, `--permission` blocks fs access, child processes, workers, native addons, WASI (and network in newer versions) unless allowed.
- Node's docs call it a **seat belt**: it stops *trusted* code from accidentally touching things it shouldn't. It does **not** protect against **malicious** code, which can bypass it. Use OS/container isolation for untrusted code.

<Info>
**🆕 2025–2026**

- Permission model **stable** in Node **23.5.0 / 22.13.0** (Dec 2024 / Jan 2025); **Node 24.0.0** (2025-05-06) renamed the flag from `--experimental-permission` to `--permission`.
- **Node 25.0.0** (2025-10-15) added `--allow-net` (network gating) and `--allow-inspector`. Current docs also list `--permission-audit` (log instead of deny) and `process.permission.drop()` to give up access after startup.
- **Node 24** runtime-deprecated `url.parse()` (use WHATWG `URL`, which avoids parser-differential bugs).
- **Node 20 reached end-of-life in 2026**; Node 26 becomes LTS in Oct 2026. See [Node.js 22 to 24+ - What's New](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%2022%20to%2024%2B%20-%20What's%20New).
- npm supply-chain attacks were a major 2025 theme: the **chalk/debug** compromise (2025-09-08) and the self-replicating **Shai-Hulud** worm (2025-09-14), followed by npm's plan (2025-09-22) for 2FA-only publishing, short-lived tokens and trusted publishing. Details and defences in [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security). Backend impact is worse than frontend: install scripts run on build servers with cloud credentials.
</Info>

## Pitfalls
- Trusting `X-Forwarded-For` without configuring `trust proxy` correctly (rate limits bypassed or all users share one IP).
- CORS `origin: '*'` with credentials (browsers refuse it, and people then reflect any origin, which is worse).
- Returning Mongo/ORM errors verbatim to clients.
- Logging `req.headers` (leaks `Authorization` and cookies).

## Senior perspective
- Threat-model each endpoint: who can call it, what input, what can it reach (DB, network, filesystem)?
- Make secure the default: shared validation middleware, a safe HTTP client wrapper (timeouts, SSRF protection), a DB layer that only allows parameterised queries.
- Security in CI: SAST (Semgrep, CodeQL), dependency scanning, secret scanning, container image scanning, and SBOMs.
- Plan for incident response: rotate secrets quickly, audit logs with request IDs, know how to roll a patched Node version out fast.

## Interview questions

<Accordion title="How do you prevent SQL/NoSQL injection in Node?">
Parameterised queries or a query builder/ORM that parameterises, and schema validation that ensures fields are the expected types (blocking Mongo operator objects like `{"$ne": null}`).
</Accordion>

<Accordion title="What is prototype pollution?">
Injecting properties into `Object.prototype` via keys like `__proto__` in merged user input, which changes behaviour of every object (auth bypass, DoS, sometimes RCE). Prevent with schema validation, safe merges, `Object.create(null)`/`Map`, and `--disable-proto`.
</Accordion>

<Accordion title="What is ReDoS and why is it worse in Node?">
A regex with catastrophic backtracking takes exponential time on crafted input. Because Node runs JS on one thread, one request can block all others. Use safe patterns, input length limits, or RE2.
</Accordion>

<Accordion title="Is Node's permission model a sandbox for untrusted code?">
No. It's a seat belt for trusted code; the docs state malicious code can bypass it. Use containers, separate processes/users, or isolates with strict boundaries.
</Accordion>

<Accordion title="How do you secure npm dependencies?">
Lockfiles with `npm ci`, automated updates, audits and malware scanning, restricting install scripts, preferring provenance-signed packages, minimal dependencies, and reviewing new additions.
</Accordion>

## Related
- [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index)
- [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security) · [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) · [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) · [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) · [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization)
- [Building HTTP Servers - Express, Fastify & Hono](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Building%20HTTP%20Servers%20-%20Express%2C%20Fastify%20%26%20Hono) · [Node.js 22 to 24+ - What's New](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%2022%20to%2024%2B%20-%20What's%20New)

## Sources
- [Node.js: Security best practices](https://nodejs.org/en/learn/getting-started/security-best-practices)
- [Node.js docs: Permissions](https://nodejs.org/api/permissions.html)
- [Node.js 24.0.0 release notes](https://nodejs.org/en/blog/release/v24.0.0) · [Node.js 25.0.0 release notes](https://nodejs.org/en/blog/release/v25.0.0)
- [Node.js releases (support status)](https://nodejs.org/en/about/previous-releases)
- [OWASP Node.js Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Nodejs_Security_Cheat_Sheet.html)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Express: Production security best practices](https://expressjs.com/en/advanced/best-practice-security.html)
