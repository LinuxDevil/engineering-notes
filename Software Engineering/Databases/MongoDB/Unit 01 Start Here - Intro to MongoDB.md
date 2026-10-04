---
title: "Unit 01 Start Here - Intro to MongoDB"
tags: [databases, databases/mongodb]
aliases: [Intro to MongoDB, MongoDB Unit 1, mongosh basics, What is MongoDB]
updated: 2026-09-27
---
# Unit 01 Start Here - Intro to MongoDB

<Note>
**TL;DR**

MongoDB is a general-purpose **document database**: data is stored as JSON-like documents (BSON on disk and on the wire) grouped into collections, instead of rows in fixed tables. This first unit introduces the course, its **Instruqt** in-browser lab, and the `mongosh` shell commands you use in every later unit (`db.help()`, `show dbs`, `use`, `show collections`, `db.collection.help()`).
</Note>

> Course: MongoDB University, Unit 1 · Prev: none · Next: [Unit 02 Getting Started with MongoDB Atlas](/Software%20Engineering/Databases/MongoDB/Unit%2002%20Getting%20Started%20with%20MongoDB%20Atlas)

## Original notes
In this section, we learn how to use the IDE instruqt

We also tackle the db.help() command

## The lab environment
- The course labs run in **Instruqt**, a browser-based sandbox: a terminal tab (with `mongosh` and the Atlas CLI preinstalled) and an editor tab. Nothing to install locally.
- Each lab gives you a task, you run commands, then click **Check** to validate.
- To practise outside the course: a free Atlas **M0** cluster, or a local server with Docker: `docker run -d -p 27017:27017 mongodb/mongodb-community-server:latest`, then `mongosh`.

## What MongoDB is
- A **document database**: each record is a document of field/value pairs, with nested objects and arrays allowed.
- Documents in one **collection** do not have to share a schema (flexible schema), but you can enforce rules with schema validation ([Unit 03 MongoDB and the Document Model](/Software%20Engineering/Databases/MongoDB/Unit%2003%20MongoDB%20and%20the%20Document%20Model)).
- Distributed by design: replica sets for high availability, sharding for horizontal scale.
- Offered as self-managed **Community** and **Enterprise** Server, and as the managed cloud service **Atlas** ([Unit 02 Getting Started with MongoDB Atlas](/Software%20Engineering/Databases/MongoDB/Unit%2002%20Getting%20Started%20with%20MongoDB%20Atlas)).

### Document model vs relational, at a glance
| Relational (SQL) | MongoDB |
|---|---|
| Database | Database |
| Table | Collection |
| Row | Document |
| Column | Field |
| Fixed schema, `ALTER TABLE` to change | Flexible schema, optional validation |
| Normalize, then `JOIN` | Embed data read together; `$lookup` when needed |
| Primary key | `_id` (often an `ObjectId`) |
| SQL | MQL (MongoDB Query Language) and aggregation pipelines |

Example: a blog post with its tags and comments is one document, not three tables joined.

```json
{
  "_id": { "$oid": "66f5e3a1c2b4a1d2e3f40001" },
  "title": "Intro to MongoDB",
  "tags": ["databases", "nosql"],
  "author": { "name": "Ali", "handle": "ali" },
  "comments": [
    { "by": "sara", "text": "Nice", "at": { "$date": "2026-09-27T10:00:00Z" } }
  ]
}
```

### Where it fits
- Good fit: content and catalogs, user profiles, IoT and event data, mobile/web backends, data with varying shape, apps that read an aggregate as a whole.
- Weaker fit: heavy ad-hoc joins across many entities, strict relational reporting. MongoDB has had multi-document ACID transactions since 4.0, but if every operation needs them, a relational model may be simpler.
- See [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types) for how document stores compare with key-value, wide-column and graph databases.

## Learning path overview
| # | Unit | You learn |
|---|---|---|
| 1 | this note | course, lab, `mongosh` basics |
| 2 | [Unit 02 Getting Started with MongoDB Atlas](/Software%20Engineering/Databases/MongoDB/Unit%2002%20Getting%20Started%20with%20MongoDB%20Atlas) | Atlas, clusters, Atlas CLI |
| 3 | [Unit 03 MongoDB and the Document Model](/Software%20Engineering/Databases/MongoDB/Unit%2003%20MongoDB%20and%20the%20Document%20Model) | documents, BSON, `_id`, flexible schema |
| 4 | [Unit 04 MongoDB Data Modeling Intro](/Software%20Engineering/Databases/MongoDB/Unit%2004%20MongoDB%20Data%20Modeling%20Intro) | embed vs reference, patterns |
| 5 | [Unit 05 Connecting to a MongoDB Database](/Software%20Engineering/Databases/MongoDB/Unit%2005%20Connecting%20to%20a%20MongoDB%20Database) | connection strings, Compass |
| 6 | [Unit 06 Connecting to MongoDB in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2006%20Connecting%20to%20MongoDB%20in%20Node.js) | Node.js driver |
| 7 | [Unit 07 MongoDB CRUD Operations Insert and Find Documents](/Software%20Engineering/Databases/MongoDB/Unit%2007%20MongoDB%20CRUD%20Operations%20Insert%20and%20Find%20Documents) | insert, find, operators |
| 8 | [Unit 08 MongoDB CRUD Operations Replace and Delete Documents](/Software%20Engineering/Databases/MongoDB/Unit%2008%20MongoDB%20CRUD%20Operations%20Replace%20and%20Delete%20Documents) | update, replace, delete |
| 9 | [Unit 09 MongoDB CRUD Operations Modifying Query Results](/Software%20Engineering/Databases/MongoDB/Unit%2009%20MongoDB%20CRUD%20Operations%20Modifying%20Query%20Results) | sort, limit, projection, count |
| 10 | [Unit 10 MongoDB CRUD Operations in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2010%20MongoDB%20CRUD%20Operations%20in%20Node.js) | CRUD and transactions in Node.js |
| 11 | [Unit 11 MongoDB Aggregation](/Software%20Engineering/Databases/MongoDB/Unit%2011%20MongoDB%20Aggregation) | aggregation pipelines |
| 12 | [Unit 12 MongoDB Indexes](/Software%20Engineering/Databases/MongoDB/Unit%2012%20MongoDB%20Indexes) | indexes, ESR, `explain()` |
| 13 | [Unit 13 MongoDB Atlas Search](/Software%20Engineering/Databases/MongoDB/Unit%2013%20MongoDB%20Atlas%20Search) | full-text and vector search |

