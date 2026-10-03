---
tags: [databases, databases/mongodb]
aliases: [MongoDB Indexes, Compound Index, Multikey Index, ESR Rule, MongoDB explain, MongoDB Unit 12]
updated: 2026-09-27
---
# Unit 12 MongoDB Indexes

> [!tldr] TL;DR
> An **index** is an ordered data structure (a B-tree) holding a small part of each document plus a pointer to it, so queries can find and sort data without reading the whole collection. Without one, MongoDB does a **collection scan** (`COLLSCAN`) and sorts in memory. Types covered: **single-field**, **compound** (usable by queries on its **prefix**; order fields by the **ESR rule: Equality, Sort, Range**) and **multikey** (on arrays). Check usage with `explain()`. Indexes speed reads but cost **writes**, memory and disk, so drop the ones you don't use.

> Course: MongoDB University, Unit 12 · Prev: [[Unit 11 MongoDB Aggregation]] · Next: [[Unit 13 MongoDB Atlas Search]]

## What this unit covers
How indexes support efficient queries, their trade-offs, single-field and compound indexes, multikey indexes, checking whether queries use indexes, and deleting indexes.

## What indexes are
- A special **ordered** data structure that stores a small portion of the data (the indexed fields) with a pointer to the document's location.
- Because it's ordered, it is easy to search efficiently (and to return results already sorted).
- **Without indexes**: MongoDB reads every document (**collection scan**) to match a query and sorts results in memory.
- **With indexes**: MongoDB fetches only the documents the index identifies.
- Every frequent query should be supported by an index.
- Cost: every insert/update/delete must also update each index, so indexes add **write** overhead and use RAM and disk. Too many indexes are bad; delete unnecessary ones.
- Every collection has a unique index on `_id` by default.

## 1. Single-field indexes
```js
db.customers.createIndex({ birthdate: 1 })                // 1 = ascending, -1 = descending
db.customers.createIndex({ email: 1 }, { unique: true })  // unique constraint: no two docs share a value
```

List indexes:

```js
db.customers.getIndexes()
```

See whether a query uses an index with `explain()`; look at the **winning plan** stages:

```js
db.customers.explain().find({ birthdate: { $gt: ISODate("1995-08-01") } })
db.customers.find({ email: "a@b.com" }).explain("executionStats")
```

| Stage | Meaning |
|---|---|
| `IXSCAN` | the query uses an index (and which one) |
| `COLLSCAN` | full collection scan, no index used |
| `FETCH` | documents are being read from the collection |
| `SORT` | documents are sorted **in memory** (no index for the sort) |
| `PROJECTION_COVERED` | everything needed came from the index, no documents fetched (a **covered query**) |

In `executionStats`, compare `nReturned`, `totalKeysExamined` and `totalDocsExamined`: ideally all three are close.

