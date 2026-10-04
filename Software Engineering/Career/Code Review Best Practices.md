---
title: "Code Review Best Practices"
tags: [career, career/code-review]
aliases: [Code Review, Code Reviews, PR Review, Reviewing Pull Requests, Reviewing AI Code]
updated: 2026-09-27
---
# Code Review Best Practices

<Note>
**TL;DR**

Code review exists to keep the **codebase healthy over time** and to share knowledge, not to prove who's smartest. Google's standard: approve a change once it **definitely improves the overall code health**, even if it isn't perfect. Authors send small, well-described changes; reviewers respond quickly, focus on design, correctness and tests before style, and write kind, specific comments with a clear severity. With AI generating more code in 2025–2026, review is now the main quality gate, so review AI-written code **more** carefully, not less.
</Note>

<Info>
**🆕 2025–2026**

- **2025-04-04**: GitHub Copilot code review became generally available for paid Copilot users; AI reviewers (Copilot, CodeRabbit, and others) are now common as a **first pass** ([GitHub changelog](https://github.blog/changelog/2025-04-04-copilot-code-review-now-generally-available/)).
- **2025**: DORA's *State of AI-assisted Software Development* found AI adoption linked to higher throughput **and** higher delivery instability; small batches and strong version control practices are among the capabilities that decide whether AI helps ([DORA 2025](https://dora.dev/dora-report-2025/)). Bigger AI-generated PRs make review the bottleneck.
- **2025**: Stack Overflow survey: 66% of developers cite AI answers that are "almost right, but not quite" as their top frustration ([survey](https://survey.stackoverflow.co/2025/ai)), which is exactly what reviewers must catch.
</Info>

## Why we review
1. Catch defects and design problems early (cheaper than in production).
2. Keep the code consistent and understandable for the next person.
3. Spread knowledge: at least two people understand each change.
4. Mentor and learn in both directions.
5. Security and compliance (two-person rule on production changes).

## What to look for (in priority order)

| Priority | Area | Questions |
|---|---|---|
| 1 | **Design** | Does this belong here? Right abstraction? Does it fit the architecture? Is it needed at all? |
| 2 | **Functionality** | Does it do what the author intended, and is that what users need? Edge cases, concurrency, errors? |
| 3 | **Complexity** | Can it be simpler? Over-engineering for hypothetical futures? |
| 4 | **Tests** | Tests for the behaviour and edge cases? Would they fail if the code broke? |
| 5 | **Security & data** | Input validation, authz checks, secrets, PII in logs, SQL built from strings ([Backend Security Essentials](/Software%20Engineering/Security/Backend%20Security%20Essentials)) |
| 6 | **Operability** | Logs, metrics, feature flags, migrations safe to roll back, performance at scale |
| 7 | **Naming & comments** | Clear names; comments explain *why*, not *what* |
| 8 | **Style** | Automate it with formatters and linters; don't spend human attention here |
| 9 | **Docs** | README, API docs, changelog updated |

Google's guide also says: review **every line** you're asked to review, look at the change in the context of the whole file/system, and point out good things too.

## For authors: making changes easy to review
- **Small PRs**: aim for under ~200–400 changed lines; split refactors from behaviour changes; stack PRs if needed.
- A description that answers: *what*, *why*, *how to test*, *risks*, screenshots for UI, links to the issue/design doc.
- Self-review the diff first; run tests and linters locally.
- Mark AI-generated parts if your team asks for it, and **understand every line** before requesting review; you are the author.
- Respond to every comment (fix, explain, or agree to a follow-up ticket). Don't take comments personally.

```markdown
## What
Add idempotency keys to POST /payments.
## Why
Retries from the mobile client created duplicate charges (INC-231).
## How
- New `idempotency_keys` table (migration is additive, safe to roll back)
- Middleware stores key + response hash for 24h
## Testing
Unit tests for the middleware, integration test replaying the same request 3x.
## Risks
Extra DB write per payment (~2 ms p99 in staging).
```

## For reviewers: giving good feedback
- **Speed**: respond within one business day (Google recommends at most one business day for the first response); fast reviews keep batch sizes small.
- Label severity so authors know what blocks:

| Prefix | Meaning |
|---|---|
| (no prefix) or **blocking** | Must change before merge |
| **nit:** | Minor, optional polish |
| **suggestion / consider:** | Author decides |
| **question:** | I want to understand, not necessarily change |
| **FYI / praise:** | Learning or appreciation |

(Conventional Comments formalises this: `issue:`, `suggestion:`, `nitpick:`, `question:`, `praise:`, with `(blocking)`/`(non-blocking)`.)

- Comment on the **code, not the person**: "This function could return early to reduce nesting" rather than "You wrote this badly".
- Explain the **why** and offer an alternative. Ask questions when unsure.
- Don't block on personal preference; if it's not in the style guide and the code is fine, let it go.
- Move long debates to a call, then record the outcome in the PR.
- Approve with comments when the remaining items are nits.

## Reviewing AI-generated code
AI output is plausible by construction, so errors hide in the details.

<Info>
**Checklist for AI-written changes**

- **Does it solve the actual problem?** AI often solves a nearby problem or the general case with the wrong constraints.
- **Hallucinated APIs**: methods, options or packages that don't exist, or exist in a different version. Check imports and new dependencies (also for typosquats: "slopsquatting").
- **Edge cases and error paths**: empty inputs, nulls, timeouts, partial failures; AI tends to write the happy path.
- **Security**: string-built SQL, missing authz checks, disabled TLS verification, secrets inline, overly broad permissions.
- **Tests that test nothing**: tests asserting the implementation's current output, mocking everything, or modified to pass.
- **Duplication**: re-implementing helpers that already exist in the codebase.
- **Over-engineering**: unnecessary abstractions, config options, comments that restate the code.
- **Consistency**: project conventions, logging, error handling patterns.
- **Size**: ask for a split if the PR is too big to review properly.
</Info>

Use AI reviewers as a first pass for obvious issues, but a human stays accountable for design and correctness.

## Pitfalls
<Warning>
**Common mistakes**

- Rubber-stamping ("LGTM") big PRs you didn't read.
- Nit-picking style that a formatter should handle.
- Reviews that sit for days, pushing authors to batch more into each PR.
- Rewriting the author's solution to your preferred one when theirs is fine.
- Treating review as the only quality gate (tests, CI, monitoring matter more).
</Warning>

## Senior perspective
- Set team norms: review SLAs, PR size guidelines, severity labels, what's automated, who must review what (CODEOWNERS).
- Review for **the system**: will this scale, is it observable, what happens on failure, how does it migrate?
- Use reviews to **teach**: link docs, explain principles, pair on complex ones. Also let juniors review your code.
- Watch team metrics: time to first review, PR size, rework rate. Slow reviews are a delivery problem, not a personal one.

## Interview questions
<Accordion title="What do you look for in a code review?">
In order: design and whether the change is needed, correctness and edge cases, simplicity, tests, security and operability, then naming and docs. Style is left to automated tools.
</Accordion>

<Accordion title="How do you give critical feedback in a review without hurting the relationship?">
Comment on the code, explain why, suggest an alternative, mark severity, ask questions when unsure, and acknowledge good parts. Move heated threads to a conversation.
</Accordion>

<Accordion title="When should a reviewer approve a PR that isn't perfect?">
When it clearly improves the codebase's overall health and has no blocking issues. Remaining nits can be optional or follow-ups; perfection blocks progress.
</Accordion>

<Accordion title="How do you review AI-generated code?">
Verify it solves the right problem, check every API and dependency exists and is correct, look hard at edge cases, error paths, security and test quality, and ensure the author understands and owns it. Ask for smaller PRs if needed.
</Accordion>

<Accordion title="How do you keep reviews fast in a busy team?">
Small PRs, review SLAs, automated formatting/linting/tests, clear descriptions, CODEOWNERS to route reviews, and treating reviews as higher priority than starting new work.
</Accordion>

## Related
- [Career Index](/Software%20Engineering/Career/Career%20Index)
- [Senior Engineer in the AI Era](/Software%20Engineering/Career/Senior%20Engineer%20in%20the%20AI%20Era) · [Communication & Influence for Engineers](/Software%20Engineering/Career/Communication%20%26%20Influence%20for%20Engineers) · [Mentoring & Leading Without Authority](/Software%20Engineering/Career/Mentoring%20%26%20Leading%20Without%20Authority)
- [Working With Legacy Code](/Software%20Engineering/Career/Working%20With%20Legacy%20Code) · [Secure SDLC & Threat Modeling](/Software%20Engineering/Security/Secure%20SDLC%20%26%20Threat%20Modeling) · [Frontend Testing](/Software%20Engineering/Frontend/13%20Testing/Frontend%20Testing)

## Sources
- [Google Engineering Practices: Code Review](https://google.github.io/eng-practices/review/) (reviewer and author guides)
- [Conventional Comments](https://conventionalcomments.org/)
- [GitHub changelog: Copilot code review GA (2025-04-04)](https://github.blog/changelog/2025-04-04-copilot-code-review-now-generally-available/)
- [DORA 2025: State of AI-assisted Software Development](https://dora.dev/dora-report-2025/)
- [Stack Overflow Developer Survey 2025: AI](https://survey.stackoverflow.co/2025/ai)
- Sadowski et al., [Modern Code Review: A Case Study at Google (ICSE 2018)](https://research.google/pubs/modern-code-review-a-case-study-at-google/)