## mongosh basics
`mongosh` is the MongoDB Shell, a JavaScript REPL (built on Node.js) connected to a deployment.

```js
db.help()                    // methods available on the current database
show dbs                     // list databases (only those with data)
use bank                     // switch to "bank" (created lazily on first write)
db                           // print the current database name
show collections             // list collections in the current db
db.accounts.help()           // methods available on a collection
db.accounts.findOne()        // peek at one document
db.accounts.countDocuments() // how many documents
db.accounts.insertOne({ account_id: "MDB1", balance: 100 }) // creates the collection if missing
exit                         // leave the shell
```

Tips:
- `it` pages through the next batch of cursor results.
- Because it is JavaScript, you can write loops and variables: `const ids = db.accounts.find({}, {_id: 1}).toArray()`.
- `show dbs` hides empty databases; `use x` does not create anything until you insert.

<Info>
**🆕 2025–2026**

- **MongoDB 8.0** became GA on **2024-10-02**. MongoDB reports ~36% faster reads and ~32% faster mixed read/write workloads vs 7.0 (vendor benchmarks). ([press release](https://www.mongodb.com/company/newsroom/press-releases/mongo-db-announces-general-availability-of-mongo-db-8-0))
- **MongoDB 6.0 reached end of life on 2025-07-31** (5.0 ended 2024-10-31; 7.0 ends 2027-08-31; 8.0 is supported until 2029-10-31). ([lifecycle schedule](https://www.mongodb.com/legal/support-policy/lifecycles))
- **MongoDB 8.2 (2025-09-16)** was the first minor release shipped for Community and Enterprise Server, not only Atlas. **MongoDB 8.3** followed in **May 2026** (release notes date 8.3.0 to 2026-05-04; announced 2026-05-07). ([8.2](https://www.mongodb.com/products/updates/mongodb-8-2-is-now-available/), [8.3](https://www.mongodb.com/docs/manual/release-notes/8.3/))
</Info>

## Senior perspective
- **Pick the data model from access patterns**, not from "SQL vs NoSQL" fashion. Ask: what are the top queries, what is read together, what grows without bound?
- **Versions matter**: stay on a supported major (8.0 today) and check the lifecycle page when planning upgrades. Rapid releases (8.2, 8.3) bring features earlier but have shorter support windows on self-managed deployments.
- **The shell is for exploration and ops, not app logic.** Production code uses a driver with pooling, retries and timeouts ([Unit 06 Connecting to MongoDB in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2006%20Connecting%20to%20MongoDB%20in%20Node.js)).
- **Know the guarantees**: single-document writes are atomic; multi-document atomicity needs transactions ([Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID)); durability and read freshness depend on write concern and read concern ([Unit 10 MongoDB CRUD Operations in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2010%20MongoDB%20CRUD%20Operations%20in%20Node.js)).

## Interview questions
<Accordion title="What is a document database and how does it differ from a relational one?">
It stores records as self-contained documents (field/value pairs with nesting and arrays) grouped into collections, with a flexible schema. Relational databases store rows in tables with a fixed schema and join normalized tables. Document stores favour keeping data read together in one document; relational favours normalization and joins.
</Accordion>

<Accordion title="Name the mongosh commands to list databases, switch database and list collections.">
`show dbs`, `use <db>`, `show collections`. `db.help()` and `db.<collection>.help()` list available methods.
</Accordion>

<Accordion title="When would you not choose MongoDB?">
When the workload is dominated by complex ad-hoc joins and relational reporting, when most operations need multi-entity transactions, or when the team and tooling are built around SQL and the data is naturally tabular.
</Accordion>

<Accordion title="Is MongoDB schemaless?">
Not really. It has a flexible schema: documents in a collection can differ, but the application still has an implicit schema, and you can enforce one with `$jsonSchema` validation.
</Accordion>

## Related
- [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types) · [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization)

## Sources
- [MongoDB University: Introduction to MongoDB](https://learn.mongodb.com/)
- [MongoDB Manual: Introduction](https://www.mongodb.com/docs/manual/introduction/)
- [mongosh documentation](https://www.mongodb.com/docs/mongodb-shell/)
- [MongoDB Software Lifecycle Schedules](https://www.mongodb.com/legal/support-policy/lifecycles)
- [MongoDB 8.0 GA press release](https://www.mongodb.com/company/newsroom/press-releases/mongo-db-announces-general-availability-of-mongo-db-8-0)
