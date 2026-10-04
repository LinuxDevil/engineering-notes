---
title: "Reactive Spring - Ch 2 Java Refresher"
tags: [backend, backend/java, book]
aliases: [Java Refresher, Lambdas, Functional Interface, Method References, Abstract class vs interface Java]
updated: 2026-09-27
---
# Reactive Spring - Ch 2 Java Refresher

<Note>
**TL;DR**

The book's quick Java refresher: **type inference** with `var`, **lambdas** (to methods what anonymous classes are to classes), **functional interfaces** (exactly one abstract method, the target type of a lambda), and **method references** (`obj::method`). Plus the classic comparison of **abstract classes vs interfaces**. These features are everywhere in Reactor and Spring code.
</Note>

*Part of [Reactive Spring Book](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20Book) (book notes). Renamed from "Chapter 2".*

## Type inference

You can use `var` so that local variable types are inferred (Java 10+):

```java
Customer customer = new Customer();
// can now be
var customer = new Customer();
```

`var` is only for local variables with an initializer; the type is still static (fixed at compile time).

## Lambdas

A **lambda** is to a method definition what an anonymous class is to a class definition. You can create your own **functional interface**:

```java
@FunctionalInterface
interface MyHandler {
    String handle(int one, Date two);
}

// This can be expressed in Java as follows
MyHandler handler = (one, two) -> "Hello";

void process(MyHandler m) { ... }
process((one, two) -> "Hello");
```

## Method references

If you want to cut out the intermediate lambda, you can use a delegate (method reference):

```java
class Delegate {
    String pleaseHandle(int one, Date two) {
        return one + " : " + two;
    }
}

// This can be expressed as follows:
Delegate delegate = new Delegate();
process(delegate::pleaseHandle);
```

| Kind | Syntax | Equivalent lambda |
|---|---|---|
| Static method | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| Bound instance method | `delegate::pleaseHandle` | `(a, b) -> delegate.pleaseHandle(a, b)` |
| Unbound instance method | `String::toUpperCase` | `s -> s.toUpperCase()` |
| Constructor | `ArrayList::new` | `() -> new ArrayList<>()` |

## Abstract class, interface, functional interface, method reference

1. **Abstract class**: a class that can't be instantiated. It can contain abstract methods (no body) and concrete methods, plus state (fields) and constructors. Its purpose is to be extended by subclasses, providing a common template.
2. **Interface**: a contract of behaviours a class promises to implement. Originally only abstract methods; since Java 8 it can contain `default` and `static` methods (and `private` methods since Java 9). A class can implement any number of interfaces. Interfaces aren't part of the class hierarchy, so any class from any inheritance tree can implement them. They can hold constants but not instance state.
3. **Functional interface**: an interface with **exactly one** abstract method (it may still have default/static methods). It's the type of a lambda or method reference. Examples: `Runnable` (`run()`), `Function<T,R>`, `Supplier<T>`.
4. **Method reference**: a reference to a method without executing it (Java 8+), a more concise lambda. Instead of `x -> System.out.println(x)` write `System.out::println`. Can reference static methods, instance methods and constructors.

|   | Abstract class | Interface | Functional interface | Method reference |
| --- | --- | --- | --- | --- |
| Definition | A class that can't be instantiated and is meant to be extended | A contract of behaviours a class promises to implement | An interface with exactly one abstract method | A reference to a method without executing it |
| Keyword | `abstract` | `interface` | `interface` (+ optional `@FunctionalInterface`) | `::` operator |
| Example | `public abstract class Animal { abstract void makeSound(); }` | `public interface Flyable { void fly(); }` | `@FunctionalInterface public interface Runnable { void run(); }` | `System.out::println` |
| Methods with implementation | Yes | `default`/`static` since Java 8 | `default`/`static` since Java 8 | n/a |
| Abstract methods | Any number | Any number | Exactly one | n/a |
| Instance state / constructors | Yes | No (constants only) | No | n/a |
| Can be instantiated | No | No | No (but a lambda creates an instance) | No (it *produces* an instance of a functional interface) |
| Extended / implemented | Extended (single inheritance) | Implemented (many) | Implemented, usually via lambdas | n/a |
| Purpose | Share code and state among related classes | Define a contract across unrelated classes | Target type for lambdas/method references | Concise lambdas |

See also [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface) and [Java Language Essentials](/Software%20Engineering/Backend/Java/Java%20Language%20Essentials).

<Info>
**🆕 Since the book**

Java 16–25 added records, sealed interfaces, pattern matching (21) and `void main()` compact source files (25). Sealed interfaces + records are now the idiomatic way to model closed sets of types. See [Modern Java 17 to 25](/Software%20Engineering/Backend/Java/Modern%20Java%2017%20to%2025).
</Info>

## Interview questions

<Accordion title="What is a functional interface?">
An interface with exactly one abstract method, so a lambda or method reference can implement it. `@FunctionalInterface` makes the compiler enforce that.
</Accordion>

<Accordion title="When would you choose an abstract class over an interface?">
When related classes share state or implementation and you need constructors or protected members. Use interfaces for capabilities/contracts that unrelated classes can implement, and because a class can implement many interfaces but extend only one class.
</Accordion>

<Accordion title="What are the four kinds of method references?">
Static (`Integer::parseInt`), bound instance (`obj::method`), unbound instance (`String::length`), and constructor (`ArrayList::new`).
</Accordion>

## Related
- [Reactive Spring Book](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20Book) · [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index)
- [Java Language Essentials](/Software%20Engineering/Backend/Java/Java%20Language%20Essentials) · [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface) · [Reactive Spring - Ch 3 Spring Fundamentals](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%203%20Spring%20Fundamentals)

## Sources
- Josh Long, *Reactive Spring* (2020), chapter 2
- [dev.java: Lambda expressions](https://dev.java/learn/lambdas/)
- [JEP 286: Local-Variable Type Inference](https://openjdk.org/jeps/286)
