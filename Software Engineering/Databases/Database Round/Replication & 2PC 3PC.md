---
title: "Replication & 2PC 3PC"
tags: [databases, databases/distributed, databases/postgres]
aliases: [Replication, Master-Slave, Leader-Follower, Primary-Replica, Multi-Master, Synchronous Replication, Asynchronous Replication, Replication Lag, 2PC, Two-Phase Commit, 3PC, Three-Phase Commit, Distributed Transactions, Saga]
updated: 2026-09-27
---
# Replication & 2PC 3PC

<Note>
**TL;DR**

**Replication** keeps copies of the same data on several nodes for availability, lower read latency and read scaling. Three shapes: **single-leader** (one primary takes writes, followers copy its log), **multi-leader** (several nodes take writes, conflicts must be resolved) and **leaderless** (any node takes writes, quorums). Copies are **synchronous** (safe, slower) or **asynchronous** (fast, can lose recent writes, followers lag). **Distributed transactions** across nodes need an atomic commit protocol: **2PC** is correct but blocks if the coordinator dies; **3PC** avoids blocking only under unrealistic network assumptions. In practice, microservices use **sagas** and the **outbox pattern** instead.
</Note>

This note merges the course notes with the original *Overview 1* (a summary of Kleppmann, *DDIA* ch. 5 and 12) and the distributed-database part of *Overview* (Silberschatz).

## Replication

Replication means keeping multiple copies of a database and sharing changes between those duplicate instances to keep them consistent. A set of replicated nodes is usually called a **cluster** (see [Clustering and Scaling](/Software%20Engineering/Databases/Database%20Round/Clustering%20and%20Scaling)).

**Why replicate?**

1. **Availability**: survive a node or zone failure.
2. **Latency**: serve users from a nearby replica.
3. **Scalability**: spread reads across replicas (load-balance them), keep some replicas for reporting.

## Leaders and followers (single-leader replication)

Also called master-slave, primary-replica, active/passive, master/backup, or one-way replication.

- **One leader** accepts all writes and DDL.
- **One or more followers** receive the leader's change stream and apply it. Reads can go to the leader or the followers.
- Reads can be distributed across replicas to increase throughput, while writes go to the master.
- If the leader fails, a follower is **promoted** (by an orchestrator or a vote). Until then writes are unavailable: the leader is a single point of failure for writes.

![Screenshot_2023-07-29_at_3.54.08_PM](/Attachments/Screenshot_2023-07-29_at_3.54.08_PM.png)

![Screenshot_2023-07-29_at_3.55.51_PM](/Attachments/Screenshot_2023-07-29_at_3.55.51_PM.png)

![Screenshot_2023-07-24_at_2.16.59_PM](/Attachments/Screenshot_2023-07-24_at_2.16.59_PM.png)

When requests overwhelm one server, send **writes to the master** and **reads to the replicas**:

![Screenshot_2023-07-24_at_2.23.47_PM](/Attachments/Screenshot_2023-07-24_at_2.23.47_PM.png)

**How the change stream is shipped**: statement-based (fragile: `now()`, random), **physical WAL shipping** (Postgres streaming replication: byte-identical copy, same major version), or **logical/row-based** (MySQL row binlog, Postgres logical replication: can replicate a subset and across versions, which enables zero-downtime upgrades).

## Synchronous vs asynchronous replication

| Strategy | Commit returns when | Consistency | Risk |
|---|---|---|---|
| **Synchronous** | leader **and** the sync replica(s) have the change | Strong (no loss on leader failure) | Higher write latency; a slow/dead sync replica blocks writes |
| **Asynchronous** | leader has it; replicas catch up in the background | **Eventual** | Leader failure loses unreplicated writes; reads from replicas are stale |
| **Semi-synchronous** | leader + at least one replica has **received/logged** it | In between | Common compromise (MySQL semi-sync, Postgres `ANY 1 (...)`) |

