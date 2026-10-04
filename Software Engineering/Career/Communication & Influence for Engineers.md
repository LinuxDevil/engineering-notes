---
title: "Communication & Influence for Engineers"
tags: [career, career/communication]
aliases: [Communication for Engineers, Influence, Technical Communication, Stakeholder Management, Writing for Engineers]
updated: 2026-09-27
---
# Communication & Influence for Engineers

<Note>
**TL;DR**

Past mid-level, your impact is limited less by your code than by how well you **explain, persuade and align** people. Communicate **bottom line first**, adapt to the audience (engineers want the how, managers want the impact and risk, executives want the decision), prefer **writing** for anything that needs thought or a record, and build **trust** before you need it. Influence without authority comes from credibility, understanding other people's goals, data, and making it easy to say yes.
</Note>

## The idea
- Communication is part of the job, not overhead: a correct design no one understands doesn't ship.
- Influence = **credibility** (track record, expertise) × **relationships** (trust, understanding their goals) × **clarity** (a well-framed proposal).
- Most conflict between engineers and other functions comes from different goals and missing context, not bad intentions.

## Bottom line up front (BLUF)
Start with the conclusion or the ask, then the supporting detail.

<Info>
**Status update: before and after**

**Before:** "So we started looking at the payment retries on Monday and found the logs were incomplete, then Sam checked the mobile client and…"

**After:** "**We'll miss Friday's date by about one week.** Cause: duplicate charges come from mobile retries, which needs a server-side fix (idempotency keys). Options: (A) ship Friday without the retry fix, keeping the manual refund process, or (B) ship next Friday with the fix. I recommend B. I need a decision by Wednesday."
</Info>

## Know your audience
| Audience | Cares about | Give them |
|---|---|---|
| Engineers on your team | How it works, edge cases, effort | Design details, code, diagrams |
| Other teams | Impact on them, dependencies, timelines | Interfaces, what you need from them and by when |
| Product managers | User value, scope, dates, trade-offs | Options with cost/benefit, risks in product terms |
| Engineering managers | Delivery, risk, people, quality | Status vs plan, risks, help needed |
| Executives | Business outcome, cost, risk, the decision | One page: problem, recommendation, cost, risk |
| Non-technical stakeholders | What it means for them | Analogies, no jargon, concrete examples |

Translate technical problems into business terms: not "the DB needs sharding" but "without this, checkout slows down during sales from Q2 and we lose about X% of orders".

## Writing
- **Write to think.** A written proposal exposes gaps a meeting hides.
- Structure: summary → context → proposal → alternatives → ask ([Technical Design Docs & RFCs](/Software%20Engineering/Career/Technical%20Design%20Docs%20%26%20RFCs)).
- Short sentences, headings, bullets, tables. Put the ask and deadline in bold.
- Async by default for status and proposals; meetings for decisions that need discussion or conflict resolution.
- Messages: one topic per message, with enough context to act on without a follow-up ("no hello" style: say the question in the first message).
- Incident and status writing: facts, impact, next update time.

## Meetings
- Every meeting has a purpose, an agenda and an owner; decisions and action items are written down afterwards.
- As a senior: facilitate, draw out quieter people, summarise disagreements fairly, and close with "so we decided X; owner Y; by Z".

## Influence without authority
1. **Understand their goals** first (1:1 conversations). What does the other team or PM need to succeed?
2. **Frame your proposal in their terms**: how it helps their goals.
3. **Bring evidence**: data, a prototype, an incident history, a cost estimate.
4. **Socialise early and privately** before big meetings; no one likes surprises in public.
5. **Make it easy to say yes**: small first step, reversible, you do the work.
6. **Share credit** generously; let others co-own the idea.
7. **Disagree and commit** when the decision goes the other way, and support it fully.

See [Mentoring & Leading Without Authority](/Software%20Engineering/Career/Mentoring%20%26%20Leading%20Without%20Authority) for leading projects and people.

## Handling disagreement
- Separate the problem from the person; restate their position until they agree you understood it.
- Agree on goals and decision criteria first, then compare options against them.
- Escalate **together** when needed ("we disagree; can you help us decide?"), not behind someone's back.
- Give feedback with the SBI model: **S**ituation, **B**ehaviour, **I**mpact ("In yesterday's review (S), you rewrote the PR yourself (B); Ana felt her work wasn't trusted (I)").

## Pitfalls
<Warning>
**Common mistakes**

- Burying the ask at the end of a long message.
- Jargon with non-engineers; condescension with anyone.
- Only communicating when things go badly (build trust with regular updates).
- Winning arguments and losing relationships.
- "Silent disagreement": agreeing in the meeting and resisting afterwards.
- Using AI to write long, generic messages; readers notice and skim. Use it to shorten and clarify.
</Warning>

## Senior perspective
- Your writing is how you scale: design docs, strategy docs, onboarding guides and postmortems reach people you'll never meet.
- Build a network across teams before you need it (coffee chats, helping in other teams' channels, reviewing their docs).
- Make the invisible visible: summarise impact for your manager regularly, and highlight others' contributions.
- Read the room politically without being political: know who decides, who influences, and what they care about.

## Interview questions
<Accordion title="How do you explain a technical trade-off to a non-technical stakeholder?">
Start with the decision and its business impact, use an analogy, present two or three options with cost, risk and time in their terms, and give a recommendation.
</Accordion>

<Accordion title="Tell me about a time you convinced another team to change their approach.">
STAR: understood their goals, brought data or a prototype, socialised early, framed it as helping them, offered to do part of the work, shared credit. Include the outcome.
</Accordion>

<Accordion title="How do you communicate bad news, like a missed deadline?">
Early, directly, bottom line first, with the reason, the new forecast and options, and a recommendation. Never surprise people at the deadline.
</Accordion>

<Accordion title="How do you handle a disagreement with your manager?">
Understand their reasoning, share mine with data privately, agree on criteria, and if they still decide otherwise, disagree and commit, while noting risks in writing if they're significant.
</Accordion>

## Related
- [Career Index](/Software%20Engineering/Career/Career%20Index)
- [Mentoring & Leading Without Authority](/Software%20Engineering/Career/Mentoring%20%26%20Leading%20Without%20Authority) · [Technical Design Docs & RFCs](/Software%20Engineering/Career/Technical%20Design%20Docs%20%26%20RFCs) · [Estimation & Planning](/Software%20Engineering/Career/Estimation%20%26%20Planning)
- [Behavioral Interviews & STAR Method](/Software%20Engineering/Interviews/Behavioral%20Interviews%20%26%20STAR%20Method) · [Engineering Levels - Junior to Staff](/Software%20Engineering/Career/Engineering%20Levels%20-%20Junior%20to%20Staff)

## Sources
- Tanya Reilly, *The Staff Engineer's Path* (O'Reilly, 2022)
- Will Larson, *Staff Engineer* (2021) and [staffeng.com: Getting in the room](https://staffeng.com/guides/getting-in-the-room/)
- Barbara Minto, *The Pyramid Principle* (bottom-line-first writing)
- Kim Scott, *Radical Candor* (2017)
- [Center for Creative Leadership: SBI feedback model](https://www.ccl.org/articles/leading-effectively-articles/closing-the-gap-between-intent-vs-impact-sbii/)
- Amazon's [six-page narrative memo practice](https://www.aboutamazon.com/news/company-news/2017-letter-to-shareholders) (2017 shareholder letter)
