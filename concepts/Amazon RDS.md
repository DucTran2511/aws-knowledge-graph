---
tags: [concept, database, high-availability, managed-service]
aliases: [RDS, Relational Database Service, Amazon RDS, AWS RDS]
date: 2026-04-30
---

# Amazon RDS (Relational Database Service)

**Amazon RDS** is a **managed** relational database service that makes it easy to set up, operate, and scale a relational database in the cloud. It removes the undifferentiated heavy lifting of database administration — patching, backups, replication, failover — so you can focus on your application.

## Why Use RDS Over a Database on EC2?

Running your own database on [[EC2]] gives you full OS-level control, but you take on **all** operational burden. RDS trades a small amount of control for massive operational savings.

| Responsibility | DB on EC2 | RDS (Managed) |
|---|---|---|
| **OS Patching** | You | AWS |
| **DB Patching** | You | AWS |
| **Backups** | You | Automated (with PITR) |
| **High Availability** | You design & build | One checkbox (Multi-AZ) |
| **Horizontal Read Scaling** | You design & build | Read Replicas |
| **Storage Scaling** | You manage EBS | Storage Auto Scaling |
| **Monitoring** | You install tools | CloudWatch integration built-in |
| **SSH Access** | ✅ Yes | ❌ No |

> [!WARNING]
> **You cannot SSH into RDS instances.** This is the trade-off for a fully managed service. If a question mentions needing OS-level access or custom database engine configurations not supported by RDS, the answer is a self-managed DB on [[EC2]].

---

## Supported Database Engines

RDS supports **six** database engines:

| Engine | Type | Notes |
|---|---|---|
| **PostgreSQL** | Open Source | Exam favorite alongside MySQL |
| **MySQL** | Open Source | Most popular open-source RDBMS |
| **MariaDB** | Open Source | MySQL fork, community-driven |
| **Oracle** | Commercial | Bring Your Own License (BYOL) or License Included |
| **Microsoft SQL Server** | Commercial | License Included or BYOL |
| **[[Amazon Aurora]]** | AWS Proprietary | Cloud-native, MySQL/PostgreSQL compatible — separate deep-dive |

> [!TIP]
> **Exam Pattern:** If a question says "cloud-native" or "5x MySQL performance" or "3x PostgreSQL performance," the answer is **[[Amazon Aurora]]**, not standard RDS.

---

## RDS Storage

RDS uses [[Elastic Block Store (EBS) Volumes|EBS]] under the hood. You choose from three storage types:

| Storage Type | IOPS | Use Case |
|---|---|---|
| **gp2 / gp3** (General Purpose SSD) | Up to 16,000 IOPS | Most workloads |
| **io1 / io2** (Provisioned IOPS SSD) | Up to 256,000 IOPS | I/O-intensive, latency-sensitive (OLTP) |
| **Magnetic** (previous gen) | Low | Legacy, not recommended |

### Storage Auto Scaling

RDS can **automatically increase storage** when it detects you're running out of free space. You set a **Maximum Storage Threshold** and RDS handles the rest.

**Auto Scaling triggers when ALL of these are true:**
- Free storage is < 10% of allocated storage
- Low-storage condition lasts at least 5 minutes
- 6 hours have passed since the last storage modification

```
Storage Auto Scaling:

Allocated ▲
          │                    ┌─────── Maximum Storage Threshold
          │                    │
  100 GB  │████████████████████│
          │█████████████████   │  ← Auto-scaled (free space was < 10%)
   80 GB  │████████████████    │
          │██████████████      │
   50 GB  │█████████████       │  ← Initially provisioned
          │                    │
          └────────────────────────► Time
```

> [!NOTE]
> Storage can only scale **up**, never down. You cannot reduce allocated storage on an existing RDS instance.

---

## RDS Read Replicas

*Main article: [[RDS Read Replicas]]*

Read Replicas provide **horizontal read scaling** by creating **asynchronous**, read-only copies of your primary database.

### Key Facts

- Up to **15 Read Replicas** (Aurora) or **5 Read Replicas** (other engines).
- Replicas can be in the **same AZ**, **cross-AZ**, or **cross-Region**.
- Replication is **ASYNC** → reads are **eventually consistent**.
- Replicas can be **promoted** to a standalone database (breaks replication permanently).
- Each replica has its **own DNS endpoint**.

