---
tags: [principles, principles/concurrency]
aliases: [Concurrency, Parallelism, Concurrency vs Parallelism, Processes and Threads, Threads, Thread Pool, IPC, Sync vs Async, Race Condition, Deadlock, Virtual Threads]
updated: 2026-09-27
---
# Concurrency And Parallelism

> [!tldr] TL;DR
> **Concurrency** is *dealing with* many things at once (tasks interleave and make progress over overlapping time); **parallelism** is *doing* many things at once (tasks literally run simultaneously on multiple cores). A **process** is a running program with its own memory; a **thread** is a unit of execution inside a process that shares the heap with its siblings. Shared mutable state + threads = race conditions, so use locks, atomics, immutability or message passing. Modern runtimes lean on lightweight concurrency: event loops (Node.js), coroutines (Kotlin, Python asyncio), goroutines and **Java virtual threads**.

## The idea

- **Parallelism**: doing more than one thing **at the same time** (needs multiple cores/CPUs).

![[Untitled 24.png]]

- **Concurrency**: the **illusion** of doing more than one thing at the same time, by switching between tasks. On a single core it's achieved with **context switching**; on multiple cores it can also use parallelism.

![[Untitled 25.png]]

> "Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once." (Rob Pike, *Concurrency is not Parallelism*, 2012)

A barista analogy: one barista juggling three orders (grind, steam, pour, switch) is concurrent. Three baristas each making an order are parallel. Three baristas juggling nine orders are both.

## Processes

A **process** is an instance of a running program.

![[Untitled 26.png]]

