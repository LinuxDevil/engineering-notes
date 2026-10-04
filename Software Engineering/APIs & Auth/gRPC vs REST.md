---
title: "gRPC vs REST"
tags: [apis, apis/grpc]
aliases: [gRPC, RPC, Protocol Buffers, Protobuf, Connect RPC, gRPC-Web]
updated: 2026-09-27
---
# gRPC vs REST

<Note>
**TL;DR**

**REST** models an API as resources over HTTP with (usually) JSON; it is universal, human-readable and cache-friendly. **gRPC** models it as typed remote functions defined in **Protocol Buffers**, runs over **HTTP/2**, generates client/server code, and supports **streaming** in both directions. Use gRPC for internal service-to-service calls where performance and strict contracts matter; use REST (or GraphQL) at the public/browser edge. **Connect** and **gRPC-Web** bridge gRPC to browsers.
</Note>

![Untitled 20](/Attachments/Untitled%2020.png)

## What is an API

APIs (Application Programming Interfaces) are a software intermediary that establishes the rules for **applications to interact and talk to each other**. An API specifies the **types of requests** one application (web page, mobile app, service) can make to another, how to make them, which **data formats** to use, and the conventions callers must follow.

![Untitled 21](/Attachments/Untitled%2021.png)

### APIs and microservices

In a **monolithic application**, all functionality ships as a single unit from a single codebase. A **microservice architecture** splits it into smaller services that communicate over the network (HTTP, gRPC, messaging). Those services talk to each other **through APIs**, so the choice of API style directly affects latency, coupling and reliability. See [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture).

## What is RPC

**RPC (Remote Procedure Call)** lets a program call a function in another process or machine as if it were local, without handling the network details itself.

- Client-server model: the client calls a stub; the stub serialises arguments, sends them, waits, and deserialises the result. In the classic synchronous style the caller blocks until the response arrives (modern RPC stacks are async).
- Like REST, RPC defines **rules of interaction**: which procedures exist, their parameters and return types.
- Classic variants discussed in textbooks: **callback RPC**, **broadcast RPC**, **batch-mode RPC**.
- History: Sun RPC/ONC RPC (used by NFS) ran mostly over **UDP**, falling back to TCP for large messages. That statement applies to those old systems; modern RPC frameworks (gRPC, Thrift, Connect) run over TCP-based HTTP/2 (or HTTP/1.1).
- RPC vs REST in one line: RPC exposes **verbs** (`CreateOrder()`), REST exposes **nouns** (`POST /orders`).

## What is REST

REST responses are usually JSON (sometimes XML) over HTTP, and each piece of the system is exposed as a **resource** manipulated with `GET`, `POST`, `PUT`, `PATCH`, `DELETE`. Typical RESTful services run over HTTP/HTTPS on TCP/IP. Many so-called REST APIs are really "RPC over HTTP+JSON" (`POST /sendEmail`), which is fine as long as it is consistent. Design guidance: [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design).

## What is gRPC

gRPC is an open-source RPC framework created at Google (2015), now a **CNCF** project. The official expansion is recursive, "gRPC Remote Procedure Calls" (the "g" jokingly changes meaning each release); it is commonly called "Google RPC".

- Uses **HTTP/2** under the hood, but the HTTP mapping is hidden from developers: you call methods, not URLs.
- The contract is written in an **IDL** (interface definition language), by default **Protocol Buffers** (`.proto`), which describes both the service and the message structures.
- `protoc`/`buf` **generates** typed client stubs and server interfaces for Go, Java, C#, Python, Node, Rust, Swift, Kotlin, etc.

### Worked example

```protobuf
// orders/v1/orders.proto
syntax = "proto3";
package orders.v1;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);                       // unary
  rpc WatchOrder(WatchOrderRequest) returns (stream OrderEvent);       // server streaming
  rpc UploadItems(stream Item) returns (UploadSummary);               // client streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);          // bidirectional
}

message GetOrderRequest { string id = 1; }
message Order {
  string id = 1;
  int64 amount_cents = 2;
  string currency = 3;
  reserved 4;                 // field 4 was removed: never reuse its number
  repeated Item items = 5;
}
message Item { string sku = 1; int32 qty = 2; }
message WatchOrderRequest { string id = 1; }
message OrderEvent { string status = 1; }
message UploadSummary { int32 count = 1; }
message ChatMessage { string text = 1; }
```

