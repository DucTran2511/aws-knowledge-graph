---
tags: [concept, database, high-availability, managed-service, aurora]
aliases: [Aurora, Amazon Aurora, Aurora MySQL, Aurora PostgreSQL, Aurora Serverless, Aurora Global Database]
date: 2026-04-30
---

# Amazon Aurora

**Amazon Aurora** is a **cloud-native**, fully managed relational database engine built by AWS. It is compatible with **MySQL** and **PostgreSQL** but re-engineered from the ground up to take full advantage of the cloud. Aurora is part of [[Amazon RDS]] — you manage it through the same RDS console, APIs, and CLI.

Aurora is an **exam heavyweight** — expect multiple questions on its architecture, replication, and unique features.

---

## Why Aurora?

Aurora separates **compute** from **storage** and re-architects both for the cloud:

- **5x throughput** of standard MySQL on RDS.
- **3x throughput** of standard PostgreSQL on RDS.
- Storage **automatically grows** from 10 GB up to **128 TB** — no provisioning needed.
- Up to **15 Read Replicas** with sub-10ms replication lag.
- **Instantaneous failover** — HA is built into the architecture, not bolted on.
- Costs ~20% more than standard RDS, but significantly more efficient.

---

## Aurora Storage Architecture

This is the key differentiator. Aurora's storage is a **distributed, fault-tolerant, self-healing** storage system that is completely separate from the compute instances.

### 6-Way Replication Across 3 AZs

Data is automatically replicated **6 copies** across **3 [[Availability Zones (AZ)|Availability Zones]]** (2 copies per AZ).

```
                    ┌──────────────────────────────────────────────┐
                    │          Aurora Shared Storage Volume         │
                    │        (Distributed across 3 AZs)            │
                    │                                              │
                    │  AZ-a          AZ-b          AZ-c            │
                    │  ┌────┐       ┌────┐       ┌────┐           │
                    │  │Copy│       │Copy│       │Copy│           │
                    │  │ 1  │       │ 3  │       │ 5  │           │
                    │  └────┘       └────┘       └────┘           │
                    │  ┌────┐       ┌────┐       ┌────┐           │
                    │  │Copy│       │Copy│       │Copy│           │
                    │  │ 2  │       │ 4  │       │ 6  │           │
                    │  └────┘       └────┘       └────┘           │
                    │                                              │
                    │  • Needs 4/6 copies for WRITES              │
                    │  • Needs 3/6 copies for READS               │
                    │  • Self-healing with peer-to-peer replication│
                    │  • Storage striped across 100s of volumes    │
                    └──────────────────────────────────────────────┘
```

### Fault Tolerance

| Scenario | Impact |
|---|---|
| **Lose 1 copy** | No impact (still 5/6 — writes and reads work) |
| **Lose 2 copies** | Writes still work (4/6), reads still work (4/6) |
| **Lose an entire AZ** | Still have 4 copies across 2 AZs — fully operational |
| **Lose 3 copies** | Reads still work (3/6), writes pause until recovery |

> [!NOTE]
> Aurora storage is **self-healing**. If a copy is lost, Aurora automatically repairs it using the remaining copies — no manual intervention, no operational burden.

---

## Aurora Cluster Architecture

```
                    ┌────────────────────────────────────────┐
                    │            Aurora Cluster               │
                    │                                        │
                    │  ┌──────────────┐                      │
                    │  │  Writer      │ ← Cluster Endpoint   │
                    │  │  Instance    │   (DNS for writes)   │
                    │  └──────┬───────┘                      │
                    │         │                              │
                    │         │  Shared Storage Volume       │
                    │         │  (6 copies / 3 AZs)         │
                    │         │                              │
                    │  ┌──────┴──────────────────────┐       │
                    │  │     Read Replicas (up to 15) │       │
                    │  │  ┌────┐ ┌────┐ ┌────┐       │       │
                    │  │  │ R1 │ │ R2 │ │ R3 │ ...   │ ← Reader Endpoint │
                    │  │  └────┘ └────┘ └────┘       │   (DNS, auto LB)  │
                    │  └─────────────────────────────┘       │
                    └────────────────────────────────────────┘
```

### Endpoints

