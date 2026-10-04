---
title: "Overview - Locking & Distributed DB (original)"
---
**Locks**

A lock is a mechanism that prevents conflicts among transactions that access the same database item. Two types of locks are discussed:

1. Shared locks (S-locks), which allow the transaction that holds the lock to read (but not write) the locked item.
2. Exclusive locks (X-locks), which allow the transaction that holds the lock to both read and write the locked item.

**Two-Phase Locking (2PL)**

2PL is a protocol to govern transactions' acquisition and release of locks. It has two phases:

3. Growing phase: the transaction may obtain locks but cannot release any.
4. Shrinking phase: the transaction may release locks but cannot obtain any.

By following 2PL, a system ensures conflict serializability - a key condition for the correct execution of concurrent transactions.

**Strict Two-Phase Locking**

In strict 2PL, a transaction doesn't release any of its locks until it either commits or aborts. Strict 2PL is stricter than basic 2PL because it doesn't allow any transaction to access an item that has been updated but not yet committed by another transaction. This protects the system from cascading abort situations.

**Implementation of Locks**

This section covers the practical aspects of how locks are implemented in a database management system, including lock tables, lock managers, and the methods used to keep track of which transactions hold which locks.

**Deadlocks**

Deadlocks occur when two or more transactions indefinitely wait for each other to release locks - something that will never happen. The chapter discusses the conditions under which deadlocks occur and how they can be prevented, avoided, or detected and resolved.

**Granularity of Locks**

The chapter ends with a discussion on the granularity of locks, which refers to the size or extent of the data that is locked by a single lock operation. It could range from an entire database (coarse granularity) to individual rows or even fields within rows (fine granularity). The choice of lock granularity can significantly affect system performance and concurrency level.

**Distributed Databases**

A distributed database is a database that consists of multiple interconnected databases spread across different physical locations that communicate and coordinate their actions via a network. It appears as a single database to a user.

5. **Transparency**: One of the main goals of a distributed database is to provide data transparency, location transparency, and replication transparency.
6. **Data Fragmentation**: This is about the distribution of data across different sites. There are three types of data fragmentation: horizontal, vertical, and hybrid.
7. **Data Replication**: This is the process of storing copies of the same data on multiple nodes to ensure data reliability, availability, and fault tolerance.
8. **Distributed Query Processing**: This section discusses how a DBMS interprets and optimizes a query on a distributed database.

**Two-Phase Commit (2PC) Protocol**

2PC is a distributed algorithm that ensures all parts of a distributed transaction either all commit or all abort, thus maintaining consistency in the distributed system.

9. **Preparation Phase**: The coordinator sends a query to commit message to all participants and waits until it has received a reply from all participants.
10. **Commit Phase**: If all participants respond with an agreement message, the coordinator sends a commit message to all of them; if any participant votes 'No,' the coordinator sends an abort message to all participants.

The chapter also discusses the problems that can arise during the 2PC protocol (like failures) and how they can be addressed.

**Three-Phase Commit (3PC) Protocol**

3PC is a more fault-tolerant version of the 2PC protocol. It introduces a new phase, 'Pre-commit,' to avoid the uncertainty that can occur when a coordinator fails during the commit phase in 2PC.

**Global Serialization**

This section covers techniques to ensure that the execution of transactions in a distributed system is globally serializable, meaning that the result of executing all transactions is the same as if the transactions were executed serially, in some order, on a single system.