---
title: "Database Internals"
tags: [databases, databases/internals]
aliases: [Database internals, Query Lifecycle, Heap, Row ID, CTID, Tuple, How a query executes]
updated: 2026-09-27
---
# Database Internals

<Note>
**TL;DR**

A relational database stores each table as a **heap** of fixed-size **pages** on disk and reads/writes whole pages (8 KB in Postgres, 16 KB in InnoDB), never single rows. A query goes through **parse → analyze/validate → rewrite → plan/optimize → execute**; the executor scans the heap or walks an **index** (a separate B-tree pointing at row locations) and then joins, aggregates and sorts. Most performance work is about reading fewer pages: good indexes, selective filters, and keeping the hot set in the buffer pool.
</Note>

## Storage concepts

### Table, row, tuple

- A **table** is stored as pages/blocks on disk, each page a few KB. Rows are packed into pages; the DBMS tracks where each table's data lives.
- A **row** is what the user sees. A **tuple** is the physical version of a row stored in a page. With MVCC one row can have several tuples (old versions kept for running transactions, dead ones waiting for vacuum, one current). See [MVCC & Vacuum](/Software%20Engineering/Databases/Core%20Concepts/MVCC%20%26%20Vacuum).

### Row ID (CTID)

Most databases keep an internal, system-maintained row identifier.

