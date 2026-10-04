---
title: "Prompt & Context Engineering"
tags: [ai-ml, ai-ml/llm, ai-ml/prompting]
aliases: [Prompt Engineering, Context Engineering, System Prompts, Few-shot Prompting, Structured Outputs, Prompt Caching]
updated: 2026-09-27
---
# Prompt & Context Engineering

<Note>
**TL;DR**

**Prompt engineering** is writing clear instructions for an LLM: a role and rules in the **system prompt**, the task with its context, **examples** (few-shot), and a defined **output format** (ideally enforced with **structured outputs**). **Context engineering**, the 2025 term, widens the lens to *everything* in the context window across a whole agent run: instructions, retrieved documents, tool definitions and results, memory and history, curated to the **smallest set of high-signal tokens**. **Prompt caching** makes the stable parts of that context cheap and fast to resend.
</Note>

## Part 1: Prompt engineering

### Anatomy of a good prompt

```text
SYSTEM
  Role and audience         You are a support assistant for Acme's billing product.
  Rules and constraints     Only answer from the provided docs. If unsure, say so.
  Output format             Reply in Markdown, under 150 words, end with a doc link.
USER
  Context (delimited)       <docs> ... </docs>
  Examples                  <example> ... </example>
  The task                  Answer the customer's question below.
  Input                     <question>Why was I charged twice?</question>
```

Principles:
1. **Be clear and direct.** Write as if briefing a smart new colleague with no context: the goal, who it's for, what good looks like, and what to avoid.
2. **Give context and reasons.** "Keep it short *because it's shown in a mobile notification*" works better than "keep it short".
3. **Separate instructions from data** with delimiters (XML-style tags, Markdown headings, fenced blocks). This improves accuracy and is a first (not sufficient) step against prompt injection.
4. **Show, don't just tell: examples.** A few diverse examples of input → output teach format and edge cases faster than paragraphs of rules.
5. **Specify the output**: format, length, schema, tone. Better still, enforce it with structured outputs.
6. **Say what to do, not only what not to do.** "Write in plain prose paragraphs" beats "don't use bullet points".
7. **Let it think when the task is hard**: ask for reasoning before the answer, or use a reasoning model / extended thinking. See [Transformers & How LLMs Work](/Software%20Engineering/AI%20%26%20ML/Transformers%20%26%20How%20LLMs%20Work).
8. **Break complex tasks into steps** (prompt chaining) rather than one mega-prompt. See [AI Agents & Agentic Patterns](/Software%20Engineering/AI%20%26%20ML/AI%20Agents%20%26%20Agentic%20Patterns).
9. **Iterate against evals**, not vibes: keep a test set and measure every prompt change ([LLM Evaluation & Observability](/Software%20Engineering/AI%20%26%20ML/LLM%20Evaluation%20%26%20Observability)).

