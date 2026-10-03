---
tags: [principles, principles/patterns]
aliases: [Structural Design Patterns, Adapter Pattern, Decorator Pattern, Facade, Proxy Pattern, Composite Pattern, Bridge Pattern, Flyweight]
updated: 2026-09-27
---
# Structural Patterns

> [!tldr] TL;DR
> Structural patterns describe **how objects and classes are composed** into larger structures while keeping them flexible. The seven GoF patterns: **Adapter** (make incompatible interfaces fit), **Decorator** (add behaviour by wrapping), **Proxy** (control access through a stand-in), **Facade** (simple front for a complex subsystem), **Composite** (treat trees and leaves alike), **Bridge** (split abstraction from implementation) and **Flyweight** (share state to save memory). Adapter, Decorator, Proxy and Facade look alike (a wrapper) but differ in **intent**.

Overview and the other groups: [[Design Patterns]] · [[Creational Patterns]] · [[Behavioral Patterns]].

## Adapter

**Intent:** convert the interface of a class into another interface clients expect. Also known as **Wrapper**. Lets you keep your own domain interface and plug third-party code behind it (an *anti-corruption* layer in [[Domain-Driven Design (DDD)|DDD]] terms, a *driven adapter* in hexagonal architecture).

```ts
// Our port (what the domain wants)
interface PaymentGateway {
  charge(amountCents: number, currency: string, customerId: string): Promise<{ id: string }>;
}

// Third-party SDK with a different shape (imagine it's from npm)
declare class AcmePaySdk {
  createPayment(req: { value: string; ccy: string; payer: string }): Promise<{ paymentRef: string }>;
}

// Adapter: translates our interface into theirs
class AcmePayAdapter implements PaymentGateway {
  constructor(private sdk: AcmePaySdk) {}
  async charge(amountCents: number, currency: string, customerId: string) {
    const res = await this.sdk.createPayment({
      value: (amountCents / 100).toFixed(2), ccy: currency, payer: customerId,
    });
    return { id: res.paymentRef };
  }
}
```

Switching provider now means writing another adapter; the domain doesn't change. Java examples: `Arrays.asList` (array → List), `InputStreamReader` (byte stream → character reader).

## Decorator

**Intent:** attach additional responsibilities to an object **dynamically** by wrapping it in an object with the **same interface**. A flexible alternative to subclassing for extending behaviour, and decorators stack.

```java
public interface UserRepository { Optional<User> findById(String id); }

public final class JdbcUserRepository implements UserRepository {
    public Optional<User> findById(String id) { /* SELECT ... */ return Optional.empty(); }
}

public final class CachingUserRepository implements UserRepository {
    private final UserRepository inner; private final Map<String, User> cache = new ConcurrentHashMap<>();
    public CachingUserRepository(UserRepository inner) { this.inner = inner; }
    public Optional<User> findById(String id) {
        User hit = cache.get(id);
        if (hit != null) return Optional.of(hit);
        Optional<User> u = inner.findById(id);
        u.ifPresent(x -> cache.put(id, x));
        return u;
    }
}

public final class TimedUserRepository implements UserRepository {
    private final UserRepository inner; private final MeterRegistry metrics;
    public TimedUserRepository(UserRepository inner, MeterRegistry m) { this.inner = inner; this.metrics = m; }
    public Optional<User> findById(String id) {
        return metrics.timer("users.find").record(() -> inner.findById(id));
    }
}

UserRepository repo = new TimedUserRepository(new CachingUserRepository(new JdbcUserRepository(ds)), metrics);
```

Java I/O is the textbook case: `new BufferedReader(new InputStreamReader(new FileInputStream(f)))`.

```ts
// Functional decorator: a higher-order function with the same signature
const withRetry = <A extends unknown[], R>(fn: (...a: A) => Promise<R>, times = 3) =>
  async (...args: A): Promise<R> => {
    for (let i = 0; ; i++) {
      try { return await fn(...args); }
      catch (e) { if (i >= times - 1) throw e; }
    }
  };
const fetchUserWithRetry = withRetry(fetchUser);
```

