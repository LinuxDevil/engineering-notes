---
title: "Networking - HTTP, Caching & Real-time"
tags: [frontend, frontend/browser, frontend/networking]
aliases: [Networking, HTTP Caching, Cache-Control, CORS, WebSockets vs SSE, Long polling, HTTP/2, HTTP/3]
updated: 2026-09-27
---
# Networking - HTTP, Caching & Real-time

<Note>
**TL;DR**

Front end performance is mostly network performance. Know how **HTTP/1.1, HTTP/2 and HTTP/3** differ (connections, multiplexing, head-of-line blocking), how the **HTTP cache** decides whether to reuse a response (`Cache-Control`, `ETag`, `Last-Modified`, `Expires`), how **CORS** and preflight work, how to use `fetch` properly (errors, timeouts, cancellation), and how to pick a **real-time** transport: polling, long-polling, SSE, WebSockets or WebTransport.
</Note>

## 1. Browser networking in one picture

{/* diagram source: /diagrams/diagram-242.mmd */}
![diagram](/diagrams/diagram-242.svg)

- **Latency** (round trips) usually matters more than bandwidth for web pages. A new HTTPS connection over TCP costs ~2–3 round trips before the first request (TCP + TLS 1.2; TLS 1.3 saves one). HTTP/3 over QUIC combines transport and TLS handshakes (1 RTT, 0-RTT on resumption).
- Reuse connections (keep-alive, HTTP/2 multiplexing), move content closer (CDN edge), and warm up origins early (`preconnect`, see [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path)).
- Browsers open ~6 TCP connections per origin for HTTP/1.1; one connection per origin for HTTP/2 and HTTP/3.

## 2. HTTP/1.1 vs HTTP/2 vs HTTP/3

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | **QUIC over UDP** |
| Format | Text | Binary frames | Binary frames |
| Concurrency | One request at a time per connection (pipelining unusable) → ~6 parallel connections | **Multiplexing**: many streams over one connection | Multiplexing with **independent streams** |
| Head-of-line blocking | At HTTP level | Fixed at HTTP level, but a lost TCP packet stalls **all** streams | **Solved**: loss only stalls the affected stream |
| Header compression | None | HPACK | QPACK |
| Handshake | TCP + TLS | TCP + TLS | Combined (1-RTT, 0-RTT resume) |
| Connection migration | No | No | Yes (connection ID survives Wi-Fi → 4G switch) |
| Server push | n/a | Existed, now **removed** from Chrome; use `103 Early Hints` + preload instead | Not used |

Implications for front end:
- **HTTP/1.1-era hacks hurt on HTTP/2+**: domain sharding (extra connections, no shared prioritization), giant concatenated bundles (poor caching granularity), sprites. With HTTP/2 prefer reasonably split, well-cached chunks.
- Don't go to the other extreme: hundreds of tiny modules still add per-request overhead and compression is worse on small files.
- HTTP/3 especially helps mobile/lossy networks (relevant for travellers on airport Wi-Fi).
- `103 Early Hints` lets the server send `Link: rel=preload/preconnect` headers while it's still generating the HTML.

## 3. HTTP caching

To cache responses on the client, the browser looks mainly at four response headers: **`Cache-Control`, `ETag`, `Last-Modified` and `Expires`**. There are two separate questions:
1. **Freshness**: can I reuse the stored response *without asking the server*? (`Cache-Control: max-age`, `Expires`)
2. **Validation**: once stale, is my copy still OK? Ask with a **conditional request** (`ETag` → `If-None-Match`, `Last-Modified` → `If-Modified-Since`); a `304 Not Modified` with an empty body means "yes, reuse it".

{/* diagram source: /diagrams/diagram-243.mmd */}
![diagram](/diagrams/diagram-243.svg)

### ETag (entity tag)
A validator string (often a hash or version) that identifies a specific version of the resource.
```http
HTTP/1.1 200 OK
ETag: "33a64df5"
Cache-Control: no-cache
```
Next time, the browser revalidates:
```http
GET /api/airports HTTP/1.1
If-None-Match: "33a64df5"
```
If the resource hasn't changed, the server answers **`304 Not Modified`** with an **empty body** and the browser reuses its copy. If it changed, it answers `200` with the new body and a new `ETag`. (Weak ETags `W/"..."` mean "semantically equivalent", strong ones mean byte-identical.)