- **Postgres**: the `ctid` system column = `(page number, item number)`, the tuple's physical position. It changes when the row is updated (new tuple), so never use it as an application key.
- **MySQL InnoDB**: the table *is* a B+tree clustered on the **primary key**, so the primary key plays the role of the row id (if there's no PK, InnoDB creates a hidden 6-byte row id).

![Screenshot_2023-07-22_at_11.48.57_PM](/Attachments/Screenshot_2023-07-22_at_11.48.57_PM.png)

### Page

A page is a fixed-size chunk that maps to a location on disk. Depending on the storage model (row vs column store) a page holds rows or column values. A page fits many rows, depending on row size.

- Postgres: 8 KB pages. MySQL InnoDB: 16 KB. SQL Server and Oracle: 8 KB. MongoDB WiredTiger: variable, with configurable maximums.
- A read IO fetches at least one page. You get all the other rows in that page "for free".

![Screenshot_2023-07-22_at_11.49.35_PM](/Attachments/Screenshot_2023-07-22_at_11.49.35_PM.png)

Details of page layout: [Database Pages](/Software%20Engineering/Databases/Database%20Round/Database%20Pages).

### IO

An IO is a read or write request to the disk. We try to minimise them.

- An IO can't read a single row; it reads one or more pages with many rows.
- The DB loads the page, keeps the rows it needs, throws the rest away, and deserialises the result into memory.
- Some IOs are served by the **OS page cache** instead of the device. Postgres relies heavily on the OS cache in addition to its own buffer pool.

![Screenshot_2023-07-22_at_11.49.43_PM](/Attachments/Screenshot_2023-07-22_at_11.49.43_PM.png)

### Heap

The heap is the data structure where the table's rows actually live: all its pages one after another, holding **everything** (all columns). Searching the heap without help means reading lots of pages, which is expensive. That's why we need indexes: they tell us exactly which page(s) of the heap to read.

![Screenshot_2023-07-22_at_11.49.52_PM](/Attachments/Screenshot_2023-07-22_at_11.49.52_PM.png)

### Index (B-tree)

An index is a **separate** data structure that points into the heap. It holds part of the data (the indexed columns) and lets you find values quickly; then you go to the heap for the rest of the row.

- You can index one column or several.
- The index is also stored in pages and costs IO to read. Small indexes fit in memory; large ones live on disk.
- The most common structure is the **B-tree** (B+tree).
- A **clustered index** (InnoDB, SQL Server) organises the table itself around the key, usually the primary key. **Postgres has only secondary indexes**: every index, including the primary key's, points to a `ctid` in the heap.

![Screenshot_2023-07-22_at_11.50.07_PM](/Attachments/Screenshot_2023-07-22_at_11.50.07_PM.png)

More: [Database Indexes](/Software%20Engineering/Databases/Database%20Round/Indexes/Database%20Indexes) · [Primary vs Secondary Key](/Software%20Engineering/Databases/Database%20Round/Indexes/Primary%20vs%20Secondary%20Key).

### Example: how data sits on disk

![Screenshot_2023-07-22_at_11.52.15_PM](/Attachments/Screenshot_2023-07-22_at_11.52.15_PM.png)

![Screenshot_2023-07-22_at_11.50.56_PM](/Attachments/Screenshot_2023-07-22_at_11.50.56_PM.png)

The employee with `emp_id = 10` is stored on **page 0** with **row_id 1**.

Without an index:

```sql
SELECT * FROM emp WHERE emp_id = 10000;
```

The DB reads every page and checks each `emp_id` until it finds a match (and, unless `emp_id` is unique, keeps going to the end). Some databases parallelise this: several workers each scan part of the heap. Still expensive.

![Screenshot_2023-07-22_at_11.55.20_PM](/Attachments/Screenshot_2023-07-22_at_11.55.20_PM.png)

With an index on `emp_id`, the DB walks the B-tree to the leaf entry for 10000, reads the `(page, item)` pointer, and fetches just that heap page.

![Screenshot_2023-07-22_at_11.57.31_PM](/Attachments/Screenshot_2023-07-22_at_11.57.31_PM.png)

![Screenshot_2023-07-22_at_11.58.29_PM](/Attachments/Screenshot_2023-07-22_at_11.58.29_PM.png)

## Tuples in PostgreSQL (MVCC in one minute)

- `UPDATE` never modifies a tuple in place. It creates a **new version** and marks the old one dead.
- `DELETE` doesn't free space immediately; the tuple is marked dead and stays until `VACUUM` removes it. Many dead tuples mean table and index **bloat**.
- A rolled-back `INSERT` leaves its tuple behind, marked dead.
- Hidden columns on each tuple: `ctid` (location), `xmin` (creating transaction, "birth"), `xmax` (deleting transaction, "death").

```sql
SELECT ctid, xmin, xmax, * FROM products WHERE id = 1;
```

![Screenshot_2023-07-26_at_3.43.51_PM](/Attachments/Screenshot_2023-07-26_at_3.43.51_PM.png)

```sql
UPDATE products SET name = 'Cupcake' WHERE id = 1;
SELECT ctid, xmin, xmax, * FROM products WHERE id = 1;
```

![Screenshot_2023-07-26_at_3.44.11_PM](/Attachments/Screenshot_2023-07-26_at_3.44.11_PM.png)

The `ctid` moved and `xmin` changed: it's a new tuple. Full story: [MVCC & Vacuum](/Software%20Engineering/Databases/Core%20Concepts/MVCC%20%26%20Vacuum).

## What happens when you run a query?

Take this query (the original note had broken syntax; this is the correct form):

```sql
SELECT e.name, s.amount
FROM employees e
JOIN salaries s ON e.id = s.emp_id
ORDER BY s.amount;
```

{/* diagram source: /diagrams/diagram-116.mmd */}
![diagram](/diagrams/diagram-116.svg)

<Note>
**1. Parsing**

- Checks the SQL text for syntax errors: keywords, operators, identifiers in the right places.
- Converts it into a **parse tree** (query tree), a data structure the DBMS can manipulate, where nodes represent operations (join, aggregation), tables or values.

```text
                  ORDER BY s.amount
                         |
                    PROJECT e.name, s.amount
                         |
                   JOIN (e.id = s.emp_id)
                    /            \
             employees e      salaries s
```
</Note>

<Note>
**2. Validation (semantic analysis)**

The DBMS checks that tables `employees` and `salaries` and columns `name`, `id`, `amount`, `emp_id` exist, that you have privileges, and that data types are compatible (e.g. `e.id = s.emp_id` compares comparable types).
</Note>

<Note>
**3. Rewriting**

Views are expanded into their defining queries and rules are applied (Postgres rule system).
</Note>

<Note>
**4. Optimization**

The planner generates candidate **execution plans** and estimates each one's **cost** (IO, CPU, memory, network) using statistics the database maintains: row counts, value distributions and most common values per column, which indexes exist. It picks the cheapest. Bad statistics lead to bad plans, which is why `ANALYZE` matters.
</Note>

<Note>
**5. Query plan**

A tree of operators with a specific order and data flow. For our query it might be:
```text
Sort (s.amount)
  └─ Hash Join (e.id = s.emp_id)
       ├─ Seq Scan on salaries s
       └─ Hash
            └─ Seq Scan on employees e
```
In words: scan employees and build a hash table on `id`; scan salaries and probe the hash table for each `emp_id`; combine `name` and `amount`; sort by amount; return rows.
</Note>

<Note>
**6. Scans or seeks**

- **Sequential scan**: no usable index (or most rows are needed): read heap pages in order.
- **Index scan**: walk the B-tree, then fetch each matching heap page (random IO).
- **Index-only scan**: all needed columns are in the index and pages are all-visible: skip the heap.
- **Bitmap scan**: collect matching row locations from one or more indexes into a bitmap, then read heap pages in physical order.
Details: [Database Indexes](/Software%20Engineering/Databases/Database%20Round/Indexes/Database%20Indexes).
</Note>

<Note>
**7. Join algorithms**

1. **Nested loop**: for each row of the outer table, look up matches in the inner one. Great when the outer side is small and the inner has an index.
2. **Hash join**: build a hash table on the smaller input, then scan the other and probe. Great for large equi-joins without useful ordering.
3. **Merge join**: both inputs sorted on the join key, then merged in one pass. Great when inputs are already sorted (indexes) or huge.
</Note>

<Note>
**8. Aggregation and sorting**

After producing matching rows, the executor aggregates (`GROUP BY` via hashing or sorting) and sorts (`ORDER BY`). Sorting uses in-memory quicksort when data fits in `work_mem`, otherwise an external merge sort spilling to disk; a top-N heapsort is used for `ORDER BY ... LIMIT n`. Then rows are returned to the client.
</Note>

## Reading plans: `EXPLAIN ANALYZE` with `BUFFERS`

- `EXPLAIN` shows the chosen plan and cost estimates **without running** the query.
- `EXPLAIN ANALYZE` **runs** it and adds actual times and row counts.
- `BUFFERS` shows how many blocks were **hit** in the buffer pool vs **read** from the OS cache/disk (and dirtied/written).

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM emp WHERE emp_id = 10000;
```

Be careful: `ANALYZE` executes the statement, so wrap writes in `BEGIN; ... ROLLBACK;`. Full guide: [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN).

## Senior perspective

- Think in **pages, not rows**: narrow rows, good locality (clustered data, `CLUSTER`, BRIN for append-only time data), and covering indexes all reduce page reads.
- **Statistics drive plans.** After bulk loads run `ANALYZE`; for correlated columns use `CREATE STATISTICS`; watch for estimate vs actual row mismatches in plans.
- Know the memory hierarchy: CPU cache → buffer pool (`shared_buffers`) → OS page cache → SSD. A "slow query" is often a cold-cache query.
- InnoDB vs Postgres storage is a frequent interview topic: clustered PK with secondary indexes storing the PK (InnoDB) vs heap plus secondary indexes storing `ctid` (Postgres). It affects PK choice (random UUIDs hurt InnoDB inserts badly), update cost and index-only scans.

<Info>
**🆕 2025–2026**

- **PostgreSQL 18 (2025-09-25)** introduced an **asynchronous I/O** subsystem (`io_method` = `worker` (default), `io_uring` on Linux, or `sync`) that can speed up sequential scans, bitmap heap scans and vacuum by issuing reads ahead concurrently; PostgreSQL reports up to 3x faster reads from storage in some cases. `EXPLAIN ANALYZE` now includes `BUFFERS` output automatically.
- **PostgreSQL 19 (beta as of 2026-09-24)** auto-scales I/O workers (`io_min_workers`/`io_max_workers`) and adds an `IO` option to `EXPLAIN ANALYZE` for async I/O statistics.
</Info>

## Interview questions

<Accordion title="Walk me through what happens when a SQL query is executed.">
Parse into a tree, validate names/types/privileges, rewrite views, plan by enumerating strategies and costing them with statistics, then execute the chosen operator tree: scans (seq/index/bitmap), joins (nested loop/hash/merge), aggregation, sort, and stream results to the client.
</Accordion>

<Accordion title="Why do databases read pages instead of rows?">
Storage devices and the OS work in blocks; a fixed page size lets the engine cache, address (file + offset = page number × page size) and log uniformly. Reading a page amortises the IO cost over all rows in it, so small rows and good locality mean fewer IOs.
</Accordion>

<Accordion title="What's the heap and why do we need indexes?">
The heap is the unordered collection of a table's pages holding full rows. Finding a value requires scanning all pages. An index is a separate sorted structure mapping key values to row locations so the engine reads only the few pages it needs.
</Accordion>

<Accordion title="How does Postgres's storage differ from InnoDB's?">
Postgres stores rows in a heap and all indexes (including the PK) are secondary, pointing to ctids; updates create new tuples. InnoDB stores rows in a B+tree clustered by primary key; secondary indexes store the PK value, so a secondary lookup does two tree traversals, and updates happen in place with undo logs for MVCC.
</Accordion>

<Accordion title="When would the planner choose a sequential scan even though an index exists?">
When a large fraction of rows match (random index + heap access would cost more than reading all pages sequentially), when the table is tiny, when statistics suggest low selectivity, or when the predicate isn't sargable for the index (function on the column, type mismatch, leading wildcard).
</Accordion>

## Related
- [Databases Index](/Software%20Engineering/Databases/Databases%20Index) · [Database Round](/Software%20Engineering/Databases/Database%20Round/Database%20Round)
- [Database Pages](/Software%20Engineering/Databases/Database%20Round/Database%20Pages) · [Database Indexes](/Software%20Engineering/Databases/Database%20Round/Indexes/Database%20Indexes) · [Primary vs Secondary Key](/Software%20Engineering/Databases/Database%20Round/Indexes/Primary%20vs%20Secondary%20Key) · [Row Vs Column Oriented Databases](/Software%20Engineering/Databases/Database%20Round/Row%20Vs%20Column%20Oriented%20Databases)
- [MVCC & Vacuum](/Software%20Engineering/Databases/Core%20Concepts/MVCC%20%26%20Vacuum) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN) · [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID)

## Sources
- PostgreSQL docs, The Path of a Query: https://www.postgresql.org/docs/current/query-path.html
- PostgreSQL docs, Using EXPLAIN: https://www.postgresql.org/docs/current/using-explain.html
- PostgreSQL docs, System Columns: https://www.postgresql.org/docs/current/ddl-system-columns.html
- MySQL docs, Clustered and Secondary Indexes: https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html
- PostgreSQL 18 release: https://www.postgresql.org/about/news/postgresql-18-released-3142/
- PostgreSQL 19 Beta 1: https://www.postgresql.org/about/news/postgresql-19-beta-1-released-3313/
- Hussein Nasser, *Fundamentals of Database Engineering*
- Alex Petrov, *Database Internals* (O'Reilly, 2019)