> [!note] Decorator pattern vs `@decorator` syntax
> TypeScript/JS decorators (TC39 Stage 3, supported since TypeScript 5.0), Python decorators and Java annotations are **language features** for annotating or wrapping declarations. They are often used to implement the Decorator pattern (e.g. a `@memoize` method decorator), but they are not the same thing.

## Proxy

**Intent:** provide a surrogate that **controls access** to another object. Same interface as the real subject, like Decorator, but the purpose is access control rather than adding features.

Kinds: **virtual proxy** (lazy loading: Hibernate lazy associations), **protection proxy** (authorisation), **remote proxy** (gRPC/RMI stubs make a network call look local), **caching proxy**, **smart reference** (reference counting, logging).

```ts
// JS built-in Proxy: intercept property access (used by Vue 3 reactivity, MobX, Immer)
const target = { balance: 100 };
const guarded = new Proxy(target, {
  set(obj, prop, value) {
    if (prop === "balance" && value < 0) throw new RangeError("balance can't be negative");
    return Reflect.set(obj, prop, value);
  },
});
guarded.balance = 50;   // ok
// guarded.balance = -1; // RangeError
```

Spring AOP creates proxies around beans for `@Transactional`, `@Cacheable` and `@Async`. **Gotcha:** a method calling another annotated method on `this` bypasses the proxy, so the annotation has no effect.

## Facade

**Intent:** provide one simple, unified interface to a complex subsystem. Callers get the 90% use case without learning every class.

```ts
class CheckoutFacade {
  constructor(
    private inventory: InventoryService, private pricing: PricingService,
    private payments: PaymentGateway, private shipping: ShippingService,
    private email: EmailService,
  ) {}

  async placeOrder(cart: Cart, customer: Customer) {
    await this.inventory.reserve(cart.items);
    const total = this.pricing.total(cart, customer);
    const charge = await this.payments.charge(total.cents, total.currency, customer.id);
    const shipment = await this.shipping.schedule(cart, customer.address);
    await this.email.sendConfirmation(customer, { charge, shipment });
    return { chargeId: charge.id, trackingNo: shipment.trackingNo };
  }
}
```

An application service / use case in layered architecture is often a facade. An API gateway or a BFF is a facade at system level. Risk: the facade becomes a god object; keep it thin and delegate.

## Composite

**Intent:** compose objects into **tree structures** and let clients treat individual objects (leaves) and compositions (branches) uniformly.

```ts
interface FsNode { name: string; size(): number; }

class FileNode implements FsNode {
  constructor(public name: string, private bytes: number) {}
  size() { return this.bytes; }
}
class Folder implements FsNode {
  private children: FsNode[] = [];
  constructor(public name: string) {}
  add(n: FsNode) { this.children.push(n); return this; }
  size() { return this.children.reduce((s, c) => s + c.size(), 0); }   // recursion hides the tree
}

const root = new Folder("root")
  .add(new FileNode("a.txt", 120))
  .add(new Folder("img").add(new FileNode("logo.png", 2048)));
root.size(); // 2168
```

Everywhere in UI: the DOM, React/Angular component trees, scene graphs, menus, organisation charts, expression trees.

## Bridge

**Intent:** decouple an **abstraction** from its **implementation** so the two can vary independently. Avoids a class explosion when there are two dimensions of variation (N shapes × M renderers would need N×M subclasses).

```java
interface Renderer { void circle(double x, double y, double r); }            // implementation side
class SvgRenderer    implements Renderer { public void circle(double x, double y, double r) { /* <circle> */ } }
class CanvasRenderer implements Renderer { public void circle(double x, double y, double r) { /* ctx.arc */ } }

abstract class Shape {                                                         // abstraction side
    protected final Renderer renderer;
    protected Shape(Renderer r) { this.renderer = r; }
    abstract void draw();
}
class Circle extends Shape {
    private final double x, y, r;
    Circle(Renderer rr, double x, double y, double r) { super(rr); this.x = x; this.y = y; this.r = r; }
    void draw() { renderer.circle(x, y, r); }
}
new Circle(new SvgRenderer(), 0, 0, 5).draw();
```

