---
title: "Behavior-Driven Development (BDD)"
tags: [principles, principles/testing]
aliases: [BDD, Behaviour-Driven Development, Behavior Driven Development, Gherkin, Given When Then, Specification by Example, ATDD, Example Mapping]
updated: 2026-09-27
---
# Behavior-Driven Development (BDD)

<Note>
**TL;DR**

**BDD** extends TDD outward to the whole team: business, developers and testers agree on **concrete examples** of behaviour before building, write them in a shared language (**Given / When / Then**), and automate them as acceptance tests. Its value is the **conversation** (discovery), not the tooling; Gherkin/Cucumber are optional. Use it where requirements are ambiguous and stakeholders care about rules; skip the tooling where only developers read the tests.
</Note>

## The idea

Dan North introduced BDD around 2006 because teams struggled with TDD's vocabulary ("what should I test? what's a unit?"). Replacing "test" with "**behaviour**" and "should" made tests read as specifications, and led to the **Given–When–Then** template for acceptance criteria, co-developed with Chris Matts.

BDD has three practices (as described by the Cucumber team):

{/* diagram source: /diagrams/diagram-359.mmd */}
![diagram](/diagrams/diagram-359.svg)

1. **Discovery**: a short workshop ("Three Amigos": business/product, development, testing) explores a user story through examples, rules and open questions.
2. **Formulation**: the useful examples are written as structured scenarios in business language.
3. **Automation**: scenarios are wired to code and run in CI, becoming **living documentation** that can't drift from the system.

Related names: **ATDD** (Acceptance Test-Driven Development) and **Specification by Example** (Gojko Adzic) describe essentially the same practice.

## Example mapping (discovery)

Matt Wynne's example mapping uses four colours of cards for one story:

| Card | Example for "Free shipping" story |
|---|---|
| 🟨 **Story** | As a shopper I get free shipping on large orders |
| 🟦 **Rules** | Orders ≥ €50 ship free · Only within the EU · Gift cards don't count towards the total |
| 🟩 **Examples** (per rule) | €49.99 → €4.95 shipping · €50.00 → free · €60 incl. €20 gift card → €4.95 |
| 🟥 **Questions** | Is the threshold before or after discount codes? |

Lots of red cards = the story isn't ready. Lots of blue = the story is too big; split it.

## Formulation: Gherkin

```gherkin
Feature: Free shipping on large orders
  Orders of €50 or more (excluding gift cards) ship free within the EU.

  Rule: The threshold is €50 of eligible items

    Scenario: Order just below the threshold pays shipping
      Given my cart contains items worth €49.99
      And I ship to "Netherlands"
      When I check out
      Then the shipping cost is €4.95

    Scenario Outline: Gift cards don't count toward free shipping
      Given my cart contains items worth €<items> and gift cards worth €<giftcards>
      When I check out
      Then the shipping cost is €<shipping>

      Examples:
        | items | giftcards | shipping |
        | 50.00 | 0         | 0.00     |
        | 40.00 | 20.00     | 4.95     |
```

Writing good scenarios:
- **Declarative, not imperative**: "When I check out", not "When I click the button with id #checkout". UI details belong in step code.
- One behaviour per scenario; business language from the domain's [ubiquitous language](/Software%20Engineering/Principles/Domain-Driven%20Design%20(DDD)).
- Given = context (past), When = one event/action, Then = observable outcome.
- Keep the number of scenarios small; edge cases can live in unit tests.

## Automation

```ts
// Cucumber.js step definitions (TypeScript)
import { Given, When, Then } from "@cucumber/cucumber";
import assert from "node:assert/strict";

Given("my cart contains items worth €{float}", function (amount: number) {
  this.cart = Cart.withItemsWorth(amount);
});
Given("I ship to {string}", function (country: string) {
  this.address = { country };
});
When("I check out", async function () {
  this.quote = await checkoutService.quote(this.cart, this.address);   // call the domain/API, not the UI
});
Then("the shipping cost is €{float}", function (expected: number) {
  assert.equal(this.quote.shipping, expected);
});
```

Tools: Cucumber (Java, JS, Ruby), SpecFlow's successor **Reqnroll** (.NET), Behave (Python), Karate, Serenity BDD; Playwright/Cypress for UI-level scenarios. Many teams get most of the benefit with plain test frameworks using BDD-style names:

```ts
describe("Shipping quote", () => {
  describe("given an EU order of exactly €50", () => {
    it("then shipping is free", () => { /* ... */ });
  });
});
```

## BDD vs TDD

| | TDD | BDD |
|---|---|---|
| Audience | Developers | Whole team, including business |
| Granularity | Units of code behaviour | Features/business rules (outside-in) |
| Language | Code | Domain language, Given/When/Then |
| Primary goal | Design feedback, regression safety | Shared understanding, the right thing built |
| Typical tools | JUnit, Vitest, RSpec | Cucumber, Reqnroll, Behave (optional) |

