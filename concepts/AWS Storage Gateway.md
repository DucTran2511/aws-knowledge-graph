---
tags: [concept, storage, hybrid, on-premises, migration, gateway]
aliases: [Storage Gateway, AWS Storage Gateway, S3 File Gateway, FSx File Gateway, Volume Gateway, Tape Gateway]
date: 2026-05-21
---

# AWS Storage Gateway

**AWS Storage Gateway** is a **hybrid cloud storage service** that gives on-premises applications virtually unlimited cloud storage access using standard storage protocols. It acts as a **bridge** between on-premises infrastructure and AWS cloud storage.

> [!IMPORTANT]
> **Core exam concept:** Storage Gateway is the answer when the question involves **on-premises applications** that need to access **AWS cloud storage** with **low-latency local caching**. It is NOT for cloud-native workloads — it's for **hybrid architectures** where data needs to flow between on-premises and AWS.

---

## Why Storage Gateway?

```
  The Problem:
  ┌──────────────────┐                    ┌──────────────────┐
  │  On-Premises     │   Slow WAN link    │  AWS Cloud       │
  │  Data Center     │───────────────────►│                  │
  │                  │                    │  Infinitely      │
  │  Petabytes of    │   Can't just       │  scalable        │
  │  existing data   │   "lift and shift" │  storage (S3)    │
  │                  │   everything       │                  │
  └──────────────────┘                    └──────────────────┘

  The Solution:
  ┌──────────────────┐                    ┌──────────────────┐
  │  On-Premises     │                    │  AWS Cloud       │
  │                  │                    │                  │
  │  ┌──────────┐    │   Encrypted       │  ┌─────────┐    │
  │  │ Storage  │    │   connection       │  │   S3    │    │
  │  │ Gateway  │────│───────────────────►│  │  EBS    │    │
  │  │ (VM/HW)  │    │   over internet   │  │  Glacier│    │
  │  │          │    │   or Direct Connect│  └─────────┘    │
  │  │ Local    │    │                    │                  │
  │  │ cache    │    │                    │                  │
  │  └──────────┘    │                    │                  │
  └──────────────────┘                    └──────────────────┘
```

Storage Gateway is deployed as a **VM** (VMware, Hyper-V, KVM) or as a **hardware appliance** in your data center. It stores a **local cache** of frequently accessed data for low-latency access while transparently moving data to AWS.

---

## The Four Gateway Types

```
  ┌───────────────────────────────────────────────────────────────────┐
  │                   AWS Storage Gateway Types                       │
  │                                                                   │
  │  ┌─────────────────────┐   ┌─────────────────────────────────┐   │
  │  │  S3 File Gateway    │   │  FSx File Gateway               │   │
  │  │  NFS / SMB → S3     │   │  SMB → FSx for Windows          │   │
  │  └─────────────────────┘   └─────────────────────────────────┘   │
  │                                                                   │
  │  ┌─────────────────────┐   ┌─────────────────────────────────┐   │
  │  │  Volume Gateway     │   │  Tape Gateway                   │   │
  │  │  iSCSI → EBS/S3     │   │  iSCSI-VTL → S3/Glacier        │   │
  │  └─────────────────────┘   └─────────────────────────────────┘   │
  └───────────────────────────────────────────────────────────────────┘
```

---

## S3 File Gateway

Presents [[Amazon S3]] objects as files via **NFS** or **SMB** protocols. On-premises applications access S3 as if it were a local file share.

```
  On-Premises                              AWS Cloud
  ┌─────────────────────┐                  ┌──────────────────────┐
  │  Application Server │                  │                      │
  │       │              │                  │  ┌────────────────┐  │
  │       │ NFS / SMB    │                  │  │  S3 Standard   │  │
  │       ▼              │                  │  │  S3 Standard-IA│  │
  │  ┌──────────────┐    │   HTTPS         │  │  S3 One Zone-IA│  │
  │  │ S3 File      │────│────────────────►│  │  S3 Int-Tier   │  │
  │  │ Gateway      │    │                  │  └───────┬────────┘  │
  │  │              │    │                  │          │            │
  │  │ Local cache  │    │                  │   Lifecycle Rules     │
  │  │ (recent data)│    │                  │          │            │
  │  └──────────────┘    │                  │  ┌───────▼────────┐  │
  │                      │                  │  │  S3 Glacier    │  │
  └─────────────────────┘                  │  └────────────────┘  │
                                           └──────────────────────┘
```

| Attribute | Detail |
|---|---|
| **Protocol** | **NFS** (v3, v4.1) and **SMB** |
| **Backend** | [[Amazon S3]] (Standard, Standard-IA, One Zone-IA, Intelligent-Tiering) |
| **Local cache** | Most recently used data cached locally for low-latency access |
| **Lifecycle** | Use S3 Lifecycle Rules to transition to Glacier (gateway stores in S3 first) |
| **Integration** | IAM roles for S3 access, [[Amazon S3\|S3 Bucket Policies]], AD for SMB authentication |
| **Use case** | Extend on-premises file storage to the cloud, backup, migration |