## 2. Multikey indexes
- An index on an **array** field. MongoDB creates one index entry per unique array element. Created with the same `createIndex()`; it becomes multikey automatically.
- Works on arrays of scalars, of sub-documents, and nested arrays. Can be single-field or compound.
- A compound index can include **at most one** array field per document.
- Multikey index scans are usually followed by a `FETCH` stage (the index can't cover the query).

```js
db.customers.createIndex({ accounts: 1 })
db.customers.find({ accounts: 627788 }).explain()   // IXSCAN with isMultiKey: true
```

![[Screenshot_2023-04-27_at_3.31.33_PM.png]]

## 3. Compound indexes
An index on several fields. It supports queries that match on a **prefix** of the index fields.

```js
db.accounts.createIndex({ active: 1, birthday: -1, name: 1 })
```

Prefixes of this index: `{ active }`, `{ active, birthday }`, `{ active, birthday, name }`.

```js
// Can use the index (they include the prefix field "active")
db.accounts.find({ active: true })
db.accounts.find({ active: true, birthday: { $lt: ISODate("2000-01-01") } })
db.accounts.find({ active: true }).sort({ birthday: -1 })
db.accounts.find({ active: true, name: "Ali" })   // uses prefix { active }, then filters name in the index
```

> [!note] Correction
> The original note listed `db.accounts.find({name: 'Ali'}, {active: true})` as using the prefix. The second argument of `find()` is a **projection**, not a filter, so that query filters only on `name` and **cannot** use the index prefix. Use `find({ active: true, name: 'Ali' })` instead.

Queries that don't include the first field can't use the compound index efficiently, even if they include other indexed fields:

```js
db.accounts.find({ name: "Ali" })
db.accounts.find({ birthday: ISODate("1998-04-30") })
db.accounts.find({ name: "Ali", birthday: ISODate("1998-04-30") })
```

### Field order: the ESR rule
Order compound index fields as **Equality, Sort, Range**:

1. **Equality** fields first: exact matches narrow the scan to fewer entries, reducing work.
2. **Sort** fields next: the index already returns documents in order, so there's no in-memory sort.
3. **Range** fields last (`$gt`, `$lt`, `$in` with many values, regex): putting them before the sort field would break the index order and force an in-memory sort.

```js
// Query: active users with a name after "M", newest birthday first
db.accounts.find({ active: true, name: { $gt: "M" } }).sort({ birthday: -1 })

// ESR index: E = active, S = birthday, R = name
db.accounts.createIndex({ active: 1, birthday: -1, name: 1 })
```

- **Sort direction matters** for multi-field sorts: an index `{ a: 1, b: -1 }` supports `sort({ a: 1, b: -1 })` and its inverse `sort({ a: -1, b: 1 })`, but not `sort({ a: 1, b: 1 })`.
- **Covered queries**: if you project only indexed fields and exclude `_id`, the plan shows `PROJECTION_COVERED` and is faster because no documents are fetched.

```js
db.accounts.find({ active: true }, { _id: 0, active: 1, birthday: 1, name: 1 })
```

## Deleting indexes
- First make sure the index isn't being used (`$indexStats`, Performance Advisor).
- You can delete any index except the one on `_id`.
- Recreating an index later costs time and resources, so consider **hiding** it first.

```js
db.collection.dropIndex("active_1_birthday_-1_name_1")   // by name or key pattern
db.collection.dropIndexes()                              // all except _id
db.collection.dropIndexes(["index1_1", "index2_1"])       // a list

db.collection.hideIndex("active_1_birthday_-1_name_1")   // planner ignores it, still maintained
db.collection.unhideIndex("active_1_birthday_-1_name_1") // instant, no rebuild
```

Unhiding is much faster than rebuilding, so hide, watch performance, then drop.

```js
db.accounts.aggregate([{ $indexStats: {} }])   // accesses.ops per index since last restart
```

## Other index types (beyond the course)
| Type | Use |
|---|---|
| Text | legacy full-text (prefer Atlas Search) |
| 2dsphere | geospatial queries |
| Hashed | even distribution for hashed sharding |
| TTL | auto-expire documents (`expireAfterSeconds`) |
| Partial | index only docs matching a filter (smaller) |
| Sparse | skip docs missing the field |
| Wildcard | unknown/dynamic field names |
| Clustered | collection stored in `_id` order (5.3+) |

> [!info]+ 🆕 2025–2026
> **MongoDB 8.3 (May 2026)**: multi-planning with a **cost-based ranker (CBR)** as backup is now the default plan-selection mechanism for eligible queries, with new `serverStatus` and `$queryStats` metrics. Plans you see in `explain()` may differ from older versions. ([8.3 release notes](https://www.mongodb.com/docs/manual/release-notes/8.3/))

## Senior perspective
- **Working set**: indexes should fit in RAM (the WiredTiger cache). If they don't, every query pays disk I/O.
- **Index builds**: since 4.2, builds hold an exclusive lock only briefly at start and end, but still consume CPU/IO and replicate to secondaries. On large collections, build during low traffic; Atlas can do **rolling index builds** (one node at a time).
- **Write amplification**: each index adds work to every write. High-ingest collections should have few, carefully chosen indexes.
- **Low-selectivity fields** (booleans, status with 3 values) make poor leading fields unless combined with others or used in a partial index.
- **Unique indexes** are how you enforce invariants (idempotency keys, emails) under concurrency; application checks alone race.
- **Sharding**: the shard key needs a supporting index; queries without the shard key scatter to all shards. See [[Sharding & Partitioning]].
- Use Atlas Performance Advisor and `$queryStats` to find missing and unused indexes; see [[Database Indexes]] and [[Query Optimization & EXPLAIN]].

## Interview questions
> [!question]- What does the ESR rule say?
> Order compound index fields Equality first, then Sort, then Range. Equality narrows the scan, sort fields next let the index return ordered results, and range fields last avoid breaking that order and causing in-memory sorts.

> [!question]- What is an index prefix and why does it matter?
> The leading subset of a compound index's fields. A query can use the compound index only if it filters (or sorts) on a prefix, e.g. `{a, b, c}` serves `{a}`, `{a, b}`, `{a, b, c}` but not `{b}` or `{c}` alone.

> [!question]- How do you tell if a query uses an index?
> Run `explain("executionStats")`. `IXSCAN` means an index is used; `COLLSCAN` means a full scan; a `SORT` stage means an in-memory sort. Compare keys examined and docs examined with docs returned.

> [!question]- What is a multikey index and what are its limits?
> An index on an array field with one entry per element. A compound index can include at most one array field per document, and multikey indexes can't produce covered queries on the array field.

> [!question]- What is a covered query?
> One where the filter and projection use only fields in a single index (and `_id` is excluded or indexed), so MongoDB answers from the index without fetching documents.

> [!question]- How would you safely remove an index you think is unused?
> Check `$indexStats` and Performance Advisor over a representative period, hide it with `hideIndex()`, monitor latency, then drop it. Unhide instantly if something regresses.

## Related
- [[MongoDB]] · [[Databases Index]]
- [[Database Indexes]] · [[Query Optimization & EXPLAIN]] · [[Sharding & Partitioning]]

## Sources
- [MongoDB Manual: Indexes](https://www.mongodb.com/docs/manual/indexes/)
- [Compound Indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/)
- [The ESR (Equality, Sort, Range) Guideline](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/)
- [Multikey Indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-multikey/)
- [Explain Results](https://www.mongodb.com/docs/manual/reference/explain-results/)
- [Hidden Indexes](https://www.mongodb.com/docs/manual/core/index-hidden/)
- [MongoDB 8.3 release notes](https://www.mongodb.com/docs/manual/release-notes/8.3/)
- [MongoDB University: MongoDB Indexes](https://learn.mongodb.com/)
