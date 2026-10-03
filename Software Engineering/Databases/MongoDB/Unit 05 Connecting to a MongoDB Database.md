---
tags: [databases, databases/mongodb]
aliases: [MongoDB Connection String, mongodb+srv, MongoDB Compass, MongoDB Unit 5]
updated: 2026-09-27
---
# Unit 05 Connecting to a MongoDB Database

> [!tldr] TL;DR
> A **connection string** tells a client (mongosh, Compass, a driver) which hosts to contact and with which options. There are two formats: the **standard** `mongodb://host1,host2,...` and the **DNS seed list** `mongodb+srv://cluster.example.net`, which looks up the host list in a DNS **SRV** record and turns **TLS on** by default. Connect with `mongosh` for the shell, **Compass** for a GUI, or a **driver** in your app. Most connection failures are network access (IP access list) or authentication.

> Course: MongoDB University, Unit 5 · Prev: [[Unit 04 MongoDB Data Modeling Intro]] · Next: [[Unit 06 Connecting to MongoDB in Node.js]]

## Connection string formats
A connection string lets us connect to a cluster; it describes the hosts and the options used to connect.

1. **Standard format** (`mongodb://`): lists every host explicitly. Works for standalone servers, replica sets and sharded clusters (list the `mongos` routers).
2. **DNS seed list format** (`mongodb+srv://`): gives one hostname; the client resolves a DNS **SRV** record to get the list of servers (and a TXT record for default options such as `replicaSet` and `authSource`). You can rotate servers in DNS without reconfiguring clients. Atlas always gives you this format.

```text
mongodb://user:pass@host1:27017,host2:27017,host3:27017/mydb?replicaSet=rs0&tls=true
mongodb+srv://user:pass@cluster0.abcde.mongodb.net/mydb?retryWrites=true&w=majority
```

## Components of a connection string
```text
mongodb+srv://admin:<password>
	@cluster0.rvygg17.mongodb.net/
	?retryWrites=true&w=majority
```

| Part | Meaning |
|---|---|
| `mongodb` | **scheme/prefix**: this is a MongoDB connection string |
| `+srv` | use a **DNS SRV seed list** lookup for the hosts; also sets `tls=true` by default; no port allowed in the host |
| `admin:<password>` | username and password (URL-encode special characters) |
| `cluster0.rvygg17.mongodb.net` | host. With `+srv` it is the SRV name; with the standard format each `host:port`, default port **27017** |
| `/mydb` | optional default database (also default `authSource` unless set) |
| `?retryWrites=true&w=majority` | options |

(The original note mixed these up: it said the `mongodb` prefix uses the DNS seed list and `+srv` "sets security to true". In fact the seed list and TLS both come from `+srv`.)

Common options:
- `retryWrites=true`: the driver retries certain failed writes **once** after a transient network error or primary failover (default on in modern drivers).
- `w=majority`: write concern; acknowledge writes only after a majority of replica-set members have them.
- `appName`, `maxPoolSize`, `connectTimeoutMS`, `serverSelectionTimeoutMS`, `readPreference`, `authSource`.

## Connecting with mongosh
The MongoDB Shell is built on the Node.js REPL, so you have full JavaScript inside it.

![[Screenshot_2023-04-22_at_9.32.08_PM.png]]

```bash
mongosh "mongodb+srv://cluster0.rvygg17.mongodb.net/myFirstDatabase" --apiVersion 1 --username admin
```

`--username` without `--password` makes mongosh prompt for the password, which keeps it out of your shell history. `--apiVersion 1` pins the Stable API.

![[Screenshot_2023-04-22_at_9.30.08_PM.png]]

Get the connection string from the Atlas CLI:

```bash
atlas clusters connectionStrings describe myAtlasClusterEDU
```

## Connecting with Compass
**Compass** is MongoDB's desktop GUI: paste the connection string, then browse and query data, see schema analysis, view `explain` plans, manage indexes and build aggregation pipelines visually.

## Connecting from application drivers
Drivers (Node.js, Python, Java, Go, C#, ...) connect your application with the same connection string. See [[Unit 06 Connecting to MongoDB in Node.js]] and the driver docs on mongodb.com/docs/drivers.

## Troubleshooting Atlas connection errors
| Error | Likely cause | Fix |
|---|---|---|
| Timeout / `MongoServerSelectionError` | your IP is not in the project's **IP access list**, firewall, VPN | add your IP (`atlas accessLists create --currentIp`), check private endpoint/peering |
| `querySrv ENOTFOUND` / `ECONNREFUSED` on SRV | DNS cannot resolve SRV records (some corporate/ISP DNS) | use another DNS resolver or the standard `mongodb://` string from Atlas |
| `bad auth : authentication failed` | wrong username/password, wrong `authSource`, unencoded special chars | reset the database user password, URL-encode it |
| Not authorized on db | user lacks the role | grant a role scoped to that database |

## Senior perspective
- **Never hard-code credentials**: load the URI from an environment variable or a secrets manager; prefer short-lived credentials (AWS IAM, OIDC/workload identity, X.509) where supported.
- **Timeouts**: set `serverSelectionTimeoutMS` and operation timeouts (`timeoutMS` in recent drivers) so a failing cluster doesn't hang requests.
- **Write concern and read preference** are part of the connection contract: `w=majority` for durability, `readPreference=secondaryPreferred` only if the app tolerates stale reads.
- **Retryable writes** make single-document writes safe to retry after failovers (idempotent at the server via a transaction number), but not `updateMany`/`deleteMany`.
- **One client per process** with a connection pool; see [[Connection Pooling]].

## Interview questions
> [!question]- What is the difference between mongodb:// and mongodb+srv://?
> `mongodb://` lists hosts explicitly. `mongodb+srv://` gives one hostname; the driver looks up DNS SRV (hosts) and TXT (options) records, enables TLS by default, and lets operators change servers without changing client config. No port may be given with `+srv`.

> [!question]- What does retryWrites=true do?
> The driver automatically retries a supported write once if it fails due to a transient network error or a primary election. The server deduplicates using the session's transaction number so the write isn't applied twice.

> [!question]- Your app can't connect to Atlas. How do you debug it?
> Check the error type: server selection timeout usually means the IP access list or network (VPN, firewall, private endpoint); SRV DNS errors mean DNS resolution; auth errors mean user, password encoding or `authSource`. Test with `mongosh` from the same host.

> [!question]- What does w=majority mean?
> The write is acknowledged only after a majority of voting replica-set members have applied it, so it survives a primary failover and won't be rolled back.

## Related
- [[MongoDB]] · [[Databases Index]]
- [[Connection Pooling]] · [[Replication & 2PC 3PC]]

## Sources
- [MongoDB Manual: Connection Strings](https://www.mongodb.com/docs/manual/reference/connection-string/)
- [Retryable Writes](https://www.mongodb.com/docs/manual/core/retryable-writes/)
- [Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/)
- [MongoDB Compass](https://www.mongodb.com/docs/compass/)
- [Atlas: Troubleshoot connection issues](https://www.mongodb.com/docs/atlas/troubleshoot-connection/)
- [MongoDB University: Connecting to a MongoDB Database](https://learn.mongodb.com/)
