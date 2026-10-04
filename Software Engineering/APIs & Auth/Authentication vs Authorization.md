---
title: "Authentication vs Authorization"
tags: [apis, apis/auth]
aliases: [AuthN vs AuthZ, AuthN, AuthZ, Authorization Models, RBAC, ABAC, ReBAC, Access Control]
updated: 2026-09-27
---
# Authentication vs Authorization

<Note>
**TL;DR**

**Authentication (AuthN)** answers *who are you?* and **authorization (AuthZ)** answers *what are you allowed to do?* AuthN always comes first and produces an identity (a session or token with claims); AuthZ then checks that identity against a policy on **every request**. Authorization models range from simple roles (**RBAC**) to attributes (**ABAC**) to relationships (**ReBAC**, Google Zanzibar style). Broken authorization, not broken login, is the #1 API vulnerability.
</Note>

## The idea

- **Authentication** is the process of verifying who someone is (a user, a service, a device).
- **Authorization** is the process of verifying what specific applications, files, and data that identity has access to.
- HTTP status codes follow the same split: **401 Unauthorized** really means *unauthenticated* (who are you?), **403 Forbidden** means *authenticated but not allowed*.

| Authentication | Authorization |
| --- | --- |
| First step of identity and access management | Always takes place after authentication |
| Visible to and partially changeable by the user (password, passkey, MFA device) | Not visible to or changeable by the user; set by the organization/app |
| Verifies **who** the user is | Determines **what** resources a user can access and which actions they can perform |
| Works through passwords, one-time codes, passkeys, biometrics, client certificates | Works through roles, permissions, policies and ownership rules maintained by the app |
| Output: an identity (session, ID token, access token with claims) | Output: allow / deny (sometimes with a filtered result set) |
| Example: an employee signs in with a passkey | Example: once signed in, an employee sees their own payslip; an HR manager sees everyone's in their department |
| Standards: [WebAuthn](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA), OpenID Connect, SAML | Standards: [OAuth 2.x](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) scopes, XACML, Cedar, OPA/Rego |

<Warning>
**OAuth is authorization, not authentication**

OAuth 2.x delegates *access* (an app can call an API on your behalf). It does not tell the client who you are. **OpenID Connect** adds the ID token that does. See [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect).
</Warning>

## How it fits in a request

{/* diagram source: /diagrams/diagram-029.mmd */}
![diagram](/diagrams/diagram-029.svg)

- Coarse checks (is the token valid, has it scope `invoices:read`?) can live at the [gateway](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies).
- **Fine-grained checks (does user 7 own invoice 42?) must live in the service**, next to the data. A gateway cannot know ownership.

## Authorization models

| Model | Decision based on | Example rule | Good for | Weak spot |
|---|---|---|---|---|
| **ACL** (access control list) | a list of who can access each object | `doc:42 → [alice: edit, bob: view]` | file systems, small apps | hard to manage at scale |
| **RBAC** (role-based) | the user's role(s) | `role=admin → can delete users` | internal tools, most B2B SaaS | "role explosion" (`editor_eu_readonly_v2`), no per-object rules |
| **ABAC** (attribute-based) | attributes of user, resource, action, environment | `user.dept == doc.dept && time < 18:00` | regulated data, multi-tenant rules | policies get hard to reason about and audit |
| **ReBAC** (relationship-based) | a graph of relationships | `user → member → team → owner → folder → parent → doc` | Google Drive / GitHub style sharing | needs a dedicated service (Zanzibar, SpiceDB, OpenFGA) |
| **PBAC** (policy-based) | policies as code evaluated by an engine | Rego / Cedar policies | centralised, testable policy | another moving part in the request path |