```
                    ┌────────────────────┐
                    │   Primary DB       │
                    │   (Reads + Writes) │
                    └────────┬───────────┘
                             │ ASYNC Replication
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  Read Replica │ │ Read Replica │ │ Read Replica │
    │  (Same AZ)   │ │ (Cross-AZ)   │ │ (Cross-Region)│
    │  Reads Only  │ │  Reads Only  │ │  Reads Only   │
    └──────────────┘ └──────────────┘ └──────────────┘
```

### Use Cases
1. **Reporting / Analytics** — Offload heavy queries to a replica so the primary isn't impacted.
2. **Read-heavy applications** — Distribute read traffic across replicas.
3. **Cross-Region DR** — Promote a cross-region replica if the primary region fails.

### Network Cost (Exam Favorite!)

| Scenario | Data Transfer Cost |
|---|---|
| Replica in **same Region** (same or different AZ) | **Free** ✅ |
| Replica in **different Region** | **Charged** 💰 (~$0.02/GB) |

> [!IMPORTANT]
> **Exam trap:** Same-region Read Replica replication is **free** (even cross-AZ). Cross-region replication incurs data transfer costs. This is a heavily tested fact.

### Read Replica vs Multi-AZ

| Feature | Read Replica | Multi-AZ |
|---|---|---|
| **Purpose** | **Read scaling** (performance) | **High availability** (disaster recovery) |
| **Replication** | **Asynchronous** | **Synchronous** |
| **Accessible?** | ✅ Yes — serve read traffic | ❌ No — standby is passive (except Multi-AZ Cluster) |
| **Automatic Failover?** | ❌ No — manual promotion | ✅ Yes — automatic DNS failover |
| **Cross-Region?** | ✅ Yes | ❌ No (same Region, different AZ) |

---

## RDS Multi-AZ Deployments

Multi-AZ provides **high availability** and **automatic failover** for your RDS database.

### Multi-AZ DB Instance (Classic)

The traditional HA deployment:

1. RDS creates a **standby replica** in a different [[Availability Zones (AZ)|AZ]].
2. Replication is **synchronous** — every write to the primary is immediately replicated.
3. The standby is **NOT accessible** — no reads, no writes. It's purely a failover target.
4. Failover is **automatic** via DNS. Your application connects via a single DNS endpoint that automatically flips to the standby.

```
                  ┌─────────────────────────────────┐
                  │           Single DNS Endpoint    │
                  │         (db.xxxx.rds.amazonaws.com) │
                  └──────────────┬──────────────────┘
                                 │
                     ┌───────────┴───────────┐
                     ▼                       ▼
              ┌─────────────┐         ┌─────────────┐
              │  PRIMARY    │  SYNC   │  STANDBY    │
              │  (AZ-a)     │────────►│  (AZ-b)     │
              │  R/W        │         │  NOT        │
              │             │         │  ACCESSIBLE │
              └─────────────┘         └─────────────┘
```

**Failover triggers:**
- Primary DB instance failure
- AZ outage
- DB instance type change
- OS patching on the primary
- Manual failover (via "Reboot with failover")

**Failover time:** ~60 seconds.

### Multi-AZ DB Cluster (Newer)

A more modern deployment option for MySQL and PostgreSQL:

| Feature | Multi-AZ Instance | Multi-AZ Cluster |
|---|---|---|
| **Architecture** | 1 Primary + 1 Standby | 1 Writer + 2 Readable Standbys |
| **AZs** | 2 | 3 |
| **Standbys readable?** | ❌ No | ✅ Yes (can serve reads) |
| **Replication** | Synchronous | Semi-synchronous |
| **Failover time** | ~60 seconds | ~35 seconds |
| **Write latency** | Higher | Lower (up to 2x faster commits) |

> [!TIP]
> **Exam Pattern:** Multi-AZ is about **availability**, NOT read scaling. If a question asks about read performance, the answer is Read Replicas (or Multi-AZ Cluster for its readable standbys). If it asks about HA/DR with automatic failover, the answer is Multi-AZ.

