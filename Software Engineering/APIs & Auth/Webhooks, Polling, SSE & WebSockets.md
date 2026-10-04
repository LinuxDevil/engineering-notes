---
title: "Webhooks, Polling, SSE & WebSockets"
tags: [apis, apis/realtime, backend]
aliases: [Webhooks, Polling, Long Polling, SSE, Server-Sent Events, WebSockets, Real-time APIs, Push vs Pull]
updated: 2026-09-27
---
# Webhooks, Polling, SSE & WebSockets

<Note>
**TL;DR**

Four ways for a server to tell someone "something happened": **polling** (client asks repeatedly), **webhooks** (server POSTs to another *server*), **SSE** (server streams events to a client over one long HTTP response) and **WebSockets** (full-duplex socket). From the backend side the hard parts are not the protocols but **delivery guarantees** (retries, signatures, idempotency for webhooks) and **connection state at scale** (fan-out, sticky sessions, pub/sub backplane, proxy timeouts for SSE/WebSockets). This note is the server view; the browser view is in [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time).
</Note>

## Choosing

| | Short polling | Long polling | Webhooks | SSE | WebSockets |
|---|---|---|---|---|---|
| Direction | client → server | client → server (held) | server → server | server → client | both |
| Consumer | any | any | a **server** with a public URL | browsers, apps | browsers, apps |
| Transport | HTTP | HTTP | HTTP POST | HTTP (`text/event-stream`) | HTTP upgrade → WS frames |
| Latency | interval | near real-time | near real-time | real-time | real-time |
| Server cost | many wasted requests | a held request per client | one request per event | one open connection per client | one open connection per client |
| Reconnect / resume | n/a | client re-polls | sender retries | built-in (`Last-Event-ID`) | DIY |
| Typical use | status of a job every 10 s | chat fallback | Stripe/GitHub events to your backend | notifications, dashboards, **LLM token streaming** | chat, collaboration, games |

Rule of thumb: **server-to-server → webhooks**; **server-to-browser, one-way → SSE**; **interactive, bidirectional → WebSockets**; **simple or firewall-hostile → polling**.

{/* diagram source: /diagrams/diagram-034.mmd */}
![diagram](/diagrams/diagram-034.svg)

## Polling (and the async request pattern)

```http
POST /exports          → 202 Accepted, Location: /exports/exp_1
GET  /exports/exp_1    → 200 {"status":"running","progress":0.4}   Retry-After: 5
GET  /exports/exp_1    → 200 {"status":"done","url":"https://…"}
```

- Return `Retry-After` to pace pollers; use `ETag`/`If-None-Match` so unchanged polls cost a 304.
- **Long polling**: the server holds the request until data arrives or ~30 s passes, then the client reconnects. Needs async I/O (thousands of held requests) and proxy timeouts above the hold time.

## Webhooks (server view: sending and receiving)

### Sending webhooks reliably

{/* diagram source: /diagrams/diagram-035.mmd */}
![diagram](/diagrams/diagram-035.svg)

- Use the **transactional outbox** so an event is never lost or sent for a rolled-back change.
- Deliver from **background workers** ([Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues)) with short timeouts (5–10 s), exponential backoff over hours/days, and a dead-letter queue.
- Delivery is **at-least-once** and **unordered**: include a unique event `id`, a timestamp, and a type; tell consumers to dedupe.
- **Sign every payload** and include a timestamp to prevent replay.
- Security for the sender: prevent **SSRF** (customers can register `http://169.254.169.254/…`); resolve and block private IP ranges, use an egress proxy.
- Offer a dashboard: delivery logs, manual replay, secret rotation (support two active secrets during rotation).

The **Standard Webhooks** spec (used by Svix, OpenAI, others) standardises headers:

```http
POST /webhooks/acme HTTP/1.1
Content-Type: application/json
webhook-id: msg_2mNz8vKx
webhook-timestamp: 1790001234
webhook-signature: v1,K5oZfzN95Z9UVu1EsfQmfVNQhnkZ2pj9o9NDN/H/pI4=

{"type":"invoice.paid","id":"evt_91","created":"2026-09-27T10:00:00Z","data":{"invoice_id":"inv_7"}}
```

### Receiving webhooks

