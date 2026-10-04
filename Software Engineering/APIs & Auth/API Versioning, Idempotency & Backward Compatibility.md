---
title: "API Versioning, Idempotency & Backward Compatibility"
tags: [apis, apis/design]
aliases: [API Versioning, Idempotency, Idempotency Key, Backward Compatibility, Breaking Changes, Deprecation, Sunset Header]
updated: 2026-09-27
---
# API Versioning, Idempotency & Backward Compatibility

<Note>
**TL;DR**

An API is a promise: once clients depend on it you can **add** but not **take away**. Prefer **additive, backward-compatible** evolution; when you must break, ship a **new version** (URL, header or date-based), announce it with `Deprecation`/`Sunset` headers, and measure usage before removal. **Idempotency** is the other half of reliability: networks fail, clients retry, so every write that can be retried must produce the same effect once, typically via an **`Idempotency-Key`** stored with the result.
</Note>

## Part 1: Backward compatibility

### What is a breaking change?

| Usually safe (additive) | Breaking |
|---|---|
| Add a new endpoint | Remove or rename an endpoint/field |
| Add an optional request field | Make an optional field required |
| Add a response field | Change a field's type or format (`"42"` → `42`) |
| Add a new enum value *if clients were told to expect unknowns* | Change the meaning of a field or default behaviour |
| Relax validation | Tighten validation |
| Add an optional query parameter | Change status codes or error `type`s clients rely on |
| | Change pagination, auth or rate-limit semantics |

<Note>
**Hyrum's Law**

"With a sufficient number of users of an API, all observable behaviours of your system will be depended on by somebody." Even field order, timing or error text can break someone. Document what is contractual.
</Note>

### The tolerant reader

Clients should **ignore unknown fields**, not fail on new enum values, and not depend on ordering. Servers should be **strict in what they emit and liberal in what they accept** (within reason; unknown fields in requests are often better rejected to catch typos).

### Versioning strategies

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| URL path | `/v1/orders` | obvious, easy routing & caching | "version the whole API" for one change; URLs change |
| Header / media type | `Accept: application/vnd.acme.v2+json` or `Api-Version: 2` | clean URLs, per-resource | less visible, harder to test in a browser, needs `Vary` |
| Query param | `?api-version=2024-10-01` (Azure) | explicit, easy | pollutes URLs/caches |
| **Date-based pinning** | `Stripe-Version: 2026-08-27` | small frequent changes; each account pinned to a version; server translates old ↔ new | needs a translation layer ("version change modules") |
| No versioning, evolve only (GraphQL style) | `@deprecated` fields | no forks | requires discipline and usage tracking |

**How Stripe-style date versioning works**: the core code always speaks the latest version; a chain of small transformation modules converts requests/responses back to whatever version the client is pinned to. New accounts get the latest; upgrades are explicit and per-account.

For gRPC/protobuf the rules are field-number based: see [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST).

### Deprecation and sunset

```http
HTTP/1.1 200 OK
Deprecation: @1767225600
Sunset: Wed, 30 Sep 2026 23:59:59 GMT
Link: <https://developer.example.com/migrate-v2>; rel="deprecation"; type="text/html"
```

- `Deprecation` (RFC 9745, March 2025): the resource is or will be deprecated; value is a Structured Field date (`@` + Unix seconds).
- `Sunset` (RFC 8594): when it will stop working.
- Process: announce → headers + docs + emails → **measure usage per client** (logs by API key) → brownouts (planned short outages) → remove.

## Part 2: Idempotency

### Why

{/* diagram source: /diagrams/diagram-027.mmd */}
![diagram](/diagrams/diagram-027.svg)

`GET`, `PUT`, `DELETE` are idempotent by definition; `POST` and `PATCH` are not. Retries happen everywhere: client libraries, mobile apps on flaky networks, gateways, queues with at-least-once delivery ([Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues)), webhook senders.

### The Idempotency-Key pattern

1. Client generates a unique key (UUID) **per logical operation** and sends it with every retry.
2. Server, atomically: if the key is new, record it as `in_progress` with a hash of the request; process; store the response.
3. Retry with the same key: return the **stored response** (same status and body), don't re-execute.
4. Same key + **different payload** → `422` (misuse). Same key while the first is still running → `409 Conflict`.
5. Expire keys after a window (Stripe: 24 hours).

```sql
CREATE TABLE idempotency_keys (
  key            text        NOT NULL,
  account_id     bigint      NOT NULL,
  request_hash   text        NOT NULL,
  status         text        NOT NULL,        -- in_progress | completed
  response_code  int,
  response_body  jsonb,
  created_at     timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (account_id, key)              -- scope keys per client
);
```

```ts
async function createPayment(req, res) {
  const key = req.get('Idempotency-Key');
  if (!key) return problem(res, 400, 'missing-idempotency-key');
  const hash = sha256(JSON.stringify(req.body));

  const existing = await db.tx(async (t) => {
    const inserted = await t.oneOrNone(
      `INSERT INTO idempotency_keys (key, account_id, request_hash, status)
       VALUES ($1, $2, $3, 'in_progress')
       ON CONFLICT DO NOTHING RETURNING key`, [key, req.user.accountId, hash]);
    if (inserted) return null;                              // we own it
    return t.one(`SELECT * FROM idempotency_keys WHERE key=$1 AND account_id=$2`,
                 [key, req.user.accountId]);
  });

  if (existing) {
    if (existing.request_hash !== hash) return problem(res, 422, 'idempotency-key-reused');
    if (existing.status === 'in_progress') return problem(res, 409, 'request-in-progress');
    return res.status(existing.response_code).json(existing.response_body); // replay
  }

  const payment = await payments.charge(req.body, { idempotencyKey: key }); // pass it downstream too
  await db.none(`UPDATE idempotency_keys SET status='completed', response_code=201, response_body=$3
                 WHERE key=$1 AND account_id=$2`, [key, req.user.accountId, payment]);
  res.status(201).json(payment);
}
```

