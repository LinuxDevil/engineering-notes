---
title: "Java Backend Preparation Checklist"
tags: [backend, backend/java, checklist]
aliases: [Preparation, Java Preparation, Java Backend Checklist, Java Interview Prep]
updated: 2026-09-27
---
# Java Backend Preparation Checklist

<Note>
**TL;DR**

The original preparation list for a Java backend role (Spring Boot, Reactor, PostgreSQL + jOOQ, OOP, system design, distributed systems, messaging, SOLID, problem solving), turned into a tickable checklist where every item links to the notes that cover it. Renamed from "Preparation" on 2026-09-27. Work top to bottom; the Java core block was added so the list covers beginner → senior.
</Note>

## Java core (added)
- [ ] Language: types, collections, generics, exceptions, streams, records, sealed types → [Java Language Essentials](/Software%20Engineering/Backend/Java/Java%20Language%20Essentials)
- [ ] What changed in Java 17 → 25 (and 26/27) → [Modern Java 17 to 25](/Software%20Engineering/Backend/Java/Modern%20Java%2017%20to%2025)
- [ ] Concurrency, executors, `CompletableFuture`, virtual threads → [Java Concurrency & Virtual Threads](/Software%20Engineering/Backend/Java/Java%20Concurrency%20%26%20Virtual%20Threads)
- [ ] JVM memory, GC, JIT, tuning and diagnostics → [JVM Internals - Memory & Garbage Collection](/Software%20Engineering/Backend/Java/JVM%20Internals%20-%20Memory%20%26%20Garbage%20Collection)

## The original list
- [ ] **Integration**: how services talk and how to test them together
  - [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture) · [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) · [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) · [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets)
  - Integration tests with Testcontainers → [Spring Boot Essentials](/Software%20Engineering/Backend/Java/Spring%20Boot%20Essentials) (Testing section)
- [ ] **SpringBoot**
  - [Spring Boot Essentials](/Software%20Engineering/Backend/Java/Spring%20Boot%20Essentials) (DI/IoC, auto-config, profiles, JPA, transactions, testing)
  - [Spring Boot 3 to 4 - What's New](/Software%20Engineering/Backend/Java/Spring%20Boot%203%20to%204%20-%20What's%20New)
  - Book background: [Reactive Spring - Ch 3 Spring Fundamentals](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%203%20Spring%20Fundamentals)
- [ ] **Reactive Programming (Reactor)**
  - [Reactor - Mono & Flux](/Software%20Engineering/Backend/Java/Java%20Reactive%20Course/Reactor%20-%20Mono%20%26%20Flux) · [Reactive Spring - Ch 4 IO & Reactive Streams](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%204%20IO%20%26%20Reactive%20Streams) · [Reactive Spring - Ch 5 Reactor & Backpressure](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%205%20Reactor%20%26%20Backpressure)
  - When *not* to use it: [Java Concurrency & Virtual Threads](/Software%20Engineering/Backend/Java/Java%20Concurrency%20%26%20Virtual%20Threads) (virtual threads vs reactive)
- [ ] [Reactive Spring Book](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20Book)
- [ ] [Java Reactive Course](/Software%20Engineering/Backend/Java/Java%20Reactive%20Course/Java%20Reactive%20Course)
- [ ] **PostgreSQL**
  - [Databases Index](/Software%20Engineering/Databases/Databases%20Index) · [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Isolation Levels & Read Phenomena](/Software%20Engineering/Databases/Core%20Concepts/Isolation%20Levels%20%26%20Read%20Phenomena) · [Database Indexes](/Software%20Engineering/Databases/Database%20Round/Indexes/Database%20Indexes) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN) · [MVCC & Vacuum](/Software%20Engineering/Databases/Core%20Concepts/MVCC%20%26%20Vacuum) · [PostgreSQL 17 to 18 - What's New](/Software%20Engineering/Databases/Core%20Concepts/PostgreSQL%2017%20to%2018%20-%20What's%20New)
  - Connection pools from Java: [Connection Pooling](/Software%20Engineering/Databases/Database%20Round/Connection%20Pooling)
- [ ] **jOOQ** → [JPA, Hibernate & jOOQ - Data Access in Java](/Software%20Engineering/Backend/Java/JPA%2C%20Hibernate%20%26%20jOOQ%20-%20Data%20Access%20in%20Java) (absorbed the old "jOOQ (java Object Oriented Querying)" note)
- [ ] **OOP** → [OOP](/Software%20Engineering/Principles/OOP) · [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface) · [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns) · [Reactive Spring - Ch 2 Java Refresher](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%202%20Java%20Refresher)
- [ ] **System Design** → [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index) · [System Design Interview Framework](/Software%20Engineering/System%20Design/System%20Design%20Interview%20Framework) · [Scalability Fundamentals](/Software%20Engineering/System%20Design/Scalability%20Fundamentals) · [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies)
- [ ] **Distributed Systems** → [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns) · [Consistency Models & Consensus](/Software%20Engineering/System%20Design/Consistency%20Models%20%26%20Consensus) · [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem) · [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC)
- [ ] **Message Brokers** → [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming)
- [ ] **High availability and stability via pub-sub** → [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) · [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns) · [Case Study - Notification System](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Notification%20System)
- [ ] **SOLID & Code Principles** → [SOLID](/Software%20Engineering/Principles/SOLID) · [DRY, KISS & YAGNI](/Software%20Engineering/Principles/DRY%2C%20KISS%20%26%20YAGNI) · [Principles Index](/Software%20Engineering/Principles/Principles%20Index)
- [ ] [Problem Solving](/Software%20Engineering/DSA/01%20Foundations/Problem%20Solving) → and the rest of [DSA Index](/Software%20Engineering/DSA/DSA%20Index)

## Suggested order
<Info>
**Four-week plan**

1. Week 1: Java core block + [OOP](/Software%20Engineering/Principles/OOP) + [SOLID](/Software%20Engineering/Principles/SOLID) + daily [Problem Solving](/Software%20Engineering/DSA/01%20Foundations/Problem%20Solving)
2. Week 2: [Spring Boot Essentials](/Software%20Engineering/Backend/Java/Spring%20Boot%20Essentials) → [JPA, Hibernate & jOOQ - Data Access in Java](/Software%20Engineering/Backend/Java/JPA%2C%20Hibernate%20%26%20jOOQ%20-%20Data%20Access%20in%20Java) → PostgreSQL notes
3. Week 3: [Java Concurrency & Virtual Threads](/Software%20Engineering/Backend/Java/Java%20Concurrency%20%26%20Virtual%20Threads) → [Reactor - Mono & Flux](/Software%20Engineering/Backend/Java/Java%20Reactive%20Course/Reactor%20-%20Mono%20%26%20Flux) → [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming)
4. Week 4: [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index) + [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns) + mock interviews; skim [Spring Boot 3 to 4 - What's New](/Software%20Engineering/Backend/Java/Spring%20Boot%203%20to%204%20-%20What's%20New) and [Modern Java 17 to 25](/Software%20Engineering/Backend/Java/Modern%20Java%2017%20to%2025) for "what's new" questions
</Info>

## Related
- [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index) · [Interviews Index](/Software%20Engineering/Interviews/Interviews%20Index)
