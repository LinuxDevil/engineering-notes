---
title: "The Hard Parts of Servers & Node.js"
tags: [backend, backend/node, course]
aliases: [Servers & Node.js Hard Parts, Node Hard Parts, Hard Parts of Node]
updated: 2026-09-27
---
# The Hard Parts of Servers & Node.js

<Note>
**TL;DR**

Course notes for Will Sentance's *The Hard Parts of Servers & Node.js* (Frontend Masters). The course explains what really happens when JavaScript talks to the network and the file system: JS saves and runs code, Node's C++ (via **libuv**) does the I/O, and the **event loop** decides when your auto-run callbacks go on the call stack. For the topic-by-topic reference notes see [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index).
</Note>

Course link: [frontendmasters.com/courses/servers-node-js](https://frontendmasters.com/courses/servers-node-js/introduction/)

## Course notes (in course order)
1. [Introduction to Node.js](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Introduction%20to%20Node.js): what Node is, V8 + libuv + C++ bindings, opening a socket with `http.createServer`
2. [Using Node API’s](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Using%20Node%20API%E2%80%99s): how Node auto-runs your handler and auto-fills `req`/`res`
3. [Node with HTTP](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node%20with%20HTTP): routing, the request/response cycle, middleware
4. [Node.js Hard Parts - File System](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Hard%20Parts%20-%20File%20System): `fs`, the thread pool, the call stack
5. [Node.js Hard Parts - Streams & Event Loop Queues](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Hard%20Parts%20-%20Streams%20%26%20Event%20Loop%20Queues): streams, chunks, the six queues and the rules the event loop follows
6. [Node JS Under the hood!](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node%20JS%20Under%20the%20hood!): runtime anatomy, libuv, event loop, micro/macrotasks
7. [V8 Engine - Ignition, TurboFan & Garbage Collection](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/V8%20Engine%20-%20Ignition%2C%20TurboFan%20%26%20Garbage%20Collection): the V8 half of the old "under the hood" note (split out)
8. [Node.js Questions](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Questions): interview Q&A collected alongside the course

## Where to go next
- [Node.js Event Loop & libuv](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Event%20Loop%20%26%20libuv): the precise, up-to-date phase model
- [Building HTTP Servers - Express, Fastify & Hono](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Building%20HTTP%20Servers%20-%20Express%2C%20Fastify%20%26%20Hono)
- [Node.js Concurrency - Worker Threads, Cluster & Child Processes](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Concurrency%20-%20Worker%20Threads%2C%20Cluster%20%26%20Child%20Processes)

## Related
- [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index)
- [JavaScript the hard parts v2](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/JavaScript%20the%20hard%20parts%20v2) · [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks) · [Closures & Memory Model](/Software%20Engineering/Frontend/01%20JavaScript/Closures%20%26%20Memory%20Model)

## Sources
- Frontend Masters, *The Hard Parts of Servers & Node.js* (Will Sentance): https://frontendmasters.com/courses/servers-node-js/
- Node.js docs: https://nodejs.org/docs/latest/api/
