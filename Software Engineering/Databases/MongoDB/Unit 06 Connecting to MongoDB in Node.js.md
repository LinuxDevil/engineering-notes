---
title: "Unit 06 Connecting to MongoDB in Node.js"
tags: [databases, databases/mongodb]
aliases: [MongoDB Node.js Driver, MongoClient, MongoDB Unit 6]
updated: 2026-09-27
---
# Unit 06 Connecting to MongoDB in Node.js

<Note>
**TL;DR**

The official **`mongodb`** npm package is the Node.js driver. You create **one `MongoClient` per process** from a connection string kept in an environment variable, reuse it everywhere (it holds a connection pool, `maxPoolSize` default 100), and close it only on shutdown. Since driver 4.7 the client connects automatically on the first operation, so `await client.connect()` is optional. Driver **7.0** (2025-11-06) requires Node.js 20.19+.
</Note>

> Course: MongoDB University, Unit 6 · Prev: [Unit 05 Connecting to a MongoDB Database](/Software%20Engineering/Databases/MongoDB/Unit%2005%20Connecting%20to%20a%20MongoDB%20Database) · Next: [Unit 07 MongoDB CRUD Operations Insert and Find Documents](/Software%20Engineering/Databases/MongoDB/Unit%2007%20MongoDB%20CRUD%20Operations%20Insert%20and%20Find%20Documents)

## What is a driver?
A library that lets your application talk to MongoDB. It handles connection pooling, server discovery and monitoring (which node is primary), authentication, TLS, retries, and turning JavaScript objects into **BSON** and back. The Node.js driver uses the `bson` package for that serialization.

Install it:

```bash
npm install mongodb
```

## Get the connection string
In Atlas: **Connect → Drivers**, pick Node.js and copy the string. The course lab used this one:

```bash
mongodb+srv://admin:admin@cluster0.rvygg17.mongodb.net/?retryWrites=true&w=majority
```

<Warning>
**Credentials in this note**

The string above embeds a username and password (`admin`/`admin`) in plain text. Treat it as exposed: **change that user's password** (and don't use `admin` as a password). In code, never inline the URI; read it from an environment variable (`process.env.MONGODB_URI`) or a secrets manager, and keep `.env` files out of git.
</Warning>

## Course version (CommonJS)
```js
const { MongoClient } = require('mongodb');
const uri = require('./atlas_uri.js');

const client = new MongoClient(uri);
const dbName = "bank";

const connectToDB = async () => {
    try {
        await client.connect();
        console.log("Connected to DB");
    } catch (error) {
        console.log(error);
    }
};

const main = async () => {
    try {
        await connectToDB();
    } catch (error) {
        console.log(error);
    } finally {
        await client.close();
        console.log("Disconnected from DB");
    }
}

main();
```

This is fine for a one-off script. In a server it would be wrong to connect and close per request.

## Modern version (ESM, reused client)
```js
// db.js
import { MongoClient } from 'mongodb';

const uri = process.env.MONGODB_URI;               // e.g. set in .env / secrets manager
if (!uri) throw new Error('MONGODB_URI is not set');

export const client = new MongoClient(uri, {
  appName: 'bank-api',
  maxPoolSize: 50,                  // default 100
  serverSelectionTimeoutMS: 5000,
});

export const db = client.db('bank');

// Close once, on shutdown
process.on('SIGTERM', async () => { await client.close(); process.exit(0); });
```

```js
// accounts.js
import { db } from './db.js';

export async function getAccount(accountId) {
  // No explicit client.connect(): the driver connects on first use (4.7+)
  return db.collection('accounts').findOne({ account_id: accountId });
}
```

Quick connectivity check:

```js
await db.command({ ping: 1 });
```

TypeScript users can type collections: `db.collection<Account>('accounts')`.

## Common pitfalls
- **New `MongoClient` per request**: opens new pools every time, exhausts connections, adds TLS handshake latency.
- **Serverless functions** (Lambda, Vercel): create the client **outside** the handler so warm invocations reuse it; keep `maxPoolSize` small.
- Forgetting `await` on operations: the promise is dropped and errors are unhandled.
- Catching and only `console.log`-ing connection errors, then continuing as if connected.

<Info>
**🆕 2025–2026**

**Node.js driver v7.0.0 (released 2025-11-06)**:
- Minimum **Node.js 20.19.0**; ships with BSON 7.
- Closes **idle connections even when `minPoolSize` is 0** (`maxIdleTimeMS` is honoured).
- Allocates sessions for cursors **lazily**.
- Boolean options no longer accept non-boolean values (e.g. the string `"false"` in options objects).
- MONGODB-AWS auth now requires `@aws-sdk/credential-providers`; credentials in the URI are no longer accepted. MONGODB-CR removed. Dropping a non-existent collection returns `false` instead of throwing.
([release announcement](https://www.mongodb.com/community/forums/t/mongodb-nodejs-driver-7-0-0-released/331061))
</Info>

## Senior perspective
- **Pool sizing**: total connections = instances × `maxPoolSize`. With 40 pods × 100 you can hit cluster connection limits; size pools from measured concurrency, and watch the Atlas connections metric. See [Connection Pooling](/Software%20Engineering/Databases/Database%20Round/Connection%20Pooling).
- **Timeouts and retries**: set `serverSelectionTimeoutMS`, use `timeoutMS` (CSOT) for per-operation deadlines, keep `retryWrites`/`retryReads` on.
- **Graceful shutdown**: close the client on `SIGTERM` so in-flight operations finish and connections are released.
- **Secrets**: use env vars injected from a vault, or passwordless auth (AWS IAM, OIDC workload identity) so there is nothing to leak.
- **ODMs**: Mongoose adds schemas and middleware but also overhead and its own quirks; the native driver plus a validator (Zod) is a lighter option.

## Interview questions
<Accordion title="Why should you reuse one MongoClient instead of creating one per request?">
A `MongoClient` owns a connection pool and server monitoring. Creating one per request repeats DNS/TLS/auth handshakes, multiplies connections to the cluster and can exhaust server limits. One client per process, shared by all requests, reuses pooled connections.
</Accordion>

<Accordion title="Do you need to call client.connect()?">
Not since driver 4.7: the first operation connects automatically. Calling it explicitly is still useful at startup to fail fast if the database is unreachable.
</Accordion>

<Accordion title="What is maxPoolSize and what is its default?">
The maximum number of concurrent connections the client keeps per server. Default 100. Requests beyond it wait for a free connection (up to `waitQueueTimeoutMS`).
</Accordion>

<Accordion title="How do you handle MongoDB connections in serverless functions?">
Create the client in module scope (outside the handler) so warm invocations reuse it, keep the pool small, set timeouts, and consider connection limits across many concurrent function instances.
</Accordion>

## Related
- [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Connection Pooling](/Software%20Engineering/Databases/Database%20Round/Connection%20Pooling) · [Unit 10 MongoDB CRUD Operations in Node.js](/Software%20Engineering/Databases/MongoDB/Unit%2010%20MongoDB%20CRUD%20Operations%20in%20Node.js)

## Sources
- [MongoDB Node.js Driver docs](https://www.mongodb.com/docs/drivers/node/current/)
- [Node.js driver: connection pools](https://www.mongodb.com/docs/drivers/node/current/connect/connection-options/connection-pools/)
- [Node.js driver 7.0.0 release](https://www.mongodb.com/community/forums/t/mongodb-nodejs-driver-7-0-0-released/331061)
- [MongoDB University: Connecting to MongoDB in Node.js](https://learn.mongodb.com/)