### Key Behaviors

- Files written through the gateway are stored as **individual S3 objects** (1:1 mapping).
- Metadata (permissions, timestamps) stored as S3 **object metadata**.
- Once objects are in S3, they can be managed with the full S3 toolset: lifecycle rules, replication, analytics, etc.
- Most recently used data is kept in a **local cache** for fast access.

> [!TIP]
> **Exam Pattern:** "On-premises NFS share backed by S3" → **S3 File Gateway**. "Cache frequently accessed files locally, store everything in S3" → **S3 File Gateway**. Data transitions to Glacier happen through S3 Lifecycle Rules (not the gateway directly).

---

## FSx File Gateway

Provides low-latency **on-premises access** to a fully managed **[[Amazon FSx|FSx for Windows File Server]]** in AWS.

```
  On-Premises                              AWS Cloud
  ┌─────────────────────┐                  ┌──────────────────────┐
  │  Windows Servers     │                  │                      │
  │       │              │                  │  ┌────────────────┐  │
  │       │ SMB          │                  │  │ FSx for Windows│  │
  │       ▼              │                  │  │ File Server    │  │
  │  ┌──────────────┐    │                  │  │                │  │
  │  │ FSx File     │────│── VPN / DX ────►│  │ • SMB          │  │
  │  │ Gateway      │    │                  │  │ • AD Integrated│  │
  │  │              │    │                  │  │ • DFS          │  │
  │  │ Local cache  │    │                  │  └────────────────┘  │
  │  └──────────────┘    │                  │                      │
  └─────────────────────┘                  └──────────────────────┘
```

| Attribute | Detail |
|---|---|
| **Protocol** | **SMB** |
| **Backend** | [[Amazon FSx\|FSx for Windows File Server]] |
| **Local cache** | Frequently accessed data cached locally |
| **Use case** | Group file shares, home directories, CMS — when data lives in FSx for Windows |
| **Connectivity** | Requires **VPN** or **AWS Direct Connect** |

> [!NOTE]
> FSx File Gateway is specifically for **FSx for Windows File Server**. If the question asks for on-premises access to a Windows file share in AWS with local caching → **FSx File Gateway**.

---

## Volume Gateway

Presents cloud-backed block storage volumes to on-premises applications via the **iSCSI** protocol. There are two modes:

### Cached Volumes

```
  On-Premises                              AWS Cloud
  ┌─────────────────────┐                  ┌──────────────────────┐
  │  Application        │                  │                      │
  │       │              │                  │  ┌────────────────┐  │
  │       │ iSCSI        │                  │  │  S3            │  │
  │       ▼              │                  │  │  (full data)   │  │
  │  ┌──────────────┐    │                  │  │                │  │
  │  │ Volume GW    │────│── HTTPS ───────►│  └───────┬────────┘  │
  │  │ (Cached)     │    │                  │          │           │
  │  │              │    │                  │  ┌───────▼────────┐  │
  │  │ Cache:       │    │                  │  │  EBS Snapshots │  │
  │  │ hot data     │    │                  │  │  (point-in-time│  │
  │  │ locally      │    │                  │  │   backups)     │  │
  │  └──────────────┘    │                  │  └────────────────┘  │
  └─────────────────────┘                  └──────────────────────┘

  Primary data: S3  |  Cache: On-premises  |  Low latency for hot data
```

### Stored Volumes

```
  On-Premises                              AWS Cloud
  ┌─────────────────────┐                  ┌──────────────────────┐
  │  Application        │                  │                      │
  │       │              │                  │  ┌────────────────┐  │
  │       │ iSCSI        │                  │  │  S3            │  │
  │       ▼              │                  │  │  (async backup)│  │
  │  ┌──────────────┐    │                  │  │                │  │
  │  │ Volume GW    │────│── HTTPS ───────►│  └───────┬────────┘  │
  │  │ (Stored)     │    │   (async        │          │           │
  │  │              │    │    snapshots)    │  ┌───────▼────────┐  │
  │  │ Full data    │    │                  │  │  EBS Snapshots │  │
  │  │ stored       │    │                  │  └────────────────┘  │
  │  │ locally      │    │                  │                      │
  │  └──────────────┘    │                  │                      │
  └─────────────────────┘                  └──────────────────────┘

  Primary data: On-premises  |  Backups: S3 as EBS Snapshots
```

| Mode | Primary Data Location | On-Premises Storage | Use Case |
|---|---|---|---|
| **Cached Volumes** | **S3** (entire dataset in cloud) | Cache only (frequently accessed) | Reduce on-premises storage costs |
| **Stored Volumes** | **On-premises** (entire dataset local) | Full dataset | Low-latency access to full dataset, async backup to AWS |

Both modes back up data as **EBS Snapshots** in S3, which can be used to create [[Elastic Block Store (EBS) Volumes|EBS volumes]] for EC2 instances.

