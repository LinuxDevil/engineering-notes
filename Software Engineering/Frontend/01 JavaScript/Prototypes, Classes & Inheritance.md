---
title: "Prototypes, Classes & Inheritance"
tags: [frontend, frontend/javascript]
aliases: [Prototypes, Prototype chain, Prototypal inheritance, Classes, Inheritance, OOP in JavaScript, private fields]
updated: 2026-09-27
---
# Prototypes, Classes & Inheritance

<Note>
**TL;DR**

Every JS object has a hidden link, `[[Prototype]]`, to another object. When a property isn't found on the object, the engine follows that link, and the next, up the **prototype chain** until it reaches `null`. That's JavaScript's inheritance: objects delegate to other objects. `class` is (mostly) nicer syntax over constructor functions and prototypes, plus real extras: `#private` fields, `static` blocks, and ergonomic `extends`/`super`.
</Note>

## The prototype chain

- `[[Prototype]]` is an internal slot. Read it with `Object.getPrototypeOf(obj)` (the old `obj.__proto__` accessor still works but is legacy).
- **`Fn.prototype`** is a normal property that exists on **functions**. It is the object that becomes the `[[Prototype]]` of instances created with `new Fn()`. Don't confuse the two.
- Property **reads** walk the chain; property **writes** create/update an own property on the object itself (unless a setter is found on the chain).

```js
const animal = { eats: true, walk() { return `${this.name} walks`; } };
const rabbit = Object.create(animal);   // rabbit.[[Prototype]] = animal
rabbit.name = 'Bugs';

rabbit.eats;                            // true  (found on animal)
rabbit.walk();                          // 'Bugs walks' (this = rabbit, the receiver)
Object.hasOwn(rabbit, 'eats');          // false
'eats' in rabbit;                       // true  (in checks the chain)
Object.getPrototypeOf(animal) === Object.prototype; // true
Object.getPrototypeOf(Object.prototype);            // null (end of chain)
```

{/* diagram source: /diagrams/diagram-227.mmd */}
![diagram](/diagrams/diagram-227.svg)

Built-ins work the same way: `[1, 2].map` is found on `Array.prototype`; `[].__proto__.__proto__ === Object.prototype`; functions delegate to `Function.prototype` (that's where `call`/`apply`/`bind` live).

## Constructor functions and `new`

Before `class`, "classes" were functions called with `new`:

```js
function User(name) {
  this.name = name;                           // own property per instance
}
User.prototype.greet = function () {          // shared by all instances (one copy)
  return `Hi, I'm ${this.name}`;
};

const u = new User('Ali');
u.greet();                                    // "Hi, I'm Ali"
Object.getPrototypeOf(u) === User.prototype;  // true
u.constructor === User;                       // true (via User.prototype.constructor)
```

What `new User('Ali')` does:

{/* diagram source: /diagrams/diagram-228.mmd */}
![diagram](/diagrams/diagram-228.svg)

<Info>
**Implement `new` yourself (interview)**

```js
function myNew(Ctor, ...args) {
  const obj = Object.create(Ctor.prototype);
  const result = Ctor.apply(obj, args);
  return result !== null && (typeof result === 'object' || typeof result === 'function') ? result : obj;
}
```
</Info>

Methods go on the **prototype** (shared, one function in memory) rather than inside the constructor (a new function per instance).

## Classes

```js
class User {
  static count = 0;                 // static (class-level) field
  role = 'member';                  // public instance field
  #password;                        // private instance field

  constructor(name, password) {
    this.name = name;
    this.#password = password;
    User.count++;
  }

  greet() { return `Hi, I'm ${this.name}`; }   // goes on User.prototype
  get initials() { return this.name[0]; }       // accessor on the prototype
  checkPassword(p) { return this.#password === p; }
  static create(name) { return new User(name, 'temp'); } // on User itself
}

typeof User;                        // 'function': a class is a special function
Object.getPrototypeOf(new User('a', 'b')) === User.prototype; // true
```

What `class` adds over constructor functions:
- Class bodies are always **strict mode**.
- Calling a class without `new` throws `TypeError`.
- Class declarations are hoisted but in the **TDZ** until evaluated ([Hoisting, TDZ & var-let-const](/Software%20Engineering/Frontend/01%20JavaScript/Hoisting%2C%20TDZ%20%26%20var-let-const)).
- Methods are **non-enumerable** and not constructible.
- Real **private** members, **static blocks**, and correct built-in subclassing (`class MyErr extends Error`, `extends Array`).

### Inheritance: `extends` and `super`

```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);                    // must call super() before using `this`
    this.breed = breed;
  }
  speak() { return `${super.speak()} (woof)`; } // override + call parent
}