| Endpoint | Purpose | Notes |
|---|---|---|
| **Cluster Endpoint** (Writer) | Points to the **primary writer** instance | Always connects to the current writer, even after failover |
| **Reader Endpoint** | Points to **read replicas** with connection load balancing | Automatically distributes read connections across replicas |
| **Custom Endpoint** | Points to a **subset** of instances you define | Use case: route analytical queries to larger instance types |
| **Instance Endpoint** | Points to a **specific instance** | For debugging or specific routing needs |

> [!TIP]
> **Exam Pattern:** "How does the application find the writer after failover?" → The **Cluster Endpoint** automatically updates. Applications use the cluster endpoint and never need to change connection strings.

---

## Aurora Replicas vs RDS Read Replicas

| Feature | Aurora Replica | RDS Read Replica |
|---|---|---|
| **Max count** | **15** | **5** |
| **Replication type** | Shared storage (virtually instant) | Async from primary (network-based) |
| **Replication lag** | Typically < **10 ms** | Can be **seconds to minutes** |
| **Failover target?** | ✅ Yes — automatic failover | ❌ No — manual promotion |
| **Impact on primary** | None (reads from shared storage) | Some (replication uses primary resources) |
| **Cross-Region?** | Via Aurora Global Database | ✅ Yes |
| **Endpoint** | Reader Endpoint (auto-LB) | Each replica has its own endpoint |

---

## Aurora Auto Scaling

Aurora can automatically **add or remove Read Replicas** based on demand:

- Define a **scaling policy** (e.g., target 70% CPU utilization across replicas).
- Aurora adds replicas when load increases, removes them when load decreases.
- Replicas are automatically registered with the **Reader Endpoint**.

---

## Aurora Serverless

**Aurora Serverless** automatically starts up, shuts down, and scales capacity **up or down** based on your application's needs. You don't manage any database instances.

### Aurora Serverless v2 (Current)

| Feature | Detail |
|---|---|
| **Scaling** | Instant, fine-grained (scales in increments of **0.5 ACUs**) |
| **Range** | Configure min and max **Aurora Capacity Units (ACUs)** |
| **Compatible** | Works with all Aurora features (Global DB, cloning, Multi-AZ) |
| **Use cases** | Unpredictable workloads, dev/test, multi-tenant SaaS |
| **Pricing** | Pay per ACU-second |

```
ACU ▲
    │           ╱╲
    │          ╱  ╲         ╱╲
    │    ╱╲   ╱    ╲       ╱  ╲
    │   ╱  ╲ ╱      ╲     ╱    ╲
    │  ╱    ╲╱        ╲   ╱      ╲
    │ ╱                ╲ ╱        ╲
    │╱                  ╲╱         ╲──── Min ACU
    └───────────────────────────────► Time
    
    Capacity follows demand automatically
    Pay only for what you use
```

> [!TIP]
> **Exam Pattern:** "Unpredictable/intermittent/infrequent workload" + "database" + "cost-effective" → **Aurora Serverless**. Also good for "dev/test environments" and "multi-tenant applications."

---

## Aurora Global Database

Designed for **globally distributed applications** requiring cross-region disaster recovery and low-latency reads worldwide.

### How It Works

- **1 Primary Region** — handles all writes.
- Up to **5 Secondary Regions** — read-only, with up to 16 Read Replicas each.
- Cross-region replication lag: typically **< 1 second**.
- **Promoting** a secondary region to primary: **RTO < 1 minute**.

```
                    ┌───────────────────────────┐
                    │  PRIMARY REGION (us-east-1)│
                    │  ┌─────────┐              │
                    │  │ Writer  │  + Replicas  │
                    │  └────┬────┘              │
                    └───────┼──────────────────┘
                            │ Replication (< 1 sec)
              ┌─────────────┼─────────────┐
              ▼                           ▼
┌──────────────────────┐    ┌──────────────────────┐
│ SECONDARY (eu-west-1)│    │ SECONDARY (ap-se-1)  │
│ Read-only cluster    │    │ Read-only cluster     │
│ (up to 16 replicas)  │    │ (up to 16 replicas)   │
└──────────────────────┘    └──────────────────────┘
```

> [!TIP]
> **Exam Pattern:** "Cross-region disaster recovery" + "RTO < 1 minute" + "global read scaling" → **Aurora Global Database**.

