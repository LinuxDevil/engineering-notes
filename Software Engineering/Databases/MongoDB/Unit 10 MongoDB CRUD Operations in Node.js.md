---
title: "Unit 10 MongoDB CRUD Operations in Node.js"
tags: [databases, databases/mongodb]
aliases: [MongoDB Node.js CRUD, MongoDB Transactions Node.js, withTransaction, MongoDB Unit 10]
updated: 2026-09-27
---
# Unit 10 MongoDB CRUD Operations in Node.js

<Note>
**TL;DR**

In Node.js you work with plain JavaScript objects; the driver **serializes them to BSON** (MongoDB's storage and wire format) and back. CRUD methods mirror the shell (`insertOne`, `find`, `updateOne`, `deleteMany`, ...) and return promises with result objects (`insertedId`, `modifiedCount`, `deletedCount`). For all-or-nothing changes across documents, use a **multi-document transaction** with `session.withTransaction()`, passing `{ session }` to every operation. Transactions are limited to 60 s by default and cost more than single-document writes.
</Note>

> Course: MongoDB University, Unit 10 · Prev: [Unit 09 MongoDB CRUD Operations Modifying Query Results](/Software%20Engineering/Databases/MongoDB/Unit%2009%20MongoDB%20CRUD%20Operations%20Modifying%20Query%20Results) · Next: [Unit 11 MongoDB Aggregation](/Software%20Engineering/Databases/MongoDB/Unit%2011%20MongoDB%20Aggregation)

## What this unit covers
How BSON documents look in Node.js; inserting, querying, updating and deleting from a Node.js app; and creating a multi-document transaction. The examples use a `bank` database with `accounts` and `transfers` collections, and assume a shared `client` from [Unit 06 Connecting to MongoDB in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2006%20Connecting%20to%20MongoDB%20in%20Node.js).

## BSON in Node.js
- **BSON is MongoDB's binary format** for storing documents and sending them over the wire. Node.js itself does not "use BSON to store data" (the original note's wording); you write JS objects and the driver (via the `bson` package) converts them.
- BSON is optimized for storage, retrieval and transmission, and has more types than JSON: `Date`, `ObjectId`, `Decimal128`, `Long`, `Binary`, etc.
- The original note said BSON is "more secure than JSON against JSON injection". That is not a real property of the format. Injection risk in MongoDB comes from putting untrusted objects into query filters (e.g. `{ $ne: null }`), whatever the encoding. Validate input types instead.

```js
import { ObjectId, Decimal128 } from 'mongodb';

const doc = {
  _id: new ObjectId(),                    // BSON ObjectId
  balance: Decimal128.fromString('100.50'),
  opened: new Date(),                     // BSON Date
};
```

## Setup used by the examples
```js
const { MongoClient, ObjectId } = require('mongodb');
const client = new MongoClient(process.env.MONGODB_URI);

const dbname = "bank";
const collection_name = "accounts";
const accountsCollection = client.db(dbname).collection(collection_name);

// Course helper; optional since driver 4.7 (auto-connect)
const connectToDatabase = async () => {
  await client.connect();
  console.log(`Connected to the ${dbname} database`);
};
```

## Insert
### insertOne
```js
const sampleAccount = {
  account_holder: "Linus Torvalds",
  account_id: "MDB829001337",
  account_type: "checking",
  balance: 50352434,
};

const main = async () => {
  try {
    await connectToDatabase();
    const result = await accountsCollection.insertOne(sampleAccount);
    console.log(`Inserted document: ${result.insertedId}`);
  } catch (err) {
    console.error(`Error inserting document: ${err}`);
  } finally {
    await client.close();
  }
};

main();
```

### insertMany
```js
const sampleAccounts = [
  { account_id: "MDB011235813", account_holder: "Ada Lovelace", account_type: "checking", balance: 60218 },
  { account_id: "MDB829000001", account_holder: "Muhammad ibn Musa al-Khwarizmi", account_type: "savings", balance: 267914296 },
];

const main = async () => {
  try {
    await connectToDatabase();
    const result = await accountsCollection.insertMany(sampleAccounts);
    console.log(`Inserted ${result.insertedCount} documents`);
    console.log(result); // { acknowledged, insertedCount, insertedIds: { '0': ObjectId, '1': ObjectId } }
  } catch (err) {
    console.error(`Error inserting documents: ${err}`);
  } finally {
    await client.close();
  }
};
```

