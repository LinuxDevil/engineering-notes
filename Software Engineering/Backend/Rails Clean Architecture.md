---
title: "Rails Clean Architecture"
tags: [backend, backend/architecture, backend/rails]
aliases: [Clean Architecture in Rails, Service Objects, Rails Service Objects, PORO, Fat Model Skinny Controller, Use Cases]
updated: 2026-09-27
---
# Rails Clean Architecture

<Note>
**TL;DR**

Rails' defaults (MVC + Active Record) are great for CRUD but push business logic into fat models or controllers as apps grow. **Clean Architecture** keeps business rules independent of frameworks by splitting code into **entities** (enterprise rules), **use cases** (application actions, usually Rails "service objects" written with the **Command pattern**) and **adapters** (DB, HTTP APIs, mailers). In Rails you can adopt it gradually: `app/services` for use cases, POROs for domain logic, `app/adapters` for external systems, and keep controllers skinny. Don't add the layers to simple CRUD.
</Note>

## Building blocks

- **PORO**: *Plain Old Ruby Object*, basically a Ruby class that doesn't inherit from a Rails base class. Easy to test, no framework magic.
- **Services**: a **service** provides a piece of functionality used by other components (here, controllers, jobs, rake tasks). Services aren't an intrinsic Rails feature but a popular design convention (frameworks like Angular use the same idea).
  - Files in `app/services` are **autoloaded** by Zeitwerk like any `app/*` folder; in production they're **eager loaded** (`config.eager_load = true`). Keep file and constant names matching (`app/services/orders/place_order.rb` → `Orders::PlaceOrder`) or eager loading fails at boot.

### The Service Object pattern

The Service Object pattern creates classes that represent **business logic or behaviour** that would otherwise sit too close to the database in an Active Record model, or in the controller.

Breaking business rules into **entities**, **use cases** and **adapters** is a fairer split of labour than simply lifting and shifting controller code into one big service object. That said, when an application is (mostly) simple CRUD, this extra layer of complexity isn't needed at all.

## Clean Architecture in one picture

{/* diagram source: /diagrams/diagram-057.mmd */}
![diagram](/diagrams/diagram-057.svg)

**The dependency rule**: source code dependencies point **inwards**. Entities know nothing about use cases; use cases know nothing about controllers, HTTP or Stripe; they talk to adapters through small interfaces (in Ruby, duck-typed objects injected via the constructor).

## Services (and other friends): the patterns used

### Command pattern

A **behavioural** pattern that fully decouples the **sender** (the object that invokes an operation) from the **receiver** (the object that performs it). A command object encapsulates a request with everything needed to execute it, typically exposing one method (`call`/`execute`). Rails service objects with a single `call` are commands.

### Adapter pattern

An Adapter "**converts the interface of a class into another interface that a client wants**": it lets you use a class with an incompatible interface through the interface your code expects. Also known as **Wrapper**. In Rails: wrappers around external APIs, payment providers, storage.

### Active Record pattern

An **architectural pattern** (Martin Fowler, *Patterns of Enterprise Application Architecture*) for data access:
- An Active Record object **wraps a row** in a database table or view, **encapsulates the database access** and **adds domain logic** on that data. One object carries both data and behaviour.
- Its interface includes `insert`/`update`/`delete` (`save`, `destroy`) and attributes that map almost directly to table columns.
- Simple and productive, but it couples domain logic to the persistence schema, which is exactly what Clean Architecture tries to loosen.

### Mapping Clean Architecture to Rails

- **Use cases → `app/services`.** The Services convention in Rails matches use-case classes representing actions a user can take on the system, especially with the Command pattern approach, the same convention Clean Architecture uses. *Just put your use cases in `app/services`.*
- **Entities → domain objects.** They represent enterprise-level business rules: code likely shared across use cases and business functions. Introduce them only when you need to share behaviour between use cases, or to abstract use cases from the database and Active Record. ActiveModel (`ActiveModel::Model`, `ActiveModel::Attributes`) helps define them. Debate with your team whether they sit in `app/models` beside the DB models or in `app/domain`.
- **Adapters → `app/adapters`.** Rails developers already use the Adapter pattern for external APIs and databases.

