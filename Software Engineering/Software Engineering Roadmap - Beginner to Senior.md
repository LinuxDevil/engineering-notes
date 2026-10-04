---
title: "Software Engineering Roadmap - Beginner to Senior"
tags: [index, roadmap, checklist]
aliases: [SE Roadmap, Beginner to Senior, Learning Path, Engineering Roadmap]
updated: 2026-09-27
---
# Software Engineering Roadmap - Beginner to Senior

<Note>
**TL;DR**

One path through the whole vault, in four stages. Each stage lists what to learn, the note that teaches it, and a **"you're ready to move on when…"** check. Tick an item when you can explain it out loud and write its example from memory. The stages roughly match the expectations in [Engineering Levels - Junior to Staff](/Software%20Engineering/Career/Engineering%20Levels%20-%20Junior%20to%20Staff): *Foundations* (pre-job), *Junior*, *Mid-level*, *Senior*.
</Note>

{/* diagram source: /diagrams/diagram-388.mmd */}
![diagram](/diagrams/diagram-388.svg)

<Info>
**How the levels differ**

The difference is not how many technologies you know. It is **scope and judgment**:
- A **junior** completes well-defined tasks with guidance.
- A **mid-level** engineer owns features end to end.
- A **senior** owns ambiguous problems, makes trade-offs explicit, prevents incidents, and makes the people around them better.

See [Engineering Levels - Junior to Staff](/Software%20Engineering/Career/Engineering%20Levels%20-%20Junior%20to%20Staff) and [Senior Engineer in the AI Era](/Software%20Engineering/Career/Senior%20Engineer%20in%20the%20AI%20Era).
</Info>

---

## Stage 1: Foundations
*Goal: write correct programs and understand what the computer and the network are doing.*

### Programming & CS
- [ ] A language deeply. JavaScript: [JavaScript Index](/Software%20Engineering/Frontend/01%20JavaScript/JavaScript%20Index). Java: [Java Language Essentials](/Software%20Engineering/Backend/Java/Java%20Language%20Essentials)
- [ ] [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis) → [Arrays & Strings](/Software%20Engineering/DSA/02%20Data%20Structures/Arrays%20%26%20Strings) → [Hash Tables](/Software%20Engineering/DSA/02%20Data%20Structures/Hash%20Tables) → [Linked Lists](/Software%20Engineering/DSA/02%20Data%20Structures/Linked%20Lists) → [Stacks & Queues](/Software%20Engineering/DSA/02%20Data%20Structures/Stacks%20%26%20Queues)
- [ ] [Recursion & Backtracking](/Software%20Engineering/DSA/03%20Algorithms/Recursion%20%26%20Backtracking) → [Trees & Binary Search Trees](/Software%20Engineering/DSA/02%20Data%20Structures/Trees%20%26%20Binary%20Search%20Trees) → [Sorting Algorithms](/Software%20Engineering/DSA/03%20Algorithms/Sorting%20Algorithms) → [Binary Search](/Software%20Engineering/DSA/04%20Patterns/Binary%20Search)
- [ ] [Problem Solving](/Software%20Engineering/DSA/01%20Foundations/Problem%20Solving) (the UMPIRE method for any coding problem)
- [ ] [How the Internet Works](/Software%20Engineering/CS%20Fundamentals/How%20the%20Internet%20Works) → [Encoding, Serialization & Character Sets](/Software%20Engineering/CS%20Fundamentals/Encoding%2C%20Serialization%20%26%20Character%20Sets)

### Tools
- [ ] [Linux & Shell Essentials](/Software%20Engineering/DevOps/Linux%20%26%20Shell%20Essentials)
- [ ] [Git Essentials & Workflows](/Software%20Engineering/DevOps/Git%20Essentials%20%26%20Workflows) (branch, commit, rebase, recover with reflog)

