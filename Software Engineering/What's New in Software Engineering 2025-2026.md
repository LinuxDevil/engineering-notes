---
title: "What's New in Software Engineering 2025-2026"
tags: [index, whats-new]
aliases: [What's New 2025-2026, SE Changelog, 2025-2026 Changes]
updated: 2026-09-27
---
# What's New in Software Engineering 2025-2026

<Note>
**TL;DR**

This note is a vault-wide digest of what changed from January 2025 to September 2026. Each item has a date and links to the note that explains it; those notes cite the primary sources. Frontend has its own, more detailed digest: [What's New in Frontend 2025-2026](/Software%20Engineering/Frontend/What's%20New%20in%20Frontend%202025-2026). DevOps has one too: [What's New in DevOps 2025-2026](/Software%20Engineering/DevOps/What's%20New%20in%20DevOps%202025-2026).

**The seven shifts that matter most for a senior engineer:**
1. **AI coding agents became normal tooling.** Claude Code, Copilot coding agent, Codex and Cursor all shipped in this period, along with AGENTS.md and spec-driven development. The research is mixed: DORA 2025 calls AI an *amplifier* of existing practices, and METR found experienced developers were 19% *slower* while believing they were faster. Judgment and review skills matter more than ever. See [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development) and [Senior Engineer in the AI Era](/Software%20Engineering/Career/Senior%20Engineer%20in%20the%20AI%20Era).
2. **MCP became the standard way to connect tools to models.** It had four spec revisions and moved to the Linux Foundation's Agentic AI Foundation. In July 2026 it became stateless. See [MCP](/Software%20Engineering/AI%20%26%20ML/MCP).
3. **Supply chain security got real.** Examples: the Shai-Hulud npm worm, the tj-actions compromise, and OWASP Top 10:2025 putting *Software Supply Chain Failures* at #3. The EU CRA reporting duties started in Sept 2026. See [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security) and [Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling).
4. **New LTS runtimes.** Java 25 LTS, Node 24 LTS and Spring Boot 4 / Spring Framework 7 all shipped. Virtual threads no longer pin, and Node strips TypeScript types natively. See [Modern Java 17 to 25](/Software%20Engineering/Backend/Java/Modern%20Java%2017%20to%2025), [Node.js 22 to 24+ - What's New](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%2022%20to%2024%2B%20-%20What's%20New) and [Spring Boot 3 to 4 - What's New](/Software%20Engineering/Backend/Java/Spring%20Boot%203%20to%204%20-%20What's%20New).
5. **Kubernetes networking changed.** ingress-nginx was retired in March 2026, and Gateway API is the path forward. See [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API).
6. **Post-quantum TLS became the default.** Hybrid ML-KEM key exchange is on by default in browsers and OpenSSL 3.5. TLS certificate lifetimes start shrinking toward 47 days. See [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers).
7. **The data layer moved.** PostgreSQL 18 added async I/O and `uuidv7()`. Kafka 4 dropped ZooKeeper and added queues. Redis 8 returned to an open-source (AGPL) licence, and Valkey 9 shipped. See [PostgreSQL 17 to 18 - What's New](/Software%20Engineering/Databases/Core%20Concepts/PostgreSQL%2017%20to%2018%20-%20What's%20New) and [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming).
</Note>

## AI & ML
| When | What | Note |
|---|---|---|
| 2025-01-20 | DeepSeek-R1: open-weight reasoning model (MIT licence) | [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026) |
| 2025-02-24 → 05-22 | Claude Code research preview, then GA | [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development) |
| 2025-03-26 | MCP spec: Streamable HTTP, OAuth 2.1 authorization | [MCP](/Software%20Engineering/AI%20%26%20ML/MCP) |
| 2025-06 | "Context engineering" becomes the term of art | [Prompt & Context Engineering](/Software%20Engineering/AI%20%26%20ML/Prompt%20%26%20Context%20Engineering) |
| 2025-06-18 | MCP spec: structured tool output, elicitation | [MCP](/Software%20Engineering/AI%20%26%20ML/MCP) |
| 2025-07-10 | METR study: experienced developers 19% slower with AI tools | [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development) |
| 2025-08 | AGENTS.md convention; GPT-5 (08-07) | [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development), [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026) |
| 2025-09-02 | GitHub Spec Kit (spec-driven development) | [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development) |
| 2025-10-16 | Agent Skills | [AI Agents & Agentic Patterns](/Software%20Engineering/AI%20%26%20ML/AI%20Agents%20%26%20Agentic%20Patterns) |
| 2025-11-18 | Gemini 3 | [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026) |
| 2025-11-25 | MCP spec: tasks, URL elicitation | [MCP](/Software%20Engineering/AI%20%26%20ML/MCP) |
| 2025-12-09 | Agentic AI Foundation (MCP, AGENTS.md) under the Linux Foundation; OWASP Top 10 for Agentic Applications | [MCP](/Software%20Engineering/AI%20%26%20ML/MCP), [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps) |
| 2026-03-12 | A2A protocol v1.0 | [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026) |
| 2026-07-28 | MCP spec becomes stateless; TypeScript SDK v2 | [MCP](/Software%20Engineering/AI%20%26%20ML/MCP) |

## Backend runtimes & frameworks
| When | What | Note |
|---|---|---|
| 2025-03-18 | JDK 24: Stream Gatherers; virtual threads stop pinning in `synchronized` (JEP 491) | [Java Concurrency & Virtual Threads](/Software%20Engineering/Backend/Java/Java%20Concurrency%20%26%20Virtual%20Threads) |
| 2025-03-31 | Express 5.1 becomes the npm default | [Building HTTP Servers - Express, Fastify & Hono](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Building%20HTTP%20Servers%20-%20Express%2C%20Fastify%20%26%20Hono) |
| 2025-05-19 | Hibernate ORM 7.0 (JPA 3.2, Apache 2.0) | [JPA, Hibernate & jOOQ - Data Access in Java](/Software%20Engineering/Backend/Java/JPA%2C%20Hibernate%20%26%20jOOQ%20-%20Data%20Access%20in%20Java) |
| 2025-09-16 | **JDK 25 LTS**: scoped values, compact source files, flexible constructors | [Modern Java 17 to 25](/Software%20Engineering/Backend/Java/Modern%20Java%2017%20to%2025), [OOP](/Software%20Engineering/Principles/OOP) |
| 2025-03-21 | Next.js middleware auth bypass CVE-2025-29927: check auth in the data layer, not only in middleware/proxy | [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization) |
| 2025-09-22 | Auth.js joins Better Auth and goes into maintenance mode | [Next.js Authentication & Authorization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Authentication%20%26%20Authorization) |
| 2025-11-20 | BullMQ 5.64: built-in job cancellation (`cancelJob`, AbortSignal) | [BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice) |
| 2026-03-18 | Next.js 16.2: Deployment Adapters API stable | [Next.js Self-Hosting & Deployment Options](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Self-Hosting%20%26%20Deployment%20Options) |
| 2026-03-19 | GitHub Actions: `timezone` for scheduled workflows | [Playwright Automation as a Cron Job](/Software%20Engineering/Backend/Scheduled%20Jobs/Playwright%20Automation%20as%20a%20Cron%20Job) |
| 2026-07-30 | BullMQ 6.0: PostgreSQL backend; repeatable jobs removed (use Job Schedulers) | [BullMQ Job Schedulers in Practice](/Software%20Engineering/Backend/Scheduled%20Jobs/BullMQ%20Job%20Schedulers%20in%20Practice) |
| 2026-08-03 | Next.js 16.3: Instant Navigations, `catchError`/`retry()` stable, `@next/playwright` | [Next.js Performance Optimization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Performance%20Optimization), [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found) |
| 2026-09 | RabbitMQ delayed-message-exchange plugin archived | [Cron with Message Queues - Architecture](/Software%20Engineering/Backend/Scheduled%20Jobs/Cron%20with%20Message%20Queues%20-%20Architecture) |
| 2025-10-22 | Rails 8.1 (Active Job Continuations) | [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues), [Rails Project Structure](/Software%20Engineering/DevOps/Rails%20Project%20Structure) |
| 2025-10-28 | **Node 24 LTS** | [Node.js 22 to 24+ - What's New](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%2022%20to%2024%2B%20-%20What's%20New) |
| 2025-11-11 | Node TypeScript type stripping stable (25.2 / 24.12) | [Node.js 22 to 24+ - What's New](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%2022%20to%2024%2B%20-%20What's%20New) |
| 2025-11-13 / 11-20 | **Spring Framework 7** (Jakarta EE 11, JSpecify, API versioning) / **Spring Boot 4** | [Spring Boot 3 to 4 - What's New](/Software%20Engineering/Backend/Java/Spring%20Boot%203%20to%204%20-%20What's%20New) |
| 2025-11-21 | Spring Modulith 2.0: modular monoliths go mainstream | [Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles) |
| 2026-01-19 | `require(esm)` stable in Node | [Node.js 22 to 24+ - What's New](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%2022%20to%2024%2B%20-%20What's%20New) |
| 2026-03-10 | Node moves to one major release per year from v27, every release LTS | [Node.js 22 to 24+ - What's New](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%2022%20to%2024%2B%20-%20What's%20New) |
| 2026-03-17 | JDK 26: HTTP/3 client, "final means final" (JEP 500) | [Modern Java 17 to 25](/Software%20Engineering/Backend/Java/Modern%20Java%2017%20to%2025), [Immutability Across Languages](/Software%20Engineering/Principles/Immutability%20Across%20Languages) |
| 2026-05-05 | Node 26: Temporal on by default | [Node.js 22 to 24+ - What's New](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%2022%20to%2024%2B%20-%20What's%20New) |
| 2026-09-15 | JDK 27: compact object headers on by default | [JVM Internals - Memory & Garbage Collection](/Software%20Engineering/Backend/Java/JVM%20Internals%20-%20Memory%20%26%20Garbage%20Collection) |

## APIs, auth & protocols
| When | What | Note |
|---|---|---|
| 2025-01 | RFC 9700: OAuth 2.0 Security Best Current Practice | [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) |
| 2025-03 | RFC 9745: `Deprecation` HTTP header | [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility) |
| 2025-07/08 | NIST SP 800-63-4 final: synced passkeys accepted, no password composition rules | [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA), [OWASP Top 10](/Software%20Engineering/Security/OWASP%20Top%2010) |
| 2025-09-08 | GraphQL spec, September 2025 edition (OneOf inputs) | [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST) |
| 2025-09-19 | OpenAPI 3.2.0 | [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) |
| 2026-03-02 | OAuth 2.1 still a draft (-15) | [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) |
| 2026-06 | RFC 10008: HTTP `QUERY` method | [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) |
| 2026-08 | RFC 10017: OAuth for browser-based apps (recommends BFF) | [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies) |
| 2026-08-25 | WebAuthn Level 3 becomes a W3C Recommendation | [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA) |
| 2026 | AI gateways: Envoy AI Gateway 1.0 (06), Kong AI Gateway 2.0 (09) | [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) |

## Data, messaging & systems
| When | What | Note |
|---|---|---|
| 2025-03-18 | **Kafka 4.0**: ZooKeeper removed (KRaft only) | [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) |
| 2025-05-01 | Redis 8.0: AGPLv3 option, vector sets | [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies) |
| 2025-07-21 | Amazon SQS fair queues | [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) |
| 2025-09-25 | **PostgreSQL 18**: async I/O, `uuidv7()`, virtual generated columns | [PostgreSQL 17 to 18 - What's New](/Software%20Engineering/Databases/Core%20Concepts/PostgreSQL%2017%20to%2018%20-%20What's%20New), [Unique ID Generation](/Software%20Engineering/System%20Design/Unique%20ID%20Generation) |
| 2025-09 | DuckDB 1.4, its first LTS; MongoDB 8.2 (09-16) brings self-managed Search and Vector Search (preview) | [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types), [Unit 13 MongoDB Atlas Search](/Software%20Engineering/Databases/MongoDB/Unit%2013%20MongoDB%20Atlas%20Search) |
| 2025-05 / 06 | Databricks buys Neon; Snowflake buys Crunchy Data (serverless Postgres consolidates) | [PostgreSQL 17 to 18 - What's New](/Software%20Engineering/Databases/Core%20Concepts/PostgreSQL%2017%20to%2018%20-%20What's%20New) |
| 2026-02-26 | pgvector 0.8.2 fixes CVE-2026-3172 | [Vector Databases & pgvector](/Software%20Engineering/Databases/Core%20Concepts/Vector%20Databases%20%26%20pgvector) |
| 2026-04-30 | MySQL 8.0 end of life (9.7 LTS released 04-21) | [Database Migrations & Schema Evolution](/Software%20Engineering/Databases/Core%20Concepts/Database%20Migrations%20%26%20Schema%20Evolution) |
| 2026-09-24 | PostgreSQL 19 Beta 4 (`REPACK CONCURRENTLY`, parallel autovacuum); GA expected around October 2026 | [PostgreSQL 17 to 18 - What's New](/Software%20Engineering/Databases/Core%20Concepts/PostgreSQL%2017%20to%2018%20-%20What's%20New) |
| 2025-10-20 | AWS us-east-1 outage (DynamoDB DNS automation race) | [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns), [Site Reliability Engineering & Incident Management](/Software%20Engineering/DevOps/Site%20Reliability%20Engineering%20%26%20Incident%20Management) |
| 2025-10-21 | Valkey 9.0 | [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies) |
| 2025-11-18, 12-05 | Cloudflare global outages lead to its "Fail Small" plan | [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns) |
| 2025-12-02 | S3: 50 TB objects; S3 Vectors GA | [Blob Storage & File Uploads](/Software%20Engineering/System%20Design/Blob%20Storage%20%26%20File%20Uploads) |
| 2026-02-17 | Kafka 4.2: share groups ("queues for Kafka") production-ready | [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) |

## DevOps & cloud
| When | What | Note |
|---|---|---|
| 2025-02-27 | IBM completes the HashiCorp acquisition | [Infrastructure as Code](/Software%20Engineering/DevOps/Infrastructure%20as%20Code) |
| 2025-03-14 | tj-actions/changed-files compromise (CVE-2025-30066) | [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines) |
| 2025-04-23 | Kubernetes 1.33: native sidecars GA | [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production) |
| 2025-08-15 | GitHub Actions can enforce SHA pinning | [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines) |
| 2025-11-10 | Docker Engine 29: containerd image store by default | [Docker Basics](/Software%20Engineering/DevOps/Docker/Docker%20Basics) |
| 2025-11-12 | Helm 4.0 | [Helm & Kustomize](/Software%20Engineering/DevOps/Kubernetes/Helm%20%26%20Kustomize) |
| 2025-12-17 | Kubernetes 1.35: in-place Pod resize GA; Docker Hardened Images free | [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production), [Dockerfile Best Practices & Image Security](/Software%20Engineering/DevOps/Docker/Dockerfile%20Best%20Practices%20%26%20Image%20Security) |
| 2026-01-05 | DORA: five metrics, adding deployment rework rate | [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines) |
| 2025-05-06 → 2026-08-04 | Argo CD 3.0 → 3.5 (annotation tracking, OCI sources, Helm 4, Git commit signature checks); Image Updater 1.0 | [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice) |
| 2025-06-03 | Google archives Kaniko (Chainguard maintains a fork) | [Jenkins - Pipelines as Code](/Software%20Engineering/DevOps/Delivery%20Platform/Jenkins%20-%20Pipelines%20as%20Code) |
| 2025-11-01 / 2026-03-02 | Grafana Agent and Promtail reach end of life; Alloy is the collector | [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) |
| 2026-03-18 | Next.js 16.2: skew detection based on `deploymentId` | [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes) |
| 2026-04-15 | Jenkins LTS requires Java 21+ | [Jenkins - Pipelines as Code](/Software%20Engineering/DevOps/Delivery%20Platform/Jenkins%20-%20Pipelines%20as%20Code) |
| 2026-04-21 / 06-01 | Grafana 13 (Git Sync GA); Tempo 3.0 | [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) |
| 2026-08-21 | `prom-client` moves to `@prometheus-io/client` | [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes) |
| 2026-03-24 | **ingress-nginx retired**; move to Gateway API | [Kubernetes Networking - Services, Ingress & Gateway API](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Networking%20-%20Services%2C%20Ingress%20%26%20Gateway%20API) |
| 2026-04-22 / 08-26 | Kubernetes 1.36 (user namespaces GA) / 1.37 (HPA scale-to-zero beta) | [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production) |

## Security & cryptography
| When | What | Note |
|---|---|---|
| 2025-04 | CA/B Forum: TLS certificate lifetime drops to 200 days (2026-03-15) and 47 days (2029) | [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers) |
| 2025-04-08 | OpenSSL 3.5 LTS: ML-KEM; hybrid post-quantum key share by default | [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers) |
| 2025-05 | OWASP ASVS 5.0 | [OWASP Top 10](/Software%20Engineering/Security/OWASP%20Top%2010) |
| 2025-09 | npm chalk/debug compromise and the Shai-Hulud worm | [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security), [Node.js Security Best Practices](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Security%20Best%20Practices) |
| 2025-11 | **OWASP Top 10:2025** (Supply Chain Failures #3, Mishandling of Exceptional Conditions #10) | [OWASP Top 10](/Software%20Engineering/Security/OWASP%20Top%2010) |
| 2025-11-12 | SLSA v1.2 (Source Track) | [Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling) |
| 2026-01-15 | Let's Encrypt: 6-day and IP-address certificates GA | [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers) |
| 2026-09-11 | EU Cyber Resilience Act reporting duties apply (full application 2027-12-11) | [Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling) |

## Principles, careers & interviews
| When | What | Note |
|---|---|---|
| 2025-02 | GitClear: sharp rise in duplicated code attributed to AI assistants | [DRY, KISS & YAGNI](/Software%20Engineering/Principles/DRY%2C%20KISS%20%26%20YAGNI) |
| 2025-06-25 | Kent Beck, "Augmented Coding": TDD as the guardrail for AI agents | [Test-Driven Development (TDD)](/Software%20Engineering/Principles/Test-Driven%20Development%20(TDD)) |
| 2025-06 → 10 | Canva, then Meta, introduce AI-assisted coding interview rounds; Google brings back in-person rounds | [Problem Solving](/Software%20Engineering/DSA/01%20Foundations/Problem%20Solving), [Interview Process & Preparation Plan](/Software%20Engineering/Interviews/Interview%20Process%20%26%20Preparation%20Plan) |
| 2025 | Stack Overflow survey: 84% use AI, 46% distrust its accuracy | [Senior Engineer in the AI Era](/Software%20Engineering/Career/Senior%20Engineer%20in%20the%20AI%20Era) |
| 2025-09-24 | DORA 2025: AI as an amplifier | [Senior Engineer in the AI Era](/Software%20Engineering/Career/Senior%20Engineer%20in%20the%20AI%20Era) |
| 2025-10 | *Clean Code*, 2nd edition | [Clean Code & Refactoring](/Software%20Engineering/Principles/Clean%20Code%20%26%20Refactoring) |
| 2025-11 | Thoughtworks Radar puts spec-driven development in "Assess" | [Behavior-Driven Development (BDD)](/Software%20Engineering/Principles/Behavior-Driven%20Development%20(BDD)) |

<Warning>
**Freshness**

This digest was compiled on 2026-09-27. Fast-moving areas (models, K8s, Node, Java) go stale within months. Each linked note's **Sources** section points to the official page to re-check.
</Warning>

## Related
- [Software Engineering](/Software%20Engineering/Software%20Engineering) · [Software Engineering Roadmap - Beginner to Senior](/Software%20Engineering/Software%20Engineering%20Roadmap%20-%20Beginner%20to%20Senior)
- [What's New in Frontend 2025-2026](/Software%20Engineering/Frontend/What's%20New%20in%20Frontend%202025-2026) · [What's New in DevOps 2025-2026](/Software%20Engineering/DevOps/What's%20New%20in%20DevOps%202025-2026) · [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026)
