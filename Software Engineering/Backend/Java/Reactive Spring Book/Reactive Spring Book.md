---
title: "Reactive Spring Book"
tags: [backend, backend/java, book]
aliases: [Reactive Spring, Reactive Spring (Josh Long)]
updated: 2026-09-27
---
# Reactive Spring Book

<Note>
**TL;DR**

Notes from **Josh Long's *Reactive Spring*** (2020). The book builds from a Java refresher and Spring fundamentals (DI, configuration, auto-configuration) to the reason for reactive programming (threads are expensive, I/O makes them idle), the **Reactive Streams** contract and **Project Reactor**. Chapter notes were renamed on 2026-09-27 to unique names. The book predates virtual threads (Java 21) and Spring Boot 3/4; see the topic notes for the current picture.
</Note>

## Chapter notes
| Chapter | Note | Topic |
|---|---|---|
| 2 | [Reactive Spring - Ch 2 Java Refresher](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%202%20Java%20Refresher) | `var`, lambdas, functional interfaces, method references, abstract class vs interface |
| 3 | [Reactive Spring - Ch 3 Spring Fundamentals](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%203%20Spring%20Fundamentals) | DI/IoC, Java config, `BeanPostProcessor`, component scanning, starters, profiles |
| 4 | [Reactive Spring - Ch 4 IO & Reactive Streams](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%204%20IO%20%26%20Reactive%20Streams) | Why reactive, threads vs I/O, the four Reactive Streams interfaces |
| 5 | [Reactive Spring - Ch 5 Reactor & Backpressure](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%205%20Reactor%20%26%20Backpressure) | Cooperative multithreading, backpressure, Mono/Flux, signals |

## What changed since the book (2020 → 2026)
- **Spring Boot 3** (2022): Java 17, `jakarta.*`; **Boot 4 / Framework 7** (Nov 2025): Jakarta EE 11, JSpecify, API versioning, HTTP service clients. See [Spring Boot 3 to 4 - What's New](/Software%20Engineering/Backend/Java/Spring%20Boot%203%20to%204%20-%20What's%20New).
- **Virtual threads** (Java 21, 2023) make blocking code scale, reducing the need for reactive in request/response services. See [Java Concurrency & Virtual Threads](/Software%20Engineering/Backend/Java/Java%20Concurrency%20%26%20Virtual%20Threads).
- WebFlux, Reactor and R2DBC remain supported and are the right tool for streaming and backpressure-heavy systems. See [Reactor - Mono & Flux](/Software%20Engineering/Backend/Java/Java%20Reactive%20Course/Reactor%20-%20Mono%20%26%20Flux).

## Related
- [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index) · [Java Reactive Course](/Software%20Engineering/Backend/Java/Java%20Reactive%20Course/Java%20Reactive%20Course) · [Spring Boot Essentials](/Software%20Engineering/Backend/Java/Spring%20Boot%20Essentials)

## Sources
- Josh Long, *Reactive Spring* (2020): https://reactivespring.io/
- [Spring WebFlux reference](https://docs.spring.io/spring-framework/reference/web/webflux.html)
