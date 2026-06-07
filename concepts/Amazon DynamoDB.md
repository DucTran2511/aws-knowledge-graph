---
tags: [concept, serverless, database, nosql, dynamodb, key-value]
aliases: [DynamoDB, Amazon DynamoDB, DDB, DAX, DynamoDB Accelerator, DynamoDB Streams, Global Tables]
date: 2026-05-29
---

# Amazon DynamoDB

**Amazon DynamoDB** is a fully managed, serverless **NoSQL key-value and document database**. It delivers single-digit millisecond performance at any scale with built-in security, backup, and in-memory caching. DynamoDB is a core pillar of the [[Serverless on AWS]] ecosystem.

> [!IMPORTANT]
> **Core exam concept:** DynamoDB is the answer for **serverless NoSQL**, **single-digit millisecond latency**, **key-value lookups**, and **auto-scaling database**. If the question mentions "relational," "SQL joins," or "ACID transactions across tables" → **[[Amazon RDS]]/[[Amazon Aurora]]**, not DynamoDB.

---

## Table Structure

```
  ┌──────────────────────────────────────────────────────────────┐
  │                       DynamoDB Table                          │
  │                                                              │
  │  Primary Key = Partition Key (PK)  +  Sort Key (SK)         │
  │                (required)              (optional)            │
  │                                                              │
  │  ┌──────────┬──────────┬──────────┬──────────┬──────────┐   │
  │  │    PK    │    SK    │  Attr1   │  Attr2   │  Attr3   │   │
  │  ├──────────┼──────────┼──────────┼──────────┼──────────┤   │
  │  │ user#1   │ order#1  │  item_A  │  $10.00  │          │   │
  │  │ user#1   │ order#2  │  item_B  │  $25.00  │  "rush"  │   │
  │  │ user#2   │ order#1  │  item_C  │  $5.00   │          │   │
  │  │ user#3   │          │  item_D  │          │  "VIP"   │   │
  │  └──────────┴──────────┴──────────┴──────────┴──────────┘   │
  │                                                              │
  │  • Each item can have DIFFERENT attributes (schemaless)      │
  │  • Max item size: 400 KB                                     │
  │  • Partition Key determines which partition stores data      │
  │  • PK only = Simple Primary Key                              │
  │  • PK + SK = Composite Primary Key                           │
  └──────────────────────────────────────────────────────────────┘
```

### Primary Key Design

| Key Type | Structure | Query Capability |
|---|---|---|
| **Simple Primary Key** | Partition Key only | Get exact item by PK |
| **Composite Primary Key** | Partition Key + Sort Key | Query all items with same PK, filter/sort by SK |

> [!CAUTION]
> **Exam critical:** Good partition key design = **high cardinality** (many unique values). Bad PK = hot partitions → throttling. Example: `user_id` = good PK (many users). `status` = bad PK (only a few values like "active"/"inactive" → hot partition).

---

## Secondary Indexes

```
  BASE TABLE:
  PK = user_id, SK = order_date

  ┌──────────┬────────────┬──────────┬──────────┐
  │ user_id  │ order_date │ product  │ amount   │
  ├──────────┼────────────┼──────────┼──────────┤
  │ user#1   │ 2026-01-15 │ laptop   │ $1200    │
  │ user#1   │ 2026-03-22 │ mouse    │ $25      │
  │ user#2   │ 2026-02-10 │ keyboard │ $75      │
  └──────────┴────────────┴──────────┴──────────┘

  GSI (Global Secondary Index):        LSI (Local Secondary Index):
  PK = product, SK = amount            PK = user_id (SAME), SK = product
  → Query: "all laptops over $500"     → Query: "user#1's orders by product"
  → Different PK, different view       → Same PK, different sort order
```

| Feature | Global Secondary Index (GSI) | Local Secondary Index (LSI) |
|---|---|---|
| **Partition Key** | **Different** from base table | **Same** as base table |
| **Sort Key** | Optional, can be different | **Different** from base table |
| **Max per table** | **20** | **5** |
| **When to create** | **Anytime** (add/remove dynamically) | **At table creation only** |
| **Capacity** | **Own** RCU/WCU (independent) | **Shares** table's RCU/WCU |
| **Consistency** | Eventually consistent only | Strongly or eventually consistent |
| **Size limit** | No limit | 10 GB per partition key value |

> [!WARNING]
> **Exam trap:** "Query by a different partition key" → **GSI**. "Query by a different sort key within the same partition" → **LSI**. **If you forgot to add an LSI at table creation, you must recreate the table.** GSIs can have their own provisioned throughput — if under-provisioned, writes to the base table will be throttled (**GSI throttling back-pressure**).

