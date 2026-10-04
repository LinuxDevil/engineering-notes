---
title: "Database Round"
tags: [databases, index]
aliases: [Database Round Index, DB Fundamentals, Fundamentals of Database Engineering]
updated: 2026-09-27
---
# Database Round

<Note>
**Start here for database fundamentals**

Notes from Hussein Nasser's *Fundamentals of Database Engineering* style course, mixed with textbook summaries (Silberschatz *Database System Concepts*, Kleppmann *DDIA*). Order: how data sits on disk → indexes → transactions and concurrency → scaling and distribution → operations. Concept notes created during the 2026 restructure live in `Databases/Core Concepts/` and are linked here where they belong in the sequence.
</Note>

{/* diagram source: /diagrams/diagram-118.mmd */}
![diagram](/diagrams/diagram-118.svg)

## 1. Storage: how a database holds data (beginner)
1. [Database Internals](/Software%20Engineering/Databases/Database%20Round/Database%20Internals): heap, pages, IO, row ids, what happens when a query runs
2. [Database Pages](/Software%20Engineering/Databases/Database%20Round/Database%20Pages): page layout, buffer pool, small vs large pages
3. [Row Vs Column Oriented Databases](/Software%20Engineering/Databases/Database%20Round/Row%20Vs%20Column%20Oriented%20Databases): OLTP vs OLAP storage

## 2. Indexing (beginner → intermediate)
4. [Database Indexes](/Software%20Engineering/Databases/Database%20Round/Indexes/Database%20Indexes): B-tree/B+tree, LSM trees, bloom filters, scan types, covering indexes
5. [Primary vs Secondary Key](/Software%20Engineering/Databases/Database%20Round/Indexes/Primary%20vs%20Secondary%20Key): clustered (IOT) vs heap + secondary indexes
6. [Keyset Pagination vs Offset](/Software%20Engineering/Databases/Database%20Round/Keyset%20Pagination%20vs%20Offset): why `OFFSET` is slow and how to paginate with an index
- Deeper: [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)

## 3. Transactions and concurrency (intermediate)
7. [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID): the four guarantees, lifecycle, durability and WAL
8. [Isolation Levels & Read Phenomena](/Software%20Engineering/Databases/Core%20Concepts/Isolation%20Levels%20%26%20Read%20Phenomena): dirty/non-repeatable/phantom reads, write skew, what Postgres/MySQL do
9. [Locking & Concurrency Control](/Software%20Engineering/Databases/Core%20Concepts/Locking%20%26%20Concurrency%20Control): shared/exclusive locks, 2PL, deadlocks, optimistic vs pessimistic, Postgres lock modes
10. [MVCC & Vacuum](/Software%20Engineering/Databases/Core%20Concepts/MVCC%20%26%20Vacuum): row versions, dead tuples, VACUUM, bloat, wraparound

## 4. Scaling and distribution (intermediate → senior)
11. [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem): CAP, PACELC, consistency models
12. [Clustering and Scaling](/Software%20Engineering/Databases/Database%20Round/Clustering%20and%20Scaling): vertical vs horizontal, failover, redundancy, clusters
13. [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC): leader/follower, sync vs async, lag, 2PC/3PC, sagas
14. [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning): shard keys, consistent hashing, Postgres range partitioning

## 5. Operations (senior)
15. [Connection Pooling](/Software%20Engineering/Databases/Database%20Round/Connection%20Pooling): pool sizing, PgBouncer
16. [Database Backup](/Software%20Engineering/Databases/Database%20Round/Database%20Backup): RPO/RTO, PITR, pg_dump/pgBackRest, 3-2-1, restore testing
- Also: [Database Migrations & Schema Evolution](/Software%20Engineering/Databases/Core%20Concepts/Database%20Migrations%20%26%20Schema%20Evolution) · [PostgreSQL 17 to 18 - What's New](/Software%20Engineering/Databases/Core%20Concepts/PostgreSQL%2017%20to%2018%20-%20What's%20New)

<Info>
**Suggested study path**

