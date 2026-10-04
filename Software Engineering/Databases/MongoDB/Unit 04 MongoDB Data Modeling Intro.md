---
title: "Unit 04 MongoDB Data Modeling Intro"
tags: [databases, databases/mongodb]
aliases: [MongoDB Data Modeling, Embedding vs Referencing, MongoDB Schema Design, MongoDB Schema Patterns, MongoDB Unit 4]
updated: 2026-09-27
---
# Unit 04 MongoDB Data Modeling Intro

<Note>
**TL;DR**

Data modeling decides how data is stored and how entities relate. MongoDB's rule of thumb: **data that is accessed together should be stored together**. You model relationships (one-to-one, one-to-many, many-to-many) by **embedding** related data in a document or **referencing** it by `_id` in another collection. Embedding gives fast single-read access and atomic updates; referencing avoids duplication and unbounded growth. Watch for anti-patterns like unbounded arrays, bloated documents and too many collections.
</Note>

> Course: MongoDB University, Unit 4 · Prev: [Unit 03 MongoDB and the Document Model](/Software%20Engineering/Databases/MongoDB/Unit%2003%20MongoDB%20and%20the%20Document%20Model) · Next: [Unit 05 Connecting to a MongoDB Database](/Software%20Engineering/Databases/MongoDB/Unit%2005%20Connecting%20to%20a%20MongoDB%20Database)

## What data modeling is
Data modeling is the process of defining how data is stored and what relationships exist among your entities. The organization of the data is the **schema**.

To design a schema, start from the application:
1. What does my application do?
2. What data will I store?
3. How will users access this data?
4. What data will be valuable to me?

<Tip>
**Data that is accessed together should be stored together.**


</Tip>

You can still normalize data with references when that fits better. In MongoDB you design **for the application's queries first**, unlike relational design, which starts from the entities.

## Types of relationships
![Screenshot_2023-04-22_at_7.12.39_PM](/Attachments/Screenshot_2023-04-22_at_7.12.39_PM.png)

### One-to-one
An entity in one set is connected to exactly one entity in another set (a user and their profile settings). Usually modelled inside a **single document**:

```js
{ _id: 1, name: "Ali", address: { street: "Main St 1", city: "Cairo" } }
```

### One-to-many
An entity in one set is connected to any number of entities in another set (a movie and its cast). One way to model it is a **nested array**:

![Screenshot_2023-04-22_at_7.14.56_PM](/Attachments/Screenshot_2023-04-22_at_7.14.56_PM.png)

```js
{ _id: 10, title: "The Matrix", cast: [ { name: "Keanu Reeves" }, { name: "Carrie-Anne Moss" } ] }
```

### Many-to-many
Any number of entities in one set are connected to any number in another (students and courses). Model it with arrays of references on one or both sides, or embed a small, stable subset.

```js
// students
{ _id: "s1", name: "Sara", course_ids: ["c1", "c2"] }
// courses
{ _id: "c1", title: "Databases", student_count: 120 }
```

## Two ways to model relationships
![Untitled 34](/Attachments/Untitled%2034.png)

![Screenshot_2023-04-22_at_7.16.53_PM](/Attachments/Screenshot_2023-04-22_at_7.16.53_PM.png)

### Embedding
Put related data **inside** the document.

- Works for **all** relationship types: one-to-one, one-to-many and many-to-many. (The original note said "only one-to-many or many-to-many"; embedding is also the default for one-to-one.) It fits best when the child data is read with the parent, belongs to it, and is **bounded** in size.
- Avoids application-side joins, gives better read performance (one read), and lets you update related data in a **single atomic write**.

<Warning>
**Costs of embedding**

- Documents can become **large**, and large documents make reads slower: the whole document is loaded into memory (the WiredTiger cache), which can slow the application for end users.
- Continuously adding data without limit creates an **unbounded document** that can grow past the **16 MB** BSON limit. Updates to huge documents are also expensive because the storage engine rewrites the document.
- **Both large documents and unbounded documents are schema anti-patterns.**
</Warning>

### Referencing
Store the `_id` of a document from another collection as a **link** between the two. This is also called **linking** or **data normalization**.

```js
// orders
{ _id: 501, customer_id: 42, total: NumberDecimal("99.90") }
// customers
{ _id: 42, name: "Ali" }
```

- No duplication, documents stay small, each side can grow independently.
- Reading related data needs a second query or a `$lookup` join ([Unit 11 MongoDB Aggregation](/Software%20Engineering/Databases/MongoDB/Unit%2011%20MongoDB%20Aggregation)), which costs extra resources and impacts read performance.

| | Embedding | Referencing |
|---|---|---|
| Reads | one query | 2 queries or `$lookup` |
| Atomic updates | yes, single document | needs a transaction across docs |
| Duplication | possible | none |
| Growth | bounded only | unbounded OK |
| Best for | "contains"/owned data read together | large, shared, or independently queried data |

## What to avoid
When creating a data model, avoid:
1. Unbounded documents when embedding.
2. Poor query performance.
3. Poor write performance.
4. Too much memory being used.

