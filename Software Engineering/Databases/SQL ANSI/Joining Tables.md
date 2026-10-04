---
title: "Joining Tables"
tags: [databases, databases/sql]
aliases: [Joins, INNER JOIN, Equijoin, Theta Join, Cartesian Product, Join Algorithms, Hash Join]
updated: 2026-09-27
---
# Joining Tables

<Note>
**TL;DR**

Relational design splits data into tables linked by keys; a **join** puts it back together at query time. An **inner join** (equijoin) returns only rows that match on the join condition. Forget the condition and you get a **Cartesian product** (every row times every row). Prefer explicit `JOIN ... ON` syntax. Under the hood the planner chooses **nested loop, hash or merge** joins; equijoins are the case these algorithms handle best.
</Note>

> Course: SQL in 10 Minutes, Lesson 12 · Prev: [Working with Subqueries](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Subqueries) · Next: [Creating Advanced Joins](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Advanced%20Joins)

Join types you'll meet: *INNER, OUTER (LEFT, RIGHT, FULL), CROSS, SELF, NATURAL*.

In relational theory there are two families:
1. **Inner joins**: theta, equi, natural.
2. **Outer joins**: left, right, full (see [Creating Advanced Joins](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Advanced%20Joins)).

## Understanding joins

### Understanding relational tables

Suppose a table holds a product list, one product per row, with a description, a price, and the vendor that makes it. Several products come from the same vendor. Where should vendor information (name, address, contact) live? Not with each product, because:

1. The vendor info is the same for every product from that vendor; repeating it wastes time and storage.
2. If the vendor's info changes, you'd have to update every product row.
3. Repeated data is rarely entered identically every time, and inconsistent data is very hard to report on.

Having multiple copies of the same data is never good, and that is the basis of relational design: split information into multiple tables, one per kind of thing, related through common values (hence *relational*). See [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization).

Here, a `Vendors` table holds one row per vendor with a unique identifier, the **primary key**. `Products` stores only product information plus the vendor ID (the `Vendors` primary key, stored as a **foreign key**). The vendor ID relates the two tables.

The result: relational data is stored efficiently and changed easily. The book concludes that relational databases therefore *scale* far better than non-relational ones.

<Warning>
**That scaling claim is dated**

It's true that normalized schemas avoid update anomalies and stay consistent as data grows. But "relational scales better than non-relational" is not a general truth today. Many NoSQL stores (Cassandra, DynamoDB, MongoDB) were built for horizontal write scaling, while relational databases traditionally scale up and scale reads with replicas, and need sharding (Citus, Vitess) or distributed SQL (CockroachDB, Spanner, YugabyteDB) to scale writes. The honest answer is "it depends on the access patterns and consistency needs". See [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning) and [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem).
</Warning>

**Scale:** handle increasing load without failing. A well-designed database or application *scales well*.

### Why joins?

Splitting data into multiple tables gives efficient storage, easier changes and consistency. The price: to show a product with its vendor name, you must combine tables at query time.

**Join:** a mechanism to associate tables within a `SELECT`. With special syntax you join multiple tables into a single output, and the join matches the correct rows in each table on the fly.

<Note>
A join is not a physical entity; it doesn't exist in the database tables. The DBMS builds it as needed, and it lasts only for the duration of the query.
</Note>

## Creating a join

List all the tables to include and say how they are related.

### Example 1: WHERE-clause join (old style)

```sql
SELECT vend_name, prod_name, prod_price
FROM Vendors, Products
WHERE Vendors.vend_id = Products.vend_id;
```

**Analysis:** `prod_name` and `prod_price` come from one table, `vend_name` from another. The `FROM` lists two tables, and the `WHERE` tells the DBMS to match `vend_id` in `Vendors` with `vend_id` in `Products`. The columns are **fully qualified** (`Vendors.vend_id`, `Products.vend_id`) because plain `vend_id` would be ambiguous. One `SELECT` returns data from two tables.

### The importance of the join condition