They combine as **double-loop TDD**: an outer failing acceptance scenario, satisfied by many inner red–green–refactor unit cycles (Freeman & Pryce).

{/* diagram source: /diagrams/diagram-360.mmd */}
![diagram](/diagrams/diagram-360.svg)

<Info>
**🆕 2025–2026**

- **Spec-driven development** with AI agents revives BDD ideas: AWS **Kiro** (announced 2025-07-14) writes requirements as user stories with acceptance criteria in the EARS notation before design and tasks, and GitHub **Spec Kit** (2025-09-02) structures work as spec → plan → tasks. Well-written Given/When/Then examples make good agent acceptance criteria, but Thoughtworks (Radar Vol. 33, Nov 2025, "Assess") cautions the workflows are still elaborate and opinionated.
- In .NET, **Reqnroll** ("a reboot of the SpecFlow project", SpecFlow-compatible) is the actively maintained option; the specflow.org site now redirects to Tricentis' ShiftSync community. SpecFlow's exact end-of-life date is (unverified).
</Info>

## Pitfalls

<Warning>
**Common BDD failures**

- **Tool-first BDD**: developers write Gherkin alone after coding. You pay Cucumber's maintenance cost without the conversation's benefit.
- **Imperative UI scripts** in Gherkin ("click", "fill in") that break on every UI tweak.
- Huge step-definition libraries with regex spaghetti and shared mutable "world" state.
- Testing everything through the UI: slow, flaky suites. Automate scenarios against the API or domain layer when possible.
- Scenarios nobody from the business reads: then plain test code is cheaper.
</Warning>

## Senior perspective

- Push for **discovery** even if you don't use Gherkin: 25-minute example mapping sessions catch misunderstandings before code exists, which is the cheapest time to fix them.
- Choose the automation level per scenario (domain, API, UI). Keep UI scenarios to a handful of critical journeys.
- Treat feature files as product documentation: organise by capability, prune regularly, review wording with product.
- Measure value: if scenarios are never read outside engineering, simplify to BDD-style unit/integration tests.

## Interview questions

<Accordion title="What is BDD and how does it differ from TDD?">
BDD is a collaborative practice where the team discovers and agrees on behaviour through concrete examples, formulates them as Given/When/Then scenarios and automates them as acceptance tests. TDD is a developer practice for designing code in red–green–refactor cycles. BDD works outside-in at the feature level and often drives TDD inside.
</Accordion>

<Accordion title="What makes a good Gherkin scenario?">
Declarative business language, one behaviour, clear context (Given), a single action (When) and observable outcomes (Then), no UI implementation details, independent of other scenarios.
</Accordion>

<Accordion title="Who are the &quot;Three Amigos&quot;?">
Business/product, development and testing perspectives meeting before implementation to explore a story with examples, uncover rules and questions, and agree on acceptance criteria.
</Accordion>

<Accordion title="When would you not use Cucumber?">
When no non-developer reads or contributes to the scenarios. The translation layer (steps, regex, world state) then only adds cost; use BDD-style names in the normal test framework instead.
</Accordion>

## Related
- [Principles Index](/Software%20Engineering/Principles/Principles%20Index)
- [Test-Driven Development (TDD)](/Software%20Engineering/Principles/Test-Driven%20Development%20(TDD)) · [Domain-Driven Design (DDD)](/Software%20Engineering/Principles/Domain-Driven%20Design%20(DDD)) · [Clean Code & Refactoring](/Software%20Engineering/Principles/Clean%20Code%20%26%20Refactoring)
- [Frontend Testing](/Software%20Engineering/Frontend/13%20Testing/Frontend%20Testing)

## Sources
- [Dan North: Introducing BDD (2006)](https://dannorth.net/introducing-bdd/)
- [Cucumber: Behaviour-Driven Development](https://cucumber.io/docs/bdd/) · [Gherkin reference](https://cucumber.io/docs/gherkin/reference/)
- [Matt Wynne: Introducing Example Mapping (2015)](https://cucumber.io/blog/bdd/example-mapping-introduction/)
- Gojko Adzic, *Specification by Example* (Manning, 2011)
- [Martin Fowler: GivenWhenThen](https://martinfowler.com/bliki/GivenWhenThen.html)
- [Reqnroll](https://reqnroll.net/)
- [AWS: Introducing Kiro (2025)](https://kiro.dev/blog/introducing-kiro/) · [GitHub Blog: Spec Kit (2025)](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/) · [Thoughtworks Radar: Spec-driven development](https://www.thoughtworks.com/en-us/radar/techniques/spec-driven-development)
