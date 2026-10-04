---
title: "Creational Patterns"
tags: [principles, principles/patterns]
aliases: [Creational Design Patterns, Singleton, Factory Method, Abstract Factory, Builder Pattern, Prototype Pattern, Factory Pattern]
updated: 2026-09-27
---
# Creational Patterns

<Note>
**TL;DR**

Creational patterns control **how objects get created** so that client code doesn't hard-wire concrete classes or deal with complex construction. The five GoF patterns: **Singleton** (one instance), **Factory Method** (defer the choice of class), **Abstract Factory** (families of related objects), **Builder** (step-by-step construction) and **Prototype** (clone an existing object). In modern code, **dependency injection** and plain factory functions replace most of them; Builder remains very common.
</Note>

Overview and the other groups: [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns) · [Structural Patterns](/Software%20Engineering/Principles/Structural%20Patterns) · [Behavioral Patterns](/Software%20Engineering/Principles/Behavioral%20Patterns).

## Why creation needs patterns

`new ConcreteClass(...)` inside business logic couples it to one implementation, one set of constructor arguments and one lifecycle. Creational patterns move that decision somewhere else: a factory, a builder, or the composition root (see [Dependency Injection & Inversion of Control](/Software%20Engineering/Principles/Dependency%20Injection%20%26%20Inversion%20of%20Control)).

## Singleton

**Intent:** ensure a class has exactly one instance and provide a global access point.

```java
// Java: the simplest thread-safe singleton is an enum (Effective Java, Item 3)
public enum AppConfig {
    INSTANCE;
    private final Properties props = load();
    public String get(String key) { return props.getProperty(key); }
    private static Properties load() { /* read file */ return new Properties(); }
}
AppConfig.INSTANCE.get("db.url");

// Lazy holder idiom: lazy and thread-safe without synchronisation
public final class Registry {
    private Registry() {}
    private static class Holder { static final Registry INSTANCE = new Registry(); }
    public static Registry getInstance() { return Holder.INSTANCE; }
}
```

```ts
// TypeScript/JS: an ES module is evaluated once, so a module-level instance is a singleton
export const logger = createLogger({ level: "info" });
```

<Warning>
**Singleton is the most criticised GoF pattern**

It's **global mutable state** with a nice name: hidden dependencies (you can't see from a constructor that a class uses it), hard to replace in tests, and a concurrency hazard. Prefer: create one instance at startup and **inject** it (a "singleton scope" in Spring/NestJS/Angular DI). Keep true singletons for stateless or read-only things.
</Warning>

## Factory Method

**Intent:** define an interface for creating an object but let subclasses (or a function/parameter) decide which class to instantiate.

```ts
interface Notifier { send(to: string, msg: string): Promise<void>; }
class EmailNotifier implements Notifier { async send(to: string, msg: string) { /* SMTP */ } }
class SmsNotifier   implements Notifier { async send(to: string, msg: string) { /* Twilio */ } }
class PushNotifier  implements Notifier { async send(to: string, msg: string) { /* FCM */ } }

// "Simple factory" / factory function: the most common modern form
type Channel = "email" | "sms" | "push";
export function createNotifier(channel: Channel): Notifier {
  switch (channel) {
    case "email": return new EmailNotifier();
    case "sms":   return new SmsNotifier();
    case "push":  return new PushNotifier();
  }
}
```

```java
// Classic GoF Factory Method: creation is a hook overridden by subclasses
abstract class Dialog {
    void render() { Button ok = createButton(); ok.onClick(this::close); ok.draw(); }
    protected abstract Button createButton();            // the factory method
    void close() {}
}
class WindowsDialog extends Dialog { protected Button createButton() { return new WindowsButton(); } }
class WebDialog     extends Dialog { protected Button createButton() { return new HtmlButton(); } }
```

Also common: **static factory methods** with meaningful names instead of constructors: `List.of(...)`, `Optional.empty()`, `Duration.ofSeconds(5)`, `Money.euros(10)`. They can cache instances, return subtypes and validate before creating.

## Abstract Factory

