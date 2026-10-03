---
tags: [principles, principles/solid, ruby]
aliases: [SOLID in Ruby, Ruby SOLID]
updated: 2026-09-27
---
# SOLID Principles with Ruby

> [!tldr] TL;DR
> Companion to [[SOLID]]: the same five principles shown in **Ruby** with a running *painter/printer* example. Ruby is dynamically typed and has no `interface` keyword, so the principles are expressed through **duck typing**, small classes, modules and **constructor injection**. The theory, trade-offs and interview questions live in [[SOLID]]; this note is the Ruby lens.

**SOLID** = **S**ingle Responsibility, **O**pen–Closed, **L**iskov Substitution, **I**nterface Segregation, **D**ependency Inversion (Robert C. Martin). What changes in Ruby:

| Principle | Statically typed (Java) | Ruby idiom |
|---|---|---|
| SRP | Split classes | Split classes; extract POROs, service objects, concerns sparingly |
| OCP | Subclasses / interfaces | Duck-typed collaborators, registries (hash of classes), modules |
| LSP | Enforced partly by the compiler | Only by **tests** and discipline; `raise NotImplementedError` in base classes |
| ISP | Small interfaces | Small **roles** (sets of messages); modules as mixins; don't inherit what you don't need |
| DIP | Inject interfaces | Inject any object that responds to the right messages |

## Single Responsibility Principle (SRP)

> "A class should have one, and only one, reason to change." (Robert C. Martin)

Every class should have one job. Here `Painter` both paints and prints, so it changes when painting rules change **and** when printing changes:

```ruby
# ❌ Two reasons to change
class Painter
  def paint
    # painting functionality
    print_paint
  end

  def print_paint
    # printing functionality
  end
end
```

Separate the responsibilities:

```ruby
# ✅ One reason to change each
class Painter
  def paint
    # painting functionality; returns a Painting
  end
end

class Printer
  def print(painting)
    # printing functionality
  end
end

painting = Painter.new.paint
Printer.new.print(painting)
```

This decouples the responsibilities and makes change predictable: we can change printing without touching painting.

## Open–Closed Principle (OCP)

> Software entities (classes, modules, functions) should be *open* for **extension** but *closed* for **modification**. Martin has called it "the most important principle of object-oriented design".

Write code so you can **add new behaviour without changing existing code**. Our `Printer` is generic; what if we now need to print on A4 paper, or on canvas? Editing `Printer#print` with a new `if` each time risks breaking what works. Instead, give each variant its own class and select it through a registry:

```ruby
class CanvasPrinter
  def print(painting)
    # print on canvas
  end
end

class PaperPrinter
  def initialize(size: :a4)
    @size = size
  end

  def print(painting)
    # print on paper of @size
  end
end

# New printers are added here (or registered by plugins); Painter never changes
PRINTERS = {
  canvas: CanvasPrinter,
  paper:  PaperPrinter
}.freeze

class PrintJob
  def initialize(printers: PRINTERS)
    @printers = printers
  end

  def call(painting, type)
    printer_class = @printers.fetch(type) { raise ArgumentError, "unknown printer #{type}" }
    printer_class.new.print(painting)
  end
end

PrintJob.new.call(painting, :canvas)
```

Adding a `PosterPrinter` means adding a class and a registry entry; `PrintJob` is closed for modification.

## Liskov Substitution Principle (LSP)

Derived classes must be completely substitutable for their base classes. We made the design extensible (OCP), so callers no longer know which concrete class they get. That only works if every variant **behaves consistently**:
- accepts the same number and kinds of arguments,
- returns the same kind of value,
- responds to the same method names,
- raises no surprising errors and keeps the base class's promises.

Define a base class that documents the contract:

```ruby
class Artist
  # @return [Painting]
  def paint
    raise NotImplementedError, "#{self.class} must implement #paint"
  end
end

class ComicPainter < Artist
  def paint
    # paints comics, returns a Painting
  end
end

class LandscapePainter < Artist
  def paint
    # paints landscapes, returns a Painting
  end
end

[ComicPainter.new, LandscapePainter.new].each { |artist| Printer.new.print(artist.paint) }
```

Any subclass must implement `paint`, or calling it raises. The caller can rely on `paint` existing and returning a painting.

> [!warning] Ruby can't enforce LSP for you
> `raise NotImplementedError` only fails at runtime, and nothing checks return types. Protect substitutability with a **shared example group** that every implementation runs:
> ```ruby
> RSpec.shared_examples "an artist" do
>   it "returns a Painting from #paint" do
>     expect(subject.paint).to be_a(Painting)
>   end
> end
>
> RSpec.describe ComicPainter    do it_behaves_like "an artist" end
> RSpec.describe LandscapePainter do it_behaves_like "an artist" end
> ```
> A subclass that violates LSP, e.g. `def paint = nil` or one that requires an extra argument, fails this contract test.

## Interface Segregation Principle (ISP)

> "Clients should not be forced to depend upon interfaces that they don't use." (Robert C. Martin)