- Each process has its **own isolated memory space**; one process can't read another's memory directly (the OS enforces this).
- A runtime may use one or several processes. A **multi-process model** (e.g. running several Node.js processes via `cluster`/PM2, or Chrome's process-per-site) uses more memory and resources than a single multi-threaded process (e.g. one JVM), but a crash in one process doesn't take the others down.
- Creating a process is expensive compared with a thread.

### Inter-process communication (IPC)

Because memory is isolated, processes communicate through the OS:

| Mechanism | How it works | Example |
|---|---|---|
| **Files / memory-mapped files** | Processes read and write the same file; with `mmap` a portion of RAM is mapped to the file so several processes see it | Databases (SQLite, LMDB) |
| **Signals** | Tiny asynchronous notifications carrying just a signal number | `SIGTERM` (graceful stop), `SIGKILL` (force kill, can't be caught), `SIGINT` (Ctrl+C), `SIGHUP` (reload config) |
| **Pipes** | Connect one process's standard output to another's standard input | `ls \| grep hello`: `ls` stdout → pipe → `grep` stdin |
| **Sockets** | **Unix domain sockets** (same machine only, fast) or **network sockets** (TCP/UDP, across machines). One side listens, the other connects | Docker daemon socket, PostgreSQL local socket; browser → server over TCP |
| **Shared memory** | A memory segment mapped into several processes; fastest, needs synchronisation | `shm_open`, `SharedArrayBuffer` between workers |
| **Message queues / message passing** | The OS or a broker delivers discrete messages | POSIX message queues, D-Bus; at system scale Kafka, RabbitMQ |

```text
ls | grep hello

[ls] --stdout--> ( pipe ) --stdin--> [grep] --stdout--> terminal
```

## Threads

A **thread** is the smallest sequence of instructions the OS can schedule. A thread is what executes your code; every process has at least one (the main thread).

![[Untitled 25.png]]

A process typically has several threads, e.g. in the JVM:
- code-execution thread(s)
- garbage-collection thread(s)
- network / I/O threads
- other runtime threads (JIT compiler, timers)

Threads are **lighter** than processes and different threads can run on different CPU cores in parallel.

### Thread anatomy: stack and heap

```text
Process
├── Heap (shared by ALL threads of the process)
│     objects, closures, caches... memory leaks live here
├── Thread 1 ── Stack: local variables, parameters, return addresses (call chain)
├── Thread 2 ── Stack
└── Thread 3 ── Stack
```

Because the **heap is shared**, two threads can touch the same object at the same time. That is both the power (cheap communication) and the danger (races) of threads.

### Thread overhead

- **Thread-per-request** (one OS thread per user/request) is simple but costly at scale.
- Spawning a new OS thread is relatively slow (a system call, stack allocation).
- The OS limits the number of threads.
- Each thread reserves memory for its stack (the JVM default is typically **1 MB** per platform thread on 64-bit Linux, configurable with `-Xss`).
- **Contention**: two or more threads compete for the same (or related) resource, so at least one runs slower than it would alone:
  - **Locks**: threads wait to acquire the same lock.
  - **CPU time**: more runnable threads than cores means context switching.
  - **Shared resources**: cache lines (false sharing), connection pools, I/O.

## Thread pools

A thread pool is a **manager for threads**: a fixed (or bounded) set of worker threads takes tasks from a queue. If the maximum number of threads hasn't been reached it may create another; otherwise the task waits in the queue for a free worker (or is rejected when the queue is full).

It **sacrifices some availability to make the system more reliable (stable)**: under load, requests wait or get rejected instead of the server creating unlimited threads and crashing.

```java
ExecutorService pool = Executors.newFixedThreadPool(8);   // bounded concurrency
Future<Report> f = pool.submit(() -> buildReport(id));
Report r = f.get(2, TimeUnit.SECONDS);
pool.shutdown();
```

Sizing rule of thumb (Goetz, *Java Concurrency in Practice*): CPU-bound work ≈ number of cores (+1); I/O-bound work ≈ cores × (1 + wait time / compute time). Always bound the queue.

## Synchronous vs asynchronous

These describe the **programming model**: whether a caller waits for a task to finish before moving on.

### Synchronous

Executing tasks one after another; each waits for the previous one to complete.

Single thread:

![[Untitled 28.png]]

Multiple threads (each thread still runs its own tasks one by one):

![[Untitled 29.png]]

### Asynchronous

You can switch to a different task without waiting for the previous one to complete; you get notified (callback, promise/future, event) when it's done.

Single thread (e.g. the Node.js event loop, see [[Event Loop - Microtasks & Macrotasks]]):

![[Untitled 30.png]]

Multiple threads:

![[Untitled 31.png]]

- The async programming model helps you achieve **concurrency**.
- Async programming in a **multi-threaded** environment is a way to achieve **parallelism**.

```ts
// Concurrency on ONE thread: both requests are in flight at once, the thread never blocks
const [user, orders] = await Promise.all([fetchUser(id), fetchOrders(id)]);

// Parallelism in JS needs extra threads: Worker threads / Web Workers
```

### Summing up

| Axis | Values | What it describes |
|---|---|---|
| **Concurrency vs parallelism** | interleaved vs simultaneous | The **way tasks are executed** |
| **Synchronous vs asynchronous** | wait vs don't wait | The **programming model** |
| **Single-threaded vs multi-threaded** | one vs many threads | The **environment** of task execution |

Concurrency and parallelism relate to how the machine performs the work; sync/async is how you write the code.

## Concurrency hazards

### Race condition

The result depends on the timing of threads. The classic lost update:

```java
class Counter {
    private int count = 0;
    void increment() { count++; }   // read, add, write: three steps, not atomic
}
// Two threads call increment() 1,000,000 times each → result is often < 2,000,000
```

Fixes, from simplest:

```java
private final AtomicInteger count = new AtomicInteger();   // lock-free atomic
void increment() { count.incrementAndGet(); }

synchronized void increment() { count++; }                 // mutual exclusion (lock)

// Or avoid sharing: each thread counts locally and results are combined (map-reduce),
// or state is immutable and replaced atomically.
```

A **data race** (unsynchronised access where at least one is a write) is a specific kind of race condition; in Java, C++ and Go it also means undefined or surprising visibility (one thread may never see another's write without `volatile`/locks/happens-before).

### Deadlock, livelock, starvation

- **Deadlock**: T1 holds lock A and waits for B; T2 holds B and waits for A. Needs all four Coffman conditions: mutual exclusion, hold-and-wait, no preemption, circular wait. Prevent by **always acquiring locks in a global order**, using timeouts (`tryLock`), or avoiding nested locks.
- **Livelock**: threads keep reacting to each other and make no progress (two people stepping aside in a corridor).
- **Starvation**: a thread never gets the resource because others always win (unfair locks, priority inversion).

### Tools, from low to high level

| Level | Tools |
|---|---|
| Primitives | mutex/`synchronized`, `ReentrantLock`, read-write locks, semaphores, condition variables, atomics/CAS, `volatile` |
| Data structures | `ConcurrentHashMap`, blocking queues, copy-on-write lists |
| Executors | thread pools, `CompletableFuture`, fork/join |
| Models | **message passing** (Go channels, actors in Erlang/Akka), **async/await** + event loop, **structured concurrency**, software transactional memory (Clojure) |
| Design | **immutability** (see [[Immutability Across Languages]]), confinement (one owner thread), share-nothing processes |

> "Do not communicate by sharing memory; instead, share memory by communicating." (Go proverb)

## Concurrency models by runtime

| Runtime | Model | Parallel CPU work? |
|---|---|---|
| **Node.js / browser JS** | Single-threaded event loop; async I/O via libuv thread pool | Via `worker_threads` / Web Workers |
| **Java** | OS (platform) threads, thread pools; **virtual threads** since Java 21 | Yes |
| **Go** | Goroutines (M:N scheduled) + channels | Yes |
| **Kotlin** | Coroutines on dispatchers (structured concurrency built in) | Yes (on JVM threads) |
| **Python** | asyncio for I/O; threads limited by the GIL; multiprocessing for CPU; **free-threaded build** optional since 3.13 | Processes; free-threaded build |
| **Rust** | OS threads + async runtimes (Tokio); ownership prevents data races at compile time | Yes |
| **Erlang/Elixir** | Millions of isolated lightweight processes, message passing, supervisors | Yes |

### Virtual threads (Java 21+)

Virtual threads are cheap, JVM-managed threads mounted on a small pool of OS "carrier" threads. When a virtual thread blocks on I/O it unmounts, freeing the carrier. You write simple blocking code, one thread per task, and still handle huge numbers of concurrent I/O-bound tasks.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        int n = i;   // lambdas need an effectively final variable
        executor.submit(() -> { Thread.sleep(Duration.ofSeconds(1)); return fetch(n); });
    }
}   // 10,000 concurrent blocking tasks without 10,000 OS threads
```

They help **I/O-bound** throughput, not CPU-bound speed, and you still need to limit access to scarce resources (DB connections) with semaphores or pools.

> [!info]+ 🆕 2025–2026
> - **Java 24 (March 2025), JEP 491**: virtual threads no longer get **pinned** to their carrier when blocking inside `synchronized` blocks. This removed the main caveat of Java 21 virtual threads; libraries no longer need to replace `synchronized` with `ReentrantLock` for virtual-thread friendliness.
> - **Java 25 LTS (2025-09-16)**: **Scoped Values** finalised (JEP 506), an immutable, bounded alternative to `ThreadLocal` that works well with many virtual threads. **Structured Concurrency** remains in preview: fifth preview in Java 25 (JEP 505), sixth in Java 26 (JEP 525), seventh in Java 27 (JEP 533, GA 2026-09-15).
> - **Python 3.14 (October 2025)**: the **free-threaded (no-GIL) build is officially supported** (PEP 779), though still optional; and **multiple interpreters** are available in the standard library (PEP 734, `concurrent.interpreters`), each with its own GIL, for parallelism inside one process.

### Structured concurrency

Treat a group of concurrent subtasks as one unit: they start inside a scope, and the scope doesn't exit until all finish; if one fails, the others are cancelled. It prevents leaked tasks and lost errors. Kotlin coroutines (`coroutineScope`), Swift task groups, Python `asyncio.TaskGroup` (3.11+) and Java's preview `StructuredTaskScope` implement it.

```python
async def load_dashboard(user_id):
    async with asyncio.TaskGroup() as tg:          # all-or-nothing scope
        profile = tg.create_task(get_profile(user_id))
        orders = tg.create_task(get_orders(user_id))
    return profile.result(), orders.result()       # if one raised, the other was cancelled
```

## Pitfalls

> [!warning] Common mistakes
> - Assuming `async` means "runs in parallel": in Node.js a CPU-heavy loop inside an async function still blocks the event loop.
> - Unbounded parallelism: `Promise.all(10_000 requests)` or unbounded thread creation overwhelms downstream services. Use a limit (p-limit, semaphores, bounded pools).
> - Check-then-act races: `if (!map.containsKey(k)) map.put(k, v)` → use `putIfAbsent` / `computeIfAbsent` or a DB unique constraint.
> - Holding locks while doing I/O or calling unknown code.
> - Blocking calls on event-loop or reactive threads.
> - `ThreadLocal` leaks in thread pools (values survive into the next task).
> - Forgetting that the **database** is the biggest shared resource: concurrency bugs often become lost updates in SQL (use transactions, `SELECT ... FOR UPDATE`, optimistic locking with version columns).

## Senior perspective

- First decide if the work is **I/O-bound** (use async, event loops or virtual threads) or **CPU-bound** (use parallelism: processes, thread pools sized to cores, workers).
- Prefer designs that avoid shared mutable state: immutable messages, single-writer ownership, queues. Locks are the last resort, not the first.
- Every concurrent system needs **backpressure** and **timeouts**; otherwise queues grow without bound under load.
- Concurrency bugs are timing-dependent: rely on stress tests, race detectors (Go `-race`, ThreadSanitizer), and design reviews more than on example tests.
- Across machines the same problems appear as distributed-systems problems (idempotency, ordering, exactly-once illusions). See [[Software Architecture Styles]].

## Interview questions

> [!question]- What's the difference between concurrency and parallelism?
> Concurrency: structuring a program so multiple tasks make progress over overlapping periods (interleaving, possible on one core). Parallelism: tasks executing at the same instant on multiple cores. Concurrency is about structure, parallelism about execution.

> [!question]- Process vs thread?
> A process is an isolated running program with its own address space; creating one is expensive and communication needs IPC. Threads live inside a process, share its heap (each has its own stack), are cheaper to create and switch, but shared memory requires synchronisation.

> [!question]- What is a race condition and how do you prevent it?
> When correctness depends on the timing of concurrent operations on shared state (e.g. lost updates from `count++`). Prevent it with atomic operations, locks, confinement, immutability, or message passing; at the DB level with transactions and constraints.

> [!question]- What conditions are required for a deadlock and how do you avoid one?
> Mutual exclusion, hold and wait, no preemption, circular wait. Avoid by ordering lock acquisition globally, using timeouts/`tryLock`, reducing lock scope, or avoiding locks through message passing.

> [!question]- How is Node.js concurrent if it's single-threaded?
> JavaScript runs on one thread with an event loop; I/O is delegated to the OS (epoll/kqueue/IOCP) and libuv's thread pool. When an operation completes, its callback is queued and run by the loop. CPU-bound work blocks the loop, so it belongs in worker threads.

> [!question]- What problem do Java virtual threads solve?
> Thread-per-request code blocks OS threads on I/O, and OS threads are expensive, so servers ran out of threads. Virtual threads are cheap and unmount while blocked, allowing millions of concurrent blocking tasks with simple synchronous code. They don't make CPU-bound code faster.

> [!question]- Why do we use thread pools?
> To reuse threads (avoid creation cost), bound concurrency (protect memory and downstream systems), and queue work under load. The trade-off: requests may wait or be rejected when the pool is saturated.

## Related
- [[Principles Index]]
- [[Immutability Across Languages]] · [[Functional Programming Principles]] · [[Software Architecture Styles]]
- JavaScript runtime: [[Event Loop - Microtasks & Macrotasks]] · [[Promises, Async-Await & Cancellation]] · [[Web Workers, Service Workers & PWAs]]

## Sources
- [Rob Pike: Concurrency is not Parallelism (2012)](https://go.dev/blog/waza-talk)
- Brian Goetz et al., *Java Concurrency in Practice* (Addison-Wesley, 2006)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444) · [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [JEP 506: Scoped Values](https://openjdk.org/jeps/506) · [JEP 505: Structured Concurrency (Fifth Preview)](https://openjdk.org/jeps/505) · [JDK 26](https://openjdk.org/projects/jdk/26/) · [JDK 27](https://openjdk.org/projects/jdk/27/)
- [What's new in Python 3.14 (PEP 779, PEP 734)](https://docs.python.org/3/whatsnew/3.14.html) · [PEP 779](https://peps.python.org/pep-0779/)
- [Python docs: asyncio.TaskGroup](https://docs.python.org/3/library/asyncio-task.html#task-groups)
- [Node.js: Don't block the event loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)
- [man7: signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html) · [pipe(7)](https://man7.org/linux/man-pages/man7/pipe.7.html) · [unix(7)](https://man7.org/linux/man-pages/man7/unix.7.html)
