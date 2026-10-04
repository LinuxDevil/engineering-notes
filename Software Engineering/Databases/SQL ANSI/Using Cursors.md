---
title: "Using Cursors"
tags: [databases, databases/sql, databases/postgres]
aliases: [Cursors, DECLARE CURSOR, FETCH, WITH HOLD, Server-side Cursors]
updated: 2026-09-27
---
# Using Cursors

<Note>
**TL;DR**

A cursor is a named result set on the server that you step through a few rows at a time: `DECLARE` it, (`OPEN` it), `FETCH` rows, `CLOSE` it. They're useful for streaming huge results without loading them all into memory. In PostgreSQL, SQL-level cursors live **inside a transaction** unless declared `WITH HOLD`, and most drivers expose them as "server-side cursors". Don't use cursors to loop and update row by row when one set-based statement would do.
</Note>

> Course: SQL in 10 Minutes, Lesson 21 · Prev: [Managing Transaction Processing](/Software%20Engineering/Databases/SQL%20ANSI/Managing%20Transaction%20Processing) · Next: [Understanding Advanced SQL Features](/Software%20Engineering/Databases/SQL%20ANSI/Understanding%20Advanced%20SQL%20Features)

## Understanding cursors

Retrieval works with sets of rows called **result sets**: all rows matching a statement, zero or more. A plain `SELECT` gives you no way to get the first row, then the next, or the previous 10. That's by design in a relational DBMS.

**Cursor:** a query stored on the DBMS server; not the `SELECT` statement itself but the **result set** it retrieves. Once the cursor exists, applications can scroll or browse through the data as needed.

<Note>
**Cursors and transactions**

The original note says "a cursor can only be declared inside a transaction". That's true for **PostgreSQL SQL-level cursors declared without `WITH HOLD`**: they exist only until the transaction ends (outside a transaction block, `DECLARE` fails). `WITH HOLD` cursors survive the commit, and SQL Server / Oracle / MySQL cursors inside procedures follow their own scoping rules. In PostgreSQL the cursor doesn't compute everything up front; rows are produced as you `FETCH` them.
</Note>

<Note>
**SQLite**

SQLite has no SQL cursor statements; its C API steps through results with `sqlite3_step()`, which plays the same role.
</Note>

Different DBMSs support different cursor options. Common ones:
1. Flag a cursor as **read-only**, so data can be read but not updated or deleted through it.
2. Control **direction**: forward, backward, first, last, absolute or relative position (a *scrollable* cursor).
3. Flag some columns as editable and others not.
4. **Scope**: make the cursor visible only to the request that created it (e.g. a stored procedure) or to all requests.
5. Make the DBMS **copy** the retrieved data (a snapshot, "insensitive" cursor) instead of pointing at live data, so it doesn't change between open and fetch.

Cursors matter most for interactive apps that scroll through data, and for processing results too large to load at once.

## Working with cursors

Steps:
1. **Declare** (define) the cursor: this names it and specifies the `SELECT` and options. No data is retrieved yet.
2. **Open** it for use (in DBMSs that separate the steps). This executes the query.
3. **Fetch** rows as needed.
4. **Close** it, and in some DBMSs **deallocate** it.

Once declared, a cursor can be opened and closed as often as needed. While open, fetches can be as frequent as needed.

## Creating cursors

The book's generic example:

```sql
DECLARE CustCursor CURSOR
FOR
SELECT * FROM Customers
WHERE cust_email IS NULL;
```