### Last-Modified
Similar to ETag but based on a date:
```http
Last-Modified: Tue, 22 Sep 2026 10:00:00 GMT
```
The browser revalidates with `If-Modified-Since: <that date>`. It has one-second resolution and is less precise than ETag; if both are present, `ETag`/`If-None-Match` takes precedence. Without `max-age`, browsers may apply **heuristic freshness** based on `Last-Modified` (typically 10% of the time since last modification), a common source of "why is this stale?" bugs, so always set `Cache-Control` explicitly.

### Expires
An **absolute** expiry date from HTTP/1.0: `Expires: Wed, 21 Oct 2026 07:28:00 GMT`. It depends on clocks being in sync. If `Cache-Control: max-age` (or `s-maxage`) is present, **`Expires` is ignored**. Prefer `max-age`.

### Cache-Control directives

| Directive | Meaning |
|---|---|
| `max-age=N` | Fresh for N seconds (browser and shared caches). |
| `s-maxage=N` | Overrides `max-age` for **shared caches** (CDN/proxies) only. |
| `public` | Any cache may store it, even for authenticated requests. |
| `private` | Only the **browser** may store it (not CDNs). For user-specific responses. |
| **`no-cache`** | **May be stored, but must be revalidated with the server before every use.** It does *not* mean "don't cache". |
| **`no-store`** | **Must not be stored anywhere.** Use for sensitive data (bank details, boarding passes with PII). |
| `must-revalidate` | Once stale, must not be used without successful revalidation (no serving stale on errors). |
| `immutable` | The resource will never change while fresh, so don't even revalidate on reload. Perfect for hashed filenames. |
| `stale-while-revalidate=N` | After expiry, may serve the stale copy for up to N more seconds while revalidating in the background. |
| `stale-if-error=N` | May serve stale if the origin errors (supported by some CDNs). |

<Info>
**Caching recipe for a typical SPA / Next.js app**

```http
# Hashed static assets: /assets/app.3f9a2c.js, /_next/static/...
Cache-Control: public, max-age=31536000, immutable

# HTML documents: always check, but allow 304s
Cache-Control: no-cache
ETag: "page-v42"

# Public API data that can be a bit stale (airport list)
Cache-Control: public, max-age=60, stale-while-revalidate=600

# Per-user API responses
Cache-Control: private, no-cache

# Sensitive data
Cache-Control: no-store
```
Why this works: HTML is revalidated so it always points to the latest hashed bundle names; bundles themselves can be cached "forever" because a change produces a new filename (**cache busting**).
</Info>

Other cache-related points:
- `Vary: Accept-Encoding, Accept-Language` tells caches to store separate variants per request header. `Vary: Cookie` effectively kills CDN caching.
- Reload behaviour: a normal reload revalidates the main document; hard reload (Ctrl+Shift+R) bypasses the cache.
- Layers of caches: memory cache → **service worker** ([Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs)) → HTTP disk cache → CDN → origin. Plus the **bfcache** (back/forward cache) which keeps whole pages in memory; `Cache-Control: no-store` on the document and `unload` listeners can prevent it.
- `fetch(url, { cache: 'no-store' | 'reload' | 'no-cache' | 'force-cache' })` controls the HTTP cache per request.
- Framework data caches are a separate layer: [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching), [Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query).

## 4. CORS and preflight

The **same-origin policy** stops scripts on `https://flights.example.com` from reading responses from another origin (scheme + host + port). **CORS** lets the *server* opt in.

**Simple requests** (GET/HEAD/POST with only "safe" headers and `Content-Type` of `text/plain`, `multipart/form-data` or `application/x-www-form-urlencoded`) are sent directly; the browser checks the response's `Access-Control-Allow-Origin` before letting JS read it.

Anything else (PUT/PATCH/DELETE, `Content-Type: application/json`, custom headers like `Authorization`) triggers a **preflight**:

{/* diagram source: /diagrams/diagram-244.mmd */}
![diagram](/diagrams/diagram-244.svg)

```http
Access-Control-Allow-Origin: https://flights.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true      # needed for cookies; then Allow-Origin can't be *
Access-Control-Max-Age: 600                 # cache the preflight result
Access-Control-Expose-Headers: X-Request-Id # let JS read non-safelisted response headers
Vary: Origin
```

Key points:
- CORS is enforced **by the browser**, not the server. curl/Postman ignore it. It protects users' data, not your API; still authenticate requests.
- A CORS error means the request may have **reached** the server; the browser just hides the response.
- To send cookies cross-origin: `fetch(url, { credentials: 'include' })` + `Access-Control-Allow-Credentials: true` + an explicit origin + `SameSite=None; Secure` cookies.
- Avoid preflights on hot paths by keeping requests "simple", or cache them with `Access-Control-Max-Age`. A same-origin **BFF/proxy** (e.g. Next.js route handlers, `proxy.ts`) removes CORS entirely.
- `mode: 'no-cors'` gives an **opaque** response you can't read; it doesn't "fix" CORS.

## 5. fetch done properly

```js
async function getJSON(url, { timeout = 8000, signal, ...init } = {}) {
  const res = await fetch(url, {
    ...init,
    headers: { Accept: 'application/json', ...init.headers },
    // combine a caller's signal (e.g. React effect cleanup) with a timeout
    signal: signal ? AbortSignal.any([signal, AbortSignal.timeout(timeout)]) : AbortSignal.timeout(timeout),
  });
  if (!res.ok) {                           // fetch only rejects on NETWORK errors, not 404/500
    const body = await res.text().catch(() => '');
    throw new Error(`HTTP ${res.status}: ${body.slice(0, 200)}`);
  }
  return res.json();
}
```

- `fetch` resolves on any HTTP status; check `res.ok`/`res.status`.
- The body can be consumed **once** (`res.clone()` if you need it twice).
- Cancellation: `AbortController` / `AbortSignal.timeout()` / `AbortSignal.any()`; errors are `AbortError` / `TimeoutError` ([Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation)).
- Streaming responses: `res.body` is a `ReadableStream`; decompress with `DecompressionStream` (Compression Streams are Baseline Widely available, 2025-11-09).
- `keepalive: true` (or `navigator.sendBeacon`) for analytics sent while the page unloads.
- Old APIs: `XMLHttpRequest` still exists (and is the only built-in way to get **upload progress** events), `axios` wraps it.

<Info>
**Race-condition-safe search (typeahead)**

```js
let controller;
input.addEventListener('input', debounce(async (e) => {
  controller?.abort();                  // cancel the previous in-flight request
  controller = new AbortController();
  try {
    const data = await getJSON(`/api/airports?q=${encodeURIComponent(e.target.value)}`, { signal: controller.signal });
    render(data);
  } catch (err) {
    if (err.name !== 'AbortError') showError(err);
  }
}, 250));
```
See [Case Study - Autocomplete](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Autocomplete).
</Info>

## 6. Real-time communication

How do we get updates from server to client ("talking to the server")?

{/* diagram source: /diagrams/diagram-245.mmd */}
![diagram](/diagrams/diagram-245.svg)

| | Short polling | Long polling | SSE (EventSource) | WebSockets | WebTransport |
|---|---|---|---|---|---|
| Direction | Client pulls | Client pulls (server holds) | **Server → client** | **Bidirectional** | Bidirectional, multiple streams + datagrams |
| Transport | HTTP | HTTP | HTTP (one long response) | TCP after HTTP Upgrade (`ws://`, `wss://`) | HTTP/3 (QUIC) |
| Data | Any | Any | UTF-8 text events | Text or binary | Binary streams, unreliable datagrams |
| Reconnect | n/a | Manual | **Built-in** (`retry`, `Last-Event-ID`) | Manual (heartbeats, backoff) | Manual |
| Latency | Up to the interval | Low | Low | Lowest | Lowest, no head-of-line blocking |
| Infra | Trivial | Easy, but many open requests | Easy, works with HTTP/2 proxies/CDNs | Needs sticky/stateful servers, LB support | Needs HTTP/3 server |
| Good for | Rare updates, simple dashboards | Legacy/fallback | Notifications, live prices, feeds, **LLM token streaming** | Chat, collaboration, multiplayer, trading | Games, media, high-frequency low-latency data |