**Cartesian product:** the result of joining tables **without** a join condition. The row count is the rows of the first table multiplied by the rows of the second.

```sql
SELECT vend_name, prod_name, prod_price
FROM Vendors, Products;   -- 6 vendors x 9 products = 54 mostly meaningless rows
```

**Cross join:** another name for a join that returns a Cartesian product. Written explicitly as `CROSS JOIN`, it has real uses: generating combinations (sizes x colours), or pairing every row with a calendar.

<Warning>
With the comma syntax, forgetting one condition in a multi-table `WHERE` silently produces a Cartesian product. Explicit `JOIN ... ON` makes a missing condition a syntax error. That's the main reason to prefer it.
</Warning>

## Inner joins

An **equijoin** is a join based on testing **equality** between two tables. It's also called an **inner join**, and you can use explicit syntax for it.

An inner join returns rows from both tables that satisfy the condition. It's the most widely used join and the default: plain `JOIN` means `INNER JOIN`.

Inner joins in relational algebra:

**Theta join:** joins two tables on any condition θ, using any comparison operator (`=`, `<`, `>`, `<=`, `>=`, `<>`). It's the general case of a join.
`A ⋈ (A.col2 > B.col2) B`

```sql
-- Theta (non-equi) join: which price band does each product fall in?
SELECT p.prod_name, b.band_name
FROM Products p
JOIN PriceBands b ON p.prod_price >= b.min_price AND p.prod_price < b.max_price;
```

**Equijoin:** a theta join that uses only equality.
`A ⋈ (A.col2 = B.col2) B`

<Warning>
**Correction**

The original note said equijoins are "the most difficult operation to implement efficiently" and a source of RDBMS performance problems. That's wrong. Equijoins are exactly what **hash joins** and **merge joins** are designed for, and they run in roughly linear time. It's the **non-equi** theta joins (ranges, `<>`) that are harder, often falling back to nested loops.
</Warning>

**Natural join:** joins on all columns with the **same name** in both tables, and keeps only one copy of each common column. More in [Creating Advanced Joins](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Advanced%20Joins).

### Example 2: explicit `INNER JOIN` syntax

```sql
SELECT vend_name, prod_name, prod_price
FROM Vendors
INNER JOIN Products ON Vendors.vend_id = Products.vend_id;
```

**Analysis:** same result as Example 1. The relationship is now part of the `FROM` clause as `INNER JOIN`, and the condition goes in `ON` instead of `WHERE`. ANSI SQL prefers this syntax: joins in `FROM`, filters in `WHERE`.

When both columns have the same name, `USING` is a shortcut: `JOIN Products USING (vend_id)` (not in SQL Server).

## Joining multiple tables

SQL has no limit on the number of tables in a `SELECT`. The rules are the same: list the tables, then define each relationship.

<Note>
**Maximum number of tables in a join**

The standard sets no limit, but implementations do:
- **MySQL/MariaDB: 61 tables** per join (this is where "61" comes from; the original note wrongly attributed it to Postgres).
- **PostgreSQL: no hard limit.** But the planner only searches join orders exhaustively up to `join_collapse_limit` / `from_collapse_limit` (default **8**) tables, and switches to the genetic query optimizer (GEQO) at `geqo_threshold` (default **12**) `FROM` items, so plans for very wide joins can be less optimal.
- SQL Server: 256 tables per query.
</Note>

### Example 3

```sql
SELECT prod_name, vend_name, prod_price, quantity
FROM OrderItems, Products, Vendors
WHERE Products.vend_id = Vendors.vend_id
  AND OrderItems.prod_id = Products.prod_id
  AND order_num = 20007;
```

**Analysis:** lists the items in order `20007`. Order items reference products by product ID; products reference vendors by vendor ID. The `FROM` lists three tables, the `WHERE` defines both join conditions, and one more condition filters order `20007`.

The same with explicit joins:

```sql
SELECT p.prod_name, v.vend_name, p.prod_price, oi.quantity
FROM OrderItems oi
JOIN Products p ON p.prod_id = oi.prod_id
JOIN Vendors v  ON v.vend_id = p.vend_id
WHERE oi.order_num = 20007;
```

## How databases execute joins

| Algorithm | How it works | Good for |
|---|---|---|
| **Nested loop** | For each outer row, look up matching inner rows (ideally via an index) | Small outer input + indexed inner; any join condition |
| **Hash join** | Build a hash table on the smaller input, probe with the other | Large unsorted inputs, **equality** conditions only |
| **Merge join** | Sort both inputs on the join key (or read them from indexes), then walk them together | Large inputs already sorted; equality (and some range) conditions |

{/* diagram source: /diagrams/diagram-135.mmd */}
![diagram](/diagrams/diagram-135.svg)

MySQL used only nested loops until 8.0.18 added hash joins. `EXPLAIN` shows which one was picked ([Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)).

## Beyond the book / Senior perspective

- **Index your foreign keys.** Joins from parent to child (`Vendors` to `Products` on `vend_id`) and `ON DELETE` checks need an index on the child's FK column. PostgreSQL doesn't create it automatically.
- **Join fan-out.** Joining a parent to two independent child tables (orders to items and to payments) multiplies rows; aggregates then double-count. Aggregate each child first.
- **Row estimates drive join choice.** A bad cardinality estimate (stale statistics, correlated columns) can make the planner pick a nested loop over millions of rows. `ANALYZE` and extended statistics (`CREATE STATISTICS`) help.
- **Keep join columns the same type.** Joining `int` to `varchar` forces casts and can disable index use.
- **Denormalize deliberately**, not by accident: materialized views, summary tables or JSON columns are valid when read patterns demand it ([Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization)).

## Interview questions

<Accordion title="What's a Cartesian product and how do you get one by accident?">
Every row of one table paired with every row of the other (m x n rows). You get it by omitting a join condition, most easily with comma-style joins in `FROM`.
</Accordion>

<Accordion title="Inner join vs equijoin vs theta join?">
A theta join uses any comparison; an equijoin is a theta join with only equality. "Inner join" is the SQL construct that returns only matching rows; most inner joins are equijoins.
</Accordion>

<Accordion title="Name the three physical join algorithms and when each is chosen.">
Nested loop (small outer input, indexed inner, or non-equi conditions), hash join (large inputs, equality), merge join (inputs sorted on the key, e.g. from indexes). The planner picks based on estimated sizes, indexes and memory.
</Accordion>

<Accordion title="Is there a limit to how many tables you can join?">
Not in the standard. MySQL limits a join to 61 tables. PostgreSQL has no hard limit, but exhaustive join-order search stops at `join_collapse_limit` (8) and GEQO takes over at 12 `FROM` items.
</Accordion>

<Accordion title="Why prefer `JOIN ... ON` over comma joins?">
It separates join logic from filters, makes a missing condition an error rather than a Cartesian product, and is the only way to express outer joins in standard SQL.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Creating Advanced Joins](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Advanced%20Joins) · [Working with Subqueries](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Subqueries) · [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN) · [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive)
- Scaling: [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning) · [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem) · [Database Round](/Software%20Engineering/Databases/Database%20Round/Database%20Round)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 12 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [MySQL docs: Limits on table join (61 tables)](https://dev.mysql.com/doc/refman/8.4/en/join.html)
- [PostgreSQL docs: Planner method configuration (join_collapse_limit, geqo_threshold)](https://www.postgresql.org/docs/current/runtime-config-query.html)
- [PostgreSQL docs: Planner/optimizer (join strategies)](https://www.postgresql.org/docs/current/planner-optimizer.html)
- [Use The Index, Luke: Joins](https://use-the-index-luke.com/sql/join)
- [SQL Server docs: Maximum capacity specifications](https://learn.microsoft.com/en-us/sql/sql-server/maximum-capacity-specifications-for-sql-server)
