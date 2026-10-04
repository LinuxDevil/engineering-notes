---
title: "Case Study - News Feed Backend"
tags: [system-design, system-design/case-study]
aliases: [News Feed Backend, Timeline Design, Twitter Timeline, Fan-out on Write, Fan-out on Read]
updated: 2026-09-27
---
# Case Study - News Feed Backend

<Note>
**TL;DR**

A news feed shows each user recent posts from people they follow, ranked. The core decision is **fan-out on write** (push post IDs into each follower's precomputed feed cache: fast reads, expensive for celebrities) vs **fan-out on read** (pull and merge at read time: cheap writes, slow reads). Real systems use a **hybrid**: push for normal users, pull for celebrities, merged at read time, then **ranked**. Feeds store **IDs only**; post and user objects are hydrated from caches. The client side of this problem is in [Case Study - News Feed](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20News%20Feed).
</Note>

## 1. Requirements
**Functional**: publish a post (text, media); view home feed of followed accounts, newest or ranked; paginate (infinite scroll); like/comment counts.
**Non-functional**: feed load p99 < 200 ms; new posts visible to followers within seconds (eventual consistency OK); highly available; read-heavy.

## 2. Estimates
- 300 M DAU; each opens the feed 10×/day → 3 B feed reads/day ≈ **35k QPS** avg, ~100k peak.
- 50 M posts/day ≈ 600 writes/s.
- Average 200 followers → fan-out on write = 50 M × 200 = **10 B feed inserts/day** (~115k/s). Celebrities with 100 M followers make single posts explode.
- Feed cache: keep 500 post IDs × 8 B × 300 M users ≈ **1.2 TB** in Redis (sharded). Fine.

## 3. API
```http
POST /v1/posts {text, mediaIds[]} → 201 {postId}
GET  /v1/feed?cursor=<opaque>&limit=20 → {items:[{post, author, counts}], nextCursor}
```
Cursor-based pagination (by score/ID), not offsets, because the feed changes while scrolling ([Keyset Pagination vs Offset](/Software%20Engineering/Databases/Database%20Round/Keyset%20Pagination%20vs%20Offset)).

## 4. Data model
| Store | Contents |
|---|---|
| Posts DB (sharded by post_id) | post_id (Snowflake, time-sortable), author_id, text, media keys, created_at |
| Social graph (sharded by user_id) | follows(follower_id, followee_id), both directions indexed |
| Feed cache (Redis sorted sets) | `feed:{user_id}` → ZSET of post_id scored by time or rank |
| Counters | likes/comments per post (sharded counters, async aggregation) |
| Object storage + CDN | Media ([Blob Storage & File Uploads](/Software%20Engineering/System%20Design/Blob%20Storage%20%26%20File%20Uploads)) |

## 5. High-level design
{/* diagram source: /diagrams/diagram-394.mmd */}
![diagram](/diagrams/diagram-394.svg)

## 6. Deep dives
### Fan-out strategies
| | Fan-out on write (push) | Fan-out on read (pull) |
|---|---|---|
| Write cost | O(followers) per post | O(1) |
| Read cost | O(1): read precomputed list | O(followees): query and merge |
| Freshness | Delay while fan-out runs | Always fresh |
| Wasted work | Pushes to inactive users | None |
| Bad case | Celebrity with 100 M followers | User following 5,000 accounts |

**Hybrid**: push for authors with < ~10k–100k followers; for celebrities, store their recent posts in a hot cache and **pull and merge** at read time. Skip fan-out to users inactive for 30+ days (rebuild their feed on next login).

### Ranking
1. **Candidate generation**: posts from followed accounts (the feed cache), plus recommendations.
2. **Scoring**: ML model predicts engagement (like, comment, dwell) from features (author affinity, post age, media type, past behavior).
3. **Re-ranking / policies**: diversity (no 5 posts in a row from one author), freshness boosts, integrity demotions.
Serve a ranked snapshot per session so pagination is stable (cursor points into the snapshot).

### Hydration and caching
Feed returns post IDs; the feed service multi-gets post objects, author profiles and counters from caches (`MGET`), falling back to DBs. Counts are approximate and aggregated asynchronously. See [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies).

### Deletes and privacy
Deleting a post or blocking someone: filter at read time (check post still exists, visibility rules) rather than trying to remove from millions of feed caches synchronously; clean up lazily.

## 7. Trade-offs and failure modes
- Feed cache loss → rebuild on read from graph + posts (slow but correct); protect with replicas.
- Fan-out backlog during spikes (major events): feed staleness grows; prioritize active users' fan-out.
- Hot celebrity posts → like counter hot key: sharded counters or in-memory aggregation.

## Senior perspective
- Treat the feed cache as a **derived view** that can always be recomputed.
- Measure **fan-out lag** (post created → visible in followers' feeds) as an SLO.
- Ranking is where product value lives; the infra must support fast experiments (A/B tests on ranking models).
- Cost: fan-out on write multiplies writes; know your follower distribution before choosing thresholds.

## Interview questions
<Accordion title="Push vs pull for feeds?">
Push precomputes each follower's feed at write time (fast reads, expensive for high-follower authors); pull builds the feed at read time (cheap writes, slow reads for users following many). Use a hybrid: push for most, pull for celebrities, merge at read time.
</Accordion>

<Accordion title="How do you paginate a feed that keeps changing?">
Cursor-based pagination using the score/post ID of the last item (or a session snapshot for ranked feeds), never offsets, so new posts don't shift pages and cause duplicates or gaps.
</Accordion>

<Accordion title="What do you store in the feed cache?">
Only post IDs (and maybe scores) in a sorted set capped to a few hundred entries. Post content and user data are hydrated from their own caches so updates/deletes don't need to touch millions of feeds.
</Accordion>

<Accordion title="How do you handle a celebrity posting?">
Don't fan out; put the post in a celebrity-posts cache. When followers load their feed, merge in recent posts from the celebrities they follow.
</Accordion>

## Related
- [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index) · [System Design Interview Framework](/Software%20Engineering/System%20Design/System%20Design%20Interview%20Framework)
- Frontend side: [Case Study - News Feed](/Software%20Engineering/Frontend/15%20System%20Design/Case%20Study%20-%20News%20Feed) · [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index)
- [Caching Strategies](/Software%20Engineering/System%20Design/Caching%20Strategies) · [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) · [Unique ID Generation](/Software%20Engineering/System%20Design/Unique%20ID%20Generation) · [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning) · [Case Study - Notification System](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Notification%20System)

## Sources
- Alex Xu, *System Design Interview*, vol. 1, ch. 11 "Design a News Feed System"
- Martin Kleppmann, *Designing Data-Intensive Applications*, ch. 1 (Twitter home timeline example)
- [Twitter/X: The Infrastructure Behind Twitter: Scale (2017)](https://blog.x.com/engineering/en_us/topics/infrastructure/2017/the-infrastructure-behind-twitter-scale)
- [Meta Engineering: Serving Facebook Multifeed (2015)](https://engineering.fb.com/2015/03/10/production-engineering/serving-facebook-multifeed-efficiency-performance-gains-through-redesign/)
- [X: Twitter's recommendation algorithm (2023, open source)](https://blog.x.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm)
