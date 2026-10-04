---
title: "Unit 09 MongoDB CRUD Operations Modifying Query Results"
tags: [databases, databases/mongodb]
aliases: [MongoDB Cursor, MongoDB sort limit, MongoDB Projection, countDocuments, MongoDB Unit 9]
updated: 2026-09-27
---
# Unit 09 MongoDB CRUD Operations Modifying Query Results

<Note>
**TL;DR**

`find()` returns a **cursor**, a pointer to the result set. Chain cursor methods to shape results: `.sort({ field: 1 | -1 })`, `.limit(n)` (takes a **number**), `.skip(n)`. Pass a **projection** as the second argument of `find()` to return only some fields (1 = include, 0 = exclude; don't mix, except `_id`). Count with `countDocuments(filter)`. Sorting on indexed fields avoids in-memory sorts.
</Note>

> Course: MongoDB University, Unit 9 · Prev: [Unit 08 MongoDB CRUD Operations Replace and Delete Documents](/Software%20Engineering/Databases/MongoDB/Unit%2008%20MongoDB%20CRUD%20Operations%20Replace%20and%20Delete%20Documents) · Next: [Unit 10 MongoDB CRUD Operations in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2010%20MongoDB%20CRUD%20Operations%20in%20Node.js)

## Cursors
A **cursor** is a pointer to the result set of a query. The server returns results in **batches**; the client fetches more (`getMore`) as you iterate. In `mongosh`, type `it` to see the next batch. Cursors time out after 10 minutes of inactivity by default.

## Sorting: cursor.sort()
```js
db.collection.find(<filter>).sort({ <field>: 1 })   // 1 = ascending, -1 = descending
```

```js
db.companies.find({ category_code: "music" }).sort({ name: 1 })
db.companies.find().sort({ founded_year: -1, name: 1 })   // multi-key sort, order matters
```

## Limiting: cursor.limit()
`limit()` takes a **number**, not an object (the original note had `.limit({ ... })`):

```js
db.collection.find(<filter>).limit(<number>)

db.companies.find({ category_code: "music" })
  .sort({ number_of_employees: -1, _id: 1 })
  .limit(3)
```

Order of chaining doesn't matter in the shell: the server always applies **sort, then skip, then limit**.

## Course example: sort + limit + $elemMatch
```js
db.sales.find({
  storeLocation: "London",
  items: { $elemMatch: { name: { $in: ["paper", "laptop", "backpack", "printer"] } } }
}).sort({ saleDate: -1 }).limit(3)
```
The three most recent London sales that included at least one of those items.

## Projection: return only some fields
Projections reduce bandwidth and memory by returning only the fields you need. The projection is the **second argument** of `find()` (not inside the filter object as the original note showed):

```js
db.collection.find(<filter>, <projection>)
```

- `1` includes a field, `0` excludes it.
- You can't mix inclusion and exclusion in one projection, **except `_id`**, which is included by default and can be excluded in an inclusion projection.

```js
// inclusion
db.inspections.find({ sector: "Restaurant - 818" }, { business_name: 1, result: 1 })
// inclusion + exclude _id
db.sales.find(
  { "customer.age": { $lt: 30 }, "customer.satisfaction": { $gt: 3 } },
  { "customer.age": 1, "customer.satisfaction": 1, _id: 0 }
)
// exclusion
db.inspections.find({ result: { $in: ["Pass", "Warning"] } }, { date: 0, "address.zip": 0 })
```

## Counting documents
```js
db.collection.countDocuments(<filter>, <options>)

db.sales.countDocuments({ storeLocation: "Denver", couponUsed: true })
db.trips.countDocuments({ tripduration: { $gt: 120 }, usertype: "Subscriber" })
```

`countDocuments` runs an aggregation and is exact. `estimatedDocumentCount()` uses collection metadata: instant, no filter, approximate. The old `count()` is deprecated in drivers.

## Pagination
```js
// Offset pagination: simple, but skip(n) still walks n entries, so deep pages get slow
db.posts.find().sort({ _id: -1 }).skip(40).limit(20)

// Keyset (range) pagination: fast at any depth, uses the index
db.posts.find({ _id: { $lt: lastSeenId } }).sort({ _id: -1 }).limit(20)
```

## Common pitfalls
- Sorting on an unindexed field of a big result: in-memory `SORT` stage, limited to 100 MB unless it spills to disk.
- Sorting on a non-unique field for pagination: ties produce duplicates or gaps between pages; add `_id` as a tiebreaker.
- Large `skip()` values for pagination.

## Senior perspective
- Design indexes so `sort` is served by the index (see ESR in [Unit 12 MongoDB Indexes](/Software%20Engineering/Databases/MongoDB/Unit%2012%20MongoDB%20Indexes)); an explain plan with a `SORT` stage on a hot query is a red flag.
- **Covered queries**: if the filter and projection use only indexed fields (and exclude `_id` or index it), MongoDB answers from the index without fetching documents.
- Prefer keyset pagination for feeds and APIs; use `countDocuments` sparingly on large collections (it scans matching documents/index entries). Cache or estimate totals.
- Close or exhaust cursors; long-lived cursors hold server resources.

## Interview questions
<Accordion title="What is a cursor and why does MongoDB use one?">
A pointer to a query's result set that streams documents in batches. It avoids loading the entire result into memory on client or server at once.
</Accordion>

<Accordion title="In what order does MongoDB apply sort, skip and limit?">
Always sort first, then skip, then limit, regardless of the order you chain them.
</Accordion>

<Accordion title="Why is skip-based pagination slow and what's the alternative?">
`skip(n)` must walk past n index entries or documents, so cost grows with page depth. Keyset pagination filters on the last seen sort key (`_id < lastId`) and uses the index directly.
</Accordion>

<Accordion title="Can you mix 1 and 0 in a projection?">
No, except for `_id`, which you can exclude in an inclusion projection.
</Accordion>

<Accordion title="countDocuments vs estimatedDocumentCount?">
`countDocuments(filter)` is accurate and supports filters but scans. `estimatedDocumentCount()` reads collection metadata, is O(1), takes no filter, and may be inaccurate after unclean shutdowns or on sharded clusters with orphans.
</Accordion>

## Related
- [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Database Indexes](/Software%20Engineering/Databases/Database%20Round/Indexes/Database%20Indexes) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)

## Sources
- [cursor.sort()](https://www.mongodb.com/docs/manual/reference/method/cursor.sort/)
- [cursor.limit()](https://www.mongodb.com/docs/manual/reference/method/cursor.limit/)
- [Project Fields to Return from Query](https://www.mongodb.com/docs/manual/tutorial/project-fields-from-query-results/)
- [db.collection.countDocuments()](https://www.mongodb.com/docs/manual/reference/method/db.collection.countDocuments/)
- [MongoDB University: Modifying Query Results](https://learn.mongodb.com/)