## Schema design patterns and anti-patterns
**Schema design patterns** are guidelines that help developers plan, organize and model data.

Common schema **anti-patterns**:
1. Massive (unbounded) arrays
2. A massive number of collections
3. Bloated documents
4. Unnecessary indexes
5. Queries without indexes
6. Data that's accessed together but stored in different collections

You can spot them in Atlas using:
1. **Data Explorer** (schema and index views)
2. **Performance Advisor** (slow queries, suggested and redundant indexes)

### Common design patterns (beyond the course)
| Pattern | Idea | Example |
|---|---|---|
| **Extended reference** | copy a few frequently read fields of the referenced doc | order stores `customer: { _id, name }` |
| **Subset** | embed only the most-used part of a big array | product keeps the 10 latest reviews; the rest in `reviews` |
| **Bucket** | group many small items into fixed-size documents | IoT readings, one doc per sensor per hour |
| **Computed** | precompute values on write instead of every read | `review_count`, `avg_rating` updated with `$inc` |
| **Outlier** | handle the rare huge case separately | a celebrity's followers overflow into extra docs with `has_extras: true` |
| **Polymorphic** | different shapes in one collection with a `type` field | `products` with books and shirts |
| **Schema versioning** | `schemaVersion` field, migrate lazily | v1 `phone`, v2 `contacts: []` |
| **Attribute** | turn many similar fields into an array of k/v pairs to index once | `specs: [{k:"color", v:"red"}]` |

Bucket example:

```js
{
  sensor_id: "s-17",
  start: ISODate("2026-09-27T10:00:00Z"),
  count: 3,
  readings: [
    { t: ISODate("2026-09-27T10:00:05Z"), temp: 21.4 },
    { t: ISODate("2026-09-27T10:00:10Z"), temp: 21.5 },
    { t: ISODate("2026-09-27T10:00:15Z"), temp: 21.5 }
  ]
}
// append with an upsert, capping the bucket size
db.readings.updateOne(
  { sensor_id: "s-17", count: { $lt: 200 } },
  { $push: { readings: { t: new Date(), temp: 21.6 } }, $inc: { count: 1 }, $setOnInsert: { start: new Date() } },
  { upsert: true }
)
```

For time-series data specifically, prefer native **time series collections** (MongoDB 5.0+), which apply bucketing for you.

## Senior perspective
- **Model from the workload**: list the top queries with their frequency and latency targets, then design documents so each hot query is one indexed read.
- **Read/write ratio drives duplication**: duplicate (extended reference, computed) when reads dominate and the copied data rarely changes; accept the cost of updating copies.
- **Unbounded growth is the most common production failure**: arrays that grow per event (comments, logs, followers) eventually hit 16 MB or make every update slow. Cap them (subset/bucket) or reference.
- **Atomicity boundary = document**: whatever must change together atomically should ideally live in one document, so you do not need multi-document transactions ([Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID)).
- **Sharding**: embedded data co-locates on one shard automatically; references across collections may cross shards, making `$lookup` and transactions more expensive ([Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning)).
- Compare with relational normalization in [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization).

## Interview questions
<Accordion title="When do you embed and when do you reference?">
Embed when the data is read together, owned by the parent, and bounded (one-to-one, one-to-few). Reference when the related data is large, unbounded, shared by many parents, or queried independently. Hybrid patterns (extended reference, subset) sit in between.
</Accordion>

<Accordion title="What is an unbounded array and why is it a problem?">
An array that grows without limit (e.g. all comments on a post). It can exceed the 16 MB limit, makes documents slow to read and update, and bloats indexes on the array (multikey). Fix with the subset or bucket pattern, or move items to their own collection.
</Accordion>

<Accordion title="Explain the bucket and computed patterns.">
Bucket groups many small related items (time-series points) into one document per time window or size, cutting document count and index size. Computed stores derived values (totals, counts) updated at write time so reads don't recompute them.
</Accordion>

<Accordion title="How would you model a many-to-many relationship like students and courses?">
Arrays of references on one or both sides (`course_ids` on students, maybe `student_ids` on courses if bounded), optionally with an extended reference of the course title. If enrollment has its own data (grade, date), use a separate `enrollments` collection, like a join table.
</Accordion>

<Accordion title="How do you find schema problems in an existing Atlas deployment?">
Performance Advisor (slow queries, index suggestions), the schema anti-pattern checks in Data Explorer, `explain()` on hot queries, and collection stats for document sizes and index usage (`$indexStats`).
</Accordion>

## Related
- [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization) · [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types) · [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning)

## Sources
- [MongoDB Manual: Data Modeling](https://www.mongodb.com/docs/manual/data-modeling/)
- [Schema design patterns (building with patterns)](https://www.mongodb.com/docs/manual/data-modeling/design-patterns/)
- [Schema design anti-patterns](https://www.mongodb.com/developer/products/mongodb/schema-design-anti-pattern-summary/)
- [Time series collections](https://www.mongodb.com/docs/manual/core/timeseries-collections/)
- [MongoDB University: MongoDB Data Modeling Intro](https://learn.mongodb.com/)