Ruby has no `interface` keyword, but it has **roles**: the set of messages an object is expected to respond to. ISP in Ruby means: keep roles small, and don't make classes inherit (or mix in) behaviour they don't need.

The original `Artist` is generic and tends to grow. Suppose it gains `sketch` and `frame` too. A `Sketcher` then inherits `paint` and `frame` it can't honour:

```ruby
# ❌ One fat base class for every kind of artist
class Artist
  def paint  = raise(NotImplementedError)
  def sketch = raise(NotImplementedError)
  def frame  = raise(NotImplementedError)
end

class Sketcher < Artist
  def sketch = "pencil drawing"
  # paint and frame are inherited but meaningless for a Sketcher
end
```

Split the behaviour into small role modules and include only what each class needs:

```ruby
# ✅ Small roles
module Paints
  def paint = raise(NotImplementedError, "#{self.class}#paint")
end

module Sketches
  def sketch = raise(NotImplementedError, "#{self.class}#sketch")
end

class LandscapePainter
  include Paints
  def paint = "oil landscape"
end

class Sketcher
  include Sketches
  def sketch = "pencil drawing"
end

class Illustrator                 # needs both roles
  include Paints
  include Sketches
  def paint  = "comic colouring"
  def sketch = "comic pencils"
end
```

Clients also segregate on their side: a `Gallery` that only exhibits needs objects that respond to `paint`; it shouldn't care about `sketch`. Check with `respond_to?(:paint)` at boundaries, or better, with tests. (Ruby 3's RBS and Sorbet can declare interfaces statically if the team uses type checking.)

## Dependency Inversion Principle (DIP)

> Depend on abstractions, not concretions. High-level modules should not depend on low-level modules; both depend on abstractions. **Abstractions should not depend on details; details should depend on abstractions.**

To paint the next Mona Lisa, a painter needs dimensions from a database. If `ComicPainter` opens a MySQL connection itself, business logic is welded to the database. Invert it: the painter depends on an abstract *data source role*, and the caller decides which concrete source to pass in.

```ruby
class DataSource
  def query(sql) = raise(NotImplementedError)
end

class MySQLSource < DataSource
  def initialize(config)
    @config = config
  end
  def query(sql)
    # run SQL against MySQL
  end
end

class InMemorySource < DataSource        # handy for tests
  def initialize(rows)
    @rows = rows
  end
  def query(_sql) = @rows
end

class ComicPainter < Artist
  def initialize(data_source)
    @data_source = data_source            # injected, not created
  end

  def paint
    dims = @data_source.query("SELECT width, height FROM paintings WHERE name = 'mona_lisa'")
    # paint using dims
  end
end

# Production wiring (composition root)
ComicPainter.new(MySQLSource.new(Rails.configuration.database_configuration))
# Test wiring
ComicPainter.new(InMemorySource.new([{ width: 77, height: 53 }]))
```

Thanks to duck typing, the `DataSource` base class is optional: any object that responds to `query` works. It remains useful as documentation and a place for the `NotImplementedError` guard. We removed hard-coded dependencies and made the painter testable without a database.

## Summary

SOLID helps decouple code and make modification cheaper. The five principles support each other: small responsibilities (SRP) produce small roles (ISP); callers depend on those roles (DIP); new variants plug in (OCP) as long as they honour the contract (LSP). A well-designed Ruby codebase is adaptable, easy to modify and pleasant to join.

> [!tip] Rails context
> In Rails apps these principles usually show up as service objects, form objects, query objects and policies extracted from fat models and controllers. See [[Design Patterns#Application-level patterns (Rails and friends)]] and [[Rails Clean Architecture]].

## Interview questions

> [!question]- How do you apply ISP in a language without interfaces?
> Think in roles (the messages a collaborator must understand). Keep them small, express them as modules or documented duck types, avoid inheriting unrelated behaviour, and verify roles with shared contract tests (or RBS/Sorbet interfaces).

> [!question]- How do you enforce LSP in Ruby?
> Only through tests and conventions: base classes that raise `NotImplementedError`, RSpec shared examples that every implementation runs, and optionally static checking with Sorbet/RBS.

> [!question]- What does dependency injection look like in Ruby?
> Pass collaborators into `initialize` (often with defaults: `def initialize(source: MySQLSource.new)`), so tests or callers can substitute any object with the same interface. No container is needed.

## Related
- [[Principles Index]]
- [[SOLID]] (main note: theory, Java/TypeScript examples, trade-offs)
- [[Abstract Class and Interface]] · [[Dependency Injection & Inversion of Control]] · [[Design Patterns]] · [[Rails Clean Architecture]]

## Sources
- Robert C. Martin, [The Principles of OOD](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)
- Sandi Metz, *Practical Object-Oriented Design: An Agile Primer Using Ruby*, 2nd ed. (Addison-Wesley, 2018): roles, duck typing, dependency injection
- [RSpec: shared examples](https://rspec.info/features/3-13/rspec-core/example-groups/shared-examples/)
- [RBS: Ruby type signatures (interfaces)](https://github.com/ruby/rbs/blob/master/docs/syntax.md)
