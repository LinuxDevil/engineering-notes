---
title: "API Security"
tags: [security, security/api]
aliases: [OWASP API Security Top 10, API Top 10, BOLA, RBAC vs ABAC vs ReBAC, Authorization Models]
updated: 2026-09-27
---
# API Security

<Note>
**TL;DR**

APIs expose data and business logic directly, so their biggest risks are **authorization** failures, not exotic exploits: 3 of the OWASP API Security Top 10 (2023) items are about authorization at object, property and function level. Authenticate every call, authorize **every object and every field**, limit resource consumption, keep an inventory of all APIs, and treat third-party APIs as untrusted. For authorization design, start with **RBAC**, add **ABAC** conditions for context, and use **ReBAC** when permissions follow relationships (documents shared in folders, orgs and teams).
</Note>

## OWASP API Security Top 10 (2023)
The current edition is 2023 (the previous was 2019).

| # | Risk | Example | Fix |
|---|---|---|---|
| **API1** | Broken Object Level Authorization (BOLA) | `GET /accounts/42` returns another user's account | Check ownership/permission for every object id |
| **API2** | Broken Authentication | No rate limit on login, JWT `alg:none` accepted, long-lived tokens | Standard auth (OIDC), MFA, rate limits, full token validation |
| **API3** | Broken Object Property Level Authorization (BOPLA) | Response includes `passwordHash`; `PATCH` accepts `isAdmin` (mass assignment) | Response DTOs; allow-listed writable fields per role |
| **API4** | Unrestricted Resource Consumption | `?limit=1000000`, huge GraphQL queries, unlimited SMS sends | Pagination caps, rate limits, quotas, timeouts, query cost limits |
| **API5** | Broken Function Level Authorization (BFLA) | A normal user calls `POST /admin/users` | Deny by default; role checks per route/operation |
| **API6** | Unrestricted Access to Sensitive Business Flows | Bots buy all concert tickets, mass account creation, coupon abuse | Business-level limits, bot detection, device fingerprinting, CAPTCHA/step-up |
| **API7** | Server-Side Request Forgery | Webhook URL points to `169.254.169.254` | Destination allow-lists, block internal IPs ([Backend Security Essentials](/Software%20Engineering/Security/Backend%20Security%20Essentials)) |
| **API8** | Security Misconfiguration | Verbose errors, permissive CORS, missing TLS, default configs | Hardened config, automated checks |
| **API9** | Improper Inventory Management | Old `/v1` still live without auth; forgotten staging APIs | API inventory, gateway as single entry, retire old versions |
| **API10** | Unsafe Consumption of APIs | Trusting data from a partner API; following its redirects | Validate third-party responses, timeouts, TLS, limits |

Changes vs 2019: "Excessive Data Exposure" and "Mass Assignment" merged into API3 (BOPLA); new API6 (business flows), API7 (SSRF) and API10 (unsafe consumption); "Injection" and "Insufficient Logging" dropped as separate items (still important; see [OWASP Top 10](/Software%20Engineering/Security/OWASP%20Top%2010)).

## Authentication for APIs
| Caller | Recommended |
|---|---|
| Browser SPA of your own app | Session cookie via a BFF, or OAuth 2 authorization code + PKCE ([Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage)) |
| Mobile app | OAuth 2 authorization code + PKCE with the system browser |
| Third-party apps acting for users | OAuth 2 / OpenID Connect with scopes and consent ([OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect)) |
| Service to service | mTLS (service mesh) or client credentials / workload identity tokens |
| Simple server integrations | API keys: scoped, rotatable, hashed at rest, sent in a header |

Validate JWT access tokens fully: signature with a fixed algorithm, `iss`, `aud`, `exp`/`nbf`, and scopes. Keep them short-lived.

## Authorization patterns

### Where to enforce
{/* diagram source: /diagrams/diagram-380.mmd */}
![diagram](/diagrams/diagram-380.svg)
- The gateway handles coarse checks (valid token, scope `orders:read`).
- The **service** must do object-level checks: only it knows who owns order 42.
- Centralise the policy (a library or a policy engine) so every endpoint asks the same question the same way: `can(user, "read", order)`.

### RBAC vs ABAC vs ReBAC
| Model | Decision based on | Example rule | Good for | Weakness |
|---|---|---|---|---|
| **RBAC** (role-based) | User's roles → permissions | `admin` can `delete_user` | Simple apps, internal tools, coarse access | Role explosion (`editor_eu_marketing_readonly`); can't express ownership |
| **ABAC** (attribute-based) | Attributes of user, resource, action, environment | `user.department == doc.department AND time < 18:00` | Context rules, compliance, multi-tenant conditions | Policies harder to audit; "who can access X?" is hard to answer |
| **ReBAC** (relationship-based) | Graph of relationships | `viewer` of doc if `member` of a team that is `editor` of the parent folder | Sharing, hierarchies, Google Drive/GitHub-style permissions | Needs a relationship store kept in sync; new infrastructure |

```text
# RBAC
role admin: users:*, orders:*
role support: orders:read

# ABAC (Cedar-style)
permit(principal, action == Action::"read", resource)
when { principal.tenant == resource.tenant && resource.status != "archived" };

# ReBAC (Zanzibar / OpenFGA-style tuples)
document:plan#parent@folder:q3
folder:q3#editor@team:growth#member
team:growth#member@user:alice
→ check(user:alice, viewer, document:plan) = true
```
In practice most systems combine them: RBAC for coarse roles, ownership checks (a simple ReBAC), and ABAC conditions for tenant and context. Tools: OPA/Rego, AWS Cedar (Verified Permissions), OpenFGA and SpiceDB (Zanzibar-inspired), Casbin; ASP.NET Core policies and Spring Security method security for in-app checks.

