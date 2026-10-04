---
title: "Unit 08 MongoDB CRUD Operations Replace and Delete Documents"
tags: [databases, databases/mongodb]
aliases: [MongoDB updateOne, MongoDB replaceOne, MongoDB upsert, findAndModify, MongoDB Unit 8]
updated: 2026-09-27
---
# Unit 08 MongoDB CRUD Operations Replace and Delete Documents

<Note>
**TL;DR**

`replaceOne()` swaps a whole document (keeping `_id`). `updateOne()` / `updateMany()` change specific fields with **update operators** (`$set`, `$inc`, `$push`, `$unset`, ...). `upsert: true` inserts when nothing matches. `findAndModify()` / `findOneAndUpdate()` update **and return** a document atomically. `deleteOne()` / `deleteMany()` remove documents. Each single-document write is atomic; `updateMany` is **not** atomic as a whole.
</Note>

> Course: MongoDB University, Unit 8 · Prev: [Unit 07 MongoDB CRUD Operations Insert and Find Documents](/Software%20Engineering/Databases/MongoDB/Unit%2007%20MongoDB%20CRUD%20Operations%20Insert%20and%20Find%20Documents) · Next: [Unit 09 MongoDB CRUD Operations Modifying Query Results](/Software%20Engineering/Databases/MongoDB/Unit%2009%20MongoDB%20CRUD%20Operations%20Modifying%20Query%20Results)

## What this unit covers
Replacing whole documents, updating individual fields, upserting, and removing documents.

## Replacing documents
```js
db.collection.replaceOne(filter, replacement, options)
```
- `filter`: query that matches the document to replace (first match only).
- `replacement`: the new document. No update operators allowed; `_id` is kept.
- `options`: e.g. `{ upsert: true }`.

```js
db.books.replaceOne(
  { _id: ObjectId("6282afeb441a74a98dbbec4e") },
  { title: "Data Science Fundamentals for Python and MongoDB", isbn: "1484235967", publishedDate: new Date("2018-05-10") }
)
```
Any field not in the replacement is gone afterwards.

## Updating documents
```js
db.collection.updateOne(<filter>, <update>, { <options> })
```
- `filter`: query that matches the document to update.
- `update`: operators describing the change:
  - **`$set`**: adds new fields or replaces the value of existing ones.
  - **`$push`**: appends a value to an array; creates the array if the field is missing.
  - `$inc` (add a number), `$unset` (remove a field), `$addToSet` (append if not present), `$pull` (remove matching elements), `$min`/`$max`, `$currentDate`.
- `options`:
  - **`upsert: true`**: insert a document built from the filter and update if no document matches.

```js
db.podcasts.updateOne(
  { _id: ObjectId("5e8f8f8f8f8f8f8f8f8f8f8f") },
  { $set: { subscribers: 98562 } }
)

db.podcasts.updateOne(
  { title: "The Developer Hub" },
  { $push: { hosts: { $each: ["Nic Raboy", "Jesse Hall"] } } }  // $each pushes several
)
```

Upsert example from the course:

```js
db.birds.updateOne(
  { common_name: "Robin Redbreast" },
  {
    $inc: { sightings: 1 },
    $set: { last_updated: new Date() }
  },
  { upsert: true }
)
```

## Update and return the document: findAndModify()
Correct signature: a **single options document**, not positional arguments.

```js
db.collection.findAndModify({
  query: <filter>,
  sort: <document>,
  update: <update>,       // or remove: true
  new: <boolean>,         // true = return the modified doc, false (default) = the original
  fields: <projection>,
  upsert: <boolean>
})
```

```js
db.podcasts.findAndModify({
  query: { _id: ObjectId("6261a92dfee1ff300dc80bf1") },
  update: { $inc: { subscribers: 1 } },
  new: true
})
```

Why not `updateOne` + `findOne`? Two round trips, and another client could change the document in between. `findAndModify` is one atomic operation. Modern equivalents: `findOneAndUpdate`, `findOneAndReplace`, `findOneAndDelete` (in the Node.js driver, `returnDocument: "after"` instead of `new: true`).

