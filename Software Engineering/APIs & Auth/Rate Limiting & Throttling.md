---
title: "Rate Limiting & Throttling"
tags: [apis, apis/reliability, backend]
aliases: [Rate Limiting, Throttling, Token Bucket, Leaky Bucket, Sliding Window, 429 Too Many Requests, RateLimit Headers]
updated: 2026-09-27
---
# Rate Limiting & Throttling

<Note>
**TL;DR**

**Rate limiting** caps how many requests a client may make in a time window and rejects the excess with **429 Too Many Requests** + `Retry-After`; **throttling** usually means slowing or queueing the excess instead of rejecting. The classic algorithms are **fixed window**, **sliding window (log or counter)**, **token bucket** (allows bursts) and **leaky bucket** (smooths output). In a multi-instance system the counter lives in a shared store like **Redis** with atomic scripts. Advertise limits with the (still draft) `RateLimit` / `RateLimit-Policy` headers.
</Note>

## Why rate limit

- **Protect availability**: stop one client (buggy retry loop, scraper, DDoS) from starving everyone.
- **Fairness and pricing**: free tier 100 req/min, paid 10,000.
- **Cost control**: expensive endpoints (LLM calls, SMS, exports).
- **Security**: slow down credential stuffing, OTP brute force, enumeration ([API4: Unrestricted Resource Consumption](/Software%20Engineering/Security/API%20Security)).

## What to key on

| Key | Use |
|---|---|
| API key / OAuth client ID | per-customer quotas (the normal case) |
| User ID | per-user fairness inside one customer |
| IP address | unauthenticated endpoints, login; beware NAT/CGNAT and IPv6 ranges (limit per /64) |
| Endpoint / cost | stricter limits on expensive routes; GraphQL: by query cost |
| Tenant | noisy-neighbour protection in multi-tenant SaaS |

Usually **layered**: a coarse IP limit at the edge (CDN/WAF/[Nginx](/Software%20Engineering/Backend/Nginx)), per-key quotas at the [gateway](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies), and business limits (e.g. 5 password attempts) in the service.

## Algorithms

| Algorithm | Idea | Bursts | Memory | Accuracy |
|---|---|---|---|---|
| Fixed window counter | count per `floor(now/window)` | up to **2×** at window edges | 1 counter | poor at edges |
| Sliding window log | store every timestamp; count those in last W | none beyond limit | O(requests) | exact |
| Sliding window counter | weighted mix of current and previous window counts | small | 2 counters | ~approximate, good |
| Token bucket | bucket of capacity B refills at rate R; each request takes a token | yes, up to B | 2 values | good |
| Leaky bucket (queue) | requests enter a queue drained at a constant rate | absorbed into queue, output is smooth | queue | good; adds latency |
| GCRA | token bucket expressed as a single "theoretical arrival time" | yes | 1 value | good (used by redis-cell, Stripe-like limiters) |

### Fixed window's edge problem

Limit 100/min: 100 requests at 12:00:59 and 100 at 12:01:00 are both allowed, i.e. 200 in two seconds.

### Sliding window counter (the practical approximation)

```text
estimate = count_current + count_previous × (1 − elapsed_in_current / window)
```
At 25% into the current minute with 40 now and 80 last minute: `40 + 80 × 0.75 = 100` → at the limit.

### Token bucket (in memory, TypeScript)

```ts
class TokenBucket {
  private tokens: number;
  private last = Date.now();
  constructor(private capacity: number, private refillPerSec: number) {
    this.tokens = capacity;
  }
  tryRemove(cost = 1): boolean {
    const now = Date.now();
    this.tokens = Math.min(this.capacity,
      this.tokens + ((now - this.last) / 1000) * this.refillPerSec);
    this.last = now;
    if (this.tokens >= cost) { this.tokens -= cost; return true; }
    return false;
  }
}
// 10 req/s sustained, bursts up to 20
const bucket = new TokenBucket(20, 10);
```

### Leaky bucket (as a queue)

