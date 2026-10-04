---
title: "Productivity & Deep Work"
tags: [career, career/productivity]
aliases: [Productivity, Deep Work, Focus, Developer Productivity, Flow State]
updated: 2026-09-27
---
# Productivity & Deep Work

<Note>
**TL;DR**

Engineering output depends on long, uninterrupted blocks of focus (**deep work**) and on **working on the right things**. Protect focus time, batch shallow work (email, chat, meetings), keep a single trusted task list, and reduce the cost of context switches. At team level, productivity is a system property: fast feedback loops, small batches, low friction and good tooling matter more than individual heroics. Measure outcomes, not hours or lines of code.
</Note>

## The idea
- **Deep work** (Cal Newport): focused, distraction-free work that pushes your ability and creates value; hard to replicate. **Shallow work**: logistical tasks done while distracted.
- A **maker's schedule** (Paul Graham) works in half-day units; one meeting in the middle of the afternoon can destroy it. A manager's schedule works in hour slots. Senior engineers live in both and must manage the clash.
- Context switches are expensive: getting back into complex code after an interruption often takes many minutes.

## Personal practices

### Protect focus
- Block **2–4 hour focus sessions** in the calendar; cluster meetings (e.g. afternoons, or a no-meeting day).
- Notifications off during focus; check chat at set times. Set status: "Focusing until 12:00, ping for incidents".
- One task at a time; write down stray thoughts to handle later.
- Before stopping, write a note on where you are and the next step (makes restarting cheap).

### Choose the right work
- Daily: pick the **one most important thing** before opening chat.
- Weekly: review priorities against team goals; say no (or "not now") to low-value requests.
- Eisenhower matrix: important vs urgent. Seniors spend more time on important-not-urgent work (design, tooling, mentoring, debt).
- Time-box investigations and ask for help after the box.

### Systems
- A single trusted task list (GTD-style capture → clarify → organise → review).
- Keep an engineering log / brag document: what you did, learned, decided. It helps with focus, reviews and promotions.
- Automate repetitive work (scripts, aliases, snippets, templates). Rule of thumb: if you do it a third time, automate it.
- Master your tools: editor shortcuts, debugger, git, shell, search.

### Energy
- Sleep, breaks and exercise are productivity tools, not luxuries. Work in cycles (e.g. 50–90 minutes, then a break).
- Sustainable pace beats crunch; crunch creates bugs and burnout.

## Team productivity
| Lever | Example |
|---|---|
| Fast feedback | CI under ~10 minutes, hot reload, fast tests, preview environments |
| Small batches | Small PRs, trunk-based development, feature flags ([Code Review Best Practices](/Software%20Engineering/Career/Code%20Review%20Best%20Practices)) |
| Less waiting | Review SLAs, fewer handoffs, self-service infrastructure |
| Fewer interruptions | Rotating "interrupt handler"/on-call buddy shields the rest of the team |
| Clear priorities | One visible roadmap, limited work in progress |
| Good docs | Onboarding, runbooks, ADRs cut repeated questions |

### Measuring (carefully)
- **DORA metrics** for delivery: deployment frequency, lead time for changes, change failure rate, time to restore (plus rework rate added by DORA in 2024).
- **SPACE framework**: Satisfaction, Performance, Activity, Communication/collaboration, Efficiency/flow. Use several dimensions; never a single metric.
- **DevEx**: feedback loops, cognitive load, flow state.
- Never measure individuals by lines of code, commits or PR counts (Goodhart's law: when a measure becomes a target, it stops being a good measure).

## AI tools and productivity
- AI assistants remove some typing and lookup work, but add review and verification work. Evidence is mixed: DORA 2025 reports that over 80% of respondents believe AI increased their productivity, while METR's 2025 randomised trial found experienced developers on familiar codebases were **19% slower** with early-2025 AI tools despite believing they were faster. See [Senior Engineer in the AI Era](/Software%20Engineering/Career/Senior%20Engineer%20in%20the%20AI%20Era).
- Use AI where it clearly helps (boilerplate, tests scaffolding, unfamiliar APIs, explanations) and keep deep thinking (design, debugging hard problems) as focused human work.
- Agent-style tools running in the background can be a new source of interruptions; batch their reviews like any other shallow work.

## Pitfalls
<Warning>
**Common mistakes**

- Being "busy" (chat, meetings) instead of productive.
- Saying yes to everything; the important work gets the leftovers.
- Heroic late nights that become the team's normal.
- Optimising personal tooling endlessly instead of doing the work.
- Measuring individuals with activity metrics.
</Warning>

## Senior perspective
- Your calendar is a design decision: protect your own focus and the team's.
- Remove friction for everyone: a 5-minute faster CI pipeline saves more than any personal trick.
- Model healthy behaviour: sustainable hours, clear priorities, saying no with reasons.
- Balance maker and manager time deliberately (e.g. focus mornings, meetings afternoons).

## Interview questions
<Accordion title="How do you manage your time with many competing requests?">
Clarify impact and urgency with requesters, keep a single prioritised list, protect focus blocks for the most important work, batch shallow tasks, and say no or "later" with a reason, escalating priority conflicts to my manager.
</Accordion>

<Accordion title="How would you improve a team's productivity?">
Measure where time goes (lead time, review wait, CI time, interruptions, survey developers), fix the biggest bottlenecks first (slow CI, big PRs, unclear priorities), and track DORA/SPACE-style metrics over time, never individual activity counts.
</Accordion>

<Accordion title="What are the DORA metrics?">
Deployment frequency, lead time for changes, change failure rate and failed-deployment recovery time (time to restore), with rework rate added more recently. They measure software delivery throughput and stability.
</Accordion>

<Accordion title="Has AI made you more productive? How do you know?">
Give an honest, specific answer: which tasks it speeds up, which it doesn't, how you verify output, and whether you've measured (e.g. cycle time) rather than relied on feeling, mentioning that studies show perception and measurement can differ.
</Accordion>

## Related
- [Career Index](/Software%20Engineering/Career/Career%20Index)
- [Senior Engineer in the AI Era](/Software%20Engineering/Career/Senior%20Engineer%20in%20the%20AI%20Era) · [Estimation & Planning](/Software%20Engineering/Career/Estimation%20%26%20Planning) · [Debugging Methodology](/Software%20Engineering/Career/Debugging%20Methodology) · [Code Review Best Practices](/Software%20Engineering/Career/Code%20Review%20Best%20Practices)

## Sources
- Cal Newport, *Deep Work* (2016)
- Paul Graham, [Maker's Schedule, Manager's Schedule](https://www.paulgraham.com/makersschedule.html) (2009)
- Forsgren, Storey et al., [The SPACE of Developer Productivity (ACM Queue, 2021)](https://queue.acm.org/detail.cfm?id=3454124)
- Noda, Storey, Forsgren, Greiler, [DevEx: What Actually Drives Productivity (ACM Queue, 2023)](https://queue.acm.org/detail.cfm?id=3595878)
- [DORA: Software delivery performance metrics](https://dora.dev/guides/dora-metrics-four-keys/)
- [Google Cloud: Announcing the 2025 DORA report (2025-09-24)](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report)
- [METR (2025-07-10): Early-2025 AI and experienced OSS developer productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/)
- David Allen, *Getting Things Done* (rev. ed. 2015)