**Intent:** provide an interface for creating **families** of related objects without specifying concrete classes, guaranteeing the products are compatible.

```ts
interface Button { render(): string; }
interface Checkbox { render(): string; }
interface UiFactory { button(label: string): Button; checkbox(label: string): Checkbox; }

const lightUi: UiFactory = {
  button: l => ({ render: () => `<button class="light">${l}</button>` }),
  checkbox: l => ({ render: () => `<input type="checkbox" class="light"> ${l}` }),
};
const darkUi: UiFactory = {
  button: l => ({ render: () => `<button class="dark">${l}</button>` }),
  checkbox: l => ({ render: () => `<input type="checkbox" class="dark"> ${l}` }),
};

function settingsPage(ui: UiFactory) {          // never mixes light buttons with dark checkboxes
  return [ui.checkbox("Notifications").render(), ui.button("Save").render()].join("\n");
}
settingsPage(prefersDark ? darkUi : lightUi);
```

Real examples: JDBC (`Connection` creates matching `Statement`s), cloud SDK clients per provider, cross-platform UI toolkits. Cost: adding a new *product type* means changing every factory.

## Builder

**Intent:** separate the construction of a complex object from its representation, building it step by step. It makes construction gradual and readable and avoids "telescoping constructors" (`new Pizza(true, false, true, null, 12)`).

```java
public final class HttpRequestSpec {
    private final String url; private final String method;
    private final Map<String, String> headers; private final Duration timeout;

    private HttpRequestSpec(Builder b) {
        this.url = Objects.requireNonNull(b.url, "url");
        this.method = b.method; this.headers = Map.copyOf(b.headers); this.timeout = b.timeout;
    }
    public static Builder builder(String url) { return new Builder(url); }

    public static final class Builder {
        private final String url; private String method = "GET";
        private final Map<String, String> headers = new HashMap<>();
        private Duration timeout = Duration.ofSeconds(10);
        private Builder(String url) { this.url = url; }
        public Builder method(String m) { this.method = m; return this; }
        public Builder header(String k, String v) { headers.put(k, v); return this; }
        public Builder timeout(Duration t) { this.timeout = t; return this; }
        public HttpRequestSpec build() { return new HttpRequestSpec(this); }   // validate here
    }
}

var req = HttpRequestSpec.builder("https://api.example.com/orders")
        .method("POST").header("Authorization", "Bearer …").timeout(Duration.ofSeconds(3))
        .build();
```

The JDK's own `java.net.http.HttpRequest.newBuilder()`, `StringBuilder`, Lombok's `@Builder` and Kotlin's named/default arguments (which make most builders unnecessary) are everyday examples.

```ts
// TypeScript: an options object with defaults often replaces a builder
function createServer({ port = 8080, tls = false, cors = [] as string[] } = {}) { /* ... */ }
createServer({ port: 3000, tls: true });
```

**Test Data Builder** is a big win in tests: `anOrder().withItems(3).paid().build()` keeps tests focused on what matters. See [Test-Driven Development (TDD)](/Software%20Engineering/Principles/Test-Driven%20Development%20(TDD)).

## Prototype

**Intent:** create new objects by **copying an existing instance** (the prototype) instead of instantiating from a class. Useful when construction is expensive or when you want "the same as this, but...".

```ts
const baseInvoice = { currency: "EUR", taxRate: 0.21, lines: [] as Line[] };

const inv1 = structuredClone(baseInvoice);        // deep copy
const inv2 = { ...baseInvoice, currency: "USD" }; // shallow copy with an override

// JavaScript objects literally inherit from prototypes
const animal = { speak() { return `${this.name} makes a sound`; } };
const dog = Object.create(animal, { name: { value: "Rex" } });
```

```kotlin
data class Config(val host: String, val port: Int, val tls: Boolean)
val prod = Config("api.example.com", 443, tls = true)
val staging = prod.copy(host = "staging.example.com")      // prototype via copy
```

Pitfall: **shallow vs deep copy**. Java's `Cloneable`/`clone()` is widely considered broken (Effective Java, Item 13); prefer copy constructors or static copy factories.