> [!WARNING]
> **Exam trap:** Don't confuse Cached vs Stored volumes:
> - **Cached**: primary data in S3, cache on-premises → saves local storage
> - **Stored**: primary data on-premises, backups to S3 → low-latency full access

---

## Tape Gateway

Replaces physical tape backup infrastructure with cloud-backed **virtual tapes** stored in S3 and Glacier. Uses the **iSCSI Virtual Tape Library (VTL)** protocol.

```
  On-Premises                              AWS Cloud
  ┌─────────────────────┐                  ┌──────────────────────┐
  │  Backup Application  │                  │                      │
  │  (Veeam, NetBackup, │                  │  ┌────────────────┐  │
  │   Commvault, etc.)  │                  │  │  S3            │  │
  │       │              │                  │  │  (Virtual Tapes)│  │
  │       │ iSCSI-VTL    │                  │  └───────┬────────┘  │
  │       ▼              │                  │          │           │
  │  ┌──────────────┐    │                  │     Archive         │
  │  │ Tape         │────│── HTTPS ───────►│          │           │
  │  │ Gateway      │    │                  │  ┌───────▼────────┐  │
  │  │              │    │                  │  │ S3 Glacier     │  │
  │  │ Virtual tapes│    │                  │  │ or Deep Archive│  │
  │  │ (local cache)│    │                  │  └────────────────┘  │
  │  └──────────────┘    │                  │                      │
  └─────────────────────┘                  └──────────────────────┘

  Backup software sees virtual tapes → data actually goes to S3/Glacier
```

| Attribute | Detail |
|---|---|
| **Protocol** | iSCSI VTL (Virtual Tape Library) |
| **Backend** | S3 (active tapes), S3 Glacier / Deep Archive (archived tapes) |
| **Compatible** | Works with leading backup software (Veeam, Veritas NetBackup, Commvault, etc.) |
| **Use case** | Replace physical tape backup with cloud storage — no application changes needed |

> [!TIP]
> **Exam Pattern:** "Replace on-premises tape backup with cloud storage" or "iSCSI VTL" → **Tape Gateway**. The beauty is that the backup software doesn't know the difference — it still "writes tapes", but the tapes are virtual and backed by S3/Glacier.

---

## Gateway Deployment Options

| Deployment | Description |
|---|---|
| **On-premises VM** | Install as a VM on VMware ESXi, Microsoft Hyper-V, or Linux KVM |
| **Hardware appliance** | AWS-provided physical hardware appliance for customers without VM infrastructure |
| **Amazon EC2** | Run as an EC2 instance for cloud-side testing or disaster recovery scenarios |

---

## Storage Gateway Decision Matrix

```
  "I need to..."

  Access S3 files from on-premises via NFS/SMB?
  └──► S3 File Gateway

  Access FSx for Windows from on-premises via SMB?
  └──► FSx File Gateway

  Present block storage (iSCSI) to on-premises apps, backed by S3?
  └──► Volume Gateway (Cached or Stored)

  Replace physical tape backup infrastructure?
  └──► Tape Gateway
```

---

## Comparison Table

| Feature | S3 File GW | FSx File GW | Volume GW (Cached) | Volume GW (Stored) | Tape GW |
|---|---|---|---|---|---|
| **Protocol** | NFS / SMB | SMB | iSCSI | iSCSI | iSCSI-VTL |
| **Backend** | S3 | FSx Windows | S3 | On-prem + S3 | S3 + Glacier |
| **Data location** | S3 | FSx | S3 (cache local) | Local (backup S3) | S3/Glacier |
| **Local cache** | ✅ | ✅ | ✅ | ❌ (full local) | ✅ |
| **Use case** | File shares → S3 | File shares → FSx | Block → cloud | Block + backup | Tape replacement |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Storage Gateway type:**
> - "On-premises NFS/SMB file share to S3" → **S3 File Gateway**
> - "On-premises Windows file share to FSx" → **FSx File Gateway**
> - "On-premises block storage backed by cloud" → **Volume Gateway**
> - "Low-latency full local access with cloud backup" → **Volume Gateway (Stored)**
> - "Primary data in cloud, cache on-premises" → **Volume Gateway (Cached)**
> - "Replace tape backup" or "virtual tape library" → **Tape Gateway**
> - "Hybrid cloud storage" or "on-premises to AWS" → Think **Storage Gateway**
> - "Backup to Glacier from on-premises" → **Tape Gateway** (tapes archived to Glacier)
>
> **Key facts:**
> - Storage Gateway runs as a **VM** or **hardware appliance** on-premises
> - S3 File Gateway: files stored as S3 objects (1:1 mapping), lifecycle rules for Glacier
> - Volume Gateway Cached: primary data in S3, hot data cached locally
> - Volume Gateway Stored: primary data local, async snapshots to S3/EBS
> - Tape Gateway: works with existing backup software — no changes needed
> - All data encrypted in transit (HTTPS/SSL) and at rest
> - S3 File Gateway does NOT support Glacier directly — use S3 Lifecycle Rules
