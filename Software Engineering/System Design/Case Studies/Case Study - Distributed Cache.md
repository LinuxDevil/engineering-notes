---
title: "Case Study - Distributed Cache"
tags: [system-design, system-design/case-study]
aliases: [Distributed Cache Design, Design Redis, Design Memcached, Key-Value Cache]
updated: 2026-09-27
---
# Case Study - Distributed Cache

<Note>
**TL;DR**

Design a Redis/Memcached-like in-memory cache cluster: an in-node **hash table + LRU/LFU eviction with TTLs**, data **partitioned** across nodes with consistent hashing or hash slots, **replicated** (async primary → replicas) for availability, with **failure detection and failover** (gossip or a coordinator), and clients that route keys to the right node. The hard parts are hot keys, resharding without downtime, memory efficiency, and the consistency you promise during failover.
</Note>

## 1. Requirements
**Functional**: `GET`, `SET key value [TTL]`, `DEL`; eviction when full; optional atomic ops (`INCR`, CAS).
**Non-functional**: p99 < 1 ms within a datacenter; 1 M+ ops/s cluster-wide; horizontal scaling; tolerate node failure with minimal data loss (it's a cache, so some loss is OK); online resharding.

## 2. Estimates
- 1 M ops/s, 90% reads; 500 M keys × (100 B key+overhead + 1 KB value) ≈ **550 GB** → ~10 nodes × 64 GB usable, ×2 with replicas.
- Per node ~100k ops/s → 10 primaries handle 1 M ops/s with headroom for hot spots.
- Network: 1 M × 1 KB = 1 GB/s cluster-wide ≈ 8 Gbps; spread across nodes.

## 3. API / protocol
Text or binary protocol over TCP (e.g. RESP), pipelining and multi-get supported:
```text
SET user:42 "{...}" EX 300      → OK
GET user:42                      → "{...}"  | (nil)
MGET a b c                       → [..]
CAS key expected_version value   → OK | CONFLICT
```

## 4. Single-node design
- **Hash table** (open addressing or chaining) with incremental rehashing so resizing doesn't pause.
- **Eviction**: approximate LRU/LFU by sampling (Redis samples a few keys and evicts the best candidate) rather than a global linked list, saving memory and locks. See [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies).
- **TTL**: lazy expiry on access + periodic active sampling of keys with TTLs.
- **Memory**: slab allocator (Memcached) or jemalloc; avoid fragmentation; compact encodings for small values.
- **Concurrency**: single-threaded event loop per core/shard (Redis-style, no locks, [Operating Systems Essentials](/Software%20Engineering/CS%20Fundamentals/Operating%20Systems%20Essentials) epoll) or multithreaded with striped locks (Memcached). I/O threads for network parsing.

## 5. High-level design
{/* diagram source: /diagrams/diagram-393.mmd */}
![diagram](/diagrams/diagram-393.svg)

## 6. Deep dives
### Partitioning
- Keys → 16,384 **hash slots** → nodes (Redis Cluster) or a **consistent hash ring** with virtual nodes ([Consistent Hashing](/Software%20Engineering/System%20Design/Consistent%20Hashing)).
- **Routing**: smart clients cache the slot map and handle `MOVED` redirects; or a proxy tier (Twemproxy, Envoy, mcrouter) keeps clients simple at the cost of a hop.

### Replication and failover
- Primary acks writes before replicating (async) → low latency, but a failover can lose the last writes. Acceptable for a cache; document it.
- Failure detection: nodes gossip heartbeats; when a majority of primaries agree a node is down, its replica is promoted (epoch/term numbers prevent split brain, see [Consistency Models & Consensus](/Software%20Engineering/System%20Design/Consistency%20Models%20%26%20Consensus)).
- Alternative: external coordinator (Sentinel, etcd) holds the authoritative topology.

### Resharding
Move slots from node A to B while serving traffic: migrate keys (or whole slots atomically, as in Valkey 9), redirect clients with `ASK` during migration, then update the slot map. Throttle migration to protect latency.

### Hot keys and big keys
- Detect with sampling/metrics. Mitigate hot keys by client-side near cache (short TTL), key replication (`key#0..#7`), or reading from replicas.
- Split big values; cap value size (Memcached 1 MB default).

### Consistency with the database
The cache is a derived copy: cache-aside with delete-on-write and TTLs; lease tokens (Facebook memcache) to stop stale sets after invalidation; CDC-driven invalidation for strictness.

### Persistence (optional)
Snapshots (fork + copy-on-write) and/or append-only log for warm restarts. Warm restarts matter because a cold cache can overload the DB.

## 7. Trade-offs and failure modes
| Decision | Option A | Option B |
|---|---|---|
| Routing | Smart client (no hop, complex clients) | Proxy (simple clients, extra hop) |
| Replication | Async (fast, may lose writes) | Sync/quorum (slower, safer) |
| Threads | Single-threaded shards (simple, predictable) | Multithreaded (better per-node throughput) |
| Eviction | Sampled LRU/LFU (cheap) | Exact LRU (memory/locks) |

Failure scenarios: node loss (replica promotion, brief misses), network partition (minority side stops accepting writes), cluster-wide restart (cold cache → stampede on DB; use warm-up and request coalescing).

## Senior perspective
- Decide what the cache **promises**: best-effort (typical) or durable (then it's a database: consider MemoryDB-style designs with a durable log).
- Protect the DB from the cache's failure modes: circuit breaker on misses, rate-limited recomputation.
- Capacity: plan for 2× memory headroom during resharding/forks (copy-on-write).
- Multi-tenant clusters need per-tenant quotas and key prefixes; noisy neighbors are common.

<Info>
**🆕 2025–2026**

The open-source landscape changed: **Valkey** (Linux Foundation fork of Redis 7.2) reached **9.0 on 2025-10-21** with atomic slot migration, hash-field TTLs and multi-DB cluster mode, while **Redis 8.0 (2025-05-01)** became available under **AGPLv3** again with vector sets and the former Stack modules in core. ([Valkey 9.0](https://valkey.io/blog/introducing-valkey-9/), [Redis AGPLv3](https://redis.io/blog/agplv3/))
</Info>

## Interview questions
<Accordion title="How do clients find the node for a key?">
Hash the key to a slot or ring position and look it up in a cached cluster map. Nodes reply with redirects (MOVED/ASK) when the map is stale, and clients refresh it. Alternatively, a proxy does the routing.
</Accordion>

<Accordion title="What happens when a primary fails?">
Peers detect missed heartbeats, a replica is promoted with a new epoch, and the slot map is updated. Writes acknowledged by the old primary but not yet replicated are lost; clients see errors briefly and then retry against the new primary.
</Accordion>

<Accordion title="How do you add a node without downtime?">
Move a subset of slots (or vnode ranges) to the new node gradually, serving from the old owner and redirecting during migration, then flip ownership and update clients. Consistent hashing/slots keep the moved fraction around 1/N.
</Accordion>

<Accordion title="How is eviction implemented efficiently?">
Approximate LRU/LFU: sample a handful of keys and evict the worst one, instead of maintaining a global ordered structure. Expired keys are removed lazily on access plus periodic sampling.
</Accordion>

<Accordion title="How would you handle a key receiving 500k reads/s?">
Client-side near cache with short TTL, replicate the key across nodes under suffixed names and pick randomly, allow reads from replicas, and coalesce misses.
</Accordion>

## Related
- [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index) · [System Design Interview Framework](/Software%20Engineering/System%20Design/System%20Design%20Interview%20Framework)
- [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies) · [Consistent Hashing](/Software%20Engineering/System%20Design/Consistent%20Hashing) · [Consistency Models & Consensus](/Software%20Engineering/System%20Design/Consistency%20Models%20%26%20Consensus) · [Case Study - Rate Limiter](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Rate%20Limiter) · [Operating Systems Essentials](/Software%20Engineering/CS%20Fundamentals/Operating%20Systems%20Essentials)

## Sources
- [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Redis: approximated LRU / LFU eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Nishtala et al.: Scaling Memcache at Facebook (NSDI 2013)](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala)
- [DeCandia et al.: Dynamo (SOSP 2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Valkey 9.0 announcement](https://valkey.io/blog/introducing-valkey-9/)