<Info>
**Long polling**

```js
async function poll(cursor) {
  try {
    const res = await fetch(`/api/updates?cursor=${cursor}`, { signal: AbortSignal.timeout(35_000) });
    if (res.status === 204) return poll(cursor);          // server timeout, nothing new
    const { events, next } = await res.json();
    events.forEach(apply);
    poll(next);
  } catch {
    setTimeout(() => poll(cursor), 2000);                 // back off on error
  }
}
```
</Info>

<Info>
**Server-Sent Events: live fare updates**

```js
const es = new EventSource('/api/fares/stream?route=DXB-LHR', { withCredentials: true });
es.addEventListener('price', (e) => updatePrice(JSON.parse(e.data)));
es.onerror = () => { /* the browser reconnects automatically */ };
// es.close();
```
```text
# Server response (Content-Type: text/event-stream)
event: price
id: 1042
data: {"flight":"EK1","amount":812}

retry: 5000
```
`EventSource` can't send custom headers (e.g. `Authorization`); use cookies, or read a streamed `fetch()` body yourself (which is how most AI chat UIs stream tokens). With HTTP/1.1, each SSE connection uses one of the ~6 per-origin connections; HTTP/2 removes that limit.
</Info>

<Info>
**WebSocket with reconnect + heartbeat**

```js
function connect(url, onMessage, attempt = 0) {
  const ws = new WebSocket(url);
  let heartbeat;
  ws.onopen = () => {
    attempt = 0;
    ws.send(JSON.stringify({ type: 'subscribe', channel: 'booking:42' }));
    heartbeat = setInterval(() => ws.send('{"type":"ping"}'), 25_000);
  };
  ws.onmessage = (e) => onMessage(JSON.parse(e.data));
  ws.onclose = () => {
    clearInterval(heartbeat);
    const delay = Math.min(30_000, 1000 * 2 ** attempt) + Math.random() * 1000; // exponential backoff + jitter
    setTimeout(() => connect(url, onMessage, attempt + 1), delay);
  };
  return ws;
}
```
Notes: messages can be lost while disconnected, so resync state on reconnect (send a last-seen id). WebSocket has no built-in backpressure; `bufferedAmount` tells you how much is queued. Use one socket per app (or a SharedWorker to share across tabs).
</Info>

**WebTransport** (HTTP/3-based) offers multiple independent streams and unreliable datagrams. It isn't Baseline yet; it is one of the **Interop 2026** focus areas, so feature-detect `'WebTransport' in window`.

```js
const wt = new WebTransport('https://game.example.com:4433/session');
await wt.ready;
const writer = wt.datagrams.writable.getWriter();
writer.write(new Uint8Array([1, 2, 3])); // unreliable, unordered: fine for positions
```

Other related APIs: **WebRTC** (peer-to-peer audio/video/data channels, also in Interop 2026), **Beacon API** (`navigator.sendBeacon` for fire-and-forget analytics), **BroadcastChannel** (tab-to-tab, not network).

## 7. REST vs GraphQL (and friends)

- **REST**: resources + HTTP verbs; great HTTP caching, simple; risk of over/under-fetching and many round trips.
- **GraphQL**: one endpoint, client asks for exactly the fields it needs; great for complex UIs aggregating many sources; caching moves into the client (Apollo/urql normalized cache); POST-by-default defeats HTTP caching (use persisted queries + GET).
- **gRPC / tRPC / RPC-style server functions** (React Server Functions, Next.js Server Actions) are other options.
- Full comparison: [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST), [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST).

<Info>
**🆕 2025–2026**

