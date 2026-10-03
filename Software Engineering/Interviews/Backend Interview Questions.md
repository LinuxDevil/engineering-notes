---
tags: [interviews, interviews/backend]
aliases: [Backend Interview Q&A, Server-side Interview Questions, Node and Java Interview Questions]
updated: 2026-09-27
---
# Backend Interview Questions

> [!tldr] TL;DR
> Model answers for the backend questions asked most often: Node.js, Java & Spring, HTTP/API design, databases, distributed systems, security and operations. Each answer is short enough to say out loud (30–90 seconds) and links to the topic note for depth. Juniors should know sections 1–4 well; seniors are expected to go deep on 4–7 and to talk about trade-offs and failure modes.

> [!info] How to use
> Try each answer aloud before expanding it. For the underlying topics go to [[Node.js Index]], [[Java & Spring Index]], [[Databases Index]], [[System Design Index]] and [[Security Index]].

## 1. Node.js
> [!question]- How does Node.js handle many concurrent connections with one thread?
> JavaScript runs on one thread, but I/O is non-blocking: libuv hands network I/O to the OS (epoll/kqueue/IOCP) and runs file system, DNS lookup and crypto work on a thread pool (4 threads by default). Callbacks are queued and the event loop runs them when the stack is free. It is great for I/O-bound work and bad for CPU-bound work on the main thread. See [[Node.js Index]].

> [!question]- What are the phases of the Node.js event loop?
> timers (`setTimeout`/`setInterval`) → pending callbacks → poll (I/O) → check (`setImmediate`) → close callbacks. Between each callback, `process.nextTick` queue runs first, then promise microtasks. So `nextTick` > promises > timers/immediates.

> [!question]- How do you handle CPU-heavy work in Node?
> Move it off the event loop: `worker_threads`, a separate service/queue worker, or native addons. Split long loops, and use streaming for big payloads. Measure event-loop lag (`perf_hooks.monitorEventLoopDelay`).