### Design basics
- [ ] [OOP](/Software%20Engineering/Principles/OOP) → [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface) → [DRY, KISS & YAGNI](/Software%20Engineering/Principles/DRY%2C%20KISS%20%26%20YAGNI) → [Clean Code & Refactoring](/Software%20Engineering/Principles/Clean%20Code%20%26%20Refactoring)

### Web basics
- [ ] [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path) → [CSS Fundamentals - Cascade, Box Model & Contexts](/Software%20Engineering/Frontend/04%20CSS/CSS%20Fundamentals%20-%20Cascade%2C%20Box%20Model%20%26%20Contexts) → [DOM, Events & Event Delegation](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/DOM%2C%20Events%20%26%20Event%20Delegation)
- [ ] [Introduction to SQL](/Software%20Engineering/Databases/SQL%20ANSI/Introduction%20to%20SQL) and the [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) course in order

<Check>
**Ready to move on when…**

You can build a small CRUD app on your own, use Git without fear, solve easy LeetCode problems, and explain what happens when you type a URL.
</Check>

---

## Stage 2: Junior (ship features)
*Goal: ship working, tested features in a real codebase.*

### Backend & APIs
- [ ] [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) → [HTTP Headers vs Body - Where Data Belongs](/Software%20Engineering/APIs%20%26%20Auth/HTTP%20Headers%20vs%20Body%20-%20Where%20Data%20Belongs) → [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization) → [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies)
- [ ] Node: [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index) (event loop, streams, [Building HTTP Servers - Express, Fastify & Hono](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Building%20HTTP%20Servers%20-%20Express%2C%20Fastify%20%26%20Hono)), **or** Java: [Spring Boot Essentials](/Software%20Engineering/Backend/Java/Spring%20Boot%20Essentials)

### Databases
- [ ] [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization) → [Database Indexes](/Software%20Engineering/Databases/Database%20Round/Indexes/Database%20Indexes) → [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) → [Keyset Pagination vs Offset](/Software%20Engineering/Databases/Database%20Round/Keyset%20Pagination%20vs%20Offset)
- [ ] The [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB) course, for the document model

### Frontend
- [ ] [React Index](/Software%20Engineering/Frontend/05%20React/React%20Index) (or [Angular Index](/Software%20Engineering/Frontend/08%20Angular/Angular%20Index)) → [State Management Overview](/Software%20Engineering/Frontend/06%20State%20Management/State%20Management%20Overview)

### Quality
- [ ] [Test-Driven Development (TDD)](/Software%20Engineering/Principles/Test-Driven%20Development%20(TDD)) → [Frontend Testing](/Software%20Engineering/Frontend/13%20Testing/Frontend%20Testing) → [Code Review Best Practices](/Software%20Engineering/Career/Code%20Review%20Best%20Practices) (how to receive a review)
- [ ] [SOLID](/Software%20Engineering/Principles/SOLID) → [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns) ([Creational Patterns](/Software%20Engineering/Principles/Creational%20Patterns), [Structural Patterns](/Software%20Engineering/Principles/Structural%20Patterns), [Behavioral Patterns](/Software%20Engineering/Principles/Behavioral%20Patterns))

### Shipping & DSA
- [ ] [Docker Basics](/Software%20Engineering/DevOps/Docker/Docker%20Basics) → [Docker Compose](/Software%20Engineering/DevOps/Docker/Docker%20Compose) → [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines)
- [ ] DSA patterns: [Two Pointers](/Software%20Engineering/DSA/04%20Patterns/Two%20Pointers), [Sliding Window](/Software%20Engineering/DSA/04%20Patterns/Sliding%20Window), [Prefix Sum](/Software%20Engineering/DSA/04%20Patterns/Prefix%20Sum), [Tree Traversal - DFS & BFS](/Software%20Engineering/DSA/03%20Algorithms/Tree%20Traversal%20-%20DFS%20%26%20BFS), [Graph BFS-DFS patterns](/Software%20Engineering/DSA/04%20Patterns/Graph%20BFS-DFS%20patterns)

### Working with AI
- [ ] [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development): use AI tools, but understand and test every line they produce

