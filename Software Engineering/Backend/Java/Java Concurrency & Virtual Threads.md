---
title: "Java Concurrency & Virtual Threads"
tags: [backend, backend/java]
aliases: [Java Concurrency, Virtual Threads, Project Loom, ExecutorService, CompletableFuture, Structured Concurrency, Java threads, synchronized]
updated: 2026-09-27
---
# Java Concurrency & Virtual Threads

<Note>
**TL;DR**

Java runs code on **threads** that share the heap, so you need **synchronization** (locks, `volatile`, atomics, concurrent collections) to avoid races. You rarely create threads yourself: submit tasks to an **`ExecutorService`** and compose async results with **`CompletableFuture`**. Since Java 21, **virtual threads** make threads cheap (millions per JVM), so simple blocking code scales for I/O-bound servers, often removing the need for reactive programming. **Structured concurrency** (treating a group of subtasks as one unit) is still a preview API in 2026.
</Note>

## Platform threads vs virtual threads

{/* diagram source: /diagrams/diagram-041.mmd */}
![diagram](/diagrams/diagram-041.svg)

- A **platform thread** wraps an OS thread: expensive to create, ~1 MB stack by default, a few thousand per JVM. See [Reactive Spring - Ch 4 IO & Reactive Streams](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%204%20IO%20%26%20Reactive%20Streams) for why that motivated reactive programming.
- A **virtual thread** is scheduled by the JVM onto a small pool of **carrier** threads. When it blocks on I/O (socket read, `Thread.sleep`, `BlockingQueue.take`, JDBC), the JVM **unmounts** it and the carrier runs another virtual thread. Its stack lives on the heap and grows as needed.

```java
// 10,000 concurrent blocking tasks on virtual threads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i ->
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));   // blocks the virtual thread, not the carrier
            return i;
        }));
}   // close() waits for all tasks: ~1 s total

Thread.ofVirtual().name("worker-", 0).start(() -> System.out.println("hi"));
```

In Spring Boot 3.2+: `spring.threads.virtual.enabled=true` runs request handling (Tomcat/Jetty), `@Async`, schedulers and more on virtual threads.

### Rules for virtual threads
- **Don't pool them.** Create one per task; they're cheap. Limit concurrency to downstreams with a `Semaphore`, not a thread-pool size.
- They help **I/O-bound** work, not CPU-bound work (still limited by cores).
- **Pinning**: before Java 24, blocking inside `synchronized` pinned the carrier. **JEP 491 (Java 24)** fixed that; native calls/JNI frames still pin. Detect with JFR event `jdk.VirtualThreadPinned`.
- `ThreadLocal` works but costs memory per thread with millions of threads; prefer **scoped values** (final in Java 25).

## The basics: threads, races and the memory model

```java
class Counter {
    private int count = 0;
    void increment() { count++; }            // read-modify-write: NOT atomic → race condition
}
```

Fixes:

```java
class SyncCounter { private int c; synchronized void inc() { c++; } }            // intrinsic lock
class AtomicCounter { private final AtomicInteger c = new AtomicInteger(); void inc() { c.incrementAndGet(); } } // CAS
class LongAdderCounter { private final LongAdder c = new LongAdder(); void inc() { c.increment(); } }          // high contention
```

- **Java Memory Model**: without a *happens-before* relation (lock release→acquire, `volatile` write→read, thread start/join), one thread may not see another's writes.
- `volatile` gives visibility and ordering, **not** atomicity of compound operations.
- `ReentrantLock` adds `tryLock`, timeouts and fairness; `ReadWriteLock`/`StampedLock` for read-heavy data.
- **Deadlock**: two threads acquire locks in opposite orders. Fix with a global lock order or `tryLock` with timeout.
- Prefer immutable objects and confinement over shared mutable state.

## Executors and futures

```java
ExecutorService pool = Executors.newFixedThreadPool(8);          // CPU-bound: ~#cores
Future<Report> f = pool.submit(() -> buildReport());
Report r = f.get(5, TimeUnit.SECONDS);                           // blocks caller
pool.shutdown();
```

| Executor | Use |
|---|---|
| `newFixedThreadPool(n)` | Bounded CPU work |
| `newVirtualThreadPerTaskExecutor()` | Blocking I/O tasks (Java 21+) |
| `newScheduledThreadPool(n)` | Delayed / periodic tasks |
| `ForkJoinPool` / `commonPool()` | Divide-and-conquer, parallel streams, default for `CompletableFuture.*Async` |
| `ThreadPoolExecutor` (custom) | Explicit queue size and rejection policy (backpressure) |