- In synchronous mode the change isn't marked complete until copied to the other node(s). In Postgres you choose how many and which standbys to wait for with priorities: `synchronous_standby_names = 'FIRST 1 (standby1, standby2)'` or quorum `'ANY 2 (s1, s2, s3)'`. (Cassandra's equivalent is consistency level `QUORUM`.)
- Asynchronous mode marks the write complete immediately and ships it in the background; if the primary fails, recent changes can be lost.

## Setting up new followers

A new follower takes a **consistent snapshot** of the leader (e.g. `pg_basebackup`) while the leader keeps accepting writes. It then replays the leader's log from the snapshot's position (in Postgres, the WAL from that LSN) to catch up, and continues streaming.

## Handling node outages

- **Follower failure**: on restart it knows the last log position it applied and requests everything after it (catch-up recovery). The leader must retain that log (Postgres replication slots do this; beware unbounded WAL growth if a replica disappears).
- **Leader failure (failover)**: detect (timeouts), choose the most up-to-date follower, reconfigure clients. Risks: lost async writes, **split brain** (two leaders), timeouts too short (unnecessary failovers) or too long (long outages).

## Replication lag and consistency

With async followers, a read can hit a replica that is behind. Anomalies and fixes:

| Problem | Example | Fix |
|---|---|---|
| **Read-your-writes** | User updates profile, refreshes, sees the old one | Read own recent data from the leader for N seconds, or wait for the replica to reach the write's LSN |
| **Monotonic reads** | Refresh shows a comment, next refresh (other replica) it's gone | Pin a user to one replica |
| **Consistent prefix** | See an answer before the question | Keep causally related writes in one partition; causal consistency |

```sql
-- Postgres: how far behind are replicas?
SELECT application_name, state, sync_state, replay_lag FROM pg_stat_replication;
```

## Multi-leader replication (master-master)

Several nodes accept writes and replicate to each other, scaling writes and allowing each region/data center (or offline device) to keep working when disconnected. Each master may own a different section of the data while holding the whole database.

The catch: **you must resolve write conflicts** (the same row edited in two places): last-write-wins, merge functions, CRDTs, or avoid conflicts by routing each record's writes to a "home" leader. (The original note mentioned 2PC/3PC for this; those coordinate atomic commits across nodes, whereas multi-leader systems usually resolve conflicts asynchronously.)

## Leaderless (peer-to-peer) replication

All nodes are equal and any node can accept writes, which must reach the others. Clients (or a coordinator node) write to W replicas and read from R replicas; with **R + W > N** reads see at least one up-to-date copy. Stale copies are fixed by **read repair**, **hinted handoff** and **anti-entropy** (Merkle trees). Examples: Cassandra, ScyllaDB, Riak, Dynamo-style systems. See [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem).

## Hands-on: Postgres streaming replication with Docker (course lab)

The course's manual steps, with file names corrected (`postgresql.conf`, `host.docker.internal`). The password here is a local demo value; never reuse it.

```bash
# primary
docker run --name pmaster -p 5432:5432 -e POSTGRES_PASSWORD=postgres \
  -v /Volumes/AloushSSD/_Fundementals\ of\ DB/replication/pmaster_data:/var/lib/postgresql/data -d postgres
# replica
docker run --name preplica -p 5433:5432 -e POSTGRES_PASSWORD=postgres \
  -v /Volumes/AloushSSD/_Fundementals\ of\ DB/replication/preplica_data:/var/lib/postgresql/data -d postgres

# stop both containers, then seed the replica with a copy of the primary's data dir
mv preplica_data preplica_data_backup
cp -R pmaster_data preplica_data
```

On the **primary**, allow replication connections in `pg_hba.conf`:

```text
host replication postgres all md5      # prefer scram-sha-256 on modern versions
```

On the **replica**, in `postgresql.conf`:

```ini
primary_conninfo = 'application_name=standby1 host=host.docker.internal port=5432 user=postgres password=postgres'
```

and create an empty `standby.signal` file in the replica's data directory (`touch standby.signal`).

On the **primary**, make `standby1` synchronous in `postgresql.conf`:

```ini
synchronous_standby_names = 'first 1 (standby1)'
```