```ts
// Requests are queued and processed at a fixed rate: smooth output, bounded queue.
class LeakyBucket {
  private queue: Array<() => void> = [];
  constructor(private maxQueue: number, ratePerSec: number) {
    setInterval(() => this.queue.shift()?.(), 1000 / ratePerSec);
  }
  submit(job: () => void): boolean {
    if (this.queue.length >= this.maxQueue) return false; // overflow → 429/503
    this.queue.push(job); return true;
  }
}
```

### Distributed token bucket in Redis (atomic Lua)

In-memory limiters only work per instance. With 10 pods behind a load balancer, each would allow the full quota. Use one shared store and do read-modify-write **atomically**:

```lua
-- KEYS[1] = bucket key, ARGV = capacity, refill_per_sec, now_ms, cost
local b = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local cap, rate, now, cost = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3]), tonumber(ARGV[4])
local tokens = tonumber(b[1]) or cap
local ts = tonumber(b[2]) or now
tokens = math.min(cap, tokens + (now - ts) / 1000 * rate)
local allowed = 0
if tokens >= cost then tokens = tokens - cost; allowed = 1 end
redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('PEXPIRE', KEYS[1], math.ceil(cap / rate * 1000) + 1000)
return { allowed, math.floor(tokens) }
```

```ts
const [allowed, remaining] = await redis.eval(script, 1, `rl:${apiKey}`, 100, 10, Date.now(), 1);
if (!allowed) {
  res.set('Retry-After', '1');
  return res.status(429).type('application/problem+json')
    .send({ type: 'https://api.example.com/problems/rate-limited', title: 'Too many requests', status: 429 });
}
```

Use server time from one source (or Redis `TIME`) to avoid clock skew between app servers.

### Nginx example (leaky bucket with burst)

```nginx
limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;
server {
  location /api/ {
    limit_req zone=perip burst=20 nodelay;   # allow bursts of 20, then 10 r/s
    limit_req_status 429;
    proxy_pass http://app;
  }
}
```

