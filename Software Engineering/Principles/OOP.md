---
tags: [principles, principles/oop]
aliases: [Object-Oriented Programming, Object Oriented Programming, OOP Pillars, Four Pillars of OOP, Polymorphism, Encapsulation, Inheritance]
updated: 2026-09-27
---
# OOP

> [!tldr] TL;DR
> **Object-oriented programming** organises code around **objects**: data plus the behaviour that guards it. The four classic pillars are **encapsulation, abstraction, inheritance and polymorphism**. Senior practice is more selective than textbook OOP: encapsulate invariants, program to interfaces, prefer **composition over inheritance**, and use plain immutable data (records, sealed types, pattern matching) where objects add no value.

![[Untitled 16.png]]

## Basic terminology

| Term | Meaning | Example |
|---|---|---|
| **Class** | A template or blueprint describing what state an object holds and what it can do | `class BankAccount { ... }` |
| **Object** | A working entity created from a class, with its own state | `new BankAccount("Ali")` |
| **Instance** | Practically the same as object; "instance of X" stresses which class it came from | `acc instanceof BankAccount` |
| **Method** | A behaviour of a class: what an object can do | `acc.deposit(100)` |
| **Field / attribute** | A piece of state stored in the object | `private long balanceCents` |
| **Constructor** | Code that runs when an object is created and must leave it in a valid state | `BankAccount(String owner)` |
| **Interface** | A contract: a set of operations without (necessarily) an implementation | `interface Payable` |

## The four pillars

### 1. Encapsulation

Encapsulation wraps the data (fields) and the code that acts on it (methods) into one unit and **hides the data** so it can only change through that unit's methods. This is also called **data hiding**. The real goal is not "add getters and setters"; it is to **protect invariants** (rules that must always be true).

```java
// ❌ Anemic: anyone can put the object in an invalid state
public class BankAccount {
    public long balanceCents;
}
account.balanceCents = -500; // nothing stops this

// ✅ Encapsulated: the only way to change state keeps the invariant (balance >= 0)
public class BankAccount {
    private long balanceCents;

    public void withdraw(long amountCents) {
        if (amountCents <= 0) throw new IllegalArgumentException("amount must be positive");
        if (amountCents > balanceCents) throw new InsufficientFundsException();
        balanceCents -= amountCents;
    }
    public long balanceCents() { return balanceCents; }
}
```

> [!tip] "Tell, don't ask"
> Instead of asking an object for its data and deciding outside (`if (acc.getBalance() > x) acc.setBalance(...)`), tell the object what to do (`acc.withdraw(x)`). The decision lives next to the data. Getter/setter pairs for every field are encapsulation in name only.

### 2. Abstraction

> "You abstract only when you discover the abstraction."

Abstraction hides **how** something works and exposes only **what** it does. The user of `List` knows `add` and `get`; whether it is an `ArrayList` or a `LinkedList` is hidden.

There are two kinds:

1. **Control abstraction**: languages let you express operations at a high level instead of moving bits around. In a Pascal-like language, `a := (1 + 2) * 5` hides loads, adds, multiplies and stores.
2. **Data abstraction**: defining the *behaviour* of a data structure separately from its representation. `Collection` / `List` are the abstraction; `ArrayList` and `LinkedList` are different concrete representations.

**Encapsulation vs abstraction**

- Encapsulation is about **packaging**: *how* data is accessed (methods) and *what* is visible (access modifiers).
- Abstraction is about **what the class does for you** at a conceptual level, ignoring details that do not matter for your purpose.
- Abstraction is a form of generalisation: code that applies to more than one specific situation. It includes encapsulation, information hiding and generalisation. Inheritance is one way to express it.
- Encapsulation hides unnecessary data inside a capsule; abstraction shows only the essential features. **Encapsulation enables abstraction.**

See [[Abstract Class and Interface]] for the language tools used to express abstraction, and [[Coupling, Cohesion & Modularity]] for "deep modules" (a lot of functionality behind a small interface).

### 3. Inheritance

Inheritance is the process where one class acquires the fields and methods of another (`class Car extends Vehicle`). It models an **is-a** relationship and enables code reuse and subtype polymorphism.

