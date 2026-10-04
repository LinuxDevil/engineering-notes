---
title: "Consistency Models & Consensus"
tags: [system-design, system-design/distributed-systems]
aliases: [Consistency Models, Consensus, Raft, Paxos, Quorum, Vector Clocks, CRDT, Leader Election, Eventual Consistency, Linearizability]
updated: 2026-09-27
---
# Consistency Models & Consensus

<Note>
**TL;DR**

A **consistency model** is a contract about what reads can return when data is replicated. From strongest to weakest: **linearizable** (behaves like one copy), **sequential**, **causal** (cause is seen before effect), **read-your-writes / monotonic reads** (session guarantees), **eventual** (replicas converge if writes stop). Stronger models cost latency and availability during partitions ([CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem), PACELC). **Consensus** (Paxos, Raft) lets a group of nodes agree on a sequence of values despite failures; it powers leader election, replicated logs and coordination stores (etcd, ZooKeeper, KRaft). **Quorums**, **vector clocks** and **CRDTs** are the tools for leaderless and multi-leader systems.
</Note>

## Why this matters
As soon as data lives on more than one node (replicas, caches, regions), two clients can see different values. You must decide which anomalies are acceptable per feature: a "like" count can be eventually consistent; a bank balance or a username uniqueness check cannot.

## The spectrum
| Model | Guarantee | Example anomaly it prevents | Cost |
|---|---|---|---|
| **Linearizable** (strong) | Every op appears to take effect atomically at one instant between call and return; everyone sees the same latest value | Reading an old value after someone else's write completed | Coordination on every op; unavailable in a partition (CP) |
| **Sequential** | All nodes see ops in the same order, consistent with each client's program order | Different nodes disagreeing on order | Still needs global ordering |
| **Causal** | Ops that are causally related are seen in order by everyone; concurrent ops may be seen in any order | Seeing a reply before the comment it answers | Track dependencies (vector clocks); available under partition |
| **Read-your-writes** | A client sees its own writes | Profile edit "disappears" after save | Sticky routing or version tokens |
| **Monotonic reads** | A client never sees time go backwards | Comment appears, then vanishes on refresh | Sticky replica or version tokens |
| **Eventual** | Replicas converge if writes stop | Nothing (only convergence) | Cheapest, highest availability |

Note: **ACID "C"** (invariants) and **isolation levels** (serializable, snapshot) are about transactions on one database; see [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) and [Isolation Levels & Read Phenomena](/Software%20Engineering/Databases/Core%20Concepts/Isolation%20Levels%20%26%20Read%20Phenomena). *Strict serializability* = serializable + linearizable.

<Info>
**Read-your-writes with replicas**

User updates their bio (goes to primary), page reloads, read hits a lagging replica → old bio. Fixes: read from primary for N seconds after a write; route the user's reads to the primary; or return the write's LSN/version and have the replica wait until it has caught up to it.
</Info>

## PACELC
CAP only talks about partitions. PACELC: **if Partition**, choose **A**vailability or **C**onsistency; **Else**, choose **L**atency or **C**onsistency. DynamoDB/Cassandra default to PA/EL; Spanner and etcd are PC/EC.

## Quorums (leaderless replication)
With **N** replicas, write to **W** and read from **R**. If **W + R > N**, every read quorum overlaps every write quorum, so a read sees at least one copy of the latest write.

| N | W | R | Property |
|---|---|---|---|
| 3 | 2 | 2 | Balanced, tolerates 1 node down for both |
| 3 | 3 | 1 | Fast reads, writes fail if any node down |
| 3 | 1 | 1 | Fast, only eventual (W+R ≤ N) |

Extras: **read repair** (fix stale replicas on read), **hinted handoff** and **sloppy quorums** (write to a stand-in during failure; weakens the guarantee), **anti-entropy** with Merkle trees. Quorums alone are not linearizable (concurrent writes, clock-based last-write-wins).