<Check>
**Ready to move on when…**

You take a ticket from vague to merged with little help, your PRs need few review rounds, and you write tests without being asked.
</Check>

---

## Stage 3: Mid-level (own services)
*Goal: own a service or feature area end to end, including how it behaves in production.*

### Deeper backend
- [ ] [Node.js Event Loop & libuv](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Event%20Loop%20%26%20libuv) / [Java Concurrency & Virtual Threads](/Software%20Engineering/Backend/Java/Java%20Concurrency%20%26%20Virtual%20Threads) → [Concurrency Primitives](/Software%20Engineering/CS%20Fundamentals/Concurrency%20Primitives) → [Operating Systems Essentials](/Software%20Engineering/CS%20Fundamentals/Operating%20Systems%20Essentials)
- [ ] [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility) → [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling) → [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) → [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets)
- [ ] Scheduled jobs, built end to end: [Scheduled Jobs Index](/Software%20Engineering/Backend/Scheduled%20Jobs/Scheduled%20Jobs%20Index). Covers cron + MQ, BullMQ, K8s CronJobs, a control API, a Next.js dashboard, and Playwright as a cron job
- [ ] Next.js in depth: [Next.js Index](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Index) (data and mutations, auth, route handlers/BFF, performance, testing, self-hosting)
- [ ] [OAuth 2.x & OpenID Connect](/Software%20Engineering/APIs%20%26%20Auth/OAuth%202.x%20%26%20OpenID%20Connect) → [Passkeys, WebAuthn & MFA](/Software%20Engineering/APIs%20%26%20Auth/Passkeys%2C%20WebAuthn%20%26%20MFA) → [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST) / [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST)

### Databases in depth
- [ ] [Isolation Levels & Read Phenomena](/Software%20Engineering/Databases/Core%20Concepts/Isolation%20Levels%20%26%20Read%20Phenomena) → [Locking & Concurrency Control](/Software%20Engineering/Databases/Core%20Concepts/Locking%20%26%20Concurrency%20Control) → [MVCC & Vacuum](/Software%20Engineering/Databases/Core%20Concepts/MVCC%20%26%20Vacuum)
- [ ] [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN) → [Database Migrations & Schema Evolution](/Software%20Engineering/Databases/Core%20Concepts/Database%20Migrations%20%26%20Schema%20Evolution) → [Connection Pooling](/Software%20Engineering/Databases/Database%20Round/Connection%20Pooling) → [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types)

### Architecture
- [ ] [Coupling, Cohesion & Modularity](/Software%20Engineering/Principles/Coupling%2C%20Cohesion%20%26%20Modularity) → [Dependency Injection & Inversion of Control](/Software%20Engineering/Principles/Dependency%20Injection%20%26%20Inversion%20of%20Control) → [Domain-Driven Design (DDD)](/Software%20Engineering/Principles/Domain-Driven%20Design%20(DDD)) → [Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles)

### Operations
- [ ] [Kubernetes Basics](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Basics) → [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production) → [Observability - Logs, Metrics & Traces](/Software%20Engineering/DevOps/Observability%20-%20Logs%2C%20Metrics%20%26%20Traces) → [Cloud Fundamentals](/Software%20Engineering/DevOps/Cloud%20Fundamentals) → [Infrastructure as Code](/Software%20Engineering/DevOps/Infrastructure%20as%20Code)
- [ ] Hands-on delivery: [Kubernetes Delivery Platform - End to End](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Delivery%20Platform%20-%20End%20to%20End). Covers [Jenkins - Pipelines as Code](/Software%20Engineering/DevOps/Delivery%20Platform/Jenkins%20-%20Pipelines%20as%20Code) → [Argo CD in Practice](/Software%20Engineering/DevOps/Delivery%20Platform/Argo%20CD%20in%20Practice) → [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) → [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling) → [Deploying Node.js Apps to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Node.js%20Apps%20to%20Kubernetes) → [Deploying Next.js Micro-frontends to Kubernetes](/Software%20Engineering/DevOps/Delivery%20Platform/Deploying%20Next.js%20Micro-frontends%20to%20Kubernetes)

