---
title: "AI & ML Index"
tags: [ai-ml, index]
aliases: [AI Home, ML Home, AI and ML, AI & ML Home, AI MOC]
updated: 2026-09-27
---
# AI & ML Index

<Note>
**Start here**

The entry point to all AI and machine learning notes: from classic ML (the AWS DeepRacer and DeepLearning.AI course notes) through neural networks and transformers, to building LLM applications (prompting, context engineering, RAG, agents, MCP), running them safely in production (evals, observability, security), and working with AI coding agents. Every topic note has worked examples, a 🆕 2025–2026 box where things changed, and interview questions.
</Note>

{/* diagram source: /diagrams/diagram-001.mmd */}
![diagram](/diagrams/diagram-001.svg)

## 🧭 Quick links
- 🆕 [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026): dated model releases, agent SDKs, MCP vs A2A
- 🤖 [AI Agents & Agentic Patterns](/Software%20Engineering/AI%20%26%20ML/AI%20Agents%20%26%20Agentic%20Patterns) + [MCP](/Software%20Engineering/AI%20%26%20ML/MCP): the core of modern LLM apps
- 🛡️ [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps): OWASP LLM and Agentic Top 10s
- 🧑‍💻 [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development): coding agents, AGENTS.md, reviewing AI code

## 1. Beginner: machine learning foundations
Course notes: [Machine Learning](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Machine%20Learning) (AWS DeepRacer "Introduction to machine learning" + Coursera/DeepLearning.AI)

| Concepts | The ML process | Case studies |
|---|---|---|
| [Introduction to Machine Learning](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Introduction%20to%20Machine%20Learning) | [Major steps in ML](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Major%20steps%20in%20ML) | [Case Studies that use ML](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Case%20Studies%20that%20use%20ML) |
| [Components of ML](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Components%20of%20ML) | [Step 1 Define The problem](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%201%20Define%20The%20problem) | [Example 1 Predicting Home Prices](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Example%201%20Predicting%20Home%20Prices) |
| [Supervised vs Unsupervised Machine learning](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Supervised%20vs%20Unsupervised%20Machine%20learning) | [Step 2 Building Dataset](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%202%20Building%20Dataset) | [Example 2 Micro-genre Book Genre](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Example%202%20Micro-genre%20Book%20Genre) |
| [ML Core Concepts](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/ML%20Core%20Concepts) | [Step 3 Train the model](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%203%20Train%20the%20model) | [Example 3 Detecting Spills](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Example%203%20Detecting%20Spills) |
| | [Step 4 Evaluate the model](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%204%20Evaluate%20the%20model) | |
| | [Step 5 Use the model](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%205%20Use%20the%20model) | |

## 2. Intermediate: how modern AI works and how to build with it
1. [Neural Networks & Deep Learning Basics](/Software%20Engineering/AI%20%26%20ML/Neural%20Networks%20%26%20Deep%20Learning%20Basics): perceptron, backprop, activations, CNN/RNN, overfitting
2. [Transformers & How LLMs Work](/Software%20Engineering/AI%20%26%20ML/Transformers%20%26%20How%20LLMs%20Work): tokens, attention, training stages, reasoning models, sampling
3. [Prompt & Context Engineering](/Software%20Engineering/AI%20%26%20ML/Prompt%20%26%20Context%20Engineering): system prompts, few-shot, structured outputs, context engineering, prompt caching
4. [Embeddings & Semantic Search](/Software%20Engineering/AI%20%26%20ML/Embeddings%20%26%20Semantic%20Search): vectors, similarity, ANN indexes, hybrid search
5. [Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG)): chunking, retrieval, reranking, agentic RAG, evaluation
   - Storage: [Vector Databases & pgvector](/Software%20Engineering/Databases/Core%20Concepts/Vector%20Databases%20%26%20pgvector) (Databases domain)
6. [Fine-tuning vs RAG vs Prompting](/Software%20Engineering/AI%20%26%20ML/Fine-tuning%20vs%20RAG%20vs%20Prompting): decision guide, LoRA/QLoRA

## 3. Senior: agents, production and engineering practice
- [AI Agents & Agentic Patterns](/Software%20Engineering/AI%20%26%20ML/AI%20Agents%20%26%20Agentic%20Patterns): agent loop, tools, memory, workflow patterns, multi-agent, failure modes
- [MCP](/Software%20Engineering/AI%20%26%20ML/MCP): Model Context Protocol, building servers, spec history, security
- [LLM Evaluation & Observability](/Software%20Engineering/AI%20%26%20ML/LLM%20Evaluation%20%26%20Observability): golden sets, graders, LLM-as-judge, tracing
- [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps): prompt injection, OWASP Top 10s, tool permissioning
- [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development): coding agents, spec-driven development, AGENTS.md, the senior role
- [AI Workflow](/Software%20Engineering/AI%20%26%20ML/AI%20Workflow): my scope → architect → develop → check → test → review workflow
- [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026): models, SDKs, MCP vs A2A
- Frontend side: [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI) (MCP in dev tooling, WebMCP, Chrome built-in AI, streaming UI)

## Suggested study paths
<Info>
**🔁 Beginner to senior (in order)**