## Query
### find
```js
// Document used as a filter for find()
const documentsToFind = { balance: { $gt: 4700 } };

const main = async () => {
  try {
    await connectToDatabase();
    const cursor = accountsCollection.find(documentsToFind);        // no await: returns a cursor
    const docCount = accountsCollection.countDocuments(documentsToFind);
    await cursor.forEach((doc) => console.log(doc));
    console.log(`Found ${await docCount} documents`);
  } catch (err) {
    console.error(`Error finding documents: ${err}`);
  } finally {
    await client.close();
  }
};
```

Other ways to consume a cursor:

```js
const docs = await accountsCollection.find(filter).sort({ balance: -1 }).limit(10).toArray();
for await (const doc of accountsCollection.find(filter)) { /* stream one by one */ }
```

### findOne
```js
const documentToFind = { _id: new ObjectId("62a3638521a9ad028fdf77a3") };

const result = await accountsCollection.findOne(documentToFind);  // document or null
console.log(result);
```

## Update
### updateOne
```js
const documentToUpdate = { _id: new ObjectId("62d6e04ecab6d8e130497482") };
const update = { $inc: { balance: 100 } };

const result = await accountsCollection.updateOne(documentToUpdate, update);
result.modifiedCount === 1
  ? console.log("Updated one document")
  : console.log("No documents updated");
```

### updateMany
(The original snippet declared `bank` but used `accountsCollection`; fixed to use one name.)

```js
const documentsToUpdate = { account_type: "checking" };
const update = { $push: { transfers_complete: "TR413308000" } };

const result = await accountsCollection.updateMany(documentsToUpdate, update);
result.modifiedCount > 0
  ? console.log(`Updated ${result.modifiedCount} documents`)
  : console.log("No documents updated");
```

## Delete
### deleteOne
```js
const documentToDelete = { _id: new ObjectId("62d6e04ecab6d8e13049749c") };

const result = await accountsCollection.deleteOne(documentToDelete);
result.deletedCount === 1
  ? console.log("Deleted one document")
  : console.log("No documents deleted");
```

### deleteMany
```js
const documentsToDelete = { balance: { $lt: 500 } };

const result = await accountsCollection.deleteMany(documentsToDelete);
result.deletedCount > 0
  ? console.log(`Deleted ${result.deletedCount} documents`)
  : console.log("No documents deleted");
```

## Multi-document transactions
A multi-document transaction is a group of operations that all succeed or all fail together: **atomicity**. It also gives snapshot isolation to its reads and makes all its writes visible at commit.

- A transaction has a **60-second** runtime limit by default (server parameter `transactionLifetimeLimitSeconds`); longer ones are aborted.
- Requires a replica set or sharded cluster (every Atlas cluster qualifies).

Steps:
1. Start a **client session**.
2. Optionally define transaction options (read concern, write concern, read preference).
3. Run the sequence of operations inside the transaction, passing the session to each.
4. Commit (or abort) and release the session's resources.

{/* diagram source: /diagrams/diagram-130.mmd */}
![diagram](/diagrams/diagram-130.svg)

### Walkthrough (rebuilt from the course steps)
**1. Create the variables used in the transaction**

```js
// Collections
const accounts = client.db("bank").collection("accounts");
const transfers = client.db("bank").collection("transfers");

// Account information
const account_id_sender = "MDB574189300";
const account_id_receiver = "MDB343652528";
const transaction_amount = 100;
```

**2. Start a new session**

```js
const session = client.startSession();
```

**3. Begin a transaction with `withTransaction()`**

```js
const transactionResults = await session.withTransaction(async () => {
  // Operations will go here
});
```

**4. Decrement the sender's balance**

```js
const senderUpdate = await accounts.updateOne(
  { account_id: account_id_sender },
  { $inc: { balance: -transaction_amount } },
  { session }
);
```

**5. Increment the receiver's balance**

```js
const receiverUpdate = await accounts.updateOne(
  { account_id: account_id_receiver },
  { $inc: { balance: transaction_amount } },
  { session }
);
```

**6. Create a transfer document and insert it into `transfers`**

```js
const transfer = {
  transfer_id: "TR21872187",
  amount: 100,
  from_account: account_id_sender,
  to_account: account_id_receiver,
};

const insertTransferResults = await transfers.insertOne(transfer, { session });
```

**7. Add the `transfer_id` to the sender's `transfers_complete` array**

```js
const updateSenderTransferResults = await accounts.updateOne(
  { account_id: account_id_sender },
  { $push: { transfers_complete: transfer.transfer_id } },
  { session }
);
```

**8. Add the `transfer_id` to the receiver's `transfers_complete` array**