> [!question]- What are streams and why use them?
> Streams process data in chunks (Readable, Writable, Duplex, Transform) so memory stays constant for large files or responses. Use `pipeline()` to connect them with proper error handling and **backpressure** (the producer pauses when the consumer's buffer is full).

> [!question]- How do you handle errors in an Express/Fastify app?
> `async` handlers with a central error middleware (Express 5 forwards rejected promises automatically), typed/operational errors mapped to HTTP status codes, no stack traces to clients, logging with a request id. Crash and restart on unknown programmer errors (`uncaughtException`) rather than continuing in a bad state.

> [!question]- CommonJS vs ES modules in Node?
> CJS: `require`, synchronous, dynamic. ESM: `import`, static (tree-shakable), top-level await, the standard. Modern Node versions can `require()` ESM modules synchronously (unflagged in Node 22.12/20.19) as long as they have no top-level await, which eases migration.

## 2. Java & Spring
> [!question]- How does the JVM manage memory and garbage collection?
> Heap (objects, split into young and old generations for most collectors), stacks per thread, and metaspace for class metadata. Generational GC assumes most objects die young. G1 is the default; ZGC (generational since JDK 21) offers sub-millisecond pauses for large heaps. See [[Java & Spring Index]].

> [!question]- What are virtual threads and when do they help?
> Lightweight threads (Project Loom, final in JDK 21) scheduled by the JVM onto a few carrier OS threads. Blocking I/O parks the virtual thread cheaply, so thread-per-request code scales to many concurrent requests without reactive code. They don't speed up CPU-bound work. JDK 24 removed most pinning when blocking inside `synchronized`. Spring Boot enables them with `spring.threads.virtual.enabled=true`.

> [!question]- `HashMap` internals?
> An array of buckets; the key's `hashCode` (spread) picks a bucket; collisions form a linked list that becomes a red-black tree after 8 entries (Java 8+). Resizes (doubles) at load factor 0.75. Keys need consistent `equals` and `hashCode`; not thread-safe (use `ConcurrentHashMap`).

> [!question]- What does Spring's dependency injection do? What are bean scopes?
> The IoC container creates objects (beans) and injects their dependencies, preferably through constructors. Scopes: singleton (default, one per container), prototype (new each time), request, session. A singleton must be thread-safe and stateless.

> [!question]- How does `@Transactional` work and what are its pitfalls?
> Spring wraps the bean in a proxy that begins/commits/rolls back a transaction around the method. Pitfalls: self-invocation bypasses the proxy; only unchecked exceptions roll back by default; it doesn't work on private methods; long transactions hold locks and connections; and the default propagation `REQUIRED` joins an existing transaction.

> [!question]- What is the N+1 query problem in JPA/Hibernate?
> Loading N parents and then lazily loading each child collection produces 1 + N queries. Fix with `JOIN FETCH`, entity graphs, batch fetching, or DTO projections, and detect it with SQL logging or tests that count queries.

> [!question]- Reactive (WebFlux) vs Spring MVC?
> WebFlux uses non-blocking I/O with Reactor (`Mono`/`Flux`) and backpressure: efficient for streaming and huge concurrency but harder to write and debug, and every layer must be non-blocking. With virtual threads, MVC covers most high-concurrency cases with simpler code.

## 3. HTTP and API design
> [!question]- What makes a good REST API?
> Resource-oriented URLs (`/orders/123`), correct methods and status codes, consistent errors (RFC 9457 Problem Details), pagination (cursor-based for large data), filtering, versioning strategy, idempotency for retries, authentication and authorization on every endpoint, and an OpenAPI contract.

> [!question]- Which HTTP methods are safe and idempotent?
> Safe (no side effects): GET, HEAD, OPTIONS. Idempotent (repeating has the same effect): GET, HEAD, OPTIONS, PUT, DELETE. POST and PATCH are not idempotent by default; make POST safe to retry with an `Idempotency-Key` header stored server-side.

> [!question]- REST vs GraphQL vs gRPC?
> REST: simple, cacheable, universal. GraphQL: clients choose fields, one endpoint, great for varied frontends, but caching, N+1 resolvers and query-cost limits are harder. gRPC: binary Protobuf over HTTP/2, fast, typed, streaming, ideal service-to-service, poor native browser support.

> [!question]- Offset vs cursor pagination?
> Offset (`LIMIT 20 OFFSET 10000`) is simple but the database still scans skipped rows, and results shift when rows are inserted. Cursor/keyset (`WHERE id > :last ORDER BY id LIMIT 20`) uses an index and is stable, but can't jump to page N.

> [!question]- How do you version an API?
> URL (`/v2/`), header, or media type. More important than the mechanism: make additive, backward-compatible changes by default, deprecate with timelines and `Deprecation`/`Sunset` headers, and track client usage before removing things.

## 4. Databases
> [!question]- What do ACID properties mean?
> Atomicity (all or nothing), Consistency (constraints hold), Isolation (concurrent transactions don't see each other's partial work, to the chosen level), Durability (committed data survives crashes, via the write-ahead log). See [[Databases Index]].

> [!question]- Explain isolation levels and the anomalies they prevent.
> Read uncommitted (dirty reads possible), read committed (no dirty reads; PostgreSQL default), repeatable read (no non-repeatable reads; MySQL InnoDB default), serializable (behaves like one-at-a-time execution). Anomalies: dirty read, non-repeatable read, phantom read, lost update, write skew.

> [!question]- How does a B-tree index speed up queries and when does it not help?
> A balanced tree of sorted keys gives O(log n) lookups and range scans. It doesn't help with low-selectivity columns, functions on the column (`WHERE lower(email) = ...` without an expression index), leading wildcards (`LIKE '%x'`), or when the column order of a composite index doesn't match the query. Indexes slow writes and use space.

> [!question]- SQL vs NoSQL: how do you choose?
> SQL for relational data, joins, transactions and flexible queries. Document/key-value/wide-column stores for specific access patterns at huge scale, flexible schemas, or very high write throughput. Most systems start relational; choose NoSQL for a proven access pattern, not fashion.

> [!question]- Replication vs sharding?
> Replication copies the same data to several nodes for read scaling and availability (with replication lag). Sharding splits data across nodes by a key for write and storage scaling, at the cost of cross-shard queries, rebalancing and hot keys.

> [!question]- How do you prevent lost updates?
> Optimistic locking (a `version` column: `UPDATE ... WHERE id = ? AND version = ?`, retry on 0 rows), pessimistic locking (`SELECT ... FOR UPDATE`), atomic updates (`SET stock = stock - 1 WHERE stock > 0`), or serializable isolation.

> [!question]- What is connection pooling and why does it matter?
> Opening DB connections is expensive, and databases support limited connections. A pool (HikariCP, pgbouncer) reuses a fixed set. Size it small (roughly cores × 2 on the DB side is a common starting point) and remember total connections = pool size × app instances.

## 5. Distributed systems and system design
> [!question]- What does the CAP theorem say, and what is PACELC?
> During a network **P**artition, a distributed store must choose **C**onsistency or **A**vailability. PACELC adds: **E**lse (no partition) you trade **L**atency against **C**onsistency. In practice you choose per operation. See [[System Design Index]].

> [!question]- How do you design for idempotency and retries?
> Retries with exponential backoff and jitter; idempotency keys for writes; deduplication on the consumer side for at-least-once messaging; upserts; and the outbox pattern to publish events atomically with a DB change.

> [!question]- How does caching go wrong?
> Stale data (need a TTL or invalidation strategy), cache stampede on expiry (request coalescing, early refresh), hot keys, cache penetration for missing keys (cache negatives), and inconsistency between cache and DB on write (cache-aside: write DB, then delete the cache key).

> [!question]- Queue vs log-based streaming (RabbitMQ vs Kafka)?
> Queues deliver each message to one consumer and delete it after ack; good for task distribution and routing. Kafka is an append-only partitioned log; consumers track offsets, can replay, and order is kept per partition; good for event streaming and high throughput.

> [!question]- How would you design a rate limiter?
> Token bucket or sliding window counters keyed by user/API key/IP, stored in Redis with atomic operations (Lua script) for a distributed limit. Return `429` with `Retry-After`. Decide fail-open vs fail-closed when Redis is down. Apply at the gateway and per service.

> [!question]- Monolith vs microservices?
> Start with a well-modularised monolith; move to services when teams need independent deployment and scaling. Microservices add network failures, distributed data, observability and operational cost. The main driver is organisational, not technical.

## 6. Security
> [!question]- How do you prevent SQL injection?
> Parameterised queries / prepared statements everywhere, ORMs used correctly (no string-built raw SQL), allow-lists for identifiers like sort columns, least-privilege DB users. See [[Backend Security Essentials]].

> [!question]- Authentication vs authorization; where do most API breaches come from?
> Authentication proves who you are; authorization decides what you may do. Most API breaches are authorization failures: Broken Object Level Authorization (IDOR), where the server doesn't check that the object belongs to the caller. See [[API Security]] and [[OWASP Top 10]].

> [!question]- How should services store and use secrets?
> Never in code or images. Use a secrets manager or cloud KMS, inject at runtime, prefer short-lived credentials (workload identity, OIDC federation), rotate, and scan repos for leaks. See [[Secrets Management]].

> [!question]- How should passwords be stored?
> With a slow, salted password hash: Argon2id (preferred), scrypt, or bcrypt, with parameters tuned to your hardware. Never plain hashes like SHA-256. See [[Cryptography Basics for Engineers]] and [[Passkeys, WebAuthn & MFA]].

## 7. Operations and production
> [!question]- What are the three pillars of observability?
> Logs (events), metrics (aggregates over time), traces (a request across services). OpenTelemetry is the standard way to produce all three. Add structured logs with trace ids and SLO-based alerts.

> [!question]- A service's p99 latency doubled after a deploy. What do you do?
> Check whether to roll back first (mitigate before diagnosing). Then compare dashboards before/after: which endpoints, which dependencies (DB, cache, downstream), resource saturation (CPU, GC, connection pool), and traces for slow requests. Diff the deploy. See [[Debugging Methodology]].

> [!question]- How do you deploy without downtime?
> Rolling or blue-green deploys behind a load balancer with health checks, canary releases with automatic rollback, feature flags, and backward-compatible DB migrations (expand → migrate → contract).

> [!question]- What is graceful shutdown?
> On SIGTERM: stop accepting new requests (fail readiness), finish in-flight requests within a timeout, close DB pools and consumers, flush logs, then exit. Kubernetes sends SIGTERM then SIGKILL after the grace period.

## Senior perspective
- For every answer, add **the trade-off** and **the failure mode**: "X is faster but under a partition we'd…".
- Expect "how would you debug/operate this?" follow-ups. Mention metrics, alerts, runbooks and rollbacks.
- Be ready to discuss a system you built: data model, scaling limits you hit, and what you'd redesign.

## Related
- [[Interviews Index]]
- [[Node.js Index]] · [[Java & Spring Index]] · [[Databases Index]] · [[System Design Index]] · [[System Design Interview Framework]]
- [[Security Index]] · [[API Security]] · [[Backend Security Essentials]]
- [[Interview Process & Preparation Plan]]

## Sources
- [Node.js docs: The Node.js event loop](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- [Node.js docs: Modules, loading ECMAScript modules using require()](https://nodejs.org/api/modules.html#loading-ecmascript-modules-using-require)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444), [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [Spring Framework: Transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110), [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [PostgreSQL docs: Transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- Martin Kleppmann, *Designing Data-Intensive Applications* (O'Reilly)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
