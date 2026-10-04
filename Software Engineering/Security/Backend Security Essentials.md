---
title: "Backend Security Essentials"
tags: [security, security/backend]
aliases: [Backend Security, Server-side Security, SQL Injection, SSRF, IDOR, Mass Assignment]
updated: 2026-09-27
---
# Backend Security Essentials

<Note>
**TL;DR**

Most backend breaches come from a short list: **injection** (fixed with parameterised queries), **broken access control / IDOR** (fixed by checking ownership on every object), **SSRF** (fixed by controlling where the server may connect), **mass assignment** (fixed with explicit DTOs), leaked **secrets**, weak **input validation**, missing **security headers**, and **logging** that either misses attacks or leaks data. Treat every input as untrusted, deny by default, and put controls in shared code so each endpoint doesn't reinvent them.
</Note>

## Mental model: trust boundaries
Every time data crosses a boundary (browser → API, API → DB, service → service, user file → parser), validate it on the receiving side and encode it for the next interpreter.

{/* diagram source: /diagrams/diagram-381.mmd */}
![diagram](/diagrams/diagram-381.svg)

## 1. Injection

### SQL injection
```js
// ❌ vulnerable
const rows = await db.query(`SELECT * FROM users WHERE email = '${email}'`);
// email = "x' OR '1'='1"  → returns every user
// email = "x'; DROP TABLE users; --"  → worse, if stacked queries are allowed
```
```js
// ✅ parameterised: the driver sends SQL and values separately
const rows = await db.query('SELECT * FROM users WHERE email = $1', [email]);
```
```java
// ✅ Java / JDBC
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE email = ?");
ps.setString(1, email);
```
```python
# ✅ Python (psycopg)
cur.execute("SELECT * FROM users WHERE email = %s", (email,))
```

Why it works: the database parses the query **before** the value arrives, so the value can never become SQL syntax.

Things parameters **can't** bind: table names, column names, `ORDER BY` direction. Use an allow-list:
```js
const SORTABLE = { name: 'name', created: 'created_at' };
const col = SORTABLE[req.query.sort] ?? 'created_at';
const dir = req.query.dir === 'asc' ? 'ASC' : 'DESC';
await db.query(`SELECT * FROM products ORDER BY ${col} ${dir} LIMIT $1`, [20]);
```
ORMs are safe by default but have raw escape hatches (`$queryRawUnsafe`, `Sequelize.literal`, JPQL built with `+`). Grep for them in review.

