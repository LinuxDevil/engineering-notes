---
tags: [principles, principles/oop]
aliases: [Abstract Class vs Interface, Interface vs Abstract Class, Abstract Class, Interface, Concrete Class]
updated: 2026-09-27
---
# Abstract Class and Interface

> [!tldr] TL;DR
> **Abstract classes are about generalisation; interfaces are about standardisation.** An abstract class is a partial implementation that related subclasses share (state, constructors, protected helpers). An interface is a **contract** any unrelated class can sign. Since Java 8 interfaces can carry `default`/`static` methods, so the real differences today are **state, constructors, access modifiers and single vs multiple inheritance**. Default to interfaces; reach for an abstract class when related types genuinely share state and code.

## The idea

Abstraction means hiding the internal implementation of a feature and showing only the functionality: *what* it does is visible, *how* it does it is hidden (see [[OOP]]).

- An **abstract class** is a class that cannot be instantiated. It hides shared implementation and leaves some methods (`abstract`) for subclasses to fill in.
- An **interface** is a contract: a set of method signatures a class promises to implement. It says nothing about how.

```java
// Interface: a capability many unrelated classes can have
public interface PaymentMethod {
    PaymentResult charge(Money amount);
    default boolean supportsRefunds() { return true; }   // Java 8+ default method
}

// Abstract class: a family of related classes sharing state and a template
public abstract class CardPayment implements PaymentMethod {
    protected final String cardToken;                     // shared state
    protected CardPayment(String cardToken) { this.cardToken = cardToken; }

    @Override
    public final PaymentResult charge(Money amount) {     // Template Method
        validate(amount);
        return doCharge(amount);                          // subclass fills this in
    }
    protected abstract PaymentResult doCharge(Money amount);
    private void validate(Money amount) { if (amount.isNegative()) throw new IllegalArgumentException(); }
}

public final class VisaPayment extends CardPayment {
    public VisaPayment(String token) { super(token); }
    @Override protected PaymentResult doCharge(Money amount) { /* call Visa gateway */ return PaymentResult.ok(); }
}

public final class BankTransfer implements PaymentMethod {   // unrelated class, same contract
    @Override public PaymentResult charge(Money amount) { /* ... */ return PaymentResult.ok(); }
    @Override public boolean supportsRefunds() { return false; }
}
```

