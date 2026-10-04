---
title: "Behavioral Interviews & STAR Method"
tags: [interviews, interviews/behavioral]
aliases: [STAR Method, Behavioral Interview, Behavioural Interviews, Leadership Interview]
updated: 2026-09-27
---
# Behavioral Interviews & STAR Method

<Note>
**TL;DR**

Behavioral interviews test **how you actually worked** in the past, because past behaviour predicts future behaviour. Answer with **STAR** (Situation, Task, Action, Result), keep it to about 2 minutes, spend most of it on *your* actions and decisions, and end with a measurable result and a lesson. Prepare 8–10 stories from your own work that you can adapt to many questions. At senior level the interviewer is listening for **scope, ambiguity, influence and judgment**, not just "I built X".
</Note>

## The idea
- Interviewers ask "Tell me about a time when…" and score your answer against competencies (ownership, collaboration, conflict, delivery, growth, leadership).
- They cannot see your past work, so a clear, specific story is the evidence. Vague answers ("we usually…") score low.
- The behavioral round often decides the **level** you get, not only yes/no. The same coding performance can be down-levelled because the stories show mid-level scope.

## The STAR structure

{/* diagram source: /diagrams/diagram-354.mmd */}
![diagram](/diagrams/diagram-354.svg)

| Part | Share of time | What to say | Common mistake |
|---|---|---|---|
| **S**ituation | ~10% | Team, product, constraint, stakes | Five minutes of backstory |
| **T**ask | ~10% | What *you* owned, what success meant | Describing the team's task only |
| **A**ction | ~60% | Steps, options considered, trade-offs, who you persuaded | "We" everywhere; no reasoning |
| **R**esult | ~20% | Metrics, outcome, what you learned, what you'd change | No result, or a result with no numbers |

Variants: **STAR-L** (add Learning), **CARL** (Context, Action, Result, Learning), **SAIL** (Situation, Action, Impact, Learning). They are the same idea.

## Worked example

<Info>
**"Tell me about a time you disagreed with a technical decision."**

**S:** Our team of 6 was migrating checkout from a monolith to services. The tech lead proposed putting a message queue between checkout and payments.
**T:** I owned the payments integration and was worried about double charges during retries.
**A:** Instead of arguing in the meeting, I wrote a one-page doc comparing the queue design with a synchronous call plus an idempotency key. I ran a quick load test on staging (p99 latency and failure modes) and listed what each option needed for exactly-once charging. I shared it with the lead first, privately, then we reviewed it with the team. We agreed on the synchronous call with idempotency keys for payments, and kept the queue for notifications.
**R:** We shipped two weeks earlier than planned, had zero duplicate charges in the first quarter, and the doc format became our team's standard for design disagreements. I learned to bring data and a written proposal rather than opinions.
</Info>

Why it works: specific numbers, "I" actions, respect for the other person, a trade-off, a result, a lesson.

## Your story bank
Prepare 8–10 stories and map each to several competencies. One good story can answer 3–4 questions.

| Story (your own) | Ownership | Conflict | Failure | Ambiguity | Leadership / influence | Delivery under pressure | Mentoring |
|---|---|---|---|---|---|---|---|
| Hardest bug / incident you resolved | ✅ | | | ✅ | | ✅ | |
| Project you led end to end | ✅ | | | ✅ | ✅ | ✅ | |
| Disagreement with a peer or manager | | ✅ | | | ✅ | | |
| Something that failed / you'd redo | ✅ | | ✅ | | | | |
| Time you changed a team process | | | | | ✅ | | |
| Person you helped grow | | | | | ✅ | | ✅ |
| Tight deadline / scope cut | | ✅ | | | | ✅ | |
| Decision with incomplete data | | | | ✅ | | | |

Template for each story (keep them in this note or a private file):
```text
Title:
Competencies:
Situation (1–2 lines):
Task (my responsibility):
Actions (3–5 bullets, each with WHY):
Result (numbers):
Lesson / what I'd do differently:
Follow-up questions I expect:
```

## Common questions by theme
- **Ownership:** a time you went beyond your role; a project you drove without being asked.
- **Conflict:** disagreement with a teammate, a PM or your manager; pushing back on a deadline.
- **Failure:** a mistake that reached production; a project that was cancelled.
- **Ambiguity:** unclear requirements; a problem nobody owned.
- **Influence:** convincing another team; changing a technical direction.
- **Growth:** feedback you received and acted on; learning something fast.
- **Customer focus:** a trade-off between speed and quality for users.
- **Why us / why leaving:** keep it positive and forward-looking.

## What changes with level (senior signals)

| Level | Story scope interviewers expect |
|---|---|
| Junior | Own tasks well, ask for help at the right time, learn fast, take feedback |
| Mid | Own features end to end, handle a conflict inside the team, unblock yourself |
| Senior | Own a project or system across teams, make trade-offs explicit, mentor others, improve how the team works, handle ambiguity without being told what to do |
| Staff+ | Set direction across several teams, influence without authority, pick what *not* to do, multi-quarter impact, align leadership, grow other seniors |

