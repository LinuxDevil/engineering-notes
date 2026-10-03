---
tags: [databases, databases/sql, databases/postgres]
aliases: [SQL Index, SQL Indexes, Index Types, Composite Index, Covering Index, Partial Index, Expression Index, CREATE INDEX CONCURRENTLY, GIN, BRIN]
updated: 2026-09-27
---
# SQL Indexes Deep Dive

> [!tldr] TL;DR
> An index is a separate, sorted (or hashed) structure that maps column values to rows, turning a full scan into a few page reads. Pick the **type** for the operator (B-tree for `=`/ranges/sorting, GIN for arrays/JSONB/full-text/trigrams, GiST for geometry and ranges, BRIN for huge naturally ordered tables, hash for equality only). Design **composite** indexes by the leftmost-prefix rule, make hot queries **covering** (index-only scans), use **partial** and **expression** indexes for special cases, build with `CONCURRENTLY` in production, and drop the ones nobody uses. There's no magic selectivity threshold: the planner compares estimated costs.

> PostgreSQL extra, after the course. Book intro to indexes: [[Understanding Advanced SQL Features]] · Internals (B+trees, LSM trees, bloom filters, scan types): [[Database Indexes]] · Prev extra: [[SQL Trigger Functions]] · Next extra: [[SQL Table Partitioning]]

This note replaces the old "SQL Index" note. The Forta "Understanding Indexes" text (the book-index analogy and indexing warnings) that was duplicated here now lives only in [[Understanding Advanced SQL Features]].

## Why indexes?

Without an index, finding rows means checking every row. An index gives the database a reference point: a sorted structure it can search in logarithmic time, then jump to the matching rows.

```sql
SELECT * FROM employee WHERE employee_name = 'Abc';
```

**Without an index:** the database must look at every row to see whether `employee_name` is `'Abc'`. It can't stop at the first match, because there may be more. This is a **full table scan** (sequential scan).

**With an index on `employee_name`:** it searches the index (a B-tree in most cases), finds the entries for `'Abc'` and follows their pointers to the rows.

What's inside an index entry? The **indexed column value(s)** and a **pointer** to the row (a heap tuple ID in PostgreSQL, the primary key in MySQL InnoDB secondary indexes). Not the other columns, unless you add them with `INCLUDE`; otherwise the index would be a second copy of the table.

## Why not index everything?

- **Storage:** every index takes space, often a large fraction of the table.
- **Write cost:** every `INSERT`, `DELETE`, and `UPDATE` of an indexed column must update every index. In PostgreSQL, indexing a frequently updated column also prevents HOT (heap-only tuple) updates, so *every* index gets a new entry.
- **Locks and maintenance:** building and rebuilding indexes takes time and locks (see `CONCURRENTLY` below).
- **Planner work:** more indexes, more plans to consider.

So index the columns that your **frequent queries** filter, join or sort on, not every column.

What gets indexed automatically:
- **Primary keys** and **unique constraints** get a unique index in every major DBMS.
- **Foreign keys:** MySQL InnoDB creates an index on the FK column automatically. **PostgreSQL and SQL Server do not**; add one yourself (joins and parent deletes need it).

## Index types in PostgreSQL