### Security
- [ ] [OWASP Top 10](/Software%20Engineering/Security/OWASP%20Top%2010) → [Backend Security Essentials](/Software%20Engineering/Security/Backend%20Security%20Essentials) → [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) → [Secrets Management](/Software%20Engineering/Security/Secrets%20Management)

### Performance
- [ ] [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies) → [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals) → [Node.js Performance & Debugging](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Performance%20%26%20Debugging) / [JVM Internals - Memory & Garbage Collection](/Software%20Engineering/Backend/Java/JVM%20Internals%20-%20Memory%20%26%20Garbage%20Collection)

### DSA & habits
- [ ] DSA: [Heaps & Priority Queues](/Software%20Engineering/DSA/02%20Data%20Structures/Heaps%20%26%20Priority%20Queues), [Graphs](/Software%20Engineering/DSA/02%20Data%20Structures/Graphs), [Dynamic Programming](/Software%20Engineering/DSA/03%20Algorithms/Dynamic%20Programming), [Intervals](/Software%20Engineering/DSA/04%20Patterns/Intervals), [Monotonic Stack & Queue](/Software%20Engineering/DSA/04%20Patterns/Monotonic%20Stack%20%26%20Queue)
- [ ] [Debugging Methodology](/Software%20Engineering/Career/Debugging%20Methodology) → [Estimation & Planning](/Software%20Engineering/Career/Estimation%20%26%20Planning) → [Technical Design Docs & RFCs](/Software%20Engineering/Career/Technical%20Design%20Docs%20%26%20RFCs)

### LLM features
- [ ] [Transformers & How LLMs Work](/Software%20Engineering/AI%20%26%20ML/Transformers%20%26%20How%20LLMs%20Work) → [Prompt & Context Engineering](/Software%20Engineering/AI%20%26%20ML/Prompt%20%26%20Context%20Engineering) → [Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG)) → [MCP](/Software%20Engineering/AI%20%26%20ML/MCP)

<Check>
**Ready to move on when…**

You are on call for your service and can debug it from logs, metrics and traces. You write a design doc others approve. You have led a feature spanning several people.
</Check>

---

## Stage 4: Senior (own systems & outcomes)
*Goal: make good decisions under ambiguity across many systems, and multiply the team.*

### System design
- [ ] [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index): [System Design Interview Framework](/Software%20Engineering/System%20Design/System%20Design%20Interview%20Framework) → [Scalability Fundamentals](/Software%20Engineering/System%20Design/Scalability%20Fundamentals) → [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) → [Consistency Models & Consensus](/Software%20Engineering/System%20Design/Consistency%20Models%20%26%20Consensus) → [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns)
- [ ] Data at scale: [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC) → [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning) → [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem) → [Consistent Hashing](/Software%20Engineering/System%20Design/Consistent%20Hashing) → [Row Vs Column Oriented Databases](/Software%20Engineering/Databases/Database%20Round/Row%20Vs%20Column%20Oriented%20Databases)
- [ ] Case studies: [Case Study - URL Shortener](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20URL%20Shortener), [Case Study - Rate Limiter](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Rate%20Limiter), [Case Study - Chat System](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Chat%20System), [Case Study - News Feed Backend](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20News%20Feed%20Backend), [Case Study - Notification System](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Notification%20System), [Case Study - Video Streaming](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Video%20Streaming), [Case Study - Distributed Cache](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Distributed%20Cache)
- [ ] Frontend system design: [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index) → [RADIO Framework](/Software%20Engineering/Frontend/15%20System%20Design/RADIO%20Framework)

### Architecture judgment
- [ ] [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture) (and when **not** to use it)
- [ ] [Architecture Decision Records (ADRs) & Technical Debt](/Software%20Engineering/Principles/Architecture%20Decision%20Records%20(ADRs)%20%26%20Technical%20Debt) → [Working With Legacy Code](/Software%20Engineering/Career/Working%20With%20Legacy%20Code) → [Micro-frontends & Module Federation](/Software%20Engineering/Frontend/09%20Architecture/Micro-frontends%20%26%20Module%20Federation)

