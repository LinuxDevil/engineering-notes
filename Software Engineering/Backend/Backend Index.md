---
title: "Backend Index"
tags: [backend, index]
aliases: [Backend Home, Backend MOC, Server-side Index, BE Home]
updated: 2026-09-27
---
# Backend Index

<Note>
**Start here**

The domain home for everything server-side: runtimes (**Node.js**, **Java & Spring**), the infrastructure in front of your app (**Nginx**, **API gateways**, BFFs), the work that happens outside the request (**background jobs**, **cron**), and application architecture (**Rails Clean Architecture**). API design and authentication have their own home in [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index). Every topic note has worked examples, a 🆕 2025–2026 box where something changed, and interview questions.
</Note>

{/* diagram source: /diagrams/diagram-038.mmd */}
![diagram](/diagrams/diagram-038.svg)

## 🧭 Areas
| Area | Home note |
|---|---|
| Node.js (event loop, streams, HTTP servers, performance, security) | [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) |
| Java & Spring (language, JVM, concurrency, reactive, data access) | [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index) |
| API design & authentication | [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index) |
| System design, caching, message brokers | [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index) |
| Databases | see the Databases folder (e.g. [Keyset Pagination vs Offset](/Software%20Engineering/Databases/Database%20Round/Keyset%20Pagination%20vs%20Offset)) |
| Security | [Security Index](/Software%20Engineering/Security/Security%20Index) |

## 🟢 Beginner
- [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design): how a good HTTP API looks
- [HTTP Headers vs Body - Where Data Belongs](/Software%20Engineering/APIs%20%26%20Auth/HTTP%20Headers%20vs%20Body%20-%20Where%20Data%20Belongs)
- [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization) → [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies)
- [Nginx](/Software%20Engineering/Backend/Nginx): reverse proxy, config, TLS, load balancing
- [Cron Jobs](/Software%20Engineering/Backend/Cron%20Jobs): scheduling with cron, `whenever`, Kubernetes CronJobs
- Runtime basics: start of [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) or [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index)

## 🟡 Intermediate
- [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues): Sidekiq, Solid Queue, BullMQ, Celery; retries with backoff; idempotency
- [Scheduled Jobs Index](/Software%20Engineering/Backend/Scheduled%20Jobs/Scheduled%20Jobs%20Index): cron + message queues, BullMQ Job Schedulers, K8s CronJobs, a job control API and a Next.js dashboard to pause/resume/run/stop jobs, Playwright as a cron job
- [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets): server-side real-time and event delivery
- [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling)
- [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility)
- [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) · [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA)
- [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST) · [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST)
- [Rails Clean Architecture](/Software%20Engineering/Backend/Rails%20Clean%20Architecture): service objects, use cases, adapters

## 🔴 Senior
- [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies): gateway vs proxy vs mesh, BFF, Gateway API, AI/MCP gateways
- [API Security](/Software%20Engineering/Security/API%20Security) (OWASP API Top 10, in the Security folder)
- [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture) · [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns) · [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) · [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies)
- Runtime internals and production tuning: the senior sections of [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) and [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index)

## Suggested study paths
<Info>
**🔁 Backend foundations (any language)**

1. [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) → [HTTP Headers vs Body - Where Data Belongs](/Software%20Engineering/APIs%20%26%20Auth/HTTP%20Headers%20vs%20Body%20-%20Where%20Data%20Belongs) → [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization) → [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies)
2. One runtime in depth: [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) *or* [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index)
3. [Nginx](/Software%20Engineering/Backend/Nginx) → [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies)
4. [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) → [Cron Jobs](/Software%20Engineering/Backend/Cron%20Jobs) → [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets)
5. [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling) → [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility) → [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect)
6. [Rails Clean Architecture](/Software%20Engineering/Backend/Rails%20Clean%20Architecture) (architecture ideas apply to any framework) → [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index)
</Info>

<Info>
**⚡ Catch up on 2025–2026 only**

[Nginx](/Software%20Engineering/Backend/Nginx) (1.30, ingress-nginx retirement) → [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) (Gateway API, AI gateways, MCP) → [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) (Sidekiq 8, Rails 8.1 continuations, BullMQ schedulers) → [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) (RFC 9700, OAuth 2.1 draft, RFC 10017) → [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA) (WebAuthn L3, NIST 800-63B-4) → [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) (OpenAPI 3.2, QUERY method) → runtime updates in [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) and [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index)
</Info>

<Info>
**🎯 Backend interview week**

[REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) · [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) · [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling) (code a token bucket) · [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) · [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) · [Nginx](/Software%20Engineering/Backend/Nginx) · runtime Q&A in [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) / [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index) · then [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index)
</Info>

## 🆕 2025–2026 in this area
- **2025-03** IngressNightmare CVEs in Kubernetes ingress-nginx → [Nginx](/Software%20Engineering/Backend/Nginx)
- **2025-03-05** Sidekiq 8.0 (profiler, job iteration, new Web UI) → [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues)
- **2025-10-22** Rails 8.1 Active Job Continuations; Solid Queue as Rails 8 default backend → [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues)
- **2025-11-11 → 2026-03** ingress-nginx retirement announced, then retired; migrate to Gateway API → [Nginx](/Software%20Engineering/Backend/Nginx) · [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies)
- **2026-02-17** Temporal $300M Series D: durable execution for long-running (agent) workflows → [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues)
- **2026-04-14** nginx 1.30.0 stable (Early Hints, HTTP/2 to upstreams, ECH, sticky sessions, MPTCP) → [Nginx](/Software%20Engineering/Backend/Nginx)
- **2026-06-23** Envoy AI Gateway 1.0, renamed Agent Router (MCP gateway) → [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies)
- **2026-07-28** MCP spec goes stateless (gateway-friendly routing headers) → [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies)
- **2026-09-01** Kong AI Gateway 2.0 GA → [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies)
- API and auth standards (OpenAPI 3.2, RFC 10008 QUERY, OAuth 2.1 draft, RFC 10017, WebAuthn L3) → see the list in [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index)
- Runtime changes → [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) · [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index)

## Related
- [Software Engineering](/Software%20Engineering/Software%20Engineering), the vault root
- [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index) · [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) · [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index) · [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index) · [Security Index](/Software%20Engineering/Security/Security%20Index)
- Frontend counterpart: [Frontend Home](/Software%20Engineering/Frontend/Frontend%20Home)