Signals interviewers write down for senior candidates:
- **Scope and impact:** the problem mattered to the business; numbers show it.
- **Judgment:** you considered options and explained why you chose one; you knew what to cut.
- **Influence:** you changed minds with writing and data, not rank.
- **Multiplier effect:** others got better because of you (reviews, mentoring, docs, tooling).
- **Self-awareness:** honest failure stories with a real lesson; no blaming.
- **Handling conflict maturely:** disagree and commit, escalate cleanly when needed.

See [Engineering Levels - Junior to Staff](/Software%20Engineering/Career/Engineering%20Levels%20-%20Junior%20to%20Staff) for the full expectations per level.

## Pitfalls
<Warning>
**Common mistakes**

- Saying "we" for the whole story. Say "we" for the context and "I" for your actions.
- A "failure" that is secretly a boast ("I work too hard"). Pick a real mistake with a real lesson.
- Blaming others or speaking badly of a past employer.
- No numbers. Estimate if needed: "cut build time from about 20 to 6 minutes".
- Rambling. Practise out loud; aim for 2 minutes and let them ask follow-ups.
- Inventing or borrowing stories. Follow-up questions ("what did your manager say?", "what would you do differently?") expose them quickly.
- Reading an AI-written script. Interviewers notice, and it hurts trust; use AI to *practise* (mock questions, feedback) not to answer live.
</Warning>

## Senior perspective
- Treat the behavioral round like a design interview about *yourself*: clear structure, trade-offs, and results.
- Amazon's Leadership Principles and similar competency lists (ownership, bias for action, dive deep, disagree and commit) are useful even for other companies: tag your stories with them.
- Prepare questions for the interviewer that show senior thinking: "How are technical decisions made across teams?", "What does the path from senior to staff look like here?", "What is the biggest risk in this team's roadmap?"
- For staff interviews, bring one story about **choosing not to build something** and one about **aligning several teams**.

## Interview questions
<Accordion title="Tell me about yourself. How should you answer?">
A 60–90 second "present, past, future" pitch: current role and what you own, one or two past highlights with impact, why this role is the next step. Tailor it to the job description; it is not a life story.
</Accordion>

<Accordion title="Tell me about a failure.">
Choose a real, meaningful mistake you owned (a bad migration, a missed estimate). STAR, then focus on what you changed afterwards (a checklist, a test, a process) and evidence it worked. Don't pick a catastrophe you caused through negligence, and don't pick a fake one.
</Accordion>

<Accordion title="Tell me about a conflict with a coworker.">
Show you sought to understand their view, moved the discussion to data and shared goals, found a decision (or escalated respectfully) and kept the relationship. End with the outcome and what you learned about communication.
</Accordion>

<Accordion title="Describe a time you had to make a decision without enough information.">
Explain the options, what data you could get cheaply, how you reduced risk (reversible choice, feature flag, time-boxed spike), the decision, and how you checked it later.
</Accordion>

<Accordion title="How do you prioritise when everything is urgent?">
Clarify impact and deadlines with stakeholders, rank by value and cost of delay, make trade-offs visible ("if we do X, Y slips by a week"), agree, and communicate. Give a concrete example.
</Accordion>

<Accordion title="Tell me about a time you influenced a decision without authority.">
Written proposal, data or a prototype, early 1:1s with key people, addressing their concerns, and letting others share ownership of the idea. See [Communication & Influence for Engineers](/Software%20Engineering/Career/Communication%20%26%20Influence%20for%20Engineers).
</Accordion>

## Related
- [Interviews Index](/Software%20Engineering/Interviews/Interviews%20Index)
- [Interview Process & Preparation Plan](/Software%20Engineering/Interviews/Interview%20Process%20%26%20Preparation%20Plan) · [Salary Negotiation & Offers](/Software%20Engineering/Interviews/Salary%20Negotiation%20%26%20Offers)
- [Engineering Levels - Junior to Staff](/Software%20Engineering/Career/Engineering%20Levels%20-%20Junior%20to%20Staff) · [Communication & Influence for Engineers](/Software%20Engineering/Career/Communication%20%26%20Influence%20for%20Engineers) · [Mentoring & Leading Without Authority](/Software%20Engineering/Career/Mentoring%20%26%20Leading%20Without%20Authority)

## Sources
- [Amazon: Leadership Principles](https://www.amazon.jobs/content/en/our-workplace/leadership-principles)
- [Google re:Work: Use structured interviewing](https://rework.withgoogle.com/guides/hiring-use-structured-interviewing/steps/introduction/)
- [Tech Interview Handbook: Behavioral interviews](https://www.techinterviewhandbook.org/behavioral-interview/)
- Gayle Laakmann McDowell, *Cracking the Coding Interview* (6th ed.), behavioral chapter
- Gergely Orosz, *The Software Engineer's Guidebook* (2023)