### Reliability
- [ ] [Site Reliability Engineering & Incident Management](/Software%20Engineering/DevOps/Site%20Reliability%20Engineering%20%26%20Incident%20Management) → [Platform Engineering & Internal Developer Platforms](/Software%20Engineering/DevOps/Platform%20Engineering%20%26%20Internal%20Developer%20Platforms) → [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux)

### Security leadership
- [ ] [Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling) → [API Security](/Software%20Engineering/Security/API%20Security) → [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security) → [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers)

### AI systems
- [ ] [Designing LLM-Powered Systems](/Software%20Engineering/System%20Design/Designing%20LLM-Powered%20Systems) → [LLM Evaluation & Observability](/Software%20Engineering/AI%20%26%20ML/LLM%20Evaluation%20%26%20Observability) → [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps) → [AI Agents & Agentic Patterns](/Software%20Engineering/AI%20%26%20ML/AI%20Agents%20%26%20Agentic%20Patterns)

### People & influence
- [ ] [Communication & Influence for Engineers](/Software%20Engineering/Career/Communication%20%26%20Influence%20for%20Engineers) → [Mentoring & Leading Without Authority](/Software%20Engineering/Career/Mentoring%20%26%20Leading%20Without%20Authority) → [Productivity & Deep Work](/Software%20Engineering/Career/Productivity%20%26%20Deep%20Work) → [Senior Engineer in the AI Era](/Software%20Engineering/Career/Senior%20Engineer%20in%20the%20AI%20Era)

<Check>
**You are operating at senior level when…**

- People seek your review on risky changes.
- Your designs anticipate failure modes.
- You say "no" or "not yet" with good reasons.
- Incidents in your area get rarer.
- Juniors you mentor get promoted.
</Check>

---

## Interview track (any stage)
[Interviews Index](/Software%20Engineering/Interviews/Interviews%20Index) → [Interview Process & Preparation Plan](/Software%20Engineering/Interviews/Interview%20Process%20%26%20Preparation%20Plan) → [DSA Index](/Software%20Engineering/DSA/DSA%20Index) → [System Design Interview Framework](/Software%20Engineering/System%20Design/System%20Design%20Interview%20Framework) → [Behavioral Interviews & STAR Method](/Software%20Engineering/Interviews/Behavioral%20Interviews%20%26%20STAR%20Method) → [Backend Interview Questions](/Software%20Engineering/Interviews/Backend%20Interview%20Questions) / [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions) / [React Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/React%20Interview%20Questions) → [Salary Negotiation & Offers](/Software%20Engineering/Interviews/Salary%20Negotiation%20%26%20Offers)

## Staying current
Read [What's New in Software Engineering 2025-2026](/Software%20Engineering/What's%20New%20in%20Software%20Engineering%202025-2026) once per quarter and update the 🆕 boxes in the notes you rely on.

## Related
- [Software Engineering](/Software%20Engineering/Software%20Engineering) (vault home)
- [Frontend Roadmap & Checklist](/Software%20Engineering/Frontend/Frontend%20Roadmap%20%26%20Checklist) (the detailed frontend checklist)
- [Java Backend Preparation Checklist](/Software%20Engineering/Backend/Java/Java%20Backend%20Preparation%20Checklist)

## Sources
- [roadmap.sh: Backend, Frontend, DevOps and System Design roadmaps](https://roadmap.sh/)
- Tanya Reilly, *The Staff Engineer's Path* (O'Reilly, 2022)
- Will Larson, [*Staff Engineer: Leadership beyond the management track*](https://staffeng.com/book)
- [Google Engineering Practices](https://google.github.io/eng-practices/)
- [DORA: 2025 State of AI-assisted Software Development](https://dora.dev/research/2025/dora-report/)