## Comparison

| Pattern | Use when | Modern alternative | Main risk |
|---|---|---|---|
| Singleton | One shared, stateless resource | DI container singleton scope, module instance | Hidden global state |
| Factory Method | Caller shouldn't know which concrete class | Factory function, DI | Over-abstraction for one type |
| Abstract Factory | Families of objects must match | Config-driven DI modules | Painful to add product types |
| Builder | Many optional params, validation, immutability | Named/default args, options objects, records + `with` methods | Boilerplate |
| Prototype | Copying is cheaper or clearer than building | Spread, `copy`, `structuredClone` | Shallow-copy bugs |

## Senior perspective

- The **composition root** (main, DI module, framework config) is the right place for most creation decisions; domain code receives ready objects.
- Builders are great at API boundaries and for immutable objects; inside a codebase, named arguments or records often suffice.
- Static factories beat constructors for readability (`Temperature.celsius(20)`) and can return cached or subtype instances.
- In code review, `new` of an infrastructure class (HTTP client, repository, clock) inside business logic is a testability smell. Inject a `Clock`, don't call `Instant.now()` directly.

## Interview questions

<Accordion title="How do you implement a thread-safe singleton in Java?">
An `enum` with one constant (serialization- and reflection-safe), or the lazy holder idiom (static nested class holds the instance; JVM class init is thread-safe). Double-checked locking also works with a `volatile` field but is more error-prone. Often better: let the DI container manage a single instance.
</Accordion>

<Accordion title="Why is Singleton considered an anti-pattern by many?">
It introduces global state and hidden dependencies, makes testing hard (can't substitute easily, state leaks between tests), and couples code to a concrete class. Dependency injection with a singleton scope gives one instance without these problems.
</Accordion>

<Accordion title="Factory Method vs Abstract Factory?">
Factory Method creates one product and lets subclasses or parameters choose the concrete class. Abstract Factory is an object with several factory methods that create a family of related products that must be used together.
</Accordion>

<Accordion title="When would you use a Builder?">
For objects with many optional parameters, when construction requires validation across fields, or to build immutable objects readably. Also for test data. In Kotlin/Python/TS, named/default parameters often make it unnecessary.
</Accordion>

<Accordion title="What's the difference between a shallow and a deep copy in Prototype?">
A shallow copy duplicates the top-level object but shares nested references; changing a nested object affects both copies. A deep copy duplicates the whole graph (`structuredClone` in JS; copy constructors in Java).
</Accordion>

## Related
- [Principles Index](/Software%20Engineering/Principles/Principles%20Index)
- [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns) · [Structural Patterns](/Software%20Engineering/Principles/Structural%20Patterns) · [Behavioral Patterns](/Software%20Engineering/Principles/Behavioral%20Patterns)
- [Dependency Injection & Inversion of Control](/Software%20Engineering/Principles/Dependency%20Injection%20%26%20Inversion%20of%20Control) · [Immutability Across Languages](/Software%20Engineering/Principles/Immutability%20Across%20Languages) · [OOP](/Software%20Engineering/Principles/OOP)
- Frontend: [Prototypes, Classes & Inheritance](/Software%20Engineering/Frontend/01%20JavaScript/Prototypes%2C%20Classes%20%26%20Inheritance)

## Sources
- Gamma et al., *Design Patterns* (1994), chapter 3 "Creational Patterns"
- [Refactoring.Guru: Creational patterns](https://refactoring.guru/design-patterns/creational-patterns)
- Joshua Bloch, *Effective Java*, 3rd ed.: Item 1 (static factories), Item 2 (builders), Item 3 (enum singleton), Item 13 (clone)
- [Java SE: java.net.http.HttpRequest.Builder](https://docs.oracle.com/en/java/javase/21/docs/api/java.net.http/java/net/http/HttpRequest.Builder.html)
- [MDN: structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone)
- Nat Pryce, [Test Data Builders](http://www.natpryce.com/articles/000714.html)
