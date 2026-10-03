---
tags: [databases, databases/mongodb]
aliases: [MongoDB insertOne, MongoDB find, MongoDB Query Operators, elemMatch, MongoDB Unit 7]
updated: 2026-09-27
---
# Unit 07 MongoDB CRUD Operations Insert and Find Documents

> [!tldr] TL;DR
> Insert with `insertOne()` / `insertMany()` (the collection is created on first insert). Read with `find()`, which returns a **cursor**, or `findOne()`. Filters are documents: `{ field: value }` is shorthand for `$eq`; add **comparison** (`$gt`, `$lt`, `$in`, ...), **logical** (`$and`, `$or`, `$not`, `$nor`) and **array** operators (`$elemMatch`, `$all`, `$size`). Use `$elemMatch` when several conditions must hold on the **same** array element.

> Course: MongoDB University, Unit 7 · Prev: [[Unit 06 Connecting to MongoDB in Node.js]] · Next: [[Unit 08 MongoDB CRUD Operations Replace and Delete Documents]]

## What this unit covers
CRUD basics by inserting and finding documents, queries with comparison and logical operators, and querying elements inside arrays.

## Inserting documents
### insertOne()
```js
db.<collection>.insertOne(<document>)
```
If the collection doesn't exist, MongoDB creates it automatically.

```js
db.grades.insertOne({
  student_id: 654321,
  products: [{ type: "exam", score: 90 }, { type: "homework", score: 59 }],
  class_id: 550
})
// { acknowledged: true, insertedId: ObjectId("...") }
```

### insertMany()
```js
db.<collection>.insertMany([ <doc1>, <doc2>, ... ], { ordered: true })
```
With `ordered: true` (default) it stops at the first error; with `ordered: false` it continues and reports all errors (faster for bulk loads).

## Finding documents
### find()
```js
db.<collection>.find()          // all documents, returns a cursor
db.zips.find({ state: "AZ" })   // implicit $eq
db.zips.findOne({ city: "PHOENIX" })
```

To retrieve specific documents use the explicit syntax `{ field: { $eq: <value> } }` or the shorter implicit `{ field: <value> }`:

```js
db.<collection>.find({
  field: { $eq: <value> }
})
```

### $in
Matches documents where the field equals **any** value in the array:

```js
db.<collection>.find({
  field: { $in: [ <value1>, <value2>, ... ] }
})

db.zips.find({ city: { $in: ["PHOENIX", "CHICAGO"] } })
```

### Comparison operators
| Operator | Meaning |
|---|---|
| `$gt` | greater than |
| `$gte` | greater than or equal |
| `$lt` | less than |
| `$lte` | less than or equal |
| `$ne` | not equal (also matches docs **missing** the field) |
| `$nin` | not in array |

```js
db.<collection>.find({ field: { <operator>: <value> } })

db.sales.find({ "items.price": { $gt: 50 } })
db.sales.find({ "customer.age": { $gte: 18, $lt: 30 } })   // range on one field
```

Fields in sub-documents use **dot notation** (the path must be quoted):

```js
db.<collection>.find({
  "subdocument.field": { <operator>: <value> }
})
```

## Querying array elements
A plain filter on an array field matches if **any** element matches: `{ products: "InvestmentStock" }` finds docs whose `products` array contains that value.

`$elemMatch` requires that **one single element** satisfies all the conditions (and that the field is an array):

```js
db.<collection>.find({
  <arrayField>: {
    $elemMatch: { <condition1>, <condition2>, ... }
  }
})
```

```js
db.transactions.find({
  transactions: {
    $elemMatch: {
      amount: { $lte: 4500 },
      transaction_code: "sell"
    }
  }
})
```

Why it matters:

```js
// Any element with amount <= 4500 AND any (possibly different) element with code "sell"
db.transactions.find({ "transactions.amount": { $lte: 4500 }, "transactions.transaction_code": "sell" })
// The SAME element must satisfy both
db.transactions.find({ transactions: { $elemMatch: { amount: { $lte: 4500 }, transaction_code: "sell" } } })
```

