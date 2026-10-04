---
title: "Row Vs Column Oriented Databases"
tags: [databases, databases/internals]
aliases: [Row Store, Column Store, Columnar Database, OLTP vs OLAP, Row vs Column]
updated: 2026-09-27
---
# Row Vs Column Oriented Databases

<Note>
**TL;DR**

A **row store** writes each row's columns together, so one page read returns whole records: ideal for **OLTP** (fetch/update a few rows by key). A **column store** writes each column's values together, so an aggregate over one column reads only that column and compresses it heavily: ideal for **OLAP** (scan millions of rows, few columns). Row stores: Postgres, MySQL, SQL Server, Oracle. Column stores: ClickHouse, DuckDB, Snowflake, BigQuery, Redshift, Parquet files. Many systems now blend both (HTAP).
</Note>

## Row-oriented database (row store)

![Screenshot_2023-07-23_at_4.48.37_PM](/Attachments/Screenshot_2023-07-23_at_4.48.37_PM.png)

- Rows are stored one after another on disk (first row, second row, ...); a row can span multiple blocks if it's large.

```text
1001,1,John,Smith,111,101000,1/1/1991,eng,1/1/2011 | 1002,2,Kary,White,222,102000,2/2/1992,mgr,2/1/2012 | ...
```

- A single block IO fetches **several rows with all their columns**. To find one row without an index you may need many IOs, but once you find it, you have every column.

Queries against `emp` with **no indexes**:

```sql
SELECT first_name FROM emp WHERE ssn = 666;   -- Q1
SELECT * FROM emp WHERE id = 1;               -- Q2
SELECT SUM(salary) FROM emp;                  -- Q3
```

