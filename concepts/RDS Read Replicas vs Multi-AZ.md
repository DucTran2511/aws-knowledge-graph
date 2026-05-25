---
tags: [concept, database, high-availability, scaling, exam-critical]
aliases: [Read Replica vs Multi-AZ, Multi-AZ vs Read Replica, RDS HA vs Scaling]
date: 2026-04-30
---

# RDS Read Replicas vs Multi-AZ

This is one of the **most frequently tested** topics on the SAA-C03 exam. AWS wants you to know exactly when to use each — and they will try to trick you.

**One sentence to remember:**
> **Multi-AZ = Availability (standby).** **Read Replicas = Scalability (read traffic).**

---

## Side-by-Side Comparison

| Feature | Read Replica | Multi-AZ (Instance) | Multi-AZ (Cluster) |
|---|---|---|---|
| **Purpose** | **Read scaling** (performance) | **High availability** (DR) | **HA + read scaling** |
| **Replication** | **Asynchronous** | **Synchronous** | **Semi-synchronous** |
| **Data consistency** | **Eventually consistent** | Always in sync | Nearly in sync |
| **Readable?** | ✅ Yes — serves read traffic | ❌ No — standby is passive | ✅ Yes — standbys serve reads |
| **Writable?** | ❌ Read-only | Only primary | Only writer |
| **Automatic failover?** | ❌ No — manual promotion | ✅ Yes — DNS flip (~60s) | ✅ Yes — DNS flip (~35s) |
| **Cross-Region?** | ✅ Yes | ❌ No (same Region only) | ❌ No |
| **Max count** | 5 (RDS) / 15 ([[Amazon Aurora\|Aurora]]) | 1 standby | 2 readable standbys |
| **AZs involved** | Any | 2 AZs | 3 AZs |
| **DNS endpoint** | Each replica has its **own** endpoint | **Single** endpoint (auto-flips) | Cluster + Reader endpoints |
| **Network cost** | Same-Region: **Free** / Cross-Region: **Paid** | **Free** (same-Region sync) | **Free** |
| **Can be promoted?** | ✅ Yes (becomes standalone R/W) | N/A | N/A |
| **Use case** | Reporting, analytics, read-heavy apps | Production HA, planned maintenance | HA + moderate read scaling |

---

## Visual Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         READ REPLICAS                               │
│                     (Horizontal Read Scaling)                        │
│                                                                     │
│   App (writes) ─────────────────► PRIMARY DB                        │
│                                      │                              │
│                                      │  ASYNC (eventually consistent)│
│                          ┌───────────┼───────────┐                  │
│                          ▼           ▼           ▼                  │
│   App (reads) ──────► Replica 1  Replica 2  Replica 3              │
│                        (AZ-a)     (AZ-b)    (Region-2)             │
│                                                                     │
│   • Each replica has its OWN endpoint                               │
│   • Reads may be slightly stale                                     │
│   • Can promote any replica to standalone                           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                         MULTI-AZ (Instance)                         │
│                      (High Availability / DR)                       │
│                                                                     │
│                     Single DNS Endpoint                             │
│                  (db.xxxx.rds.amazonaws.com)                        │
│                          │                                          │
│              ┌───────────┴───────────┐                              │
│              ▼                       ▼                              │
│        ┌───────────┐          ┌───────────┐                        │
│        │  PRIMARY  │  SYNC    │  STANDBY  │                        │
│        │  (AZ-a)   │────────►│  (AZ-b)   │                        │
│        │  R/W      │          │  NO ACCESS│                        │
│        └───────────┘          └───────────┘                        │
│                                                                     │
│   • ONE endpoint — automatically flips on failover                  │
│   • Standby is INVISIBLE to your application                        │
│   • Zero data loss (synchronous)                                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                       MULTI-AZ (Cluster)                            │
│                    (HA + Read Scaling Combined)                      │
│                                                                     │
│              Writer Endpoint        Reader Endpoint                 │
│                    │                      │                          │
│                    ▼                      ▼                          │
│             ┌───────────┐    ┌───────────────────────┐             │
│             │  WRITER   │    │ READABLE   READABLE   │             │
│             │  (AZ-a)   │    │ STANDBY    STANDBY    │             │
│             │  R/W      │    │ (AZ-b)     (AZ-c)     │             │
│             └─────┬─────┘    └─────────────────────┘               │
│                   │  Semi-sync         ▲                            │
│                   └────────────────────┘                            │
│                                                                     │
│   • TWO endpoints: Writer + Reader                                  │
│   • Standbys CAN serve read traffic                                 │
│   • Faster failover (~35s vs ~60s)                                  │
│   • MySQL & PostgreSQL only                                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## The Key Distinction (Exam Gold)

