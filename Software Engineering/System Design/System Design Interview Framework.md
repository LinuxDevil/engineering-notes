---
title: "System Design Interview Framework"
tags: [system-design, system-design/interview]
aliases: [SD Framework, System Design Framework, Back of the Envelope, Latency Numbers, Napkin Math]
updated: 2026-09-27
---
# System Design Interview Framework

<Note>
**TL;DR**

A backend system design answer is a structured conversation: **clarify requirements → estimate scale → define the API → model the data → draw a high-level design → deep dive on the hard parts → discuss trade-offs and failure modes**. Numbers drive decisions, so memorize a handful of latency and capacity figures and practise back-of-the-envelope math. For front end design interviews use [RADIO Framework](/Software%20Engineering/Frontend/15%20System%20Design/RADIO%20Framework) instead (see [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index)).
</Note>

## The idea

Interviewers are not looking for "the" architecture. They look for:
- Can you **scope** an ambiguous problem?
- Do you make decisions **from numbers**, not buzzwords?
- Do you know the **building blocks** (load balancers, caches, queues, databases, CDNs) and their failure modes?
- Can you **reason about trade-offs** (consistency vs availability, latency vs cost, simplicity vs flexibility)?

## The 7 steps (45–60 minute interview)

{/* diagram source: /diagrams/diagram-412.mmd */}
![diagram](/diagrams/diagram-412.svg)

### 1. Requirements
- **Functional**: the 3–5 core use cases. Write them down. Explicitly park the rest ("out of scope: analytics, admin UI").
- **Non-functional**: scale (DAU, QPS), latency targets (p99), availability (99.9% = 8.8 h/year down, 99.99% = 53 min/year), consistency needs, durability, geography, compliance (GDPR, data residency), cost.
- Ask: read-heavy or write-heavy? Is stale data OK? What's the hardest part?

### 2. Estimates (back-of-the-envelope)
Goal: order of magnitude, not precision. Round aggressively.

| Handy conversion | Value |
|---|---|
| Seconds per day | ~86,400 ≈ **10⁵** |
| 1 M requests/day | ≈ **12 QPS** average |
| 100 M requests/day | ≈ 1,200 QPS avg, peak ≈ 2–3× |
| 1 KB × 1 B records | 1 TB |
| 1 char (ASCII/UTF-8 Latin) | 1 byte |
| Seconds per month | ~2.6 M |

<Info>
**Worked estimate: photo sharing app**

- 100 M DAU, each views 50 photos and uploads 0.1 photo/day.
- Reads: 100 M × 50 / 10⁵ s ≈ **50,000 QPS** avg, ~150 k peak. Clearly read-heavy → caches + CDN.
- Writes: 100 M × 0.1 / 10⁵ ≈ **100 uploads/s**.
- Storage: 10 M photos/day × 2 MB ≈ **20 TB/day** ≈ 7 PB/year → object storage, not a database ([Blob Storage & File Uploads](/Software%20Engineering/System%20Design/Blob%20Storage%20%26%20File%20Uploads)).
- Metadata: 10 M rows/day × 500 B ≈ 5 GB/day → one DB for a while, shard later ([Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning)).
</Info>

### Latency numbers every engineer should know
Approximate, modern hardware; the ratios matter more than exact values.

| Operation | Time | Relative |
|---|---|---|
| L1 cache reference | ~1 ns | |
| Branch mispredict | ~3 ns | |
| L2 cache reference | ~4 ns | |
| Mutex lock/unlock (uncontended) | ~17 ns | |
| Main memory (DRAM) reference | ~100 ns | 100× L1 |
| Compress 1 KB (fast codec) | ~2 µs | |
| Send 1 KB over 10 Gbps network | ~1 µs | |
| Read 4 KB random from NVMe SSD | ~20–100 µs | |
| Read 1 MB sequentially from memory | ~5–10 µs | |
| Round trip inside a datacenter | ~0.5 ms | |
| Read 1 MB sequentially from SSD | ~50 µs–1 ms | |
| Redis GET over the network | ~0.2–1 ms | |
| Disk (HDD) seek | ~2–10 ms | |
| Round trip same region, cross-AZ | ~1–2 ms | |
| Round trip US East ↔ US West | ~60–70 ms | |
| Round trip US ↔ Europe | ~80–150 ms | |
| TLS 1.3 + TCP new connection | 2 RTT (1 RTT with QUIC) | |

