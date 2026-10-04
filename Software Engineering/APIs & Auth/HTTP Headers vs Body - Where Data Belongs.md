---
title: "HTTP Headers vs Body - Where Data Belongs"
tags: [apis, apis/http]
aliases: [Data by headers, HTTP Headers, Headers vs Body, Custom Headers, Correlation ID, Request ID, traceparent]
updated: 2026-09-27
---
# HTTP Headers vs Body - Where Data Belongs

<Note>
**TL;DR**

Put **metadata about the message** (auth, content type, caching, tracing, idempotency, versioning hints) in **headers**, and put **the business data the handler works on** in the **body** (or path/query for identifying and filtering resources). Headers are read by every hop (proxies, CDNs, gateways, logs), are size-limited and unstructured unless you use RFC 9651 Structured Fields. Don't invent `X-` prefixed headers any more (RFC 6648).
</Note>

## The rule of thumb

It is usually a good idea to use the **headers for metadata** and the **body for the data used by the business logic**.

| Put it in… | When | Examples |
|---|---|---|
| **Path** | it identifies the resource | `/orders/42` |
| **Query string** | it filters, sorts or pages a read; it is safe to log and cache | `?status=paid&limit=50&cursor=abc` |
| **Headers** | it describes the request/response, or infrastructure must read it without parsing the body | `Authorization`, `Content-Type`, `Accept`, `If-None-Match`, `Idempotency-Key`, `traceparent` |
| **Body** | it is the payload the business logic processes, may be large or nested, or is sensitive | the order JSON, file upload |

Points the original note made, still true:
1. Over plain HTTP (not HTTPS), proxies can read and modify headers (and bodies). Use TLS everywhere.
2. Interoperability matters: some clients, SDKs, low-code tools and proxies make custom headers awkward or strip unknown ones. If a value must survive every intermediary, prefer a standard header or the body.
3. Data that routers, load balancers, firewalls and gateways need should be in headers, so they can act **without parsing the body**; keep the body for application-specific information.

## Standard headers a backend engineer should know

| Area | Request | Response |
|---|---|---|
| Content negotiation | `Accept`, `Accept-Encoding`, `Accept-Language`, `Content-Type` | `Content-Type`, `Content-Encoding`, `Vary` |
| Auth | `Authorization: Bearer …`, `Cookie`, `DPoP` | `WWW-Authenticate`, `Set-Cookie` |
| Caching & conditional | `If-None-Match`, `If-Modified-Since`, `If-Match` | `Cache-Control`, `ETag`, `Last-Modified`, `Age` |
| Concurrency control | `If-Match: "v7"` on PUT/PATCH | `412 Precondition Failed` if stale |
| Rate limiting | | `Retry-After`, `RateLimit`, `RateLimit-Policy` (draft) |
| Lifecycle | | `Deprecation` (RFC 9745), `Sunset` (RFC 8594), `Link` |
| Proxies | `Forwarded`, `X-Forwarded-For/-Proto/-Host` | `Via` |
| Tracing | `traceparent`, `tracestate` (W3C Trace Context) | often echoed `X-Request-Id` |
| Security | `Origin` | `Strict-Transport-Security`, `Content-Security-Policy`, CORS `Access-Control-*` |

HTTP caching and CORS are covered from the browser side in [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time).

## Custom headers

- **Don't use the `X-` prefix for new headers.** RFC 6648 (2012) deprecated it: when a header becomes standard, the `X-` name gets stuck forever (`X-Forwarded-For` vs `Forwarded`). Use a clear name, optionally vendor-prefixed: `Acme-Tenant`, `Stripe-Version`.
- Header names are **case-insensitive**; HTTP/2 and HTTP/3 lowercase them on the wire.
- Keep them small. Servers commonly cap total header size around 8–16 KB (Nginx `large_client_header_buffers` defaults to 4 × 8k), and big headers hurt HPACK/QPACK compression.
- For structured values use **RFC 9651 Structured Field Values** (lists, dictionaries, typed items) instead of ad-hoc formats:

```http
Acme-Features: beta, "dark-mode";enabled=?1
RateLimit-Policy: "burst";q=100;w=60
```

## Idempotency keys

A client retrying a `POST /payments` after a timeout could charge twice. The client generates a unique key per logical operation and sends it in a header; the server stores the result and replays it for retries.

```http
POST /v1/payments HTTP/1.1
Idempotency-Key: 8e03978e-40d5-43e8-bc93-6894a57f9324
Content-Type: application/json

{"amount": 5000, "currency": "EUR"}
```

- The key is **metadata about the request**, not business data, so it belongs in a header.
- Standardisation: `draft-ietf-httpapi-idempotency-key-header` (latest -07, Oct 2025, still an Internet-Draft). Stripe popularised the pattern.
- Full server-side implementation: [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility).

## Correlation IDs and distributed tracing

A **correlation ID** (request ID) ties together every log line and span that belongs to one user request as it crosses services.

```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^v  ^------------ trace-id -------------^ ^-- parent span --^ ^flags (sampled)
```

- Prefer the **W3C Trace Context** `traceparent` header (a W3C Recommendation, used by OpenTelemetry) over home-grown `X-Correlation-Id`. Many stacks still add a short `X-Request-Id` for humans.
- The edge (gateway/Nginx) creates it if missing; every service **propagates** it on outgoing calls and message headers (Kafka, queues), and includes it in every log line.