1. [Machine Learning](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Machine%20Learning) course notes in order, with [ML Core Concepts](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/ML%20Core%20Concepts) alongside
2. [Neural Networks & Deep Learning Basics](/Software%20Engineering/AI%20%26%20ML/Neural%20Networks%20%26%20Deep%20Learning%20Basics) → [Transformers & How LLMs Work](/Software%20Engineering/AI%20%26%20ML/Transformers%20%26%20How%20LLMs%20Work)
3. [Prompt & Context Engineering](/Software%20Engineering/AI%20%26%20ML/Prompt%20%26%20Context%20Engineering) → [Embeddings & Semantic Search](/Software%20Engineering/AI%20%26%20ML/Embeddings%20%26%20Semantic%20Search) → [Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG)) → [Fine-tuning vs RAG vs Prompting](/Software%20Engineering/AI%20%26%20ML/Fine-tuning%20vs%20RAG%20vs%20Prompting)
4. [AI Agents & Agentic Patterns](/Software%20Engineering/AI%20%26%20ML/AI%20Agents%20%26%20Agentic%20Patterns) → [MCP](/Software%20Engineering/AI%20%26%20ML/MCP)
5. [LLM Evaluation & Observability](/Software%20Engineering/AI%20%26%20ML/LLM%20Evaluation%20%26%20Observability) → [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps)
6. [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development) → [AI Workflow](/Software%20Engineering/AI%20%26%20ML/AI%20Workflow) → [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026)
</Info>

<Info>
**⚡ Building an LLM feature next week**

[Prompt & Context Engineering](/Software%20Engineering/AI%20%26%20ML/Prompt%20%26%20Context%20Engineering) → [Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG)) → [LLM Evaluation & Observability](/Software%20Engineering/AI%20%26%20ML/LLM%20Evaluation%20%26%20Observability) → [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps) → [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI) (streaming UI)
</Info>

<Info>
**🎯 AI engineering interview prep**

Interview questions in [Transformers & How LLMs Work](/Software%20Engineering/AI%20%26%20ML/Transformers%20%26%20How%20LLMs%20Work), [Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG)), [AI Agents & Agentic Patterns](/Software%20Engineering/AI%20%26%20ML/AI%20Agents%20%26%20Agentic%20Patterns), [LLM Evaluation & Observability](/Software%20Engineering/AI%20%26%20ML/LLM%20Evaluation%20%26%20Observability), [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps), [Fine-tuning vs RAG vs Prompting](/Software%20Engineering/AI%20%26%20ML/Fine-tuning%20vs%20RAG%20vs%20Prompting) and [Step 4 Evaluate the model](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%204%20Evaluate%20the%20model) (classic ML metrics)
</Info>

## 🆕 2025–2026 in this area
- **Reasoning models** go mainstream: DeepSeek-R1 (2025-01-20, open), Claude 3.7 Sonnet hybrid reasoning (2025-02-24), o3 (2025-04-16), GPT-5 (2025-08-07) → [Transformers & How LLMs Work](/Software%20Engineering/AI%20%26%20ML/Transformers%20%26%20How%20LLMs%20Work), [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026)
- **Model releases** through Claude Opus 5.5 (2026-09-22), GPT-5.6 (2026-07), Gemini 3.x, DeepSeek V4 (2026-04-24), Mistral 3, Qwen3/3.5 → [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026)
- **Agent SDKs**: OpenAI Agents SDK (2025-03-11), Google ADK (2025-04-09), Claude Agent SDK (2025-09-29) → [AI Agents & Agentic Patterns](/Software%20Engineering/AI%20%26%20ML/AI%20Agents%20%26%20Agentic%20Patterns)
- **MCP** spec revisions 2025-03-26, 2025-06-18, 2025-11-25 and the stateless **2026-07-28**; MCP Registry (2025-09-08); MCP Apps (2026-01-26); TypeScript SDK v2 (2026-07) → [MCP](/Software%20Engineering/AI%20%26%20ML/MCP)
- **Agentic AI Foundation** (Linux Foundation, 2025-12-09) with MCP, goose, AGENTS.md; A2A joined 2026-08-27; A2A v1.0 (2026-03-12) → [MCP](/Software%20Engineering/AI%20%26%20ML/MCP), [LLM Landscape 2025-2026](/Software%20Engineering/AI%20%26%20ML/LLM%20Landscape%202025-2026)
- **Context engineering** as a discipline (June 2025; Anthropic guide 2025-09-29) → [Prompt & Context Engineering](/Software%20Engineering/AI%20%26%20ML/Prompt%20%26%20Context%20Engineering)
- **Structured outputs** GA on Claude (2026-02-04); Gemini implicit caching (2025-05-08) → [Prompt & Context Engineering](/Software%20Engineering/AI%20%26%20ML/Prompt%20%26%20Context%20Engineering)
- **Agent evals**: Anthropic "Demystifying evals for AI agents" (2026-01-09); OpenTelemetry GenAI conventions (still in development) → [LLM Evaluation & Observability](/Software%20Engineering/AI%20%26%20ML/LLM%20Evaluation%20%26%20Observability)
- **Security**: lethal trifecta (2025-06-16), CaMeL (2025-03), OWASP Top 10 for Agentic Applications (2025-12-09) → [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps)
- **Coding agents**: Claude Code GA (2025-05-22), Copilot coding agent GA (2025-09-25), Codex GA (2025-10-06), Cursor 2.0/3, Kiro, Spec Kit, AGENTS.md; METR study (2025-07-10) → [AI-Assisted Software Development](/Software%20Engineering/AI%20%26%20ML/AI-Assisted%20Software%20Development)
- **AWS Panorama** end of support (2026-05-31) → [Example 3 Detecting Spills](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Example%203%20Detecting%20Spills)

## Related
- [Software Engineering](/Software%20Engineering/Software%20Engineering), the vault root
- [Frontend Home](/Software%20Engineering/Frontend/Frontend%20Home) · [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI)
- [Vector Databases & pgvector](/Software%20Engineering/Databases/Core%20Concepts/Vector%20Databases%20%26%20pgvector)