## Telling clients about limits

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
RateLimit-Policy: "burst";q=100;w=60, "daily";q=1000;w=86400
RateLimit: "burst";r=0;t=30
Content-Type: application/problem+json
```

- `Retry-After` (RFC 9110) is standard: seconds or an HTTP date.
- `RateLimit-Policy` = the quota policies (`q` quota, `w` window seconds, optional `qu` unit, `pk` partition key). `RateLimit` = current state (`r` remaining, `t` seconds until reset).
- Many APIs still use the older de facto `X-RateLimit-Limit/-Remaining/-Reset` (GitHub style).
- Clients should back off with **exponential backoff + jitter** and honour `Retry-After`.

<Info>
**🆕 2025–2026**

- **2025-09 → 2026-05-23:** the IETF `RateLimit` header draft moved from -10 to **-11** (Structured Fields syntax shown above). It is **still an Internet-Draft**, not an RFC, and field syntax may change; a 2026 review of the previous revision flagged it "not ready".
- AI/LLM APIs popularised **token-based quotas** (limit by tokens or cost units per minute, not requests), which the draft's `qu` quota-unit parameter is designed to express.
</Info>

## Throttling vs rate limiting vs load shedding

| | Rate limiting | Throttling | Load shedding |
|---|---|---|---|
| Trigger | client exceeded *its* quota | client exceeded rate; server slows it | *server* is overloaded, regardless of client |
| Response | 429 | delay/queue (leaky bucket) or degrade | 503 + `Retry-After`, drop low-priority work |
| Goal | fairness, pricing, abuse | smoothing | survival |

Related: **concurrency limits** (max in-flight requests per client), **adaptive limits** (reduce limits as latency rises), and **circuit breakers** on the calling side.

## Pitfalls

<Warning>
**Common mistakes**

- Per-instance in-memory limiters behind a load balancer (effective limit × N pods).
- Non-atomic `GET` then `SET` in Redis (race → over-admission). Use Lua, `INCR` + `EXPIRE`, or a module like redis-cell.
- Rate limiting by IP behind a proxy without configuring the real client IP, so everyone shares the proxy's IP.
- Making Redis a hard dependency: decide **fail-open** (allow on Redis outage, protect availability) or **fail-closed** (protect downstream) consciously.
- No `Retry-After`, so clients hammer immediately.
- Counting failed/rejected requests toward quota in a way that locks out legitimate users (or not counting them on login endpoints, which helps attackers).
</Warning>

## Senior perspective

- Limits are a **product decision** (tiers, quotas, burst allowances) as much as an engineering one. Publish them.
- Place limits in **layers**: edge (cheap, IP/DDoS), gateway (per key, quotas), service (business rules), and dependency-level bulkheads.
- Global limits across regions: either accept per-region limits or use approximate, eventually-consistent counters; exact global counting adds cross-region latency.
- Observe: 429 rate per client, top limited keys, false positives. Alert when a big customer starts getting 429s before they call you.
- Review questions: *What is the key? Where is state stored and what happens when it's down? Is it atomic? How do clients learn the limits? Are retries with jitter documented?*

## Interview questions

<Accordion title="Explain token bucket vs leaky bucket.">
Token bucket: tokens refill at rate R up to capacity B; a request consumes a token or is rejected. It allows bursts up to B while enforcing an average rate. Leaky bucket: requests enter a queue drained at a fixed rate; it smooths output to a constant rate and rejects when the queue is full, adding latency instead of allowing bursts.
</Accordion>

<Accordion title="What is wrong with a fixed window counter?">
At window boundaries a client can send the full limit at the end of one window and again at the start of the next, doubling the effective rate. Sliding window log or counter, or token bucket, fix this.
</Accordion>

<Accordion title="Design a distributed rate limiter.">
Key requests by API key/user/IP; keep counters in a shared low-latency store (Redis cluster) and update atomically with a Lua script implementing token bucket or sliding window; set TTLs; run the check at the gateway; return 429 with Retry-After and RateLimit headers; choose fail-open vs fail-closed; shard keys; consider local pre-aggregation for very high throughput and per-region limits for global scale.
</Accordion>

<Accordion title="What should the response be when a client is rate limited?">
429 Too Many Requests with `Retry-After`, ideally RateLimit/RateLimit-Policy headers and a Problem Details body. For server overload (not client quota), 503 with Retry-After.
</Accordion>

<Accordion title="How should clients handle 429s?">
Honour Retry-After; otherwise retry with exponential backoff plus jitter and a max attempts cap; only retry idempotent requests or those with idempotency keys; and reduce concurrency.
</Accordion>

## Related
- [APIs & Auth Index](/Software%20Engineering/APIs%20%26%20Auth/APIs%20%26%20Auth%20Index) · [Backend Index](/Software%20Engineering/Backend/Backend%20Index)
- [API Gateways & Reverse Proxies](/Software%20Engineering/Backend/API%20Gateways%20%26%20Reverse%20Proxies) · [Nginx](/Software%20Engineering/Backend/Nginx) · [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design) · [API Security](/Software%20Engineering/Security/API%20Security) · [API Versioning, Idempotency & Backward Compatibility](/Software%20Engineering/APIs%20%26%20Auth/API%20Versioning%2C%20Idempotency%20%26%20Backward%20Compatibility) · [Background Jobs & Task Queues](/Software%20Engineering/Backend/Background%20Jobs%20%26%20Task%20Queues)
- Other domains: [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index) · [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns) · [Security Index](/Software%20Engineering/Security/Security%20Index)

## Sources
- [draft-ietf-httpapi-ratelimit-headers-11 (23 May 2026)](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
- [RFC 6585: 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc6585.html) · [RFC 9110: Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html#name-retry-after)
- [Stripe blog: Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters)
- [Cloudflare blog: How we built rate limiting capable of scaling to millions of domains](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/)
- [Nginx: ngx_http_limit_req_module](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
- [Redis: Rate limiting patterns](https://redis.io/glossary/rate-limiting/)
- [AWS Architecture Blog: Exponential backoff and jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- Alex Xu, *System Design Interview Vol. 1*, ch. 4 "Design a Rate Limiter"
