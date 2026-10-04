---
title: "LLM Evaluation & Observability"
tags: [ai-ml, ai-ml/llmops]
aliases: [LLM Evals, Evals, LLM-as-a-Judge, Golden Dataset, LLM Tracing, LLMOps, LLM Observability]
updated: 2026-09-27
---
# LLM Evaluation & Observability

<Note>
**TL;DR**

LLM outputs are non-deterministic free text, so you can't unit-test them with `assertEquals` alone. **Evals** are the test suite: a **golden dataset** of representative inputs, plus **graders** (code checks, **LLM-as-judge** with a rubric, and human review) that score each output. Run them offline on every prompt, model or retrieval change, and online on sampled production traffic. **Observability** means **tracing** every request (prompts, retrieved docs, tool calls, tokens, latency, cost) so failures can be found, replayed and turned into new eval cases.
</Note>

## Why evals are different
- Many valid answers exist; exact-match rarely works.
- Behaviour shifts with prompt wording, model version, temperature and retrieved context.
- Failures are often **subtle** (a plausible hallucination) rather than crashes.
- Without evals, teams "vibe check" a few examples, ship, and discover regressions from users.

**Evals are to LLM apps what tests are to code.** Every serious change should come with an eval result.

## Vocabulary (Anthropic, "Demystifying evals for AI agents", 2026-01-09)
- **Task**: one input/scenario with success criteria.
- **Trial**: one run of a task (you often run several because outputs vary).
- **Grader**: logic that scores a trial: **code-based**, **model-based**, or **human**.
- **Transcript**: the full record of a trial (messages, tool calls, results).
- **Outcome**: the final state or answer being judged.
- **Harness**: the code that runs tasks, trials and graders.
- **pass@k**: at least one of k trials succeeds (capability). **pass^k**: all k trials succeed (reliability, what users feel).

## Building an eval

{/* diagram source: /diagrams/diagram-009.mmd */}
![diagram](/diagrams/diagram-009.svg)

### 1. Golden dataset
- Start small: **20–50 real examples** beats 1,000 synthetic ones. Grow it from production failures.
- Cover the **distribution** (common requests), **edge cases** (empty input, long input, other languages), and **adversarial** cases (prompt injection, policy violations).
- For each case store: input, context (if fixed), **reference answer or criteria**, tags (for slicing), and difficulty.
- Keep a separate **held-out** set so you don't overfit prompts to the dev set.

```json
{"id": "refund-017", "input": "I was charged twice for March",
 "expected_sources": ["kb/billing/duplicate-charges"],
 "criteria": ["explains refund timeline of 5-7 days", "does not promise an immediate refund", "links the billing page"],
 "tags": ["billing", "refund"]}
```

### 2. Graders

| Grader | How | Good for | Watch out |
|---|---|---|---|
| **Code-based** | exact match, regex, JSON schema validity, numeric tolerance, unit tests on generated code, "was tool X called with arg Y" | anything objectively checkable | brittle for free text |
| **Similarity** | embedding similarity, ROUGE/BLEU against a reference | rough regression signals | weak correlation with quality |
| **LLM-as-judge** | a strong model scores output against a rubric or compares two outputs | faithfulness, helpfulness, tone, rubric adherence | biases; needs calibration |
| **Human** | experts label or compare | ground truth, calibrating judges, high-stakes domains | slow, expensive, inconsistent without guidelines |

Prefer code graders wherever possible; use LLM judges for what code can't check; use humans to validate the judges.

### 3. LLM-as-judge done right

```text
You are grading a customer-support answer.
<question>{q}</question>
<retrieved_context>{ctx}</retrieved_context>
<answer>{a}</answer>

Criteria:
1. Faithfulness: every factual claim is supported by the retrieved context.
2. Completeness: addresses all parts of the question.
3. Policy: never promises refunds outside policy.

For each criterion, first quote the evidence, then give PASS or FAIL.
Return JSON: {"faithfulness": ..., "completeness": ..., "policy": ..., "reasons": "..."}
```

Known biases (Zheng et al., 2023, "Judging LLM-as-a-Judge"): **position bias** (prefers the first/second option), **verbosity bias** (prefers longer answers), **self-enhancement bias** (prefers its own model family's outputs), and limited math/reasoning grading. That paper also found strong judges agreed with humans over 80% of the time, similar to human-human agreement.

Mitigations:
- Use **specific rubrics** with binary or small-scale criteria rather than "rate 1–10".
- For pairwise comparisons, **swap the order** and only count consistent verdicts.
- Ask for **evidence/reasoning before the verdict**.
- Use a **different, strong model** as judge where possible.
- **Calibrate**: label 50–100 cases by hand and measure judge-human agreement before trusting it.

### 4. What to measure by system type

| System | Metrics |
|---|---|
| Classification / extraction | accuracy, precision/recall/F1 per class, schema validity, field-level exact match |
| RAG | recall@k, MRR, faithfulness, answer relevance, citation accuracy ([Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG))) |
| Chat assistant | rubric scores (helpful, correct, safe, on-tone), refusal rate, escalation rate |
| Agents | task success on verifiable outcomes, pass^k, steps and tokens per task, tool-error rate, cost per success |
| Code generation | tests passing, compile rate, review acceptance |
| Safety | jailbreak / injection success rate, PII leakage, policy violations |

Always track **cost** and **latency** (p50/p95 time-to-first-token and total) next to quality.

## Observability: tracing LLM apps

A **trace** captures one request end to end; **spans** capture each step.