### System prompts
The system prompt sets persistent behaviour: role, rules, tone, tools, output conventions. Keep it at the right altitude: specific enough to guide behaviour, general enough not to hard-code brittle if-else logic. Put **stable** content here (it's cacheable) and **per-request** content in the user turn.

### Zero-shot, few-shot, chain of thought

| Technique | What | When |
|---|---|---|
| **Zero-shot** | instructions only | simple, well-known tasks |
| **Few-shot** | 2–5 input/output examples | specific formats, tone, classification labels, tricky edge cases |
| **Chain of thought** | "think step by step" / reasoning before answering | math, logic, multi-step analysis (built into reasoning models) |
| **Role prompting** | "You are a senior security reviewer" | sets vocabulary, depth and focus |
| **Prefill** (some APIs) | start the assistant's reply for it (e.g. `{`) | force a format or skip preamble |

<Info>
**Few-shot classification**

```text
Classify the support ticket as billing, bug, or feature_request.
Reply with the label only.

<example><ticket>I was charged twice this month</ticket><label>billing</label></example>
<example><ticket>Export button does nothing on Safari</ticket><label>bug</label></example>
<example><ticket>Could you add dark mode?</ticket><label>feature_request</label></example>

<ticket>{{ticket_text}}</ticket>
```
Make examples **diverse** (the model copies patterns, including accidental ones like always the same length) and include tricky cases.
</Info>

### Structured outputs
Free-text JSON "please reply in JSON" breaks in production: missing fields, extra prose, invalid syntax. **Structured outputs** constrain decoding to a JSON Schema so the response always parses.

```python
# OpenAI-style (Python SDK with Pydantic)
from typing import Literal
from pydantic import BaseModel
class Ticket(BaseModel):
    category: Literal["billing", "bug", "feature_request"]
    urgency: int          # 1-5
    summary: str

resp = client.responses.parse(model=MODEL, input=ticket_text, text_format=Ticket)
ticket = resp.output_parsed   # a validated Ticket
```

- Tool/function calling uses the same mechanism: the model emits arguments matching your schema.
- Schemas still need validation of **meaning** (is the urgency sensible? does the ID exist?).
- Keep schemas simple; very deep or huge schemas reduce quality and may hit provider limits.

<Info>
**🆕 Structured outputs across providers**

- **OpenAI** Structured Outputs (`strict: true` / `json_schema`): 2024-08-06.
- **Anthropic** structured outputs: public beta 2025-11-14, **generally available 2026-02-04**.
- **Gemini** supports a subset of JSON Schema for response schemas.
</Info>

## Part 2: Context engineering

<Info>
**🆕 2025: the term "context engineering"**

In June 2025 Shopify CEO Tobi Lütke described context engineering as providing all the context needed for a task to be plausibly solvable by the LLM, and Andrej Karpathy endorsed the term over "prompt engineering". Anthropic's **"Effective context engineering for AI agents"** (2025-09-29) made it standard vocabulary for agent builders.
</Info>

**Why a new term?** In agents, the prompt you write is a small part of the context. Most tokens come from tool definitions, tool results, retrieved documents, memory and a long history that grows every step. Engineering that whole window is the job.

{/* diagram source: /diagrams/diagram-022.mmd */}
![diagram](/diagrams/diagram-022.svg)

### The problem: context rot
Models have a finite **attention budget**. As the window fills, recall and instruction-following degrade (**context rot**); information in the middle is used least well (**Lost in the Middle**). So more context is not free: every token should earn its place. Anthropic's guiding principle: find the **smallest set of high-signal tokens** that maximise the chance of the desired outcome.

### Techniques

| Technique | What it does | Example |
|---|---|---|
| **Right-altitude system prompt** | clear, minimal, organised into sections | `## Background`, `## Tools`, `## Output` |
| **Lean tools** | few, well-described, non-overlapping tools with concise results | paginate and truncate; return IDs + summaries, not raw dumps |
| **Just-in-time retrieval** | keep lightweight references (paths, URLs, IDs) and load content only when needed | agent runs `grep`/`head` instead of loading the repo |
| **Retrieval (RAG)** | fetch only relevant chunks | [Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG)) |
| **Compaction** | summarise old history when near the limit, keep decisions and open issues, drop stale tool output | long coding sessions |
| **Structured note-taking** | agent writes progress to a file outside the context and re-reads it | `NOTES.md`, todo lists |
| **Sub-agents** | delegate focused work to a fresh context that returns a condensed summary | research sub-agent returns 1–2k tokens |
| **Project memory files** | persistent instructions every session loads | `AGENTS.md`, `CLAUDE.md` ([AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development)) |
| **Skills / progressive disclosure** | only a short description is always loaded; full instructions load on demand | Agent Skills |
| **Ordering** | stable content first (cacheable), key instructions and the question near the end | long-document Q&A |

### Context engineering checklist for an agent
- What does the model need to know **at this step**, and what can it fetch itself?
- Are tool results trimmed to what matters?
- Is there a compaction or note-taking strategy for runs longer than one window?
- Do instructions conflict (system prompt vs AGENTS.md vs tool descriptions)?
- Is untrusted content (web pages, emails, user files) clearly marked and kept away from high-privilege actions? ([AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps))

## Part 3: Prompt caching

LLM providers can cache the processed prefix of a prompt (the KV cache). If the next request starts with the **exact same prefix**, the cached part is billed at a fraction of the price and processed much faster.

**Design rule: stable first, variable last.**

```text
[tools] [system prompt] [long documents / few-shot examples]  ← cached prefix
[conversation history]                                          ← grows, cache up to last turn
[new user message]                                              ← changes every call
```

Anything that changes early in the prompt (a timestamp in the system prompt, reordered tools, a per-user string at the top) **invalidates the cache** for everything after it.

| Provider | How | Key numbers (check current docs) |
|---|---|---|
| **Anthropic** | explicit `cache_control` breakpoints (up to 4), or automatic caching | 5-minute default lifetime refreshed on each hit, optional 1-hour; cache writes 1.25× (5 min) or 2× (1 h) base input price; reads about 0.1× base (lower on some newer models); minimum cacheable length 512–4,096 tokens depending on model |
| **OpenAI** | automatic for prompts ≥ 1,024 tokens; optional `prompt_cache_key` | cached input discounted (up to 90% on current models); retention minutes to hours depending on model and settings |
| **Google Gemini** | implicit caching (from 2025-05-08 on 2.5 models) plus explicit context caching | 75% discount on implicit cache hits at launch |