Subtleties:
- The side effect and the key's completion should be in the **same transaction** where possible; if the side effect is external (a card network), pass the key downstream so *it* is idempotent too.
- Don't cache `5xx` responses caused by transient failures forever; Stripe re-runs requests that failed before any side effect.
- Other techniques: **natural idempotency** (`PUT /orders/{client-generated-id}`), unique constraints (`UNIQUE(order_id)` on shipments), conditional updates (`UPDATE … WHERE status='pending'`), and deduplication tables for message consumers (the **inbox** pattern).

<Info>
**🆕 2025–2026**

- **2025-03:** RFC 9745 standardises the **`Deprecation`** header (with the `deprecation` link relation), complementing RFC 8594 `Sunset`.
- **2025-10-15:** `draft-ietf-httpapi-idempotency-key-header-07` published. `Idempotency-Key` is still an **Internet-Draft**, not an RFC, but widely implemented (Stripe, Adyen, PayPal-style APIs).
- **2025-09:** OpenAPI 3.2 lets you describe such headers and deprecations more precisely (see [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design)).
</Info>

## Pitfalls

<Warning>
**Common mistakes**

- Versioning every tiny change (v1…v17) instead of evolving additively.
- "We'll just tell clients": no usage metrics per client, so you never know when it's safe to remove.
- Idempotency key stored **after** the side effect, so two concurrent retries both execute.
- Keys scoped globally rather than per account (one client can collide with another's key).
- Treating retries of non-idempotent calls as safe in gateways/service meshes (automatic retry of POST).
- Enums: adding a value that old clients `switch` on without a default.
</Warning>

## Senior perspective

- Define a written **compatibility policy** (what counts as breaking, deprecation windows like 12 months, support for N-1 versions) and publish it.
- Put **breaking-change detection in CI** (oasdiff for OpenAPI, `buf breaking` for protobuf, GraphQL schema checks).
- Version **lifecycles cost money**: each live version multiplies tests and code paths. Date-based pinning with translation layers keeps one core implementation.
- Idempotency is an **end-to-end** property: client key → API → DB constraint → downstream provider → message consumers. Ask "what happens if this runs twice?" at every hop.
- Review questions: *Is this change additive? Which clients call this field (show me the metrics)? What happens if the client retries this POST after a timeout? Is the idempotency record written atomically with the effect?*

## Interview questions

<Accordion title="How would you version a public REST API?">
Prefer additive evolution and a tolerant-reader contract so most changes need no version. For breaking changes pick one scheme and stay consistent: URL major versions (`/v2`) are simplest; date-based version headers pinned per account (Stripe) allow frequent small changes with one codebase. Announce with Deprecation/Sunset headers, track usage per client, and remove only when usage is gone.
</Accordion>

<Accordion title="What is idempotency and which HTTP methods are idempotent?">
An operation is idempotent if doing it multiple times has the same effect as doing it once. GET, HEAD, OPTIONS, PUT and DELETE are idempotent by definition; POST and PATCH are not.
</Accordion>

<Accordion title="Design an idempotency-key mechanism for a payments API.">
Client sends a UUID `Idempotency-Key` per operation. Server atomically inserts the key (scoped per account) with a request hash and in-progress status; if it already exists, compares the hash (422 on mismatch), returns 409 if still in progress, or replays the stored response. After processing, stores status and response. Keys expire after ~24h. Pass the key downstream so external calls are idempotent too.
</Accordion>

<Accordion title="Is adding a field to a response a breaking change?">
Normally no, if clients are tolerant readers. It can break strict deserialisers or clients relying on exact shape, which is why you document the tolerant-reader expectation. Removing or renaming a field, changing its type or meaning is breaking.
</Accordion>

<Accordion title="What are the Deprecation and Sunset headers?">
Response headers that tell clients a resource is deprecated (RFC 9745, 2025) and when it will stop working (RFC 8594), usually with a `Link` to migration docs. They let tooling detect deprecated usage automatically.
</Accordion>

## Related
- [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) · [HTTP Headers vs Body - Where Data Belongs](/Software%20Engineering/APIs%20%26%20Auth/HTTP%20Headers%20vs%20Body%20-%20Where%20Data%20Belongs) · [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST) · [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST) · [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets) · [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues)
- Other domains: [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns) · [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index)

## Sources
- [draft-ietf-httpapi-idempotency-key-header (datatracker)](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- [Stripe: Idempotent requests](https://docs.stripe.com/api/idempotent_requests) · [Stripe blog: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)
- [Stripe blog: APIs as infrastructure: future-proofing Stripe with versioning](https://stripe.com/blog/api-versioning)
- [RFC 9745: The Deprecation HTTP Response Header Field (Mar 2025)](https://www.rfc-editor.org/info/rfc9745/) · [RFC 8594: The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594.html)
- [Hyrum's Law](https://www.hyrumslaw.com/)
- [Martin Fowler: TolerantReader](https://martinfowler.com/bliki/TolerantReader.html)
- [oasdiff](https://github.com/oasdiff/oasdiff)
