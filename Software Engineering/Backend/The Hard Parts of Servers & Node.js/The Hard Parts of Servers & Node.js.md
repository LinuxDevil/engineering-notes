---
tags: [backend, backend/node, course]
aliases: [Servers & Node.js Hard Parts, Node Hard Parts, Hard Parts of Node]
updated: 2026-09-27
---
# The Hard Parts of Servers & Node.js

> [!tldr] TL;DR
> Course notes for Will Sentance's *The Hard Parts of Servers & Node.js* (Frontend Masters). The course explains what really happens when JavaScript talks to the network and the file system: JS saves and runs code, Node's C++ (via **libuv**) does the I/O, and the **event loop** decides when your auto-run callbacks go on the call stack. For the topic-by-topic reference notes see [[Node.js Index]].

Course link: [frontendmasters.com/courses/servers-node-js](https://frontendmasters.com/courses/servers-node-js/introduction/)

## Course notes (in course order)
1. [[Introduction to Node.js]]: what Node is, V8 + libuv + C++ bindings, opening a socket with `http.createServer`
2. [[Using Node API’s]]: how Node auto-runs your handler and auto-fills `req`/`res`
3. [[Node with HTTP]]: routing, the request/response cycle, middleware
4. [[Node.js Hard Parts - File System]]: `fs`, the thread pool, the call stack
5. [[Node.js Hard Parts - Streams & Event Loop Queues]]: streams, chunks, the six queues and the rules the event loop follows
6. [[Node JS Under the hood!]]: runtime anatomy, libuv, event loop, micro/macrotasks
7. [[V8 Engine - Ignition, TurboFan & Garbage Collection]]: the V8 half of the old "under the hood" note (split out)
8. [[Node.js Questions]]: interview Q&A collected alongside the course

## Where to go next
- [[Node.js Event Loop & libuv]]: the precise, up-to-date phase model
- [[Building HTTP Servers - Express, Fastify & Hono]]
- [[Node.js Concurrency - Worker Threads, Cluster & Child Processes]]

## Related
- [[Node.js Index]]
- [[JavaScript the hard parts v2]] · [[Event Loop - Microtasks & Macrotasks]] · [[Closures & Memory Model]]

## Sources
- Frontend Masters, *The Hard Parts of Servers & Node.js* (Will Sentance): https://frontendmasters.com/courses/servers-node-js/
- Node.js docs: https://nodejs.org/docs/latest/api/
