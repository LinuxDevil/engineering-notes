---
title: "Node.js Hard Parts - Streams & Event Loop Queues"
tags: [backend, backend/node, course]
aliases: [Node Streams, Streams, highWaterMark, Backpressure in Node, Node queues]
updated: 2026-09-27
---
# Node.js Hard Parts - Streams & Event Loop Queues

<Note>
**TL;DR**

A **stream** processes data chunk by chunk (64 KiB by default for file streams) instead of loading it all into memory. Node emits a `data` event per chunk and auto-runs your handler, but only when the **call stack is empty**; until then the callback waits in a **queue**. Node has several queues (nextTick, promise microtasks, timers, I/O, check/`setImmediate`, close) and the **event loop** follows fixed rules to pick the next one. Use `stream.pipeline()` or async iteration so **backpressure** is handled for you.
</Note>

*Part of [The Hard Parts of Servers & Node.js](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js) (course notes). Renamed from "Streams". The precise, current phase model is in [Node.js Event Loop & libuv](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Event%20Loop%20%26%20libuv).*

## Streams: data in chunks

- Data comes from the network (or disk) as **chunks of buffers**.
- The chunk size is set by the **highWaterMark**. For `fs.createReadStream` it is **64 KiB**; you can change it.
- A stream is chunks of data, like buckets of water. Each time a chunk arrives Node **broadcasts** a `data` event, and the function we attached auto-runs with the chunk.

![Screenshot_2023-05-05_at_5.44.26_PM](/Attachments/Screenshot_2023-05-05_at_5.44.26_PM.png)

- When the file stream opens, Node broadcasts `open`.

![Screenshot_2023-05-05_at_6.07.46_PM](/Attachments/Screenshot_2023-05-05_at_6.07.46_PM.png)

- When all the data has been read, Node emits `end`, and once the underlying file descriptor is released, `close`.

![Screenshot_2023-05-05_at_6.06.13_PM](/Attachments/Screenshot_2023-05-05_at_6.06.13_PM.png)

```js
const fs = require('node:fs');

let cleanedTweets = 0;
function doOnNewBatch(chunk) {
  cleanedTweets += chunk.toString().split('\n').length; // work on 64 KiB at a time
}

const stream = fs.createReadStream('tweets.txt', { highWaterMark: 64 * 1024 });
stream.on('open', () => console.log('opened'));
stream.on('data', doOnNewBatch);
stream.on('end', () => console.log('done', cleanedTweets));
stream.on('close', () => console.log('fd closed'));
stream.on('error', err => console.error(err));
```

## "A new batch arrived while `doOnNewBatch` is still running. Now what?"

First, some definitions.

**The call stack**: JavaScript keeps track of what function is being run and where it was run from. Whenever a function is to be run, it's added to the call stack.

**The event loop**: "just a while loop that checks the queues: while there's stuff to run, go check the queues (following the rules below)." It determines which function/code runs next.

When all global code has finished and only the global context is left on the call stack, libuv and Node check whether anything is left to do. If so, the event loop takes a function from one of the queues, puts it on the call stack and creates a brand new execution context for it.

![Screenshot_2023-05-05_at_9.09.37_PM](/Attachments/Screenshot_2023-05-05_at_9.09.37_PM.png)

So the second batch's `data` callback simply **waits in a queue** until the first `doOnNewBatch` returns and the stack is empty. JS never interrupts a running function.

### The queues Node (with libuv) sets up for us

1. **Callback queue (general idea)**: functions delayed from running (and run automatically by Node) are added when the background Node task completes, or when there's activity like a request. When a new batch arrives and Node broadcasts `data`, `doOnNewBatch` goes here and waits.
2. **Timer queue** (really a **min-heap** ordered by expiry): callbacks from `setTimeout`/`setInterval` whose time has elapsed.
3. **I/O callback queue**: callbacks triggered by the file system, network sockets, etc. once their status is "done". Every auto-run function that involves I/O shares this queue.
4. **Check queue**: functions scheduled with `setImmediate`. It runs right after I/O polling, so it's a way to say "run this after the I/O callbacks of this loop iteration".
5. **Microtask queue**: highest priority, actually **two** queues:
   1. the `process.nextTick()` queue (drained first),
   2. the promise queue (`.then`/`await` continuations, `queueMicrotask`).
6. **Close queue**: callbacks for `close` events (e.g. `socket.on('close')`).

### Rules for the automatic execution of JS code by Node

![Screenshot_2023-05-05_at_6.55.37_PM](/Attachments/Screenshot_2023-05-05_at_6.55.37_PM.png)

1. Hold each deferred (auto-run) function in one of the task queues when the Node background API "completes".
2. Add the function to the call stack (i.e. execute it) **only** when the call stack is totally empty (the event loop checks this).
3. Prioritise the **timer** queue over the **I/O** queue over the **check** (`setImmediate`) queue, then close callbacks. Between **every** callback, drain the microtask queues (nextTick first, then promises).