### Going from Single-AZ to Multi-AZ

This is a **zero-downtime** operation:
1. Click "Modify" → enable Multi-AZ.
2. Internally: RDS takes a snapshot → restores it to a new standby in another AZ → establishes sync replication.
3. **No downtime, no application changes needed.**

---

## RDS Backups

### Automated Backups

- **Enabled by default.**
- Daily **full snapshot** during a configurable backup window.
- **Transaction logs** are backed up every 5 minutes.
- Together, these enable **Point-in-Time Recovery (PITR)** — restore to any second within the retention period.
- Retention period: **1 to 35 days** (set to 0 to disable, but not recommended).
- Automated backups are **deleted** when you delete the RDS instance (unless you take a final snapshot).

```
Backup Timeline:

Day 1        Day 2        Day 3        Day 4
──┬────────────┬────────────┬────────────┬──
  │Full Snap   │Full Snap   │Full Snap   │
  │ + tx logs  │ + tx logs  │ + tx logs  │
  │  every 5m  │  every 5m  │  every 5m  │
  │            │            │            │
  └── Retention Window (e.g., 7 days) ──────►
                                    ↑
                         Can restore to any second
                         within this window (PITR)
```

### Manual DB Snapshots

- **User-initiated** — you trigger them manually or via automation.
- Retained **indefinitely** until you explicitly delete them.
- Persist **even after the RDS instance is deleted**.
- Restoring a snapshot creates a **new** RDS instance with a new endpoint.

> [!TIP]
> **Cost trick:** If you have an RDS instance you only use for a few hours a week, you can **snapshot it → delete the instance → restore from snapshot** when needed. You'll only pay for snapshot storage (much cheaper than running an idle instance).

### Restore Behavior

- Restoring from either automated backup or manual snapshot creates a **brand-new RDS instance** with a **new endpoint**.
- You must update your application connection string to point to the new instance.

---

## RDS Security

### Encryption at Rest

| Feature | Detail |
|---|---|
| **Engine** | AWS KMS (AES-256) |
| **When to enable** | Must be set **at creation time** |
| **Master + Replicas** | If primary is encrypted, all replicas are encrypted |
| **Unencrypted → Encrypted** | Cannot directly encrypt. Must: snapshot → copy snapshot with encryption → restore from encrypted snapshot |

### Encryption in Transit

- Use **SSL/TLS** certificates to encrypt connections between your app and RDS.
- Enforce SSL by setting the `rds.force_ssl` parameter in the DB parameter group.

### Network Security

- Deploy RDS in a **private subnet** (no public internet access).
- Control access with **Security Groups** (just like [[EC2]]).
- Common pattern: Allow inbound traffic **only from your application's Security Group**.

### IAM Database Authentication

Instead of traditional username/password, you can authenticate to RDS using **IAM roles + authentication tokens**.

- Supported for **MySQL** and **PostgreSQL**.
- The token is obtained via the RDS API and is valid for **15 minutes**.
- Benefits: centralized access management via IAM, no passwords stored in the application.

```
┌──────────┐    1. GetAuthToken()    ┌──────────┐
│          │◄────────────────────────│          │
│  RDS API │                        │  EC2 App │
│          │────────────────────────►│  (IAM    │
└──────────┘    2. Returns token     │   Role)  │
                                    │          │
                                    └────┬─────┘
                                         │ 3. Connect with
                                         │    token + SSL
                                    ┌────▼─────┐
                                    │   RDS    │
                                    │  MySQL / │
                                    │  PostgreSQL│
                                    └──────────┘
```

> [!TIP]
> **Exam Pattern:** "Authenticate to database without storing passwords" or "centralize DB access management" → **IAM Database Authentication**.

---

## RDS Proxy

**RDS Proxy** is a fully managed, highly available database proxy that sits between your application and your RDS database.

### Why RDS Proxy?

The core problem: databases have a **limited number of connections**. Applications that open/close connections rapidly (like [[Lambda]] functions) can exhaust the connection pool.