```ts
import crypto from 'node:crypto';

app.post('/webhooks/acme', express.raw({ type: 'application/json' }), async (req, res) => {
  const id = req.get('webhook-id')!, ts = req.get('webhook-timestamp')!;
  const sigs = (req.get('webhook-signature') ?? '').split(' ');

  // 1. reject old timestamps (replay protection)
  if (Math.abs(Date.now() / 1000 - Number(ts)) > 300) return res.sendStatus(400);

  // 2. verify HMAC over id.timestamp.rawBody using the RAW body (not re-serialised JSON)
  const secret = Buffer.from(process.env.ACME_WHSEC!.replace('whsec_', ''), 'base64');
  const expected = crypto.createHmac('sha256', secret)
    .update(`${id}.${ts}.${req.body.toString()}`).digest('base64');
  const ok = sigs.some(s => {
    const v = Buffer.from(s.split(',')[1] ?? '', 'base64');
    const e = Buffer.from(expected, 'base64');
    return v.length === e.length && crypto.timingSafeEqual(v, e);   // constant-time compare
  });
  if (!ok) return res.sendStatus(401);

  // 3. dedupe (at-least-once delivery) and 4. ack fast, process async
  const inserted = await db.webhookInbox.insertIfAbsent({ id, body: req.body });
  if (inserted) await queue.add('process-webhook', { id });
  res.sendStatus(200);                   // respond within a few seconds
});
```

- Verify with the **raw bytes**; parsing then re-stringifying JSON changes the signature input.
- **Ack quickly** (2xx) and process in a job; slow handlers cause sender timeouts and duplicate deliveries.
- Don't trust ordering; fetch the latest state from the provider API if order matters ("thin events").

## Server-Sent Events (server view)

```ts
app.get('/events', (req, res) => {
  res.set({
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no',            // tell Nginx not to buffer this response
  });
  res.flushHeaders();

  const lastId = Number(req.get('Last-Event-ID') ?? 0);   // resume after reconnect
  replayMissedEvents(lastId, (e) => res.write(`id: ${e.id}\nevent: ${e.type}\ndata: ${JSON.stringify(e)}\n\n`));

  const unsubscribe = bus.subscribe(req.user.id, (e) =>
    res.write(`id: ${e.id}\nevent: ${e.type}\ndata: ${JSON.stringify(e)}\n\n`));
  const heartbeat = setInterval(() => res.write(': ping\n\n'), 15000);  // keeps proxies from idling out

  req.on('close', () => { clearInterval(heartbeat); unsubscribe(); });
});
```