Takeaways: memory is ~1000× faster than a network hop; a cross-continent round trip costs as much as millions of CPU instructions; **avoid chatty cross-region calls**. See [Computer Architecture Basics](/Software%20Engineering/CS%20Fundamentals/Computer%20Architecture%20Basics).

### Capacity rules of thumb
| Component | Rough capacity per node (well-tuned) |
|---|---|
| Stateless web/API server | 1k–10k+ simple req/s |
| PostgreSQL / MySQL | 5k–20k simple queries/s; writes lower |
| Redis / Valkey | 100k+ ops/s per core-bound instance |
| Kafka broker | hundreds of MB/s |
| Single TCP connection | limited by RTT × window; use pools |

### 3. API
Define the main endpoints with inputs/outputs; mention auth, pagination (cursor based), idempotency keys for writes. See [REST API Design](/Software%20Engineering/APIs%20%26%20Auth/REST%20API%20Design), [gRPC vs REST](/Software%20Engineering/APIs%20%26%20Auth/gRPC%20vs%20REST), [GraphQL vs REST](/Software%20Engineering/APIs%20%26%20Auth/GraphQL%20vs%20REST).
```http
POST /v1/photos            (Idempotency-Key: 7c1e...) -> 201 {id, uploadUrl}
GET  /v1/feed?cursor=abc&limit=20 -> 200 {items[], nextCursor}
```

### 4. Data model
Entities, access patterns first, then storage choice. Ask: what are the queries? what's the partition key? what needs transactions? SQL vs NoSQL: [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types), [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem).

### 5. High-level design
Draw the boxes: clients → CDN → load balancer → stateless services → cache → databases / queues / blob storage. Walk one request through it end to end.

{/* diagram source: /diagrams/diagram-413.mmd */}
![diagram](/diagrams/diagram-413.svg)

### 6. Deep dives
Pick the 2–3 hardest parts (the interviewer often chooses): hot keys, fan-out, consistency, ordering, exactly-once, scaling the write path, multi-region.

### 7. Trade-offs and failure modes
- What breaks first as load grows 10×? 100×?
- What happens when the cache, DB primary, a region, or a dependency goes down?
- Monitoring and alerting: SLIs, SLOs, dashboards ([Observability - Logs, Metrics & Traces](/Software%20Engineering/DevOps/Observability%20-%20Logs%2C%20Metrics%20%26%20Traces)).
- Cost: egress, storage tiers, over-provisioning.

## Common pitfalls
<Warning>
**Pitfalls**

- Jumping to Kafka + microservices + sharding before asking about scale.
- Estimating to three decimal places; nobody cares. Orders of magnitude.
- Designing a single box for "the database" without a partition key or access patterns.
- Ignoring failure: every arrow in the diagram can time out.
- Monologuing. Check in with the interviewer at each step.
</Warning>

## Senior perspective
- Seniors **state assumptions and SLOs explicitly** and revisit them when a deep dive contradicts them.
- They name the **bottleneck** before scaling anything, and prefer the simplest design that meets requirements ("one Postgres with read replicas gets you far").
- They talk about **operability**: deployments, migrations, backfills, schema evolution, on-call, blast radius, and cell-based architecture.
- They cite real incidents as design inputs (see 🆕 below and [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns)).

<Info>
**🆕 2025–2026**