---

## Capacity Modes

### Provisioned Mode (Default)

```
  You specify capacity. Best for predictable workloads.

  ┌──────────────────────────────────────────────────────────┐
  │  RCU (Read Capacity Unit):                                │
  │  • 1 RCU = 1 Strongly Consistent read/sec   (up to 4 KB)│
  │  • 1 RCU = 2 Eventually Consistent reads/sec (up to 4 KB)│
  │  • 1 Transactional read = 2 RCUs                         │
  │                                                           │
  │  WCU (Write Capacity Unit):                               │
  │  • 1 WCU = 1 write/sec (up to 1 KB)                     │
  │  • 1 Transactional write = 2 WCUs                        │
  └──────────────────────────────────────────────────────────┘
```

### RCU/WCU Calculation Examples

```
  READS:
  ──────
  Example 1: 10 Strongly Consistent reads/sec of 8 KB items
  = 10 × ceil(8 KB / 4 KB) = 10 × 2 = 20 RCU

  Example 2: 10 Eventually Consistent reads/sec of 8 KB items
  = (10 × ceil(8 KB / 4 KB)) / 2 = (10 × 2) / 2 = 10 RCU

  Example 3: 5 Transactional reads/sec of 8 KB items
  = 5 × ceil(8 KB / 4 KB) × 2 = 5 × 2 × 2 = 20 RCU

  WRITES:
  ───────
  Example 1: 10 writes/sec of 3 KB items
  = 10 × ceil(3 KB / 1 KB) = 10 × 3 = 30 WCU

  Example 2: 5 transactional writes/sec of 3 KB items
  = 5 × ceil(3 KB / 1 KB) × 2 = 5 × 3 × 2 = 30 WCU
```

### On-Demand Mode

```
  No capacity planning required.

  ✅ Auto-scales instantly with workload
  ✅ Pay per read/write request
  ✅ No throttling (within account limits)
  ❌ ~2.5× more expensive than provisioned
  ❌ No Auto Scaling configuration needed

  Best for:
  • New tables with unknown workloads
  • Unpredictable, spiky traffic
  • Low-admin, pay-per-use preference
```

| Mode | Planning | Cost | Scaling | Use Case |
|---|---|---|---|---|
| **Provisioned** | You set RCU/WCU | 💰 Cheaper | Manual or Auto Scaling | Predictable, steady traffic |
| **On-Demand** | None | 💰💰 More expensive | Instant, automatic | Unpredictable, spiky traffic |

> [!TIP]
> **Exam Pattern:** "Unpredictable traffic" or "new application, unknown workload" → **On-Demand mode**. "Steady, predictable traffic" or "cost optimization" → **Provisioned mode with Auto Scaling**. You can switch between modes **once per 24 hours**.

---

## Read Consistency

```
  WRITE to DynamoDB
       │
       ▼
  ┌─────────────┐    async replication    ┌─────────────┐
  │ Partition A │ ──────────────────────►│ Replica A'  │
  │ (leader)    │    (usually < 1 sec)   │             │
  └─────────────┘                        └─────────────┘

  Eventually Consistent Read (default):
  → May read from replica that hasn't received the latest write
  → 2 reads per RCU (cheaper)

  Strongly Consistent Read:
  → Always reads from the leader partition
  → 1 read per RCU (2× cost)
```

| Consistency | Behavior | Cost |
|---|---|---|
| **Eventually Consistent** (default) | Read may return stale data (replication lag ~ms) | 1 RCU = **2** reads/sec |
| **Strongly Consistent** | Read always returns latest data | 1 RCU = **1** read/sec |
| **Transactional** | ACID across multiple items/tables | **2× RCU/WCU** |

---

## DynamoDB Accelerator (DAX)

```
  Without DAX:                          With DAX:
  ┌──────┐      ┌──────────┐           ┌──────┐      ┌──────┐      ┌──────────┐
  │ App  │─────►│ DynamoDB │           │ App  │─────►│ DAX  │─────►│ DynamoDB │
  │      │      │ (ms)     │           │      │      │(μs)  │      │          │
  └──────┘      └──────────┘           └──────┘      └──────┘      └──────────┘
                                                      Cache HIT:
                                                      microsecond latency
                                                      Cache MISS:
                                                      reads from DynamoDB,
                                                      populates cache
```

### DAX Architecture

