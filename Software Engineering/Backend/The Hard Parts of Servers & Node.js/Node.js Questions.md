---
title: "Node.js Questions"
tags: [backend, backend/node, interview]
aliases: [Node Interview Questions, Node.js Interview Questions, Node Q&A]
updated: 2026-09-27
---
# Node.js Questions

<Note>
**TL;DR**

Interview questions collected while taking [The Hard Parts of Servers & Node.js](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js), with answers corrected and expanded on 2026-09-27. Each answer links to the note that covers the topic in depth. Topic notes have their own question sections too; see [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index).
</Note>

## Runtime & event loop

<Accordion title="Is Node.js single-threaded or multi-threaded?">
Your JavaScript runs on **one main thread** (one call stack, one event loop). Node as a process is multi-threaded: libuv's thread pool (4 threads by default) handles fs, `dns.lookup`, some crypto/zlib; V8 uses background threads for compilation and GC. Network I/O is handled by the OS asynchronously. You can run JS in parallel with `worker_threads`. See [Node.js Event Loop & libuv](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Event%20Loop%20%26%20libuv).
</Accordion>

<Accordion title="What is the difference between multithreading and asynchrony?">
- **Multithreading**: several threads of execution exist at once. On multiple cores they run in **parallel**; on one core the OS time-slices them (**concurrency**). Threads share memory, so you need locks or other synchronisation.
- **Asynchrony**: you start an operation and don't wait for it; the program keeps running and is notified (callback, promise, event) when it completes or fails. A single thread can be asynchronous (Node's model).
See [Concurrency And Parallelism](/Software%20Engineering/Principles/Concurrency%20And%20Parallelism).
</Accordion>

<Accordion title="How can we make Node.js use multiple threads/cores?">
- **`worker_threads`**: run JS in parallel threads inside the same process, each with its own V8 isolate and event loop; communicate with `postMessage` or share memory with `SharedArrayBuffer`. Best for CPU-bound work.
- **`cluster`** (or several processes behind a load balancer / PM2 / Kubernetes replicas): fork one Node **process** per core, each with its own V8 instance, sharing a server port. Best for scaling a stateless HTTP server.
- **`child_process.fork/spawn`**: run separate programs or Node scripts.
See [Node.js Concurrency - Worker Threads, Cluster & Child Processes](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Concurrency%20-%20Worker%20Threads%2C%20Cluster%20%26%20Child%20Processes).
</Accordion>

<Accordion title="What threads is Node.js using behind the scenes?">
The **event loop** thread (main thread, runs your JS) and the **worker pool** (libuv thread pool). Plus V8 helper threads (optimising compiler, GC) and any worker threads you create.
</Accordion>

<Accordion title="What is the event loop?">
The mechanism (conceptually a `while` loop in libuv) that decides which callback runs next. It cycles through phases: **timers → pending callbacks → idle/prepare → poll (I/O) → check (`setImmediate`) → close callbacks**. After every callback it drains `process.nextTick` callbacks, then promise microtasks. It only runs a callback when the call stack is empty. See [Node.js Event Loop & libuv](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Event%20Loop%20%26%20libuv).
</Accordion>

<Accordion title="Have you heard about the V8 engine? Why does Node use it?">
V8 is Google's open-source JS and WebAssembly engine (C++), used in Chrome. Node uses it because it's fast, embeddable and well maintained. It interprets code with **Ignition** (bytecode) and JIT-compiles hot code with **Sparkplug**, **Maglev** and **TurboFan** using runtime type feedback, instead of interpreting everything. See [V8 Engine - Ignition, TurboFan & Garbage Collection](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/V8%20Engine%20-%20Ignition%2C%20TurboFan%20%26%20Garbage%20Collection).
</Accordion>

## Async patterns

<Accordion title="What do you understand about callbacks?">
A function passed to another function to be called later: when an async operation finishes or an event is emitted. Node's classic convention is **error-first**: `(err, result) => {}`.
</Accordion>

<Accordion title="Have you heard about callback hell?">
Deeply nested callbacks when async steps depend on each other, making code hard to read and error handling repetitive. Fixes: **promises and `async/await`** (the standard today), naming and extracting functions (modularisation), `util.promisify` for callback APIs, and control-flow helpers. Generators + promises were a pre-`async/await` workaround.
```js
// callback hell
getUser(id, (e, u) => getOrders(u, (e, o) => getItems(o, (e, i) => send(i))));
// async/await
const u = await getUser(id); const o = await getOrders(u); send(await getItems(o));
```
</Accordion>

<Accordion title="What do you understand about streaming?">
Processing data piece by piece as it arrives instead of loading it all into memory first. Node streams (Readable, Writable, Duplex, Transform) do this with **backpressure** so a fast producer doesn't overwhelm a slow consumer. See [Node.js Hard Parts - Streams & Event Loop Queues](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Hard%20Parts%20-%20Streams%20%26%20Event%20Loop%20Queues).
</Accordion>