```
Question asks about...                     Answer
─────────────────────────────────────────────────────
"Disaster recovery"                    →   Multi-AZ
"Automatic failover"                   →   Multi-AZ
"Minimize downtime during failure"     →   Multi-AZ
"Reduce read latency"                  →   Read Replica
"Offload reporting queries"            →   Read Replica
"Scale read-heavy workload"            →   Read Replica
"Cross-region reads"                   →   Read Replica (cross-region)
"Cross-region DR"                      →   Read Replica (promote on failure)
"HA + some read scaling"               →   Multi-AZ Cluster
```

---

## Can You Use Both? — YES!

This is the **best-practice production architecture** and a common correct exam answer.

```
                ┌──────────────────────────────────┐
                │   Multi-AZ (High Availability)    │
                │                                   │
                │  PRIMARY          STANDBY          │
                │  (AZ-a)   SYNC   (AZ-b)          │
                │   R/W    ──────►  passive          │
                │                                   │
                └────────────┬─────────────────────┘
                             │
                             │ ASYNC
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
    ┌───────────┐     ┌───────────┐     ┌───────────┐
    │ Replica 1 │     │ Replica 2 │     │ Replica 3 │
    │ (AZ-a)    │     │ (AZ-b)    │     │ (eu-west) │
    │ Reads     │     │ Reads     │     │ Reads + DR│
    └───────────┘     └───────────┘     └───────────┘
     Same-Region       Same-Region       Cross-Region
     (free)            (free)            (paid)

    HA ✅ (Multi-AZ covers failover)
    Read scaling ✅ (Replicas serve reads)
    Cross-region DR ✅ (Promote Replica 3 if region fails)
```

---

## Common Exam Traps

> [!CAUTION]
> ### Trap 1: "Multi-AZ for read scaling"
> ❌ **Wrong.** The Multi-AZ standby (Instance mode) is **not readable**. It sits idle, waiting for failover. For read scaling, you need **Read Replicas**.
>
> *Exception:* Multi-AZ **Cluster** standbys *can* serve reads — but this is a newer feature and the exam usually means the classic Instance mode.

> [!CAUTION]
> ### Trap 2: "Read Replica for automatic failover"
> ❌ **Wrong.** Read Replicas do **not** have automatic failover. Promoting a replica is a **manual** action and **permanently breaks** the replication link. For automatic failover, you need **Multi-AZ**.

> [!CAUTION]
> ### Trap 3: "Read Replica provides consistent reads"
> ❌ **Wrong.** Because replication is **asynchronous**, Read Replicas are **eventually consistent**. If your app needs read-after-write consistency, it must read from the **primary**.