```
  ┌──────────────────────────────────────────────────┐
  │  DAX Cluster                                      │
  │                                                   │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
  │  │ Primary  │  │ Replica  │  │ Replica  │       │
  │  │  Node    │  │  Node    │  │  Node    │       │
  │  └──────────┘  └──────────┘  └──────────┘       │
  │                                                   │
  │  • Multi-AZ for high availability                │
  │  • In-memory (microsecond reads)                 │
  │  • Same API as DynamoDB (no code changes!)       │
  │  • Two caches: Item Cache + Query Cache          │
  │  • Default TTL: 5 minutes                        │
  │  • Up to 10 nodes per cluster                    │
  └──────────────────────────────────────────────────┘
```

| Feature | DAX | [[Amazon ElastiCache]] |
|---|---|---|
| **Purpose** | Cache for DynamoDB specifically | General-purpose cache (any data) |
| **API** | Same as DynamoDB (transparent) | Redis/Memcached API (different) |
| **Code changes** | Minimal (change endpoint) | Significant (caching logic) |
| **Use case** | "Make DynamoDB reads faster" | "Cache computed results, sessions, etc." |
| **Latency** | Microseconds | Sub-millisecond |

> [!CAUTION]
> **Exam critical:** "Microsecond reads from DynamoDB" or "cache for DynamoDB with no code changes" → **DAX**. "General-purpose caching" or "cache for RDS/custom app" → **ElastiCache**. DAX is **read-through** cache — NOT for write-heavy workloads. If you need to cache **aggregated/computed results**, use ElastiCache instead.

---

## DynamoDB Streams

```
  ┌──────────┐    Item changes     ┌────────────────┐    trigger    ┌──────────┐
  │ DynamoDB │───────────────────►│  DynamoDB       │────────────►│  Lambda  │
  │  Table   │  (insert/update/   │  Stream         │              │ Function │
  │          │   delete)          │  (24h retention)│              │          │
  └──────────┘                    └────────────────┘              └──────────┘
```

### Stream View Types

| View Type | What's Captured |
|---|---|
| **KEYS_ONLY** | Only the key attributes of the modified item |
| **NEW_IMAGE** | The entire item as it appears **after** modification |
| **OLD_IMAGE** | The entire item as it appeared **before** modification |
| **NEW_AND_OLD_IMAGES** | Both the new and old images of the item |

### Stream Use Cases

| Use Case | Pattern |
|---|---|
| **React to changes in real-time** | DynamoDB Streams → [[AWS Lambda]] → process/notify |
| **Cross-region replication** | Powers **Global Tables** internally |
| **Materialized views** | Stream changes → Lambda → write to another table/index |
| **Analytics pipelines** | Stream → Lambda → [[Amazon Kinesis]] Firehose → S3 → Athena |
| **Audit log** | Capture all changes with OLD_IMAGE + NEW_IMAGE |
| **Send notifications** | Stream → Lambda → [[Amazon SNS]]/SES |

> [!TIP]
> **Exam Pattern:** "React to DynamoDB changes in real-time" → **DynamoDB Streams + Lambda**. "Audit trail of all changes" → Stream with **NEW_AND_OLD_IMAGES**. Streams retain data for **24 hours**.

---

## DynamoDB Global Tables

```
  ┌────────────────┐         ┌────────────────┐
  │  us-east-1     │◄───────►│  eu-west-1     │
  │  DynamoDB      │  async  │  DynamoDB      │
  │  (read/write)  │  repli- │  (read/write)  │
  │                │  cation │                │
  └────────────────┘         └────────────────┘
         ▲                          ▲
         │                          │
         ▼                          ▼
  ┌────────────────┐
  │  ap-southeast-1│
  │  DynamoDB      │
  │  (read/write)  │
  └────────────────┘

  ✅ Multi-region, multi-active (read AND write in any region)
  ✅ Sub-second replication between regions
  ✅ Fully managed — no application changes
  ✅ "Last writer wins" conflict resolution
```

### Global Tables Requirements

| Requirement | Detail |
|---|---|
| **DynamoDB Streams** | **Must be enabled** (uses streams for replication) |
| **Capacity mode** | On-Demand or Provisioned with Auto Scaling |
| **Table must be empty** | When adding a new region replica (for new tables) |

> [!WARNING]
> **Exam trap:** "Multi-region active-active database" + "low latency" → **DynamoDB Global Tables**. Don't confuse with [[Amazon Aurora]] Global Database (which is active-passive for writes, < 1 second replication, RTO < 1 min). DynamoDB Global Tables = **active-active** (read AND write anywhere). Aurora Global = **one writer region**.

---

## DynamoDB Transactions

