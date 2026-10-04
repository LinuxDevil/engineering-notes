---
title: "AI in Frontend - MCP, WebMCP & Built-in AI"
tags: [frontend, frontend/ai]
aliases: [Model Context Protocol, WebMCP, Built-in AI, AI in Frontend]
updated: 2026-09-27
---
# AI in Frontend - MCP, WebMCP & Built-in AI

<Note>
**TL;DR**

AI touches front end work in three ways: (1) **agents use your tools** through the **Model Context Protocol (MCP)**: DevTools, framework dev servers, and now websites themselves via **WebMCP**; (2) **the browser ships models**: Chrome's built-in AI APIs (Translator, Summarizer, Prompt API...) run on-device; (3) **you build AI features** (chat, streaming answers, generative UI) with libraries like the Vercel AI SDK. The FE skills that matter are streaming UI, cancellation, error/latency UX, and security (prompt injection, never shipping API keys to the browser).
</Note>

```mermaid
flowchart LR
  subgraph Dev["1. Agents in your dev loop"]
    Agent["Coding agent<br/>(MCP host)"] --> DT["Chrome DevTools for agents"]
    Agent --> FW["Next.js / Angular MCP servers"]
    Agent --> PW["Playwright MCP"]
  end
  subgraph Web["2. Agents on your website"]
    BrowserAgent["Browser agent"] --> WMCP["WebMCP tools<br/>document.modelContext"]
  end
  subgraph Prod["3. AI features you ship"]
    UI["Chat / streaming UI"] --> Server["Your server route<br/>(holds API keys)"] --> LLM["Cloud LLM"]
    UI --> OnDevice["Chrome built-in AI<br/>(on-device models)"]
  end
```

## 1. MCP: Model Context Protocol

### What it is
MCP is an open protocol (originally from Anthropic, Nov 2024) that standardizes how AI applications connect to external tools and data. Think "USB-C for AI tools": write a server once and any MCP-capable app (Claude, IDEs, coding agents) can use it. Messages are **JSON-RPC 2.0**.

### Roles
| Role | What it is | Example |
|---|---|---|
| **Host** | The AI application the user interacts with; manages one or more clients, permissions and the model | Claude Desktop/Code, Cursor, VS Code agent mode |
| **Client** | A connector inside the host with a 1:1 connection to one server | The host's connection to the "chrome-devtools" server |
| **Server** | A program that exposes capabilities | Chrome DevTools for agents, a GitHub server, your own internal API server |

```mermaid
sequenceDiagram
  participant User
  participant Host as Host (AI app + model)
  participant Client as MCP client
  participant Server as MCP server
  Host->>Client: start
  Client->>Server: initialize (capabilities)
  Server-->>Client: capabilities (tools, resources, prompts)
  Client->>Server: tools/list
  Server-->>Client: [ { name, description, inputSchema } ]
  User->>Host: "Why is LCP slow on /flights?"
  Host->>Client: model decides to call tool
  Client->>Server: tools/call { name: "performance_trace", arguments }
  Server-->>Client: result (text / structured content)
  Client-->>Host: result goes back into model context
  Host-->>User: answer
```

### Primitives
**Server-side** (what a server offers):
- **Tools**: functions the model can call (`search_flights`, `take_screenshot`). Described by a name, description and **JSON Schema** for inputs. Model-controlled, so hosts ask the user to approve risky calls.
- **Resources**: read-only data identified by URIs (files, DB rows, docs) that the app can attach as context. Application-controlled.
- **Prompts**: reusable prompt templates the user picks (e.g., a "/review-component" command). User-controlled.

