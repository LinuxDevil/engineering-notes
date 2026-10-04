---
title: "Java Reactive Course"
tags: [backend, backend/java, course]
aliases: [Reactive Course, Java Reactive Programming Course]
updated: 2026-09-27
---
# Java Reactive Course

<Note>
**TL;DR**

Course notes on reactive programming in Java with **Project Reactor**. The starting point is the four I/O models (sync-blocking, async, non-blocking, non-blocking + async) and why reactive code tries to never hold a thread while waiting. The Reactor details (Mono, Flux, operators, schedulers, backpressure) are in [Reactor - Mono & Flux](/Software%20Engineering/Backend/Java/Java%20Reactive%20Course/Reactor%20-%20Mono%20%26%20Flux).
</Note>

## The four I/O models (the insurance-company phone call)

Imagine calling an insurance company that puts you on hold.

| Model | Phone-call analogy | In code |
|---|---|---|
| 1. **Sync + blocking** (default) | You hold the phone and wait, doing nothing else | `String body = httpClient.send(...)` on the request thread |
| 2. **Async** | You hire someone (another thread) to hold the phone for you, so *you* are free; but that person is blocked | `CompletableFuture.supplyAsync(() -> blockingCall(), pool)` |
| 3. **Non-blocking** | You ask them to call you back; you keep checking your phone | Low-level: `Selector`/NIO polling, "is it ready yet?" |
| 4. **Non-blocking + async** (the "hyped" approach) | They call you back when ready and you react; nobody waits on hold | Event loop + callbacks: Netty, Reactor, WebFlux |

{/* diagram source: /diagrams/diagram-045.mmd */}
![diagram](/diagrams/diagram-045.svg)

Reactive programming (model 4) lets a handful of threads serve thousands of concurrent requests because no thread sits idle waiting for I/O. Since Java 21, **virtual threads** give similar scalability with plain blocking code; see [Java Concurrency & Virtual Threads](/Software%20Engineering/Backend/Java/Java%20Concurrency%20%26%20Virtual%20Threads) for when each fits.

## Course notes
- [Reactor - Mono & Flux](/Software%20Engineering/Backend/Java/Java%20Reactive%20Course/Reactor%20-%20Mono%20%26%20Flux): publishers, laziness, `just` vs `fromSupplier`, operators, schedulers, backpressure

## Related
- [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index)
- [Reactive Spring Book](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20Book) · [Reactive Spring - Ch 4 IO & Reactive Streams](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%204%20IO%20%26%20Reactive%20Streams) · [Reactive Spring - Ch 5 Reactor & Backpressure](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%205%20Reactor%20%26%20Backpressure)
- [Java Concurrency & Virtual Threads](/Software%20Engineering/Backend/Java/Java%20Concurrency%20%26%20Virtual%20Threads)

## Sources
- [Project Reactor reference guide](https://projectreactor.io/docs/core/release/reference/)
- [Reactive Streams specification](https://www.reactive-streams.org/)