Typical use, an atomic counter / ticket number:

```js
db.counters.findOneAndUpdate(
  { _id: "invoice" },
  { $inc: { seq: 1 } },
  { upsert: true, returnNewDocument: true }
)
```

## Updating many documents
```js
db.collection.updateMany(<filter>, <update>, { <options> })
```
(The original note had `updateMany(filter, update, filter)`; the third argument is options.)

```js
db.books.updateMany(
  { publishedDate: { $lt: new Date("2019-01-01") } },
  { $set: { status: "LEGACY" } }
)
```

`updateMany` is **not all-or-nothing**: each document update is atomic, but if it fails halfway the earlier updates are **not rolled back**, and other clients can see a partially updated set. Use a transaction if you need atomicity across documents.

## Deleting documents
```js
db.collection.deleteOne(<filter>, <options>)
db.collection.deleteMany(<filter>, <options>)

db.podcasts.deleteOne({ _id: ObjectId("6282c9862acb966e76bbf20a") })
db.podcasts.deleteMany({ category: "crime" })
```
`deleteMany({})` removes **all** documents (but keeps the collection and indexes); `drop()` removes the collection.

## Common pitfalls
- Passing a plain object as `update` to `updateOne` (without operators) errors in modern shells; use `replaceOne` for full replacement.
- Upsert with a filter on non-unique fields under concurrency can create duplicates; back it with a **unique index**.
- `$push` onto an array forever creates an unbounded array ([Unit 04 MongoDB Data Modeling Intro](/Software%20Engineering/Databases/MongoDB/Unit%2004%20MongoDB%20Data%20Modeling%20Intro)). Use `$push` with `$slice` to cap it.

## Senior perspective
- **Atomicity**: single-document operations are atomic, including multiple operators on one document. Design so business invariants live in one document.
- **Optimistic concurrency**: include a `version` field in the filter and `$inc` it in the update; `modifiedCount === 0` means someone else won.
- **Retryable writes** cover `updateOne`, `deleteOne`, `findAndModify`, `insert*`, but **not** multi-document `updateMany`/`deleteMany`.
- **Large `updateMany`/`deleteMany`** jobs can saturate the cluster and lag replication; batch them by `_id` ranges, throttle, and consider TTL indexes for expiring data.
- **Soft deletes** (`deletedAt`) are common for audit, but every query then needs that filter and a matching index (partial index).

## Interview questions
<Accordion title="What's the difference between replaceOne and updateOne?">
`replaceOne` replaces the entire document (except `_id`) with a new one. `updateOne` modifies specific fields using update operators and leaves the rest untouched.
</Accordion>

<Accordion title="What does upsert do and what is the risk?">
If no document matches, it inserts a new one built from the filter's equality fields plus the update. Under concurrency two upserts can both insert; a unique index on the filter fields prevents duplicates (the loser gets a duplicate key error and can retry).
</Accordion>

<Accordion title="Why use findAndModify / findOneAndUpdate?">
To modify a document and get it back (before or after) atomically in one operation, e.g. counters, job queues claiming a task, reservations.
</Accordion>

<Accordion title="Is updateMany atomic?">
No. Each document's update is atomic, but the operation as a whole is not isolated or rolled back on failure. Use a multi-document transaction for all-or-nothing behaviour.
</Accordion>

## Related
- [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Unit 10 MongoDB CRUD Operations in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2010%20MongoDB%20CRUD%20Operations%20in%20Node.js)

## Sources
- [MongoDB Manual: Update Documents](https://www.mongodb.com/docs/manual/tutorial/update-documents/)
- [Update Operators](https://www.mongodb.com/docs/manual/reference/operator/update/)
- [db.collection.findAndModify()](https://www.mongodb.com/docs/manual/reference/method/db.collection.findAndModify/)
- [Delete Documents](https://www.mongodb.com/docs/manual/tutorial/remove-documents/)
- [MongoDB University: CRUD Replace and Delete](https://learn.mongodb.com/)
