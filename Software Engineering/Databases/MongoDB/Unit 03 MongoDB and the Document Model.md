---
title: "Unit 03 MongoDB and the Document Model"
tags: [databases, databases/mongodb]
aliases: [MongoDB Document Model, BSON, ObjectId, Flexible Schema, MongoDB Unit 3]
updated: 2026-09-27
---
# Unit 03 MongoDB and the Document Model

<Note>
**TL;DR**

A **document** is MongoDB's basic unit of data: an ordered set of field/value pairs, displayed as JSON and stored as **BSON** (binary JSON with extra types such as `Date`, `ObjectId`, `Decimal128`, `Int32/Int64`). Documents live in **collections**, collections live in **databases**. Every document needs a unique `_id`, auto-generated as an `ObjectId` if you omit it. The schema is **flexible** (documents in one collection can differ), and you can add **schema validation** when you need guarantees.
</Note>

> Course: MongoDB University, Unit 3 · Prev: [Unit 02 Getting Started with MongoDB Atlas](/Software%20Engineering/Databases/MongoDB/Unit%2002%20Getting%20Started%20with%20MongoDB%20Atlas) · Next: [Unit 04 MongoDB Data Modeling Intro](/Software%20Engineering/Databases/MongoDB/Unit%2004%20MongoDB%20Data%20Modeling%20Intro)

## What this unit covers
How MongoDB stores data and fits into Atlas, common use cases, supported value types, how unique identifiers are created, what a flexible schema means, and managing databases, collections and documents in the Atlas **Data Explorer**.

## Core terms
- **MongoDB** is a general-purpose document database. You model data of any shape as JSON-like documents and use one format both to store and to query.
- **Document**: the basic data unit.
- **Collection**: a group of documents. They don't have to follow the same schema.
- **Database**: a container of collections.

{/* diagram source: /diagrams/diagram-129.mmd */}
![diagram](/diagrams/diagram-129.svg)

## Document structure
```json
{
  "key": "value",
  "key2": 42,
  "nested": { "a": 1 },
  "list": [1, 2, 3]
}
```

A realistic example in `mongosh` syntax (shell helpers like `ObjectId()` and `ISODate()` are JavaScript, not JSON):

```js
{
  _id: ObjectId("66f5e3a1c2b4a1d2e3f40001"),
  account_id: "MDB829001337",
  holder: { name: "Linus Torvalds", country: "FI" },
  balance: NumberDecimal("50352.34"),   // Decimal128 for money
  opened: ISODate("2023-04-22T00:00:00Z"),
  tags: ["checking", "premium"],
  active: true
}
```

## BSON
- Documents are **stored and transmitted as BSON** and **displayed as JSON** (or Extended JSON) in tools.
- BSON is a binary encoding of JSON-like documents that adds types JSON lacks: `Date`, `ObjectId`, `Int32`, `Int64`, `Double`, `Decimal128`, `Binary`, `Timestamp`, `Regex`, and more.
- It is length-prefixed, so servers can skip over fields quickly.
- Maximum BSON document size is **16 MB**.

| Type | JSON | BSON |
|---|---|---|
| Dates | string only | native `Date` |
| Numbers | one "number" type | `Int32`, `Int64`, `Double`, `Decimal128` |
| Binary | base64 string | native `BinData` |
| Unique id | none | `ObjectId` |

## `_id` and ObjectId
- Every document **must** have a unique `_id`; there is always a unique index on it.
- If you omit it, the driver or server generates an **ObjectId**: 12 bytes = 4-byte timestamp + 5-byte random value + 3-byte counter. It is roughly sortable by creation time: `ObjectId("...").getTimestamp()`.
- You can use your own `_id` (a string, number, or compound object) if you have a natural unique key.

## Flexible schema and polymorphic data
- MongoDB supports **polymorphic data**: one collection can hold several shapes of document (e.g. `products` where books have `isbn` and shirts have `size`).
- New fields can be added without a migration. Old documents simply lack them.
- To control the structure, add optional **schema validation**:

```js
db.createCollection("accounts", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["account_id", "balance"],
      properties: {
        account_id: { bsonType: "string" },
        balance: { bsonType: ["decimal", "double", "int", "long"], minimum: 0 }
      }
    }
  },
  validationAction: "error"   // or "warn"
})
```

## Data Explorer
In the Atlas UI, **Data Explorer** (Browse Collections) lets you create databases and collections, insert, edit and delete documents, run filters, and build aggregation pipelines without the shell.

## Common pitfalls
- Storing money as `Double` (floating-point rounding); use `Decimal128` or integer cents.
- Storing dates as strings: range queries and sorting break. Use `Date`.
- Inconsistent field names or types across documents (`userId` vs `user_id`, `"42"` vs `42`): queries silently miss documents.

## Senior perspective
- "Flexible schema" moves schema enforcement to the application. In teams, add `$jsonSchema` validation (start with `validationAction: "warn"`), and version documents with a `schemaVersion` field (the **schema versioning pattern**) so code can migrate lazily.
- The **16 MB** limit is a design smell detector: if a document can approach it, your model probably has an unbounded array ([Unit 04 MongoDB Data Modeling Intro](/Software%20Engineering/Databases/MongoDB/Unit%2004%20MongoDB%20Data%20Modeling%20Intro)).
- Field names are stored in every document; very long names cost storage and cache on huge collections (compression helps, but be deliberate).
- ObjectIds are monotonically increasing per process, which makes them a poor **shard key** on their own (all inserts hit one chunk). See [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning).

## Interview questions
<Accordion title="What is BSON and why does MongoDB use it instead of JSON?">
Binary JSON: a binary, length-prefixed encoding that is fast to traverse and supports extra types (dates, 64-bit ints, Decimal128, ObjectId, binary). It is MongoDB's storage and wire format. Tools show it as JSON/Extended JSON.
</Accordion>

<Accordion title="What is an ObjectId made of?">
12 bytes: a 4-byte Unix timestamp (seconds), a 5-byte random value unique to the machine/process, and a 3-byte incrementing counter. So it is unique without coordination and roughly time-ordered.
</Accordion>

<Accordion title="Is MongoDB schemaless? How do you enforce a schema?">
It has a flexible schema, not no schema. Use `$jsonSchema` validators on the collection (with `validationLevel` and `validationAction`), plus application-level models (e.g. Mongoose, Zod).
</Accordion>

<Accordion title="What is the maximum document size and what do you do if you need more?">
16 MB. Redesign (split into referenced documents, bucket pattern) or, for files, use GridFS or object storage.
</Accordion>

## Related
- [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types) · [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization)

## Sources
- [MongoDB Manual: Documents](https://www.mongodb.com/docs/manual/core/document/)
- [BSON Types](https://www.mongodb.com/docs/manual/reference/bson-types/)
- [Schema Validation](https://www.mongodb.com/docs/manual/core/schema-validation/)
- [bsonspec.org](https://bsonspec.org/)
- [MongoDB University: MongoDB and the Document Model](https://learn.mongodb.com/)
