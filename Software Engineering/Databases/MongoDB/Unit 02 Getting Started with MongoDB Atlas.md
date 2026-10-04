---
title: "Unit 02 Getting Started with MongoDB Atlas"
tags: [databases, databases/mongodb]
aliases: [MongoDB Atlas, Atlas, Atlas CLI, MongoDB Unit 2, Flex clusters]
updated: 2026-09-27
---
# Unit 02 Getting Started with MongoDB Atlas

<Note>
**TL;DR**

**Atlas** is MongoDB's managed cloud data platform: at its core a Database-as-a-Service that runs, patches, backs up and scales MongoDB for you on AWS, Azure or GCP. Every Atlas deployment is at least a 3-node **replica set**. Today you choose between **Free (M0)**, **Flex** and **Dedicated (M10+)** clusters; the old Serverless and M2/M5 shared tiers the course describes are gone since 2026-01-22. You can drive everything from the UI or the **Atlas CLI**.
</Note>

> Course: MongoDB University, Unit 2 · Prev: [Unit 01 Start Here - Intro to MongoDB](/Software%20Engineering/Databases/MongoDB/Unit%2001%20Start%20Here%20-%20Intro%20to%20MongoDB) · Next: [Unit 03 MongoDB and the Document Model](/Software%20Engineering/Databases/MongoDB/Unit%2003%20MongoDB%20and%20the%20Document%20Model)

## What Atlas is
- A data platform built around a **DBaaS**: Atlas manages the database and its lifecycle (provisioning, upgrades, backups, monitoring).
- Each deployment automatically gets a **replica set**: a group of `mongod` servers holding copies of the same data (one primary, secondaries). See [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC).
- Extra services on top: Atlas Search and Vector Search ([Unit 13 MongoDB Atlas Search](/Software%20Engineering/Databases/MongoDB/Unit%2013%20MongoDB%20Atlas%20Search)), backups with point-in-time restore, Online Archive, Performance Advisor, Data Federation.

## Deployment types

### As the course taught it (historical, 2023)
<Note>
**Original course description (no longer current)**

There’s two types of database deployments:
1. **Serverless**: scales on demand and charges you for the resources you use. A good fit for applications with sparse or spiky activity over time.
2. **Clusters**: several MongoDB servers working together.
   1. **Dedicated**: larger, can be customized, and scales automatically.
   2. **Shared**: smaller, and includes the Free tier.
</Note>

### Current (2026)
| Tier | What it is | Use for |
|---|---|---|
| **Free (M0)** | Small shared cluster, free forever, limited storage and features | learning, prototypes, the course labs |
| **Flex** | Pay-as-you-go shared cluster with a monthly cap; replaced Serverless and M2/M5 | dev/test, small or spiky apps |
| **Dedicated (M10 and up)** | Your own VMs, auto-scaling, full feature set (sharding, private networking, advanced backup) | production |

<Info>
**🆕 2025–2026**