The abstract class above is the [[Behavioral Patterns#Template Method|Template Method]] pattern: the skeleton is fixed (`final charge`), one step varies.

## Abstract class vs interface (Java, current rules)

| | Abstract class | Interface |
|---|---|---|
| Keyword | `abstract class`, used with `extends` | `interface`, used with `implements` (interfaces `extends` other interfaces) |
| Methods | Abstract and concrete, any access level | Abstract; `default` and `static` (Java 8+); `private` helpers (Java 9+) |
| State | Any fields: `final` or not, `static` or not | Only `public static final` constants. **No instance state** |
| Constructors | Yes (run by subclasses via `super(...)`) | No |
| Member access | `private`, `protected`, package, `public` | Members are `public` by default (except `private` methods) |
| Inheritance | A class extends **one** class | A class implements **many** interfaces; an interface can extend many |
| Relationship | is-a, within one family | can-do / has-capability, across unrelated types |
| Can implement the other? | An abstract class can implement an interface (and leave methods unimplemented) | An interface cannot extend a class |
| Sealed (Java 17+) | `sealed abstract class ... permits ...` | `sealed interface ... permits ...` |

> [!note] Correction to an old rule of thumb
> "An interface can have only abstract methods" was true before Java 8. Since Java 8 interfaces can have `default` and `static` methods, and since Java 9 `private` methods. What interfaces still cannot have is **instance state** and **constructors**.

## When to use which

**Use an abstract class when:**
- Several closely related classes need to share code (put it once in the base).
- You need non-static, non-final fields so methods can read and modify the object's state.
- You need non-public members (`protected`, `private`) or a constructor that enforces setup.
- You want a template method with a fixed skeleton.

**Use an interface when:**
- You want to specify a behaviour but don't care who implements it or how.
- Unrelated classes should share a capability (`Comparable`, `Closeable`, `Serializable`).
- A class needs several capabilities (multiple inheritance of type).
- You want to depend on an abstraction for testing and swapping (the D in [[SOLID]], see [[Dependency Injection & Inversion of Control]]).

> [!tip] A common combination
> Publish an **interface** as the public type, and optionally ship an **abstract skeletal implementation** (`AbstractList`, `AbstractMap` in the JDK) that implementers may extend for convenience. Callers depend only on the interface.

## Abstract class vs concrete class

| Abstract class | Concrete class |
|---|---|
| Cannot be declared `final` (it would be useless) | Can be declared `final` |
| Cannot be instantiated with `new` | Can be instantiated with `new` |
| Declared with the `abstract` modifier | No `abstract` modifier |
| May or may not contain abstract methods | Cannot contain abstract methods |
| May implement an interface without implementing all its methods | Must implement every interface method |

## Other languages

| Language | Abstract class | Interface-like construct | Notes |
|---|---|---|---|
| **Java** | `abstract class` | `interface` (default/static/private methods) | sealed types since 17 |
| **Kotlin** | `abstract class` | `interface` (can have default bodies and abstract properties, no state) | classes are `final` by default; `sealed interface` |
| **C#** | `abstract class` | `interface` (default interface methods since C# 8) | |
| **TypeScript** | `abstract class` (exists at runtime as a class) | `interface` / `type` (erased at compile time, **structural**) | any object with the right shape satisfies an interface; no `implements` needed |
| **Dart** | `abstract class` | Every class is an implicit interface (`implements`); Dart 3 `interface class`, `base class`, `final class`, `sealed class`, `mixin` | |
| **Python** | `abc.ABC` + `@abstractmethod` | `typing.Protocol` (structural) | |
| **Ruby** | No keyword; raise `NotImplementedError` in the base | Duck typing, modules as mixins | see [[SOLID Principles with Ruby]] |
| **Go** | none | `interface` (implicit, structural) | "accept interfaces, return structs" |

```ts
// TypeScript: interfaces are structural, abstract classes are nominal-ish and exist at runtime
interface Logger { log(msg: string): void; }
const consoleLogger: Logger = { log: m => console.log(m) }; // no class needed

abstract class Job {
  constructor(protected readonly id: string) {}
  run() { console.time(this.id); this.execute(); console.timeEnd(this.id); }
  protected abstract execute(): void;
}
```

## Pitfalls

> [!warning] Common mistakes
> - **Abstract class as a dumping ground** for helpers "every subclass might need". It grows into a god base class that every subclass is coupled to.
> - **Fat interfaces** that force implementers to stub methods (`throw new UnsupportedOperationException()`): split them (Interface Segregation, [[SOLID]]).
> - **Header interfaces**: one interface per class, mirroring every public method, with a single implementation. Adds indirection without value unless you need a seam for testing or plugins.
> - **Adding an abstract method to a published interface** breaks every implementer. Add a `default` method instead, or version the interface.
> - Conflicting `default` methods from two interfaces force the class to override and choose (`A.super.m()`).

## Senior perspective

- Interfaces are **seams**: places where behaviour can be swapped (tests, plugins, adapters). Put them where variation actually exists or where you cross a boundary (I/O, third-party APIs), not everywhere. See [[Coupling, Cohesion & Modularity]].
- **Who owns the interface?** In clean/hexagonal architecture the *consumer* (domain) defines the port, and infrastructure implements it. See [[Software Architecture Styles]].
- For public libraries, interfaces are harder to evolve than abstract classes were pre-Java 8; default methods changed that but can surprise implementers. Design for evolution.
- Consider `sealed` when the set of implementations is closed; you get exhaustive `switch` and a clearer model.

## Interview questions

> [!question]- What is the difference between an abstract class and an interface?
> An abstract class can hold instance state, constructors, and members of any visibility, and a class can extend only one. An interface defines a contract with no instance state; since Java 8 it may have default and static methods (Java 9: private); a class can implement many. Abstract class = shared implementation in a family (is-a). Interface = capability across unrelated types (can-do).

> [!question]- Can an abstract class have a constructor? Why?
> Yes. It can't be instantiated directly, but subclasses call it via `super(...)` to initialise the shared state and enforce invariants.

> [!question]- Can an interface have method implementations?
> Yes, since Java 8 via `default` and `static` methods, and since Java 9 `private` methods. It still can't have instance fields or constructors.

> [!question]- Two interfaces have the same default method. What happens?
> Compile error in the implementing class unless it overrides the method; inside, it can delegate explicitly with `InterfaceA.super.method()`.

> [!question]- When would you choose an abstract class over an interface?
> When related subclasses need shared mutable state, a constructor enforcing setup, protected helpers, or a fixed algorithm skeleton (Template Method). Otherwise, prefer an interface for flexibility and multiple inheritance of type.

## Related
- [[Principles Index]]
- [[OOP]] · [[SOLID]] · [[Behavioral Patterns]] · [[Dependency Injection & Inversion of Control]]
- [[SOLID Principles with Ruby]] · [[TypeScript Essentials]]

## Sources
- [Oracle Java Tutorials: Abstract Methods and Classes](https://docs.oracle.com/javase/tutorial/java/IandI/abstract.html)
- [Oracle Java Tutorials: Default Methods](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html)
- [JEP 409: Sealed Classes](https://openjdk.org/jeps/409)
- Joshua Bloch, *Effective Java*, 3rd ed., Item 20: "Prefer interfaces to abstract classes"
- [Dart: Class modifiers](https://dart.dev/language/class-modifiers)
- [TypeScript Handbook: Classes (abstract)](https://www.typescriptlang.org/docs/handbook/2/classes.html)
