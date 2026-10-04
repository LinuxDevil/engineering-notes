---
title: "Using Wildcard Filtering"
tags: [databases, databases/sql]
aliases: [LIKE, Wildcards, ILIKE, pg_trgm, Pattern Matching]
updated: 2026-09-27
---
# Using Wildcard Filtering

<Note>
**TL;DR**

`LIKE` matches text against a pattern: `%` = any number of characters (including none), `_` = exactly one character. `[...]` sets are SQL Server-only. A pattern that **starts** with a wildcard (`'%bag'`) can't use a normal B-tree index, so it scans the table; for "contains" search on big tables use PostgreSQL's `pg_trgm` GIN index or full-text search.
</Note>

> Course: SQL in 10 Minutes, Lesson 6 · Prev: [Advanced Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Advanced%20Filtering%20Data) · Next: [Creating Calculated Fields](/Software%20Engineering/Databases/SQL%20ANSI/Creating%20Calculated%20Fields)

## Using the `LIKE` operator (predicate)

**Wildcards** are special characters used to match parts of a value. Wildcard searching works on text (string) columns; you can't `LIKE` a number or date without casting it first.

**Search pattern:** a condition made of literal text, wildcard characters, or both.

**`LIKE`** tells the DBMS to compare the pattern with a wildcard match instead of a straight equality match.

**Predicate:** technically `LIKE` is a *predicate* (an expression that returns true/false/unknown), not an operator. The result is the same; just know the word when you meet it in docs.

## The percent sign (`%`) wildcard

`%` means *match any number of occurrences of any character*, including zero.

<Note>
**Notes on `%`**

- **Case sensitivity** depends on the DBMS and collation. PostgreSQL `LIKE` is case-sensitive, so `'fish%'` won't match `Fish bean bag toy` (use `ILIKE` or `lower(col) LIKE`). MySQL and SQL Server with default case-insensitive collations match either case.
- `%` also matches zero characters: `'Fish%'` matches `Fish`.
- `%` does **not** match `NULL`. Even `WHERE prod_name LIKE '%'` skips rows where `prod_name` is `NULL`.
- Trailing spaces: in `CHAR` columns values are space-padded, so `'%toy'` might not match `'Fish bean bag toy     '`. Use `RTRIM()` or `VARCHAR`.
</Note>

### Example 1: starts with

```sql
SELECT prod_id, prod_name
FROM Products
WHERE prod_name LIKE 'Fish%';
```

**Analysis:** any value starting with `Fish` is retrieved; `%` accepts any characters after it, however many.

### Example 2: contains

```sql
SELECT prod_id, prod_name
FROM Products
WHERE prod_name LIKE '%bean bag%';
```

**Analysis:** matches any value containing `bean bag` anywhere, whatever comes before or after.

## The underscore (`_`) wildcard

`_` matches exactly **one** character.

### Example 3

```sql
SELECT prod_id, prod_name
FROM Products
WHERE prod_name LIKE '__ inch teddy bear';
```

**Analysis:** two underscores then literal text. Matches `12 inch teddy bear` and `18 inch teddy bear`, but not `8 inch teddy bear`, which has only one character before the space. With `%` instead, all three match:

```sql
SELECT prod_id, prod_name
FROM Products
WHERE prod_name LIKE '% inch teddy bear';
```

## The brackets (`[]`) wildcard

Brackets specify a **set** of characters, one of which must match at that position.

<Note>
**Support**

Only SQL Server (and MS Access) support `[]` in `LIKE`. MySQL, PostgreSQL, Oracle and SQLite do not. In PostgreSQL use `SIMILAR TO '[JM]%'` or a regex `~ '^[JM]'`; in MySQL `REGEXP '^[JM]'`.
</Note>

### Example 4 (SQL Server)

```sql
SELECT cust_contact
FROM Customers
WHERE cust_contact LIKE '[JM]%'
ORDER BY cust_contact;
```

**Analysis:** `[JM]` matches one character that is `J` or `M`; the `%` after it matches the rest of the name. Prefix with `^` to negate: any contact **not** starting with `J` or `M`.

```sql
SELECT cust_contact
FROM Customers
WHERE cust_contact LIKE '[^JM]%'
ORDER BY cust_contact;
```