- **Role-based access control (RBAC)** gives users access to information based on their role within the organization ([SailPoint: RBAC](https://www.sailpoint.com/identity-library/what-is-role-based-access-control/)).
- **Attribute-based access control (ABAC)** grants permissions on a more granular level than RBAC using a series of specific attributes ([SailPoint: ABAC](https://www.sailpoint.com/identity-library/what-is-attribute-based-access-control/)).
- In practice most systems are **RBAC + ownership checks** (RBAC for "what kind of user", plus `resource.owner_id == user.id` or `resource.tenant_id == user.tenant_id`).

### Worked example: RBAC + ownership in code

```ts
// Express-style middleware: coarse RBAC
const requireRole = (...roles: string[]) => (req, res, next) =>
  roles.some(r => req.user.roles.includes(r)) ? next() : res.status(403).end();

// Handler: fine-grained object-level check (prevents BOLA/IDOR)
app.get('/invoices/:id', requireRole('customer', 'support'), async (req, res) => {
  const invoice = await db.invoice.findUnique({ where: { id: req.params.id } });
  if (!invoice) return res.status(404).end();
  const isOwner = invoice.customerId === req.user.id;
  const isSupport = req.user.roles.includes('support') && invoice.tenantId === req.user.tenantId;
  if (!isOwner && !isSupport) return res.status(404).end(); // 404 hides existence
  res.json(invoice);
});
```

Better still: **push the check into the query** so you can never forget it:

```sql
SELECT * FROM invoices
WHERE id = $1 AND tenant_id = $2;  -- tenant scoping is part of every query
```

(PostgreSQL Row-Level Security can enforce this at the database layer.)

### Worked example: ReBAC (OpenFGA-style model)

```text
type user
type team
  relations
    define member: [user]
type folder
  relations
    define owner: [user, team#member]
    define viewer: [user, team#member] or owner
type document
  relations
    define parent: [folder]
    define viewer: [user] or viewer from parent
```

Question asked at runtime: `check(user:alice, viewer, document:roadmap)` → the engine walks the graph (alice → member of team:eng → owner of folder:plans → parent of document:roadmap) and answers `allow`.

## Service-to-service identity

Not every caller is a human:
- **API keys**: simple, but they are bearer secrets with no expiry by default. Scope them, rotate them, hash them at rest (store like passwords).
- **OAuth client credentials grant**: a service gets a short-lived access token ([OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect)).
- **mTLS / workload identity (SPIFFE)**: the service's certificate *is* its identity; common inside a service mesh.

## Pitfalls

<Warning>
**Common mistakes**

- **Checking authorization only in the UI.** Hiding a button is not access control; the API must enforce it.
- **Trusting IDs from the client** (`GET /users/123/orders` without checking 123 is the caller). This is **BOLA/IDOR**, #1 in the [OWASP API Top 10](/Software%20Engineering/Security/API%20Security).
- **Checking at the gateway only.** Gateways do coarse checks; object-level rules belong in the service.
- **Stale permissions in long-lived JWTs.** Roles baked into a token stay valid until it expires. Keep access tokens short or look up permissions server-side. See [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies).
- **Returning 403 for other tenants' resources**, which confirms they exist. Prefer 404.
- **Mass assignment**: `PATCH /users/me {"role":"admin"}` accepted because the handler spreads the body into the model.
</Warning>

## Senior perspective

- **Centralise policy, decentralise enforcement.** Each service enforces, but policies come from one place (a library, OPA sidecar, Cedar, or a ReBAC service like SpiceDB/OpenFGA) so they are consistent, testable and auditable.
- **Default deny.** New endpoints should fail closed. Add a test that every route has an explicit authorization rule.
- **Authorization affects performance.** List endpoints need "filter by what I can see" (a query), not "load everything then check each row". ReBAC systems offer `ListObjects`/lookup APIs for this; check their latency and consistency model (Zanzibar "zookies" solve the new-enemy problem).
- **Audit logs**: who did what, to what, when, and which policy allowed it. Required for SOC 2 / ISO 27001 and incident response.
- **Multi-tenancy**: tenant ID must come from the verified token, never from the request body or a header the client controls.
- Review questions: *Where is the object-level check? What happens when a role is removed mid-session? How do support/admin impersonation and break-glass access work, and are they logged?*

## Interview questions

<Accordion title="What is the difference between authentication and authorization?">
Authentication verifies identity (who you are) using credentials such as passwords, passkeys or certificates. Authorization decides what that identity may do, using roles, attributes or relationships. AuthN happens first and produces an identity; AuthZ runs on every request against that identity.
</Accordion>

<Accordion title="401 vs 403?">
401 means the request lacks valid authentication (missing/expired token); the response should carry `WWW-Authenticate`. 403 means the server knows who you are but you are not allowed. For resources in another tenant, many APIs return 404 so they don't reveal existence.
</Accordion>

<Accordion title="RBAC vs ABAC vs ReBAC?">
RBAC grants permissions through roles; simple but coarse and prone to role explosion. ABAC evaluates attributes of user, resource, action and context; flexible but harder to audit. ReBAC derives permissions from a relationship graph (owner, member, parent), which models sharing hierarchies like Google Drive; it needs a dedicated engine such as Zanzibar/SpiceDB/OpenFGA.
</Accordion>

<Accordion title="What is BOLA/IDOR and how do you prevent it?">
Broken Object Level Authorization: the API accepts an object ID from the client and returns it without checking the caller may access it. Prevent it with object-level checks in every handler, scoping queries by owner/tenant, row-level security, unguessable IDs as defence in depth, and automated tests that try other users' IDs.
</Accordion>

<Accordion title="Is OAuth an authentication protocol?">
No. OAuth 2.x is a delegated authorization framework: it issues access tokens for APIs. OpenID Connect is the identity layer on top that adds the ID token and UserInfo endpoint to authenticate the user.
</Accordion>

<Accordion title="Where should authorization be enforced in a microservice architecture?">
Coarse checks (valid token, scope) at the gateway; fine-grained, object-level checks inside each service close to the data, ideally using a shared policy engine or library so rules stay consistent. Never only in the frontend.
</Accordion>

## Related
- [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) · [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) · [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA) · [API Security](/Software%20Engineering/Security/API%20Security)
- [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) · [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design)
- Frontend view: [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) · [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)
- Other domains: [Security Index](/Software%20Engineering/Security/Security%20Index) · [API Security](/Software%20Engineering/Security/API%20Security) · [Backend Security Essentials](/Software%20Engineering/Security/Backend%20Security%20Essentials)

## Sources
- [MDN: 401 Unauthorized](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/401) · [MDN: 403 Forbidden](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/403)
- [SailPoint: What is RBAC](https://www.sailpoint.com/identity-library/what-is-role-based-access-control/) · [SailPoint: What is ABAC](https://www.sailpoint.com/identity-library/what-is-attribute-based-access-control/)
- [NIST SP 800-162: Guide to ABAC](https://csrc.nist.gov/pubs/sp/800/162/upd2/final)
- [Google: Zanzibar, Google's Consistent, Global Authorization System (USENIX ATC 2019)](https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/)
- [OpenFGA docs](https://openfga.dev/docs) · [AWS Cedar](https://www.cedarpolicy.com/)
- [OWASP API Security Top 10 (2023): API1 BOLA](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