| Type | Supports | Use for | Notes |
|---|---|---|---|
| **B-tree** (default) | `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `IS NULL`, prefix `LIKE 'abc%'` (with `C` collation or `text_pattern_ops`), sorting | Almost everything | Sorted, so it also serves `ORDER BY` and `MIN`/`MAX` |
| **Hash** | `=` only | Equality on large values (long strings) | WAL-logged and crash-safe since PostgreSQL 10; rarely better than B-tree; no uniqueness, no multi-column |
| **GIN** (generalized inverted) | "contains" operators: `@>`, `?`, `&&`, `@@` | `jsonb`, arrays, full-text (`tsvector`), trigrams (`pg_trgm`) | Many keys per row; slower to update (uses a pending list) |
| **GiST** (generalized search tree) | overlap, containment, nearest-neighbour (`<->`) | geometry (PostGIS), range types, exclusion constraints, fuzzy text | Lossy; flexible framework |
| **SP-GiST** | space-partitioned searches | IP addresses (`inet`), quadtrees, phone-number-like prefixes | Non-balanced structures |
| **BRIN** (block range) | ranges over physically ordered data | append-only time series, huge logs | Stores min/max per block range; tiny (KBs for GBs of table), only useful if the column correlates with physical order |

MySQL InnoDB mainly uses B+tree indexes (plus `FULLTEXT` and `SPATIAL` R-trees; its "adaptive hash index" is internal). SQL Server has rowstore B-trees plus **columnstore indexes** (introduced in SQL Server 2012) for analytics.

How a B-tree/B+tree works (node layout, page splits, why height stays 3 to 4 levels), how hash indexes compare, and LSM trees: [[Database Indexes]].

### The hash index trade-off

A hash index maps each value to a bucket: `"Abc" => pointer to row`. Equality lookups are very fast. But hash tables are **not sorted**, so they can't help with ranges (`age < 40`), sorting, or prefix matches. That's why B-trees are the default: they're nearly as fast for equality, and also handle ranges and ordering.

```sql
CREATE INDEX employee_name_hash ON employee USING hash (employee_name);
```

## Clustered vs non-clustered

| | Clustered (index-organized) | Non-clustered (secondary) |
|---|---|---|
| What | The table rows **are** stored in index order (the leaf level holds the rows) | A separate structure; leaves hold keys plus a pointer to the row |
| How many | One per table | Many |
| Examples | InnoDB primary key; SQL Server clustered index (by default on the PK) | Every PostgreSQL index; InnoDB/SQL Server secondary indexes |
| Pointer | n/a | InnoDB: the primary key; SQL Server heap: RID (row identifier), or the clustering key; PostgreSQL: TID `(page, offset)` |

Rule of thumb from the original note: *unique → non-clustered, primary → clustered* (SQL Server's defaults). PostgreSQL tables are always **heaps**; `CLUSTER table USING index` reorders rows once but doesn't maintain the order. Details and why PK choice matters in InnoDB: [[Primary vs Secondary Key]].

## Composite indexes and the leftmost-prefix rule

A composite index on `(a, b, c)` is sorted by `a`, then `b` within `a`, then `c`, like a phone book sorted by surname, then first name.

```sql
CREATE INDEX orders_cust_date_idx ON orders (cust_id, order_date);
```

| Query | Can use the index well? |
|---|---|
| `WHERE cust_id = 42` | Yes (leftmost column) |
| `WHERE cust_id = 42 AND order_date >= '2026-01-01'` | Yes, both columns |
| `WHERE cust_id = 42 ORDER BY order_date DESC LIMIT 10` | Yes, and no sort needed |
| `WHERE order_date >= '2026-01-01'` | Not traditionally (leading column missing); PostgreSQL 18 may use a **skip scan** if `cust_id` has few distinct values |
| `WHERE cust_id > 40 AND order_date = '2026-01-01'` | Partially: the range on `cust_id` stops the index from narrowing on `order_date` |

Column order guidelines:
1. Columns compared with **equality** first.
2. Then the **range** or **sort** column.
3. Between equality columns, order by what serves the most queries (selectivity matters less than prefix reuse).

One `(a, b)` index usually makes a separate `(a)` index redundant.

## Covering indexes and index-only scans

If every column a query needs is in the index, PostgreSQL can answer from the index alone (an **index-only scan**), skipping the table, as long as the visibility map shows the pages are all-visible (keep tables vacuumed).

```sql
-- key column for searching, INCLUDE column carried along for output (PostgreSQL 11+, SQL Server)
CREATE INDEX orders_cust_inc_idx ON orders (cust_id) INCLUDE (order_total);

EXPLAIN SELECT cust_id, order_total FROM orders WHERE cust_id = 42;
-- Index Only Scan using orders_cust_inc_idx on orders
```

`INCLUDE` columns aren't part of the sort order or uniqueness; they just ride along in the leaf. In MySQL InnoDB, every secondary index implicitly "covers" the primary key columns.

## Partial indexes

Index only the rows you query:

```sql
-- Only unprocessed jobs are ever queried; they're 0.1% of the table
CREATE INDEX jobs_pending_idx ON jobs (created_at) WHERE status = 'pending';