**Q1**: scan the heap page by page, checking `ssn` in each row until the match (and to the end if `ssn` isn't unique). Once found, `first_name` is right there in the same row.

![Screenshot_2023-07-23_at_4.56.08_PM](/Attachments/Screenshot_2023-07-23_at_4.56.08_PM.png)

**Q2**: same scan on `id`; `SELECT *` is cheap once the row is found because all columns are together. (If ids were a known sequence, the engine could map an id to a row id directly.)

![Screenshot_2023-07-23_at_4.58.05_PM](/Attachments/Screenshot_2023-07-23_at_4.58.05_PM.png)

**Q3**: read **every block** and sum `salary` from each row, dragging all the other columns through memory for nothing.

![Screenshot_2023-07-23_at_4.59.14_PM](/Attachments/Screenshot_2023-07-23_at_4.59.14_PM.png)

## Column-oriented database (column store)

![Screenshot_2023-07-23_at_5.03.16_PM](/Attachments/Screenshot_2023-07-23_at_5.03.16_PM.png)

- Each column is stored contiguously. Each value is tied to its row id (in real engines the row id is usually implicit: the value's position in the column).

```text
id:         1001:1, 1002:2, 1003:3, 1004:4, ...
first_name: John:1001, Kary:1002, Norman:1003, ...
last_name:  Smith:1001, White:1002, Freeman:1003, ...
```

- One IO reads **many values of one column**. Few IOs to aggregate a column; more IOs to reassemble a whole row across many columns.
- **DML is expensive**: inserting one row touches every column's storage; updates/deletes are usually handled by delta stores and background merges.

Same queries:

**Q1**: read the `ssn` column blocks until 666 is found (row id 1006, say), then jump to the `first_name` column and read the value for row 1006. Two columns touched, the rest untouched.

![Screenshot_2023-07-23_at_5.08.22_PM](/Attachments/Screenshot_2023-07-23_at_5.08.22_PM.png)

**Q2**: find id 1 in the `id` column, then visit **every** column to rebuild the row. `SELECT *` is the worst thing you can do to a column store.

![Screenshot_2023-07-23_at_5.11.17_PM](/Attachments/Screenshot_2023-07-23_at_5.11.17_PM.png)

**Q3**: read only the `salary` column: a handful of blocks. Column stores also **compress**: repeated values become one value with a list/range of row ids (run-length encoding, dictionary encoding), so even fewer bytes are read.

![Screenshot_2023-07-23_at_5.13.44_PM](/Attachments/Screenshot_2023-07-23_at_5.13.44_PM.png)

## Pros and cons: OLTP vs OLAP

![Screenshot_2023-07-23_at_5.15.37_PM](/Attachments/Screenshot_2023-07-23_at_5.15.37_PM.png)

| | Row store | Column store |
|---|---|---|
| Workload | **OLTP** (Online Transaction Processing): many small reads/writes by key | **OLAP** (Online Analytical Processing): big scans and aggregates |
| Single-row lookup / `SELECT *` | Fast | Slow (reassemble from many columns) |
| Insert/update/delete | Fast | Slow; batch-oriented |
| Aggregates over few columns | Reads everything | Reads only needed columns |
| Compression | Moderate | Excellent (similar values adjacent) |
| Execution style | Row at a time (Volcano) | Vectorized, SIMD over column chunks |
| Examples | PostgreSQL, MySQL, SQL Server, Oracle | ClickHouse, DuckDB, Snowflake, BigQuery, Redshift, Vertica, Apache Parquet/ORC files |

## Hybrids

- **Columnar extensions / indexes in row stores**: SQL Server columnstore indexes, Oracle In-Memory column store, MySQL HeatWave, Postgres extensions such as Citus columnar and Hydra, and DuckDB/pg_duckdb reading Postgres data.
- **HTAP** systems (TiDB with TiFlash, SingleStore) keep a row copy for transactions and a column copy for analytics.
- **Lakehouse**: columnar files (Parquet) on object storage with table formats (Apache Iceberg, Delta Lake) queried by many engines.

## Pitfalls

<Warning>
**Common pitfalls**

- Running heavy analytics on the primary OLTP database: scans evict the hot working set from cache and compete for IO. Use a replica, a warehouse, or CDC into a column store.
- Treating a column store like OLTP: single-row inserts in a loop and frequent updates kill performance. Batch inserts, append-only designs.
- `SELECT *` on wide columnar tables reads every column file.
</Warning>

## Senior perspective

- The usual architecture: OLTP row store as source of truth → CDC (Debezium, logical replication) or batch ELT → columnar warehouse/lakehouse for analytics.
- For small-to-medium analytics, embedded engines like **DuckDB** querying Parquet can replace a warehouse.
- Questions in design review: What's the read/write ratio? Are queries "few rows, all columns" or "all rows, few columns"? How fresh must analytics be?

<Info>
**🆕 2025–2026**

- **DuckDB 1.4.0 "Andium" (September 2025)** was DuckDB's first **LTS** release, supported for one year; from then on every other release is LTS. **DuckDB 1.5.0 (2026-03-09)** added a `VARIANT` type and a built-in `GEOMETRY` type.
- **Snowflake acquired Crunchy Data** (announced 2025-06-02, about $250M) and **Databricks acquired Neon** (announced 2025-05-14, about $1B): analytics platforms buying Postgres (row-store OLTP) companies to offer both sides. See [PostgreSQL 17 to 18 - What's New](/Software%20Engineering/Databases/Core%20Concepts/PostgreSQL%2017%20to%2018%20-%20What's%20New).
</Info>

## Interview questions

<Accordion title="Why are column stores faster for analytics?">
They read only the columns a query needs, compress adjacent similar values very well, and process columns in vectorized batches with SIMD. For `SUM(salary)` over 100 columns, a column store reads 1 column instead of 100.
</Accordion>

<Accordion title="Why are column stores bad at OLTP?">
Inserting or updating one row touches every column's storage; fetching a full row requires reading many column segments. They're designed for batch writes and large scans.
</Accordion>

<Accordion title="What is OLTP vs OLAP?">
OLTP: many short transactions reading/writing a few rows (orders, payments), latency-sensitive, normalized schemas. OLAP: complex queries scanning large volumes for reports and analytics, throughput-oriented, often denormalized star schemas.
</Accordion>

<Accordion title="How would you add analytics to a Postgres-backed product?">
Start with a read replica and materialized views; as volume grows, stream changes (logical replication/CDC) into a columnar store (ClickHouse, a warehouse, or Parquet + DuckDB) and serve dashboards from there, keeping the OLTP primary free of heavy scans.
</Accordion>

## Related
- [Databases Index](/Software%20Engineering/Databases/Databases%20Index) · [Database Round](/Software%20Engineering/Databases/Database%20Round/Database%20Round)
- [Database Pages](/Software%20Engineering/Databases/Database%20Round/Database%20Pages) · [Database Internals](/Software%20Engineering/Databases/Database%20Round/Database%20Internals) · [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types) · [Data Modeling & Normalization](/Software%20Engineering/Databases/Core%20Concepts/Data%20Modeling%20%26%20Normalization)

## Sources
- Abadi, Boncz, Harizopoulos, *The Design and Implementation of Modern Column-Oriented Database Systems* (2013): https://stratos.seas.harvard.edu/files/stratos/files/columnstoresfntdbs.pdf
- ClickHouse docs, What is a columnar database: https://clickhouse.com/docs/en/intro
- Microsoft docs, Columnstore indexes overview: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview
- DuckDB 1.5.0 announcement: https://duckdb.org/2026/03/09/announcing-duckdb-150 · Release cycle: https://duckdb.org/docs/current/dev/release_cycle
- TechCrunch, Databricks to buy Neon: https://techcrunch.com/2025/05/14/databricks-to-buy-open-source-database-startup-neon-for-1b/
- CNBC, Snowflake to buy Crunchy Data: https://www.cnbc.com/2025/06/02/snowflake-to-buy-crunchy-data-250-million.html
- Hussein Nasser, *Fundamentals of Database Engineering*