> [!CAUTION]
> ### Trap 4: "Multi-AZ provides cross-region DR"
> ❌ **Wrong.** Multi-AZ only works within **one Region** (different AZs). For cross-region DR, use a **Cross-Region Read Replica** and promote it if the primary region fails. Or use **[[Amazon Aurora#Aurora Global Database|Aurora Global Database]]**.

> [!CAUTION]
> ### Trap 5: "Convert a Read Replica to Multi-AZ standby"
> ❌ **Not how it works.** A Read Replica is an independent copy you promote. A Multi-AZ standby is an internal mechanism managed by AWS. They are completely separate features — but can be **combined** on the same primary.

---

## Replication Deep Dive

### Synchronous (Multi-AZ)

```
Timeline:
t=0ms   App sends WRITE ──────────────► PRIMARY
t=1ms   PRIMARY writes to local storage
t=2ms   PRIMARY sends change to STANDBY ──────► STANDBY
t=3ms   STANDBY writes to its storage
t=4ms   STANDBY acknowledges ──────────────────► PRIMARY
t=5ms   PRIMARY confirms WRITE to App ◄──────── 

✅ Zero data loss — both copies are identical
❌ Slightly higher write latency (must wait for standby ACK)
```

### Asynchronous (Read Replica)

```
Timeline:
t=0ms   App sends WRITE ──────────────► PRIMARY
t=1ms   PRIMARY writes to local storage
t=2ms   PRIMARY confirms WRITE to App ◄──────── (done! fast!)
t=3ms   PRIMARY sends change to REPLICA ──────► (background)
t=???   REPLICA eventually applies change

✅ No write latency penalty
❌ Replica may lag behind (replication lag)
❌ If primary fails before replication, data can be lost on replica
```

### Semi-Synchronous (Multi-AZ Cluster)

```
Timeline:
t=0ms   App sends WRITE ──────────────► WRITER
t=1ms   WRITER writes to local storage
t=2ms   WRITER sends change to STANDBY-1 and STANDBY-2
t=3ms   STANDBY-1 acknowledges ────────► WRITER (at least ONE ACK)
t=4ms   WRITER confirms WRITE to App ◄──────── 
t=???   STANDBY-2 applies change eventually

✅ Faster than full sync (only waits for 1 of 2 standbys)
✅ Near-zero data loss
✅ Standbys are readable
```

---

## Network Cost Summary

| Scenario | Cost |
|---|---|
| Multi-AZ replication (same Region) | **Free** ✅ |
| Read Replica — **same AZ** | **Free** ✅ |
| Read Replica — **cross-AZ** (same Region) | **Free** ✅ |
| Read Replica — **cross-Region** | **~$0.02/GB** 💰 |

> [!IMPORTANT]
> Same-region Read Replica replication is **free**, even across AZs. This is a frequently tested fact. Cross-region is where costs appear.

---

## When to Use What — Decision Flowchart

```
Do you need automatic failover if the primary fails?
├── YES → Multi-AZ ✅
│         └── Also need read scaling?
│             ├── YES → Multi-AZ + Read Replicas (combine both)
│             └── NO  → Multi-AZ alone is sufficient
│
└── NO  → Do you need to scale reads?
          ├── YES → Read Replicas ✅
          │         └── Need cross-region DR too?
          │             ├── YES → Cross-Region Read Replica (promote if disaster)
          │             └── NO  → Same-Region Read Replica
          │
          └── NO  → Single instance (dev/test only)
```

---

## Quick Quiz (Self-Test)

1. Your production database needs to survive an AZ failure automatically. **→ Multi-AZ**
2. Your reporting team's heavy queries are slowing down the production app. **→ Read Replica**
3. You need a database copy in another region for disaster recovery. **→ Cross-Region Read Replica** (or [[Amazon Aurora#Aurora Global Database|Aurora Global Database]])
4. You want both HA and read scaling. **→ Multi-AZ + Read Replicas**
5. A Read Replica is promoted. Does replication continue? **→ No, it's permanent — the replica becomes standalone.**
6. Can you read from a Multi-AZ standby? **→ No** (Instance mode) / **Yes** (Cluster mode)
7. What's the failover time for Multi-AZ Instance? **→ ~60 seconds**
8. Is same-region Read Replica replication free? **→ Yes, even cross-AZ**