-- Unique only among live rows (soft deletes)
CREATE UNIQUE INDEX users_email_live_uq ON users (email) WHERE deleted_at IS NULL;
```

Smaller, faster to maintain, and they can enforce conditional uniqueness. The query's `WHERE` must imply the index predicate. (PostgreSQL, SQLite; SQL Server calls them *filtered indexes*; MySQL has none.)

## Expression (functional) indexes

Index the result of an expression, so a non-sargable filter becomes sargable:

```sql
CREATE INDEX users_lower_email_idx ON users (lower(email));
SELECT * FROM users WHERE lower(email) = lower($1);      -- uses the index

CREATE INDEX orders_day_idx ON orders ((order_date::date));
```

The query must use the **same expression**. MySQL 8.0.13+ supports functional key parts; SQL Server uses indexed computed columns.

## Selectivity and cardinality: when does the planner use an index?

- **Cardinality** = number of distinct values in a column. **Selectivity** of a predicate = fraction of rows it returns.
- An index scan reads the index and then does (possibly random) page reads into the table. If a predicate matches a **large** fraction of rows, reading the whole table sequentially is cheaper.
- The planner decides with **statistics** (`pg_stats`: distinct counts, most-common values and their frequencies, histograms, physical correlation) and **cost estimates** (`seq_page_cost`, `random_page_cost`, `cpu_*` settings). It compares the estimated cost of a seq scan, an index scan and a bitmap scan and picks the cheapest.

> [!warning] Correction: there is no fixed "30%" rule
> The original note said "the query optimizer will avoid using the index if the cardinality is less than 30% of the record number". That's folklore. There's **no fixed threshold**. The crossover depends on the table and index size, how well the column's order correlates with the physical row order, whether an index-only scan is possible, `random_page_cost` (lower on SSDs), what's cached, and `LIMIT`. An index on a low-cardinality column can still be excellent when you query the **rare** value (`status = 'failed'` when 0.1% of rows fail); that's what partial indexes and most-common-value statistics are for. Check with `EXPLAIN ANALYZE`.

```sql
-- See what the planner knows about a column
SELECT n_distinct, most_common_vals, most_common_freqs, correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

Keep statistics fresh (`ANALYZE`, autovacuum). For correlated columns (`city`, `zip`), `CREATE STATISTICS` gives the planner multi-column knowledge. More in [[Query Optimization & EXPLAIN]].

## Worked example: blocking factor and block accesses

This example (from the original note) shows *why* indexes help, counting disk block reads. It's an idealized model: a sorted file with binary search, not a real B-tree, but the orders of magnitude are right.

Data is stored on disk in **blocks** (pages), and a block is read as a whole: the atomic unit of disk access. Blocks are chained somewhat like a linked list (data plus a pointer to the next block) and need not be contiguous.

Records can be physically sorted on only one field. Searching an **unsorted** field needs a linear search: `(N + 1) / 2` block accesses on average (N = number of blocks), and if the field isn't unique (a non-key field), the whole table must be read: `N` accesses. Searching a **sorted** field allows binary search: `log2 N` block accesses, and for a non-key field you can stop as soon as a larger value appears.

**Sample table** (5,000,000 rows, fixed-size `CHAR` so sizes are exact; MyISAM with 1,024-byte blocks):

```text
Field name        Data type      Size on disk
id (Primary key)  Unsigned INT   4 bytes
firstName         Char(50)       50 bytes
lastName          Char(50)       50 bytes
emailAddress      Char(100)      100 bytes
```

**Example 1: sorted vs unsorted field, no index**

- Record size `R = 204` bytes, block size `B = 1,024` bytes.
- Blocking factor `bfr = floor(B / R) = floor(1024 / 204) = 5` records per block.
- Blocks needed `N = r / bfr = 5,000,000 / 5 = 1,000,000` blocks.
- Linear search on `id`: on average `N / 2 = 500,000` block accesses. But `id` is sorted, so binary search needs `log2 1,000,000 ≈ 19.93`, about **20** block accesses.
- `firstName` is neither sorted nor unique: binary search is impossible, and the search must read to the end: **1,000,000** block accesses.

**Example 2: with an index on `firstName`**

An index record holds only the indexed field and a pointer, so it's much smaller:

```text
Field name        Data type      Size on disk
firstName         Char(50)       50 bytes
(record pointer)  Special        4 bytes
```

(MySQL row pointers are 2 to 5 bytes depending on table size.)