```text
trace: POST /chat  (2.8 s, $0.012, user=u_123, prompt_version=v14)
├── span: rewrite_query      model=small   120 tok in / 20 out   180 ms
├── span: hybrid_search      k=50  filters={tenant:42}           90 ms
├── span: rerank             50 → 8                              150 ms
├── span: llm.generate       model=frontier  6.1k in (5.2k cached) / 310 out   2.2 s
│      └── tool_call: get_invoice(id=A-1043) → 200 OK            110 ms
└── span: output_guardrail   pass                                 40 ms
```

Log per span: model and version, prompt template version, full inputs and outputs (with PII redaction), token counts (input, cached, output), latency, cost, tool calls with args and results, errors, and user feedback linked to the trace.

- **OpenTelemetry GenAI semantic conventions** define standard attribute names for model calls, agents and tools (and MCP); they are still in **Development** status, so expect changes. The MCP 2026-07-28 spec documents trace-context propagation in `_meta`.
- Tools: open-source and commercial LLM observability platforms (e.g. Langfuse, Arize Phoenix, LangSmith, Braintrust, Weights & Biases Weave) plus general APM (Datadog, Honeycomb) that ingest OTel spans.

### Online evaluation and feedback loops
- Run LLM judges on a **sample of production traces** (faithfulness, policy) and alert on drops.
- Capture **explicit feedback** (thumbs, edits) and **implicit signals** (retries, copy/paste, abandonment, escalation to a human).
- Turn every confirmed failure into a **new golden-set case**, so it can't regress silently.
- A/B test prompt or model changes with guardrail metrics (cost, latency, safety) as well as quality.

## Pitfalls
- Only testing happy paths; the golden set doesn't look like real traffic.
- Trusting an uncalibrated LLM judge, or letting the judge see the reference answer when grading faithfulness to context.
- Reporting a single average; regressions hide in slices (language, customer tier, long inputs).
- Running each case once for a non-deterministic system (use multiple trials and look at pass^k).
- Logging prompts with PII or secrets in plain text; traces are sensitive data.
- Evals that never run: not wired into CI, so they rot.

## Senior perspective
- **Eval-driven development**: write the eval before changing the prompt or model; a change ships only if it beats the baseline without regressing guardrail metrics.
- Wire evals into **CI** (fast subset on every PR, full suite nightly or before release) and into **model upgrade** checklists.
- Own the **cost of evals** too (judge calls on thousands of cases add up); cache and sample.
- Define **SLOs** for LLM features: success rate, p95 latency, cost per request, safety violation rate.
- Treat traces as **production data** with retention limits, redaction and access control.
- Reviewers ask: what's the eval set and where did it come from? How was the judge validated? What's the rollback plan if online metrics drop?

## Interview questions

<Accordion title="How do you evaluate an LLM feature before shipping?">
Define success criteria, build a golden dataset from real and edge cases, run multiple trials, grade with code checks where possible and a calibrated LLM judge with a rubric elsewhere, compare against the baseline on quality, cost and latency, and look at slices, not just averages.
</Accordion>

<Accordion title="What are the weaknesses of LLM-as-a-judge and how do you mitigate them?">
Position, verbosity and self-preference biases and weak grading of math/reasoning. Mitigate with specific binary rubrics, evidence-before-verdict, swapping order in pairwise comparisons, using a different strong model, and measuring agreement with human labels.
</Accordion>

<Accordion title="pass@k vs pass^k?">
pass@k is the chance at least one of k attempts succeeds (useful when you can pick the best, e.g. with tests). pass^k is the chance all k succeed, which reflects the reliability users experience on repeated use; it drops fast if per-trial success isn't very high.
</Accordion>

<Accordion title="What would you put in a trace for an LLM request?">
The whole chain: prompt template version, model and parameters, inputs and outputs (redacted), retrieved documents and scores, each tool call with arguments and results, token counts including cached tokens, latency per step, cost, errors, and user feedback linked to the trace ID.
</Accordion>

<Accordion title="How do production failures feed back into development?">
Sample and review traces (plus automated online judges and user feedback), confirm failures, add them to the golden dataset with expected behaviour, fix, and verify the eval passes before release so the issue can't silently regress.
</Accordion>

## Related
- [AI & ML Index](/Software%20Engineering/AI%20%26%20ML/AI%20%26%20ML%20Index)
- [Step 4 Evaluate the model](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%204%20Evaluate%20the%20model) (classic ML metrics)
- [Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG)) · [AI Agents & Agentic Patterns](/Software%20Engineering/AI%20%26%20ML/AI%20Agents%20%26%20Agentic%20Patterns) · [Prompt & Context Engineering](/Software%20Engineering/AI%20%26%20ML/Prompt%20%26%20Context%20Engineering)
- [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps) · [Fine-tuning vs RAG vs Prompting](/Software%20Engineering/AI%20%26%20ML/Fine-tuning%20vs%20RAG%20vs%20Prompting)

## Sources
- Anthropic, [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (2026-01-09)
- Anthropic, [Define success criteria and build evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)
- OpenAI, [Evals guide](https://platform.openai.com/docs/guides/evals) · [Graders](https://developers.openai.com/api/docs/guides/graders)
- Zheng et al. (2023), [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685)
- OpenTelemetry, [GenAI semantic conventions](https://github.com/open-telemetry/semantic-conventions-genai)
- MCP spec 2026-07-28 changelog (OpenTelemetry `_meta` conventions): https://modelcontextprotocol.io/specification/2026-07-28/changelog
- Hamel Husain, [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/)