- Plain HTTP: works through most proxies, CDNs and HTTP/2 (many streams per connection, which removes HTTP/1.1's ~6 connections per origin limit).
- Built-in reconnection and `Last-Event-ID` resume: keep a short event history (Redis Stream) to replay.
- Text only (UTF-8). Great for **LLM token streaming**; many AI APIs stream responses as SSE.

## WebSockets (server view)

- Starts as HTTP `GET` with `Upgrade: websocket` → `101 Switching Protocols`; then framed, full-duplex messages over the same TCP connection.
- **Authenticate at the handshake** (cookie or a short-lived ticket in the URL/first message; browsers can't set custom headers on `new WebSocket()`). Check `Origin` to prevent cross-site WebSocket hijacking.
- Design an **application protocol** on top: message types, acks, request IDs, versioning, heartbeats (ping/pong), backpressure.

```nginx
# Nginx: proxy WebSockets and SSE
location /ws/ {
  proxy_pass http://app;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_read_timeout 3600s;               # default 60s would cut idle sockets
}
location /events {
  proxy_pass http://app;
  proxy_buffering off;                    # SSE must not be buffered
  proxy_read_timeout 3600s;
}
```

### Scaling persistent connections

{/* diagram source: /diagrams/diagram-036.mmd */}
![diagram](/diagrams/diagram-036.svg)

- Each node only knows its own sockets; a **pub/sub backplane** (Redis, NATS, Kafka) fans messages out to the node holding the recipient.
- **Connection count** is the capacity metric (memory per socket, file descriptors, `ulimit`), not RPS.
- Deploys drop every connection: drain gracefully, have clients reconnect with **jittered backoff** to avoid a thundering herd.
- L4/L7 load balancers need long idle timeouts; sticky sessions only if you keep per-connection state server-side.
- Managed options: Ably, Pusher, AWS API Gateway WebSockets, Cloudflare Durable Objects; frameworks: Socket.IO (adds fallbacks, rooms, its own protocol), Phoenix Channels, ActionCable.

<Info>
**🆕 2025–2026**

- **MCP transports**: the 2025-03-26 MCP spec replaced its original HTTP+SSE transport with **Streamable HTTP** (a single endpoint that can reply with JSON or an SSE stream). The **2026-07-28** spec makes the protocol core **stateless** (removes the `initialize` handshake and `Mcp-Session-Id`), adds `Mcp-Method`/`Mcp-Name` routing headers, and formally **deprecates the legacy HTTP+SSE transport** with a one-year transition. See [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI).
- **2025-09:** OpenAPI 3.2 adds first-class **streaming media types** (SSE, JSON Lines), so SSE endpoints can finally be described properly (see [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design)).
</Info>

## Pitfalls

<Warning>
**Common mistakes**

- Webhook receivers doing heavy work before returning 2xx (timeouts → duplicate deliveries).
- Verifying signatures on re-serialised JSON, or comparing with `===` (timing attack).
- No dedupe on the receiver; no replay protection (timestamp window).
- SSE behind a buffering proxy/compression layer: events arrive in big delayed chunks.
- Default proxy/LB idle timeouts (60 s) killing SSE/WebSockets; no heartbeats.
- Keeping per-user state in one WebSocket node's memory without a backplane.
- Sending webhooks synchronously inside the request that triggered them.
</Warning>

## Senior perspective

- Webhooks are a **distributed system you run for your customers**: publish delivery guarantees (at-least-once, retry schedule, ordering), signatures, IP ranges, and a replay tool.
- Consider offering an **events API** (`GET /events?after=cursor`) alongside webhooks so consumers can catch up after downtime.
- For real-time UIs, start with **SSE** unless you truly need client→server streaming; it is simpler to operate and plays well with HTTP infrastructure.
- Capacity plan by concurrent connections and fan-out rate; load test reconnect storms.
- Review questions: *What happens when the consumer is down for 3 days? How do we rotate the signing secret? How does a client resume after reconnect? What's the proxy idle timeout at each hop?*

## Interview questions

<Accordion title="Webhooks vs polling: trade-offs?">
Webhooks push events immediately and avoid wasted requests, but require the consumer to expose a public HTTPS endpoint, verify signatures, handle retries, duplicates and out-of-order delivery. Polling is simple and firewall-friendly but adds latency and load. Many providers offer both, with an events list API for catch-up.
</Accordion>

<Accordion title="How do you make webhook delivery reliable?">
Write events with a transactional outbox, deliver from background workers with timeouts, exponential backoff and jitter over a long window, dead-letter after N attempts, include unique event IDs for dedupe, sign payloads with a timestamp, and provide delivery logs and manual replay.
</Accordion>

<Accordion title="How do you verify a webhook?">
Compute an HMAC (e.g. SHA-256) over the raw body plus ID and timestamp using the shared secret, compare in constant time, reject timestamps outside a small window, and dedupe by event ID.
</Accordion>

<Accordion title="SSE vs WebSockets?">
SSE is one-way server→client over plain HTTP with automatic reconnect and Last-Event-ID resume; simple and proxy-friendly, text only. WebSockets are full-duplex, binary-capable, need an upgrade and an application protocol with your own reconnect logic. Use SSE for notifications and LLM streaming, WebSockets for chat, collaboration and games.
</Accordion>

<Accordion title="How do you scale WebSockets horizontally?">
Put a pub/sub backplane (Redis, NATS, Kafka) behind the socket servers so any node can publish to users connected elsewhere; size by concurrent connections; configure long LB/proxy timeouts and heartbeats; drain on deploy and use jittered client reconnects.
</Accordion>

## Related
- [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues) · [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility) · [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) · [Nginx](/Software%20Engineering/Backend/Nginx) · [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) · [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST)
- Frontend: [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) · [AI in Frontend - MCP, WebMCP & Built-in AI](/Software%20Engineering/Frontend/16%20AI%20in%20Frontend/AI%20in%20Frontend%20-%20MCP%2C%20WebMCP%20%26%20Built-in%20AI)
- Other domains: [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) · [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns) · [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index)

## Sources
- [WHATWG HTML: Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
- [Standard Webhooks specification](https://www.standardwebhooks.com/) · [GitHub repo](https://github.com/standard-webhooks/standard-webhooks)
- [Stripe: Webhooks best practices](https://docs.stripe.com/webhooks#best-practices)
- [Nginx: WebSocket proxying](https://nginx.org/en/docs/http/websocket.html)
- [MCP 2026-07-28 specification release post](https://blog.modelcontextprotocol.io/posts/2026-07-28/) · [MCP specification](https://modelcontextprotocol.io/specification/2026-07-28)
- [OpenAPI v3.2.0](https://spec.openapis.org/oas/v3.2.0.html)
- [microservices.io: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html)