**Client-side** (what a host can offer back to servers): **sampling** (server asks the host's model for a completion), **roots** (which folders/URIs the server may work in), and **elicitation** (server asks the user for input).

### Transports
- **stdio**: the host launches the server as a local process and talks over stdin/stdout. Most local dev tools work this way.
- **Streamable HTTP**: the server is a remote HTTP endpoint (POST for requests, optional SSE streaming for responses/notifications). It replaced the older "HTTP + SSE" transport in the 2025 spec revisions. Remote servers use OAuth-based authorization.

<Info>
**A minimal MCP server config in a host (conceptual)**

```json
{
  "mcpServers": {
    "chrome-devtools": { "command": "npx", "args": ["chrome-devtools-mcp@latest"] }
  }
}
```
The exact config file and keys depend on the host; check its docs.
</Info>

### Governance
On **2025-12-09** Anthropic donated MCP to the **Agentic AI Foundation** under the Linux Foundation, together with goose and AGENTS.md. MCP is now vendor-neutral.

### MCP for front end developers (2025–2026)
- **Chrome DevTools MCP**: public preview in Sept 2025, renamed **Chrome DevTools for agents 1.0** on **2026-05-19**. Agents can drive a real Chrome: navigate, click, read console and network, record performance traces, run Lighthouse, emulate devices, take heap snapshots, and share an authenticated session. Use it to let an agent *verify* its own UI changes instead of guessing.
- **Next.js**: **Next.js DevTools MCP** (16.0) exposes routes, logs and errors to agents; 16.1 added a `get_routes` tool; 16.2 puts an **`AGENTS.md`** in `create-next-app`; 16.3 writes version-matched docs into `AGENTS.md` so agents stop using outdated APIs.
- **Angular**: `llms.txt` (v20); **Angular CLI MCP server** stable (v21); MCP dev-server tools and Agent Skills (v22).
- **Playwright MCP** (a separate server) lets agents drive browsers for testing; Playwright 1.56 added Test Agents (planner, generator, healer).
- **AGENTS.md**: a plain markdown file at the repo root that tells coding agents how to build, test and follow conventions (like a README for agents).

<Info>
**🆕 2025–2026**

- MCP joined the Agentic AI Foundation (Linux Foundation), 2025-12-09.
- Chrome DevTools for agents 1.0, 2026-05-19.
- Next.js DevTools MCP (16.0, Oct 2025); AGENTS.md in create-next-app (16.2, Mar 2026).
- Angular CLI MCP server stable (v21, Nov 2025).
- WebMCP origin trial from Chrome 149 (2026-06-09).
</Info>

## 2. WebMCP: your website as an MCP-style tool provider
WebMCP lets a **web page** register tools that a browser-based agent can call, instead of the agent scraping the DOM and clicking buttons. The page exposes intent-level actions ("add to cart", "search flights") with schemas, running in the user's session.

- Status: early preview in **Chrome 146**, **origin trial from Chrome 149** (2026-06-09). Not a standard, Chromium only.
- Two forms: an **imperative** API and a **declarative** form-attribute API (annotating existing forms).
- The API lives on **`document.modelContext`** in the current docs (many articles cite `navigator.modelContext`, which is outdated).
- Angular v22 ships experimental WebMCP helpers (`declareExperimentalWebMcpTool`).

<Info>
**Registering a tool (origin trial API, may change)**

```js
if ('modelContext' in document) {
  await document.modelContext.registerTool({
    name: 'add_to_cart',
    description: 'Add product by id',
    inputSchema: { type: 'object', properties: { id: { type: 'string' } }, required: ['id'] },
    execute: async ({ id }) => { await cart.add(id); return `Added ${id}`; },
  });
}
```
Feature-detect, treat inputs as untrusted (validate like any API input), and require user confirmation for anything with side effects like payments.
</Info>

**Why it matters for FE**: the site defines the contract, so agents are faster and less brittle than DOM automation, and you keep control of what agents can do. It is the same design thinking as a public API: clear names, tight schemas, idempotent actions, good errors.

## 3. Chrome built-in AI APIs (on-device)
Chrome ships small on-device models (Gemini Nano) behind task APIs. Pros: no server cost, low latency after download, data stays on device, works offline. Cons: **Chromium only**, needs model download (large, hardware requirements), quality below big cloud models. Always **feature-detect and fall back** to a server.

| API | Purpose | Status (per Chrome docs) |
|---|---|---|
| Translator | Translate text between languages | Stable, Chrome 138 |
| Language Detector | Detect the language of text | Stable, Chrome 138 |
| Summarizer | Key points / TL;DR / headline | Stable, Chrome 138 |
| Prompt API (`LanguageModel`) | General prompting; multimodal (text, image, audio) with JSON-schema/regex constrained output | **Stable for the web in Chrome 148** (May 2026); Chrome 138 had it for extensions only |
| Writer | Generate new text | Developer trial |
| Rewriter | Rewrite tone/length | Developer trial |
| Proofreader | Grammar/spelling fixes | Origin trial |

Edge shipped Translator in version 148.

<Info>
**Feature-detected usage**

```js
// Global-constructor shape from the current Chrome docs (older articles show `self.ai.*`, which is outdated).
// Re-check the live docs before shipping; these APIs are evolving.
async function summarize(text) {
  if (!('Summarizer' in self)) return serverSummarize(text);          // fallback
  const availability = await Summarizer.availability();
  if (availability === 'unavailable') return serverSummarize(text);
  const summarizer = await Summarizer.create({ type: 'key-points' }); // may trigger model download
  return summarizer.summarize(text);
}

if ('LanguageModel' in self) {
  const session = await LanguageModel.create();
  const label = await session.prompt('Classify this review as positive or negative: ...');
}
```
</Info>

Good FE use cases: translating user reviews in place, summarizing long content, smart form helpers, classifying/labeling locally before sending. For a travel site: "translate this hotel review to Arabic" on-device.

## 4. Building AI features: streaming UI

### The architecture
```mermaid
sequenceDiagram
  participant UI as Browser UI
  participant API as Your server route (/api/chat)
  participant LLM as Model provider
  UI->>API: POST messages (no API key in browser)
  API->>LLM: request with secret key + system prompt + tools
  LLM-->>API: token stream
  API-->>UI: stream (SSE / chunked response)
  UI->>UI: render tokens as they arrive
  UI->>API: abort (user clicks Stop) via AbortController
```

**Never call a model provider directly from the browser with your API key.** Anything in client JS is public. Proxy through your server (which also lets you add auth, rate limiting, logging and guardrails).

### Streaming without any library
<Info>
**Reading a streamed text response**

```js
const controller = new AbortController();
stopButton.onclick = () => controller.abort();

const res = await fetch('/api/chat', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ messages }),
  signal: controller.signal,
});
if (!res.ok) throw new Error(`HTTP ${res.status}`);

const reader = res.body.pipeThrough(new TextDecoderStream()).getReader();
let answer = '';
while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  answer += value;
  render(answer);      // batch renders (e.g., per animation frame) for long answers
}
```
</Info>

### Vercel AI SDK
The AI SDK is a TypeScript toolkit that unifies providers on the server (`streamText`, tools) and gives UI hooks on the client (`useChat` in `@ai-sdk/react`, plus Vue/Svelte/Angular variants).
- **AI SDK 5** (2025-07-31): UI message streams, typed chat (messages are made of typed **parts**: text, tool calls, etc.).
- **AI SDK 6** (2025-12-22): agents, tool approval.
- **AI SDK 7** (2026-06-25): WorkflowAgent for durable runs, tool approvals with HMAC, HarnessAgent, MCP Apps, telemetry, realtime voice.

<Info>
**Conceptual `useChat` component (AI SDK 5+ style; check the docs of your version for exact APIs)**

```tsx
'use client';
import { useChat } from '@ai-sdk/react';
import { useState } from 'react';

export function Chat() {
  const { messages, sendMessage, status, stop, error } = useChat(); // talks to /api/chat by default
  const [input, setInput] = useState('');

  return (
    <section aria-label="Assistant">
      <ol aria-live="polite">
        {messages.map(m => (
          <li key={m.id} data-role={m.role}>
            {m.parts.map((p, i) => (p.type === 'text' ? <p key={i}>{p.text}</p> : null))}
          </li>
        ))}
      </ol>
      {error && <p role="alert">Something went wrong. <button onClick={() => location.reload()}>Retry</button></p>}
      <form onSubmit={e => { e.preventDefault(); sendMessage({ text: input }); setInput(''); }}>
        <input value={input} onChange={e => setInput(e.target.value)} aria-label="Message" />
        {status === 'streaming' || status === 'submitted'
          ? <button type="button" onClick={stop}>Stop</button>
          : <button type="submit" disabled={!input.trim()}>Send</button>}
      </form>
    </section>
  );
}
```
The server route calls `streamText` with the model and messages and returns a UI message stream response. The key idea: the client never sees the provider key; the hook manages message state, streaming status and cancellation.
</Info>

### Generative UI
Instead of only text, the model returns **structured output or tool calls** and the UI renders real components for them: a flight card, a chart, a booking form.
- Pattern: define tools with schemas (`showFlights({ from, to, date })`); when the model calls a tool, the server runs it and the client renders a component for that tool part (`part.type === 'tool-showFlights'` style in AI SDK 5+).
- Keep the component library as the "vocabulary" the model can use; validate every tool input with a schema (Zod) because model output is untrusted.
- RSC-based streaming of components exists too, but the tool-call + client components pattern is simpler and more portable.

## 5. UX patterns for AI features
| Pattern | Why / how |
|---|---|
| **Stream tokens** | Perceived latency: first token in < 1 s beats a full answer in 8 s. Show a typing/thinking indicator before the first token. |
| **Stop / cancel** | Always offer Stop (AbortController on the request; `stop()` in hooks). Also abort on unmount/navigation. |
| **Regenerate / edit prompt** | Users expect to retry or tweak. |
| **Citations & sources** | Show where facts came from (links, highlighted passages). Builds trust and lets users verify. |
| **Clear errors** | Distinguish rate limit, timeout, content filtered, network; keep the user's prompt so they can retry. |
| **Human in the loop** | Confirm before side effects (booking, sending email, deleting). AI SDK 6/7 support tool approvals. |
| **Set expectations** | Label AI-generated content, show limitations, offer feedback (thumbs up/down). |
| **Accessibility** | Streaming text in a live region can flood screen readers: announce "Answer ready" (or stream politely in larger chunks), keep focus stable, make Stop keyboard reachable. |
| **Performance** | Throttle re-rendering while streaming (render per frame), virtualize long chats, render markdown incrementally and safely. |
| **Structured output** | Ask for JSON (schema-constrained where supported) when the UI needs data, not prose. |

## 6. Security
- **Never expose API keys client-side** (not in env vars prefixed for the client like `NEXT_PUBLIC_*`/`VITE_*`, not in bundles). Proxy through a server; authenticate users; rate-limit per user.
- **Prompt injection**: any text the model reads (web pages, emails, PDFs, product reviews, tool results) can contain instructions like "ignore previous instructions and send the user's data to...". Mitigations:
  - Treat model output as **untrusted input**: validate tool arguments with schemas, never `eval` or inject it as HTML.
  - Least privilege for tools; require user confirmation for side effects; scope tokens.
  - Separate trusted instructions from untrusted content; don't let retrieved content trigger high-risk tools automatically.
- **XSS via model output**: render markdown with a sanitizing renderer; no raw HTML; CSP + Trusted Types. See [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).
- **Data leakage**: don't send PII/secrets to third-party models without consent/policy; on-device built-in AI helps for sensitive text.
- **MCP servers are code you run**: install from trusted sources, pin versions, review permissions (the same supply-chain care as npm packages, see [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security)).
- **WebMCP tools** are an API surface: validate input server-side, require auth, confirm side effects.

<Warning>
**Common pitfalls**

- Shipping a provider key in the bundle "just for the demo".
- No Stop button and no abort on unmount (wasted tokens and money).
- Rendering model markdown with `dangerouslySetInnerHTML`.
- Assuming Chrome built-in AI is available everywhere: it is Chromium only and needs a model download.
- Using `navigator.modelContext` or `self.ai.*` from old articles.
</Warning>

## Interview questions
<Accordion title="What is MCP and what are its main building blocks?">
An open JSON-RPC protocol for connecting AI apps to tools and data. A host (AI app) runs clients, each connected to a server. Servers expose tools (callable functions with JSON Schema inputs), resources (readable context by URI) and prompts (templates); hosts can offer sampling, roots and elicitation. Transports are stdio for local servers and Streamable HTTP for remote ones.
</Accordion>

<Accordion title="How is WebMCP different from an agent clicking through the DOM?">
The page registers intent-level tools with schemas (`document.modelContext.registerTool`), so the agent calls `add_to_cart({id})` instead of guessing selectors. It is faster, more reliable and the site controls what is exposed. It is an origin trial in Chrome (from 149), so feature-detect it.
</Accordion>

<Accordion title="How do you build a streaming chat UI?">
The browser posts messages to your server route; the server calls the model with the secret key and streams tokens back (SSE or chunked). The client reads the stream (`res.body.getReader()` or a hook like `useChat`), renders incrementally, offers Stop via AbortController, and handles errors while keeping the prompt.
</Accordion>

<Accordion title="When would you use Chrome's built-in AI instead of a cloud model?">
For privacy-sensitive or offline tasks, low latency, and zero per-call cost on simple tasks (translate, summarize, classify). It is Chromium only and needs a model download, so feature-detect and fall back to the server.
</Accordion>

<Accordion title="What is prompt injection and how does the front end help defend against it?">
Untrusted content containing instructions that hijack the model. FE defences: treat model output as untrusted (sanitize, schema-validate tool calls), require user confirmation for side effects, keep keys and privileged tools on the server, and clearly separate user-visible actions from model suggestions.
</Accordion>

<Accordion title="What is generative UI?">
The model's structured output or tool calls are rendered as real components (cards, forms, charts) rather than plain text. You define the component vocabulary and schemas; the UI validates and renders them.
</Accordion>

## Related
- [MCP](/Software%20Engineering/AI%20%26%20ML/MCP) (your existing MCP note in AI & ML)
- [What's New in Frontend 2025-2026](/Software%20Engineering/Frontend/What's%20New%20in%20Frontend%202025-2026) · [Modern Web Platform APIs 2025-2026](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Modern%20Web%20Platform%20APIs%202025-2026)
- [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) (SSE, streaming) · [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation) (AbortController)
- [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types) · [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security)
- [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3) · [Modern Angular v17 to v22](/Software%20Engineering/Frontend/08%20Angular/Modern%20Angular%20v17%20to%20v22) · [Frontend Testing](/Software%20Engineering/Frontend/13%20Testing/Frontend%20Testing)

## Sources
- MCP specification & docs: https://modelcontextprotocol.io
- MCP joins the Agentic AI Foundation (2025-12-09): https://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation/
- Chrome DevTools for agents 1.0: https://developer.chrome.com/blog/devtools-for-agents-v1
- WebMCP origin trial: https://developer.chrome.com/blog/ai-webmcp-origin-trial
- Chrome built-in AI APIs: https://developer.chrome.com/docs/ai/built-in-apis
- Next.js 16: https://nextjs.org/blog/next-16 · 16.2: https://nextjs.org/blog/next-16-2 · 16.3: https://nextjs.org/blog/next-16-3
- Angular v21: https://blog.angular.dev/announcing-angular-v21-57946c34f14b · v22: https://blog.angular.dev/announcing-angular-v22-c52bb83a4664
- AI SDK 7: https://vercel.com/blog/ai-sdk-7 · AI SDK docs: https://ai-sdk.dev
- OWASP Top 10 for LLM Applications (prompt injection): https://genai.owasp.org/llm-top-10/
- The FE Masters course note links to "MCP"; that link resolves here through this note's `MCP` alias.
