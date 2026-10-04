---
title: "Domain-Driven Design (DDD)"
tags: [principles, principles/ddd]
aliases: [DDD, Domain Driven Design, Bounded Context, Ubiquitous Language, Aggregate, Value Object, Entity, Domain Events, Context Map, Anti-Corruption Layer]
updated: 2026-09-27
---
# Domain-Driven Design (DDD)

<Note>
**TL;DR**

**DDD** (Eric Evans, 2003) is an approach to complex software where the code's structure and language mirror the business domain, built in close collaboration with domain experts. **Strategic DDD** splits a big domain into **bounded contexts**, each with its own **ubiquitous language** and model, and maps how they relate (**context map**). **Tactical DDD** gives building blocks inside a context: **entities, value objects, aggregates, domain events, repositories, domain services**. Use the strategic part almost everywhere; use the tactical patterns where business rules are genuinely complex (the **core domain**), not for CRUD.
</Note>

## Why DDD

Most software complexity comes from the **domain** (insurance rules, logistics, payments), not the technology. When the code uses different words and shapes than the business, every requirement needs translation, and translation is where bugs live. DDD's bet: invest modelling effort where the business differentiates itself, and make the model explicit in code.

## Strategic design

### Subdomains: where to invest

| Type | Meaning | Strategy | Example (food delivery) |
|---|---|---|---|
| **Core domain** | What makes the business win | Best people, rich model, build in-house | Dispatch & delivery-time prediction |
| **Supporting subdomain** | Needed, specific to you, not a differentiator | Simpler design, maybe outsource | Restaurant onboarding |
| **Generic subdomain** | Solved problem, same for everyone | **Buy / use SaaS** | Payments, auth, email |

### Ubiquitous language

A shared, precise vocabulary used by domain experts **and** in code, tests, APIs and conversations **within one bounded context**. If the business says "a courier *accepts* an *offer*", the code has `Courier.accept(Offer)`, not `DriverJobAssignmentManager.update()`.

- Build a glossary; when a term is ambiguous, that's a modelling discovery.
- Language changes → code changes (rename refactorings are domain work).

### Bounded contexts

A **bounded context** is an explicit boundary inside which a model and its language are consistent. The same word can mean different things in different contexts, and that's fine.

{/* diagram source: /diagrams/diagram-365.mmd */}
![diagram](/diagrams/diagram-365.svg)

"Order" in Ordering is a cart with prices; in Dispatch the relevant concept is a *Delivery* with locations; in Billing it's an *Invoice*. Forcing one shared `Order` class across all three creates a bloated model everyone is afraid to touch.

Bounded contexts are the natural boundaries for **modules in a modular monolith**, **microservices** and **team ownership** (see [Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles) and [Coupling, Cohesion & Modularity](/Software%20Engineering/Principles/Coupling%2C%20Cohesion%20%26%20Modularity)). A context usually belongs to one team; a team may own several.

### Context maps

A context map documents how bounded contexts (and their teams) relate:

| Relationship | Meaning |
|---|---|
| **Partnership** | Two teams succeed or fail together; coordinate closely |
| **Shared Kernel** | Small shared model/code both depend on (use sparingly) |
| **Customer–Supplier** | Upstream supplies what downstream needs; downstream has a say in priorities |
| **Conformist** | Downstream simply adopts the upstream model (no leverage) |
| **Anti-Corruption Layer (ACL)** | Downstream translates the upstream model into its own, protecting its language ([Adapter](/Software%20Engineering/Principles/Structural%20Patterns#adapter)/Facade at context scale) |
| **Open Host Service** | Upstream offers a well-defined protocol/API for many consumers |
| **Published Language** | A documented shared exchange format (e.g. event schemas, iCal, a JSON schema) |
| **Separate Ways** | No integration; duplicate instead |

```ts
// Anti-corruption layer: legacy CRM's model never leaks into our Ordering context
class CrmCustomerTranslator {
  toCustomer(crm: LegacyCrmRecord): Customer {
    return Customer.create({
      id: CustomerId.of(crm.CUST_NO),
      name: `${crm.FNAME} ${crm.LNAME}`.trim(),
      email: EmailAddress.parse(crm.EMAIL_ADDR),
      tier: crm.VIP_FLAG === "Y" ? "gold" : "standard",
    });
  }
}
```

### Discovering the model: Event Storming

Alberto Brandolini's **Event Storming** is a workshop where developers and domain experts put orange sticky notes of **domain events** (past tense: *OrderPlaced*, *CourierAssigned*, *PaymentFailed*) on a long wall in time order, then add commands, actors, policies ("whenever X then Y"), read models and hotspots. Clusters and language shifts reveal bounded contexts.

## Tactical design

Example domain throughout: a **food-delivery Ordering context**.

### Value objects

Immutable objects defined **only by their values**, with no identity. Two `Money(10, "EUR")` are interchangeable. They validate themselves and carry behaviour, which kills primitive obsession.

```ts
export class Money {
  private constructor(readonly cents: number, readonly currency: "EUR" | "USD") {
    if (!Number.isInteger(cents) || cents < 0) throw new Error("invalid amount");
  }
  static of(cents: number, currency: "EUR" | "USD") { return new Money(cents, currency); }
  add(o: Money) {
    if (o.currency !== this.currency) throw new Error("currency mismatch");
    return new Money(this.cents + o.cents, this.currency);          // returns a new value
  }
  multiply(n: number) { return new Money(Math.round(this.cents * n), this.currency); }
  equals(o: Money) { return this.cents === o.cents && this.currency === o.currency; }
}
```

Others: `EmailAddress`, `Address`, `DateRange`, `Quantity`, `OrderId`. In Java/Kotlin these are ideal as **records/data classes** (see [Immutability Across Languages](/Software%20Engineering/Principles/Immutability%20Across%20Languages)).

### Entities

Objects with a **continuous identity** over time; attributes may change but it's still "the same" thing (`Order #123`, `Courier 42`). Equality is by ID.

### Aggregates

An **aggregate** is a cluster of entities and value objects treated as **one consistency boundary**, accessed only through its **root**. Rules (invariants) inside the aggregate are always consistent after each transaction.

Vaughn Vernon's rules of thumb:
1. Protect **true invariants** inside the boundary.
2. Design **small** aggregates.
3. Reference other aggregates **by ID only**.
4. Use **eventual consistency** (domain events) between aggregates.
5. One transaction modifies **one** aggregate.

```ts
type OrderStatus = "draft" | "placed" | "cancelled";

export class Order {                                   // aggregate root
  private lines: OrderLine[] = [];
  private status: OrderStatus = "draft";
  private events: DomainEvent[] = [];

  private constructor(readonly id: OrderId, readonly customerId: CustomerId,   // other aggregate by ID
                      readonly restaurantId: RestaurantId) {}

  static start(id: OrderId, customerId: CustomerId, restaurantId: RestaurantId) {
    return new Order(id, customerId, restaurantId);
  }

  addItem(menuItemId: MenuItemId, unitPrice: Money, qty: number) {
    if (this.status !== "draft") throw new DomainError("Order is no longer editable");
    if (qty < 1 || qty > 20) throw new DomainError("Quantity must be 1..20");
    this.lines.push(new OrderLine(menuItemId, unitPrice, qty));
  }

  total(): Money {
    return this.lines.reduce((sum, l) => sum.add(l.subtotal()), Money.of(0, "EUR"));
  }

  place(minimumOrder: Money) {                         // invariant: minimum order value
    if (this.lines.length === 0) throw new DomainError("Empty order");
    if (this.total().cents < minimumOrder.cents) throw new DomainError("Below minimum order");
    this.status = "placed";
    this.events.push({ type: "OrderPlaced", orderId: this.id, total: this.total(), at: new Date() });
  }

  pullEvents() { const e = this.events; this.events = []; return e; }
}
```

Nobody outside can add an `OrderLine` directly, so the invariants (editable only in draft, quantity limits, minimum order) can't be bypassed.

### Domain events

Something that **happened** in the domain that experts care about, named in the past tense: `OrderPlaced`, `PaymentCaptured`, `CourierAssigned`. They decouple aggregates and contexts: Billing and Dispatch react to `OrderPlaced` without Ordering knowing about them.

To publish reliably with the database write, use the **transactional outbox**: save the aggregate and the event in the same DB transaction, then a relay publishes outbox rows to the broker.

### Repositories

A collection-like interface to load and save **whole aggregates** by ID, hiding persistence. Defined in the domain (a *port*), implemented in infrastructure (an *adapter*); see [SOLID](/Software%20Engineering/Principles/SOLID) (DIP).

```ts
export interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;        // one aggregate per transaction
}
```

One repository per aggregate root, not per table.

### Domain services, application services, factories

- **Domain service**: domain logic that doesn't naturally belong to one entity (`DeliveryFeeCalculator` using distance and surge rules). Stateless, in domain language.
- **Application service / use case**: orchestrates a user intent: load aggregate, call domain methods, save, publish events. No business rules.
- **Factory**: encapsulates complex creation of aggregates (see [Creational Patterns](/Software%20Engineering/Principles/Creational%20Patterns)).

```ts
export class PlaceOrder {                                 // application service
  constructor(private orders: OrderRepository, private restaurants: RestaurantDirectory,
              private outbox: Outbox) {}
  async execute(cmd: { orderId: string }) {
    const order = await this.orders.findById(OrderId.of(cmd.orderId));
    if (!order) throw new NotFoundError();
    const minimum = await this.restaurants.minimumOrderFor(order.restaurantId);
    order.place(minimum);                                 // rules live in the aggregate
    await this.orders.save(order);
    await this.outbox.add(order.pullEvents());            // same transaction in practice
  }
}
```

### Layering inside a bounded context

{/* diagram source: /diagrams/diagram-366.mmd */}
![diagram](/diagrams/diagram-366.svg)

The domain depends on nothing; infrastructure depends on the domain. This is the core of clean/hexagonal/onion architecture ([Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles), [Rails Clean Architecture](/Software%20Engineering/Backend/Rails%20Clean%20Architecture)).

## When DDD fits (and when it doesn't)

| Fits | Overkill |
|---|---|
| Complex, evolving business rules | CRUD/admin apps, reporting |
| Access to domain experts | No one can explain the rules |
| Long-lived core system | Short-lived prototype |
| Several teams on one large domain | One small team, small domain |

Even for simple systems, **bounded contexts and ubiquitous language** are cheap and valuable. The tactical patterns are the expensive part.

## Pitfalls

<Warning>
**Common DDD mistakes**

- **Tactical-only DDD**: entities/repositories everywhere, no bounded contexts or conversations with experts. All ceremony, little benefit.
- **Anemic domain model**: aggregates are data bags, rules live in services (fine for CRUD, not for the core domain).
- **Huge aggregates** (a `Customer` containing all orders): lock contention, slow loads. Keep them small; reference by ID.
- **One canonical enterprise model** shared by all services: the opposite of bounded contexts.
- **Microservice per entity** (`OrderService`, `OrderLineService`): chatty, distributed-monolith architecture. Split by bounded context, not by noun.
- Leaking ORM annotations and framework types throughout the domain (a pragmatic compromise sometimes, but a conscious one).
</Warning>

## Senior perspective

- Start with **Event Storming** or domain storytelling when a system is new or confusing; it aligns teams faster than documents.
- Use **Team Topologies** thinking: bounded contexts, team boundaries and service boundaries should line up (Conway's law).
- **Consistency boundaries are a business question**: ask experts "must these two things always be consistent immediately, or is a few seconds acceptable?" It usually is, which enables small aggregates and events.
- Version your events and published language; they're public contracts.
- Invest the rich model only in the core domain; buy generic subdomains.

## Interview questions

<Accordion title="What is a bounded context and why does it matter?">
An explicit boundary within which one model and ubiquitous language apply consistently. It lets different parts of the business use the same term with different meanings without a bloated shared model, and it's the natural unit for modules, services and team ownership.
</Accordion>

<Accordion title="Entity vs value object?">
An entity has an identity that persists as attributes change (equality by ID, e.g. Order #123). A value object has no identity, is defined by its attributes, is immutable and replaceable (Money, Address).
</Accordion>

<Accordion title="What is an aggregate? How big should it be?">
A cluster of domain objects treated as one consistency boundary, changed only through its root, with invariants enforced in each transaction. As small as the true invariants allow; reference other aggregates by ID and use domain events for eventual consistency between them.
</Accordion>

<Accordion title="What is an anti-corruption layer?">
A translation layer between your bounded context and another (often legacy or third-party) system, converting their model into yours so their concepts don't leak into your domain.
</Accordion>

<Accordion title="Why one aggregate per transaction?">
Multi-aggregate transactions create contention and couple aggregates, and they don't scale to distributed systems. If a use case changes several aggregates, update one and react to its domain event to update the others (eventually consistent), or reconsider the boundaries.
</Accordion>

<Accordion title="How do microservices relate to DDD?">
Bounded contexts are a good first cut for service boundaries (one context → one or a few services, never one service spanning several contexts). Domain events become integration events between services. Many teams start with a modular monolith using the same boundaries.
</Accordion>

<Accordion title="When would you not use DDD?">
For simple CRUD, reporting, or generic subdomains, and when there's no access to domain experts. Strategic ideas (contexts, language) still help; the tactical patterns would be overhead.
</Accordion>

## Related
- [Principles Index](/Software%20Engineering/Principles/Principles%20Index)
- [Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles) · [Coupling, Cohesion & Modularity](/Software%20Engineering/Principles/Coupling%2C%20Cohesion%20%26%20Modularity) · [Behavior-Driven Development (BDD)](/Software%20Engineering/Principles/Behavior-Driven%20Development%20(BDD))
- [OOP](/Software%20Engineering/Principles/OOP) · [Immutability Across Languages](/Software%20Engineering/Principles/Immutability%20Across%20Languages) · [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns) · [Rails Clean Architecture](/Software%20Engineering/Backend/Rails%20Clean%20Architecture)

## Sources
- Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software* (Addison-Wesley, 2003) · [DDD Reference (free PDF, 2015)](https://www.domainlanguage.com/ddd/reference/)
- Vaughn Vernon, *Implementing Domain-Driven Design* (2013) · [Effective Aggregate Design (PDF)](https://www.dddcommunity.org/library/vernon_2011/)
- Vlad Khononov, *Learning Domain-Driven Design* (O'Reilly, 2021)
- [Martin Fowler: BoundedContext](https://martinfowler.com/bliki/BoundedContext.html) · [DDD_Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
- [DDD Crew: Context Mapping](https://github.com/ddd-crew/context-mapping)
- Alberto Brandolini, [Introducing EventStorming](https://www.eventstorming.com/)
- [Chris Richardson: Transactional outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html)