1. Storage: Internals → Pages → Row vs Column
2. Indexes → Primary vs Secondary Key → Keyset Pagination → [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)
3. Transactions & ACID → Isolation Levels → Locking → MVCC & Vacuum
4. CAP → Clustering and Scaling → Replication → Sharding
5. Connection Pooling → Database Backup → Migrations
6. Interview drill: answer every `Interview questions` block without opening it.
</Info>

## Original syllabus (the user's plan, with where each topic lives now)

| Topic | Sub-topics (from the original outline) | Covered in |
|---|---|---|
| **ACID** | Intro to A/C/I/D, importance, practical examples, drawbacks | [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) |
| **Transaction boundaries** | Definition, how to define them, impact on performance and concurrency, examples per DBMS | [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Managing Transaction Processing](/Software%20Engineering/Databases/SQL%20ANSI/Managing%20Transaction%20Processing) |
| **Isolation levels** | Definition, RU/RC/RR/Serializable, pros and cons, impact on ACID | [Isolation Levels & Read Phenomena](/Software%20Engineering/Databases/Core%20Concepts/Isolation%20Levels%20%26%20Read%20Phenomena) |
| **How locks work** | Basic locking, shared vs exclusive, deadlocks (prevention, detection), optimistic vs pessimistic | [Locking & Concurrency Control](/Software%20Engineering/Databases/Core%20Concepts/Locking%20%26%20Concurrency%20Control) |
| **CAP theorem** | C/A/P, systems favouring each, the trade-off | [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem) |
| **Distributed transactions** | Problems and solutions, role of ACID and CAP, intro to 2PC | [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC) |
| **2PC** | Protocol, coordinator and participants, vote and commit phases, pros and cons | [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC) |
| **SQL** | SELECT/INSERT/UPDATE/DELETE, joins, subqueries, indexes, procedures and functions, best practices and optimization | [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN) |
| **Database management** | DBMS overview, normalization/denormalization, performance tuning, security | [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN) |
| **Scaling** | Vertical vs horizontal, when to scale, strategies and trade-offs, impact on design | [Clustering and Scaling](/Software%20Engineering/Databases/Database%20Round/Clustering%20and%20Scaling) · [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning) |
| **Cluster** | Definition, fail-over / load-balancing / distributed clusters, HA and scalability, examples | [Clustering and Scaling](/Software%20Engineering/Databases/Database%20Round/Clustering%20and%20Scaling) |
| **Replication** | Concept, master-slave / multi-master / peer-to-peer, replication lag, strategies per DBMS | [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC) |
| **Procedures (backup, recovery, migration)** | Importance, full/incremental/differential, disaster recovery, migrations and their challenges | [Database Backup](/Software%20Engineering/Databases/Database%20Round/Database%20Backup) · [Database Migrations & Schema Evolution](/Software%20Engineering/Databases/Core%20Concepts/Database%20Migrations%20%26%20Schema%20Evolution) |
| **Bulk operations** | Bulk insert, bulk update, bulk delete, performance considerations | See below |

### Bulk operations (quick reference)

```sql
-- bulk insert: one statement, many rows (or COPY for large loads)
INSERT INTO products (id, name, price) VALUES (1,'A',10), (2,'B',12), (3,'C',9);
COPY products FROM '/data/products.csv' WITH (FORMAT csv, HEADER true);   -- psql: \copy for client-side files

-- bulk update from a set of values
UPDATE products p SET price = v.price
FROM (VALUES (1, 11.0), (2, 13.0)) AS v(id, price)
WHERE p.id = v.id;

-- bulk delete in batches to keep transactions short and replicas happy
DELETE FROM events WHERE id IN (
  SELECT id FROM events WHERE created_at < now() - interval '90 days' LIMIT 10000
);
```

Performance considerations: batch sizes of thousands (not one giant transaction), drop/recreate or defer secondary indexes for huge loads, use `COPY` over row-by-row inserts, prefer dropping partitions to mass deletes, `ANALYZE` afterwards, and watch replication lag and WAL volume.

## Archived originals
`Databases/_Archive/` holds the notes as they were before the restructure: `ACID Overview (original)`, `Overview - Locking & Distributed DB (original)`, `Overview 1 - Replication & Data Systems (original)`.

## Related
- [Databases Index](/Software%20Engineering/Databases/Databases%20Index) · [Software Engineering](/Software%20Engineering/Software%20Engineering)
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB)