```
Without RDS Proxy:                        With RDS Proxy:

┌──────┐                                  ┌──────┐
│ λ #1 │──┐                               │ λ #1 │──┐
└──────┘  │                               └──────┘  │
┌──────┐  │   Hundreds of connections     ┌──────┐  │   Few pooled
│ λ #2 │──┼──────────────────► RDS        │ λ #2 │──┼───► RDS ───► RDS
└──────┘  │   (connection exhaustion!)    └──────┘  │    Proxy     DB
┌──────┐  │                               ┌──────┐  │   (connection
│ λ #3 │──┘                               │ λ #3 │──┘    pooling)
└──────┘                                  └──────┘
  ...                                       ...
┌──────┐                                  ┌──────┐
│ λ #N │──┘                               │ λ #N │──┘
└──────┘                                  └──────┘
```

### Key Features

| Feature | Detail |
|---|---|
| **Connection Pooling** | Shares and reuses DB connections across many app connections |
| **Reduced Failover Time** | Up to **66% reduction** in failover time for Multi-AZ |
| **IAM Authentication** | Enforce IAM auth — no DB credentials in application code |
| **Secrets Manager** | Stores credentials securely in [[Secrets Manager]] |
| **Supports** | MySQL, PostgreSQL, MariaDB, SQL Server |
| **Deployment** | Runs in your [[VPC]] — **never publicly accessible** |
| **Lambda Integration** | Perfect for serverless — solves the connection exhaustion problem |

> [!TIP]
> **Exam Pattern:** "Lambda functions connecting to RDS" + "too many connections" or "connection timeout" → **RDS Proxy**. Also, "reduce Multi-AZ failover time" → **RDS Proxy**.

---

## RDS Custom

**RDS Custom** gives you access to the **underlying OS and database** while still benefiting from RDS automation. It's a middle ground between full RDS and a self-managed DB on EC2.

- Available for **Oracle** and **SQL Server** only.
- You can SSH, install patches, enable native features, and access the underlying OS.
- RDS automation (backups, HA, scaling) still works, but you can pause it when doing customizations.

> [!NOTE]
> **Exam Pattern:** "Need OS-level access" + "Oracle or SQL Server" + "still want managed backups" → **RDS Custom**.

---

## Storage Auto Scaling — Summary

| Parameter | Value |
|---|---|
| **Trigger** | Free storage < 10% for 5+ min AND 6+ hours since last modification |
| **Action** | Automatically increases storage |
| **Maximum** | You set a **Maximum Storage Threshold** |
| **Direction** | Scale **up** only (never down) |
| **Downtime** | None |
| **Supported** | All RDS engines |

---

## RDS Events & Notifications

- RDS produces **events** for DB instance changes (failover, snapshot, parameter change, etc.).
- Subscribe to events via **Amazon SNS** topics.
- Use **Amazon EventBridge** for more complex event-driven architectures.

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → RDS:**
> - "Managed relational database" → **RDS**
> - "Automatic failover, high availability" → **RDS Multi-AZ**
> - "Read scaling, reporting queries" → **RDS Read Replicas**
> - "Lambda + too many DB connections" → **RDS Proxy**
> - "No password, IAM-based DB auth" → **IAM Database Authentication**
> - "Encrypt an unencrypted database" → Snapshot → Copy with encryption → Restore
> - "Reduce failover time" → **RDS Proxy** (66% faster)
> - "OS-level access + managed Oracle/SQL Server" → **RDS Custom**
> - "Cloud-native, 5x MySQL, 3x PostgreSQL" → **[[Amazon Aurora]]** (not standard RDS)
> - "Restore to a specific point in time" → **PITR** (automated backups)
> - "Database storage growing unpredictably" → **Storage Auto Scaling**
>
> **Key defaults:**
> - Automated backup retention: **7 days** (configurable 0–35)
> - Storage Auto Scaling: free space < **10%** for **5 min**, **6 hours** between modifications
> - Multi-AZ Instance failover: **~60 seconds**
> - Multi-AZ Cluster failover: **~35 seconds**
> - IAM Auth token validity: **15 minutes**
> - Read Replicas: up to **5** (standard) or **15** (Aurora)
> - Same-region replica replication cost: **Free**
> - Cross-region replica replication cost: **~$0.02/GB**