<Warning>
**`newCachedThreadPool` and unbounded queues**

Cached pools create unbounded platform threads; fixed pools use an unbounded queue. Under load both can exhaust memory. In production configure `ThreadPoolExecutor` with a bounded queue and a rejection policy, or use virtual threads + semaphores.
</Warning>

## CompletableFuture

```java
CompletableFuture<User> user = CompletableFuture.supplyAsync(() -> users.find(id), ioExecutor);
CompletableFuture<List<Order>> orders = CompletableFuture.supplyAsync(() -> ordersApi.forUser(id), ioExecutor);

CompletableFuture<Dashboard> dashboard = user
    .thenCombine(orders, Dashboard::new)                 // both in parallel
    .orTimeout(2, TimeUnit.SECONDS)
    .exceptionally(ex -> Dashboard.empty());             // fallback
```

- `thenApply` (map), `thenCompose` (flatMap), `thenCombine` (zip), `allOf`/`anyOf`.
- Always pass an **executor** for blocking work; the default common pool is sized for CPU and shared JVM-wide.
- Errors propagate as `CompletionException`; handle with `exceptionally`/`handle`/`whenComplete`.

With virtual threads the same code can be written sequentially:

```java
try (var ex = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<User> u = ex.submit(() -> users.find(id));
    Future<List<Order>> o = ex.submit(() -> ordersApi.forUser(id));
    return new Dashboard(u.get(), o.get());
}
```

## Structured concurrency (preview)

Treats subtasks forked in a scope as a unit: if one fails, the others are cancelled; the scope can't outlive its block; thread dumps show the hierarchy.

```java
// JDK 25+ preview API (JEP 505/525/533): compile & run with --enable-preview
Response handle(String id) throws InterruptedException {
    try (var scope = StructuredTaskScope.open()) {          // default: fail if any subtask fails
        Subtask<User> user   = scope.fork(() -> users.find(id));
        Subtask<Integer> cnt = scope.fork(() -> orders.count(id));
        scope.join();                                        // waits; cancels siblings on failure
        return new Response(user.get(), cnt.get());
    }
}
```

<Info>
**🆕 2025–2026**