```js
const updateReceiverTransferResults = await accounts.updateOne(
  { account_id: account_id_receiver },
  { $push: { transfers_complete: transfer.transfer_id } },
  { session }
);
```

**9. Log success or failure**

```js
if (transactionResults) {
  console.log("Transaction completed successfully.");
} else {
  console.log("Transaction failed.");
}
```

**10. Catch errors and close the session**

```js
try {
  // steps 3–9
} catch (err) {
  console.error(`Transaction aborted: ${err}`);
  process.exit(1);
} finally {
  await session.endSession();
  await client.close();
}
```

### Full course solution
```js
const { MongoClient } = require("mongodb");
require("dotenv").config();

const uri = process.env.MONGODB_URI;
const client = new MongoClient(uri);

// Collections
const accounts = client.db("bank").collection("accounts");
const transfers = client.db("bank").collection("transfers");

// Account information
let account_id_sender = "MDB574189300";
let account_id_receiver = "MDB343652528";
let transaction_amount = 100;

// Start the client session
const session = client.startSession();

// Use withTransaction to start a transaction, execute the callback, and commit the transaction
// The callback for withTransaction must be async/await
// Note: Each individual operation must be awaited and have the session passed in as an argument
const main = async () => {
  try {
    const transactionResults = await session.withTransaction(async () => {
      // Step 1: Update the sender balance and pass along the `session` object
      const updateSenderResult = await accounts.updateOne(
        { account_id: account_id_sender },
        { $inc: { balance: -1 * transaction_amount } },
        { session }
      );
      console.log('Updated the sender', updateSenderResult.matchedCount);

      // Step 2: Update the account receiver balance
      const updateReceiverResults = await accounts.updateOne(
        { account_id: account_id_receiver },
        { $inc: { balance: transaction_amount } },
        { session }
      );
      console.log(
        `${updateReceiverResults.matchedCount} document(s) matched the filter, updated ${updateReceiverResults.modifiedCount} document(s) for the receiver account.`
      );

      // Step 3: Insert the transfer document
      const transfer = {
        transfer_id: "TR21872187",
        amount: 100,
        from_account: account_id_sender,
        to_account: account_id_receiver,
      };

      const insertTransferResults = await transfers.insertOne(transfer, {
        session,
      });
      console.log(
        `Successfully inserted ${insertTransferResults.insertedId} into the transfers collection`
      );

      // Step 4: Update the transfers_complete field for the sender account
      const updateSenderTransferResults = await accounts.updateOne(
        { account_id: account_id_sender },
        { $push: { transfers_complete: transfer.transfer_id } },
        { session }
      );
      console.log(
        `${updateSenderTransferResults.matchedCount} document(s) matched in the transfers collection, updated ${updateSenderTransferResults.modifiedCount} document(s) for the sender account.`
      );
      // Step 5: Update the transfers_complete field for the receiver account
      const updateReceiverTransferResults = await accounts.updateOne(
        { account_id: account_id_receiver },
        { $push: { transfers_complete: transfer.transfer_id } },
        { session }
      );
      console.log(
        `${updateReceiverTransferResults.matchedCount} document(s) matched in the transfers collection, updated ${updateReceiverTransferResults.modifiedCount} document(s) for the receiver account.`
      );
    });

    console.log("Committing transaction ...");
    // If the callback for withTransaction returns successfully without throwing an error, the transaction will be committed
    if (transactionResults) {
      console.log("The reservation was successfully created.");
    } else {
      console.log("The transaction was intentionally aborted.");
    }
  } catch (err) {
    console.error(`Transaction aborted: ${err}`);
    process.exit(1);
  } finally {
    await session.endSession();
    await client.close();
  }
};

main();
```

Notes on the solution:
- `withTransaction` commits if the callback resolves, aborts if it throws, and **retries** the callback on `TransientTransactionError` and the commit on `UnknownTransactionCommitResult`. So the callback must be safe to run more than once (no external side effects like sending emails inside it).
- To abort on a business rule (e.g. insufficient funds), throw inside the callback, or call `await session.abortTransaction()`.
- A production version should check the sender balance in the filter: `{ account_id: sender, balance: { $gte: amount } }` and throw if `modifiedCount === 0`.

Shorter modern form (driver 6.x+), which manages the session for you:

```js
await client.withSession(async (session) =>
  session.withTransaction(async () => {
    const r = await accounts.updateOne(
      { account_id: sender, balance: { $gte: amount } },
      { $inc: { balance: -amount } },
      { session }
    );
    if (r.modifiedCount !== 1) throw new Error('Insufficient funds');
    await accounts.updateOne({ account_id: receiver }, { $inc: { balance: amount } }, { session });
  }, { readConcern: { level: 'snapshot' }, writeConcern: { w: 'majority' } })
);
```