- **Feb 2025**: Atlas stopped letting you create **M2/M5** shared clusters and **Serverless instances**; you create **Flex** clusters instead.
- **2026-01-22**: Atlas **no longer supports M2/M5 clusters and Serverless instances**; existing M2/M5 clusters were migrated to Flex, and Serverless instances were migrated to Free, Flex or Dedicated based on usage. ([Flex migration docs](https://www.mongodb.com/docs/atlas/flex-migration/))
- **2025-09-30**: **Atlas Device Sync** and the **Device SDKs** (Realm), plus the **Atlas Data API** and **HTTPS Endpoints**, reached **end of life** (announced September 2024). The "Device Sync" feature below is historical; mobile apps now sync through their own backend or third-party sync tools. ([deprecation notice](https://www.mongodb.com/community/forums/t/atlas-device-sync-end-of-life-and-deprecation/296687))
</Info>

## Atlas features from the course
- Operational insights, backups with **point-in-time restore**, and **Online Archive** (moves cold data to cheaper object storage while keeping it queryable).
- **Atlas Search**: build a full-text search experience in the cloud, next to your data.
- **Atlas Device Sync** keeps data synced between the cloud and mobile devices. ⚠️ End of life on 2025-09-30, see above.
- **Atlas Data Lake**: in the course, a way to extract data from the database into your own low-cost storage for analytics. Its current status is not confirmed here; check the current docs. The current product for querying S3/Azure/GCS data and archives alongside Atlas is **Atlas Data Federation**.

## Organizations and projects
- **Organizations**: the top level. Group and define users and teams, grant them access to projects, hold billing.
- **Projects**: inside an organization. Define and organize resources (clusters, database users, network access lists). A common setup is one project per environment (dev, staging, prod).

{/* diagram source: /diagrams/diagram-128.mmd */}
![diagram](/diagrams/diagram-128.svg)

## Atlas CLI
Register (or log in) from the terminal. The course note had a typo (`altlas`):

```bash
atlas auth register   # new account
atlas auth login      # existing account
```

Then copy the verification code and paste it into the web page that opens.

You can set up a cluster from the CLI. This is the command as used in the course lab:

```bash
atlas setup --clusterName myAtlasClusterEDU 
	--provider AWS 
	--currentIp 
	--skipSampleData 
	--username myAtlasDBUser 
	--password myatlas-001 | 
	tee atlas_cluster_details.txt
```

<Warning>
**Credentials in this note**

The command above contains a real database password in plain text, and `tee` writes it to a file. Treat that password as exposed: **rotate it** (or delete the lab user) in Atlas. In your own scripts use a placeholder or an environment variable, e.g. `--password "$ATLAS_DB_PASSWORD"`, or omit `--password` and let the CLI prompt. Do not commit files like `atlas_cluster_details.txt`.
</Warning>

Also note the multi-line command needs `\` at the end of each line to run in bash. A safe version:

```bash
atlas setup --clusterName myAtlasClusterEDU \
  --provider AWS \
  --currentIp \
  --skipSampleData \
  --username "$ATLAS_DB_USER" \
  --password "$ATLAS_DB_PASSWORD"
```

This step leaves you connected to the cluster in `mongosh`.

![Screenshot_2023-04-22_at_5.56.16_PM](/Attachments/Screenshot_2023-04-22_at_5.56.16_PM.png)

Load the sample datasets (`sample_mflix`, `sample_supplies`, `sample_analytics` and others used in later units):

```bash
atlas clusters loadSampleData myAtlasClusterEDU
```

Other useful commands:

```bash
atlas clusters list
atlas clusters describe myAtlasClusterEDU
atlas accessLists create --currentIp          # allow your IP
atlas dbusers list
```

## Common pitfalls
- **Connection refused/timeouts**: your IP is not on the project's access list ([Unit 05 Connecting to a MongoDB Database](/Software%20Engineering/Databases/MongoDB/Unit%2005%20Connecting%20to%20a%20MongoDB%20Database)).
- Using `0.0.0.0/0` in the access list "to make it work" exposes the cluster to the internet; use specific IPs, VPC peering or private endpoints in production.
- Free/Flex tiers have limits (storage, operations, some features); do not benchmark on them.

## Senior perspective
- **Environment isolation**: separate projects (and ideally organizations or at least teams) for prod; least-privilege database users per service, not one `readWriteAnyDatabase` user for everything.
- **Networking**: private endpoints / VPC peering, TLS always on (it is enforced by Atlas).
- **Backups**: know your RPO/RTO; continuous backup with point-in-time restore costs more but is what you want for production. Test restores.
- **Sizing and scaling**: vertical scaling (bigger tier) first; sharding when a single replica set cannot hold the working set or write load. See [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning).
- **Infrastructure as code**: manage clusters, users and access lists with Terraform or the Atlas Kubernetes Operator, not by clicking.
- **Vendor lifecycle risk**: Atlas has retired products (Device Sync, Data API, Serverless). Avoid building core architecture on preview or niche services without an exit plan.

## Interview questions
<Accordion title="What does Atlas manage for you compared with self-hosting MongoDB?">
Provisioning, replica-set configuration, patching and version upgrades, backups and restore, monitoring and alerts, auto-scaling and security defaults (TLS, auth). You still own schema design, indexes, queries and access control choices.
</Accordion>

<Accordion title="Why is every Atlas cluster a replica set?">
For high availability and durability: if the primary fails, a secondary is elected primary automatically, and data exists on multiple nodes. It also enables majority write concern, change streams and transactions.
</Accordion>

<Accordion title="What replaced Atlas Serverless instances and M2/M5 clusters?">
Flex clusters (since February 2025). Support for Serverless and M2/M5 ended on 2026-01-22 and existing ones were migrated.
</Accordion>

<Accordion title="How do organizations and projects differ in Atlas?">
Organizations hold billing, users and teams and contain projects. Projects hold the actual resources (clusters, database users, network access) and are the usual boundary between environments or applications.
</Accordion>

## Related
- [MongoDB](/Software%20Engineering/Databases/MongoDB/MongoDB) · [Databases Index](/Software%20Engineering/Databases/Databases%20Index)
- [Replication & 2PC 3PC](/Software%20Engineering/Databases/Database%20Round/Replication%20%26%202PC%203PC) · [Sharding & Partitioning](/Software%20Engineering/Databases/Database%20Round/Sharding%20%26%20Partitioning)

## Sources
- [MongoDB Atlas documentation](https://www.mongodb.com/docs/atlas/)
- [Atlas: Migrate to Flex clusters](https://www.mongodb.com/docs/atlas/flex-migration/)
- [Atlas CLI: atlas setup](https://www.mongodb.com/docs/atlas/cli/current/command/atlas-setup/)
- [Atlas Device Sync end of life and deprecation](https://www.mongodb.com/community/forums/t/atlas-device-sync-end-of-life-and-deprecation/296687)
- [Atlas Data Federation](https://www.mongodb.com/docs/datalake/)
- [MongoDB University: Getting Started with MongoDB Atlas](https://learn.mongodb.com/)
