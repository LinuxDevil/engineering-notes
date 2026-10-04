---
title: "DRY, KISS & YAGNI"
tags: [principles, principles/clean-code]
aliases: [DRY, KISS, YAGNI, Dry & Kiss, Don't Repeat Yourself, Keep It Simple Stupid, You Aren't Gonna Need It, Rule of Three, WET, AHA]
updated: 2026-09-27
---
# DRY, KISS & YAGNI

<Note>
**TL;DR**

**DRY** (Don't Repeat Yourself): every piece of *knowledge* should have one authoritative home. **KISS** (Keep It Simple, Stupid): prefer the simplest design that works. **YAGNI** (You Aren't Gonna Need It): don't build for requirements you don't have yet. They pull against each other: over-DRY code creates the wrong abstraction, and "flexible" code violates YAGNI. Seniors apply them with the **rule of three** and a bias toward code that is easy to change later.
</Note>

| Principle | Origin | Core question |
|---|---|---|
| **DRY** | Hunt & Thomas, *The Pragmatic Programmer* (1999) | "If this rule changes, how many places must I edit?" |
| **KISS** | Attributed to Kelly Johnson, Lockheed Skunk Works (1960s) | "Is there a simpler way that still meets the need?" |
| **YAGNI** | Extreme Programming (Ron Jeffries, Kent Beck, late 1990s) | "Do we need this *now*, for a real requirement?" |

## DRY: Don't Repeat Yourself

> "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system." (Hunt & Thomas)

DRY is about **knowledge**, not identical-looking text. Two functions with the same lines that represent **different** business rules are not duplication; merging them couples things that will change for different reasons. Conversely, the same rule expressed in Java, SQL and a TypeScript validator *is* duplication even though the code looks different.

To follow DRY, divide the system into pieces: split logic into small reusable units and call them where needed, instead of copy-pasting blocks inside long methods. Helper/utility modules in libraries are a familiar example: each piece of logic lives in exactly one place.

```ts
// ❌ The "is this user an adult?" rule lives in three places
if (user.age >= 18) showCheckout();          // UI
if (req.body.age < 18) throw new Error();     // API
// SQL: WHERE age >= 18                        // report

// ✅ One authoritative home (and generated or shared wherever possible)
export const ADULT_AGE = 18;
export const isAdult = (age: number) => age >= ADULT_AGE;
```

Where DRY matters beyond code:
- **Schema as the source of truth**: generate types/validators/clients from one OpenAPI, Protobuf, GraphQL or Zod schema instead of hand-writing each.
- **Configuration**: one place for a timeout or feature flag.
- **Docs**: generate API docs from code/comments rather than maintaining a copy.

### When not to DRY: the wrong abstraction

> "Duplication is far cheaper than the wrong abstraction." (Sandi Metz, 2016)

The failure mode: two similar pieces are merged into a shared function; later each caller needs a variation, so the function grows boolean parameters and `if` branches until nobody understands it.

```ts
// Smell: a "shared" function steered by flags
function formatPrice(p: number, forInvoice = false, hideCurrency = false, legacyRounding = false) { /* ... */ }
```

Heuristics:
- **Rule of three**: tolerate duplication twice; extract on the third occurrence, when the real shared shape is visible.
- **AHA** (Avoid Hasty Abstractions, Kent C. Dodds) and **WET** ("write everything twice") are the same idea.
- Only merge code that changes **for the same reason** (see SRP in [SOLID](/Software%20Engineering/Principles/SOLID)).
- If an abstraction has become wrong, **inline it back** into callers, then re-extract what is truly shared.

## KISS: Keep It Simple, Stupid

The simplest solution that meets the requirements is usually the most reliable and the easiest to change. Code is read far more often than it is written, so optimise for the reader.

Practical rules:
- Keep functions **small and focused** on one task. A common guideline is to be suspicious of methods longer than roughly 40–50 lines or with deep nesting; it's a prompt to look, not a hard limit.
- If a method has many conditions, break them into well-named smaller functions. It's easier to read, test and debug.
- Think of several solutions, then pick the simplest one that works.
- Prefer boring, well-known technology and language features over clever ones.