```
  ┌─────────────────────────────────────────────────────┐
  │  TransactWriteItems (up to 100 items, 4 MB)          │
  │                                                      │
  │  Operation 1: Put item in Orders table      ── ✅   │
  │  Operation 2: Update item in Inventory table── ✅   │
  │  Operation 3: Delete item in Cart table     ── ✅   │
  │                                                      │
  │  ALL succeed or ALL fail (ACID)                      │
  │  Consumes 2× WCU/RCU (compared to standard ops)    │
  └─────────────────────────────────────────────────────┘
```

> [!NOTE]
> DynamoDB supports **ACID transactions** across multiple items and tables using `TransactWriteItems` and `TransactGetItems`. Each transactional operation costs **2× the normal RCU/WCU**. Max 100 items or 4 MB per transaction.

---

## Other DynamoDB Features

| Feature | Description |
|---|---|
| **TTL (Time-to-Live)** | Automatically delete expired items. Free — no WCU consumed. Deletion is async (may take 48h). |
| **Point-in-Time Recovery (PITR)** | Continuous backups, restore to any second in last **35 days**. |
| **On-Demand Backup** | Full backups for long-term retention. No performance impact. |
| **S3 Export** | Export table data to S3 (Parquet/JSON) **without consuming RCU**. Requires PITR enabled. |
| **S3 Import** | Import from S3 (CSV/JSON/ION) to create a **new** table. No WCU consumed. |
| **PartiQL** | SQL-compatible query language for DynamoDB (SELECT, INSERT, UPDATE, DELETE). |
| **Conditional Writes** | Write only if a condition is met (optimistic locking / concurrency control). |
| **Batch Operations** | `BatchGetItem` (up to 100 items), `BatchWriteItem` (up to 25 items). |

---

## DynamoDB Security

| Feature | Description |
|---|---|
| **Encryption at rest** | Default: AWS owned key. Optional: KMS managed key or customer managed key. |
| **Encryption in transit** | HTTPS (TLS) by default. |
| **IAM policies** | Fine-grained access control (per-table, per-item, per-attribute). |
| **VPC Endpoints** | Gateway Endpoint (free) for private access from VPC. |
| **Backup encryption** | Backups are encrypted with the same key as the source table. |

---

## Key DynamoDB Limits

| Parameter | Value |
|---|---|
| **Max item size** | **400 KB** |
| **Max GSIs per table** | **20** |
| **Max LSIs per table** | **5** (defined at creation only) |
| **Partition key max size** | 2,048 bytes |
| **Sort key max size** | 1,024 bytes |
| **Max tables per region** | 2,500 (soft limit) |
| **Max provisioned throughput** | 40,000 RCU / 40,000 WCU per table (soft limit) |
| **Max items per `BatchGetItem`** | 100 items or 16 MB |
| **Max items per `BatchWriteItem`** | 25 items or 16 MB |
| **Max items per Transaction** | 100 items or 4 MB |
| **Stream retention** | 24 hours |
| **DAX cluster max nodes** | 10 |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → DynamoDB:**
> - "Serverless NoSQL database" → **DynamoDB**
> - "Single-digit millisecond latency" → **DynamoDB**
> - "Key-value store" or "document database" → **DynamoDB**
> - "Microsecond read latency" → **DAX** (DynamoDB cache)
> - "Multi-region active-active database" → **DynamoDB Global Tables**
> - "React to database changes in real-time" → **DynamoDB Streams + Lambda**
> - "Unpredictable traffic, no capacity planning" → **On-Demand mode**
> - "Cost-optimize predictable workload" → **Provisioned with Auto Scaling**
> - "Auto-delete expired data" → **TTL** (free, no WCU)
> - "Export DynamoDB data to S3 for analytics" → **S3 Export** (no RCU, needs PITR)
> - "ACID transactions across items" → **DynamoDB Transactions** (2× RCU/WCU)
> - "SQL queries on DynamoDB" → **PartiQL**
>
> **Key facts:**
> - Max item size: 400 KB. For larger objects → store in S3, save S3 key in DynamoDB.
> - 20 GSI (add anytime, own capacity). 5 LSI (creation only, shares capacity).
> - RCU: 1 strongly / 2 eventually consistent reads per 4 KB. WCU: 1 write per 1 KB.
> - Transactional reads/writes cost 2× RCU/WCU.
> - DAX = DynamoDB-specific cache (same API, microsecond). ElastiCache = general purpose.
> - Global Tables: multi-active, requires Streams enabled, sub-second replication.
> - Streams: 24h retention, 4 view types (KEYS_ONLY → NEW_AND_OLD_IMAGES).
> - VPC Gateway Endpoint for DynamoDB = **free** private access.
> - TTL deletes are async and may take up to 48 hours. They don't consume WCU.
