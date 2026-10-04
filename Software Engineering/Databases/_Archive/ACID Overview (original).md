---
title: "ACID Overview (original)"
---
"Database System Concepts" by Silberschatz, Korth, and Sudarshan is a comprehensive book and a recognized textbook for understanding database systems. Below is a high-level overview of Chapter 15, which focuses on Transactions.

**Transactions**

A transaction is a unit of program execution that accesses and possibly updates various data items. The main properties of transactions are summarized by the acronym ACID, standing for Atomicity, Consistency, Isolation, and Durability.

1. **Atomicity**: This means that a transaction is an atomic unit of processing; it is either performed in its entirety or not performed at all.
2. **Consistency**: This means that a transaction brings the database from one consistent state to another consistent state. Consistency here refers to the integrity constraints defined on a database schema.
3. **Isolation**: This means that even though transactions are executed concurrently, it appears to each transaction, T, that others are executed either before T or after T, but not during its execution. This property ensures that the execution of transactions concurrently will result in a state that is equivalent to a state obtained if these were executed serially in some order.
4. **Durability**: Once a transaction completes successfully (commits), its changes to the database state survive failures.

The chapter explains these concepts in detail and also explains how they're achieved in practical systems.

**Transaction State**

A transaction in its lifetime goes through several states: Active, Partially Committed, Committed, Failed, and Aborted. The chapter covers the lifecycle of a transaction and the various events that can cause state transitions.

**Implementation of Atomicity and Durability**

The book also discusses how Atomicity and Durability are implemented in databases. This is usually done via a recovery manager, which ensures that the database remains in a consistent state in the presence of failures. This typically involves techniques such as write-ahead logging and check-pointing.

**Concurrent Executions**

Transactions often execute concurrently, for performance reasons. The book talks about schedules (i.e., the order in which operations from concurrent transactions are executed) and introduces the concept of conflict serializability as a condition for a concurrent execution to be correct.

**Recovery and Isolation**

The book finally discusses how recovery from failures is managed and how Isolation among transactions is maintained. This involves mechanisms like undo and redo operations, locking, and timestamp-based protocols.

This is a high-level overview and for a deep understanding, it's advisable to read the chapter thoroughly and work out the examples and exercises provided in the text. It may also be beneficial to supplement your reading with practical exercises, such as implementing simple transactions in SQL and observing their behavior under different conditions (e.g., concurrent access, system crashes, etc.).

**Concurrency Control**

The purpose of concurrency control is to ensure the serializability of transactions in a multiuser database environment, i.e., to make sure that the database remains in a consistent state even when transactions are executed concurrently.

**Lock-Based Protocols**

The chapter begins with a detailed discussion on lock-based protocols. These protocols manage the interaction between transactions by forcing them to acquire and release locks on the database objects (like tables or rows) that they need to read or write.

- Shared and Exclusive Locks: Shared locks allow multiple transactions to read (but not write) the same object. An exclusive lock on an object prohibits any other transaction from reading or writing the object.
- Two-Phase Locking (2PL) Protocol: This is a commonly used protocol where each transaction must obtain all the locks it needs before it releases any lock. The protocol consists of two phases - a growing phase, in which a transaction acquires all the locks it needs and does not release any lock, and a shrinking phase, in which a transaction releases locks and does not acquire any new lock.

**Deadlocks**

Deadlocks are situations where a group of transactions are all waiting for locks held by some other transaction in the group, creating a cycle of dependencies that can never be resolved. The chapter discusses how to prevent, avoid, and detect deadlocks, and what to do when a deadlock occurs.

**Concurrency Control without Locking**

The chapter also discusses methods for achieving concurrency control without locking. These methods involve assigning a timestamp to each transaction and using these timestamps to determine the order in which conflicting operations are allowed to proceed.

- Timestamp-Based Protocols: In these protocols, the order in which transactions are allowed to proceed is determined by their timestamps. A transaction may be forced to wait or abort if it conflicts with another transaction with an earlier timestamp.
- Validation-Based Protocol: Also known as optimistic concurrency control, this approach assumes that conflicts between transactions are rare and allows transactions to proceed without acquiring locks. However, before a transaction is allowed to commit, it must validate that its execution has not conflicted with any other transactions.
- Multiversion Concurrency Control: This approach allows multiple versions of the same database object to exist at the same time, providing a different version to each transaction and thus avoiding conflicts.

**Granularity of Data**

Finally, the chapter covers the granularity of data items and how to control it. Granularity refers to the size of the data item that forms the basis of the control mechanism. Coarse-granularity locks (like table-level locks) offer more concurrency but risk more conflicts, while fine-granularity locks (like row-level locks) minimize conflicts but can be more resource-intensive to manage.