Start both and check `SELECT * FROM pg_stat_replication;` on the primary.

<Tip>
**The modern shortcut**

Instead of copying the data directory with the server stopped, run `pg_basebackup -h primary -U replicator -D /var/lib/postgresql/data -R -X stream -C -S replica1` on the empty replica. `-R` writes `primary_conninfo` and `standby.signal` for you; `-C -S` creates a replication slot. Use a dedicated role with the `REPLICATION` attribute rather than the superuser.
</Tip>

## Distributed databases (textbook view)

A **distributed database** is several interconnected databases at different sites that coordinate over a network and appear to the user as one database.

- **Transparency**: users shouldn't need to know where data lives (location transparency), that it's split (fragmentation transparency) or copied (replication transparency).
- **Data fragmentation**: **horizontal** (rows split across sites, i.e. sharding), **vertical** (columns split), or **hybrid**. See [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning).
- **Data replication**: copies at several sites for reliability, availability and fault tolerance.
- **Distributed query processing**: the optimizer also minimises data shipped between sites (e.g. semi-joins).
- **Global serializability**: executing all transactions across all sites must be equivalent to some serial order, typically via distributed 2PL or timestamp ordering plus an atomic commit protocol.

## Two-phase commit (2PC)

An **atomic commitment protocol** that ensures a transaction spanning several nodes either commits on **all** of them or aborts on all.

Roles: one **coordinator** (transaction manager) and several **participants** (resource managers).

1. **Prepare (voting) phase**: the coordinator sends *prepare* to every participant. Each executes the transaction up to the point of commit, makes it durable (writes it to its log) and **holds locks** on what it touched, then votes **Yes** ("I can commit") or **No** ("I can't, I abort").
2. **Commit (decision) phase**: if **all** voted Yes, the coordinator logs the decision and sends *commit*; if **any** voted No (or timed out), it sends *abort*. Participants apply the decision and release locks.

{/* diagram source: /diagrams/diagram-122.mmd */}
![diagram](/diagrams/diagram-122.svg)

**Why we need it (the Careem burger example)**: ordering food needs two independent steps: the restaurant accepts the order, and a driver is assigned. Without atomic commitment, one can succeed and the other fail: food prepared with no driver, or a driver sent to a restaurant that never accepted.

![Screenshot_2023-07-24_at_2.34.47_PM](/Attachments/Screenshot_2023-07-24_at_2.34.47_PM.png)

2PC splits the flow: the **prepare phase reserves** (hold the food slot, hold a driver), and the **commit phase books** both.

![Screenshot_2023-07-24_at_2.36.35_PM](/Attachments/Screenshot_2023-07-24_at_2.36.35_PM.png)

If both reservations fail, the transaction fails; if one succeeds and the other fails, the successful reservation is cancelled and the whole transaction aborts.

![Screenshot_2023-07-24_at_2.44.10_PM](/Attachments/Screenshot_2023-07-24_at_2.44.10_PM.png)

![Screenshot_2023-07-24_at_2.44.21_PM](/Attachments/Screenshot_2023-07-24_at_2.44.21_PM.png)

**Advantages**: atomic commit across nodes; combined with locking, isolation of the distributed transaction.

**Disadvantages**:

- **Blocking**: if the coordinator crashes after participants voted Yes but before sending the decision, participants are **in doubt**: they can't commit or abort on their own and keep holding locks until the coordinator recovers.
- **Slow**: multiple round trips and forced log writes (fsyncs) on every node.
- Locks held across the network amplify contention and deadlocks.

Postgres supports being a 2PC participant with `PREPARE TRANSACTION 'id'` / `COMMIT PREPARED 'id'` (`max_prepared_transactions` must be > 0). Forgotten prepared transactions hold locks and block VACUUM ([MVCC & Vacuum](/Software%20Engineering/Databases/Core%20Concepts/MVCC%20%26%20Vacuum)). XA is the standard interface used by Java transaction managers.

## Three-phase commit (3PC)

