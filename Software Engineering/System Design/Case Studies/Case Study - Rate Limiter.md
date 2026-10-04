---
title: "Case Study - Rate Limiter"
tags: [system-design, system-design/case-study]
aliases: [Rate Limiter Design, Distributed Rate Limiter, Token Bucket Design]
updated: 2026-09-27
---
# Case Study - Rate Limiter

<Note>
**TL;DR**

A distributed rate limiter caps how many requests a client (user, API key, IP, tenant) can make in a time window, protecting services from abuse and overload. Pick an algorithm (**token bucket** for bursts, **sliding window counter** for accuracy at low memory), keep counters in a **shared low-latency store** (Redis/Valkey) updated **atomically** with a Lua script, enforce at the **gateway**, return **429** with `Retry-After` and rate-limit headers, and decide whether to **fail open or closed** when the store is down. Algorithms themselves are covered in [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling).
</Note>

## 1. Requirements
**Functional**: limit by key (API key, user, IP, endpoint) with configurable rules (e.g. 100 req/min per key, 10 login attempts/15 min per IP); return 429 when exceeded; expose remaining quota.
**Non-functional**: < 1–2 ms added latency; accurate across many gateway nodes; highly available; rules changeable without deploy.

## 2. Estimates
- 1 M active API keys, 50k req/s across 50 gateway nodes.
- Each check = 1 Redis round trip (~0.5 ms). 50k ops/s is easy for a small Redis cluster.
- Memory: sliding window counter = 2 counters × 1 M keys × ~50 B ≈ 100 MB. Sliding log would be far bigger.

## 3. API / interface
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 12
RateLimit-Policy: "default";q=100;w=60
RateLimit: "default";r=0;t=12
```
(Standard `RateLimit` / `RateLimit-Policy` headers are an IETF draft; many APIs still use `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.)

Rules config:
```yaml
- match: {route: "/v1/login", by: ip}
  limit: 10
  window: 15m
- match: {plan: free, by: api_key}
  limit: 100
  window: 1m
  burst: 20
```

## 4. Algorithm choice (summary)
| Algorithm | Memory/key | Bursts | Accuracy |
|---|---|---|---|
| Fixed window counter | 1 counter | 2× burst at window edges | Low |
| Sliding window log | All timestamps | Exact | Exact, memory-heavy |
| **Sliding window counter** | 2 counters | Smooth | ~Exact (weighted estimate) |
| **Token bucket** | tokens + timestamp | Allows bursts up to capacity | Good; most APIs (AWS, Stripe) |
| Leaky bucket | queue | Smooths output rate | Good for shaping |

## 5. High-level design
{/* diagram source: /diagrams/diagram-396.mmd */}
![diagram](/diagrams/diagram-396.svg)

## 6. Deep dives
### Atomicity: why a Lua script
Two gateway nodes doing `GET` then `SET` race and both allow the last token. Run the whole check-and-update atomically in Redis:
```lua
-- KEYS[1]=bucket key; ARGV: capacity, refill_per_sec, now_ms, cost
local cap, rate, now, cost = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3]), tonumber(ARGV[4])
local b = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(b[1]) or cap
local ts = tonumber(b[2]) or now
tokens = math.min(cap, tokens + (now - ts) / 1000 * rate)   -- refill
local allowed = tokens >= cost
if allowed then tokens = tokens - cost end
redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('PEXPIRE', KEYS[1], math.ceil(cap / rate * 1000) + 1000)
return { allowed and 1 or 0, tokens }
```
Use the Redis server's time (`redis.call('TIME')`) or tolerate small skew between gateway clocks.

### Sliding window counter
```text
estimate = count(current window) + count(previous window) × (1 − elapsed_fraction_of_current)
e.g. limit 100/min, prev=80, curr=30, 25% into current → 30 + 80×0.75 = 90 → allow
```

### Where to enforce
- **Edge/CDN/WAF**: per-IP floods, cheap.
- **API gateway**: per key/tenant/plan ([API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies)).
- **Service level**: protect specific expensive operations or downstreams (concurrency limits).
- Client side: SDKs honour `Retry-After` with backoff ([Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns)).

### Scaling and latency
- Shard keys across Redis cluster slots ([Consistent Hashing](/Software%20Engineering/System%20Design/Consistent%20Hashing)); one key's operations stay on one shard.
- **Local + global hybrid**: each node keeps a local token bucket and periodically syncs/borrows quota from the global store; trades accuracy for fewer round trips at very high QPS.
- Hot key (one huge tenant): split its quota across N sub-keys.

### Failure handling
- Redis unavailable → **fail open** (allow; protect with a coarse local limiter) for most APIs, **fail closed** for security-critical limits (login, OTP).
- Tight timeout (~5 ms) on the limiter call; circuit breaker around it.

## 7. Trade-offs
- Exactness vs cost: sliding log is exact but memory-heavy; counters are approximations.
- Global accuracy vs latency: central store adds a round trip; local limits drift.
- Multi-region: per-region limits (simple, clients can exceed globally by ×regions) vs global sync (latency).

## Senior perspective
- Rate limits are a **product feature**: document them, return headers, give quotas by plan, and alert customers before they hit limits.
- Distinguish **rate limiting** (per client fairness) from **load shedding** (protect the service regardless of who).
- Observe: blocked requests per rule and per tenant; a spike might mean an attack or a broken client retry loop.
- Test with a replay of production traffic before tightening limits.

## Interview questions
<Accordion title="How do you rate limit across 50 gateway nodes?">
Keep counters in a shared store like Redis, keyed by client and rule, updated atomically with a Lua script (or `INCR` + `EXPIRE` for fixed windows). Optionally add local buckets synced periodically to cut latency.
</Accordion>

<Accordion title="Token bucket vs sliding window?">
Token bucket permits controlled bursts up to the bucket size while enforcing an average rate; great for APIs. Sliding window counters enforce "N per window" smoothly without edge bursts. Choose by whether bursts are acceptable.
</Accordion>

<Accordion title="What happens if Redis goes down?">
Decide per rule: fail open for general API limits (availability over strictness), backed by a coarse in-memory limiter; fail closed for abuse-sensitive endpoints like login. Keep the limiter call on a short timeout.
</Accordion>

<Accordion title="What should the response look like?">
HTTP 429 with `Retry-After` and quota headers (limit, remaining, reset), and a clear error body. Clients should back off with jitter.
</Accordion>

## Related
- [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index) · [System Design Interview Framework](/Software%20Engineering/System%20Design/System%20Design%20Interview%20Framework)
- [Rate Limiting & Throttling](/Software%20Engineering/APIs%20%26%20Auth/Rate%20Limiting%20%26%20Throttling) · [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) · [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies) · [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns) · [Case Study - Distributed Cache](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Distributed%20Cache)

## Sources
- Alex Xu, *System Design Interview*, vol. 1, ch. 4 "Design a Rate Limiter"
- [Stripe: Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters)
- [Cloudflare: How we built rate limiting capable of scaling to millions of domains](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/)
- [IETF draft: RateLimit header fields for HTTP](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
- [Redis docs: Scripting with Lua](https://redis.io/docs/latest/develop/interact/programmability/eval-intro/)
- [RFC 6585: 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc6585#section-4)