---

## Aurora Cloning

Aurora can create a **clone** of your production database in minutes, regardless of database size.

### How It Works — Copy-on-Write

1. The clone initially **shares the same storage pages** as the source.
2. Data pages are only copied when **modified** in either the source or clone.
3. Result: cloning is nearly **instant** and costs almost nothing until you start writing.

### Use Cases
- Create a **staging** or **test** environment from production data without impacting production.
- Run **analytical queries** on a clone without affecting the live database.

### Limitations
- Same Region only.
- Max ~15 clones per source.

> [!NOTE]
> Cloning is **much faster and cheaper** than snapshot → restore. Use cloning when you need a quick copy in the same Region. Use snapshots for cross-Region copies.

---

## Aurora Machine Learning

Aurora can integrate with AWS ML services directly via SQL:

- **Amazon SageMaker** — use any ML model.
- **Amazon Comprehend** — sentiment analysis.

You call ML predictions as **SQL functions** — no ML expertise required.

Use case: fraud detection, ad targeting, sentiment analysis, product recommendations — all from within your SQL queries.

---

## Aurora Backups

Same as [[Amazon RDS]] backups, with enhancements:

| Feature | Aurora | Standard RDS |
|---|---|---|
| **Automated backups** | Always enabled, cannot be disabled | Can be disabled (retention = 0) |
| **Retention** | 1–35 days | 1–35 days |
| **PITR** | ✅ | ✅ |
| **Backtrack** | ✅ Rewind DB to a point in time **in-place** (no new instance) | ❌ Not available |

### Aurora Backtrack

Unlike PITR (which creates a new instance), **Backtrack** rewinds the existing database to a previous point in time **in-place** — no new instance, no endpoint change.

- Configurable backtrack window (up to **72 hours**).
- Must be enabled at cluster creation time.

---

## Aurora Security

Same security model as [[Amazon RDS]]:
- **Encryption at rest**: KMS (must be enabled at creation).
- **Encryption in transit**: SSL/TLS.
- **IAM Authentication**: Token-based auth for MySQL & PostgreSQL.
- **Security Groups**: Network-level access control.
- **No SSH access** — fully managed.
- **Audit logs**: Can be sent to CloudWatch Logs.

---

## RDS vs Aurora — Decision Table

| Requirement | RDS | Aurora |
|---|---|---|
| **MySQL/PostgreSQL compatible** | ✅ | ✅ |
| **Oracle, SQL Server, MariaDB** | ✅ | ❌ |
| **Max Read Replicas** | 5 | 15 |
| **Storage auto-grow** | Manual threshold | Auto to 128 TB |
| **6-way replication** | ❌ | ✅ |
| **Instant failover** | ~60 sec | ~30 sec |
| **Serverless option** | ❌ | ✅ |
| **Global Database** | Cross-Region Read Replicas | ✅ (< 1 sec replication) |
| **Backtrack** | ❌ | ✅ |
| **Cost** | Lower | ~20% higher |
| **Cloning** | ❌ | ✅ |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Aurora:**
> - "5x MySQL / 3x PostgreSQL performance" → **Aurora**
> - "Cloud-native database" → **Aurora**
> - "15 Read Replicas" → **Aurora**
> - "Sub-millisecond replication lag" → **Aurora** (shared storage)
> - "Storage auto-grows to 128 TB" → **Aurora**
> - "Unpredictable workload + database" → **Aurora Serverless**
> - "Global DR + RTO < 1 minute" → **Aurora Global Database**
> - "Rewind database in-place" → **Aurora Backtrack**
> - "Clone production for testing" → **Aurora Cloning**
> - "ML predictions in SQL queries" → **Aurora Machine Learning**
> - "Cross-region read scaling" → **Aurora Global Database**
>
> **Key numbers:**
> - Storage: auto-grows **10 GB → 128 TB**
> - Replication: **6 copies** across **3 AZs**
> - Write quorum: **4/6**, Read quorum: **3/6**
> - Read Replicas: up to **15**
> - Replication lag: < **10 ms** (typically < 1 ms)
> - Global DB replication lag: < **1 second**
> - Global DB promotion: RTO < **1 minute**
> - Backtrack window: up to **72 hours**
> - Secondary Regions (Global DB): up to **5**