<Info>
**🆕 2025–2026**

**Node.js driver 7.0.0 (2025-11-06)** requires Node.js 20.19.0+, allocates sessions for cursors lazily, closes idle pooled connections even with `minPoolSize: 0`, and rejects non-boolean values for boolean options. Cursors also no longer send a default `batchSize` of 1000 on `getMore`. Check these when upgrading code like the examples above. ([release notes](https://www.mongodb.com/community/forums/t/mongodb-nodejs-driver-7-0-0-released/331061))
</Info>

## Senior perspective
- **Prefer single-document atomicity**: model so most operations touch one document; transactions are the exception, not the default. They add latency (extra round trips, majority commit), hold locks/snapshots, and can abort with write conflicts under contention.
- **Limits**: 60 s default lifetime (`transactionLifetimeLimitSeconds`), keep them short and small (MongoDB recommends modifying no more than ~1,000 documents per transaction). Large transactions pressure the WiredTiger cache.
- **Write concern / read concern / read preference**:
  - Write concern `w: "majority"` (default since 5.0) means acknowledged writes survive failover.
  - Read concern `"majority"` returns only majority-committed data; `"snapshot"` inside transactions; `"linearizable"` for strongest single-doc reads.
  - Read preference `primary` (default) vs `secondary*` trades freshness for offloading reads.
- **Retryable writes** handle transient errors for single operations; transactions have their own retry labels handled by `withTransaction`.
- **Sharded transactions** span shards with a two-phase commit coordinated by the router: more expensive. See [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC) and [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID).
- **Idempotency**: include a unique `transfer_id` with a unique index so a retried request can't create a second transfer.

## Interview questions
<Accordion title="What is BSON and how does the Node.js driver use it?">
BSON is MongoDB's binary document format for storage and the wire protocol. The driver serializes JS objects to BSON when sending and deserializes responses into JS objects, mapping types like `Date`, `ObjectId`, `Decimal128` and `Long`.
</Accordion>

<Accordion title="How do you run a multi-document transaction in Node.js?">
Start a session, call `session.withTransaction(async () => { ... })`, pass `{ session }` to every operation inside, then `endSession()`. `withTransaction` commits on success, aborts on error, and retries transient errors.
</Accordion>

<Accordion title="What is the default time limit on a transaction and what happens when it's exceeded?">
60 seconds (`transactionLifetimeLimitSeconds`). The server aborts the transaction, and the commit fails.
</Accordion>

<Accordion title="Why should the withTransaction callback be idempotent?">
The driver may re-run the whole callback on a `TransientTransactionError`. Side effects outside the database (emails, HTTP calls) would repeat.
</Accordion>

<Accordion title="When do you need a transaction in MongoDB, and when can you avoid it?">
Needed when an invariant spans multiple documents or collections (money transfer between two accounts, order + inventory). Avoid it by embedding data that changes together in one document, since single-document writes are atomic.
</Accordion>

<Accordion title="Explain write concern, read concern and read preference.">
Write concern: how many nodes must acknowledge a write (`w: 1`, `w: "majority"`, `j: true`). Read concern: what consistency level the data read has (`local`, `majority`, `snapshot`, `linearizable`). Read preference: which members serve reads (`primary`, `primaryPreferred`, `secondary`, `secondaryPreferred`, `nearest`).
</Accordion>

## Related
- [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Transactions & ACID](/Software%20Engineering/Databases/Database%20Round/Transactions%20%26%20ACID) · [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC) · [Connection Pooling](/Software%20Engineering/Databases/Database%20Round/Connection%20Pooling) · [CAP Theorem](/Software%20Engineering/Databases/Database%20Round/CAP%20Theorem)

## Sources
- [Node.js driver: CRUD operations](https://www.mongodb.com/docs/drivers/node/current/crud/)
- [Node.js driver: Transactions](https://www.mongodb.com/docs/drivers/node/current/crud/transactions/)
- [MongoDB Manual: Transactions and production considerations](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/)
- [Read Concern](https://www.mongodb.com/docs/manual/reference/read-concern/) · [Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/) · [Read Preference](https://www.mongodb.com/docs/manual/core/read-preference/)
- [BSON specification](https://bsonspec.org/)
- [Node.js driver 7.0.0 release](https://www.mongodb.com/community/forums/t/mongodb-nodejs-driver-7-0-0-released/331061)
- [MongoDB University: CRUD Operations in Node.js](https://learn.mongodb.com/)