```go
// Go client: always set a deadline
ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
defer cancel()
order, err := client.GetOrder(ctx, &ordersv1.GetOrderRequest{Id: "42"})
if status.Code(err) == codes.NotFound { /* handle */ }
```

```bash
# Call it from the CLI (like curl for gRPC; needs server reflection)
grpcurl -d '{"id":"42"}' localhost:50051 orders.v1.OrderService/GetOrder
```

The same order as REST would be `GET /orders/42` returning JSON.

## gRPC vs REST: comparison

### HTTP/1.1 vs HTTP/2

REST APIs follow a **request-response** model and are often served over **HTTP/1.1**, where one connection carries one request at a time (clients open several connections to get parallelism; pipelining is effectively unused). REST works fine over HTTP/2 and HTTP/3 too, and then gets multiplexing and header compression for free, but the programming model stays one request → one response (streaming needs SSE, WebSockets, or chunked responses).

gRPC is **built on HTTP/2**. Many calls are **multiplexed** as independent streams over one TCP connection, and the framework exposes **streaming** as a first-class feature. Four call types:

- **Unary**: one request, one response (like a normal REST call).
- **Server streaming**: one request, a stream of responses, then a final status (e.g. live order updates).
- **Client streaming**: a stream of requests, one response (e.g. uploading telemetry).
- **Bidirectional streaming**: both sides stream independently, in any order, over the same call. The client starts the call; either side can finish its half.

![Untitled 22](/Attachments/Untitled%2022.png)

### Browser support

REST is supported by every browser and tool. Browsers don't expose the HTTP/2 framing (trailers) that native gRPC needs, so browsers use:
- **gRPC-Web** plus a translating proxy (Envoy), or
- **Connect** protocol (Buf): a simple HTTP/1.1 or HTTP/2 protocol that is gRPC-compatible on the server side, so the same handlers serve gRPC, gRPC-Web and Connect (plain `curl` with JSON works). Connect joined the CNCF as a Sandbox project in April 2024.

So gRPC is mainly used for **internal** service-to-service traffic, mobile apps, and high-throughput pipelines.

### Payload format

gRPC uses **Protocol Buffers**: a compact **binary** format with numbered fields. Messages are smaller and faster to parse than JSON, and schemas make them strongly typed with generated classes in each language. Protobuf can also be mapped to JSON (canonical JSON mapping) for debugging.

REST mostly uses **JSON**: flexible, **human-readable**, needs no schema to get started, and every language parses it. The cost is larger payloads and slower (de)serialisation, and without a schema (OpenAPI/JSON Schema) the contract is implicit.

### Summary table

| Feature | gRPC | REST |
| --- | --- | --- |
| Model | call methods on services (RPC) | manipulate resources (nouns + HTTP verbs) |
| Transport | HTTP/2 (required) | HTTP/1.1, HTTP/2 or HTTP/3 |
| Streaming | unary, server, client, bidirectional | request/response; streaming via SSE/WebSockets |
| Browser support | needs gRPC-Web or Connect | universal |
| Contract & codegen | `.proto` is mandatory; native codegen | optional OpenAPI; codegen via OpenAPI Generator etc. |
| Payload | Protobuf (binary) by default | JSON (text) typically |
| Human-debuggable | needs tools (grpcurl, Buf Studio, Postman) | curl, browser |
| Caching | no HTTP caching (POST) | HTTP caches, CDNs, ETags |
| Errors | gRPC status codes (`NOT_FOUND`, `DEADLINE_EXCEEDED`, `UNAVAILABLE`…) + rich error details | HTTP status codes + Problem Details (RFC 9457) |
| Deadlines/cancellation | built in, propagated across hops | ad hoc (timeouts per client) |
| Load balancing | needs L7 (HTTP/2-aware) or client-side LB | any L4/L7 LB |

## Pitfalls

<Warning>
**Common mistakes**

