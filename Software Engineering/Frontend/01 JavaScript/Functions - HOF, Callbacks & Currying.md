---
title: "Functions - HOF, Callbacks & Currying"
tags: [frontend, frontend/javascript]
aliases: [Functions, Higher Order Functions, HOF, Callbacks, Currying, First-class functions, Anonymous functions, Pure functions, Function composition]
updated: 2026-09-27
---
# Functions - HOF, Callbacks & Currying

<Note>
**TL;DR**

In JavaScript, functions are **first-class**: they are objects, so you can store them in variables, pass them as arguments (**callbacks**) and return them from other functions. A function that takes or returns a function is a **higher-order function (HOF)**. HOFs let you write generic code and pass in the part that varies, which keeps code DRY. **Currying** and **partial application** are HOF techniques built on [closures](/Software%20Engineering/Frontend/01%20JavaScript/Closures%20%26%20Memory%20Model).
</Note>

## Ways to define a function

| Form | Syntax | Hoisted with body? | Own `this`/`arguments`? | Notes |
|---|---|---|---|---|
| Declaration | `function add(a, b) {}` | Yes | Yes | Best for top-level named helpers |
| Expression | `const add = function (a, b) {}` | No | Yes | Can be named: `function inner() {}` for recursion and stack traces |
| Arrow | `const add = (a, b) => a + b` | No | No (lexical) | Concise; ideal for callbacks; can't be `new`-ed |
| Method shorthand | `{ add(a, b) {} }` | n/a | Yes | Can use `super`; not constructible |
| IIFE | `(function () { ... })()` / `(() => {})()` | n/a | | Runs once, private scope |
| Generator / async | `function* g() {}`, `async function f() {}` | as declarations | Yes | See [Iterators, Generators & Iterator Helpers](/Software%20Engineering/Frontend/01%20JavaScript/Iterators%2C%20Generators%20%26%20Iterator%20Helpers), [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation) |

**Anonymous functions** are functions with no name, usually intended to be used once, which is why you see them mostly as callbacks: `arr.map(function (x) { return x * 2; })`, `button.onclick = () => {}`. Tip: give non-trivial callbacks a name (or assign them to a `const`) to get readable stack traces and to be able to `removeEventListener` them.

### Parameters

```js
function greet(name = 'guest', ...rest) {   // default + rest parameters
  console.log(name, rest, arguments.length); // `arguments` only in non-arrow functions
}
greet();                  // guest [] 0
greet('Ali', 1, 2);       // Ali [1, 2] 3
greet.length;             // 0 (params before the first default/rest)
```

- Primitives are passed **by value**; objects are passed **by sharing** (a copy of the reference). Reassigning a parameter doesn't affect the caller, mutating an object parameter does. See [Types, Coercion & Equality](/Software%20Engineering/Frontend/01%20JavaScript/Types%2C%20Coercion%20%26%20Equality).

## Functions are first-class objects

![Screenshot_2023-08-27_at_11.04.24_PM](/Attachments/Screenshot_2023-08-27_at_11.04.24_PM.png)

- **First class** means a function can do everything an object can: be assigned to a variable, stored in an array/object, passed as an argument, returned from a function, and have properties.
- Behind the scenes, functions **are** objects (callable objects with an internal `[[Call]]` method): `typeof fn === 'function'`, `fn instanceof Object === true`, `fn.name`, `fn.length`, and you can add properties to them.

That's how we can pass a function as a parameter: it's just a value.

## Why HOFs? Generalise the part that changes

When we write functions we should think about **reusability** and not break the **DRY** principle.

### Bad: one function per operation

![Screenshot_2023-08-26_at_7.10.39_PM](/Attachments/Screenshot_2023-08-26_at_7.10.39_PM.png)

```js
function copyArrayAndMultiplyBy2(array) {
  const output = [];
  for (let i = 0; i < array.length; i++) output.push(array[i] * 2);
  return output;
}
const myArray = [1, 2, 3];
const result = copyArrayAndMultiplyBy2(myArray);
// copyArrayAndDivideBy2, copyArrayAndAdd3 ... would repeat everything except one line
```

