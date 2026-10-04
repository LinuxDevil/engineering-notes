---
title: "this, call, apply & bind"
tags: [frontend, frontend/javascript]
aliases: [this, this keyword, call apply bind, bind, Context Binding, Arrow functions and this]
updated: 2026-09-27
---
# this, call, apply & bind

<Note>
**TL;DR**

The value of `this` is decided **by how a function is called**, not where it's defined (arrow functions are the exception: they take `this` from the surrounding scope). Four call-site rules cover everything: `new` > explicit (`call`/`apply`/`bind`) > implicit (`obj.method()`) > default (`undefined` in strict mode, global object in sloppy mode). `call`, `apply` and `bind` let you set `this` explicitly.
</Note>

## What `this` is

- `this` is part of the [execution context](/Software%20Engineering/Frontend/01%20JavaScript/Execution%20Context%20%26%20Call%20Stack): it refers to the object the currently executing code is "about".
- It's bound **per call**, when the function's execution context is created. The same function can have a different `this` on every call.
- Regular functions have a **dynamic** `this`; arrow functions have a **lexical** `this` (they don't have their own).

## The rules, in order of precedence

{/* diagram source: /diagrams/diagram-231.mmd */}
![diagram](/diagrams/diagram-231.svg)

<Info>
**One function, four `this` values**

```js
'use strict';
function whoAmI() { return this; }

const obj = { name: 'obj', whoAmI };

whoAmI();                   // undefined           (default binding, strict)
obj.whoAmI();               // obj                 (implicit binding)
whoAmI.call({ name: 'x' }); // { name: 'x' }       (explicit binding)
new whoAmI();               // a new whoAmI {}     (new binding)
```
</Info>

### 1. Default binding

A plain call `fn()`: `this` is `undefined` in strict mode (modules and classes are always strict), or `globalThis` (`window`) in sloppy mode.

### 2. Implicit binding and "lost `this`"

`obj.method()` sets `this = obj`. But the binding belongs to the **call**, so pulling the method off the object loses it:

```js
const user = {
  name: 'Ali',
  greet() { return `Hi ${this.name}`; },
};

user.greet();               // 'Hi Ali'
const greet = user.greet;   // just a function reference now
greet();                    // TypeError in strict mode (this is undefined)
setTimeout(user.greet, 0);  // same problem: called as a plain function
button.addEventListener('click', user.greet); // this = button, not user
```

Fixes: `user.greet.bind(user)`, an arrow wrapper `() => user.greet()`, or an arrow class field.

### 3. Explicit binding: `call`, `apply`, `bind`

These methods let us **explicitly set what `this` refers to**.

| Method | Invokes now? | Arguments | Returns |
|---|---|---|---|
| `fn.call(thisArg, a, b)` | Yes | one by one | the function's result |
| `fn.apply(thisArg, [a, b])` | Yes | as an **array** (or array-like) | the function's result |
| `fn.bind(thisArg, a)` | **No** | optional, pre-filled (partial application) | a **new bound function** |

Memory trick: **c**all = **c**ommas, **a**pply = **a**rray, **b**ind = **b**ack later.

<Info>
**The owner's original example**

```js
function setName() {
  this.name = 'Mohammad';
}

const object = {
  name: 'Ali',
  changeName: function () {
    // try each one; `this` here is `object`
    setName.call(this);
    // setName.apply(this);
    // setName.bind(this)();   // bind returns a function, so we must invoke it
  },
};

console.log('Name: ', object.name); // Name:  Ali
object.changeName();
console.log('Name: ', object.name); // Name:  Mohammad
```
Without `call`/`apply`/`bind`, `setName()` would run with the default binding and set `name` on the global object (sloppy) or throw (strict).
</Info>

<Info>
**More `call` / `apply` / `bind`**

```js
function introduce(greeting, punctuation) {
  return `${greeting}, I'm ${this.name}${punctuation}`;
}
const ali = { name: 'Ali' };

introduce.call(ali, 'Hi', '!');          // "Hi, I'm Ali!"
introduce.apply(ali, ['Hello', '.']);    // "Hello, I'm Ali."

const aliSays = introduce.bind(ali, 'Hey'); // this + first arg fixed
aliSays('?');                            // "Hey, I'm Ali?"

