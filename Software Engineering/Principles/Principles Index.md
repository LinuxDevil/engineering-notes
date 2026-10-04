---
title: "Principles Index"
tags: [principles, index]
aliases: [Principles Home, Design Principles, Software Design Principles, Architecture Principles, Principles MOC]
updated: 2026-09-27
---
# Principles Index

<Note>
**Start here**

Entry point for **software design principles and architecture**: how to structure code so it stays easy to change, from OOP basics and SOLID through design patterns, clean code, testing practices (TDD/BDD), domain-driven design and architecture styles. Every note has worked examples, a 🆕 2025–2026 box where something changed, and interview questions. Read top to bottom for a beginner → senior path.
</Note>

{/* diagram source: /diagrams/diagram-372.mmd */}
![diagram](/diagrams/diagram-372.svg)

## 🌱 Beginner: paradigms and building blocks
| Note | What you'll learn |
|---|---|
| [OOP](/Software%20Engineering/Principles/OOP) | Classes, objects, the four pillars, overloading vs overriding, composition, data-oriented Java |
| [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface) | When to use each, modern Java rules (default/private/sealed), other languages |
| [Immutability Across Languages](/Software%20Engineering/Principles/Immutability%20Across%20Languages) | `const` vs `final` vs `val`, `Object.freeze`, records, Dart `const` |
| [DRY, KISS & YAGNI](/Software%20Engineering/Principles/DRY%2C%20KISS%20%26%20YAGNI) | The three everyday heuristics and how they conflict |
| [MVC](/Software%20Engineering/Principles/MVC) | Model–View–Controller and its cousins MVP, MVVM, MVI |

## 🌿 Intermediate: design principles and patterns
| Note | What you'll learn |
|---|---|
| [SOLID](/Software%20Engineering/Principles/SOLID) | The five principles with Java/TypeScript examples and trade-offs |
| [SOLID Principles with Ruby](/Software%20Engineering/Principles/SOLID%20Principles%20with%20Ruby) | The same principles in Ruby (duck typing, roles, injection) |
| [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns) | The GoF map, problem → pattern table, Rails application patterns |
| [Creational Patterns](/Software%20Engineering/Principles/Creational%20Patterns) | Singleton, Factory Method, Abstract Factory, Builder, Prototype |
| [Structural Patterns](/Software%20Engineering/Principles/Structural%20Patterns) | Adapter, Decorator, Proxy, Facade, Composite, Bridge, Flyweight |
| [Behavioral Patterns](/Software%20Engineering/Principles/Behavioral%20Patterns) | Strategy, Observer, Command, State, Template Method, Chain of Responsibility, Visitor... |
| [Functional Programming Principles](/Software%20Engineering/Principles/Functional%20Programming%20Principles) | Pure functions, HOFs, composition, functional core / imperative shell |
| [Clean Code & Refactoring](/Software%20Engineering/Principles/Clean%20Code%20%26%20Refactoring) | Naming, functions, code smells, refactoring catalogue with before/after |
| [Test-Driven Development (TDD)](/Software%20Engineering/Principles/Test-Driven%20Development%20(TDD)) | Red–green–refactor, test doubles, pyramid vs trophy vs honeycomb |
| [Behavior-Driven Development (BDD)](/Software%20Engineering/Principles/Behavior-Driven%20Development%20(BDD)) | Discovery, Given/When/Then, example mapping, double-loop TDD |
| [Dependency Injection & Inversion of Control](/Software%20Engineering/Principles/Dependency%20Injection%20%26%20Inversion%20of%20Control) | DI vs DIP vs IoC, composition root, containers and scopes |

## 🌳 Senior: modularity and architecture
| Note | What you'll learn |
|---|---|
| [Coupling, Cohesion & Modularity](/Software%20Engineering/Principles/Coupling%2C%20Cohesion%20%26%20Modularity) | Deep vs shallow modules, information hiding, connascence, Law of Demeter |
| [Domain-Driven Design (DDD)](/Software%20Engineering/Principles/Domain-Driven%20Design%20(DDD)) | Bounded contexts, context maps, aggregates, value objects, domain events |
| [Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles) | Layered, hexagonal/clean, modular monolith, microservices, event-driven, serverless, CQRS/ES |
| [Architecture Decision Records (ADRs) & Technical Debt](/Software%20Engineering/Principles/Architecture%20Decision%20Records%20(ADRs)%20%26%20Technical%20Debt) | Recording decisions, managing debt as a business tool |
| [Concurrency And Parallelism](/Software%20Engineering/Principles/Concurrency%20And%20Parallelism) | Processes, threads, races, deadlocks, virtual threads, structured concurrency |

Elsewhere in the vault: [Rails Clean Architecture](/Software%20Engineering/Backend/Rails%20Clean%20Architecture) (Backend) · [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture) · [Frontend Design Patterns](/Software%20Engineering/Frontend/09%20Architecture/Frontend%20Design%20Patterns) · [Frontend Testing](/Software%20Engineering/Frontend/13%20Testing/Frontend%20Testing) · [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index)

<Info>
**🧭 Suggested study path**

