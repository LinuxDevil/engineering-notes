---
title: "Execution Context & Call Stack"
tags: [frontend, frontend/javascript]
aliases: [Execution Context, Call Stack, Thread of Execution, Global Execution Context]
updated: 2026-09-27
---
# Execution Context & Call Stack

<Note>
**TL;DR**

JavaScript runs code with **one thread of execution** (go line by line) plus **memory** (store data as you go). The pair is an **execution context**. A new one is created for the global code and for **every function call**, and the engine tracks which one is running with the **call stack** (LIFO). Almost every "how does JS work" topic (hoisting, scope, closures, `this`, the event loop) is a detail of this model.
</Note>

## Thread of execution + memory

![Screenshot_2023-08-26_at_6.03.43_PM](/Attachments/Screenshot_2023-08-26_at_6.03.43_PM.png)

- **Thread of execution**: the ability to go through the code, line by line, and execute it.
- **Memory (variable environment)**: a place to save data (labels → values) while we go.
- Combining the two gives an **execution context**: *the environment in which a piece of code is evaluated and executed*.
- JavaScript is **single-threaded** (one command at a time) and **synchronously executed** (each line runs in the order it appears). Asynchronous behaviour comes from the host (browser/Node), not from the language running two things at once; see [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks).

<Info>
**Walkthrough: `multiplyBy2` (from [[Principles]])**

```js
const num = 3;
function multiplyBy2(inputNumber) {
  const result = inputNumber * 2;
  return result;
}
const output = multiplyBy2(num);
const newOutput = multiplyBy2(10);
```
```text
---------------- Global Memory ----------------
num: 3
multiplyBy2: <Function>          (whole definition stored)
output: <uninitialized>

  output = multiplyBy2(3)  --> create a new execution context
    ---- Local Memory ----
    inputNumber: 3            (parameter = label, 3 = argument)
    result: 6
    return result  --> look up `result`, hand 6 back as the value of output

output: 6
newOutput: <uninitialized>

  newOutput = multiplyBy2(10) --> create a new execution context
    ---- Local Memory ----
    inputNumber: 10
    result: 20
    return result

newOutput: 20
-----------------------------------------------
```
Each call gets a **brand-new** local memory; nothing from the first call survives into the second (unless a closure keeps it, see [Closures & Memory Model](/Software%20Engineering/Frontend/01%20JavaScript/Closures%20%26%20Memory%20Model)).
</Info>

## What is inside an execution context

The owner's short model (good enough for interviews):

| Component | What it holds |
|---|---|
| **Variable environment** | Variables, function declarations and the arguments of the function |
| **Scope chain** | Ordered access to all variables the context can reach: its own environment, then its parent's (outer) environment, and so on up to global |
| **`this` binding** | The object the code is currently "about" (decided by *how* the function was called) |

The ECMAScript spec is a bit more precise. An execution context has:

- **LexicalEnvironment**: where `let`, `const`, `class` and function declarations (and block scopes) live. Each environment has an **outer** reference, which is what forms the scope chain. See [Scope & Lexical Environment](/Software%20Engineering/Frontend/01%20JavaScript/Scope%20%26%20Lexical%20Environment).
- **VariableEnvironment**: where `var` declarations live (for a function it starts out as the same environment).
- **PrivateEnvironment**: the `#private` names of the enclosing class.
- The **`this` binding** lives on the function's environment record (arrow functions don't have one, they look it up lexically). See [this, call, apply & bind](/Software%20Engineering/Frontend/01%20JavaScript/this%2C%20call%2C%20apply%20%26%20bind).
- Bookkeeping: the running **Function** object, the **Realm** (which set of globals: `Array`, `Object`, ... of this window/iframe), and the **ScriptOrModule**.

## Types of execution context

| Type | Created when | Notes |
|---|---|---|
| **Global** | Once, when a script starts | Runs all top-level code. In a classic browser script `this` is `window` (`globalThis`) |
| **Module** | Once per ES module | Top-level `this` is `undefined`; top-level variables are module-scoped. See [Modules (ESM)](/Software%20Engineering/Frontend/01%20JavaScript/Modules%20(ESM)) |
| **Function** | **Every time a function is invoked** | Its details (especially `this`) depend on *how* it is called |
| **Eval** | Code passed to `eval()` | Evaluates a string as code. Never use `eval`: security (XSS), performance (defeats optimisation) and CSP problems |

## Two phases: creation and execution

Every execution context goes through two phases. This is the real mechanism behind "hoisting" (see [Hoisting, TDZ & var-let-const](/Software%20Engineering/Frontend/01%20JavaScript/Hoisting%2C%20TDZ%20%26%20var-let-const)).

{/* diagram source: /diagrams/diagram-212.mmd */}
![diagram](/diagrams/diagram-212.svg)

## The call stack

![Screenshot_2023-08-08_at_3.27.27_PM](/Attachments/Screenshot_2023-08-08_at_3.27.27_PM.png)

