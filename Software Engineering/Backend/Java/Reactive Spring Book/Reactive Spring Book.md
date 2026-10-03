---
tags: [backend, backend/java, book]
aliases: [Reactive Spring, Reactive Spring (Josh Long)]
updated: 2026-09-27
---
# Reactive Spring Book

> [!tldr] TL;DR
> Notes from **Josh Long's *Reactive Spring*** (2020). The book builds from a Java refresher and Spring fundamentals (DI, configuration, auto-configuration) to the reason for reactive programming (threads are expensive, I/O makes them idle), the **Reactive Streams** contract and **Project Reactor**. Chapter notes were renamed on 2026-09-27 to unique names. The book predates virtual threads (Java 21) and Spring Boot 3/4; see the topic notes for the current picture.

## Chapter notes
| Chapter | Note | Topic |
|---|---|---|
| 2 | [[Reactive Spring - Ch 2 Java Refresher]] | `var`, lambdas, functional interfaces, method references, abstract class vs interface |
| 3 | [[Reactive Spring - Ch 3 Spring Fundamentals]] | DI/IoC, Java config, `BeanPostProcessor`, component scanning, starters, profiles |
| 4 | [[Reactive Spring - Ch 4 IO & Reactive Streams]] | Why reactive, threads vs I/O, the four Reactive Streams interfaces |
| 5 | [[Reactive Spring - Ch 5 Reactor & Backpressure]] | Cooperative multithreading, backpressure, Mono/Flux, signals |

## What changed since the book (2020 → 2026)
- **Spring Boot 3** (2022): Java 17, `jakarta.*`; **Boot 4 / Framework 7** (Nov 2025): Jakarta EE 11, JSpecify, API versioning, HTTP service clients. See [[Spring Boot 3 to 4 - What's New]].
- **Virtual threads** (Java 21, 2023) make blocking code scale, reducing the need for reactive in request/response services. See [[Java Concurrency & Virtual Threads]].
- WebFlux, Reactor and R2DBC remain supported and are the right tool for streaming and backpressure-heavy systems. See [[Reactor - Mono & Flux]].

## Related
- [[Java & Spring Index]] · [[Java Reactive Course]] · [[Spring Boot Essentials]]

## Sources
- Josh Long, *Reactive Spring* (2020): https://reactivespring.io/
- [Spring WebFlux reference](https://docs.spring.io/spring-framework/reference/web/webflux.html)