1. **Foundations (week 1):** [OOP](/Software%20Engineering/Principles/OOP) → [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface) → [Immutability Across Languages](/Software%20Engineering/Principles/Immutability%20Across%20Languages) → [Functional Programming Principles](/Software%20Engineering/Principles/Functional%20Programming%20Principles)
2. **Principles (week 2):** [DRY, KISS & YAGNI](/Software%20Engineering/Principles/DRY%2C%20KISS%20%26%20YAGNI) → [SOLID](/Software%20Engineering/Principles/SOLID) (+ [SOLID Principles with Ruby](/Software%20Engineering/Principles/SOLID%20Principles%20with%20Ruby)) → [Dependency Injection & Inversion of Control](/Software%20Engineering/Principles/Dependency%20Injection%20%26%20Inversion%20of%20Control) → [Coupling, Cohesion & Modularity](/Software%20Engineering/Principles/Coupling%2C%20Cohesion%20%26%20Modularity)
3. **Patterns (week 3):** [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns) → [Creational Patterns](/Software%20Engineering/Principles/Creational%20Patterns) → [Structural Patterns](/Software%20Engineering/Principles/Structural%20Patterns) → [Behavioral Patterns](/Software%20Engineering/Principles/Behavioral%20Patterns) → [MVC](/Software%20Engineering/Principles/MVC)
4. **Craft (week 4):** [Clean Code & Refactoring](/Software%20Engineering/Principles/Clean%20Code%20%26%20Refactoring) → [Test-Driven Development (TDD)](/Software%20Engineering/Principles/Test-Driven%20Development%20(TDD)) → [Behavior-Driven Development (BDD)](/Software%20Engineering/Principles/Behavior-Driven%20Development%20(BDD))
5. **Architecture (week 5+):** [Domain-Driven Design (DDD)](/Software%20Engineering/Principles/Domain-Driven%20Design%20(DDD)) → [Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles) → [Architecture Decision Records (ADRs) & Technical Debt](/Software%20Engineering/Principles/Architecture%20Decision%20Records%20(ADRs)%20%26%20Technical%20Debt) → [Concurrency And Parallelism](/Software%20Engineering/Principles/Concurrency%20And%20Parallelism)
</Info>

<Info>
**🎯 Interview crash course**

[SOLID](/Software%20Engineering/Principles/SOLID) · [OOP](/Software%20Engineering/Principles/OOP) (pillars, overloading vs overriding) · [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface) · [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns) (+ Strategy, Observer, Decorator, Adapter, Builder, Singleton) · [Concurrency And Parallelism](/Software%20Engineering/Principles/Concurrency%20And%20Parallelism) · [Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles) (monolith vs microservices, hexagonal, CQRS) · [Domain-Driven Design (DDD)](/Software%20Engineering/Principles/Domain-Driven%20Design%20(DDD)) (bounded context, aggregate) · [Test-Driven Development (TDD)](/Software%20Engineering/Principles/Test-Driven%20Development%20(TDD)) (mocks vs stubs)
</Info>

## 🆕 2025–2026 in this area
- **Java 25 LTS (2025-09-16)**: flexible constructor bodies, compact source files, module imports; data-oriented style (records + sealed + patterns) is idiomatic → [OOP](/Software%20Engineering/Principles/OOP)
- **Java 24 (2025-03)**: virtual threads stop pinning in `synchronized` (JEP 491); **Java 25** finalises Scoped Values; Structured Concurrency still preview through **Java 27 (2026-09-15)** → [Concurrency And Parallelism](/Software%20Engineering/Principles/Concurrency%20And%20Parallelism)
- **Python 3.14 (2025-10)**: free-threaded build officially supported; multiple interpreters in stdlib → [Concurrency And Parallelism](/Software%20Engineering/Principles/Concurrency%20And%20Parallelism)
- **Java 26 (2026-03-17)**: JEP 500 "make final mean final" warnings; Stable Values renamed **Lazy Constants** → [Immutability Across Languages](/Software%20Engineering/Principles/Immutability%20Across%20Languages)
- **TC39 withdrew Records & Tuples (2025-04)**; Composites proposal follows → [Immutability Across Languages](/Software%20Engineering/Principles/Immutability%20Across%20Languages)
- **Clean Code 2nd edition (2025-10)** and the Ousterhout vs Martin debate (2024-09 → 2025-02) → [Clean Code & Refactoring](/Software%20Engineering/Principles/Clean%20Code%20%26%20Refactoring), [Coupling, Cohesion & Modularity](/Software%20Engineering/Principles/Coupling%2C%20Cohesion%20%26%20Modularity)
- **AI assistants and code quality**: GitClear duplication study (2025-02), DORA 2025 "AI as amplifier" (2025-09) → [DRY, KISS & YAGNI](/Software%20Engineering/Principles/DRY%2C%20KISS%20%26%20YAGNI)
- **TDD with AI agents**: Kent Beck's "Augmented Coding" (2025-06-25) → [Test-Driven Development (TDD)](/Software%20Engineering/Principles/Test-Driven%20Development%20(TDD))
- **Spec-driven development**: AWS Kiro (2025-07-14), GitHub Spec Kit (2025-09-02), Thoughtworks Radar "Assess" (2025-11) → [Test-Driven Development (TDD)](/Software%20Engineering/Principles/Test-Driven%20Development%20(TDD)), [Behavior-Driven Development (BDD)](/Software%20Engineering/Principles/Behavior-Driven%20Development%20(BDD))
- **Modular monolith mainstream**: *Fundamentals of Software Architecture* 2nd ed. (2025), Spring Modulith 2.0 (2025-11-21) and 2.1 (2026-06-11) → [Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles)
- **AGENTS.md (2025-08) and the Agentic AI Foundation (2025-12-09)**: decision records as AI context → [Architecture Decision Records (ADRs) & Technical Debt](/Software%20Engineering/Principles/Architecture%20Decision%20Records%20(ADRs)%20%26%20Technical%20Debt)
- **Kotlin context parameters** stable in Kotlin 2.4.0 (2026-07-14) → [Dependency Injection & Inversion of Control](/Software%20Engineering/Principles/Dependency%20Injection%20%26%20Inversion%20of%20Control)

## Related
- [Software Engineering](/Software%20Engineering/Software%20Engineering), the vault root
- [Frontend Home](/Software%20Engineering/Frontend/Frontend%20Home) for frontend-specific patterns, testing and architecture