Real outages worth citing when you discuss failure modes:
- **2025-10-20, AWS us-east-1**: a race condition in DynamoDB's automated DNS management left the regional endpoint with an empty DNS record; the disruption cascaded to EC2 launches, Network Load Balancer health checks and many AWS services, and to thousands of customer apps. Lesson: hidden dependencies on a single region and on DNS automation. ([AWS post-event summary](https://aws.amazon.com/message/101925/))
- **2025-11-18, Cloudflare**: a database permission change made a Bot Management feature file double in size; the proxy's hard size limit caused failures across the network for ~3 hours. Lesson: validate internally generated config like user input; roll out config progressively. ([Cloudflare blog](https://blog.cloudflare.com/18-november-2025-outage/))
- **2025-12-05, Cloudflare**: a global configuration change (made while mitigating a React Server Components CVE) propagated instantly and broke ~28% of HTTP traffic for ~25 minutes. Cloudflare then announced a "fail small" plan with staged config rollouts. ([Cloudflare blog](https://blog.cloudflare.com/5-december-2025-outage/), [Code Orange: Fail Small](https://blog.cloudflare.com/fail-small-resilience-plan/))
</Info>

## Interview questions
<Accordion title="How do you convert daily active users into QPS?">
Requests/day ÷ ~10⁵ seconds. 10 M DAU × 20 requests = 200 M/day ≈ 2,000 QPS average; multiply by 2–3 for peak. State the peak factor as an assumption.
</Accordion>

<Accordion title="What does 99.99% availability mean in downtime?">
About 52.6 minutes per year, or ~4.3 minutes per month. Each extra nine is 10× less downtime and usually much more than 10× the cost. Serial dependencies multiply: two 99.9% services in series give ~99.8%.
</Accordion>

<Accordion title="Why does the order &quot;requirements before design&quot; matter?">
Requirements set the constraints that make one design better than another. Without read/write ratio, latency and consistency targets, every choice is arbitrary, and you may over-engineer (or miss the real bottleneck).
</Accordion>

<Accordion title="Name three numbers that change architecture decisions.">
DRAM (~100 ns) vs datacenter round trip (~0.5 ms) vs cross-continent round trip (~100 ms). They justify in-process caches, co-locating chatty services, and CDNs / regional deployments.
</Accordion>

<Accordion title="How do you pick what to deep dive on?">
The part that is hardest at the stated scale, or the one with the biggest correctness risk: e.g. fan-out for feeds, ordering for chat, idempotency for payments, hot keys for caches. Ask the interviewer which they prefer.
</Accordion>

## Related
- [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index)
- [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index) · [RADIO Framework](/Software%20Engineering/Frontend/15%20System%20Design/RADIO%20Framework)
- [Scalability Fundamentals](/Software%20Engineering/System%20Design/Scalability%20Fundamentals) · [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies) · [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) · [Consistency Models & Consensus](/Software%20Engineering/System%20Design/Consistency%20Models%20%26%20Consensus)
- Case studies: [Case Study - URL Shortener](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20URL%20Shortener) · [Case Study - Rate Limiter](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Rate%20Limiter) · [Case Study - Chat System](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Chat%20System)

## Sources
- Alex Xu, *System Design Interview: An Insider's Guide* (vol. 1 and 2)
- Martin Kleppmann, *Designing Data-Intensive Applications* (O'Reilly)
- [Jeff Dean: Numbers everyone should know (via Peter Norvig, "Teach Yourself Programming in Ten Years")](https://norvig.com/21-days.html#answers)
- [Colin Scott: Latency numbers every programmer should know (interactive)](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
- [Google SRE Book: Embracing Risk / availability](https://sre.google/sre-book/embracing-risk/)
- [AWS: Summary of the Amazon DynamoDB service disruption, us-east-1, Oct 2025](https://aws.amazon.com/message/101925/)
- [Cloudflare: 18 November 2025 outage](https://blog.cloudflare.com/18-november-2025-outage/)
