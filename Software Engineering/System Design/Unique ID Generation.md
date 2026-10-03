---
tags: [system-design, system-design/ids]
aliases: [ID Generation, UUID, UUIDv7, UUIDv4, Snowflake ID, ULID, Distributed IDs]
updated: 2026-09-27
---
# Unique ID Generation

> [!tldr] TL;DR
> Distributed systems need IDs that are unique without a central counter. **UUIDv4** is 122 random bits: trivially unique but random order hurts B-tree indexes. **UUIDv7** (RFC 9562, 2024) puts a millisecond Unix timestamp first, so IDs are roughly time-ordered and index-friendly; it's the modern default and PostgreSQL 18 generates it natively with `uuidv7()`. **Snowflake** IDs pack timestamp + machine ID + sequence into 64 bits (compact, sortable, needs worker-ID coordination). **ULID** is a similar 128-bit, sortable, Base32 format that predates v7.

## Requirements to ask about
- Uniqueness across nodes/regions without coordination?
- **Sortable by time**? (feeds, pagination, index locality)
- Size: 64-bit (fits `BIGINT`, cheaper indexes) or 128-bit?
- Must not leak information (creation time, volume, sequential guessability)?
- Generation rate per node (e.g. 10k IDs/s)?

## Options
| Scheme | Bits | Sortable | Coordination | Notes |
|---|---|---|---|---|
| DB auto-increment | 64 | Yes | Single DB (bottleneck) | Simple; leaks counts; hard to shard. Use step/offset per shard, or ranges |
| Ticket server (Flickr) | 64 | Yes | Central service | Batches of IDs; SPOF unless doubled |
| **UUIDv4** | 128 (122 random) | No | None | Random inserts fragment B-tree indexes |
| **UUIDv7** | 128 (48-bit ms timestamp + 74 random/counter) | Yes (ms) | None | RFC 9562; the recommended UUID for new DB keys |
| **ULID** | 128 (48-bit ms + 80 random) | Yes | None | 26-char Crockford Base32; not an RFC |
| **Snowflake** | 64 (41 ms + 10 machine + 12 seq) | Yes | Worker IDs must be unique | Twitter/X, Discord, Instagram variants |
| KSUID | 160 | Yes (s) | None | Segment's format |
| NanoID | configurable | No | None | Short URL-safe random strings |

### Snowflake layout
```text
 0 | 41 bits: ms since custom epoch | 10 bits: machine id | 12 bits: sequence
 sign  (~69 years)                    (1024 workers)        (4096 per ms per worker)
```
```python
import time, threading
class Snowflake:
    EPOCH = 1_700_000_000_000  # custom epoch (ms)
    def __init__(self, worker_id):
        assert 0 <= worker_id < 1024
        self.worker, self.seq, self.last, self.lock = worker_id, 0, -1, threading.Lock()
    def next_id(self):
        with self.lock:
            now = int(time.time() * 1000)
            if now < self.last:
                raise RuntimeError("clock moved backwards")   # or wait until last
            if now == self.last:
                self.seq = (self.seq + 1) & 0xFFF
                if self.seq == 0:                                # 4096 used this ms
                    while now <= self.last: now = int(time.time() * 1000)
            else:
                self.seq = 0
            self.last = now
            return ((now - self.EPOCH) << 22) | (self.worker << 12) | self.seq
```
Pitfalls: **clock going backwards** (NTP step) → duplicates unless you refuse or wait; **worker ID assignment** needs ZooKeeper/etcd, a DB lease, or pod ordinal (StatefulSet).

### UUIDv7 layout
```text
 unix_ts_ms (48) | ver=7 (4) | rand_a (12) | var (2) | rand_b (62)
 019a3c5e-8b2f-7c41-9d3e-5a1b2c3d4e5f
 └─ timestamp ─┘  └ version 7
```
RFC 9562 allows using part of the random bits as a counter for monotonic ordering within the same millisecond.

## Why random keys hurt databases
B-tree primary keys (Postgres, MySQL/InnoDB clustered index) insert in key order. Random UUIDv4 inserts land on random leaf pages → page splits, poor cache locality, more WAL and bigger indexes. Time-ordered IDs (v7, Snowflake, auto-increment) append to the "right edge" of the index, keeping the hot pages in memory. See [[Database Indexes]] and [[Computer Architecture Basics]] (locality).