```text
------ Global Memory ------
copyArrayAndMultiplyBy2: <function>
myArray: [1, 2, 3]
result: <uninitialized>
    result = copyArrayAndMultiplyBy2([1, 2, 3])  → new execution context
    ------------ Local Memory ------------ | ---- Call Stack ----
    array: [1, 2, 3]                       | global, copyArrayAndMultiplyBy2
    output: []
    1 * 2 pushed → output: [2]
    2 * 2 pushed → output: [2, 4]
    3 * 2 pushed → output: [2, 4, 6]
    look up output and return it           | global
------ End of execution context ------
result: [2, 4, 6]
```

### Good: pass the varying logic in as a function

Leave part of the functionality **undetermined** and pass it in as a parameter:

![Screenshot_2023-08-27_at_10.52.01_PM](/Attachments/Screenshot_2023-08-27_at_10.52.01_PM.png)

```js
function copyArrayAndManipulate(array, instructions) {
  const output = [];
  for (let i = 0; i < array.length; i++) output.push(instructions(array[i]));
  return output;
}
function multiplyBy2(input) { return input * 2; }

const result = copyArrayAndManipulate([1, 2, 3], multiplyBy2); // [2, 4, 6]
```

```text
------ Global Memory ------
copyArrayAndManipulate: <function>
multiplyBy2: <function>
result: <uninitialized>
    result = copyArrayAndManipulate([1, 2, 3], multiplyBy2) → new execution context
    ------------ Local Memory ------------ | ---- Call Stack ----
    array: [1, 2, 3]                       | global, copyArrayAndManipulate
    instructions: <function multiplyBy2>
    output: []
    call instructions(1) → new execution context
        Local Memory: input: 1 → return 2  | global, copyArrayAndManipulate, multiplyBy2
    push 2 → output: [2]                   | global, copyArrayAndManipulate
    call instructions(2) → input: 2 → return 4
    push 4 → output: [2, 4]
    call instructions(3) → input: 3 → return 6
    push 6 → output: [2, 4, 6]
    look up output and return it           | global
------ End of execution context ------
result: [2, 4, 6]
```

- `copyArrayAndManipulate` is the **higher-order function** (it takes a function).
- `multiplyBy2` is the **callback** (the function passed in to be called back later).
- This is exactly what `Array.prototype.map` is. **HOFs and callbacks keep our code DRY.**

## Higher-order functions

A **higher-order function** is any function that **takes a function as an argument, returns a function, or both**.

| Takes a function | Returns a function | Both |
|---|---|---|
| `map`, `filter`, `reduce`, `forEach`, `sort`, `addEventListener`, `setTimeout`, `Promise.then` | `bind`, `makeAdder`, currying helpers | `debounce`, `throttle`, `memoize`, `once`, React HOCs, Redux middleware |

```js
const numbers = [1, 2, 3, 4];
const double = numbers.map((x) => x * 2);         // [2, 4, 6, 8]
const evens  = numbers.filter((x) => x % 2 === 0); // [2, 4]
const sum    = numbers.reduce((acc, x) => acc + x, 0); // 10
```

<Info>
**`map` vs `forEach`**

```js
const out = [1, 2, 3].map(x => x * 2);      // returns a NEW array [2, 4, 6]
const r = [1, 2, 3].forEach(x => x * 2);    // returns undefined; only for side effects
```
Use `map` to transform, `forEach` for side effects. Neither can be stopped early (use `for...of` with `break`, or `some`/`find`). Neither waits for `async` callbacks.
</Info>

<Info>
**`debounce`: a HOF that takes and returns a function**

```js
function debounce(fn, delay = 300) {
  let timer;                               // closure state
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}
input.addEventListener('input', debounce(e => search(e.target.value), 250));
```
Used in [Case Study - Autocomplete](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Autocomplete). `throttle` is the sibling: run at most once per interval.
</Info>

## Callbacks

A **callback** is a function passed to another function to be called later: synchronously (`map`, `sort`) or asynchronously (`setTimeout`, event listeners, Node-style `fs.readFile(path, (err, data) => {})`).

**Callback hell** (the "pyramid of doom") is nested async callbacks that are hard to read, handle errors in, and reason about:

```js
getUser(id, (err, user) => {
  if (err) return handle(err);
  getOrders(user, (err, orders) => {
    if (err) return handle(err);
    getDetails(orders[0], (err, details) => {
      /* ...and deeper */
    });
  });
});
```

Fixes: named functions, then [Promises and async/await](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation):

```js
const user = await getUser(id);
const orders = await getOrders(user);
const details = await getDetails(orders[0]);
```

Another callback problem is **inversion of control**: you trust the library to call your callback exactly once, with the right arguments. Promises fix that too.

## Pure functions

A **pure function**:
1. always returns the **same output for the same input** (no dependence on outside mutable state, time, randomness), and
2. has **no side effects** (doesn't mutate its arguments or globals, no I/O, no DOM).

```js
const add = (a, b) => a + b;                 // pure
let total = 0;
const addToTotal = (x) => (total += x);      // impure: mutates outer state
const addItem = (cart, item) => [...cart, item];          // pure (new array)
const addItemBad = (cart, item) => { cart.push(item); return cart; }; // impure (mutates)
```

Pure functions are easy to test, memoize and reason about; React components and reducers must be pure ([State, Props & Data Flow](/Software%20Engineering/Frontend/05%20React/State%2C%20Props%20%26%20Data%20Flow), [Redux & Redux Toolkit](/Software%20Engineering/Frontend/06%20State%20Management/Redux%20%26%20Redux%20Toolkit)).

## Currying

**Currying** transforms a function that takes several arguments, `f(a, b, c)`, into a **sequence of functions that each take one argument**, `f(a)(b)(c)`. Each call returns a new function waiting for the next argument, and closures remember the earlier ones.

```js
function multiply(a) {
  return function (b) {
    return function (c) {
      return a * b * c;
    };
  };
}

const double = multiply(2);    // a = 2
const twoDouble = double(5);   // b = 5
twoDouble(10);                 // c = 10 → 100

// arrow version
const multiplyArrow = a => b => c => a * b * c;
multiplyArrow(2)(5)(10);       // 100
```

<Info>
**A generic `curry` helper (interview favourite)**

```js
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn.apply(this, args);
    return (...next) => curried.apply(this, [...args, ...next]);
  };
}

const sum3 = (a, b, c) => a + b + c;
const cSum = curry(sum3);
cSum(1)(2)(3);   // 6
cSum(1, 2)(3);   // 6
cSum(1)(2, 3);   // 6
```
Relies on `fn.length`, so it doesn't work with default/rest parameters.
</Info>

<Info>
**Infinite currying: `sum(1)(2)(3)()`**

```js
const sum = a => b => (b === undefined ? a : sum(a + b));
sum(1)(2)(3)(); // 6
```
</Info>

### Partial application vs currying

- **Partial application**: fix *some* arguments now, get a function for the rest: `const log = logger.bind(null, 'INFO')`, or `const addTax = rate => price => price * (1 + rate)`.
- **Currying**: always one argument at a time until all are supplied. A curried function makes partial application trivial.

Real uses: configuring once and reusing (`const fetchJson = withBase('/api')`), event handlers with parameters (`onClick={select(id)}`), and point-free pipelines.

## Composition

Compose small pure functions into bigger ones.

```js
const compose = (...fns) => x => fns.reduceRight((acc, f) => f(acc), x); // right → left
const pipe    = (...fns) => x => fns.reduce((acc, f) => f(acc), x);      // left → right

const slugify = pipe(
  s => s.trim(),
  s => s.toLowerCase(),
  s => s.replace(/\s+/g, '-'),
);
slugify('  Hello World '); // 'hello-world'
```

{/* diagram source: /diagrams/diagram-214.mmd */}
![diagram](/diagrams/diagram-214.svg)

Other functional-programming ideas from the JS Concepts checklist ([Frontend Roadmap & Checklist](/Software%20Engineering/Frontend/Frontend%20Roadmap%20%26%20Checklist)): first-class functions, pure functions, currying, composing, recursion, referential transparency (an expression can be replaced by its value). Functors/monads: `Array` (`map`) and `Promise` (`then`) behave much like them; that's enough for FE interviews.

<Warning>
**Common pitfalls**

- `['1', '2', '3'].map(parseInt)` → `[1, NaN, NaN]`: `map` passes `(value, index, array)` and `parseInt` takes `(string, radix)`. Use `.map(Number)` or `s => parseInt(s, 10)`.
- `forEach` with `async` callbacks doesn't wait; use `for...of` + `await` or `Promise.all(arr.map(...))`.
- Passing a method as a callback loses `this` ([this, call, apply & bind](/Software%20Engineering/Frontend/01%20JavaScript/this%2C%20call%2C%20apply%20%26%20bind)).
- Creating new inline callbacks every render in React defeats `memo` (the React Compiler or `useCallback` helps; see [React Performance](/Software%20Engineering/Frontend/05%20React/React%20Performance)).
</Warning>

## Interview questions

<Accordion title="What is a higher-order function?">
A function that takes a function as an argument and/or returns a function. Examples: `map`, `filter`, `reduce`, `addEventListener`, `setTimeout`, `bind`, `debounce`, React HOCs.
</Accordion>

<Accordion title="What does &quot;functions are first-class citizens&quot; mean?">
Functions are values (objects): they can be assigned, stored, passed as arguments, returned, and given properties. That's what makes callbacks, HOFs and closures possible.
</Accordion>

<Accordion title="What is currying? How is it different from partial application?">
Currying turns `f(a, b, c)` into `f(a)(b)(c)`, one argument per call. Partial application fixes some arguments and returns a function for the rest (any number at once). Both use closures.
</Accordion>

<Accordion title="What is a pure function and why does it matter?">
Same input → same output, no side effects. They're predictable, testable, cacheable (memoization) and safe for React rendering and reducers.
</Accordion>

<Accordion title="What is callback hell and how do you avoid it?">
Deeply nested async callbacks with repeated error handling. Avoid by naming/splitting functions, and by using Promises and async/await (which also fix inversion of control).
</Accordion>

<Accordion title="Arrow vs regular functions?">
Arrows have lexical `this`, no `arguments`, no `prototype`, can't be used with `new`, and are always anonymous expressions (name inferred from the variable). Regular functions have dynamic `this` and their own `arguments`.
</Accordion>

<Accordion title="`map` vs `forEach`?">
`map` returns a new array of transformed values (use for transformations, chainable). `forEach` returns `undefined` and is for side effects. Neither can `break`, neither awaits async callbacks.
</Accordion>

<Accordion title="Implement `curry(fn)` / `compose` / `debounce`.">
See the examples above: `curry` collects args until `args.length >= fn.length`; `compose` is `reduceRight` over functions; `debounce` clears and resets a timer held in a closure.
</Accordion>

## Related
- [JavaScript Index](/Software%20Engineering/Frontend/01%20JavaScript/JavaScript%20Index)
- [Closures & Memory Model](/Software%20Engineering/Frontend/01%20JavaScript/Closures%20%26%20Memory%20Model) · [this, call, apply & bind](/Software%20Engineering/Frontend/01%20JavaScript/this%2C%20call%2C%20apply%20%26%20bind) · [Execution Context & Call Stack](/Software%20Engineering/Frontend/01%20JavaScript/Execution%20Context%20%26%20Call%20Stack) · [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation) · [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability)
- Patterns: [Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns) (HOCs, render props) · [Frontend Design Patterns](/Software%20Engineering/Frontend/09%20Architecture/Frontend%20Design%20Patterns) · [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns)
- Course notes: [Functions & Callbacks](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/Functions%20%26%20Callbacks) · [JavaScript the hard parts v2](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/JavaScript%20the%20hard%20parts%20v2) · [JS the hard parts v3](/Software%20Engineering/Frontend/Courses%20%26%20Books/JS%20the%20hard%20parts%20v3)
- [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions)

## Sources
- [MDN: Functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions)
- [MDN: First-class function](https://developer.mozilla.org/en-US/docs/Glossary/First-class_Function)
- [MDN: Callback function](https://developer.mozilla.org/en-US/docs/Glossary/Callback_function)
- [MDN: Array.prototype.map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map)
- [javascript.info: Currying](https://javascript.info/currying-partials)
- Frontend Masters, *JavaScript: The Hard Parts v2* (Will Sentance)