Other array operators: `$all` (contains all values), `$size` (exact length).

## Logical operators
### $and
```js
db.<collection>.find({
  $and: [ { <expression> }, { <expression> } ]
})
// Implicit AND: several conditions in ONE filter document
db.<collection>.find({ <expression>, <expression> })
```

Correction to the original note: the implicit form is a single object with several fields, `find({ a: 1, b: 2 })`, not several objects as separate arguments. The second argument of `find()` is a **projection** ([[Unit 09 MongoDB CRUD Operations Modifying Query Results]]).

```js
db.routes.find({ airline: "Southwest Airlines", stops: { $gte: 1 } })
```

Use explicit `$and` when you need the **same operator twice**, e.g. two `$or` clauses (a JS object can't have two `$or` keys):

### $or
```js
db.<collection>.find({
  $or: [ { <expression> }, { <expression> } ]
})
```

### Combined
```js
db.routes.find({
  $and: [
    { $or: [{ dst_airport: "SEA" }, { src_airport: "SEA" }] },
    { $or: [{ "airline.name": "American Airlines" }, { airplane: 320 }] }
  ]
})
```

## Common pitfalls
- Type mismatch: `{ zip: 85001 }` won't match `"85001"`.
- `$ne` / `$nin` also match documents where the field is missing, and usually can't use indexes efficiently.
- Forgetting `$elemMatch` gives "cross-element" matches.
- Unindexed `find()` on a big collection is a **collection scan** ([[Unit 12 MongoDB Indexes]]).

## Senior perspective
- **Bulk inserts**: use `insertMany` with `ordered: false` or `bulkWrite` for throughput; batch sizes are handled by the driver (max 100,000 ops/batch, 48 MB messages).
- **Write concern** controls durability of inserts: `w: "majority"` for important data; `w: 1` is faster but can lose acknowledged writes on failover.
- **Query shape and indexes**: every production filter should be backed by an index; check with `explain("executionStats")` and look at `totalDocsExamined` vs `nReturned`.
- **Injection**: never pass raw user JSON as a filter. `{ password: { $ne: null } }` from a request body is a classic NoSQL injection. Validate types, or use `$eq` explicitly.

## Interview questions
> [!question]- What does find() return?
> A cursor, a pointer to the result set that fetches documents in batches as you iterate. `findOne()` returns a single document or `null`.

> [!question]- What's the difference between filtering on "arr.field" and using $elemMatch?
> Dot-notation conditions can be satisfied by different elements; `$elemMatch` requires one element to satisfy all conditions.

> [!question]- When do you need an explicit $and?
> When combining conditions that would use the same key twice in one object, e.g. two `$or` clauses or two conditions on the same field with the same operator. Otherwise conditions in one filter document are implicitly ANDed.

> [!question]- What is NoSQL injection in MongoDB and how do you prevent it?
> Passing user-controlled objects into a filter lets attackers inject operators (`{ $ne: null }`, `$where`). Prevent it by validating/casting input types, rejecting keys starting with `$`, using `$eq`, and disabling server-side JS.

## Related
- [[MongoDB]] · [[Databases Index]]
- [[Database Indexes]] · [[Query Optimization & EXPLAIN]]

## Sources
- [MongoDB Manual: Insert Documents](https://www.mongodb.com/docs/manual/tutorial/insert-documents/)
- [Query Documents](https://www.mongodb.com/docs/manual/tutorial/query-documents/)
- [Query and Projection Operators](https://www.mongodb.com/docs/manual/reference/operator/query/)
- [$elemMatch](https://www.mongodb.com/docs/manual/reference/operator/query/elemMatch/)
- [MongoDB University: CRUD Insert and Find](https://learn.mongodb.com/)