- **Java 24 (2025-03-18)**, JEP 491: virtual threads no longer pin carriers inside `synchronized` blocks. This removed the biggest practical blocker (JDBC drivers and libraries using `synchronized`).
- **Java 25 LTS (2025-09-16)**: **Scoped Values** final (JEP 506). Structured concurrency 5th preview with a reworked API (`StructuredTaskScope.open()` + `Joiner`s).
- **Java 26 (2026-03-17)**: structured concurrency 6th preview; **Java 27 (2026-09-15)**: 7th preview. Still not final.
- **Spring Boot 4.0 (2025-11-20)**: JDK `HttpClient` auto-configuration uses virtual threads when they're enabled. See [Spring Boot 3 to 4 - What's New](/Software%20Engineering/Backend/Java/Spring%20Boot%203%20to%204%20-%20What's%20New).
</Info>

## Virtual threads vs reactive (WebFlux/Reactor)

| | Virtual threads (Spring MVC + blocking code) | Reactive (WebFlux + Reactor) |
|---|---|---|
| Programming model | Plain imperative, `try/catch`, stack traces make sense | Operator chains (`flatMap`...), different debugging |
| Scalability for I/O | High (millions of threads) | High (few threads, event loop) |
| Backpressure | Manual (semaphores, bounded queues) | Built in (Reactive Streams `request(n)`) |
| Streaming / SSE / many long-lived connections | Works, but reactive is more natural | Strong fit |
| Ecosystem | JDBC, JPA, all blocking libs just work | Needs reactive drivers (R2DBC, reactive Mongo/Redis) |
| Learning curve | Low | High |

Rule of thumb in 2026: for new request/response services use **virtual threads**; keep reactive for streaming, backpressure-heavy pipelines, or existing reactive codebases. See [Reactor - Mono & Flux](/Software%20Engineering/Backend/Java/Java%20Reactive%20Course/Reactor%20-%20Mono%20%26%20Flux) and [Reactive Spring Book](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20Book).

## Pitfalls
- Sharing a non-thread-safe object (`SimpleDateFormat`, `HashMap`) between threads.
- Calling `.get()`/`.join()` on futures inside the common pool (thread starvation).
- Swallowed exceptions in `executor.submit` (they're stored in the `Future`, never logged unless you `get()`).
- Virtual threads + a small DB connection pool: 10,000 threads will all wait on 10 connections. Size downstream limits deliberately.
- `parallelStream()` with blocking I/O inside.

## Senior perspective
- Choose the model per workload: CPU-bound → bounded platform pool ≈ cores; I/O-bound → virtual threads; streaming with backpressure → reactive.
- Concurrency limits belong at the **resource** (DB pool, HTTP client, semaphore per downstream), with timeouts and bulkheads (Resilience4j, Spring 7 `@ConcurrencyLimit`).
- Observability: thread dumps (`jcmd <pid> Thread.dump_to_file -format=json` includes virtual threads), JFR events for pinning and lock contention.
- In review: shared mutable state, lock ordering, unbounded queues, missing timeouts, blocking calls in reactive pipelines, `ThreadLocal` leaks in pools.

## Interview questions

<Accordion title="What are virtual threads and when should you use them?">
Lightweight threads managed by the JVM (Java 21, JEP 444) that unmount from their carrier OS thread when blocked. Use them for high-concurrency I/O-bound work (request handling, calling DBs/APIs) with plain blocking code. They don't speed up CPU-bound work and shouldn't be pooled.
</Accordion>

<Accordion title="What is pinning?">
A virtual thread that can't unmount while blocked, so it holds its carrier OS thread. It used to happen inside `synchronized` (fixed in Java 24, JEP 491) and still happens during native/JNI frames.
</Accordion>

<Accordion title="`synchronized` vs `volatile` vs `AtomicInteger`?">
`synchronized` gives mutual exclusion and visibility. `volatile` gives visibility/ordering for a single variable but no atomic compound operations. `AtomicInteger` gives lock-free atomic operations via CAS.
</Accordion>

<Accordion title="How does `CompletableFuture` differ from `Future`?">
`Future` only lets you block with `get()`. `CompletableFuture` can be completed manually and composed non-blockingly (`thenApply`, `thenCompose`, `thenCombine`, `allOf`) with error handling and timeouts.
</Accordion>

<Accordion title="What is structured concurrency?">
Scoping concurrent subtasks to a code block so they start and finish together: failures cancel siblings, no thread leaks, clear parent-child relationships in dumps. Preview in Java 21–27 via `StructuredTaskScope`.
</Accordion>

<Accordion title="Do virtual threads make reactive programming obsolete?">
For most request/response services, largely yes: you get similar scalability with simpler code. Reactive still wins for streaming, built-in backpressure and composing many async event sources.
</Accordion>

<Accordion title="How do you avoid deadlocks?">
Acquire locks in a consistent global order, hold them briefly, avoid calling unknown code while holding a lock, use `tryLock` with timeouts, and prefer higher-level concurrent utilities.
</Accordion>

## Related
- [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index)
- [Modern Java 17 to 25](/Software%20Engineering/Backend/Java/Modern%20Java%2017%20to%2025) · [JVM Internals - Memory & Garbage Collection](/Software%20Engineering/Backend/Java/JVM%20Internals%20-%20Memory%20%26%20Garbage%20Collection) · [Spring Boot Essentials](/Software%20Engineering/Backend/Java/Spring%20Boot%20Essentials)
- [Reactor - Mono & Flux](/Software%20Engineering/Backend/Java/Java%20Reactive%20Course/Reactor%20-%20Mono%20%26%20Flux) · [Reactive Spring - Ch 4 IO & Reactive Streams](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%204%20IO%20%26%20Reactive%20Streams) · [Reactive Spring - Ch 5 Reactor & Backpressure](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%205%20Reactor%20%26%20Backpressure)
- [Concurrency And Parallelism](/Software%20Engineering/Principles/Concurrency%20And%20Parallelism) · Node comparison: [Node.js Concurrency - Worker Threads, Cluster & Child Processes](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Concurrency%20-%20Worker%20Threads%2C%20Cluster%20%26%20Child%20Processes)

## Sources
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [JEP 506: Scoped Values](https://openjdk.org/jeps/506)
- [JEP 505: Structured Concurrency (Fifth Preview)](https://openjdk.org/jeps/505) · [JEP 533 (Seventh Preview, JDK 27)](https://openjdk.org/jeps/533)
- [Oracle docs: Virtual threads](https://docs.oracle.com/en/java/javase/25/core/virtual-threads.html)
- [Spring Boot reference: Virtual threads](https://docs.spring.io/spring-boot/reference/features/spring-application.html#features.spring-application.virtual-threads)
- Brian Goetz et al., *Java Concurrency in Practice* (Addison-Wesley, 2006)