PostgreSQL equivalents:

```sql
SELECT cust_contact FROM Customers WHERE cust_contact ~ '^[JM]';   -- regex, case-sensitive
SELECT cust_contact FROM Customers WHERE cust_contact !~ '^[JM]';  -- negated
SELECT cust_contact FROM Customers WHERE cust_contact ~* '^[jm]';  -- case-insensitive
```

## Matching a literal `%` or `_`

```sql
SELECT * FROM Products WHERE prod_desc LIKE '%100\%%' ESCAPE '\';
```

## Rules for using wildcards

1. Don't overuse wildcards. If another operator will do, use it instead.
2. Avoid wildcards at the **beginning** of a pattern unless necessary. Patterns that start with a wildcard are the slowest to process.
3. Pay attention to where you put the wildcards. Misplaced ones return the wrong data.

## Why leading wildcards are slow, and what to do

A B-tree index is sorted by the column's value, like a phone book sorted by surname.

- `LIKE 'Fish%'` is a **prefix** search: the index can jump to "Fish" and read forward. Sargable. (In PostgreSQL with a non-C collation you need an index with `text_pattern_ops` for this.)
- `LIKE '%bag'` or `'%bean%'` has no fixed prefix, so the index order is useless: full scan.

Options for "contains" search in PostgreSQL:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX products_name_trgm ON Products USING gin (prod_name gin_trgm_ops);

-- Now these can use the index
SELECT * FROM Products WHERE prod_name ILIKE '%bean bag%';
SELECT * FROM Products WHERE prod_name % 'bean bgs';  -- fuzzy similarity (typos)
```

`pg_trgm` splits text into 3-character chunks (trigrams) and indexes them, which supports `LIKE`/`ILIKE`/regex with leading wildcards and similarity search.

For word-based search (stemming, ranking), use full-text search (`tsvector`, `to_tsquery`, GIN index) or a search engine. MySQL has `FULLTEXT` indexes with `MATCH ... AGAINST`.

## Beyond the book / Senior perspective

- **User-supplied patterns** need escaping of `%` and `_`, and are still passed as parameters: `WHERE name LIKE $1` with `'%' || escaped || '%'` built in code.
- `ILIKE` is PostgreSQL-only. Portable case-insensitive search is `LOWER(col) LIKE LOWER($1)`, which needs an expression index on `LOWER(col)` to be fast.
- A "search box" with `LIKE '%term%'` on a large table is a classic production incident. Budget for trigram/FTS indexes early.
- SOUNDEX-style matching (see [Using Data Manipulation Functions](/Software%20Engineering/Databases/SQL%20ANSI/Using%20Data%20Manipulation%20Functions)) and trigram similarity are better than `LIKE` for fuzzy name search.

## Interview questions

<Accordion title="What's the difference between % and _ in LIKE?">
`%` matches zero or more characters; `_` matches exactly one.
</Accordion>

<Accordion title="Why can't `LIKE '%abc'` use a B-tree index?">
The index is ordered by the value's leading characters. Without a fixed prefix there's no range to seek to, so the database must check every row (or every index entry).
</Accordion>

<Accordion title="How would you make substring search fast in PostgreSQL?">
A GIN (or GiST) index with `pg_trgm`'s `gin_trgm_ops`, which supports `LIKE`/`ILIKE` with leading wildcards and similarity search. For word search, full-text search with a `tsvector` GIN index.
</Accordion>

<Accordion title="Does `LIKE '%'` match NULL?">
No. `NULL LIKE '%'` is UNKNOWN, so the row is filtered out.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Filtering%20Data) · [Advanced Filtering Data](/Software%20Engineering/Databases/SQL%20ANSI/Advanced%20Filtering%20Data) · [SQL Indexes Deep Dive](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20Indexes%20Deep%20Dive) · [Query Optimization & EXPLAIN](/Software%20Engineering/Databases/Core%20Concepts/Query%20Optimization%20%26%20EXPLAIN)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 6 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Pattern matching (LIKE, SIMILAR TO, regex)](https://www.postgresql.org/docs/current/functions-matching.html)
- [PostgreSQL docs: pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html)
- [SQL Server docs: LIKE](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/like-transact-sql)
- [Use The Index, Luke: LIKE filters](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning)