## Consensus
Goal: nodes agree on a value (or an ordered log of values) such that agreement is final, and progress happens while a **majority** is alive. With **2f+1** nodes you tolerate **f** failures (3 → 1, 5 → 2).

### Raft (the understandable one)
{/* diagram source: /diagrams/diagram-400.mmd */}
![diagram](/diagrams/diagram-400.svg)
1. **Terms**: logical epochs; each has at most one leader.
2. **Leader election**: a follower that misses heartbeats (randomized timeout, e.g. 150–300 ms) becomes a candidate, increments the term, requests votes. A node votes once per term and only for candidates whose log is at least as up to date.
3. **Log replication**: clients send commands to the leader; it appends and replicates via `AppendEntries`; once a **majority** stored the entry it's **committed** and applied to the state machine.
4. **Safety**: a committed entry is present in every future leader's log.

Used by etcd (Kubernetes), Consul, CockroachDB, TiKV, RabbitMQ quorum queues, Kafka KRaft, MongoDB's replication protocol (Raft-like).

### Paxos (intuition)
Proposers pick a proposal number, ask acceptors to **promise** not to accept lower numbers (phase 1), then ask them to **accept** a value (phase 2); a value chosen by a majority is decided. Multi-Paxos elects a stable leader to skip phase 1, which ends up very close to Raft. Used in Google Chubby and Spanner.

### What consensus gives you
- **Leader election** without split brain (a leader must hold a majority; stale leaders are fenced by term numbers).
- **Linearizable key-value store** (etcd, ZooKeeper) for config, service discovery, locks.
- **Replicated state machines**: same log + deterministic apply = same state.

<Warning>
**Distributed locks need fencing tokens**

A lock holder can pause (GC, VM stall) past its lease and keep writing after someone else got the lock. Issue a monotonically increasing **fencing token** with each lock grant and have the storage reject writes with older tokens. Redis-based locks (Redlock) are debated for this reason; use etcd/ZooKeeper or DB row locks when correctness matters.
</Warning>

## Time and ordering
- Physical clocks drift; NTP gives ms-level skew; **last-write-wins by timestamp silently drops writes**.
- **Lamport clocks**: a counter giving a total order consistent with causality (if A → B then L(A) < L(B)), but can't detect concurrency.
- **Vector clocks**: one counter per node; compare element-wise to tell "happened-before" vs "concurrent".

```text
Node A writes x:  VC = {A:1}
Node B (saw A)   : VC = {A:1, B:1}   → after A's write (descendant)
Node C (didn't)  : VC = {A:0, C:1}   → concurrent with both → conflict, keep siblings or merge
```
- **Hybrid logical clocks (HLC)**: physical time + logical counter (CockroachDB). **TrueTime** (Spanner): bounded clock uncertainty, wait out the uncertainty for external consistency.

## CRDTs
**Conflict-free Replicated Data Types** merge concurrent updates deterministically, without coordination; merge is commutative, associative and idempotent, so all replicas converge.
| CRDT | Idea |
|---|---|
| G-Counter | Per-node counters; value = sum; merge = element-wise max |
| PN-Counter | Two G-Counters (increments, decrements) |
| LWW-Register | Keep value with highest timestamp |
| OR-Set | Adds tagged with unique IDs; remove only observed tags (add wins) |
| Sequence CRDTs (RGA, Yjs, Automerge) | Collaborative text editing, local-first apps |

```python
class GCounter:
    def __init__(self, node): self.node, self.c = node, {}
    def inc(self, n=1): self.c[self.node] = self.c.get(self.node, 0) + n
    def value(self): return sum(self.c.values())
    def merge(self, other):
        for k, v in other.c.items(): self.c[k] = max(self.c.get(k, 0), v)
```
Used in Redis Enterprise active-active, Riak, Figma/Linear-style multiplayer, Yjs/Automerge.

