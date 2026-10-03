---
tags: [backend, backend/java, checklist]
aliases: [Preparation, Java Preparation, Java Backend Checklist, Java Interview Prep]
updated: 2026-09-27
---
# Java Backend Preparation Checklist

> [!tldr] TL;DR
> The original preparation list for a Java backend role (Spring Boot, Reactor, PostgreSQL + jOOQ, OOP, system design, distributed systems, messaging, SOLID, problem solving), turned into a tickable checklist where every item links to the notes that cover it. Renamed from "Preparation" on 2026-09-27. Work top to bottom; the Java core block was added so the list covers beginner → senior.

## Java core (added)
- [ ] Language: types, collections, generics, exceptions, streams, records, sealed types → [[Java Language Essentials]]
- [ ] What changed in Java 17 → 25 (and 26/27) → [[Modern Java 17 to 25]]
- [ ] Concurrency, executors, `CompletableFuture`, virtual threads → [[Java Concurrency & Virtual Threads]]
- [ ] JVM memory, GC, JIT, tuning and diagnostics → [[JVM Internals - Memory & Garbage Collection]]

## The original list
- [ ] **Integration**: how services talk and how to test them together
  - [[Microservices Architecture]] · [[REST API Design]] · [[Message Queues & Event Streaming]] · [[Webhooks, Polling, SSE & WebSockets]]
  - Integration tests with Testcontainers → [[Spring Boot Essentials]] (Testing section)
- [ ] **SpringBoot**
  - [[Spring Boot Essentials]] (DI/IoC, auto-config, profiles, JPA, transactions, testing)
  - [[Spring Boot 3 to 4 - What's New]]
  - Book background: [[Reactive Spring - Ch 3 Spring Fundamentals]]
- [ ] **Reactive Programming (Reactor)**
  - [[Reactor - Mono & Flux]] · [[Reactive Spring - Ch 4 IO & Reactive Streams]] · [[Reactive Spring - Ch 5 Reactor & Backpressure]]
  - When *not* to use it: [[Java Concurrency & Virtual Threads]] (virtual threads vs reactive)
- [ ] [[Reactive Spring Book]]
- [ ] [[Java Reactive Course]]
- [ ] **PostgreSQL**
  - [[Databases Index]] · [[Transactions & ACID]] · [[Isolation Levels & Read Phenomena]] · [[Database Indexes]] · [[Query Optimization & EXPLAIN]] · [[MVCC & Vacuum]] · [[PostgreSQL 17 to 18 - What's New]]
  - Connection pools from Java: [[Connection Pooling]]
- [ ] **jOOQ** → [[JPA, Hibernate & jOOQ - Data Access in Java]] (absorbed the old "jOOQ (java Object Oriented Querying)" note)
- [ ] **OOP** → [[OOP]] · [[Abstract Class and Interface]] · [[Design Patterns]] · [[Reactive Spring - Ch 2 Java Refresher]]
- [ ] **System Design** → [[System Design Index]] · [[System Design Interview Framework]] · [[Scalability Fundamentals]] · [[Caching Strategies]]
- [ ] **Distributed Systems** → [[Distributed Systems Patterns]] · [[Consistency Models & Consensus]] · [[CAP Theorem]] · [[Replication & 2PC 3PC]]
- [ ] **Message Brokers** → [[Message Queues & Event Streaming]]
- [ ] **High availability and stability via pub-sub** → [[Message Queues & Event Streaming]] · [[Distributed Systems Patterns]] · [[Case Study - Notification System]]
- [ ] **SOLID & Code Principles** → [[SOLID]] · [[DRY, KISS & YAGNI]] · [[Principles Index]]
- [ ] [[Problem Solving]] → and the rest of [[DSA Index]]

## Suggested order
> [!example]- Four-week plan
> 1. Week 1: Java core block + [[OOP]] + [[SOLID]] + daily [[Problem Solving]]
> 2. Week 2: [[Spring Boot Essentials]] → [[JPA, Hibernate & jOOQ - Data Access in Java]] → PostgreSQL notes
> 3. Week 3: [[Java Concurrency & Virtual Threads]] → [[Reactor - Mono & Flux]] → [[Message Queues & Event Streaming]]
> 4. Week 4: [[System Design Index]] + [[Distributed Systems Patterns]] + mock interviews; skim [[Spring Boot 3 to 4 - What's New]] and [[Modern Java 17 to 25]] for "what's new" questions

## Related
- [[Java & Spring Index]] · [[Backend Index]] · [[Interviews Index]]