- The call stack is the mechanism engines like V8 use to track **where the thread is** in the program.
- It is a **stack** (LIFO, last in first out): calling a function **pushes** its execution context on top; returning (or reaching the end) **pops** it off.
- JS has a **single call stack** per agent (a tab's main thread, or each worker, has its own).
- The global context sits at the bottom and stays there until the script finishes.

<Info>
**Call stack for `multiplyBy2` (from [[Principles]])**

```text
call multiplyBy2(3)     return              call multiplyBy2(10)    return
|                |     |              |    |                 |    |              |
| multiplyBy2(3) |     |              |    | multiplyBy2(10) |    |              |
|____global()____|     |__global()____|    |____global()_____|    |__global()____|
```
</Info>

<Info>
**Nested calls and the stack trace**

```js
function a() { b(); }
function b() { c(); }
function c() { console.trace('where am I?'); }
a();
// console.trace prints the stack top-down:
// c  <- b  <- a  <- (global/anonymous)
```
{/* diagram source: /diagrams/diagram-213.mmd */}
![diagram](/diagrams/diagram-213.svg)
An `Error`'s `.stack` is a snapshot of this same stack at the moment it was created.
</Info>

### Stack overflow

The stack has a finite size. Recursion without a base case keeps pushing frames until the engine throws `RangeError: Maximum call stack size exceeded`.

```js
function countDown(n) {
  if (n === 0) return;     // base case, remove it and the stack overflows
  countDown(n - 1);
}
countDown(10_000);          // fine
```

JS engines (except Safari's JSC in strict mode) do **not** implement proper tail calls, so deep recursion over large inputs should be rewritten as a loop or an explicit stack.

### Blocking the stack

While anything is on the stack, the browser cannot run other callbacks **or paint**. A long synchronous function freezes the UI (bad INP, see [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals)). Callbacks from timers, network and events only run when the stack is **empty** and the event loop picks the next task; see [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks).

```js
button.onclick = () => {
  const end = Date.now() + 3000;
  while (Date.now() < end) {} // 3 s on the stack: no clicks, no paint
};
```

## Debugging it

- DevTools **Sources → Call Stack** panel shows the live stack at a breakpoint; the **Scope** panel shows the environments of the selected frame (Local, Closure, Script, Global).
- Visualise contexts step by step: [JavaScript Visualizer (ui.dev)](https://ui.dev/javascript-visualizer).

<Warning>
**Common pitfalls**

- Thinking `setTimeout(fn, 0)` runs "immediately": it runs after the current stack **and** all microtasks are done.
- Confusing the **call stack** (execution contexts, short-lived) with the **heap** (objects, closures' environments, long-lived). See [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection).
- Assuming `this` belongs to the function definition: it is chosen per call, when the context is created.
</Warning>

## Interview questions

<Accordion title="What is an execution context?">
The environment in which JS code is evaluated: a thread of execution plus memory. It holds the variable/lexical environments (declarations, arguments), the reference to the outer environment (scope chain) and the `this` binding. One global context is created per script (one per module), and a new function context per call.
</Accordion>

<Accordion title="What is the call stack and what happens when a function is called?">
A LIFO structure of execution contexts. Calling a function creates a new context (creation phase: declarations bound, `this` decided; then execution phase) and pushes it; `return` pops it and control goes back to the caller's context.
</Accordion>

<Accordion title="Is JavaScript single-threaded? How does async work then?">
The language runs on a single thread with a single call stack. Timers, network and events are handled by the host (Web APIs / libuv); when they finish, their callbacks are queued, and the event loop pushes them onto the stack only when it is empty. Workers give you extra threads, each with its own stack and heap.
</Accordion>

<Accordion title="What causes &quot;Maximum call stack size exceeded&quot;?">
Too many nested frames, almost always unbounded recursion (missing/wrong base case) or accidental mutual recursion (e.g. a setter that sets itself, a `toJSON`/getter calling itself). Fix the base case or convert to iteration.
</Accordion>

<Accordion title="What are the phases of an execution context?">
Creation: environments are created, function declarations are bound to their functions, `var` to `undefined`, `let/const/class` are created uninitialized (TDZ), `this` and the outer reference are set. Execution: code runs top to bottom and assigns values.
</Accordion>

<Accordion title="What's the difference between the stack and the heap?">
The stack holds short-lived call frames and is popped on return. The heap holds objects, functions and any environment that must outlive its call (closures). Heap memory is reclaimed by the garbage collector when unreachable.
</Accordion>

## Related
- [JavaScript Index](/Software%20Engineering/Frontend/01%20JavaScript/JavaScript%20Index)
- [Scope & Lexical Environment](/Software%20Engineering/Frontend/01%20JavaScript/Scope%20%26%20Lexical%20Environment) · [Hoisting, TDZ & var-let-const](/Software%20Engineering/Frontend/01%20JavaScript/Hoisting%2C%20TDZ%20%26%20var-let-const) · [Closures & Memory Model](/Software%20Engineering/Frontend/01%20JavaScript/Closures%20%26%20Memory%20Model) · [this, call, apply & bind](/Software%20Engineering/Frontend/01%20JavaScript/this%2C%20call%2C%20apply%20%26%20bind)
- [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks) · [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection) · [Error Handling](/Software%20Engineering/Frontend/01%20JavaScript/Error%20Handling)
- Course notes: [JavaScript the hard parts v2](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/JavaScript%20the%20hard%20parts%20v2) · [Principles](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/Principles) · [Functions & Callbacks](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/Functions%20%26%20Callbacks) · [Asynchronous JavaScript](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/Asynchronous%20JavaScript) · [JS the hard parts v3](/Software%20Engineering/Frontend/Courses%20%26%20Books/JS%20the%20hard%20parts%20v3)
- [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions)

## Sources
- [JavaScript Visualizer (ui.dev)](https://ui.dev/javascript-visualizer)
- [MDN: Call stack](https://developer.mozilla.org/en-US/docs/Glossary/Call_stack)
- [MDN: JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model)
- [ECMAScript spec: Execution Contexts](https://tc39.es/ecma262/#sec-execution-contexts)
- Frontend Masters, *JavaScript: The Hard Parts v2* (Will Sentance)
