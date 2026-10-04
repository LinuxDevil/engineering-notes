---
title: "Dependency Injection & Inversion of Control"
tags: [principles, principles/solid]
aliases: [Dependency Injection, DI, Inversion of Control, IoC, IoC Container, DI Container, Composition Root, Service Locator, Hollywood Principle]
updated: 2026-09-27
---
# Dependency Injection & Inversion of Control

<Note>
**TL;DR**

**Inversion of Control (IoC)**: a framework or caller controls the flow and calls your code ("don't call us, we'll call you"). **Dependency Injection (DI)** is one form of IoC: an object **receives** its collaborators (usually through the constructor) instead of creating or looking them up. It's the main technique for applying the **Dependency Inversion Principle** ([SOLID](/Software%20Engineering/Principles/SOLID)). Wire everything in one **composition root**; a **DI container** (Spring, NestJS, Angular, .NET) automates the wiring but isn't required.
</Note>

## The three terms

| Term | Kind | One-liner |
|---|---|---|
| **Dependency Inversion Principle** | Design principle | High-level policy and low-level details both depend on abstractions owned by the policy side |
| **Inversion of Control** | Broad idea | The framework calls your code (event loop, servlet container, React rendering your components, test runner) |
| **Dependency Injection** | Technique | Dependencies are passed in from outside rather than created inside |
| **DI container** | Tool | Library that constructs objects and injects their dependencies automatically |

## Without and with DI

```ts
// ❌ Creates its own dependencies: hard-wired, untestable without a real DB and SMTP
class SignupService {
  private repo = new PostgresUserRepository(process.env.DB_URL!);
  private mailer = new SmtpMailer("smtp.example.com");

  async signup(email: string) {
    const user = await this.repo.create(email);
    await this.mailer.send(email, "Welcome!");
    return user;
  }
}

// ✅ Constructor injection: depends on interfaces, receives implementations
interface UserRepository { create(email: string): Promise<User>; }
interface Mailer { send(to: string, body: string): Promise<void>; }

class SignupService {
  constructor(private repo: UserRepository, private mailer: Mailer, private clock: () => Date = () => new Date()) {}
  async signup(email: string) {
    const user = await this.repo.create(email);
    await this.mailer.send(email, `Welcome! Joined ${this.clock().toISOString()}`);
    return user;
  }
}

// Test: no DB, no SMTP
const sent: string[] = [];
const service = new SignupService(
  { create: async email => ({ id: "u1", email }) },
  { send: async to => { sent.push(to); } },
  () => new Date("2026-01-01"),
);
```

## Forms of injection

| Form | How | Use when |
|---|---|---|
| **Constructor injection** | Dependencies are constructor params, stored in `final`/`readonly` fields | **Default.** Required deps, immutable, object always valid, dependencies visible |
| **Setter/property injection** | Set after construction | Optional deps with sensible defaults; circular-dependency workarounds (a smell) |
| **Method/parameter injection** | Passed to the method that needs it | Varies per call (a request context, a transaction) |
| **Field injection** (`@Autowired` on a private field) | Container writes into fields via reflection | Avoid in production code: hides deps, needs the container even in tests, can't be `final` |

## The composition root

All wiring happens in **one place near the entry point** (main, app bootstrap, framework config). The rest of the code never calls `new` on infrastructure or asks a container for things.

```ts
// main.ts: composition root ("pure DI", no container)
const db = new Pool({ connectionString: process.env.DB_URL });
const userRepo = new PostgresUserRepository(db);
const mailer = process.env.NODE_ENV === "production" ? new SesMailer() : new ConsoleMailer();
const signup = new SignupService(userRepo, mailer);
const app = createHttpApp({ signup });
app.listen(3000);
```

Pure DI (manual wiring) is perfectly fine for small and medium apps and has zero magic. A container helps when the graph gets large, when you need lifetimes/scopes, or when the framework expects one.

## DI containers

