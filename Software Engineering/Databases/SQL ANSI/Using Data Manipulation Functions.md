---
tags: [databases, databases/sql]
aliases: [SQL Functions, Text Functions, Date Functions, EXTRACT, CAST, SOUNDEX]
updated: 2026-09-27
---
# Using Data Manipulation Functions

> [!tldr] TL;DR
> SQL functions transform values: text (`UPPER`, `SUBSTRING`, `TRIM`), numbers (`ABS`, `ROUND`), dates (`EXTRACT`, `CURRENT_DATE`), conversion (`CAST`) and system info. Unlike core statements, **function names differ a lot between DBMSs**, so prefer standard ones (`CAST`, `EXTRACT`, `SUBSTRING`, `COALESCE`, `CURRENT_DATE`). Wrapping a column in a function inside `WHERE` usually stops index use; filter on ranges instead.

> Course: SQL in 10 Minutes, Lesson 8 · Prev: [[Creating Calculated Fields]] · Next: [[Summarizing Data]]

## Understanding functions

Like most languages, SQL has functions to manipulate data, usually for conversion and formatting. `RTRIM()` from the previous lesson is one.

Unlike statements such as `SELECT`, which work much the same everywhere, **functions are very DBMS-specific**. The functionality exists almost everywhere, but names and syntax vary.

| Need | Syntax by DBMS |
|---|---|
| Datatype conversion | Standard `CAST(x AS type)`: DB2, PostgreSQL, SQL Server, MySQL, Oracle, SQLite. `CONVERT()`: MariaDB, MySQL, SQL Server. Oracle also has `TO_CHAR`, `TO_NUMBER`, `TO_DATE`. PostgreSQL also has `x::type`. |
| Extract part of a string | `SUBSTR()`: DB2, Oracle, PostgreSQL, SQLite. `SUBSTRING()`: MariaDB, MySQL, SQL Server, PostgreSQL. |
| Current date | `CURRENT_DATE`: DB2, PostgreSQL, MySQL, SQLite, Oracle. `CURDATE()`: MySQL/MariaDB. `SYSDATE`: Oracle. `GETDATE()` / `CAST(GETDATE() AS date)`: SQL Server. `DATE('now')`: SQLite. |

**Portable code:** code written so that it runs on several different systems. With functions, that's hard; isolate DBMS-specific calls (views, a data-access layer) and comment them.

## Using functions

Function types:
1. **Text** functions manipulate strings (trim, pad, upper/lower case).
2. **Numeric** functions do maths (absolute values, rounding, trigonometry).
3. **Date and time** functions manipulate dates and extract parts (differences between dates, validity checks).
4. **Formatting** functions produce user-friendly output (localized dates, currencies).
5. **System** functions return DBMS-specific information (current user, version).

## Text manipulation functions

### Example 1

```sql
SELECT vend_name, UPPER(vend_name) AS vend_name_upcase
FROM Vendors
ORDER BY vend_name;
```

**Analysis:** `UPPER()` converts text to uppercase, so each vendor appears twice: as stored, and uppercased as `vend_name_upcase`.

Commonly used text functions:

| Function | Description |
|---|---|
| `LEFT()` (or a substring function) | Characters from the left of a string |
| `LENGTH()` (also `DATALENGTH()` or `LEN()` in SQL Server; `CHAR_LENGTH()` standard) | Length of a string |
| `LOWER()` | Converts to lowercase |
| `LTRIM()` | Trims white space from the left |
| `RIGHT()` (or a substring function) | Characters from the right of a string |
| `RTRIM()` | Trims white space from the right |
| `TRIM()` | Trims both sides |
| `SOUNDEX()` | A string's SOUNDEX value |
| `SUBSTR()` or `SUBSTRING()` | Extracts part of a string |
| `UPPER()` | Converts to uppercase |
| `REPLACE()` | Replaces all occurrences of a substring |

> [!note] Bytes vs characters
> MySQL `LENGTH()` returns **bytes**, `CHAR_LENGTH()` returns characters; with UTF-8, `'é'` is 2 bytes. PostgreSQL `LENGTH()` returns characters, `OCTET_LENGTH()` bytes.

**SOUNDEX** is an algorithm that converts a string into an alphanumeric code describing how it sounds, so similar-sounding words can be compared (`Michelle` and `Michael` share a code). It's not a SQL concept, but most DBMSs support it (PostgreSQL through the `fuzzystrmatch` extension).

```sql
-- Finds 'Michelle Green' even if it was typed as 'Michael Green'
SELECT cust_name, cust_contact
FROM Customers
WHERE SOUNDEX(cust_contact) = SOUNDEX('Michael Green');
```

## Date and time manipulation functions

Dates are stored in special internal formats so they can be sorted and filtered quickly; to use their parts you need functions, and these are the least portable of all.

### Example 2: extract the year