## Senior perspective
- Choose consistency **per operation**, not per system: strong for money, inventory, uniqueness; causal/session for social features; eventual for counters and analytics.
- Put consensus in a small, well-understood component (etcd, a managed DB) instead of implementing it. Consensus clusters should be 3 or 5 nodes, spread across failure domains.
- Latency: a linearizable write across regions costs at least one cross-region round trip (~70–150 ms).
- Test with fault injection (Jepsen-style): partitions, clock skew, paused processes.
- Ask in review: "What happens during a network partition between AZs? Which side keeps accepting writes? How are conflicts resolved?"

<Info>
**🆕 2025–2026**

Kafka 4.0 (2025-03-18) removed ZooKeeper completely; cluster metadata now lives in a Raft-based quorum (**KRaft**) inside Kafka itself, a good real-world example of replacing an external coordination service with an embedded consensus log. ([Apache Kafka 4.0 announcement](https://kafka.apache.org/blog/2025/03/18/apache-kafka-4.0.0-release-announcement/))
</Info>

## Interview questions
<Accordion title="Linearizability vs serializability?">
Linearizability is a recency guarantee on single objects: once a write completes, all later reads see it. Serializability is a transaction isolation guarantee: concurrent transactions behave as if run in some serial order (not necessarily real-time order). Strict serializability is both.
</Accordion>

<Accordion title="Why does W + R > N give consistent reads?">
Any read set of R nodes and write set of W nodes must share at least one node, so the read sees the latest acknowledged write (then picks the newest version). It's still not full linearizability under concurrent writes and sloppy quorums.
</Accordion>

<Accordion title="How does Raft elect a leader?">
Followers expecting heartbeats time out after a randomized interval, become candidates, increment the term, vote for themselves and request votes. Nodes grant one vote per term to candidates with an up-to-date log. A majority makes a leader; randomized timeouts make split votes rare.
</Accordion>

<Accordion title="Why do consensus clusters have an odd number of nodes?">
Tolerance depends on a majority: 3 nodes tolerate 1 failure, 4 nodes still only tolerate 1 (majority is 3). An even node adds cost without adding fault tolerance.
</Accordion>

<Accordion title="What problem do vector clocks solve?">
Detecting whether two versions are causally ordered or concurrent. With physical timestamps you'd silently drop one concurrent write; vector clocks let you keep both (siblings) and merge.
</Accordion>

<Accordion title="What is a CRDT and when would you use it?">
A data type whose replicas can be updated independently and merged deterministically to the same state. Use for multi-region active-active counters/sets, offline-first and collaborative editing, where coordination is too slow or impossible.
</Accordion>

<Accordion title="How do you give read-your-writes on top of async replicas?">
Route the user's reads to the primary for a short window after a write, or pass the write's log position/version and have reads wait for or select a replica that has caught up to it.
</Accordion>

## Related
- [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index)
- [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem) · [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC) · [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Isolation Levels & Read Phenomena](/Software%20Engineering/Databases/Core%20Concepts/Isolation%20Levels%20%26%20Read%20Phenomena)
- [Distributed Systems Patterns](/Software%20Engineering/System%20Design/Distributed%20Systems%20Patterns) · [Consistent Hashing](/Software%20Engineering/System%20Design/Consistent%20Hashing) · [Message Queues & Event Streaming](/Software%20Engineering/System%20Design/Message%20Queues%20%26%20Event%20Streaming) · [Case Study - Distributed Cache](/Software%20Engineering/System%20Design/Case%20Studies/Case%20Study%20-%20Distributed%20Cache)

## Sources
- Martin Kleppmann, *Designing Data-Intensive Applications*, ch. 5, 8, 9
- [Ongaro & Ousterhout: In Search of an Understandable Consensus Algorithm (Raft)](https://raft.github.io/raft.pdf) and [raft.github.io](https://raft.github.io/)
- [Lamport: Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [Jepsen: Consistency models](https://jepsen.io/consistency)
- [Shapiro et al.: Conflict-free Replicated Data Types](https://hal.inria.fr/inria-00609399/document)
- [Kleppmann: How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [Abadi: Consistency Tradeoffs in Modern Distributed Database System Design (PACELC)](https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf)