const d = new Dog('Rex', 'lab');
d.speak();                          // 'Rex makes a sound (woof)'
d instanceof Dog && d instanceof Animal; // true
```

`extends` sets up **two** chains:

{/* diagram source: /diagrams/diagram-229.mmd */}
![diagram](/diagrams/diagram-229.svg)

The second chain is why **static** methods are inherited too (`Dog.create` finds `Animal.create`).

### Class fields (ES2022)

- **Public fields** `x = 1;` are created on **each instance**, right after `super()` returns (or at the start of the constructor in a base class). They are *defined* (`Object.defineProperty` semantics), not assigned.
- **Arrow function fields** bind `this` permanently, handy for callbacks, but create one function **per instance** and aren't on the prototype (can't be overridden via `super`).

```js
class Button {
  label = 'Click';
  handleClick = () => console.log(this.label); // this always the instance
}
const b = new Button();
document.body.addEventListener('click', b.handleClick); // works without bind
```

### Private members: `#`

```js
class Account {
  #balance = 0;                     // private field
  static #instances = 0;            // private static field

  #log(msg) { console.log(msg); }   // private method
  get #isEmpty() { return this.#balance === 0; } // private accessor

  deposit(amount) {
    this.#balance += amount;
    this.#log(`+${amount}`);
  }
  get balance() { return this.#balance; }
}

const acc = new Account();
acc.deposit(50);
acc.balance;   // 50
acc.#balance;  // SyntaxError: private field must be declared in an enclosing class
```

- **Hard privacy**: enforced by the language, invisible to `Object.keys`, `JSON.stringify`, `Reflect.ownKeys` and Proxies. Unlike TypeScript's `private`, which is compile-time only ([TypeScript Essentials](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%20Essentials)).
- `#names` are **lexically scoped to the class body**: any code inside the class can access `#x` on *any* instance of that class (e.g. in `equals(other) { return this.#id === other.#id; }`).
- Accessing `#x` on an object that doesn't have it throws `TypeError`.

### Private brand checks with `in` (ES2022)

```js
class Point {
  #x = 0;
  static isPoint(obj) {
    return #x in obj;               // true only for objects created by Point's constructor
  }
}
Point.isPoint(new Point());         // true
Point.isPoint({ x: 0 });            // false, and no try/catch needed
```

More reliable than `instanceof`, which can be fooled by `Object.setPrototypeOf` or `Symbol.hasInstance` and fails across realms (iframes).

### Static blocks (ES2022)

Run once when the class is evaluated; they can use private names and do multi-statement static initialisation.

```js
class Config {
  static settings;
  static #secret;
  static {
    const raw = { theme: 'dark', token: 'abc' };
    Config.settings = { theme: raw.theme };
    Config.#secret = raw.token;
  }
}
```

### Getters, setters and computed members

```js
class Temperature {
  #c = 0;
  get fahrenheit() { return this.#c * 9 / 5 + 32; }
  set fahrenheit(f) { this.#c = (f - 32) * 5 / 9; }
  [Symbol.iterator]() { /* computed method name */ }
}
```

## Checking and manipulating prototypes

| Task | Use |
|---|---|
| Get the prototype | `Object.getPrototypeOf(obj)` |
| Create with a given prototype | `Object.create(proto, descriptors?)`, `Object.create(null)` for a dictionary with no inherited keys |
| Own property? | `Object.hasOwn(obj, key)` (ES2022; safer than `obj.hasOwnProperty`) |
| Is `obj` built by `C`? | `obj instanceof C` (checks `C.prototype` on the chain), or `#x in obj` for brand checks |
| Is `proto` in the chain? | `proto.isPrototypeOf(obj)` |
| Change the prototype later | `Object.setPrototypeOf`: **avoid**; it de-optimises the object in engines |

**Prototype pollution**: if user-controlled keys like `__proto__` or `constructor.prototype` reach a naive deep-merge, attackers can add properties to `Object.prototype` for every object in the app. Guard merges, use `Object.create(null)` or `Map` for dictionaries. See [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).

## OOP in JS: the four pillars

| Pillar | In JavaScript |
|---|---|
| **Encapsulation** | `#private` fields, closures, modules |
| **Abstraction** | Expose a small public API (methods), hide the rest |
| **Inheritance** | Prototype chain / `extends` |
| **Polymorphism** | Method overriding; duck typing (anything with the right methods works, e.g. anything iterable) |

JavaScript has no built-in `interface` or `abstract class` (TypeScript adds them at the type level). A runtime abstract guard:

```js
class Shape {
  constructor() { if (new.target === Shape) throw new TypeError('abstract'); }
  area() { throw new Error('not implemented'); }
}
```

### Composition over inheritance

Deep class hierarchies are fragile ("gorilla holding the banana and the whole jungle"). Prefer composing behaviour from small functions/objects, and in UI code, component composition and hooks over class inheritance ([Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns)).

```js
const canFly = (o) => ({ ...o, fly: () => `${o.name} flies` });
const canSwim = (o) => ({ ...o, swim: () => `${o.name} swims` });
const duck = canSwim(canFly({ name: 'Duck' }));
```