A more fault-tolerant variant that adds a phase so participants can decide without the coordinator. It is **non-blocking** under its assumptions: if the coordinator fails, participants can decide among themselves whether to commit or abort.

1. **CanCommit**: the coordinator asks "can you commit?"; participants answer Yes or No (without doing the work yet).
2. **PreCommit**: if all said Yes, the coordinator sends *pre-commit*; participants execute, write undo/redo logs to **disk**, and acknowledge. If anyone said No, it sends *abort*.
3. **DoCommit**: after all acknowledgements, the coordinator sends *do-commit*; participants commit, release resources, clean the undo log, and acknowledge.

The catch: 3PC assumes a **synchronous network with bounded delays** and fail-stop nodes. With real networks (partitions, arbitrary delays) it can still produce inconsistent outcomes, so it's rarely used. Production systems get non-blocking atomic commit by running the coordinator's decision through **consensus** (Paxos/Raft), as Spanner and CockroachDB do.

## Beyond 2PC: how real systems do it

- **Sagas**: a sequence of local transactions, each with a **compensating action** (cancel the restaurant order if no driver is found). No global locks; intermediate states are visible, so design for it. Orchestrated (a coordinator service or workflow engine such as Temporal) or choreographed (events).
- **Transactional outbox**: write the business row and an "event to publish" row in the **same local transaction**; a relay publishes the event (CDC with Debezium, or polling). Avoids the dual-write problem.
- **Idempotency keys** so retries don't duplicate effects.

### Ideas from DDIA ch. 12 (from the original *Overview 1*)

