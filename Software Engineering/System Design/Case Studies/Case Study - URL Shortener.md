---
title: "Case Study - URL Shortener"
tags: [system-design, system-design/case-study]
aliases: [URL Shortener, TinyURL, Bitly Design]
updated: 2026-09-27
---
# Case Study - URL Shortener

<Note>
**TL;DR**

A URL shortener maps a short code to a long URL and redirects. It's extremely **read-heavy** (~100:1), so the design centres on a **fast key-value lookup behind a cache and CDN**. Generate codes by **Base62-encoding a unique ID** (from a counter/range allocator or Snowflake) to avoid collisions, or hash + collision check for dedup. Use **302** if you need click analytics, 301 if you want browsers to cache. Analytics go through a queue, never on the redirect's critical path.
</Note>

## 1. Requirements
**Functional**: create short URL (optional custom alias, expiry); redirect `GET /{code}` to the long URL; basic click analytics.
**Non-functional**: redirect p99 < 50 ms, highly available (redirects must work even if creation is down), codes unguessable enough to prevent enumeration if links are private, no collisions.
**Out of scope**: user accounts UI, link previews.

## 2. Estimates
- 100 M new URLs/month ≈ **40 writes/s** (peak ~100).
- 100:1 read ratio → ~4,000 redirects/s avg, **~10k/s peak**.
- 5 years: 100 M × 60 = **6 B** URLs × ~500 B ≈ **3 TB**. Fits a sharded KV store or even one large DB with partitioning.
- Code length: Base62 with 7 chars = 62⁷ ≈ **3.5 trillion** codes, plenty.

## 3. API
```http
POST /v1/urls  {"longUrl": "https://...", "alias": "promo", "expiresAt": "..."}
  → 201 {"code": "aZ3kP9q", "shortUrl": "https://sho.rt/aZ3kP9q"}
GET  /{code}   → 302 Location: https://...   (404 / 410 if missing / expired)
GET  /v1/urls/{code}/stats → {clicks, byCountry, byDay}
```

## 4. Data model
```sql
CREATE TABLE urls (
  code        varchar(10) PRIMARY KEY,
  long_url    text NOT NULL,
  owner_id    bigint,
  created_at  timestamptz NOT NULL DEFAULT now(),
  expires_at  timestamptz
);
-- optional, for dedup of identical long URLs per owner:
CREATE UNIQUE INDEX ON urls (owner_id, md5(long_url));
```
Access pattern is a pure key lookup → any KV store (DynamoDB, Cassandra) or Postgres partitioned by hash of `code`.

## 5. High-level design
{/* diagram source: /diagrams/diagram-397.mmd */}
![diagram](/diagrams/diagram-397.svg)

## 6. Deep dives
### Generating the short code
| Approach | How | Pros | Cons |
|---|---|---|---|
| **Counter + Base62** | Unique integer → Base62 | No collisions, short | Sequential = enumerable; needs a distributed counter |
| **Range allocation** | Each server leases a block of 1 M IDs from a DB/etcd | No per-request coordination | Gaps on crash (fine) |
| **Hash (MD5/SHA) truncated** | First 7 Base62 chars of hash(url) | Same URL → same code (dedup) | Collisions; must check and retry with salt |
| **Random 7 chars** | CSPRNG | Unguessable | Collision check on insert (`INSERT ... ON CONFLICT`) |

```python
ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
def base62(n: int) -> str:
    s = []
    while n:
        n, r = divmod(n, 62); s.append(ALPHABET[r])
    return "".join(reversed(s)) or "0"
base62(125_000_000_000)   # '2crtgcg'  (7 chars)
```
To avoid enumerable codes from a counter, permute the integer first (e.g. a Feistel cipher or multiply by a large odd number mod 62⁷) before encoding. See [Unique ID Generation](/Software%20Engineering/System%20Design/Unique%20ID%20Generation).

### 301 vs 302
- **301 Moved Permanently**: browsers and CDNs cache it → fewer hits, but you **lose analytics** and can't change the destination.
- **302 Found / 307**: every click hits you (or the CDN with a short TTL) → accurate analytics, editable links. Most commercial shorteners use 302.

### Read path performance
- Cache-aside in Redis with long TTL ([Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies)); popular links follow a power law so a small cache gets a >90% hit ratio.
- Cache **negative lookups** briefly to stop scans for random codes hitting the DB.
- Edge caching of redirects with a short `s-maxage` for viral links.

### Analytics without slowing redirects
Emit a click event (code, timestamp, IP-derived country, user agent, referrer) to Kafka fire-and-forget; aggregate in stream processors or an OLAP store. Losing a tiny fraction of clicks is acceptable; slowing redirects is not.

### Abuse
Malware/phishing URLs: check against Safe Browsing lists on create and asynchronously re-scan; rate limit creation per user/IP ([Case Study - Rate Limiter](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Rate%20Limiter)); interstitial warning pages.

## 7. Trade-offs and failure modes
- Cache down → DB must handle 10k QPS briefly; size the DB or use replicas.
- ID allocator down → servers keep using leased ranges; only new range requests fail.
- Expired links: lazy check on read + background cleanup job.
- Custom aliases share the namespace: reserve words (`api`, `admin`), check uniqueness.

## Senior perspective
- Decouple the **read path** (tiny, cacheable, must be up) from the **write path** and analytics; they can fail independently.
- Global users: replicate the URL table to multiple regions (it's append-mostly, easy) and redirect from the nearest.
- Privacy: IP addresses in analytics are personal data (GDPR); truncate or hash.

## Interview questions
<Accordion title="How do you avoid collisions?">
Derive codes from unique IDs (counter/range allocator/Snowflake) encoded in Base62, which can't collide. If using hashes or random strings, enforce a unique constraint and retry on conflict.
</Accordion>

<Accordion title="How long should the code be?">
Work from capacity: 6 B URLs needs log₆₂(6×10⁹) ≈ 5.5 → 6 chars minimum; 7 chars (3.5 T) gives headroom and makes random guessing sparse.
</Accordion>

<Accordion title="301 or 302?">
302 (or 307) if you need analytics or editable destinations, because browsers won't cache it permanently. 301 minimizes load but you lose visibility and control.
</Accordion>

<Accordion title="How would you scale to 1 M redirects/s?">
CDN edge caching for hot codes, large Redis cluster, stateless redirect fleet in multiple regions, replicated read-only URL store per region, async analytics pipeline.
</Accordion>

## Related
- [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index) · [System Design Interview Framework](/Software%20Engineering/System%20Design/System%20Design%20Interview%20Framework)
- [Unique ID Generation](/Software%20Engineering/System%20Design/Unique%20ID%20Generation) · [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies) · [CDN & Edge Computing](/Software%20Engineering/System%20Design/CDN%20%26%20Edge%20Computing) · [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) · [Case Study - Rate Limiter](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Rate%20Limiter)

## Sources
- Alex Xu, *System Design Interview*, vol. 1, ch. 8 "Design a URL Shortener"
- [RFC 9110: HTTP Semantics, 301/302/307/308](https://www.rfc-editor.org/rfc/rfc9110#name-redirection-3xx)
- [Google Safe Browsing API](https://developers.google.com/safe-browsing)
