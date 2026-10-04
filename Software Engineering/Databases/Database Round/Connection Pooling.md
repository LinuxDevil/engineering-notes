---
title: "Connection Pooling"
tags: [databases, databases/operations, databases/postgres]
aliases: [Connection Pool, Pooling, PgBouncer, Pool Sizing, HikariCP]
updated: 2026-09-27
---
# Connection Pooling

<Note>
**TL;DR**

Opening a database connection is expensive (TCP + TLS handshake, authentication, and in Postgres a whole new server **process**). A **connection pool** opens a fixed set of connections once and lends them to requests. Pools live in the app (node-postgres `Pool`, HikariCP) and/or in a proxy (**PgBouncer**, RDS Proxy, Supavisor). Size pools **small**: roughly a few connections per CPU core of the database, not per user. The total across all app instances must stay under the DB's `max_connections`.
</Note>

## The idea

**Connection pooling** is the pattern of creating a pool of connections (usually TCP connections) and letting many clients share it.

The old way: on every request, open a connection to the database, run the query, close it. Very expensive: tens of milliseconds of setup for a query that takes one.

The better way: create a **pool** (not a single client) at startup, connected once, with a maximum size such as 20. When you run a query, the pool hands you one of its idle connections, and you give it back when done. If all 20 are busy, you wait in a queue.

{/* diagram source: /diagrams/diagram-114.mmd */}
![diagram](/diagrams/diagram-114.svg)

## Worked example: node-postgres

```js
import pg from 'pg';

const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                        // maximum connections in this pool (default 10)
  idleTimeoutMillis: 30_000,      // close connections idle this long (default 10s)
  connectionTimeoutMillis: 2_000, // how long to wait for a free connection before erroring (default 0 = forever)
});

// simple query: pool acquires and releases for you
const { rows } = await pool.query('SELECT * FROM users WHERE id = $1', [id]);

// transaction: you must hold ONE connection for all statements
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [100, 1]);
  await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [100, 2]);
  await client.query('COMMIT');
} catch (e) {
  await client.query('ROLLBACK');
  throw e;
} finally {
  client.release();               // always release, or the pool leaks and eventually hangs
}
```

The user's original parameters:

- **`connectionTimeoutMillis`**: how long to wait for a connection to become available when all are in use.
- **`idleTimeoutMillis`**: when to get rid of connections that have been idle.

Java equivalent (HikariCP, the Spring Boot default): `maximumPoolSize`, `minimumIdle`, `connectionTimeout`, `idleTimeout`, `maxLifetime`.

## Pool sizing

More connections is **not** faster. The database has a fixed number of cores and disks; beyond that, extra connections just contend for CPU, locks and memory (each Postgres backend uses several MB plus `work_mem` per sort/hash).

A widely cited starting point (from the PostgreSQL wiki, popularised by HikariCP):

```text
connections ≈ (core_count × 2) + effective_spindle_count
```

On an 8-core DB server with SSDs that's roughly 16–20 active connections, then measure. Little's law helps: `connections needed ≈ throughput × average query time`. At 2,000 queries/s of 5 ms each you need about 10 busy connections.

Budget across the fleet:

```text
app instances × pool max  +  admin/migration/replication headroom  ≤  max_connections
     30       ×    10      +              20                       =  320  → too many for a 16-core DB
```

That's where an external pooler comes in.

## External poolers: PgBouncer

PgBouncer sits between apps and Postgres, accepts thousands of cheap client connections and multiplexes them over a small number of server connections.

| Mode | Server connection is held for | Works with | Breaks |
|---|---|---|---|
| **Session** | the whole client session | everything | little multiplexing benefit |
| **Transaction** | one transaction | most web apps; the usual choice | session state: `SET` (non-local), session advisory locks, `LISTEN`, temp tables across transactions, `WITH HOLD` cursors |
| **Statement** | one statement | autocommit-only workloads | multi-statement transactions |

```ini
; pgbouncer.ini (excerpt)
[databases]
app = host=10.0.0.5 port=5432 dbname=app

[pgbouncer]
listen_port = 6432
pool_mode = transaction
max_client_conn = 5000        ; client side: cheap
default_pool_size = 20        ; server connections per user/database pair
max_prepared_statements = 200 ; protocol-level prepared statements in transaction mode (1.21+)
auth_type = scram-sha-256
```

Other options: **RDS Proxy** / **Azure** and **Cloud SQL** built-in poolers, **Supavisor** (Supabase), **PgCat** and **Odyssey** (multi-threaded, sharding/load-balancing features), and serverless drivers over HTTP/WebSockets (Neon, PlanetScale) for edge functions.

## Pitfalls

<Warning>
**Common pitfalls**