- **Data integration**: data that started out siloed in different systems gets combined, often through **derived data**: search indexes, caches and analytics stores are transformations of a source-of-truth log.
- **Unbundling databases**: instead of one monolithic database doing storage, indexing, replication and caching, compose specialised systems around an event log (Kafka + stream processors), like microservices for data.
- **Aiming for correctness**: use end-to-end arguments (idempotent operations with request IDs from the client all the way down) rather than relying on each component's transactions; audit and verify data continuously.
- The chapter also discussed hardware trends (the end of Moore's law, "doing more with less" by reducing data movement).

## Pitfalls

<Warning>
**Common pitfalls**

- Reading from async replicas right after writing (stale reads) without read-your-writes handling.
- Replication slots for dead replicas making the primary's disk fill with WAL.
- Assuming synchronous replication means zero downtime: if the only sync standby dies, writes block.
- Using 2PC across microservices owned by different teams: availability of every service multiplies, locks span the network.
- Split brain after a partition in DIY failover.
</Warning>

## Senior perspective

- For most products: single-leader, async or semi-sync replicas, automated failover (managed service or Patroni), lag monitoring, and lag-aware routing.
- Cross-region: decide whether you need multi-region writes at all; a single write region with regional read replicas is far simpler.
- Replace distributed transactions with sagas + outbox + idempotency; keep 2PC for tightly coupled resources inside one system.
- Review questions: What's the RPO on leader failure? How are clients redirected? What happens to in-flight transactions? Is every cross-service operation idempotent and compensatable?

<Info>
**🆕 2025–2026**

- **PostgreSQL 17 (2024-09-26)**: logical replication slots survive `pg_upgrade`, failover control for logical slots, and `pg_createsubscriber` converts a physical standby into a logical replica.
- **PostgreSQL 18 (2025-09-25)**: logical replication write conflicts are logged and counted in `pg_stat_subscription_stats`; `CREATE SUBSCRIPTION` streams large transactions in parallel by default; idle replication slots can be invalidated automatically (`idle_replication_slot_timeout`) to stop WAL piling up; `pg_createsubscriber --all`.
- **PostgreSQL 19 (beta, GA expected ~October 2026)**: logical replication of **sequences**, `CREATE PUBLICATION ... EXCEPT`, enabling logical replication without a restart, and a `WAIT FOR` command for read-your-writes on standbys.
</Info>

## Interview questions

<Accordion title="Synchronous vs asynchronous replication: trade-offs?">
Sync: a commit waits for replicas, so no data loss on leader failure but higher latency and writes stall if the sync replica is unavailable. Async: fast commits, but replicas lag and recent writes can be lost on failover.
</Accordion>

<Accordion title="How do you give users read-your-writes consistency with read replicas?">
Route a user's reads to the primary for a short window after they write, or track the write's LSN/timestamp and only read from replicas that have replayed past it (Postgres 19's `WAIT FOR` supports this), or pin sessions.
</Accordion>

<Accordion title="Explain two-phase commit and its main weakness.">
The coordinator asks all participants to prepare (make durable, hold locks, vote); if all vote yes it tells them to commit, otherwise abort. Weakness: it's blocking. If the coordinator fails after votes, prepared participants hold locks indefinitely until it recovers. It's also slow.
</Accordion>

<Accordion title="Why isn't 3PC used much?">
It avoids blocking only if the network has bounded delays and no partitions; under real partitions it can lead to inconsistent decisions. Consensus-based commit (Paxos/Raft) solves the problem properly.
</Accordion>

<Accordion title="How would you implement &quot;place order + charge card + assign driver&quot; across services?">
A saga: local transactions per service, each emitting events via an outbox, with compensations (refund, cancel order) if a later step fails, idempotency keys on every step, and an orchestrator or workflow engine to track state.
</Accordion>

<Accordion title="Single-leader vs multi-leader vs leaderless?">
Single-leader is simplest and consistent for writes but the leader is a write bottleneck and failover point. Multi-leader allows writes in several regions/offline but needs conflict resolution. Leaderless offers high availability with quorums and tunable consistency, at the cost of anomalies and repair mechanisms.
</Accordion>

## Related
- [Databases Index](/Software%20Engineering/Databases/Databases%20Index) · [Database Round](/Software%20Engineering/Databases/Database%20Round/Database%20Round)
- [Clustering and Scaling](/Software%20Engineering/Databases/Database%20Round/Clustering%20and%20Scaling) · [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem) · [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning) · [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Locking & Concurrency Control](/Software%20Engineering/Databases/Core%20Concepts/Locking%20%26%20Concurrency%20Control)
- [Database Backup](/Software%20Engineering/Databases/Database%20Round/Database%20Backup) · [Connection Pooling](/Software%20Engineering/Databases/Database%20Round/Connection%20Pooling) · [NoSQL Database Types](/Software%20Engineering/Databases/Core%20Concepts/NoSQL%20Database%20Types)
- Original notes: `_Archive/Overview 1 - Replication & Data Systems (original)`, `_Archive/Overview - Locking & Distributed DB (original)`

## Sources
- PostgreSQL docs, Log-Shipping Standby Servers / Synchronous Replication: https://www.postgresql.org/docs/current/warm-standby.html
- PostgreSQL docs, pg_basebackup: https://www.postgresql.org/docs/current/app-pgbasebackup.html
- PostgreSQL docs, PREPARE TRANSACTION: https://www.postgresql.org/docs/current/sql-prepare-transaction.html
- PostgreSQL 17 / 18 releases: https://www.postgresql.org/about/news/postgresql-17-released-2936/ · https://www.postgresql.org/about/news/postgresql-18-released-3142/
- PostgreSQL 19 Beta 1: https://www.postgresql.org/about/news/postgresql-19-beta-1-released-3313/
- Martin Kleppmann, *Designing Data-Intensive Applications*, ch. 5 (Replication), 9 (Consistency and Consensus), 12 (The Future of Data Systems)
- Silberschatz, Korth, Sudarshan, *Database System Concepts*, ch. 23 (Parallel and Distributed Transaction Processing)
- Skeen, *Nonblocking Commit Protocols* (1981); Gray & Lamport, *Consensus on Transaction Commit* (2006): https://www.microsoft.com/en-us/research/publication/consensus-on-transaction-commit/
- Chris Richardson, Saga pattern: https://microservices.io/patterns/data/saga.html · Transactional outbox: https://microservices.io/patterns/data/transactional-outbox.html
