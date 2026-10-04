---
title: "SQL ANSI"
tags: [databases, index]
aliases: [SQL Course, SQL in 10 Minutes, SQL ANSI Index, Ben Forta SQL]
updated: 2026-09-27
---
# SQL ANSI

<Note>
**Start here**

A course through Ben Forta's *Sams Teach Yourself SQL in 10 Minutes a Day* (5th ed.), one note per lesson in book order, plus three PostgreSQL deep dives (triggers, indexes, partitioning). Every lesson keeps the book's examples and adds a "Beyond the book / Senior perspective" section with modern practice (window functions, CTEs, upserts, sargable queries, safe migrations) and interview questions. Part of [Databases Index](/Software%20Engineering/Databases/Databases%20Index).
</Note>

{/* diagram source: /diagrams/diagram-136.mmd */}
![diagram](/diagrams/diagram-136.svg)

## The course, in book order

### Part 1: Retrieving data (beginner)
1. [Introduction to SQL](/Software%20Engineering/Databases/SQL%20ANSI/Introduction%20to%20SQL): tables, rows, columns, datatypes, and every kind of key
2. [Retrieving Information](/Software%20Engineering/Databases/SQL%20ANSI/Retrieving%20Information): `SELECT`, `DISTINCT`, `LIMIT` / `FETCH FIRST`
3. [Sorting Data](/Software%20Engineering/Databases/SQL%20ANSI/Sorting%20Data): `ORDER BY`, sort direction, `NULLS FIRST/LAST`, collations

### Part 2: Filtering (beginner)
4. [Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Filtering%20Data): `WHERE`, operators, `BETWEEN`, `IS NULL` and the `= NULL` trap
5. [Advanced Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Advanced%20Filtering%20Data): `AND`/`OR` precedence, `IN`, `NOT`, three-valued logic, `NOT IN` + NULL, SQL injection
6. [Using Wildcard Filtering](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Wildcard%20Filtering): `LIKE`, `%`, `_`, leading wildcards, `pg_trgm`