<Accordion title="What do you understand about EventEmitter?">
A class (`node:events`) that lets objects emit named events and register listeners (`on`, `once`, `off`, `emit`). It decouples the code that signals something happened from the code that reacts. **Streams, HTTP servers and sockets are built on EventEmitter** (not the other way round). Listeners run **synchronously** in registration order when `emit` is called.
```js
import { EventEmitter } from 'node:events';
const orders = new EventEmitter();
orders.on('created', o => sendEmail(o));
orders.emit('created', { id: 1 });
```
</Accordion>

<Accordion title="What is the difference between EventEmitter and the publish/subscribe pattern?">
Both decouple senders from receivers. EventEmitter is **in-process**: the emitter and listeners share memory, calls are synchronous, and the listeners attach to that specific object. Pub/sub usually goes through a **broker/topic** (Redis, Kafka, a message bus) so publishers and subscribers don't know each other, can live in different services, and delivery is async (possibly durable). EventEmitter is effectively a local observer pattern.
</Accordion>

## Frameworks & architecture

<Accordion title="What frameworks have you used with Node.js?">
Express, Koa and NestJS. Current options also include **Fastify** and **Hono**. See [Building HTTP Servers - Express, Fastify & Hono](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Building%20HTTP%20Servers%20-%20Express%2C%20Fastify%20%26%20Hono).
</Accordion>

<Accordion title="What's the difference between Express and NestJS?">
Express is a **minimal, unopinionated** HTTP framework: routing and middleware, you choose the structure. NestJS is an **opinionated application framework** built on top of Express (or Fastify): TypeScript-first, modules, controllers, providers, a built-in **dependency-injection** container, decorators, guards, pipes and interceptors, inspired by Angular. Nest gives structure for large teams; Express gives flexibility and less abstraction.
</Accordion>

<Accordion title="What is dependency injection?">
Separating **where an object is used** from **where it's created**: a class declares what it needs (usually as constructor parameters) and something else supplies it. It's a form of **Inversion of Control (IoC)**. A DI container (Nest, Spring, Angular) keeps a registry of providers identified by tokens/types and injects them at runtime. Benefits: loose coupling, easy testing with fakes. Service locator is a related but different IoC pattern (the class asks a registry). See [Spring Boot Essentials](/Software%20Engineering/Backend/Java/Spring%20Boot%20Essentials).
</Accordion>

## Data & communication

<Accordion title="What databases are you familiar with?">
MongoDB, PostgreSQL, SQLite, H2. (Node 22.5+ also ships a built-in `node:sqlite` module; see [Node.js 22 to 24+ - What's New](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%2022%20to%2024%2B%20-%20What's%20New).)
</Accordion>

<Accordion title="What's the difference between MongoDB and PostgreSQL?">
PostgreSQL is a **relational** database: tables with a fixed schema, SQL, joins, strong ACID transactions and constraints. MongoDB is a **document** database: JSON-like (BSON) documents with a flexible schema, data often embedded rather than joined, easy horizontal sharding. Postgres also stores JSON (`jsonb`), and MongoDB supports multi-document transactions, so the real choice is about data shape, relationships and consistency needs.
</Accordion>

<Accordion title="Do you have experience with WebSockets?">
Yes: I wrote a singleton that uses WebSockets to connect to Rails **Action Cable**. Node 22+ also has a global browser-compatible `WebSocket` **client** (stable since 22.4); servers still use a library such as `ws`.
</Accordion>

<Accordion title="WebSockets vs a RESTful API: what's the difference?">
REST works over request/response: the client asks, the server answers, the connection has no ongoing state. WebSockets upgrade one HTTP connection to a **persistent, full-duplex** channel so the server can push messages at any time with low overhead. Use WebSockets for chat, live dashboards, multiplayer; REST for CRUD and cacheable resources. Server-Sent Events are a simpler one-way push alternative. See [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time).
</Accordion>

<Accordion title="Do you have experience with GraphQL? What is it?">
GraphQL is a **query language and runtime for APIs**: a typed schema describes the data, and clients ask for exactly the fields they need in one request to a single endpoint. It addresses REST's over-fetching/under-fetching and multiple round trips, at the cost of harder HTTP caching, query-cost control and N+1 resolver problems (solved with DataLoader). See [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST).
</Accordion>

## Related
- [The Hard Parts of Servers & Node.js](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js) · [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index)
- [Node.js Event Loop & libuv](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Event%20Loop%20%26%20libuv) · [Node.js Concurrency - Worker Threads, Cluster & Child Processes](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Concurrency%20-%20Worker%20Threads%2C%20Cluster%20%26%20Child%20Processes) · [Building HTTP Servers - Express, Fastify & Hono](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Building%20HTTP%20Servers%20-%20Express%2C%20Fastify%20%26%20Hono)
- [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions)

## Sources
- [Node.js: The event loop, timers and process.nextTick](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- [Node.js docs: Events](https://nodejs.org/api/events.html)
- [Node.js docs: Worker threads](https://nodejs.org/api/worker_threads.html)
- [NestJS docs: Introduction](https://docs.nestjs.com/)
- [GraphQL: Introduction](https://graphql.org/learn/)
- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