(The original note had Oracle's form `DECLARE CURSOR CustCursor IS SELECT ...`, which is close to the PL/SQL syntax `CURSOR CustCursor IS SELECT ...` inside a `DECLARE` block.)

**Analysis:** `DECLARE` names the cursor, `CustCursor`, and the `SELECT` defines its contents: all customers with no email address (`NULL`). The cursor is now ready to be opened.

## Using cursors

Cursors are opened with `OPEN`:

```sql
OPEN CustCursor;   -- SQL Server / DB2; OPEN CURSOR CustCursor in some dialects
```

When `OPEN` is processed, the query runs and the data is available for browsing and scrolling.

Rows are then read with `FETCH`, which says which rows to get, from where, and where to store them (e.g. variables). Oracle syntax to fetch one row:

```sql
-- Oracle PL/SQL
DECLARE
    CURSOR CustCursor IS
        SELECT * FROM Customers WHERE cust_email IS NULL;
    CustRecord Customers%ROWTYPE;
BEGIN
    OPEN CustCursor;
    FETCH CustCursor INTO CustRecord;
    CLOSE CustCursor;
END;
```

**Analysis:** `FETCH` retrieves the current row (starting at the first automatically) into the variable `CustRecord`. Nothing is done with it in this example. In practice you'd loop until no more rows are found (`EXIT WHEN CustCursor%NOTFOUND;`).

## PostgreSQL cursors

PostgreSQL has no separate `OPEN` at SQL level: `DECLARE` both defines and opens.

```sql
BEGIN;

DECLARE cust_cur CURSOR FOR
    SELECT cust_id, cust_name
    FROM Customers
    WHERE cust_email IS NULL
    ORDER BY cust_id;

FETCH 100 FROM cust_cur;        -- first 100 rows
FETCH NEXT FROM cust_cur;       -- one more
MOVE FORWARD 50 IN cust_cur;    -- skip 50 without returning them

CLOSE cust_cur;
COMMIT;                         -- open non-holdable cursors are closed at commit anyway
```

Options:

| Option | Meaning |
|---|---|
| `SCROLL` / `NO SCROLL` | Allow fetching backwards (`FETCH PRIOR`, `FETCH ABSOLUTE n`). Scrolling may make the query slower. |
| `WITH HOLD` | Keep the cursor after `COMMIT`. PostgreSQL materializes the remaining result at commit time, which can be expensive; close it explicitly when done. |
| `WITHOUT HOLD` (default) | Cursor dies with the transaction. |
| `BINARY` | Return rows in binary format. |
| `FOR UPDATE` in the query | Lets you `UPDATE ... WHERE CURRENT OF cust_cur` on the row under the cursor. |

```sql
-- A cursor that survives the transaction
DECLARE export_cur CURSOR WITH HOLD FOR SELECT * FROM Orders ORDER BY order_num;
FETCH 1000 FROM export_cur;
...
CLOSE export_cur;
```

In PL/pgSQL, a `FOR` loop over a query uses a cursor implicitly, which is usually all you need:

```sql
DO $$
DECLARE r record;
BEGIN
  FOR r IN SELECT cust_id, cust_name FROM Customers WHERE cust_email IS NULL LOOP
    RAISE NOTICE 'No email for % (%)', r.cust_name, r.cust_id;
  END LOOP;
END $$;
```

## Closing cursors

Cursors must be closed after use; some DBMSs (SQL Server) also require explicit deallocation of their resources.

DB2, Oracle, PostgreSQL:

```sql
CLOSE CustCursor;
```

SQL Server:

```sql
CLOSE CustCursor;
DEALLOCATE CustCursor;
```

**Analysis:** `CLOSE` closes the cursor; it can't be used again until reopened. It doesn't need to be declared again; `OPEN` is enough (in DBMSs with separate `OPEN`; in PostgreSQL you re-`DECLARE`).

## Server-side cursors in drivers

The main modern use of cursors is **streaming** a large result into an application without buffering it all:

```python
# psycopg 3: a named cursor is a server-side cursor
with conn.cursor(name="big_export") as cur:
    cur.itersize = 5000                 # rows per network round trip
    cur.execute("SELECT * FROM events WHERE created_at >= %s", (since,))
    for row in cur:
        process(row)
```

```js
// node-postgres with pg-cursor
const cursor = client.query(new Cursor('SELECT * FROM events'));
let rows;
while ((rows = await cursor.read(5000)).length) { rows.forEach(process); }
```

JDBC with PostgreSQL uses a cursor when autocommit is off and `setFetchSize(n)` is set; MySQL Connector/J needs `useCursorFetch=true` or streaming mode. By default, most drivers read the **entire** result into memory, which is how "SELECT * FROM big_table" takes down an app server.

<Warning>
**Connection poolers**

PgBouncer in transaction-pooling mode doesn't support `WITH HOLD` cursors or cursors spanning transactions. Keep cursors within one transaction.
</Warning>

## Beyond the book / Senior perspective

- **Set-based beats row-by-row.** A cursor loop that updates rows one at a time ("RBAR", row by agonizing row) is often 10-100x slower than one `UPDATE ... FROM` or `INSERT ... SELECT`. Use cursors for streaming output or genuinely sequential logic, not as a `for` loop over data.
- **Cursors hold resources.** An open cursor keeps a snapshot (in PostgreSQL, blocking `VACUUM` cleanup) and possibly locks. Consume and close promptly.
- **Cursors vs pagination.** For user-facing paging, don't keep a cursor open between HTTP requests; use keyset pagination ([Retrieving Information](/Software%20Engineering/Databases/SQL%20ANSI/Retrieving%20Information)). Cursors fit batch jobs and exports.
- SQL Server cursor types (`STATIC`, `KEYSET`, `DYNAMIC`, `FAST_FORWARD`) change consistency and cost; `FAST_FORWARD READ_ONLY` is the cheap default for simple loops.

## Interview questions

<Accordion title="What is a database cursor and when would you use one?">
A server-side handle to a query's result set that you fetch from incrementally. Use it to stream very large results with bounded memory, or for procedural row-at-a-time logic that can't be expressed set-based.
</Accordion>

<Accordion title="Why are cursors often considered an anti-pattern?">
They encourage row-by-row processing, which is much slower than set-based SQL, and they hold snapshots/locks for their lifetime.
</Accordion>

<Accordion title="What does WITH HOLD do in PostgreSQL?">
Keeps the cursor usable after the transaction that created it commits; PostgreSQL materializes the remaining rows at commit. Without it, the cursor is closed at transaction end.
</Accordion>

<Accordion title="How do you process a 100 GB table in an application without running out of memory?">
Use a server-side cursor (named cursor in psycopg, fetch size in JDBC, pg-cursor in Node) or keyset-paginated batches, processing rows in chunks.
</Accordion>

## Related
- [SQL ANSI](/Software%20Engineering/Databases/SQL%20ANSI/SQL%20ANSI) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Managing Transaction Processing](/Software%20Engineering/Databases/SQL%20ANSI/Managing%20Transaction%20Processing) · [Working with Stored Procedures](/Software%20Engineering/Databases/SQL%20ANSI/Working%20with%20Stored%20Procedures) · [Retrieving Information](/Software%20Engineering/Databases/SQL%20ANSI/Retrieving%20Information) · [Connection Pooling](/Software%20Engineering/Databases/Database%20Round/Connection%20Pooling)

## Sources
- Ben Forta, *Sams Teach Yourself SQL in 10 Minutes a Day*, 5th ed., Lesson 21 ([O'Reilly](https://learning.oreilly.com/library/view/sams-teach-yourself/9780135182925/))
- [PostgreSQL docs: DECLARE](https://www.postgresql.org/docs/current/sql-declare.html), [FETCH](https://www.postgresql.org/docs/current/sql-fetch.html)
- [PostgreSQL docs: PL/pgSQL cursors](https://www.postgresql.org/docs/current/plpgsql-cursors.html)
- [psycopg 3 docs: Server-side cursors](https://www.psycopg.org/psycopg3/docs/advanced/cursors.html)
- [PgBouncer docs: Feature matrix for pooling modes](https://www.pgbouncer.org/features.html)