Defence in depth: least-privilege DB users (the app user can't `DROP`), no stacked queries, WAF only as an extra layer.

### Other injections
| Type | Vulnerable | Fix |
|---|---|---|
| NoSQL (MongoDB) | `find({ user: req.body.user, pass: req.body.pass })` with `{"pass": {"$ne": null}}` | Validate types (schema), cast to string, sanitise operators |
| OS command | `exec("convert " + filename)` | `execFile('convert', [filename])`, no shell; validate names |
| LDAP / XPath | String-built filters | Library escaping functions |
| Template (SSTI) | Rendering user input as a template | Never compile user input as a template |
| Log injection | Newlines in logged input forge entries | Structured (JSON) logging |

## 2. Broken access control and IDOR
**IDOR** (Insecure Direct Object Reference), called **BOLA** in API terms: the API authenticates the user but doesn't check the object belongs to them.

```js
// ❌ any logged-in user can read any order
app.get('/orders/:id', auth, async (req, res) => {
  res.json(await Order.findById(req.params.id));
});

// ✅ scope by owner (or check a policy)
app.get('/orders/:id', auth, async (req, res) => {
  const order = await Order.findOne({ _id: req.params.id, userId: req.user.id });
  if (!order) return res.sendStatus(404);
  res.json(order);
});
```
- Random UUIDs make guessing harder but are **not** access control.
- Check **function-level** access too: `DELETE /users/:id` must require admin, not just a hidden button in the UI.
- Put authorization in one place (middleware, policy engine, repository layer that always filters by tenant) and test every role against every endpoint.
- Multi-tenant apps: enforce tenant isolation in the data layer (e.g. PostgreSQL Row-Level Security) as a second line.

See [API Security](/Software%20Engineering/Security/API%20Security) for RBAC vs ABAC vs ReBAC.

## 3. SSRF (Server-Side Request Forgery)
The server fetches a URL the attacker controls (webhooks, "import from URL", PDF renderers, image proxies) and reaches internal systems.

Targets: cloud metadata (`http://169.254.169.254/`), internal admin APIs, `localhost` services, other VPC hosts.

```js
import { lookup } from 'node:dns/promises';
import ipaddr from 'ipaddr.js';

async function assertPublicUrl(raw) {
  const url = new URL(raw);
  if (!['https:'].includes(url.protocol)) throw new Error('scheme not allowed');
  const { address } = await lookup(url.hostname);
  const range = ipaddr.parse(address).range();
  if (range !== 'unicast') throw new Error('private address blocked'); // loopback, private, linkLocal...
  return { url, address }; // connect to THIS address to avoid DNS rebinding
}
```
Controls, strongest first:
1. Allow-list of destination hosts if possible.
2. Network egress controls: the fetching service runs in a subnet that can't reach internal ranges.
3. Resolve DNS once, validate the IP, connect to that IP (prevents DNS rebinding); don't follow redirects blindly (re-validate each hop).
4. IMDSv2 on AWS (requires a session token, blocks simple GETs).
5. Don't return raw responses to the user.

## 4. Mass assignment
Binding the whole request body to a model lets attackers set fields they shouldn't.
```js
// ❌ body: { "name": "Eve", "role": "admin", "balance": 1e6 }
await User.update(req.params.id, req.body);

// ✅ explicit allow-list via a DTO/schema
const UpdateProfile = z.object({ name: z.string().max(100), bio: z.string().max(500).optional() }).strict();
await User.update(req.user.id, UpdateProfile.parse(req.body));
```
Java/Spring: use request DTOs, not JPA entities, in `@RequestBody`. Rails: strong parameters. It's API3:2023 (Broken Object Property Level Authorization) in [API Security](/Software%20Engineering/Security/API%20Security).

## 5. Input validation
- Validate at the boundary with a schema (Zod, Joi, Bean Validation, Pydantic): type, length, range, format, allowed values.
- **Allow-list** over deny-list.
- Validation is **not** output encoding: still parameterise SQL and encode HTML.
- Limit sizes: body size, array lengths, string lengths, upload size, JSON depth, and regex complexity (ReDoS).
- Files: check type by content, not extension; store outside the web root with random names; scan; serve with `Content-Disposition: attachment` and a separate domain.
- Canonicalise before validating paths (`path.resolve`, then check it stays under the base dir) to stop `../` traversal.

## 6. Secrets
Never in source code, images, logs or client bundles. Use a secrets manager or workload identity; rotate; scan commits. Full note: [Secrets Management](/Software%20Engineering/Security/Secrets%20Management).

## 7. Authentication and sessions (backend side)
- Hash passwords with Argon2id / scrypt / bcrypt ([Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers)).
- Rate-limit login, reset and OTP endpoints per account and per IP.
- Session cookies: `HttpOnly; Secure; SameSite=Lax` (or `Strict`), rotate the session id on login, invalidate on logout and password change.
- Validate JWTs fully: signature with an expected algorithm (reject `none`), `iss`, `aud`, `exp`; keep them short-lived. See [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) and [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage).
- Prefer passkeys/MFA ([Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA)).

## 8. Security headers (for APIs and pages)
| Header | Value (typical) | Why |
|---|---|---|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` | Force HTTPS |
| `Content-Security-Policy` | Nonce-based `script-src`; for pure APIs `default-src 'none'; frame-ancestors 'none'` | XSS mitigation, clickjacking |
| `X-Content-Type-Options` | `nosniff` | Stop MIME sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Don't leak URLs |
| `Cache-Control` | `no-store` for sensitive responses | Don't cache personal data |
| CORS headers | Specific origins, never `*` with credentials | Stop cross-origin data reads |

Remove `X-Powered-By` and version banners. Browser-side details: [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).

## 9. Error handling and logging
- Return generic errors to clients (RFC 9457 Problem Details with an error id), log details server-side.
- **Fail closed** on authorization errors ([OWASP Top 10](/Software%20Engineering/Security/OWASP%20Top%2010) A10).
- Log security events: login success/failure, MFA changes, authorization denials, admin actions, input validation failures at scale. Include user id, request id, IP, and outcome.
- Never log passwords, tokens, full card numbers, session cookies or secrets; redact at the logger level.
- Send logs to a central, append-only store with retention and alerts (e.g. 50 failed logins for one account in 5 minutes).

## Pitfalls
<Warning>
**Common mistakes**

- Validating on the client only.
- "It's an internal API, no auth needed": internal networks get breached; use service identity (mTLS, signed tokens).
- Escaping instead of parameterising SQL.
- Treating UUIDs as secrets.
- Returning full DB entities in responses (sensitive fields leak).
- Catch-all exception handlers that swallow errors and continue.
</Warning>

## Senior perspective
- Build **paved roads**: a shared auth middleware, a DB layer that always filters by tenant, a safe HTTP client with SSRF protections, a logger with redaction. Individual engineers then can't easily get it wrong.
- In reviews, look at **data flow**: where does input enter, where is it interpreted (SQL, shell, HTML, URL fetch), and where is the authorization check?
- Automate: SAST rules for raw queries and `exec`, DAST in staging, authorization tests per role ([Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling)).
- Plan for breach: least privilege, segmentation, short-lived credentials and audit logs reduce the blast radius.

## Interview questions
<Accordion title="How do parameterised queries prevent SQL injection?">
The query text is sent and parsed separately from the values; values are bound as data and can't change the query's structure. Identifiers can't be parameters, so allow-list them.
</Accordion>

<Accordion title="What is IDOR/BOLA and how do you prevent it systematically?">
Accessing another user's object by changing an id. Prevent by checking ownership or permission for every object access server-side, ideally via scoped queries or a central policy, plus automated tests that use a second user's token.
</Accordion>

<Accordion title="What is SSRF and how do you defend against it?">
Making the server send requests to attacker-chosen destinations, often internal. Defend with destination allow-lists, blocking private/link-local IPs after resolution, network egress rules, IMDSv2, and re-validating redirects.
</Accordion>

<Accordion title="What is mass assignment?">
Automatically binding request fields to model properties, letting attackers set fields like `role` or `isVerified`. Use explicit DTOs or allow-listed schemas.
</Accordion>

<Accordion title="What should and shouldn't be logged?">
Log security-relevant events with context (who, what, when, result, request id). Never log credentials, tokens, secrets or unnecessary personal data. Make logs tamper-resistant and alert on them.
</Accordion>

<Accordion title="Is input validation enough to prevent injection?">
No. Validation reduces attack surface, but the fix for injection is context-correct handling at the interpreter: parameterised queries, argument arrays for commands, output encoding for HTML.
</Accordion>

## Related
- [Security Index](/Software%20Engineering/Security/Security%20Index)
- [OWASP Top 10](/Software%20Engineering/Security/OWASP%20Top%2010) · [API Security](/Software%20Engineering/Security/API%20Security) · [Secrets Management](/Software%20Engineering/Security/Secrets%20Management) · [Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling)
- [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) · [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage)
- [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) · [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA) · [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers)
- [Backend Interview Questions](/Software%20Engineering/Interviews/Backend%20Interview%20Questions)

## Sources
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
- [OWASP Mass Assignment Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)
- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [OWASP HTTP Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html)
- [AWS: Use IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