### Part 3: Calculating and summarizing (beginner → intermediate)
7. [Creating Calculated Fields](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Calculated%20Fields): concatenation, aliases, arithmetic, `CASE`, `COALESCE`
8. [Using Data Manipulation Functions](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Data%20Manipulation%20Functions): text, date and numeric functions, sargability
9. [Summarizing Data](/Software%20Engineering/Databases/SQL%20ANSI/Summarizing%20Data): `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `FILTER`, NULL behaviour
10. [Grouping Data](/Software%20Engineering/Databases/SQL%20ANSI/Grouping%20Data): `GROUP BY`, `HAVING`, `ROLLUP`, **window functions**

### Part 4: Combining data (intermediate)
11. [Working with Subqueries](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Subqueries): `IN`/`EXISTS` subqueries, correlated subqueries, CTEs, `LATERAL`
12. [Joining Tables](/Software%20Engineering/Databases/SQL%20ANSI/Joining%20Tables): inner joins, Cartesian products, join algorithms
13. [Creating Advanced Joins](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Advanced%20Joins): aliases, self/natural/outer joins, anti-joins, `FULL OUTER JOIN` support
14. [Combining Queries](/Software%20Engineering/Databases/SQL%20ANSI/Combining%20Queries): `UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT`

### Part 5: Modifying data (intermediate)
15. [Inserting Data](/Software%20Engineering/Databases/SQL%20ANSI/Inserting%20Data): `INSERT`, `INSERT SELECT`, upserts (`ON CONFLICT`, `MERGE`), `RETURNING`
16. [Updating and Deleting Data](/Software%20Engineering/Databases/SQL%20ANSI/Updating%20and%20Deleting%20Data): `UPDATE`, `DELETE`, `TRUNCATE` per DBMS, safe data fixes

### Part 6: Schema and database objects (intermediate → senior)
17. [Creating and Manipulating Tables](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20and%20Manipulating%20Tables): `CREATE`/`ALTER`/`DROP TABLE`, identity vs UUIDv7, generated columns, lock-safe migrations
18. [Using Views](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Views): views, updatable views, `WITH CHECK OPTION`, materialized views
19. [Working with Stored Procedures](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Stored%20Procedures): T-SQL and PostgreSQL procedures vs functions

### Part 7: Transactions and advanced features (senior)
20. [Managing Transaction Processing](/Software%20Engineering/Databases/SQL%20ANSI/Managing%20Transaction%20Processing): `COMMIT`, `ROLLBACK`, savepoints, autocommit, transactional DDL
21. [Using Cursors](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Cursors): `DECLARE`/`FETCH`, `WITH HOLD`, server-side cursors in drivers
22. [Understanding Advanced SQL Features](/Software%20Engineering/Databases/SQL%20ANSI/Understanding%20Advanced%20SQL%20Features): constraints, indexes, triggers, recursive CTEs

### PostgreSQL extras (senior)
- [SQL Trigger Functions](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Trigger%20Functions): trigger functions, `BEFORE`/`AFTER`, `NEW`/`OLD`/`TG_OP`, transition tables, a full audit trigger
- [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive): B-tree/hash/GIN/GiST/BRIN, composite and covering indexes, partial and expression indexes, selectivity, `CONCURRENTLY`, unused indexes, PostgreSQL 18 skip scan
- [SQL Table Partitioning](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Table%20Partitioning): RANGE/LIST/HASH, pruning, attach/detach, retention, pg_partman, partitioning vs sharding

## Suggested study paths

<Info>
**🔁 First time through (beginner → senior)**

1. Lessons 1–10 in order: you can now read and summarize any single table.
2. Lessons 11–14: combine tables. Practise until joins and `NOT EXISTS` feel natural.
3. Lessons 15–19: write data and design tables.
4. Lessons 20–22 plus [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) and [Isolation Levels & Read Phenomena](/Software%20Engineering/Databases/Core%20Concepts/Isolation%20Levels%20%26%20Read%20Phenomena).
5. The PostgreSQL extras, then [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN) and [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization).
</Info>

<Info>
**🎯 Interview crash course**

[Advanced Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Advanced%20Filtering%20Data) (NULL logic, NOT IN trap) → [Grouping Data](/Software%20Engineering/Databases/SQL%20ANSI/Grouping%20Data) (window functions, top-N per group) → [Creating Advanced Joins](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Advanced%20Joins) (ON vs WHERE, anti-joins) → [Working with Subqueries](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Subqueries) (CTEs, LATERAL) → [Inserting Data](/Software%20Engineering/Databases/SQL%20ANSI/Inserting%20Data) (upserts) → [Managing Transaction Processing](/Software%20Engineering/Databases/SQL%20ANSI/Managing%20Transaction%20Processing) → [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive) → [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)
</Info>

<Info>
**⚡ Performance-focused**

[Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Filtering%20Data) (sargable predicates) → [Using Wildcard Filtering](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Wildcard%20Filtering) (leading wildcards, pg_trgm) → [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive) → [Database Indexes](/Software%20Engineering/Databases/Database%20Round/Indexes/Database%20Indexes) → [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN) → [SQL Table Partitioning](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Table%20Partitioning) → [Retrieving Information](/Software%20Engineering/Databases/SQL%20ANSI/Retrieving%20Information) (keyset pagination)
</Info>

## 🆕 2025–2026 in this area
- PostgreSQL 18 (2025-09-25): B-tree skip scan, better `OR` index use, parallel GIN builds, `uuidv7()`: [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive)
- PostgreSQL 18: virtual generated columns (now the default kind), `uuidv7()`, temporal `WITHOUT OVERLAPS` constraints: [Creating and Manipulating Tables](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20and%20Manipulating%20Tables)
- PostgreSQL 18: `RETURNING` can reference `OLD` and `NEW` in `INSERT`/`UPDATE`/`DELETE`/`MERGE`: [Inserting Data](/Software%20Engineering/Databases/SQL%20ANSI/Inserting%20Data), [Updating and Deleting Data](/Software%20Engineering/Databases/SQL%20ANSI/Updating%20and%20Deleting%20Data)
- PostgreSQL 19 beta (GA expected ~October 2026): `INSERT ... ON CONFLICT DO SELECT ... RETURNING` listed in the release notes: [Inserting Data](/Software%20Engineering/Databases/SQL%20ANSI/Inserting%20Data)
- PostgreSQL 19 Beta 4 (2026-09-24): `MERGE`/`SPLIT PARTITIONS` reverted from v19: [SQL Table Partitioning](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Table%20Partitioning)
- MySQL 8.0 end of life (2026-04-30); current LTS lines 8.4 and 9.7: [Creating and Manipulating Tables](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20and%20Manipulating%20Tables)

## Related
- [Databases Index](/Software%20Engineering/Databases/Databases%20Index) · [Database Round](/Software%20Engineering/Databases/Database%20Round/Database%20Round) · [Software Engineering](/Software%20Engineering/Software%20Engineering)
- Core concepts: [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Isolation Levels & Read Phenomena](/Software%20Engineering/Databases/Core%20Concepts/Isolation%20Levels%20%26%20Read%20Phenomena) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN) · [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization)
- Internals: [Database Indexes](/Software%20Engineering/Databases/Database%20Round/Indexes/Database%20Indexes) · [Primary vs Secondary Key](/Software%20Engineering/Databases/Database%20Round/Indexes/Primary%20vs%20Secondary%20Key) · [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Pearson/Sams, 2019 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL documentation](https://www.postgresql.org/docs/current/)
- [Use The Index, Luke](https://use-the-index-luke.com/)