```python
# Anthropic: cache a long, stable system prompt
client.messages.create(
    model=MODEL, max_tokens=1024,
    system=[{"type": "text", "text": LONG_POLICY_DOC,
             "cache_control": {"type": "ephemeral"}}],
    messages=[{"role": "user", "content": question}],
)
# usage.cache_creation_input_tokens on the first call, cache_read_input_tokens after
```

## Common pitfalls
- Vague instructions ("make it better") and hidden assumptions the model can't know.
- Contradictory rules across system prompt, examples and tool descriptions.
- Examples that are all alike, so the model overfits to them.
- Asking for JSON in prose instead of using structured outputs.
- Dumping entire documents or huge tool outputs into context.
- Putting volatile data at the top of the prompt and killing the cache.
- Tuning prompts on one model and assuming they transfer to the next version unchanged.
- No eval set, so "improvements" are anecdotes.

## Senior perspective
- Prompts are **code**: version them, review them, test them in CI against an eval set, and roll them out like releases.
- Measure **cost, latency and cache hit rate** per request type; context bloat is the most common production cost bug.
- Keep **one source of truth** for instructions an agent reads (e.g. `AGENTS.md`) and keep it short; stale instructions actively mislead.
- Prefer **structural** guarantees (schemas, validation, allow-lists) over "please don't" in prose for anything security- or correctness-critical.
- Re-run evals on every model upgrade; newer models often need *less* prompting (and over-emphatic legacy prompts can cause over-triggering).

## Interview questions

<Accordion title="What's the difference between prompt engineering and context engineering?">
Prompt engineering focuses on writing the instructions for a single call. Context engineering manages everything in the model's context across a whole (often agentic) run: system prompt, tools, retrieved data, memory, history and tool results, curating the smallest high-signal set as the window fills.
</Accordion>

<Accordion title="When do you use few-shot examples?">
When you need a specific format, label set, tone or handling of edge cases that's easier to show than describe. Use a few diverse, representative examples and include hard cases; avoid examples that share accidental patterns.
</Accordion>

<Accordion title="How do you get reliable JSON from an LLM?">
Use the provider's structured outputs / strict tool schemas so decoding is constrained to your JSON Schema, keep the schema simple, validate semantically in code, and handle refusals or validation failures with a retry or fallback.
</Accordion>

<Accordion title="How does prompt caching work and how do you design for it?">
The provider caches the processed prefix of a prompt; later requests with an identical prefix reuse it at lower cost and latency. Put stable content (tools, system prompt, long documents, examples) first and variable content last, avoid changing anything early in the prompt, and monitor cache-read tokens.
</Accordion>

<Accordion title="What is context rot and how do you fight it in a long-running agent?">
Model performance degrades as the context fills with tokens. Fight it with lean tool outputs, just-in-time retrieval, compaction of old history, structured notes stored outside the context, and sub-agents that return condensed summaries.
</Accordion>

## Related
- [AI & ML Index](/Software%20Engineering/AI%20%26%20ML/AI%20%26%20ML%20Index)
- [Transformers & How LLMs Work](/Software%20Engineering/AI%20%26%20ML/Transformers%20%26%20How%20LLMs%20Work) · [AI Agents & Agentic Patterns](/Software%20Engineering/AI%20%26%20ML/AI%20Agents%20%26%20Agentic%20Patterns) · [Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG))
- [Fine-tuning vs RAG vs Prompting](/Software%20Engineering/AI%20%26%20ML/Fine-tuning%20vs%20RAG%20vs%20Prompting) · [LLM Evaluation & Observability](/Software%20Engineering/AI%20%26%20ML/LLM%20Evaluation%20%26%20Observability) · [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps)
- [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development) · [AI Workflow](/Software%20Engineering/AI%20%26%20ML/AI%20Workflow)

## Sources
- Anthropic, [Prompt engineering overview](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview)
- Anthropic, [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (2025-09-29)
- Anthropic, [Prompt caching docs](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching) · [Structured outputs on the Claude Developer Platform](https://claude.com/blog/structured-outputs-on-the-claude-developer-platform)
- OpenAI, [Prompt engineering guide](https://platform.openai.com/docs/guides/prompt-engineering) · [Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching) · [Introducing Structured Outputs in the API](https://openai.com/index/introducing-structured-outputs-in-the-api/) (2024-08-06)
- Google, [Gemini 2.5 models now support implicit caching](https://developers.googleblog.com/gemini-2-5-models-now-support-implicit-caching/) (2025-05-08) · [Structured output](https://ai.google.dev/gemini-api/docs/structured-output)
- Simon Willison, [Context engineering](https://simonwillison.net/2025/Jun/27/context-engineering/) (2025-06-27, citing Tobi Lütke and Andrej Karpathy)
- Liu et al. (2023), [Lost in the Middle](https://arxiv.org/abs/2307.03172)
- Wei et al. (2022), [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903)