## Worked example

```ruby
# app/controllers/orders_controller.rb: skinny, only HTTP concerns
class OrdersController < ApplicationController
  def create
    result = Orders::PlaceOrder.new.call(user: current_user, params: order_params)
    if result.success?
      render json: result.order, status: :created
    else
      render json: { errors: result.errors }, status: :unprocessable_entity
    end
  end
end
```

```ruby
# app/services/orders/place_order.rb: use case (Command)
module Orders
  class PlaceOrder
    Result = Data.define(:success?, :order, :errors)

    def initialize(payments: Adapters::StripePayments.new, notifier: OrderMailer)
      @payments, @notifier = payments, notifier          # dependencies injected → easy to fake
    end

    def call(user:, params:)
      cart = Domain::Cart.new(params[:items])             # entity: pure business rules
      return Result.new(false, nil, cart.errors) unless cart.valid?

      order = Order.transaction do
        o = Order.create!(user:, total_cents: cart.total_cents, status: "pending")
        charge = @payments.charge(amount_cents: o.total_cents, idempotency_key: "order-#{o.id}")
        o.update!(status: "paid", charge_id: charge.id)
        o
      end
      @notifier.confirmation(order.id).deliver_later       # background job
      Result.new(true, order, [])
    rescue Adapters::PaymentDeclined => e
      Result.new(false, nil, [e.message])
    end
  end
end
```

```ruby
# app/domain/cart.rb: entity (PORO), no database
module Domain
  class Cart
    attr_reader :errors
    def initialize(items) = (@items = items; @errors = [])
    def valid?
      @errors << "cart is empty" if @items.blank?
      @errors.empty?
    end
    def total_cents = @items.sum { |i| i[:price_cents] * i[:qty] }
  end
end
```

```ruby
# app/adapters/stripe_payments.rb: adapter wraps a third-party API
module Adapters
  class PaymentDeclined < StandardError; end
  class StripePayments
    def charge(amount_cents:, idempotency_key:)
      Stripe::PaymentIntent.create({ amount: amount_cents, currency: "eur", confirm: true },
                                   { idempotency_key: })
    rescue Stripe::CardError => e
      raise PaymentDeclined, e.message
    end
  end
end
```

```ruby
# spec: the use case is tested with a fake adapter, no network
it "places an order" do
  fake = double(charge: double(id: "ch_1"))
  result = Orders::PlaceOrder.new(payments: fake, notifier: double(confirmation: double(deliver_later: true)))
                             .call(user:, params: { items: [{ price_cents: 500, qty: 2 }] })
  expect(result.success?).to be true
end
```

Note: calling an external API inside a DB transaction holds the transaction open during network I/O; for high volume, create the order, commit, then charge in a job ([Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues)).

## Fat Model, Skinny Controller

"Fat Model, Skinny Controller" describes how the **M** and **C** of **MVC** ([MVC](/Software%20Engineering/Principles/MVC)) should work together: any **non-response-related logic goes in the model**, ideally in small, testable methods, while the "skinny" controller is just an interface between the view and the model.

It was the Rails community's first answer to fat controllers. Its limit: models become "god objects" with dozens of callbacks and concerns. Service objects/use cases are the next step: **skinny controller, lean model, explicit use cases**.

| Approach | Where logic lives | Good for | Breaks down when |
|---|---|---|---|
| Rails way (fat model) | models, callbacks, concerns | CRUD, small teams, prototypes | callbacks cascade, models > 1,000 lines |
| Service objects | `app/services` | multi-model workflows, external calls | services become procedural scripts with no domain model |
| Full Clean/Hexagonal | entities + use cases + ports/adapters | complex domains, long-lived systems | overkill for CRUD; ceremony slows the team |