See also [SOLID](/Software%20Engineering/Principles/SOLID) and [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns).

<Warning>
**Common pitfalls**

- Using `this` before `super()` in a subclass constructor → `ReferenceError`.
- Passing a class method as a callback loses `this` ([this, call, apply & bind](/Software%20Engineering/Frontend/01%20JavaScript/this%2C%20call%2C%20apply%20%26%20bind)).
- Arrow functions as prototype methods: impossible (they'd be fields), and arrow fields can't be overridden via `super.method()`.
- Putting arrays/objects on the **prototype** (`User.prototype.tags = []`) shares one array across all instances.
- `for...in` iterates inherited enumerable properties too; use `Object.keys`/`Object.entries`.
- Decorators (`@logged`) are a TC39 proposal, not standard JS yet; TypeScript 5.0+ implements the standard version separately from its older `experimentalDecorators` (used by Angular).
</Warning>

## Interview questions

<Accordion title="What is prototypal inheritance?">
Objects have a `[[Prototype]]` link to another object. Missing properties are looked up along this chain until `null`. So objects inherit by delegating to other objects, not by copying from classes.
</Accordion>

<Accordion title="Difference between `__proto__` and `prototype`?">
`obj.__proto__` (legacy accessor for `[[Prototype]]`) is the object `obj` delegates to. `Fn.prototype` is a property of functions/classes: the object that will become `[[Prototype]]` of instances created with `new Fn()`. So `new Fn().__proto__ === Fn.prototype`.
</Accordion>

<Accordion title="What does `new` do?">
Creates an object with `[[Prototype]]` = `Ctor.prototype`, calls the constructor with `this` bound to it, and returns it unless the constructor returns another object.
</Accordion>

<Accordion title="Are classes just syntactic sugar?">
Mostly (they compile down to constructor functions + prototypes), but not only: they're always strict, must be called with `new`, are in TDZ before declaration, have non-enumerable methods, and support `#private` members, static blocks and proper built-in subclassing, which can't be replicated exactly with functions.
</Accordion>

<Accordion title="How do you make a property truly private?">
`#private` fields/methods (language-enforced), or closures (a factory function's local variables). TypeScript `private` and `_underscore` conventions are not runtime-private.
</Accordion>

<Accordion title="What is `#x in obj`?">
An ergonomic brand check (ES2022): true if `obj` has the private field `#x` of this class, without throwing. More robust than `instanceof`.
</Accordion>

<Accordion title="How does `instanceof` work? When does it fail?">
It checks whether `C.prototype` appears in `obj`'s prototype chain. It fails across realms (an array from an iframe isn't `instanceof Array`, use `Array.isArray`), and can be fooled by changing prototypes or `Symbol.hasInstance`.
</Accordion>

<Accordion title="Composition vs inheritance?">
Inheritance is an "is-a" relationship and couples child to parent; changes ripple down. Composition builds objects from smaller behaviours ("has-a"), which is more flexible. React and modern FE favour composition.
</Accordion>

## Related
- [JavaScript Index](/Software%20Engineering/Frontend/01%20JavaScript/JavaScript%20Index)
- [this, call, apply & bind](/Software%20Engineering/Frontend/01%20JavaScript/this%2C%20call%2C%20apply%20%26%20bind) · [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability) · [Types, Coercion & Equality](/Software%20Engineering/Frontend/01%20JavaScript/Types%2C%20Coercion%20%26%20Equality) · [Error Handling](/Software%20Engineering/Frontend/01%20JavaScript/Error%20Handling) (custom error classes) · [Modern JavaScript ES2022-ES2027](/Software%20Engineering/Frontend/01%20JavaScript/Modern%20JavaScript%20ES2022-ES2027)
- [TypeScript Essentials](/Software%20Engineering/Frontend/02%20TypeScript/TypeScript%20Essentials) · [Web Components & Shadow DOM](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Components%20%26%20Shadow%20DOM) (custom elements are classes) · [Component Patterns](/Software%20Engineering/Frontend/05%20React/Component%20Patterns)
- [SOLID](/Software%20Engineering/Principles/SOLID) · [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns) · [Frontend Design Patterns](/Software%20Engineering/Frontend/09%20Architecture/Frontend%20Design%20Patterns)
- Course notes: [JS the hard parts v3](/Software%20Engineering/Frontend/Courses%20%26%20Books/JS%20the%20hard%20parts%20v3) (OOP section) · [JavaScript the hard parts v2](/Software%20Engineering/Frontend/Courses%20%26%20Books/JavaScript%20the%20Hard%20Parts%20v2/JavaScript%20the%20hard%20parts%20v2)
- [JavaScript Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/JavaScript%20Interview%20Questions)

## Sources
- [MDN: Inheritance and the prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)
- [MDN: Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)
- [MDN: Private properties](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_properties)
- [MDN: Static initialization blocks](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Static_initialization_blocks)
- [MDN: new operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new)
- [MDN: Object.hasOwn](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwn)