JDBC is a bridge: your code uses the `java.sql` API (abstraction), vendors supply drivers (implementation). SLF4J over Logback/Log4j likewise.

## Flyweight

**Intent:** share as much state as possible between many fine-grained objects to cut memory. Split state into **intrinsic** (shared, immutable: a glyph's shape, a tree's texture) and **extrinsic** (per-use, passed in: position, colour).

```ts
class TreeType {                                   // intrinsic, shared
  constructor(readonly name: string, readonly texture: ImageBitmap) {}
  draw(ctx: CanvasRenderingContext2D, x: number, y: number) { ctx.drawImage(this.texture, x, y); }
}
const treeTypes = new Map<string, TreeType>();
function getTreeType(name: string, texture: ImageBitmap) {
  if (!treeTypes.has(name)) treeTypes.set(name, new TreeType(name, texture));
  return treeTypes.get(name)!;
}
// A million trees store only {x, y, type} each; textures exist once per type.
```

Built-in examples: Java string interning and `Integer.valueOf` caching (-128..127), which is why `Integer a = 127, b = 127; a == b` is `true` but for 128 it's `false`.

## Wrapper patterns compared

| Pattern | Same interface as wrapped? | Intent |
|---|---|---|
| **Adapter** | **No**: exposes a different interface | Make incompatible things fit |
| **Decorator** | Yes | Add behaviour, stackable |
| **Proxy** | Yes | Control access (lazy, remote, auth, cache) |
| **Facade** | No: new simplified interface over **many** objects | Simplify a subsystem |

## Senior perspective

- Put an **Adapter at every third-party boundary** (payment, email, LLM providers, feature flags). It gives you a place for translation, retries, metrics and a clean test seam. Don't leak vendor types into the domain.
- Decorators are the cleanest way to add cross-cutting concerns (caching, metrics, retries, circuit breakers) without touching business code; frameworks' AOP does the same with proxies, at the cost of "magic".
- Proxies hide network calls: a remote proxy makes a slow, failure-prone call look like a cheap local method. Make latency and failure visible in the interface (async return types, explicit errors). See the *fallacies of distributed computing*.
- Facades are fine; god facades are not. If a facade grows beyond orchestration, push logic into the domain.

## Interview questions

> [!question]- Decorator vs Proxy vs Adapter?
> All wrap an object. Adapter changes the interface to one the client expects. Decorator keeps the interface and adds behaviour (stackable). Proxy keeps the interface and controls access (lazy loading, remote calls, authorisation, caching).

> [!question]- Why prefer Decorator over subclassing for adding features?
> Subclassing fixes combinations at compile time and explodes (LoggingCachingRetryingRepo). Decorators compose at runtime in any order and depend only on the interface.

> [!question]- Where have you seen Composite?
> DOM and component trees, file systems, menus, org charts, expression/AST trees; anywhere a node and a group of nodes need the same operations.

> [!question]- Why does `@Transactional` sometimes not work in Spring?
> Spring implements it with a proxy around the bean. Calling the annotated method from another method in the same class goes through `this`, not the proxy, so no transaction is started. Also, proxies only intercept public methods by default.

> [!question]- What is Bridge and when is it needed?
> Separating an abstraction hierarchy from an implementation hierarchy connected by composition, so both vary independently and you avoid N×M subclasses. Examples: JDBC API vs drivers, shapes vs renderers, SLF4J vs logging backends.

## Related
- [[Principles Index]]
- [[Design Patterns]] · [[Creational Patterns]] · [[Behavioral Patterns]]
- [[Coupling, Cohesion & Modularity]] · [[Software Architecture Styles]] (ports & adapters) · [[Domain-Driven Design (DDD)]] (anti-corruption layer)
- Frontend: [[Frontend Design Patterns]] · [[Component Patterns]]

## Sources
- Gamma et al., *Design Patterns* (1994), chapter 4 "Structural Patterns"
- [Refactoring.Guru: Structural patterns](https://refactoring.guru/design-patterns/structural-patterns)
- [MDN: Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- [Spring Framework: Understanding AOP proxies](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)
- [TypeScript 5.0 release notes: Decorators](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#decorators)