- **Leaked connections**: forgetting `release()` on an error path; the pool drains and every request times out. Use `try/finally` or helpers.
- **Pool per request** or per function invocation (serverless): recreates the problem pooling solves. Create the pool once per process; in serverless use an external pooler or HTTP driver.
- **Oversized pools**: 100s of connections per instance × many instances exhaust `max_connections` and degrade throughput.
- **Holding a connection during slow non-DB work** (HTTP calls) starves the pool.
- **Session state behind transaction pooling**: `SET search_path`, session advisory locks, prepared statements on old PgBouncer versions. Use `SET LOCAL` inside the transaction and `pg_advisory_xact_lock`.
- **Stale connections** after failover or firewall idle timeouts: set `maxLifetime`/`idleTimeout` below infrastructure timeouts and enable TCP keepalives.
</Warning>

## Senior perspective

- Watch pool metrics: wait time to acquire, active vs idle, timeouts. A growing acquire time means the DB (or your transactions) is the bottleneck, not the pool size.
- Put timeouts at every layer: `connectionTimeoutMillis` in the app, `statement_timeout` and `idle_in_transaction_session_timeout` in Postgres, `query_wait_timeout` in PgBouncer.
- Reads vs writes: separate pools for the primary and replicas; route read-only queries to replicas only if they tolerate lag.
- MongoDB drivers pool internally too (`maxPoolSize` default 100); the same sizing logic applies. See [Unit 06 Connecting to MongoDB in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2006%20Connecting%20to%20MongoDB%20in%20Node.js).
- Review questions: What's the max total connections across all replicas of this service at peak autoscale? What happens when the pool is exhausted: fail fast or queue forever?

<Info>
**🆕 2025–2026**

- **PgBouncer 1.25.0 (2025-11-09)** added LDAP authentication, client-side direct TLS connections, much faster SCRAM authentication, a `pool_idle_timeout` setting, per-user/database `query_wait_timeout`, and removed the deprecated online restart (`-R`).
- **PgBouncer 1.25.1 (2025-12-03)** fixed **CVE-2025-12819** (an unauthenticated attacker could run SQL during authentication via a malicious `search_path` startup parameter). Later 1.25.x releases fixed further denial-of-service CVEs. Keep poolers patched; they're internet-adjacent.
- **PostgreSQL 18 (2025-09-25)** introduced OAuth authentication and deprecated MD5 passwords (use SCRAM), which affects pooler `auth_type` configuration.
</Info>

## Interview questions

<Accordion title="Why is opening a DB connection per request bad?">
Each connection costs a TCP (and TLS) handshake, authentication and, in Postgres, forking a backend process with its own memory. That can dwarf query time and limits throughput; many concurrent connections also overload the server.
</Accordion>

<Accordion title="How do you size a connection pool?">
Start near (DB cores × 2) + disks for total active connections, divide across app instances, keep the fleet total under `max_connections` with headroom, then tune using acquire-wait and DB utilisation metrics. Little's law (throughput × latency) estimates busy connections.
</Accordion>

<Accordion title="What are PgBouncer's pooling modes and their trade-offs?">
Session (a server connection per client session, compatible but little sharing), transaction (server connection per transaction, best multiplexing, breaks session state) and statement (per statement, no multi-statement transactions).
</Accordion>

<Accordion title="Your service times out acquiring DB connections under load. What do you check?">
Leaked connections (missing release), long transactions or slow queries holding connections, external calls inside transactions, pool size vs DB capacity, lock waits in `pg_stat_activity`, and whether autoscaling multiplied pools past `max_connections`.
</Accordion>

<Accordion title="How do you use Postgres from serverless functions?">
Create the client outside the handler for reuse, and put an external pooler (PgBouncer, RDS Proxy, Supavisor) or an HTTP/WebSocket driver in front, since each function instance would otherwise open its own connections.
</Accordion>

## Related
- [Databases Index](/Software%20Engineering/Databases/Databases%20Index) · [Database Round](/Software%20Engineering/Databases/Database%20Round/Database%20Round)
- [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Locking & Concurrency Control](/Software%20Engineering/Databases/Core%20Concepts/Locking%20%26%20Concurrency%20Control) · [Clustering and Scaling](/Software%20Engineering/Databases/Database%20Round/Clustering%20and%20Scaling) · [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC)
- [Unit 06 Connecting to MongoDB in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2006%20Connecting%20to%20MongoDB%20in%20Node.js)

## Sources
- node-postgres, Pooling: https://node-postgres.com/features/pooling · Pool API: https://node-postgres.com/apis/pool
- HikariCP wiki, About Pool Sizing: https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing
- PostgreSQL wiki, Number Of Database Connections: https://wiki.postgresql.org/wiki/Number_Of_Database_Connections
- PgBouncer docs and changelog: https://www.pgbouncer.org/config.html · https://www.pgbouncer.org/changelog.html
- PgBouncer 1.25.0 / 1.25.1 announcements: https://postgresql.org/about/news/pgbouncer-1250-released-3169/ · https://www.postgresql.org/about/news/pgbouncer-1251-released-fixing-a-bunch-of-bugs-before-christmas-including-cve-2025-12819-3189/
- PostgreSQL 18 release: https://www.postgresql.org/about/news/postgresql-18-released-3142/
- Hussein Nasser, *Fundamentals of Database Engineering*
