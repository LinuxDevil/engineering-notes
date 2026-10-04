---
title: "MongoDB"
tags: [databases, index]
aliases: [MongoDB Home, MongoDB Index, MongoDB University, Intro to MongoDB, Mongo]
updated: 2026-09-27
---
# MongoDB

<Note>
**TL;DR**

Notes from the MongoDB University learning path **"Introduction to MongoDB"** (13 units), tidied into course notes with examples, a senior perspective and interview questions in each unit. Read them in order: Atlas and the document model first, then data modeling, connecting, CRUD, aggregation, indexes and Atlas Search. The original index note (with the lab cluster credentials) is kept in `_Archive/MongoDB (original).md`.
</Note>

{/* diagram source: /diagrams/diagram-127.mmd */}
![diagram](/diagrams/diagram-127.svg)

## 🧭 Quick links
- Course: [MongoDB University, Introduction to MongoDB](https://learn.mongodb.com/)
- Lab environment: the course ran in an **Instruqt** in-browser lab (terminal + editor). See [Unit 01 Start Here - Intro to MongoDB](/Software%20Engineering/Databases/MongoDB/Unit%2001%20Start%20Here%20-%20Intro%20to%20MongoDB).
- Cluster used in the labs: an Atlas cluster named `myAtlasClusterEDU` with a `readWriteAnyDatabase` user. Credentials are kept only in the archived original note; never paste them into new notes.

## The 13 units

### Getting started (beginner)
1. [Unit 01 Start Here - Intro to MongoDB](/Software%20Engineering/Databases/MongoDB/Unit%2001%20Start%20Here%20-%20Intro%20to%20MongoDB): what MongoDB is, the learning path, `mongosh` basics
2. [Unit 02 Getting Started with MongoDB Atlas](/Software%20Engineering/Databases/MongoDB/Unit%2002%20Getting%20Started%20with%20MongoDB%20Atlas): Atlas, clusters and tiers, Atlas CLI
3. [Unit 03 MongoDB and the Document Model](/Software%20Engineering/Databases/MongoDB/Unit%2003%20MongoDB%20and%20the%20Document%20Model): documents, collections, BSON, `_id`, flexible schema

### Modeling (beginner → intermediate)
4. [Unit 04 MongoDB Data Modeling Intro](/Software%20Engineering/Databases/MongoDB/Unit%2004%20MongoDB%20Data%20Modeling%20Intro): relationships, embedding vs referencing, schema patterns and anti-patterns

### Connecting
5. [Unit 05 Connecting to a MongoDB Database](/Software%20Engineering/Databases/MongoDB/Unit%2005%20Connecting%20to%20a%20MongoDB%20Database): connection strings, `mongosh`, Compass, drivers, troubleshooting
6. [Unit 06 Connecting to MongoDB in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2006%20Connecting%20to%20MongoDB%20in%20Node.js): the Node.js driver, one `MongoClient` per process, pooling

### CRUD (intermediate)
7. [Unit 07 MongoDB CRUD Operations Insert and Find Documents](/Software%20Engineering/Databases/MongoDB/Unit%2007%20MongoDB%20CRUD%20Operations%20Insert%20and%20Find%20Documents): `insertOne/Many`, `find`, comparison, logical and array operators
8. [Unit 08 MongoDB CRUD Operations Replace and Delete Documents](/Software%20Engineering/Databases/MongoDB/Unit%2008%20MongoDB%20CRUD%20Operations%20Replace%20and%20Delete%20Documents): `replaceOne`, `updateOne/Many`, upsert, `findAndModify`, deletes
9. [Unit 09 MongoDB CRUD Operations Modifying Query Results](/Software%20Engineering/Databases/MongoDB/Unit%2009%20MongoDB%20CRUD%20Operations%20Modifying%20Query%20Results): cursors, `sort`, `limit`, projection, `countDocuments`
10. [Unit 10 MongoDB CRUD Operations in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2010%20MongoDB%20CRUD%20Operations%20in%20Node.js): BSON in Node.js, CRUD from the driver, multi-document transactions

### Aggregation (intermediate → senior)
11. [Unit 11 MongoDB Aggregation](/Software%20Engineering/Databases/MongoDB/Unit%2011%20MongoDB%20Aggregation): pipelines, `$match`, `$group`, `$lookup`, memory limits

### Indexes (intermediate → senior)
12. [Unit 12 MongoDB Indexes](/Software%20Engineering/Databases/MongoDB/Unit%2012%20MongoDB%20Indexes): single-field, compound, multikey, ESR rule, `explain()`, hidden indexes

### Search (senior)
13. [Unit 13 MongoDB Atlas Search](/Software%20Engineering/Databases/MongoDB/Unit%2013%20MongoDB%20Atlas%20Search): search indexes, analyzers, `$search`, `compound`, facets, Vector Search and hybrid search

## Suggested study path
<Info>
**🔁 Full course, in order**

Units 01 → 13. Do the labs in `mongosh` against a free Atlas cluster (M0) or a local `mongod` / Docker container. After Unit 04, read [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization); after Unit 12, read [Database Indexes](/Software%20Engineering/Databases/Database%20Round/Indexes/Database%20Indexes) and [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN).
</Info>

<Info>
**🎯 Interview prep (MongoDB-focused backend role)**

[Unit 03 MongoDB and the Document Model](/Software%20Engineering/Databases/MongoDB/Unit%2003%20MongoDB%20and%20the%20Document%20Model) → [Unit 04 MongoDB Data Modeling Intro](/Software%20Engineering/Databases/MongoDB/Unit%2004%20MongoDB%20Data%20Modeling%20Intro) (patterns and anti-patterns) → [Unit 12 MongoDB Indexes](/Software%20Engineering/Databases/MongoDB/Unit%2012%20MongoDB%20Indexes) (ESR) → [Unit 11 MongoDB Aggregation](/Software%20Engineering/Databases/MongoDB/Unit%2011%20MongoDB%20Aggregation) → [Unit 10 MongoDB CRUD Operations in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2010%20MongoDB%20CRUD%20Operations%20in%20Node.js) (transactions, write concern) → [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types) → [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem) → [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning)
</Info>

<Info>
**⚡ Catch up on 2025–2026 only**

Read the 🆕 callouts in the units listed below.
</Info>

## 🆕 2025–2026 in MongoDB
- **2024-10-02** MongoDB 8.0 GA, MongoDB claims ~36% faster reads vs 7.0: [Unit 01 Start Here - Intro to MongoDB](/Software%20Engineering/Databases/MongoDB/Unit%2001%20Start%20Here%20-%20Intro%20to%20MongoDB)
- **2025-02** Atlas Flex clusters replace Serverless instances and M2/M5 shared tiers; support for those ended **2026-01-22**: [Unit 02 Getting Started with MongoDB Atlas](/Software%20Engineering/Databases/MongoDB/Unit%2002%20Getting%20Started%20with%20MongoDB%20Atlas)
- **2025-02-24** MongoDB announces the acquisition of Voyage AI (embeddings and rerankers): [Unit 13 MongoDB Atlas Search](/Software%20Engineering/Databases/MongoDB/Unit%2013%20MongoDB%20Atlas%20Search)
- **2025-07-31** MongoDB 6.0 end of life: [Unit 01 Start Here - Intro to MongoDB](/Software%20Engineering/Databases/MongoDB/Unit%2001%20Start%20Here%20-%20Intro%20to%20MongoDB)
- **2025-09-16** MongoDB 8.2, first minor release also shipped for Community and Enterprise Server; Search and Vector Search for self-managed in public preview; `$scoreFusion`: [Unit 13 MongoDB Atlas Search](/Software%20Engineering/Databases/MongoDB/Unit%2013%20MongoDB%20Atlas%20Search), [Unit 01 Start Here - Intro to MongoDB](/Software%20Engineering/Databases/MongoDB/Unit%2001%20Start%20Here%20-%20Intro%20to%20MongoDB)
- **2025-09-30** Atlas Device Sync / Device SDKs and Atlas Data API + HTTPS Endpoints reach end of life: [Unit 02 Getting Started with MongoDB Atlas](/Software%20Engineering/Databases/MongoDB/Unit%2002%20Getting%20Started%20with%20MongoDB%20Atlas)
- **2025-11-06** Node.js driver 7.0 (Node.js 20.19+ required): [Unit 06 Connecting to MongoDB in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2006%20Connecting%20to%20MongoDB%20in%20Node.js), [Unit 10 MongoDB CRUD Operations in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2010%20MongoDB%20CRUD%20Operations%20in%20Node.js)
- **2026-05** MongoDB 8.3: `$scoreFusion` GA, cost-based plan selection by default, new expressions: [Unit 11 MongoDB Aggregation](/Software%20Engineering/Databases/MongoDB/Unit%2011%20MongoDB%20Aggregation), [Unit 12 MongoDB Indexes](/Software%20Engineering/Databases/MongoDB/Unit%2012%20MongoDB%20Indexes), [Unit 13 MongoDB Atlas Search](/Software%20Engineering/Databases/MongoDB/Unit%2013%20MongoDB%20Atlas%20Search)

## Related
- [Databases Index](/Software%20Engineering/Databases/Databases%20Index), the Databases domain home
- [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types) · [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization) · [Vector Databases & pgvector](/Software%20Engineering/Databases/Core%20Concepts/Vector%20Databases%20%26%20pgvector)
- [Software Engineering](/Software%20Engineering/Software%20Engineering), the vault root
- `_Archive/MongoDB (original).md`: the original index note, unchanged (contains lab credentials)

## Sources
- [MongoDB University](https://learn.mongodb.com/)
- [MongoDB Manual](https://www.mongodb.com/docs/manual/)
- [MongoDB Software Lifecycle Schedules](https://www.mongodb.com/legal/support-policy/lifecycles)