- Index record size `R = 54` bytes.
- `bfr = floor(1024 / 54) = 18` records per block.
- Index blocks `N = 5,000,000 / 18 ≈ 277,778` blocks.
- Binary search of the index: `log2 277,778 ≈ 18.08`, about **19** block accesses, plus **1** to read the actual record: **20** block accesses instead of **1,000,000**.

**Cost:** the index needs 277,778 extra blocks, about 28% more space than the table. (In a real B+tree, fan-out is in the hundreds, so a lookup is 3 to 4 page reads, most of them cached.)

**When should an index be used?** Indexing fields used only for output wastes disk space and slows every insert and delete, so don't. Index fields you **search, join or sort** on. Uniqueness matters: an index on a field with 2 distinct values splits the data in half, which barely helps when you ask for the common value.

(The original note also mentioned that MyISAM stores all of a table's indexes in one file, which could hit file-system size limits. That was a real concern on old file systems; InnoDB is the default engine since MySQL 5.5.)

## When NOT to index

- Small tables (a few pages): a seq scan is already fast.
- Columns rarely used in `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`.
- Low-selectivity columns queried for common values (use a partial index for the rare ones).
- Write-heavy tables where the read benefit is small (logs, queues), beyond the essentials.
- Duplicates: `(a)` when `(a, b)` exists; two indexes on the same columns.
- Queries that can't use it anyway: `LIKE '%x%'` on a B-tree (use `pg_trgm`, see [[Using Wildcard Filtering]]), `func(col)` without an expression index, mismatched types.

## Creating indexes safely in production

A plain `CREATE INDEX` blocks **writes** to the table for the whole build (the original note's remark that a "lock needs to be done" for concurrency). On a busy table use:

```sql
CREATE INDEX CONCURRENTLY orders_cust_date_idx ON orders (cust_id, order_date);
```

- Doesn't block reads or writes; takes longer (two table scans) and can't run inside a transaction block.
- If it fails, it leaves an **INVALID** index behind: find it (`\d orders` or `pg_index.indisvalid = false`), `DROP INDEX CONCURRENTLY`, and retry.
- Rebuild bloated indexes online with `REINDEX INDEX CONCURRENTLY` (PostgreSQL 12+).
- MySQL InnoDB: `ALTER TABLE ... ADD INDEX ..., ALGORITHM=INPLACE, LOCK=NONE`; SQL Server Enterprise: `WITH (ONLINE = ON)`.

## Finding unused and missing indexes

```sql
-- Indexes never scanned since statistics were last reset (PostgreSQL)
SELECT s.schemaname,
       s.relname        AS table_name,
       s.indexrelname   AS index_name,
       s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS index_size
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan = 0
  AND NOT i.indisunique          -- keep indexes that enforce constraints
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

Check on the primary **and** replicas (stats are per server) and over a long enough window (month-end jobs) before dropping. For missing indexes, start from the slowest queries in `pg_stat_statements`, then read their plans ([[Query Optimization & EXPLAIN]]). Tables with high `seq_scan` counts in `pg_stat_user_tables` are candidates.

> [!info]+ 🆕 2025–2026
> **PostgreSQL 18 (released 2025-09-25)**:
> - **B-tree skip scan:** multicolumn B-tree indexes can now be used when the query has no `=` condition on one or more **leading** columns. For an index on `(cust_id, order_date)`, `WHERE order_date >= '2026-01-01'` can "skip" through the distinct `cust_id` values instead of scanning everything. It works best when the skipped column has few distinct values; the leftmost-prefix rule is still the right default for design.
> - **Better `OR` handling:** queries with `OR` conditions in `WHERE` can be optimized to use an index, instead of falling back to a seq scan or bitmap OR.
> - **Parallel GIN index builds**, joining B-tree and BRIN, which speeds up building JSONB, array and full-text indexes.
> - **`uuidv7()`**: timestamp-ordered UUIDs. Random UUIDv4 primary keys insert at random positions in the B-tree (page splits, poor cache locality, more WAL); UUIDv7 keys arrive in roughly increasing order, like a sequence, while still being globally unique. See [[Creating and Manipulating Tables]].

## Beyond the book / Senior perspective

- **Index for queries, not columns.** Gather the top queries by total time (`pg_stat_statements`), and design one composite/covering/partial index that serves several of them.
- **Write amplification is the hidden cost.** Ten indexes on a hot table can make writes several times slower and bloat WAL and replicas.
- **Keep the working set in memory.** Index size vs RAM is often the real constraint; partial and BRIN indexes help.
- **Sargability** discipline in code review: no functions on indexed columns in `WHERE`, matching types in joins, no leading `%`.
- **Unique constraints on partitioned tables** must include the partition key, because PostgreSQL indexes are per partition ([[SQL Table Partitioning]]).
- **Verify, don't guess:** `EXPLAIN (ANALYZE, BUFFERS)` before and after. See [[Query Optimization & EXPLAIN]].

## Interview questions

> [!question]- How does an index speed up a query, and what does it cost?
> It keeps a sorted (or hashed) structure of column values with row pointers, so lookups take O(log n) page reads instead of a full scan. Costs: storage, slower writes (every index is updated), maintenance and build-time locks.

> [!question]- Explain the leftmost-prefix rule.
> A composite index on `(a, b, c)` is sorted by `a`, then `b`, then `c`. It can efficiently serve predicates on `a`, `a,b`, or `a,b,c` (and sorting in that order), but not on `b` or `c` alone, except via PostgreSQL 18's skip scan when `a` has few distinct values.

> [!question]- What's a covering index / index-only scan?
> An index that contains every column the query needs (as keys or `INCLUDE` columns), so the database can answer from the index without visiting the table. In PostgreSQL it also needs the visibility map to be current.

> [!question]- Clustered vs non-clustered index?
> A clustered index stores the table rows in index order (one per table: InnoDB PK, SQL Server clustered index). A non-clustered index is separate and points to the rows. PostgreSQL has only non-clustered indexes on heap tables.

> [!question]- Why might the database ignore my index?
> The predicate isn't selective enough (a seq scan is estimated cheaper), statistics are stale, the query isn't sargable (function on the column, type mismatch, leading wildcard), the leading column of a composite index isn't used, or the table is tiny.

> [!question]- B-tree vs hash vs GIN vs BRIN?
> B-tree: default, equality + ranges + sorting. Hash: equality only. GIN: many keys per row (JSONB, arrays, full-text, trigrams). BRIN: tiny min/max summaries for huge tables whose column follows physical order (time series).

> [!question]- How do you add an index to a big production table without downtime?
> PostgreSQL: `CREATE INDEX CONCURRENTLY` (outside a transaction; check for and drop invalid indexes if it fails). MySQL: online DDL with `ALGORITHM=INPLACE, LOCK=NONE`. SQL Server: `ONLINE = ON`.

> [!question]- Why are random UUIDs bad primary keys, and what's the fix?
> UUIDv4 values insert at random places in the B-tree, causing page splits, poor cache locality and more WAL. Use a sequence/identity or time-ordered UUIDv7 (`uuidv7()` in PostgreSQL 18).

## Related
- [[SQL ANSI]] · [[Databases Index]]
- [[Database Indexes]] (internals) · [[Primary vs Secondary Key]] · [[Understanding Advanced SQL Features]] (book intro)
- [[Query Optimization & EXPLAIN]] · [[Using Wildcard Filtering]] (pg_trgm) · [[SQL Table Partitioning]] · [[Creating and Manipulating Tables]] (uuidv7)

## Sources
- [PostgreSQL docs: Indexes (types, multicolumn, index-only scans, partial, expression)](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL docs: CREATE INDEX (CONCURRENTLY, INCLUDE)](https://www.postgresql.org/docs/current/sql-createindex.html)
- [PostgreSQL docs: Statistics used by the planner](https://www.postgresql.org/docs/current/planner-stats.html) and [pg_stat_user_indexes](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ALL-INDEXES-VIEW)
- [PostgreSQL 18 released (skip scan, OR optimization, parallel GIN builds, uuidv7)](https://www.postgresql.org/about/news/postgresql-18-released-3142/)
- [Use The Index, Luke (Markus Winand)](https://use-the-index-luke.com/)
- [MySQL docs: How MySQL uses indexes](https://dev.mysql.com/doc/refman/8.4/en/mysql-indexes.html)
- [SQL Server docs: Clustered and nonclustered indexes](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described)
- [Stack Overflow: How does database indexing work? (source of the blocking-factor example)](https://stackoverflow.com/questions/1108/how-does-database-indexing-work)