**Types of inheritance**

1. **Single**: `B extends A`
2. **Multiple**: `C extends A, B` (C++, Python). Java, C#, TypeScript and Kotlin allow multiple inheritance of **interfaces** only, which avoids the "diamond problem" for state.
3. **Multi-level**: `C extends B extends A`
4. **Hierarchical**: `B extends A` and `C extends A`
5. **Hybrid**: a combination of the above

![[Untitled 17.png]]

> [!warning] Inheritance is the strongest form of coupling
> A subclass depends on the **implementation** of its parent, not just its interface (the *fragile base class* problem). A change to a protected method or to the order of internal calls can break every subclass. Guidelines:
> - Use inheritance only for true is-a relationships that satisfy the [[SOLID|Liskov Substitution Principle]].
> - Keep hierarchies shallow (1–2 levels).
> - Mark classes `final` (Java/Kotlin default) unless designed for extension. Joshua Bloch: "design and document for inheritance or else prohibit it".
> - Prefer **composition** ("has-a") for reuse. See [[Coupling, Cohesion & Modularity#Composition over inheritance]].

### 4. Polymorphism

Polymorphism (Greek: "many forms") means one name or interface can stand for different behaviours depending on context. It lets you write code against an abstraction and plug in new behaviour without changing the caller (the basis of the [[SOLID|Open-Closed Principle]]).

| Kind | When resolved | Mechanism | Example |
|---|---|---|---|
| **Static (compile-time, early binding)** | Compiler picks the method | **Method overloading**, generics (parametric polymorphism) | `print(int)` vs `print(String)` |
| **Dynamic (runtime, late binding)** | Runtime picks based on the object's actual type (virtual dispatch) | **Method overriding** | `shape.area()` calls `Circle.area()` |
| **Ad-hoc / structural** | Depends on language | Duck typing (Ruby, Python), structural types (TypeScript, Go interfaces) | anything with a `quack()` method |

#### Method overriding (runtime polymorphism)

A subclass re-implements a method it inherited because the parent's version does not fit. Same signature, different behaviour. The method called is decided **at runtime** from the object's real type.

```java
abstract class Shape { abstract double area(); }
class Circle extends Shape {
    private final double r;
    Circle(double r) { this.r = r; }
    @Override double area() { return Math.PI * r * r; }
}
class Square extends Shape {
    private final double side;
    Square(double side) { this.side = side; }
    @Override double area() { return side * side; }
}

List<Shape> shapes = List.of(new Circle(1), new Square(2));
double total = shapes.stream().mapToDouble(Shape::area).sum(); // dispatches per object
```

#### Method overloading (compile-time polymorphism)

Several methods in the same class share a name but take **different parameter lists** (number or types). The return type may differ, but a different return type **alone** is not enough to overload.

```java
class Printer {
    void print(int n)      { System.out.println("int " + n); }
    void print(String s)   { System.out.println("str " + s); }
    void print(int a, int b) { System.out.println(a + b); }
}
```

#### Overriding vs overloading

| | Overriding | Overloading |
|---|---|---|
| Polymorphism | Runtime | Compile time |
| Where | Between superclass and subclass | Within the same class (or inherited) |
| Signature | Same name **and** parameters | Same name, **different** parameters |
| Resolved | At runtime, by the object's type | At compile time, by the argument types |
| When it breaks | Bugs show up at runtime (harder) | Usually a compile error (easy to fix) |

> [!tip] Use `@Override` (Java) / `override` (Kotlin, C#, TypeScript 4.3+ with `noImplicitOverride`)
> The compiler then errors if you think you are overriding but the signature does not match, a classic silent bug.

## Beyond the pillars: modern OOP in practice

### Composition over inheritance

```ts
// ❌ Inheritance explosion: LoggingCachingRepository extends CachingRepository extends Repository ...
// ✅ Compose behaviours around an interface (Decorator)
interface UserRepo { find(id: string): Promise<User | null>; }

class SqlUserRepo implements UserRepo { /* ... */ }
class CachedUserRepo implements UserRepo {
  constructor(private inner: UserRepo, private cache: Map<string, User>) {}
  async find(id: string) {
    const hit = this.cache.get(id);
    if (hit) return hit;
    const user = await this.inner.find(id);
    if (user) this.cache.set(id, user);
    return user;
  }
}
const repo: UserRepo = new CachedUserRepo(new SqlUserRepo(), new Map());
```

See [[Structural Patterns]] (Decorator, Adapter) and [[Dependency Injection & Inversion of Control]].

### Objects vs data (data-oriented programming)

Not everything needs to be an object with behaviour. DTOs, API payloads, events and configuration are **just data**. Brian Goetz (Java language architect) describes **data-oriented programming**: model data as immutable, transparent carriers, model alternatives with **sealed** hierarchies, and put behaviour in functions that **pattern match** on the data.

```java
// Java 21+: sealed interface + records + exhaustive pattern matching
sealed interface Shape permits Circle, Rect {}
record Circle(double r) implements Shape {}
record Rect(double w, double h) implements Shape {}

static double area(Shape s) {
    return switch (s) {                       // compiler checks exhaustiveness
        case Circle(double r)          -> Math.PI * r * r;       // record pattern
        case Rect(double w, double h)  -> w * h;
    };                                        // no default needed: sealed
}
```

| Classic OOP (virtual methods) | Data-oriented (sealed + pattern matching) |
|---|---|
| Easy to add a **new type** (new subclass) | Easy to add a **new operation** (new function) |
| Adding an operation touches every class | Adding a type makes the compiler flag every `switch` that must handle it |
| Behaviour lives with data | Behaviour lives in functions over data |
| Good for open, pluggable hierarchies | Good for closed domains: ASTs, events, results, payment states |

This trade-off is the **expression problem**. Senior engineers mix both: objects to guard invariants and hide I/O, records and sealed types for domain values and messages. The same idea exists in Kotlin (`sealed` + `data class` + `when`), TypeScript (discriminated unions + `switch` on a `kind` field), Dart 3 (`sealed class` + records + patterns), Rust (`enum` + `match`) and C# (records + switch expressions).

> [!info]+ 🆕 2025–2026
> - **Java 25 LTS (GA 2025-09-16)** finalised **JEP 513 Flexible Constructor Bodies**: statements (validation, computing arguments) may now appear **before** `super(...)`/`this(...)`, so constructors can reject bad input before the parent runs. It also finalised **JEP 512 Compact Source Files and Instance Main Methods** (a class-less `void main()` for small programs) and **JEP 511 Module Import Declarations**.
> - Primitive types in patterns (`case int i`) is still in preview: third preview in Java 25 (JEP 507), fourth in **Java 26** (GA 2026-03-17, JEP 530) and fifth in **Java 27** (GA 2026-09-15, JEP 532).
> - Records (Java 16), sealed classes (Java 17), pattern matching for `switch` and record patterns (Java 21) are all final, so data-oriented style is idiomatic in any LTS from 21 onward.
> - TypeScript 5.8 (2025) added `--erasableSyntaxOnly`, which rejects `enum`, `namespace` and constructor **parameter properties** so code can run with Node's type stripping. Class-heavy TS written with `constructor(private x: X)` needs explicit fields under that flag. See [[TypeScript 5.8 to 7.0]].

## Pitfalls

> [!warning] Common OOP mistakes
> - **God objects**: one `Manager`/`Service` class that knows everything. Split by responsibility ([[SOLID]]).
> - **Anemic domain model**: classes with only getters/setters and all logic in "service" classes. Fine for CRUD, painful for rich rules. See [[Domain-Driven Design (DDD)]].
> - **Deep inheritance trees** built for reuse, not substitutability.
> - **Getter/setter for everything**, which leaks internals and breaks encapsulation.
> - **Shared mutable state** between objects, which becomes a concurrency bug later. See [[Immutability Across Languages]].
> - **Overusing patterns**: an interface with one implementation "just in case" (see YAGNI in [[DRY, KISS & YAGNI]]).

## Senior perspective

- Ask **"what invariant does this class protect?"** If the answer is none, it may be a record/struct plus functions.
- Keep constructors honest: an object should be valid right after construction (use factories or builders for complex setup; see [[Creational Patterns]]).
- Object identity vs value equality: entities have identity (a `User` with ID 42), value objects are equal by value (`Money(10, "EUR")`). Get `equals`/`hashCode` right, or use records/data classes.
- Inheritance across module or service boundaries is a versioning trap: prefer interfaces and composition at public API boundaries.
- In reviews look for: public mutable fields, `instanceof` chains (missing polymorphism or a missing sealed type), `protected` state, subclasses that throw `UnsupportedOperationException` (an LSP violation).

## Interview questions

> [!question]- What are the four pillars of OOP?
> **Encapsulation** (bundle data with the methods that guard it and hide the data), **abstraction** (expose what, hide how), **inheritance** (a class acquires another's members; is-a), **polymorphism** (one interface, many implementations, resolved at compile time via overloading or at runtime via overriding).

> [!question]- What is the difference between encapsulation and abstraction?
> Encapsulation is a mechanism: hiding state behind methods and access modifiers so invariants can't be broken. Abstraction is a design idea: presenting only the essential behaviour of something and hiding details. Encapsulation is one of the tools that makes abstraction possible.

> [!question]- Overloading vs overriding?
> Overloading: same method name, different parameter lists, in the same class, resolved at compile time. Overriding: a subclass redefines a method with the same signature, resolved at runtime by the object's actual type. Overriding enables runtime polymorphism.

> [!question]- Why "composition over inheritance"?
> Inheritance couples the subclass to the parent's implementation (fragile base class), fixes the relationship at compile time and forces single-hierarchy thinking. Composition depends only on an interface, can be changed at runtime, and combines behaviours freely (Decorator, Strategy). Use inheritance for genuine, substitutable is-a relationships.

> [!question]- Why doesn't Java support multiple inheritance of classes?
> To avoid the diamond problem with state and implementation: if B and C both extend A and override a method, which version does D inherit? Java allows multiple inheritance of interfaces; since Java 8 default methods can conflict, and the class must then override and pick explicitly (`B.super.m()`).

> [!question]- When would you use a sealed interface with records instead of classic polymorphism?
> When the set of variants is closed and known (payment states, AST nodes, API results) and you expect to add operations more often than variants. The compiler checks `switch` exhaustiveness, so adding a variant surfaces every place that must handle it. Use classic virtual methods when third parties need to add new implementations.

> [!question]- What is an anemic domain model and is it always bad?
> Objects that hold data with no behaviour, while logic lives in service classes. Fowler calls it an anti-pattern for complex domains because invariants are scattered. For simple CRUD or data pipelines it is fine and often simpler.

## Related
- [[Principles Index]]
- [[Abstract Class and Interface]] · [[SOLID]] · [[Design Patterns]] · [[Coupling, Cohesion & Modularity]]
- [[Functional Programming Principles]] · [[Immutability Across Languages]] · [[Domain-Driven Design (DDD)]]
- JavaScript specifics: [[Prototypes, Classes & Inheritance]]

## Sources
- [Oracle Java Tutorials: Object-Oriented Programming Concepts](https://docs.oracle.com/javase/tutorial/java/concepts/)
- Joshua Bloch, *Effective Java*, 3rd ed. (Addison-Wesley, 2018): Items 18–19 (composition over inheritance; design for inheritance or prohibit it)
- Gamma, Helm, Johnson, Vlissides, *Design Patterns* (1994): "favor object composition over class inheritance"
- [Brian Goetz: Data-Oriented Programming in Java (InfoQ, 2022)](https://www.infoq.com/articles/data-oriented-programming-java/)
- [Martin Fowler: Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html)
- [OpenJDK: JDK 25](https://openjdk.org/projects/jdk/25/) · [JEP 513 Flexible Constructor Bodies](https://openjdk.org/jeps/513) · [JDK 26](https://openjdk.org/projects/jdk/26/) · [JDK 27](https://openjdk.org/projects/jdk/27/)
- [JEP 441: Pattern Matching for switch](https://openjdk.org/jeps/441) · [JEP 440: Record Patterns](https://openjdk.org/jeps/440) · [JEP 409: Sealed Classes](https://openjdk.org/jeps/409)
- [TypeScript 5.8 release notes: --erasableSyntaxOnly](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html)