**Example: map a day number to its name** (Java)

```java
// Version 1: explicit switch
public String weekday1(int day) {
    switch (day) {
        case 1: return "Monday";
        case 2: return "Tuesday";
        case 3: return "Wednesday";
        case 4: return "Thursday";
        case 5: return "Friday";
        case 6: return "Saturday";
        case 7: return "Sunday";
        default: throw new IllegalArgumentException("day must be in range 1 to 7");
    }
}

// Version 2: guard + lookup table (less code, one place for the data)
public String weekday2(int day) {
    if (day < 1 || day > 7) throw new IllegalArgumentException("day must be in range 1 to 7");
    String[] days = {"Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"};
    return days[day - 1];
}

// Version 3 (simplest of all): don't write it, use the standard library
public String weekday3(int day) {
    return java.time.DayOfWeek.of(day)            // throws DateTimeException if not 1..7
            .getDisplayName(java.time.format.TextStyle.FULL, java.util.Locale.ENGLISH);
}
```

Version 3 is the KISS winner: no custom logic, localisation for free. (The original note used C#'s `InvalidOperationException` and `string[]`; in Java these are `IllegalArgumentException` and `String[]`.)

<Note>
**Simple is not the same as easy**

Rich Hickey's talk *Simple Made Easy* (2011): **simple** means "not intertwined" (one concept, one role); **easy** means "familiar, near at hand". A framework may be easy to start with but complex (many interleaved concerns). KISS aims at *simple*.
</Note>

## YAGNI: You Aren't Gonna Need It

> "Always implement things when you actually need them, never when you just foresee that you need them." (Ron Jeffries)

Features and extension points built "for later" have real costs even if they never get used (Martin Fowler lists them):
- **Cost of building**: time not spent on what's needed now.
- **Cost of delay**: the needed feature ships later.
- **Cost of carry**: every future change must work around the extra code.
- **Cost of repair**: when the real need arrives it's often different from the guess, so the speculative code gets rewritten anyway.

```ts
// ❌ YAGNI violation: a plugin system, 3 config layers and an abstract factory
//    for a single CSV export that one customer asked for.
// ✅ Write the CSV export. Keep it clean and tested so it's easy to generalise
//    when the second format is actually requested.
```

YAGNI **does not** mean skipping things that make code easy to change: tests, refactoring, clear names and sensible module boundaries are what make deferring decisions safe. It applies to *features and speculative generality*, not to quality.

## How they interact

{/* diagram source: /diagrams/diagram-367.mmd */}
![diagram](/diagrams/diagram-367.svg)

| Situation | Guidance |
|---|---|
| Same business rule in 3 places | DRY it now |
| Two similar-looking functions for different domains | Leave them (they'll diverge) |
| "We might support other databases one day" | YAGNI; but keep SQL behind a repository so it's not scattered |
| A clever one-liner vs a clear 5-line loop | KISS: the loop, unless the one-liner is idiomatic |

<Info>
**🆕 2025–2026: AI assistants and duplication**

GitClear's analysis of 211 million changed lines (report published **February 2025**) found that in 2024 duplicated blocks of 5+ lines grew about 8×, "moved" (refactored) lines fell by about 40%, and copy/pasted lines exceeded moved lines for the first time. Their explanation: assistants make inserting new code cheaper than finding and reusing existing code. The **2025 DORA report** (September 2025, ~5,000 respondents) similarly found AI raises throughput but also instability, acting as an "amplifier" of existing practices. Practical takeaway: in AI-assisted codebases, review specifically for re-implemented helpers, near-duplicate functions and speculative extras the prompt never asked for. Both studies are correlational, so treat them as a warning sign, not proof.
</Info>

## Senior perspective

- In review ask: "Where else does this rule live?" (DRY), "What's the simplest version?" (KISS), "Which requirement needs this?" (YAGNI).
- Duplication **across service boundaries** is often correct: sharing a "common" library between microservices couples their release cycles. Prefer duplicating small code and sharing **contracts** (schemas) instead.
- Simplicity is a system property: fewer moving parts (services, queues, caches) usually beats cleverness inside any one part. See [Software Architecture Styles](/Software%20Engineering/Principles/Software%20Architecture%20Styles).
- Keep a log of deliberate shortcuts so YAGNI decisions don't turn into forgotten debt. See [Architecture Decision Records (ADRs) & Technical Debt](/Software%20Engineering/Principles/Architecture%20Decision%20Records%20(ADRs)%20%26%20Technical%20Debt).

## Interview questions

<Accordion title="What does DRY actually mean? Is all duplicated code a DRY violation?">
DRY is about knowledge: each rule or fact should have one authoritative representation. Similar-looking code that encodes different rules (and will change for different reasons) is not a violation, and merging it creates coupling. Conversely, one rule written in three languages is a violation.
</Accordion>

<Accordion title="What is &quot;the wrong abstraction&quot; and how do you fix it?">
A shared function created too early that accumulates flags and branches as callers diverge. Fix by inlining it back into each caller, deleting unused paths, and re-extracting only what is genuinely common. The rule of three helps avoid it.
</Accordion>

<Accordion title="Explain YAGNI. Doesn't it lead to rewrites?">
Build only what current requirements need. Speculative features have build, delay, carry and repair costs, and guesses are often wrong. YAGNI relies on tests and refactoring to keep code easy to extend later, so extending when the need is real is cheap.
</Accordion>

<Accordion title="How do KISS and YAGNI differ?">
KISS is about how you implement what you need (choose the simplest design). YAGNI is about what you build at all (don't build unrequested capabilities). You can violate KISS while obeying YAGNI (a convoluted solution to a real need) and vice versa.
</Accordion>

<Accordion title="Should microservices share a common library to stay DRY?">
Usually only for stable, generic concerns (logging, auth clients) and with careful versioning. Sharing domain code couples deployments and teams. Prefer sharing schemas/contracts and accept some duplication.
</Accordion>

## Related
- [Principles Index](/Software%20Engineering/Principles/Principles%20Index)
- [Clean Code & Refactoring](/Software%20Engineering/Principles/Clean%20Code%20%26%20Refactoring) · [SOLID](/Software%20Engineering/Principles/SOLID) · [Coupling, Cohesion & Modularity](/Software%20Engineering/Principles/Coupling%2C%20Cohesion%20%26%20Modularity) · [Architecture Decision Records (ADRs) & Technical Debt](/Software%20Engineering/Principles/Architecture%20Decision%20Records%20(ADRs)%20%26%20Technical%20Debt)

## Sources
- Andrew Hunt & David Thomas, *The Pragmatic Programmer*, 20th Anniversary ed. (2019): DRY
- [Martin Fowler: Yagni](https://martinfowler.com/bliki/Yagni.html)
- [Ron Jeffries: You're NOT gonna need it!](https://ronjeffries.com/xprog/articles/practices/pracnotneed/)
- [Sandi Metz: The Wrong Abstraction (2016)](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction)
- [Kent C. Dodds: AHA Programming](https://kentcdodds.com/blog/aha-programming)
- [Rich Hickey: Simple Made Easy (2011)](https://www.infoq.com/presentations/Simple-Made-Easy/)
- [GitClear: AI Copilot Code Quality 2025](https://www.gitclear.com/ai_assistant_code_quality_2025_research) · [DevClass summary (2025-02-20)](https://www.devclass.com/ai-ml/2025/02/20/ai-is-eroding-code-quality-states-new-in-depth-report/1626250)
- [DORA: State of AI-assisted Software Development 2025](https://dora.dev/dora-report-2025/)
- [Java SE: java.time.DayOfWeek](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/time/DayOfWeek.html)