```sql
-- PostgreSQL 18+
CREATE TABLE orders (
  id uuid PRIMARY KEY DEFAULT uuidv7(),
  created_at timestamptz NOT NULL DEFAULT now(),
  total_cents bigint NOT NULL
);
SELECT uuid_extract_timestamp(id) FROM orders LIMIT 1;  -- time is embedded
```

## Trade-offs
- **Information leak**: v7/Snowflake/ULID reveal creation time; auto-increment reveals volume and is enumerable. For public identifiers where that matters, use v4 or an opaque mapping.
- **Hot partition**: time-ordered keys all go to the latest range in range-partitioned stores (Bigtable, HBase, Spanner). There, hash-prefixed or random keys spread writes better.
- **Size**: 128-bit UUIDs double index size vs `BIGINT`; store as native `uuid`, not `varchar(36)`.
- **Client-side generation** (UUIDs) lets clients create IDs offline and makes writes idempotent (retry with the same ID).

## Senior perspective
- Default: **UUIDv7** for new internal primary keys, 64-bit Snowflake-style when size matters a lot (tweets, messages at huge scale), random/opaque tokens for anything security-sensitive (never use IDs as secrets).
- Keep a separate **public slug** if you need short or human-friendly IDs (URL shortener: [[Case Study - URL Shortener]]).
- Check library behavior: monotonicity within a ms, clock regression handling, and whether the DB driver maps UUID natively.

> [!info]+ 🆕 2025–2026
> - **RFC 9562 (May 2024)** obsoleted RFC 4122 and standardized UUID versions 6, 7 and 8; v7 is recommended for new time-ordered use. ([RFC 9562](https://www.rfc-editor.org/rfc/rfc9562))
> - **PostgreSQL 18 (2025-09-25)** added a built-in `uuidv7()` function (plus `uuidv4()` alias) and `uuid_extract_timestamp()` works on v7. ([PostgreSQL 18 release](https://www.postgresql.org/about/news/postgresql-18-released-3142/), [docs: UUID functions](https://www.postgresql.org/docs/18/functions-uuid.html))

## Interview questions
> [!question]- Design an ID generator for 10k IDs/s across many services.
> Snowflake-style 64-bit IDs generated locally: timestamp + worker ID + per-ms sequence. Assign worker IDs via etcd/ZooKeeper leases or StatefulSet ordinals, handle clock regressions by waiting or refusing, and use a custom epoch. Or UUIDv7 if 128 bits is acceptable, with no coordination at all.

> [!question]- UUIDv4 vs UUIDv7 as a primary key?
> Both are unique without coordination. v4 is fully random, so inserts scatter across the B-tree causing page splits and cache misses. v7 starts with a timestamp, so inserts are nearly sequential, giving better locality and smaller, faster indexes; the trade-off is that it reveals creation time.

> [!question]- What happens if the clock goes backwards in Snowflake?
> You could generate duplicate IDs. Implementations detect `now < last` and either wait until the clock catches up, fail, or use a logical clock (keep using `last`, increment sequence).

> [!question]- Why not use a database sequence everywhere?
> One sequence is a single point of coordination and a throughput bottleneck across shards/regions, and it leaks business volume. Per-shard offsets or ranges help but complicate operations.

> [!question]- When would you avoid time-ordered IDs?
> When creation time must not be exposed, or in range-partitioned stores where monotonically increasing keys create a write hotspot on the last partition.

## Related
- [[System Design Index]]
- [[Case Study - URL Shortener]] · [[Case Study - Chat System]] · [[Sharding & Partitioning]] · [[Database Indexes]] · [[Encoding, Serialization & Character Sets]]

## Sources
- [RFC 9562: Universally Unique IDentifiers (UUIDs)](https://www.rfc-editor.org/rfc/rfc9562)
- [PostgreSQL 18 released](https://www.postgresql.org/about/news/postgresql-18-released-3142/)
- [PostgreSQL docs: UUID functions](https://www.postgresql.org/docs/18/functions-uuid.html)
- [X/Twitter engineering: Announcing Snowflake (2010)](https://blog.x.com/engineering/en_us/a/2010/announcing-snowflake)
- [ULID spec](https://github.com/ulid/spec)
- [Flickr: Ticket servers, distributed unique primary keys on the cheap](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)