```sql
/* PostgreSQL */
SELECT order_num
FROM Orders
WHERE DATE_PART('year', order_date) = 2020;

/* Standard SQL: PostgreSQL, MySQL, Oracle, DB2 */
SELECT order_num
FROM Orders
WHERE EXTRACT(year FROM order_date) = 2020;
```

**Analysis:** `EXTRACT()` pulls out part of the date (`year` here) and compares it with `2020`.

### Example 3: a date range

```sql
/* Oracle (and PostgreSQL, which also has to_date) */
SELECT order_num
FROM Orders
WHERE order_date BETWEEN to_date('2020-01-01', 'yyyy-mm-dd')
                     AND to_date('2020-12-31', 'yyyy-mm-dd');
```

**Analysis:** `to_date()` converts the two strings into dates, and `BETWEEN` finds orders between them. SQL Server has no `to_date()`; there you'd write `WHERE DATEPART(year, order_date) = 2020` or compare against date literals.

DB2, MySQL and MariaDB have a `YEAR()` function:

```sql
SELECT order_num
FROM Orders
WHERE YEAR(order_date) = 2020;
```

SQLite is a bit trickier:

```sql
SELECT order_num
FROM Orders
WHERE strftime('%Y', order_date) = '2020';
```

To filter by month, add a second condition on the month in the same way.

> [!tip] The sargable way (works everywhere, uses indexes)
> ```sql
> SELECT order_num
> FROM Orders
> WHERE order_date >= DATE '2020-01-01'
>   AND order_date <  DATE '2021-01-01';
> ```
> `YEAR(order_date) = 2020` or `EXTRACT(...)` must compute the function for every row, so an index on `order_date` can't be used. The half-open range leaves the column bare and also handles timestamps with a time part correctly.

## Numeric manipulation functions

| Function | Description |
|---|---|
| `ABS()` | Absolute value |
| `EXP()` | Exponential value |
| `SQRT()` | Square root |
| `COS()` / `SIN()` / `TAN()` | Trigonometric cosine / sine / tangent of an angle |
| `PI()` | The value of pi |
| `ROUND(x, n)` | Round to `n` decimal places |
| `CEIL()` / `FLOOR()` | Round up / down to an integer |
| `MOD(a, b)` or `a % b` | Remainder |

## Beyond the book / Senior perspective

- **Time zones.** Store instants as `timestamptz` (PostgreSQL) / `DATETIME2` + offset or UTC (SQL Server) / `TIMESTAMP` in UTC (MySQL). `timestamp without time zone` is a wall-clock value with no zone, a common source of bugs around DST.
- **Date truncation for reports:** `date_trunc('month', order_date)` (PostgreSQL), `DATE_FORMAT` (MySQL), `DATETRUNC` (SQL Server 2022+). Group by the truncated value.
- **Expression indexes** make function-based filters fast when you can't rewrite them: `CREATE INDEX ON customers (lower(email));` then `WHERE lower(email) = lower($1)`. See [[SQL Indexes Deep Dive]].
- **Volatile functions** like `now()`/`random()` in PostgreSQL: `now()` is fixed for the whole transaction; `clock_timestamp()` changes per call.
- **Portability vs power:** most teams pick one DBMS and use its functions fully. Portability matters mainly for libraries and products that ship on several databases.

## Interview questions

> [!question]- Why is `WHERE YEAR(order_date) = 2020` slow on a big table?
> The function must be evaluated for every row, so a B-tree index on `order_date` can't be used (non-sargable). Rewrite as `order_date >= '2020-01-01' AND order_date < '2021-01-01'`, or create an expression index.

> [!question]- `CAST` vs `CONVERT` vs `::`?
> `CAST(x AS type)` is standard and portable. `CONVERT` is SQL Server/MySQL-specific (SQL Server's version also takes a style for date formatting). `::` is PostgreSQL shorthand.

> [!question]- `timestamp` vs `timestamptz` in PostgreSQL?
> `timestamptz` stores an absolute instant (normalized to UTC) and converts to the session time zone on output. `timestamp` stores a zone-less wall-clock value. Use `timestamptz` for events.

> [!question]- What does SOUNDEX do?
> Encodes a string by its pronunciation (a letter plus three digits), so similar-sounding names compare equal. Useful for fuzzy name matching; trigram similarity is often better.

## Related
- [[SQL ANSI]] · [[Databases Index]]
- [[Creating Calculated Fields]] · [[Summarizing Data]] · [[Filtering Data]] · [[SQL Indexes Deep Dive]]

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 8 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: Date/time functions](https://www.postgresql.org/docs/current/functions-datetime.html)
- [PostgreSQL docs: fuzzystrmatch (soundex)](https://www.postgresql.org/docs/current/fuzzystrmatch.html)
- [MySQL docs: Date and time functions](https://dev.mysql.com/doc/refman/8.4/en/date-and-time-functions.html)
- [SQLite docs: Date and time functions](https://www.sqlite.org/lang_datefunc.html)