**Promises** are an alternative way of setting up auto-run functions: instead of Node auto-running a callback, the promise object's value gets updated in the background and that triggers the `.then` function, which is added to the **microtask** queue.

```js
setTimeout(() => console.log('timer'), 0);
setImmediate(() => console.log('check'));
fs.readFile(__filename, () => {
  console.log('io');
  setTimeout(() => console.log('timer from io'), 0);
  setImmediate(() => console.log('check from io'));   // always before 'timer from io'
});
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');
// sync, nextTick, promise, then timer/check (order may vary from main module),
// io, check from io, timer from io
```

<Note>
**Correction to the simple model**

Since libuv 1.45 (Node 20) timers are processed after the poll phase in each iteration (plus once before the loop starts). The practical rules above still hold; details in [Node.js Event Loop & libuv](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Event%20Loop%20%26%20libuv).
</Note>

## Modern stream usage (beyond the course)

The four stream types: **Readable**, **Writable**, **Duplex** (both, e.g. a TCP socket), **Transform** (duplex that changes data, e.g. gzip).

**Backpressure**: if the writer is slower than the reader, chunks pile up in memory. `writable.write()` returns `false` when its buffer passes its highWaterMark; you should pause until `drain`. `pipeline` does this for you and also destroys all streams on error.

```js
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';

await pipeline(
  createReadStream('access.log'),
  createGzip(),
  createWriteStream('access.log.gz'),
);
```

```js
// async iteration: backpressure handled by await
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

const rl = createInterface({ input: createReadStream('tweets.txt') });
for await (const line of rl) {
  await saveTweet(line);    // next line is not read until this resolves
}
```

Streaming an HTTP response instead of buffering it:

```js
http.createServer((req, res) => {
  res.setHeader('Content-Type', 'video/mp4');
  pipeline(createReadStream('movie.mp4'), res).catch(() => {});
}).listen(3000);
```

<Info>
**🆕 2025–2026**

- **Node 22** (Apr 2024) raised the default highWaterMark for generic streams from **16 KiB to 64 KiB** (`stream.setDefaultHighWaterMark()` to change it). File streams were already 64 KiB.
- **Node 26.0.0** (2026-05-05) removed the legacy internal modules `_stream_readable`, `_stream_writable`, `_stream_duplex`, `_stream_transform`, `_stream_passthrough`, `_stream_wrap`; import from `node:stream` instead.
- Web Streams (`ReadableStream`, `WritableStream`) are global and interoperate via `Readable.fromWeb()` / `Readable.toWeb()`; `fetch` bodies are web streams.
</Info>

## Pitfalls
- `.pipe()` does not forward errors or clean up; prefer `pipeline`.
- Accumulating all chunks into a string defeats the purpose of streaming for big payloads.
- Recursive `process.nextTick` starves the loop: I/O never gets a turn.

## Interview questions

<Accordion title="What is a stream and why use one?">
An abstraction for data that arrives or is produced over time in chunks. It keeps memory bounded (you never hold the whole file), lets you start processing sooner, and composes (read → transform → write).
</Accordion>

<Accordion title="What is backpressure and how does Node handle it?">
When a consumer is slower than a producer, buffered data grows. Writable streams signal this by returning `false` from `write()`; the producer should pause until `drain`. `pipeline()` and `for await` handle it automatically.
</Accordion>

<Accordion title="If a `data` event fires while your handler is still running, what happens?">
Nothing interrupts it. The next callback waits in the queue until the call stack is empty; JS runs one function to completion at a time.
</Accordion>

<Accordion title="Order of `process.nextTick`, promises, `setTimeout(0)` and `setImmediate`?">
Sync code, then nextTick queue, then promise microtasks, then timers vs check. From the main module timeout vs immediate order is not guaranteed; inside an I/O callback `setImmediate` always runs first.
</Accordion>

## Related
- [The Hard Parts of Servers & Node.js](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js) · [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index)
- [Node.js Event Loop & libuv](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Event%20Loop%20%26%20libuv) · [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks) · [Node.js Hard Parts - File System](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Hard%20Parts%20-%20File%20System)
- [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation)

## Sources
- [Node.js docs: Stream](https://nodejs.org/api/stream.html)
- [Node.js: Backpressuring in streams](https://nodejs.org/en/learn/modules/backpressuring-in-streams)
- [Node.js: The event loop, timers and process.nextTick](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- [Node.js 22 announcement (highWaterMark 64 KiB)](https://nodejs.org/en/blog/announcements/v22-release-announce)
- [Node.js 26.0.0 release notes](https://nodejs.org/en/blog/release/v26.0.0)
- Frontend Masters, *The Hard Parts of Servers & Node.js* (Will Sentance)
