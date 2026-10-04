---
title: "APIs & Auth Index"
tags: [apis, index]
aliases: [APIs Index, Auth Index, API Design Index, APIs MOC]
updated: 2026-09-27
---
# APIs & Auth Index

<Note>
**Start here**

Everything about designing, securing and operating APIs: REST, GraphQL and gRPC styles, HTTP semantics, versioning and idempotency, rate limiting, real-time delivery, and the whole identity stack (sessions, JWTs, OAuth/OIDC, passkeys, authorization models). Notes go beginner → senior, each with worked examples, a 🆕 2025–2026 box and interview questions. Infrastructure around APIs (Nginx, gateways, background jobs) lives in [Backend Index](/Software%20Engineering/Backend/Backend%20Index).
</Note>

{/* diagram source: /diagrams/diagram-028.mmd */}
![diagram](/diagrams/diagram-028.svg)

## 🟢 Beginner: the fundamentals
| Note | What you'll learn |
|---|---|
| [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) | resources, methods, status codes, pagination, Problem Details errors, OpenAPI |
| [HTTP Headers vs Body - Where Data Belongs](/Software%20Engineering/APIs%20%26%20Auth/HTTP%20Headers%20vs%20Body%20-%20Where%20Data%20Belongs) | path vs query vs headers vs body, standard and custom headers, correlation IDs |
| [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization) | AuthN vs AuthZ, 401 vs 403, RBAC/ABAC/ReBAC |
| [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) | sessions vs tokens, what a JWT really is (signed, not encrypted), refresh and revocation |
| [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST) | when a typed query language beats resource endpoints |

## 🟡 Intermediate: building real APIs
| Note | What you'll learn |
|---|---|
| [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST) | Protobuf contracts, HTTP/2 streaming, gRPC-Web and Connect |
| [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility) | breaking changes, version strategies, Deprecation/Sunset, idempotency keys |
| [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling) | token/leaky bucket, sliding windows, Redis implementation, RateLimit headers |
| [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets) | server-side real-time and event delivery, signatures, scaling connections |
| [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) | Auth Code + PKCE, client credentials, device flow, token types, DPoP, OAuth 2.1 |
| [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA) | phishing-resistant MFA, WebAuthn ceremonies, TOTP, recovery, step-up |

## 🔴 Senior: security and architecture
| Note | What you'll learn |
|---|---|
| [API Security](/Software%20Engineering/Security/API%20Security) (Security folder) | OWASP API Top 10: BOLA, mass assignment, SSRF, resource consumption, inventory |
| [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) | gateway vs proxy vs mesh, BFF pattern, Gateway API, AI/MCP gateways |
| [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) | async processing behind APIs: retries, idempotency, durable execution |

The "Senior perspective" section in every note covers production concerns (scale, failure modes, review questions).

## Suggested study paths
<Info>
**🔁 From zero to solid API engineer**

1. [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) → [HTTP Headers vs Body - Where Data Belongs](/Software%20Engineering/APIs%20%26%20Auth/HTTP%20Headers%20vs%20Body%20-%20Where%20Data%20Belongs) → [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) (browser side of HTTP)
2. [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization) → [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) → [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) → [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA)
3. [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility) → [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling) → [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets)
4. [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST) → [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST)
5. [API Security](/Software%20Engineering/Security/API%20Security) → [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) → [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues)
</Info>

<Info>
**🔐 Auth deep dive**

[Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization) → [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) → [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) → [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA) → [API Security](/Software%20Engineering/Security/API%20Security) → frontend side: [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) and [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)
</Info>

<Info>
**🎯 One week before an interview**

Day 1 [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) · Day 2 [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) + [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) · Day 3 [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling) (implement a token bucket) · Day 4 [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility) · Day 5 [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST) + [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST) · Day 6 [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets) · Day 7 [API Security](/Software%20Engineering/Security/API%20Security). Answer every "Interview questions" callout out loud.
</Info>

## 🆕 2025–2026 in this area
- **2025-01** RFC 9700 OAuth 2.0 Security BCP → [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect)
- **2025-03** RFC 9745 `Deprecation` header → [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility)
- **2025-04** RFC 9728 Protected Resource Metadata → [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect)
- **2025-07/08** NIST SP 800-63B-4: synced passkeys at AAL2, new password rules → [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA)
- **2025-09-08** GraphQL spec, September 2025 edition (OneOf inputs, schema coordinates) → [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST)
- **2025-09** OpenAPI 3.2 (streaming, QUERY/any method, structured tags, device flow) → [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design)
- **2025-10** `Idempotency-Key` header draft -07 (still a draft) → [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility)
- **2025-10-14** FIDO Passkey Index launched → [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA)
- **2026-03-02** OAuth 2.1 draft -15 (still not an RFC) → [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect)
- **2026-05-23** RateLimit headers draft -11 (still a draft) → [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling)
- **2026-06** RFC 10008 HTTP `QUERY` method → [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) · [HTTP Headers vs Body - Where Data Belongs](/Software%20Engineering/APIs%20%26%20Auth/HTTP%20Headers%20vs%20Body%20-%20Where%20Data%20Belongs)
- **2026-07-28** MCP spec: stateless core, routing headers, OAuth hardening, legacy HTTP+SSE deprecated → [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) · [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets)
- **2026-08** RFC 10017 OAuth for Browser-Based Apps (BFF recommended) → [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) · [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies)
- **2026-08-25** WebAuthn Level 3 becomes a W3C Recommendation → [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA)
- **2026-09-01** Kong AI Gateway 2.0 GA (MCP bundling and ACLs) → [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies)

## 🤖 MCP: APIs for AI agents
The Model Context Protocol turns APIs into **tools** that AI agents call. For API teams this means: good OpenAPI/GraphQL descriptions become tool descriptions; OAuth (RFC 9728 metadata, PKCE, issuer validation) secures agent access; gateways route and meter tool calls; and model-generated arguments are untrusted input. Protocol details: [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI).

## Related
- [Software Engineering](/Software%20Engineering/Software%20Engineering), the vault root
- [Backend Index](/Software%20Engineering/Backend/Backend%20Index) (Nginx, gateways, jobs, Node.js, Java & Spring)
- Frontend counterparts: [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) · [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage) · [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) · [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index)