// borrowing methods
Array.prototype.slice.call(arguments);   // old way to turn arguments into an array
Math.max.apply(null, [3, 9, 1]);         // 9 (today: Math.max(...arr))
```
</Info>

`bind` details:
- It returns a **new bound function** (a special "bound function" object wrapping the original), not a copy of the original.
- A bound function's `this` is **permanent**: later `call`/`apply`/`bind` can't change it. Only `new` overrides it (then `this` is the new object).
- Binding in render/every call creates a new function each time; in React that breaks memoization, so bind once or use arrow class fields/`useCallback`.

### 4. `new` binding

`new Fn()` creates a new object whose prototype is `Fn.prototype`, calls `Fn` with `this` = that object, and returns it (unless `Fn` returns another object). See [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance).

## Arrow functions vs regular functions

| | Regular function | Arrow function |
|---|---|---|
| `this` | Dynamic: set by the call | **Lexical**: from the enclosing scope, can't be rebound |
| `arguments` | Has its own | None (use `...args`) |
| Can be used with `new` | Yes (if not a method/class shorthand) | No: `TypeError` |
| `prototype` property | Yes | No |
| Good as | object methods, constructors, event handlers needing `this` = element | callbacks inside methods, class-field handlers |

<Info>
**Arrow functions fix "lost this" in callbacks**

```js
class Timer {
  seconds = 0;
  start() {
    setInterval(function () { this.seconds++; }, 1000); // this = undefined/window: bug
    setInterval(() => { this.seconds++; }, 1000);        // this = the Timer: works
  }
}
```
And the reverse mistake:
```js
const counter = {
  count: 0,
  inc: () => this.count++, // arrow as a METHOD: this is the outer (module) this → undefined
};
```
</Info>

## `this` in other places

| Where | `this` is |
|---|---|
| Top level of a classic script | `globalThis` (`window`) |
| Top level of an ES module | `undefined` |
| DOM event handler (regular function) | the element the listener is attached to (`event.currentTarget`) |
| Class method called as `instance.m()` | the instance |
| Class **static** method | the class itself |
| Class field arrow function `handle = () => {}` | the instance, permanently (one function per instance) |
| Inside `setTimeout(function(){})` | `globalThis` in browsers (sloppy or strict), because the timer calls it without a receiver |
| Callbacks of `map`/`forEach` etc. | `undefined` unless you pass the optional `thisArg` |

Use **`globalThis`** when you really need the global object in any environment (browser, worker, Node).

<Info>
**Polyfill `bind` (classic interview question)**

```js
Function.prototype.myBind = function (thisArg, ...preset) {
  const fn = this;
  return function bound(...args) {
    // support `new bound()`: ignore thisArg, like native bind
    if (new.target) return new fn(...preset, ...args);
    return fn.apply(thisArg, [...preset, ...args]);
  };
};
```
</Info>

<Warning>
**Common pitfalls**

- Passing `obj.method` as a callback (event listener, `setTimeout`, `promise.then`) loses `this`.
- Arrow functions as object methods or prototype methods: `this` is not the object.
- `bind` inside a render loop / on every call creates new functions (breaks `React.memo` and removal via `removeEventListener`: you must remove the *same* bound reference).
- Expecting `call`/`apply` to change an arrow function's or a bound function's `this`.
</Warning>

## Interview questions

<Accordion title="How is `this` determined?">
By the call site. Precedence: `new` (new object) > explicit `call`/`apply`/`bind` (given object) > implicit `obj.fn()` (obj) > default (`undefined` in strict, global object in sloppy). Arrow functions ignore all of these and use the enclosing scope's `this`.
</Accordion>

<Accordion title="Difference between `call`, `apply` and `bind`?">
`call` and `apply` invoke immediately with a given `this`; `call` takes arguments individually, `apply` as an array. `bind` doesn't invoke; it returns a new function with `this` (and optionally leading arguments) permanently set.
</Accordion>

<Accordion title="When should you use `bind()`?">
When a function will be called later by someone else (callback, event listener, timer) and it needs a specific `this`; or for partial application. In modern code arrow functions or arrow class fields often replace it.
</Accordion>

<Accordion title="How does `this` behave in arrow functions vs regular functions?">
Arrow functions have no own `this` (or `arguments`); they capture it lexically from where they're defined and it can't be changed by `call`/`apply`/`bind`. Regular functions get `this` from how they're called.
</Accordion>

<Accordion title="What's `this` inside an event listener?">
With a regular function, the element the listener is attached to (`currentTarget`). With an arrow function, whatever `this` was outside.
</Accordion>

<Accordion title="Can you re-bind a bound function?">
No. `bound.bind(other)` or `bound.call(other)` keep the original `this`. Only `new bound()` uses a fresh object instead.
</Accordion>

<Accordion title="Output? `const o = { x: 1, f() { return () => this.x; } }; const g = o.f(); g.call({ x: 2 });`">
`1`. The arrow captured `this` from `f`'s call (`o`), and `call` can't override an arrow's `this`.
</Accordion>

## Related
- [JavaScript Index](/Software%20Engineering/Frontend/01%20JavaScript/JavaScript%20Index)
- [Execution Context & Call Stack](/Software%20Engineering/Frontend/01%20JavaScript/Execution%20Context%20%26%20Call%20Stack) · [Functions - HOF, Callbacks & Currying](/Software%20Engineering/Frontend/01%20JavaScript/Functions%20-%20HOF%2C%20Callbacks%20%26%20Currying) · [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance) · [DOM, Events & Event Delegation](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/DOM%2C%20Events%20%26%20Event%20Delegation)
- Course notes: [JavaScript the hard parts v2](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/JavaScript%20the%20hard%20parts%20v2) · [JS the hard parts v3](/Software%20Engineering/Frontend/Courses%20%26%20Books/JS%20the%20hard%20parts%20v3)
- [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions) · [Output Questions & Tricky Snippets](/Software%20Engineering/Frontend/17%20Interview%20Prep/Output%20Questions%20%26%20Tricky%20Snippets)

## Sources
- [MDN: this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)
- [MDN: Function.prototype.call](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call) · [apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply) · [bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)
- [MDN: Arrow function expressions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)
- [MDN: globalThis](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/globalThis)