Related ideas: **Hexagonal architecture** (ports and adapters), **DDD** (entities, value objects, aggregates, bounded contexts), **form objects**, **query objects**, **Packwerk** (Shopify's tool to enforce boundaries between packages inside a Rails monolith, the "modular monolith").

## Pitfalls

<Warning>
**Common mistakes**

- "Service object" as a dumping ground: `UserService` with 40 methods. Prefer one use case per class, named by the action (`Orders::PlaceOrder`).
- Inconsistent return conventions (some raise, some return `false`, some return hashes). Pick one (a `Result` object, or dry-monads).
- Callbacks that send emails or call APIs (`after_save :notify`) hidden in models; move side effects into use cases or jobs.
- Adding entities/repositories on top of Active Record for plain CRUD, doubling code for no benefit.
- Leaking Active Record objects deep into the domain, making "pure" logic depend on the DB anyway.
</Warning>

## Senior perspective

- Introduce layers **where complexity is**, not everywhere. A billing domain deserves use cases and entities; the admin CRUD doesn't.
- Enforce boundaries with tooling (Packwerk, Zeitwerk namespaces, code review), otherwise they erode.
- The main payoff is **testability and changeability**: fast unit tests without the DB, swapping Stripe for another provider behind the adapter.
- Review questions: *Does this use case depend on HTTP or Rails internals? What does it return on failure? Where are side effects triggered and are they idempotent?*

## Interview questions

<Accordion title="What is a service object in Rails and why use it?">
A PORO encapsulating one business action (a use case) with a single public method such as `call`. It keeps controllers and models small, groups multi-model workflows and external calls in one testable place, and makes dependencies explicit.
</Accordion>

<Accordion title="Explain Clean Architecture's dependency rule.">
Source code dependencies only point inwards: frameworks and adapters depend on use cases, use cases depend on entities, and inner layers never reference outer ones. Outer details (DB, web, third-party APIs) are plugged in through interfaces.
</Accordion>

<Accordion title="What is the Active Record pattern and its trade-off?">
An object that wraps a database row, carrying both data and persistence plus domain behaviour. It is simple and productive, but couples business logic to the database schema and makes pure unit testing harder, versus the Data Mapper/repository approach.
</Accordion>

<Accordion title="Command vs Adapter pattern?">
Command encapsulates a request as an object so the invoker is decoupled from the receiver (service objects with `call`). Adapter converts one interface into another that clients expect, wrapping external libraries or APIs.
</Accordion>

<Accordion title="When would you not use Clean Architecture in Rails?">
For mostly CRUD apps, prototypes or small teams, where Rails conventions are faster and extra layers only add ceremony. Introduce it for complex domains, heavy external integrations or long-lived codebases.
</Accordion>

## Related
- [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) · [Cron Jobs](/Software%20Engineering/Backend/Cron%20Jobs) · [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility)
- [MVC](/Software%20Engineering/Principles/MVC) · [SOLID](/Software%20Engineering/Principles/SOLID) · [SOLID Principles with Ruby](/Software%20Engineering/Principles/SOLID%20Principles%20with%20Ruby) · [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns) · [OOP](/Software%20Engineering/Principles/OOP)

## Sources
- Robert C. Martin, *Clean Architecture* (2017) · [The Clean Architecture (blog, 2012)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- Martin Fowler, *Patterns of Enterprise Application Architecture* · [Active Record](https://martinfowler.com/eaaCatalog/activeRecord.html)
- [Refactoring.Guru: Command](https://refactoring.guru/design-patterns/command) · [Adapter](https://refactoring.guru/design-patterns/adapter)
- [Rails Guides: Autoloading and reloading constants (Zeitwerk)](https://guides.rubyonrails.org/autoloading_and_reloading_constants.html)
- [Alistair Cockburn: Hexagonal architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [Packwerk (Shopify)](https://github.com/Shopify/packwerk)