- **Compression Streams**: Baseline **Widely** available (2025-11-09); Firefox 147 added Brotli.
- **Interop 2026** (announced 2026-02-12) includes **fetch uploads and ranges**, **WebTransport** and **WebRTC** as focus areas.
- **Angular v22**: `HttpClient` uses `fetch` by default (see [Modern Angular v17 to v22](/Software%20Engineering/Frontend/08%20Angular/Modern%20Angular%20v17%20to%20v22)).
- **Next.js 16** replaced `middleware.ts` with `proxy.ts` (Node runtime), often where CORS headers and auth redirects now live ([Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3)).
</Info>

## Interview questions

<Accordion title="no-cache vs no-store?">
`no-cache` allows storing but requires revalidation (ETag/Last-Modified → 304) before each use. `no-store` forbids storing at all. Use `no-cache` for HTML, `no-store` for sensitive data.
</Accordion>

<Accordion title="How do ETag and Last-Modified work together with 304?">
The server sends a validator; after expiry the browser sends `If-None-Match` (ETag) or `If-Modified-Since` (date). If unchanged, the server responds `304 Not Modified` with no body and the cached body is reused. ETag wins if both exist.
</Accordion>

<Accordion title="How would you cache a React SPA's assets?">
Content-hashed JS/CSS with `max-age=31536000, immutable`; `index.html` with `no-cache` (+ ETag) so deploys are picked up immediately; APIs per endpoint (`private`, `no-cache`, or short max-age + stale-while-revalidate).
</Accordion>

<Accordion title="What is a CORS preflight and when does it happen?">
An automatic `OPTIONS` request the browser sends before a non-simple cross-origin request (non-GET/POST methods, JSON content type, custom headers) to ask the server which origins/methods/headers are allowed. Cached via `Access-Control-Max-Age`.
</Accordion>

<Accordion title="HTTP/2 vs HTTP/3?">
Both multiplex streams over one connection with header compression. HTTP/2 runs on TCP, so a lost packet blocks all streams (TCP head-of-line blocking). HTTP/3 runs on QUIC (UDP) with per-stream loss recovery, faster handshakes and connection migration.
</Accordion>

<Accordion title="WebSockets vs SSE vs long polling: which for live flight status? For chat?">
Flight status (server → client, infrequent): SSE, simple and auto-reconnecting (long polling as a fallback). Chat (bidirectional, low latency): WebSockets. Polling if updates are rare and simplicity wins.
</Accordion>

<Accordion title="Why doesn't fetch reject on a 500?">
A 500 is a valid HTTP response. `fetch` only rejects on network failures, CORS failures or aborts. Check `res.ok`.
</Accordion>

<Accordion title="How do you prevent race conditions in search-as-you-type?">
Debounce input, abort the previous request with AbortController (or ignore stale responses by request id), and cache results per query.
</Accordion>

## Related
- [How Browsers Render - Critical Rendering Path](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/How%20Browsers%20Render%20-%20Critical%20Rendering%20Path)
- [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs)
- [Browser Storage - Cookies, Web Storage & IndexedDB](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Browser%20Storage%20-%20Cookies%2C%20Web%20Storage%20%26%20IndexedDB)
- [Promises, Async-Await & Cancellation](/Software%20Engineering/Frontend/01%20JavaScript/Promises%2C%20Async-Await%20%26%20Cancellation)
- [Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query)
- [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching)
- [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST)
- [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST)
- [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)
- [Case Study - Autocomplete](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20Autocomplete)
- [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index)

## Sources
- MDN: [HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching), [Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control), [ETag](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag), [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS), [Using Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch), [Server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events), [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API), [WebTransport](https://developer.mozilla.org/en-US/docs/Web/API/WebTransport_API), [Evolution of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Evolution_of_HTTP)
- web.dev: [HTTP cache](https://web.dev/articles/http-cache), [Love your cache (stale-while-revalidate)](https://web.dev/articles/stale-while-revalidate), [Interop 2026](https://web.dev/blog/interop-2026)
- Ilya Grigorik, *High Performance Browser Networking*: [hpbn.co](https://hpbn.co/)
- Baseline data: [webstatus.dev](https://webstatus.dev/)
