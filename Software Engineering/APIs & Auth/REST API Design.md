---
tags: [apis, apis/rest]
aliases: [REST API, RESTful API Design, API Design, Pagination, Problem Details, RFC 9457, OpenAPI, HATEOAS, HTTP Status Codes]
updated: 2026-09-27
---
# REST API Design

> [!tldr] TL;DR
> A good REST API models **resources as nouns** (`/orders/42`), uses **HTTP methods for their defined semantics** (safe, idempotent), returns **precise status codes**, pages with **cursors** for large or changing lists, reports errors as **RFC 9457 Problem Details**, and is described by an **OpenAPI** document that drives docs, codegen, mocks and contract tests. Consistency across endpoints matters more than purity.

## Resources and URLs

- Use **plural nouns** and hierarchy only for real ownership: `/customers/7/orders`, then `/orders/42` (don't nest deeper than one level).
- Use lowercase, hyphens (`/shipping-addresses`), no file extensions, no verbs.
- Actions that don't fit CRUD: model them as a sub-resource or a state change (`POST /orders/42/cancellation` or `PATCH /orders/42 {"status":"cancelled"}`), or accept a pragmatic RPC-style `POST /orders/42:cancel` (Google AIP style).
- IDs: opaque strings (UUIDv7, ULID, prefixed IDs like `ord_2x8...`) rather than auto-increment integers, which leak volume and invite enumeration.

## Methods

| Method | Use | Safe | Idempotent | Typical success |
|---|---|---|---|---|
| `GET` | read | ✅ | ✅ | 200, 304 |
| `HEAD` | headers only | ✅ | ✅ | 200 |
| `POST` | create / non-idempotent action | ❌ | ❌ (use `Idempotency-Key`) | 201 + `Location`, 202, 200 |
| `PUT` | full replace (or create at a known URL) | ❌ | ✅ | 200, 204, 201 |
| `PATCH` | partial update (JSON Merge Patch RFC 7396 / JSON Patch RFC 6902) | ❌ | not necessarily | 200, 204 |
| `DELETE` | remove | ❌ | ✅ | 204, 202 |
| `QUERY` | complex read with a body (RFC 10008, June 2026) | ✅ | ✅ | 200 |

- **Safe** = no server state change intended. **Idempotent** = repeating it has the same effect as once. These properties are what let clients, proxies and gateways **retry** safely.

## Status codes that matter

| Code | When |
|---|---|
| 200 OK | success with body |
| 201 Created | resource created; return `Location` and usually the body |
| 202 Accepted | work queued ([[Background Jobs & Task Queues]]); return a status URL |
| 204 No Content | success, no body |
| 304 Not Modified | conditional GET matched `ETag` |
| 400 Bad Request | malformed syntax |
| 401 Unauthorized | not authenticated (send `WWW-Authenticate`) |
| 403 Forbidden | authenticated, not allowed |
| 404 Not Found | doesn't exist (or you may not know it exists) |
| 405 Method Not Allowed | send `Allow` |
| 409 Conflict | state conflict (duplicate, version clash, idempotency key in flight) |
| 412 Precondition Failed | `If-Match` ETag didn't match (optimistic locking) |
| 415 Unsupported Media Type | wrong `Content-Type` |
| 422 Unprocessable Content | syntactically fine, semantically invalid (validation) |
| 429 Too Many Requests | rate limited; send `Retry-After` ([[Rate Limiting & Throttling]]) |
| 500 / 502 / 503 / 504 | server bug / bad upstream / overloaded or maintenance (+`Retry-After`) / upstream timeout |

## Pagination

| | Offset/limit | Cursor (keyset) |
|---|---|---|
| Request | `?limit=20&offset=40` | `?limit=20&after=eyJpZCI6MTAwfQ` |
| SQL | `ORDER BY id LIMIT 20 OFFSET 40` | `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 21` |
| Jump to page N | ✅ | ❌ (next/prev only) |
| Performance on deep pages | degrades: DB scans and discards offset rows | constant, uses the index |
| Stable when rows are inserted/deleted | ❌ duplicates or skips | ✅ |
| Total count | easy but expensive (`COUNT(*)`) | usually omitted or approximate |
| Good for | small admin tables | feeds, large/infinite lists, public APIs |

Worked cursor example:

```sql
-- index: CREATE INDEX ON orders (customer_id, created_at DESC, id DESC);
SELECT id, created_at, total
FROM orders
WHERE customer_id = $1
  AND (created_at, id) < ($2, $3)      -- values decoded from the cursor
ORDER BY created_at DESC, id DESC
LIMIT 21;                               -- fetch one extra to know if there's a next page
```

```json
{
  "data": [ { "id": "ord_9", "created_at": "2026-09-20T10:00:00Z", "total": 4200 } ],
  "page": { "next_cursor": "eyJjIjoiMjAyNi0wOS0yMFQxMDowMDowMFoiLCJpIjoib3JkXzkifQ", "has_more": true }
}
```

Make cursors **opaque** (base64 of the sort key, optionally signed) so clients don't build them and you can change the implementation. Also offer RFC 8288 `Link: <…>; rel="next"` headers if you like.

## Filtering, sorting, sparse fields

```http
GET /orders?status=paid&created_at[gte]=2026-01-01&sort=-created_at,id&fields=id,total&limit=50
```

- Allow-list filterable and sortable fields (and index them). Never pass sort fields straight into SQL (injection, unindexed scans).
- Cap `limit` (e.g. max 100) and set a default.
- For very complex search, use `POST /orders/search` or the new `QUERY` method.

## Errors: RFC 9457 Problem Details

RFC 9457 (July 2023, obsoletes RFC 7807) defines `application/problem+json`:

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Your request is not valid.",
  "status": 422,
  "detail": "2 fields failed validation.",
  "instance": "/orders/requests/7f3c",
  "errors": [
    { "pointer": "#/items/0/qty", "detail": "must be ≥ 1" },
    { "pointer": "#/currency", "detail": "must be ISO 4217" }
  ],
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

- `type` (a URI identifying the problem kind, ideally documented), `title`, `status`, `detail`, `instance`, plus **extension members** (`errors`, `trace_id`, `balance`…).
- Never leak stack traces or SQL errors. Include a correlation ID ([[HTTP Headers vs Body - Where Data Belongs]]).
- Clients should branch on `type`, not on `detail` text.

## Other design essentials

- **Conditional requests & concurrency**: return `ETag`; require `If-Match` on updates to prevent lost updates (→ 412).
- **Long-running operations**: `202 Accepted` + `Location: /operations/abc`; client polls or gets a webhook ([[Webhooks, Polling, SSE & WebSockets]]).
- **Bulk endpoints**: define partial-failure semantics (per-item results, or 207-like body).
- **Dates**: ISO 8601/RFC 3339 in UTC. **Money**: integer minor units + currency. **Enums**: document that new values may appear.
- **Naming**: pick `snake_case` or `camelCase` and never mix.
- **Idempotency & versioning**: [[API Versioning, Idempotency & Backward Compatibility]].
- **Security**: [[API Security]].

## HATEOAS (and the Richardson Maturity Model)

**Hypermedia as the Engine of Application State**: responses contain links telling the client what it can do next, so clients follow links instead of hard-coding URLs.

```json
{
  "id": "ord_42", "status": "pending",
  "_links": {
    "self":   { "href": "/orders/ord_42" },
    "pay":    { "href": "/orders/ord_42/payment", "method": "POST" },
    "cancel": { "href": "/orders/ord_42/cancellation", "method": "POST" }
  }
}
```

Richardson Maturity Model: **Level 0** one endpoint/POST everything (SOAP-ish) → **Level 1** resources → **Level 2** HTTP verbs + status codes (where most "REST" APIs live) → **Level 3** hypermedia. In practice full HATEOAS is rare; links for pagination and available actions are the useful parts (formats: HAL, JSON:API, Siren).

## OpenAPI: the contract

```yaml
openapi: 3.1.1
info: { title: Orders API, version: 2026-09-01 }
paths:
  /orders/{id}:
    get:
      operationId: getOrder
      parameters:
        - { name: id, in: path, required: true, schema: { type: string } }
      responses:
        '200':
          description: The order
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '404':
          description: Not found
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }
components:
  schemas:
    Order:
      type: object
      required: [id, total, currency]
      properties:
        id: { type: string }
        total: { type: integer, description: Minor units }
        currency: { type: string, pattern: '^[A-Z]{3}$' }
        note: { type: [string, 'null'] }     # 3.1 = full JSON Schema 2020-12
```

- **OpenAPI 3.1** (Feb 2021) aligned schemas with **JSON Schema 2020-12** (e.g. `type: [string, "null"]` replaces `nullable`), and added top-level `webhooks`.
- **Design-first vs code-first**: design-first (write the spec, review it, generate server stubs/clients/mocks) catches design issues earlier; code-first (generate the spec from annotations) keeps it in sync. Either way, lint it (Spectral) and run breaking-change checks (oasdiff) in CI.

> [!info]+ 🆕 2025–2026
> - **2025-09 (3.2.0 released 19 Sept 2025, announced 23 Sept):** **OpenAPI 3.2** is a backward-compatible feature release: hierarchical/structured **tags** (`parent`, `kind`), first-class **streaming** media types (SSE `text/event-stream`, JSON Lines, multipart), support for **arbitrary HTTP methods** (e.g. `QUERY`), OAuth 2.0 **device authorization flow** and security metadata URLs, and clearer example/serialization semantics. 3.1 documents keep working.
> - **2026-06:** RFC 10008 standardises the **`QUERY`** method (safe, idempotent, with a body), which OpenAPI 3.2 can already describe.
> - **2025-03:** RFC 9745 `Deprecation` header for signalling endpoint deprecation.
> - APIs are increasingly consumed by **AI agents**: good `operationId`s, descriptions and examples in OpenAPI are what tools use to generate MCP tools (see [[AI in Frontend - MCP, WebMCP & Built-in AI]]).

## Pitfalls

> [!warning] Common mistakes
> - `200 OK` with `{"error": "..."}` in the body.
> - Verbs in URLs (`/getOrders`, `/createUser`) mixed with resources.
> - Offset pagination on huge, fast-changing tables.
> - Unbounded list endpoints (no default/max `limit`).
> - Leaking internal models 1:1 (DB columns, internal flags) instead of designing representations.
> - Inconsistent naming, date formats and error shapes across teams.
> - Non-idempotent `PUT`, or `GET` with side effects (crawlers and prefetchers will trigger them).

## Senior perspective

- Write an **API style guide** (or adopt Google AIP, Zalando, Microsoft REST guidelines) and enforce it with a linter in CI.
- Treat the API as a **product**: docs, examples, SDKs, changelog, deprecation policy, SLAs.
- Design for **evolution**: additive changes, tolerant readers, opaque cursors and IDs.
- Watch for **chatty** designs that force N calls per screen; consider a BFF or GraphQL.
- Review questions: *Is every list paginated and bounded? Which endpoints are retry-safe? What does a client see on partial failure? Is the error `type` documented? What is the P99 of the worst query behind this endpoint?*

## Interview questions

> [!question]- What makes an HTTP method safe or idempotent, and why does it matter?
> Safe methods (GET, HEAD, OPTIONS, QUERY) don't intend to change state. Idempotent methods (safe ones plus PUT and DELETE) have the same effect whether called once or many times. This lets clients and proxies retry after network failures without duplicating side effects. POST and PATCH are not idempotent unless you add an idempotency key.

> [!question]- Cursor vs offset pagination?
> Offset is simple and allows page jumps but gets slower with depth and returns duplicates or gaps when data changes. Cursor (keyset) pagination filters by the last seen sort key, uses the index for constant-time pages and is stable under inserts, but only supports next/previous. Use cursors for feeds and large public lists.

> [!question]- How should an API return errors?
> Correct HTTP status plus a consistent machine-readable body, ideally RFC 9457 Problem Details (`type`, `title`, `status`, `detail`, `instance`, plus extensions like field errors and a trace ID). No stack traces.

> [!question]- PUT vs PATCH?
> PUT replaces the whole resource with the representation sent and is idempotent. PATCH applies a partial modification described by a patch document (JSON Merge Patch or JSON Patch) and isn't guaranteed idempotent.

> [!question]- What is HATEOAS and is it worth it?
> Responses include hypermedia links describing available next actions, so clients navigate by links rather than hard-coded URLs (Richardson level 3). It decouples clients from URL structure, but few clients use it fully; pagination links and action links are the pragmatic subset.

> [!question]- What's new in OpenAPI 3.2?
> Released September 2025: structured/hierarchical tags, streaming media types (SSE, JSON Lines), arbitrary HTTP methods like QUERY, OAuth device flow and metadata support, and clearer examples. It is backward-compatible with 3.1.

> [!question]- How do you prevent lost updates when two clients edit the same resource?
> Optimistic concurrency: return an `ETag` (version), require `If-Match` on PUT/PATCH, and reply 412 Precondition Failed if the version changed, so the client re-reads and retries.

## Related
- [[APIs & Auth Index]] · [[Backend Index]]
- [[HTTP Headers vs Body - Where Data Belongs]] · [[API Versioning, Idempotency & Backward Compatibility]] · [[Rate Limiting & Throttling]] · [[Webhooks, Polling, SSE & WebSockets]] · [[API Security]]
- [[GraphQL vs REST]] · [[gRPC vs REST]] · [[API Gateways & Reverse Proxies]]
- Frontend: [[Networking - HTTP, Caching & Real-time]]
- Other domains: [[Keyset Pagination vs Offset]] · [[Caching Strategies]] · [[Unique ID Generation]] · [[System Design Index]]

## Sources
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)
- [RFC 10008: The HTTP QUERY Method (June 2026)](https://datatracker.ietf.org/doc/rfc10008/)
- [OpenAPI Specification v3.2.0](https://spec.openapis.org/oas/v3.2.0.html) · [OpenAPI Initiative: Announcing OpenAPI v3.2 (2025-09-23)](https://www.openapis.org/blog/2025/09/23/announcing-openapi-v3-2)
- [OpenAPI Specification v3.1.1](https://spec.openapis.org/oas/v3.1.1.html)
- [Martin Fowler: Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html)
- [Google API Improvement Proposals (AIP)](https://google.aip.dev/) · [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)
- [Use The Index, Luke: Pagination done the right way](https://use-the-index-luke.com/no-offset)
- [RFC 8288: Web Linking](https://www.rfc-editor.org/rfc/rfc8288.html)