### Worked example: BOLA + BOPLA fix in one handler
```ts
// PATCH /projects/:id   body: { name?, description?, ownerId? }
const Editable = {
  member: z.object({ name: z.string().max(80).optional(), description: z.string().max(2000).optional() }).strict(),
  owner:  z.object({ name: z.string().max(80).optional(), description: z.string().max(2000).optional(),
                     ownerId: z.string().uuid().optional() }).strict(),
};

app.patch('/projects/:id', auth, async (req, res) => {
  const project = await repo.findProjectForUser(req.params.id, req.user.id); // scoped: BOLA
  if (!project) return res.sendStatus(404);
  const role = project.ownerId === req.user.id ? 'owner' : 'member';
  const changes = Editable[role].parse(req.body);                          // field-level: BOPLA
  const updated = await repo.updateProject(project.id, changes);
  res.json(toProjectResponse(updated));                                    // response DTO: no internal fields
});
```

## Resource consumption and abuse
- Rate limits per user/API key/IP (token bucket in Redis) and per expensive operation; return `429` with `Retry-After`.
- Hard caps: page size, body size, upload size, GraphQL depth/complexity, batch size, timeouts on outbound calls.
- Cost-based limits for expensive resources (SMS, email, LLM tokens).
- Business-flow protection (API6): velocity checks, bot management, step-up verification for high-value actions.

## Inventory and lifecycle
- Every API has an owner, an OpenAPI/GraphQL schema, a data classification and an environment list.
- Route everything through a gateway so unknown ("shadow") and old ("zombie") APIs are visible.
- Deprecate with dates (`Sunset` header, RFC 8594) and actually turn old versions off.

## GraphQL-specific
- Disable introspection in production (or restrict it), enforce depth and cost limits, persisted queries for first-party clients, authorization in resolvers or a data layer (not only at the gateway), batching limits to stop brute force via aliases.

## Pitfalls
<Warning>
**Common mistakes**

- Authorization checked in the UI or the gateway only.
- Returning whole database entities and relying on the client to hide fields.
- Using sequential ids and thinking random ids would be "the fix".
- Rate limiting only by IP (attackers rotate IPs; limit per account/key too).
- Forgotten versions and test endpoints left open.
</Warning>

## Senior perspective
- Make the secure path the default: a framework-level `authorize()` that must be called (lint or test fails otherwise), repository methods that require a user/tenant context, response serializers per role.
- Write **authorization tests as a matrix**: every endpoint × every role × own/other's object. Generate them from the route table.
- Choose the authz model from the product's sharing model; migrating from RBAC to ReBAC later is expensive.
- Answer "who can access this resource and why?" for audits; ReBAC and explicit policies make that possible.

## Interview questions
<Accordion title="What is BOLA and why is it #1 in the API Top 10?">
Broken Object Level Authorization: the API doesn't verify the caller may access the specific object id in the request. It's #1 because it's extremely common, trivial to exploit (change an id) and invisible to generic scanners.
</Accordion>

<Accordion title="BOLA vs BFLA vs BOPLA?">
BOLA: wrong object (another user's order). BFLA: wrong function (a user calling an admin operation). BOPLA: wrong properties (reading sensitive fields or writing fields like `role` through mass assignment).
</Accordion>

<Accordion title="RBAC vs ABAC vs ReBAC: when would you use each?">
RBAC for coarse, stable permissions by job role. ABAC when decisions depend on attributes and context (tenant, region, time, data classification). ReBAC when access follows relationships and sharing (folders, teams, orgs), like Google Drive. Most real systems mix them.
</Accordion>

<Accordion title="How do you secure service-to-service calls?">
Mutual TLS with workload identities (e.g. SPIFFE via a service mesh) or short-lived tokens from client credentials / workload identity, plus authorization per caller service. Don't trust the network.
</Accordion>

<Accordion title="How do you protect an API from abuse and scraping?">
Authentication, per-account and per-key rate limits, quotas, pagination caps, bot detection for sensitive flows, anomaly detection on usage, and not exposing more data than a screen needs.
</Accordion>

<Accordion title="What are the main GraphQL security concerns?">
Query depth/complexity DoS, introspection leaking schema, authorization per field/resolver, batching/alias abuse for brute force, and N+1 queries causing resource exhaustion.
</Accordion>

## Related
- [Security Index](/Software%20Engineering/Security/Security%20Index)
- [OWASP Top 10](/Software%20Engineering/Security/OWASP%20Top%2010) · [Backend Security Essentials](/Software%20Engineering/Security/Backend%20Security%20Essentials) · [Secrets Management](/Software%20Engineering/Security/Secrets%20Management) · [Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling)
- [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) · [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA) · [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage)
- [Backend Interview Questions](/Software%20Engineering/Interviews/Backend%20Interview%20Questions)

## Sources
- [OWASP API Security Top 10 2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [OWASP GraphQL Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html)
- [NIST SP 800-162: Guide to ABAC](https://csrc.nist.gov/pubs/sp/800/162/upd2/final)
- [Zanzibar: Google's Consistent, Global Authorization System (USENIX ATC 2019)](https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/)
- [OpenFGA docs](https://openfga.dev/docs), [Cedar policy language](https://www.cedarpolicy.com/)
- [RFC 8594: The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594)
