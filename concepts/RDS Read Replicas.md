---
tags: [concept, database, scaling, replication]
aliases: [Read Replica, Read Replicas, RDS Read Replica, RDS Read Replicas]
date: 2026-04-30
---

# RDS Read Replicas

**Read Replicas** are **asynchronous**, read-only copies of your [[Amazon RDS]] primary database instance. They are the primary mechanism for **horizontal read scaling** in RDS. Unlike [[Amazon RDS#RDS Multi-AZ Deployments|Multi-AZ]] (which is for high availability), Read Replicas are for **performance**.

---

## How Replication Works

```
Application (Writes)──────────────────► PRIMARY DB
                                            │
                                            │ ASYNC Replication
                                            │ (eventually consistent)
                          ┌─────────────────┼─────────────────┐
                          ▼                 ▼                 ▼
                    ┌───────────┐    ┌───────────┐    ┌───────────┐
Application ───────►│ Replica 1 │    │ Replica 2 │    │ Replica 3 │
(Reads)      ──────►│ (AZ-a)   │    │ (AZ-b)   │    │ (Region-2)│
             ──────►│           │    │           │    │           │
                    └───────────┘    └───────────┘    └───────────┘
                     Same Region      Same Region      Cross-Region
```

1. Your application writes to the **primary** DB instance.
2. Changes are **asynchronously** replicated to all Read Replicas.
3. Your application reads from the **replica endpoints** — offloading the primary.
4. Because replication is async, there is **replication lag** — reads from replicas are **eventually consistent**.

> [!WARNING]
> **Eventually consistent reads.** A Read Replica may return slightly stale data (milliseconds to seconds behind the primary). If your application requires strict read-after-write consistency, read from the **primary** — not a replica.

---

## Deployment Options

| Placement | Use Case | Data Transfer Cost |
|---|---|---|
| **Same AZ** | Lowest latency reads | **Free** ✅ |
| **Cross-AZ** (same Region) | HA + read scaling | **Free** ✅ |
| **Cross-Region** | Global read latency reduction + DR | **Charged** 💰 |

### Cross-Region Read Replicas

Cross-region replicas are powerful for **disaster recovery** and **global latency reduction**, but come with extra considerations:

| Feature | Detail |
|---|---|
| **Cost** | ~$0.02/GB data transfer |
| **Encryption** | If source uses **default AWS KMS key**, cross-region replica **cannot** be created. Must use a **Customer Managed Key (CMK)** |
| **Replication lag** | Higher than same-region (network latency across regions) |
| **Promotion** | Can be promoted to standalone for DR in the target region |

---

## Limits

| Engine | Max Read Replicas | Notes |
|---|---|---|
| **MySQL** | 5 | Replicas of replicas supported (cascading) |
| **PostgreSQL** | 5 | — |
| **MariaDB** | 5 | Replicas of replicas supported |
| **Oracle** | 5 | Active Data Guard required (paid feature) |
| **SQL Server** | 5 | — |
| **[[Amazon Aurora]]** | **15** | Shared storage — virtually no lag |

---

## Read Replica Promotion

You can **promote** a Read Replica to become a **standalone, independent database** instance:

```
Before Promotion:

PRIMARY ──(async replication)──► READ REPLICA
  R/W                             R only

After Promotion:

PRIMARY                           PROMOTED DB
  R/W    (replication BROKEN)       R/W (independent)
```

### Key Facts
- Promotion is a **one-way, permanent** operation — the replication link is **severed**.
- The promoted replica becomes a **fully independent** read/write DB instance.
- The promoted instance gets a **new endpoint** — you must update your application.
- Use case: **disaster recovery** — if the primary fails, promote a cross-region replica.

---

## Read Replicas + Multi-AZ

You can **combine** both features:

```
                    ┌─────────────────────────────────────┐
                    │      Multi-AZ (High Availability)   │
                    │                                     │
                    │  PRIMARY        STANDBY             │
                    │  (AZ-a)  SYNC   (AZ-b)             │
                    │    R/W  ──────►  passive            │
                    │                                     │
                    └────────┬────────────────────────────┘
                             │
                             │ ASYNC Replication
                             ▼
                    ┌──────────────────┐
                    │  Read Replica    │  ← Can also be Multi-AZ!
                    │  (same or cross  │
                    │   region)        │
                    └──────────────────┘
```

- You can create a Read Replica of a Multi-AZ primary.
- You can also enable **Multi-AZ on the Read Replica itself** — the replica gets its own standby for extra resilience.
- This is a common architecture for **cross-region DR**: Primary (Multi-AZ) → Cross-Region Read Replica (Multi-AZ) → Promote if disaster strikes.

---

## Application Architecture Patterns

### Pattern 1: Reporting Offload

```
┌───────────────┐     Writes     ┌──────────┐
│  Web App      │───────────────►│ PRIMARY  │
│  (OLTP)       │                │  DB      │
└───────────────┘                └────┬─────┘
                                     │ async
┌───────────────┐     Reads      ┌───▼──────┐
│  Reporting    │◄───────────────│ READ     │
│  Dashboard    │                │ REPLICA  │
└───────────────┘                └──────────┘

Benefit: Heavy analytical queries don't slow down the production app.
```

### Pattern 2: Cross-Region DR

```
us-east-1:                        eu-west-1:
┌──────────────┐     async       ┌──────────────┐
│ PRIMARY (MA) │────────────────►│ Read Replica │
│ Multi-AZ     │                 │ (Multi-AZ)   │
└──────────────┘                 └──────────────┘
                                       │
                                 If us-east-1 fails:
                                       │ PROMOTE
                                       ▼
                                 ┌──────────────┐
                                 │ NEW PRIMARY  │
                                 │ (standalone) │
                                 └──────────────┘
```

### Pattern 3: Global Read Latency

```
                    PRIMARY (us-east-1)
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
         ┌────────┐  ┌────────┐  ┌────────┐
         │ Replica│  │ Replica│  │ Replica│
         │eu-west │  │ap-south│  │sa-east │
         └────────┘  └────────┘  └────────┘

Users in Europe, Asia, and South America
read from local replicas → low latency.
```

---

## Monitoring Replication Lag

- Use the **ReplicaLag** CloudWatch metric.
- High replication lag means replicas are falling behind — reads are more stale.
- Causes of high lag: write-heavy primary, under-provisioned replica, network issues.
- **Action:** Scale up the replica instance class or reduce write load on the primary.

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Read Replicas:**
> - "Offload read traffic" → **Read Replicas**
> - "Reporting/analytics on production data without impacting production" → **Read Replica**
> - "Read scaling" or "read-heavy workload" → **Read Replicas**
> - "Reduce latency for global users (reads)" → **Cross-Region Read Replicas**
> - "DR in another region" → **Cross-Region Read Replica + Promotion**
> - "Eventually consistent reads are acceptable" → **Read Replicas**
> - "Strict read-after-write consistency required" → Read from **primary**, not replica
>
> **Key facts for the exam:**
> - Replication: **asynchronous** (eventually consistent)
> - Max replicas: **5** (standard engines), **15** (Aurora)
> - Same-region replication cost: **Free** (even cross-AZ)
> - Cross-region replication cost: **~$0.02/GB**
> - Promotion is **permanent** — breaks replication
> - Cross-region encrypted replica requires **Customer Managed KMS Key** (not default key)
> - Each replica has its **own endpoint** (for standard RDS)
> - Aurora replicas share the **Reader Endpoint** (auto load-balanced)