```java
// Spring: constructor injection, no annotations needed on a single constructor
@Service
public class SignupService {
    private final UserRepository repo;
    private final Mailer mailer;
    public SignupService(UserRepository repo, Mailer mailer) { this.repo = repo; this.mailer = mailer; }
}

@Configuration
class MailConfig {
    @Bean @Profile("prod") Mailer sesMailer() { return new SesMailer(); }
    @Bean @Profile("!prod") Mailer consoleMailer() { return new ConsoleMailer(); }
}
```

```ts
// Angular: inject() function (v14+) or constructor injection; providers configure implementations
@Injectable({ providedIn: "root" })
export class SignupService {
  private repo = inject(UserRepository);
  private mailer = inject(Mailer);
}
```

Common containers: Spring (Java/Kotlin), Jakarta CDI, Guice, Dagger/Hilt (compile-time, Android), .NET `Microsoft.Extensions.DependencyInjection`, NestJS, Angular, InversifyJS/tsyringe (TS), Koin (Kotlin), dry-rb (Ruby).

### Lifetimes (scopes)

| Scope | Meaning | Typical use | Watch out |
|---|---|---|---|
| **Singleton** | One instance for the app | Stateless services, clients, pools | Must be thread-safe; no per-request state |
| **Scoped / request** | One per request/unit of work | DB context/transaction, current user | Don't inject into singletons (**captive dependency**) |
| **Transient / prototype** | New instance every injection | Lightweight stateful helpers | Cost if expensive to build |

## Service Locator (anti-pattern)

```java
// ❌ Pulls dependencies from a global registry: hidden dependencies
class SignupService {
    void signup(String email) {
        var repo = ServiceLocator.get(UserRepository.class);   // can't see this from the constructor
        ...
    }
}
```

It inverts creation but hides what a class needs, fails at runtime instead of construction time, and couples everything to the locator. Mark Seemann calls it an anti-pattern; DI makes dependencies explicit.

## DI in functional and dynamic code

- **Pass functions**: a dependency can be a function parameter (`(fetch, now) => ...`), or closed over by a factory (`makeSignup({ repo, mailer })` returns the handler).
- **Partial application / Reader pattern**: provide dependencies first, data later.
- **React**: props and **Context** are DI for components; hooks like `useQueryClient()` read injected services.
- **Ruby/Python**: default arguments (`def initialize(mailer: SmtpMailer.new)`) give injection with sensible defaults (see [SOLID Principles with Ruby](/Software%20Engineering/Principles/SOLID%20Principles%20with%20Ruby)).
- **Kotlin**: **context parameters** declare dependencies that are implicitly available from the calling scope (`context(logger: Logger) fun process() { ... }`).

<Info>
**🆕 2025–2026**

- **Kotlin context parameters**: introduced as a preview in **Kotlin 2.2.0 (June 2025)** and **stable in Kotlin 2.4.0 (2026-07-14)** (except explicit context arguments and callable references). They replace the older experimental context receivers and give a language-level way to pass ambient dependencies (logger, transaction, clock) without threading them through every call.
- **Java 25 (2025-09-16)** finalised **Scoped Values** (JEP 506): immutable, bounded per-call context (e.g. request or tenant info) that's safer than `ThreadLocal` for passing implicit context, especially with virtual threads. Not a DI container replacement, but a cleaner option for "ambient context".
- **JEP 500 (Java 26, 2026-03-17)** warns when libraries mutate `final` fields via reflection. Field-injection frameworks that write into `final` fields will need to adapt; constructor injection is unaffected.
</Info>

## Pitfalls

<Warning>
**Common mistakes**

- **Constructor over-injection** (8+ dependencies): the class does too much (SRP). Split it or introduce a facade over related collaborators.
- **Field injection** everywhere: hidden dependencies, tests require the container.
- **Captive dependencies**: a singleton holding a request-scoped object, leaking one user's data into another's request.
- **Circular dependencies**: A needs B needs A. Usually a missing third abstraction or event.
- **Injecting the container** itself (service locator in disguise).
- **Interface-per-class** for everything: DI works with concrete classes too; add interfaces where you need substitution or a boundary.
- Heavy runtime reflection/classpath scanning hurting startup (serverless, CLI). Consider compile-time DI (Dagger, Micronaut, Spring AOT/GraalVM) where startup matters.
</Warning>