```nginx
# Nginx: generate or pass through a request id
map $http_x_request_id $req_id { default $http_x_request_id; "" $request_id; }
proxy_set_header X-Request-Id $req_id;
add_header X-Request-Id $req_id always;
log_format main '$remote_addr "$request" $status rid=$req_id';
```

```ts
// Node/Express: attach to logs
app.use((req, res, next) => {
  req.id = req.get('x-request-id') ?? crypto.randomUUID();
  res.set('x-request-id', req.id);
  req.log = logger.child({ requestId: req.id });
  next();
});
```

## Pitfalls

<Warning>
**Common mistakes**

- **Secrets in query strings** (`?api_key=…`, `?access_token=…`): they end up in access logs, browser history, `Referer` headers and analytics. Use `Authorization`. OAuth 2.1 forbids bearer tokens in the query.
- **Trusting `X-Forwarded-For` blindly.** Any client can send it. Only trust it from your own proxies (Nginx `set_real_ip_from`, Express `trust proxy`).
- **Business data in headers** (e.g. `Order-Amount: 50`): invisible in most API docs, dropped by some proxies, not validated by body schemas.
- **Forgetting `Vary`** when the response depends on a header (`Accept-Language`, `Authorization`), so a CDN serves the wrong variant.
- **GET with a body**: semantics are undefined and many proxies drop it. For complex read queries use `POST` to a search endpoint, or the new **`QUERY`** method (RFC 10008, June 2026).
</Warning>

<Info>
**🆕 2025–2026**

- **2025-03:** RFC 9745 standardises the **`Deprecation`** response header (pairs with `Sunset`, RFC 8594). See [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility).
- **2025-10:** `Idempotency-Key` header draft reaches -07 (still a draft).
- **2026-05:** `RateLimit` / `RateLimit-Policy` header draft reaches -11 (still a draft). See [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling).
- **2026-06:** RFC 10008 defines the **`QUERY`** method: safe and idempotent like GET, but with a request body; responses are cacheable.
- **2026-07-28:** the MCP specification adds `Mcp-Method` / `Mcp-Name` request headers so gateways can route and meter AI tool calls without parsing JSON bodies, a textbook case of "infrastructure data goes in headers".
</Info>

## Senior perspective

- Decide header conventions **once** for the organisation (request ID, tenant, API version, idempotency) and enforce them at the gateway.
- Strip or overwrite internal headers at the edge (`X-User-Id` set by the gateway must never be accepted from outside, or anyone can impersonate).
- Headers are logged by default in many tools. Redact `Authorization`, `Cookie`, `Set-Cookie`, API keys.
- Remember header size limits across every hop (CDN, LB, Nginx, app server). JWTs stuffed with permissions can blow them up and cause mysterious `431 Request Header Fields Too Large`/`400` errors.

## Interview questions

<Accordion title="When would you put data in a header rather than the body?">
When it is metadata about the message that intermediaries or cross-cutting code must read without parsing the payload: authentication, content negotiation, caching validators, tracing IDs, idempotency keys, rate-limit info. Business data the handler processes goes in the body; resource identity in the path; read filters in the query.
</Accordion>

<Accordion title="Why was the `X-` prefix deprecated?">
RFC 6648 found that experimental `X-` headers often become de facto standards and then can't be renamed, leaving two names forever. New headers should use meaningful names without `X-`, optionally with a vendor prefix.
</Accordion>

<Accordion title="What is a correlation ID and how is it propagated?">
A unique ID for one logical request, created at the edge if absent, passed on every downstream HTTP call and message, and included in all logs and traces. The standard format is the W3C `traceparent` header used by OpenTelemetry.
</Accordion>

<Accordion title="Why should API keys never be in the URL?">
URLs are logged by servers, proxies and CDNs, stored in browser history, leaked via `Referer`, and shared in screenshots. Headers like `Authorization` are not logged by default and can be redacted.
</Accordion>

<Accordion title="What is the new QUERY method for?">
RFC 10008 (June 2026) adds QUERY: a safe, idempotent method with a request body, for complex searches that don't fit in a URL. Unlike POST, it can be retried automatically and cached.
</Accordion>

## Related
- [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) · [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility) · [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling) · [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) · [Nginx](/Software%20Engineering/Backend/Nginx)
- [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)

## Sources
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 6648: Deprecating the "X-" Prefix](https://www.rfc-editor.org/rfc/rfc6648.html)
- [RFC 9651: Structured Field Values for HTTP](https://www.rfc-editor.org/rfc/rfc9651.html)
- [RFC 7239: Forwarded HTTP Extension](https://www.rfc-editor.org/rfc/rfc7239.html)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [draft-ietf-httpapi-idempotency-key-header-07 (Oct 2025)](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- [RFC 9745: The Deprecation HTTP Response Header Field (Mar 2025)](https://www.rfc-editor.org/info/rfc9745/)
- [RFC 10008: The HTTP QUERY Method (Jun 2026)](https://datatracker.ietf.org/doc/rfc10008/)
- [MCP 2026-07-28 specification release post](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [Nginx: large_client_header_buffers](https://nginx.org/en/docs/http/ngx_http_core_module.html#large_client_header_buffers)
