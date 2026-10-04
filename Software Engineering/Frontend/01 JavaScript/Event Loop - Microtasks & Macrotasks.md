---
title: "Event Loop - Microtasks & Macrotasks"
tags: [frontend, frontend/javascript]
aliases: [Event Loop, Microtasks, Macrotasks, Task Queue, Call Stack and Event Loop]
updated: 2026-09-27
---
# Event Loop - Microtasks & Macrotasks

<Note>
**TL;DR**

JavaScript runs on **one call stack** per thread. Async work (timers, network, events) is handled by the host (browser Web APIs or Node's libuv), which queues callbacks. The **event loop** runs **one task (macrotask)**, then drains **all microtasks** (promise reactions, `queueMicrotask`, `MutationObserver`), then, if it's time for a frame, lets the browser **render** (`requestAnimationFrame` callbacks → style → layout → paint). Rendering happens *between tasks, after the microtask checkpoint*, never "inside" the microtask step.
</Note>

## The pieces

![Screenshot_2023-08-08_at_3.44.28_PM](/Attachments/Screenshot_2023-08-08_at_3.44.28_PM.png)

![Screenshot_2023-08-08_at_3.26.42_PM](/Attachments/Screenshot_2023-08-08_at_3.26.42_PM.png)

![Screenshot_2023-08-08_at_3.27.09_PM](/Attachments/Screenshot_2023-08-08_at_3.27.09_PM.png)

### Call stack
![Screenshot_2023-08-08_at_3.27.27_PM](/Attachments/Screenshot_2023-08-08_at_3.27.27_PM.png)

- The mechanism the JS engine (e.g. V8) uses to keep track of **where the thread is** in the program: which function is running and who called it.
- A **stack** (LIFO: last in, first out). Calling a function pushes a frame (its [execution context](/Software%20Engineering/Frontend/01%20JavaScript/Execution%20Context%20%26%20Call%20Stack): parameters, local variables); returning pops it.
- **JS has a single call stack** (per thread/agent), so only one piece of JS runs at a time. While the stack is busy, nothing else (no clicks, no rendering) can happen. That's why long tasks freeze the UI.

### Web APIs (host APIs)
![Screenshot_2023-08-08_at_3.27.56_PM](/Attachments/Screenshot_2023-08-08_at_3.27.56_PM.png)

- Timers, `fetch`/XHR, DOM events, etc. are **not part of the JS language**; they are provided by the browser (or by Node).
- When you call `setTimeout`, JS hands the timer to the browser and immediately moves on, freeing the call stack. When the timer fires (or the request finishes), the browser queues the callback.

### Task queue (macrotask / callback queue)
- When the browser finishes background work (timer, network response, user event), it puts the corresponding callback into a **task queue**.
- Examples of **tasks**: running the initial `<script>`, `setTimeout`/`setInterval` callbacks, DOM event dispatch (click, input), `MessageChannel`/`postMessage`, I/O callbacks. Browsers can have several task sources and pick between them (e.g. user input may be prioritised).
- Note: a `fetch()` **promise** resolves via a *microtask* once the network task that completes it runs; the `.then` callbacks are microtasks, not tasks.

### Microtask queue (job queue)
![Screenshot_2023-08-08_at_3.28.39_PM](/Attachments/Screenshot_2023-08-08_at_3.28.39_PM.png)

- Higher priority than tasks. At every **microtask checkpoint**, i.e. whenever the call stack becomes empty (after each task, after each event listener callback, after each rAF callback), the engine runs **all** queued microtasks, one by one, **until the queue is empty**.
- If a microtask queues another microtask, it also runs **before** anything else (no rendering, no next task). This means an infinite microtask chain starves rendering just like an infinite loop.
- Sources: `Promise.then/catch/finally` reactions, `await` continuations, `queueMicrotask(fn)`, `MutationObserver` callbacks.

### Event loop
![Screenshot_2023-08-08_at_3.28.59_PM](/Attachments/Screenshot_2023-08-08_at_3.28.59_PM.png)

- Its job: when the call stack is empty, take the **oldest task** from a task queue and push its callback onto the stack; after it finishes, drain microtasks; then maybe render; repeat forever.
- This is what keeps JS **non-blocking**: long waits happen in the host, and JS only runs short callbacks.

![Screenshot_2023-08-08_at_3.28.18_PM](/Attachments/Screenshot_2023-08-08_at_3.28.18_PM.png)

## One turn of the browser event loop

{/* diagram source: /diagrams/diagram-209.mmd */}
![diagram](/diagrams/diagram-209.svg)

**Corrected priority order** (the original note said "the rendering happens here" in the microtask step, which is wrong):

1. **Call stack**: run the current synchronous code (the current task) until the stack is empty.
2. **Microtasks**: drain the whole microtask queue (and any microtasks those microtasks add).
3. **Render (if due)**: rAF callbacks, then style/layout/paint. The browser decides; it usually matches the display refresh rate and skips frames when nothing changed or the tab is hidden.
4. **Next task**: pick the next macrotask and start over.

<Warning>
**Why "rendering in the microtask step" is wrong**

Microtasks block rendering: the browser cannot paint until the microtask queue is empty. Rendering is a separate step of the event loop that happens *between tasks*, after the microtask checkpoint. A consequence: DOM changes made in a promise callback are painted together with the synchronous changes of the same task (no flicker in between), while a change made in `setTimeout(fn, 0)` might be painted in a later frame.
</Warning>

## Worked example: sync vs microtask vs macrotask

```js
console.log('Script Start');            // 1: runs on the call stack

setTimeout(() => {
  console.log('setTimeout callback');   // 4: queued as a macrotask
}, 0);

Promise.resolve().then(() => {
  console.log('Promise resolved');      // 3: queued as a microtask
});

console.log('Script End');              // 2: runs on the call stack
```

Output: `Script Start`, `Script End`, `Promise resolved`, `setTimeout callback`.

How the script gets fired:

```text
|-------------------|   |------------------|   |------------|   |------------|
|   Call Stack      |   |  Web APIs        |   |  Micro     |   |   Macro    |
|-------------------|   |  (host)          |   |  (Job)     |   |   (Task)   |
|                   |   |------------------|   |  Queue     |   |   Queue    |
|   console.log     |   |  setTimeout      |   |------------|   |------------|
|   (Script Start)  |   |  timer (0 ms)    |   | Promise    |   | setTimeout |
|-------------------|   |------------------|   | .then()    |   | callback   |
|   console.log     |   |                  |   |------------|   |------------|
|   (Script End)    |   |                  |   |            |   |            |
|-------------------|   |------------------|   |------------|   |------------|

                  |-------------------|
                  |   Event Loop      |
                  |  Constantly       |
                  |  monitoring       |
                  |-------------------|
```

1. The call stack executes `console.log('Script Start')` directly.
2. `setTimeout` is encountered; its callback is handed off to the host (Web APIs), which takes care of the timer.
3. While the timer is counting (even at 0 ms; browsers also clamp deeply nested timers to ≥4 ms), the stack moves on and runs `Promise.resolve()`, placing its `.then()` callback in the **microtask queue**.
4. The stack executes `console.log('Script End')`. The script (itself a task) is done and the stack is empty.
5. **Microtask checkpoint**: the event loop drains the microtask queue and runs the `.then()` callback (`Promise resolved`).
6. (A render may happen here if a frame is due.)
7. The event loop picks the next **task**: the timer callback (`setTimeout callback`).

{/* diagram source: /diagrams/diagram-210.mmd */}
![diagram](/diagrams/diagram-210.svg)

<Info>
**Harder ordering puzzle**

```js
console.log(1);
setTimeout(() => console.log(2));
queueMicrotask(() => console.log(3));
Promise.resolve()
  .then(() => { console.log(4); setTimeout(() => console.log(5)); })
  .then(() => console.log(6));
(async () => {
  console.log(7);          // async fn body runs synchronously until the first await
  await null;
  console.log(8);          // continuation is a microtask
})();
requestAnimationFrame(() => console.log(9));
console.log(10);
```
Sync: `1 7 10`. Microtasks in FIFO order: `3 4 8 6` (6 is queued only when the first `.then` finishes). Then tasks/frames: `2` and `5` are timers; `9` runs before the next paint. Typical output: `1 7 10 3 4 8 6 2 5 9` or `... 2 9 5` or `... 9 2 5`. **rAF relative to timers is not deterministic**, it depends on when the next frame is due. Never rely on it.
</Info>

## Scheduling tools: which queue does what?

| API | Queue | Runs | Use it for |
|---|---|---|---|
| `Promise.then`, `await` continuation | microtask | ASAP after the current stack empties | Async flow control |
| `queueMicrotask(fn)` | microtask | Same as above, without allocating a promise | Batching work "at the end of this tick"; errors are reported, not swallowed |
| `MutationObserver` callback | microtask | After DOM mutations, batched | Reacting to DOM changes |
| `setTimeout(fn, 0)` / `setInterval` | task | Next available task, ≥ delay | Deferring work, letting the browser breathe (but may be clamped) |
| `MessageChannel.postMessage` | task | Next task, no clamping | Fast "yield" (React's scheduler uses it) |
| `requestAnimationFrame(fn)` | frame callback | Right **before the next paint** | Visual updates, measuring+animating in sync with the display |
| `requestIdleCallback(fn)` | idle | When the browser has spare time in a frame | Low-priority work (analytics, prefetch) |
| `scheduler.postTask(fn, {priority})` | prioritised task | `user-blocking`/`user-visible`/`background` | Prioritising tasks |
| `await scheduler.yield()` | continuation task (front of the queue) | Lets input/rendering happen, then resumes **before** other queued tasks | Breaking up long tasks to improve INP |

<Info>
**Breaking up a long task (INP)**

```js
async function processAll(items) {
  for (const item of items) {
    process(item);
    // give the browser a chance to handle input and paint
    if ('scheduler' in globalThis && 'yield' in scheduler) {
      await scheduler.yield();
    } else {
      await new Promise(r => setTimeout(r, 0));
    }
  }
}
```
Awaiting a resolved promise (`await null`) does **not** yield to rendering, it only queues a microtask. You need a *task* boundary (or `scheduler.yield`). See [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals) and [Performance Optimization Playbook](/Software%20Engineering/Frontend/10%20Performance/Performance%20Optimization%20Playbook).
</Info>

<Info>
**rAF for smooth visual updates**

```js
let scheduled = false;
window.addEventListener('scroll', () => {
  if (scheduled) return;
  scheduled = true;
  requestAnimationFrame(() => {       // once per frame, just before paint
    header.classList.toggle('compact', window.scrollY > 80);
    scheduled = false;
  });
});
```
</Info>

<Info>
**🆕 2025–2026**

- **Scheduler API** (`scheduler.postTask`, `scheduler.yield`): Baseline **Limited**, Chrome 129 and Firefox 142, no Safari. Feature-detect and fall back to `setTimeout`/`MessageChannel`.
- **Long Animation Frames (LoAF)** is Chromium-only (Chromium 123) and the recommended way to attribute slow INP to specific scripts.
- Core Web Vitals thresholds are unchanged: **INP ≤ 200 ms**.
</Info>

## Node.js vs browser (brief)

Node uses libuv; its loop runs in **phases**, and between each callback it drains two queues: the `process.nextTick` queue first, then the promise microtask queue.

{/* diagram source: /diagrams/diagram-211.mmd */}
![diagram](/diagrams/diagram-211.svg)

| | Browser | Node.js |
|---|---|---|
| Macrotask sources | Task sources (timers, events, messaging, network) | libuv phases (timers, poll/I/O, check, close) |
| Extra microtask-like queue | none | `process.nextTick` (runs **before** promise microtasks) |
| `setImmediate` | not standard | runs in the **check** phase, after poll |
| Rendering step | yes (rAF, style, layout, paint) | none |
| `setTimeout(0)` vs `setImmediate` | n/a | order is non-deterministic from the main module; inside an I/O callback `setImmediate` always wins |

```js
// Node
setTimeout(() => console.log('timeout'));
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
// nextTick, promise, then timeout/immediate (order of these two may vary)
```

## Web Workers: more threads, more loops

Each worker has **its own** call stack, heap and event loop, and communicates with the main thread through `postMessage` (a task). Heavy computation belongs there. See [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs).

<Warning>
**Common pitfalls**

- "`setTimeout(fn, 0)` runs immediately": no, it runs after the current task, after all microtasks, and possibly after a render; it's also clamped.
- Recursive microtasks (`function loop(){ queueMicrotask(loop) }`) freeze the page; recursive `setTimeout` does not.
- `await` inside a tight loop doesn't make it "async enough" for the UI; you need a task boundary.
- Mixing up "the promise callback is a microtask" with "the promise's work is async": the **executor** of `new Promise(executor)` runs **synchronously**.
</Warning>

## Interview questions

<Accordion title="Explain the event loop.">
JS has a single call stack. Async operations are handed to the host (Web APIs / libuv), which queues callbacks when done. The event loop takes one task from the task queue when the stack is empty, runs it to completion, then drains all microtasks, then (in browsers) optionally renders a frame, and repeats.
</Accordion>

<Accordion title="What are microtasks and macrotasks and how are they prioritised?">
Macrotasks (tasks): script execution, timers, events, `postMessage`, I/O. Microtasks: promise reactions, `await` continuations, `queueMicrotask`, `MutationObserver`. After every task (and whenever the stack empties) the entire microtask queue is drained before the next task or render, so microtasks always go first.
</Accordion>

<Accordion title="Where does rendering happen in the event loop?">
In a separate "update the rendering" step that runs between tasks, after the microtask checkpoint, roughly once per display frame when something changed. It runs rAF callbacks, then style, layout and paint. It never happens while microtasks are pending.
</Accordion>

<Accordion title="What is `requestAnimationFrame` for and when does it run?">
It schedules a callback right before the next paint, so visual updates happen once per frame and in sync with the display. It's paused in background tabs. Use it for animations and for batching DOM reads/writes.
</Accordion>

<Accordion title="How would you keep a long computation from blocking the UI?">
Split it into chunks and yield between them with a task boundary (`scheduler.yield()` with a `setTimeout`/`MessageChannel` fallback), move it to a Web Worker, or reduce the work (virtualization, memoization). Awaiting a resolved promise doesn't help because microtasks still block rendering.
</Accordion>

<Accordion title="What's the difference between `process.nextTick`, `setImmediate` and `setTimeout` in Node?">
`nextTick` callbacks run before promise microtasks, right after the current operation. `setImmediate` runs in the check phase after I/O polling. `setTimeout` runs in the timers phase once the delay has passed. Inside an I/O callback, `setImmediate` always fires before a 0 ms timeout.
</Accordion>

<Accordion title="Predict the output of `setTimeout` + `Promise.then` + sync logs.">
Sync logs first, then all promise callbacks (microtasks), then the timeout callback (task).
</Accordion>

## Related
- [Execution Context & Call Stack](/Software%20Engineering/Frontend/01%20JavaScript/Execution%20Context%20%26%20Call%20Stack)
- [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation)
- [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path)
- [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs)
- [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals)
- [Output Questions & Tricky Snippets](/Software%20Engineering/Frontend/17%20Interview%20Prep/Output%20Questions%20%26%20Tricky%20Snippets)
- [Asynchronous JavaScript](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/Asynchronous%20JavaScript)
- [JavaScript the hard parts v2](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/JavaScript%20the%20hard%20parts%20v2)

## Sources
- [MDN: The event loop / JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop)
- [MDN: Using microtasks (queueMicrotask)](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)
- [HTML Standard: event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)
- [Jake Archibald: Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
- [web.dev: Optimize long tasks (scheduler.yield)](https://web.dev/articles/optimize-long-tasks)
- [Node.js: The event loop, timers and process.nextTick](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- [web.dev: Web Vitals](https://web.dev/articles/vitals)