- **L4 load balancing with gRPC**: long-lived HTTP/2 connections pin all traffic to one backend. Use an L7 proxy (Envoy, Nginx `grpc_pass`, service mesh) or client-side balancing (xDS).
- **No deadlines**: a slow downstream holds resources forever. Set a deadline on every call and propagate it.
- **Breaking protobuf compatibility**: changing a field's number or type, or reusing a deleted number. Add fields; `reserved` removed ones; run `buf breaking` in CI.
- **Retrying non-idempotent calls** blindly on `UNAVAILABLE`.
- Exposing gRPC directly to third parties who expect REST/JSON; use transcoding (`google.api.http` annotations + Envoy/gRPC-Gateway) or Connect.
</Warning>

## Senior perspective

- Choose by **boundary**: public/partner/browser edge → REST (or GraphQL for product UIs); internal east-west → gRPC; async/event-driven → messaging (Kafka etc.).
- The biggest win of gRPC is often the **contract discipline** (schemas, codegen, breaking-change detection), not raw speed.
- Operability: set up reflection, health checking (`grpc.health.v1`), retries with backoff via service config, keepalive, max message size, and tracing interceptors.
- HTTP/3 for gRPC is still not standard across implementations; plan on HTTP/2.
- Review questions: *What are the deadlines and retry policies? Which calls are idempotent? How are protos versioned (`orders.v1` → `orders.v2`) and who owns the schema registry?*

## Interview questions

<Accordion title="What are the main differences between gRPC and REST?">
gRPC is contract-first RPC over HTTP/2 with binary Protobuf, generated clients, deadlines and four streaming modes. REST is a resource-oriented style over any HTTP version, usually JSON, universally supported, human-readable and cacheable. gRPC suits internal high-throughput calls; REST suits public and browser-facing APIs.
</Accordion>

<Accordion title="Why does gRPC need HTTP/2?">
It relies on HTTP/2 streams for multiplexing many calls on one connection, bidirectional streaming, and trailers to send the final status after a streamed body. Browsers don't expose trailers to JavaScript, hence gRPC-Web and Connect.
</Accordion>

<Accordion title="How do you evolve a protobuf schema without breaking clients?">
Only add new fields with new numbers; never change a field's number or type; mark removed fields and names as `reserved`; treat new enum values carefully; version the package (`v1`, `v2`) for breaking changes; enforce with `buf breaking` in CI.
</Accordion>

<Accordion title="Why is load balancing gRPC tricky?">
gRPC keeps long-lived HTTP/2 connections and multiplexes calls on them, so connection-level (L4) balancing sends everything from a client to one server. You need request-level (L7) balancing through an HTTP/2-aware proxy or client-side load balancing.
</Accordion>

<Accordion title="What is Connect RPC?">
A protocol and set of libraries from Buf that serve the same Protobuf services over gRPC, gRPC-Web and a simple HTTP/JSON-friendly Connect protocol, so browsers and curl can call them without a translating proxy. It is a CNCF Sandbox project (accepted April 2024).
</Accordion>

## Related
- [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) · [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST) · [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) · [Webhooks, Polling, SSE & WebSockets](/Software%20Engineering/APIs%20%26%20Auth/Webhooks%2C%20Polling%2C%20SSE%20%26%20WebSockets) · [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility)
- [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture) · [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)
- Other domains: [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture) · [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index)

## Sources
- [grpc.io: Introduction](https://grpc.io/docs/what-is-grpc/introduction/) · [Core concepts](https://grpc.io/docs/what-is-grpc/core-concepts/) · [Status codes](https://grpc.io/docs/guides/status-codes/) · [Deadlines](https://grpc.io/docs/guides/deadlines/)
- [gRPC: "g" stands for… (GitHub)](https://github.com/grpc/grpc/blob/master/doc/g_stands_for.md)
- [Protocol Buffers docs](https://protobuf.dev/) · [Proto best practices](https://protobuf.dev/best-practices/dos-donts/)
- [Connect RPC docs](https://connectrpc.com/docs/introduction) · [CNCF: Connect RPC project page](https://www.cncf.io/projects/connect-rpc/)
- [RFC 9113: HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html)
- [RFC 5531: ONC RPC v2](https://www.rfc-editor.org/rfc/rfc5531.html)
- [Buf: breaking change detection](https://buf.build/docs/breaking/)