## Senior perspective

- DI is a means; the goal is **explicit dependencies and a domain that doesn't depend on infrastructure**. Check imports, not annotations.
- Inject **time, randomness, IDs and configuration**: they're dependencies too, and injecting them makes tests deterministic.
- Keep the container at the edges: domain classes shouldn't carry framework annotations if you can avoid it (use configuration classes/modules to register them).
- When debugging container magic (proxies, lifecycles, conditional beans), know how to print the dependency graph and the active profiles.

## Interview questions

<Accordion title="What's the difference between DIP, DI and IoC?">
DIP is a principle about dependency direction (both sides depend on abstractions owned by the high-level policy). IoC is the broader idea that control flow is handed to a framework that calls your code. DI is a specific technique where an object receives its dependencies from outside; containers automate it.
</Accordion>

<Accordion title="Why prefer constructor injection over field injection?">
Dependencies are explicit and required, fields can be final/immutable, the object is fully valid after construction, and tests can create it with `new` and fakes without a container. Too many constructor parameters also reveal SRP problems early.
</Accordion>

<Accordion title="What is a composition root?">
The single place, close to the application entry point, where the object graph is assembled and implementations are chosen. Everything else receives dependencies and never looks them up.
</Accordion>

<Accordion title="Why is Service Locator considered an anti-pattern?">
It hides a class's dependencies inside method bodies, so you can't see or satisfy them from the API; missing registrations fail at runtime; and every class depends on the locator. DI makes dependencies explicit in constructors.
</Accordion>

<Accordion title="What are DI scopes, and what's a captive dependency?">
Singleton (one per app), scoped (one per request/unit of work), transient (new every time). A captive dependency is a longer-lived object holding a shorter-lived one (singleton holding a request-scoped DB context), causing stale state, thread-safety bugs or data leaks between requests.
</Accordion>

<Accordion title="Do you need a DI framework to do DI?">
No. Passing dependencies through constructors and wiring them in `main` ("pure DI") is DI. Containers help with large graphs, scopes and framework integration.
</Accordion>

## Related
- [Principles Index](/Software%20Engineering/Principles/Principles%20Index)
- [SOLID](/Software%20Engineering/Principles/SOLID) · [SOLID Principles with Ruby](/Software%20Engineering/Principles/SOLID%20Principles%20with%20Ruby) · [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface) · [Creational Patterns](/Software%20Engineering/Principles/Creational%20Patterns)
- [Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles) · [Test-Driven Development (TDD)](/Software%20Engineering/Principles/Test-Driven%20Development%20(TDD)) · [Coupling, Cohesion & Modularity](/Software%20Engineering/Principles/Coupling%2C%20Cohesion%20%26%20Modularity)

## Sources
- [Martin Fowler: Inversion of Control Containers and the Dependency Injection pattern (2004)](https://martinfowler.com/articles/injection.html)
- Mark Seemann & Steven van Deursen, *Dependency Injection Principles, Practices, and Patterns* (Manning, 2019) · [Seemann: Service Locator is an Anti-Pattern](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/)
- [Spring Framework: Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)
- [Angular: Dependency injection](https://angular.dev/guide/di)
- [Microsoft: .NET dependency injection, service lifetimes](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection#service-lifetimes)
- [Kotlin 2.2.0 released (JetBrains blog, June 2025)](https://blog.jetbrains.com/kotlin/2025/06/kotlin-2-2-0-released/) · [What's new in Kotlin 2.4.0](https://kotlinlang.org/docs/whatsnew24.html) · [Kotlin docs: Context parameters](https://kotlinlang.org/docs/context-parameters.html)
- [JEP 506: Scoped Values](https://openjdk.org/jeps/506) · [JEP 500: Prepare to Make Final Mean Final](https://openjdk.org/jeps/500)
