---
tags: [concept, storage, file-system, managed-service, hybrid]
aliases: [FSx, Amazon FSx, FSx for Windows, FSx for Lustre, FSx for ONTAP, FSx for OpenZFS]
date: 2026-05-21
---

# Amazon FSx

**Amazon FSx** is a family of **fully managed, high-performance file systems** built on popular open-source and commercial file system technologies. FSx eliminates the heavy lifting of setting up, maintaining, and operating file servers in the cloud.

> [!IMPORTANT]
> **Core exam concept:** FSx is the answer when the question asks for a **managed file system** that is NOT NFS/Linux-centric (that's [[Elastic File System (EFS)|EFS]]). Each FSx variant is designed for a specific workload and OS environment. Know which variant maps to which scenario.

---

## The FSx Family Overview

```
  ┌───────────────────────────────────────────────────────────────────┐
  │                        Amazon FSx Family                         │
  │                                                                   │
  │  ┌─────────────────────┐   ┌─────────────────────────────────┐   │
  │  │  FSx for Windows    │   │  FSx for Lustre                 │   │
  │  │  File Server        │   │  (HPC / ML / Big Data)          │   │
  │  │  (SMB + AD)         │   │  (POSIX-compliant)              │   │
  │  └─────────────────────┘   └─────────────────────────────────┘   │
  │                                                                   │
  │  ┌─────────────────────┐   ┌─────────────────────────────────┐   │
  │  │  FSx for NetApp     │   │  FSx for OpenZFS                │   │
  │  │  ONTAP              │   │  (ZFS-based workloads)          │   │
  │  │  (NFS + SMB + iSCSI)│   │  (Linux + NFS)                  │   │
  │  └─────────────────────┘   └─────────────────────────────────┘   │
  └───────────────────────────────────────────────────────────────────┘
```

---

## FSx for Windows File Server

A **fully managed Windows-native file system** built on Windows Server with full SMB protocol support and Microsoft Active Directory integration.

```
  ┌────────────────────┐       ┌─────────────────────────────┐
  │  Windows EC2       │ SMB   │  FSx for Windows            │
  │  Instances         │──────►│  File Server                │
  │                    │       │                             │
  │  Linux EC2         │ SMB   │  • NTFS file system         │
  │  (with SMB client) │──────►│  • Active Directory         │
  │                    │       │  • DFS Namespaces           │
  │  On-premises       │ VPN/  │  • DFS Replication          │
  │  Windows Servers   │──DX──►│  • VSS User-Driven Restores │
  └────────────────────┘       └─────────────────────────────┘
```

| Attribute | Detail |
|---|---|
| **Protocol** | **SMB** (Server Message Block) — Windows native |
| **Identity** | Integrates with **Microsoft Active Directory** (AWS Managed AD or self-managed) |
| **Features** | ACLs, user quotas, DFS Namespaces, DFS Replication, VSS shadow copies |
| **Storage** | SSD (low-latency) or HDD (broad spectrum) options |
| **Availability** | Single-AZ or **Multi-AZ** (automatic failover) |
| **Scale** | Up to 10s of GB/s throughput, millions of IOPS, 100s of PB |
| **Backup** | Daily automated backups to [[Amazon S3]] |
| **Access** | From EC2 (Windows/Linux), on-premises via VPN / [[AWS Direct Connect]] |

### Key Exam Points

- **Windows shared file system** → always FSx for Windows (never [[Elastic File System (EFS)|EFS]])
- Supports **DFS** (Distributed File System) for grouping files across multiple file systems
- Can be deployed **on-premises** via **Amazon FSx File Gateway** for low-latency local caching
- Data is **encrypted at rest** (KMS) and **in transit** (automatically)

> [!CAUTION]
> **Exam critical:** If the question mentions "Windows", "SMB", "Active Directory", or "NTFS" → the answer is **FSx for Windows File Server**, NOT EFS. EFS is for Linux NFS workloads only.

---

## FSx for Lustre

A **high-performance parallel file system** designed for workloads that need **sub-millisecond latency** and massive throughput — Machine Learning, High Performance Computing (HPC), video processing, and financial modeling.

The name "Lustre" is derived from **"Linux"** and **"Cluster"**.

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                     FSx for Lustre                               │
  │                                                                  │
  │  ┌────────────┐                    ┌─────────────────────────┐   │
  │  │ HPC / ML   │ POSIX             │  FSx for Lustre FS      │   │
  │  │ Compute    │─────────────────►  │                         │   │
  │  │ Cluster    │                    │  100s GB/s throughput   │   │
  │  │ (EC2)      │◄─────────────────  │  millions of IOPS      │   │
  │  └────────────┘                    │  sub-ms latency         │   │
  │                                    └────────────┬────────────┘   │
  │                                                 │                │
  │                                    ┌────────────▼────────────┐   │
  │                                    │  S3 Bucket (optional)   │   │
  │                                    │  Data Repository        │   │
  │                                    │  (linked to Lustre)     │   │
  │                                    └─────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────┘
```

| Attribute | Detail |
|---|---|
| **Protocol** | **POSIX-compliant** (Linux native) |
| **Performance** | 100s GB/s throughput, millions of IOPS, sub-millisecond latencies |
| **S3 Integration** | Can **transparently read/write S3** — presents S3 data as a file system |
| **Use cases** | Machine Learning training, HPC, video rendering, financial simulations, genome sequencing |
| **OS** | Linux only |

### Deployment Options

| Type | Description | Use Case |
|---|---|---|
| **Scratch** | Temporary storage, no replication, highest burst performance. Data NOT persisted if server fails. | Short-term processing, cost optimization |
| **Persistent** | Data replicated within same AZ, supports at-rest encryption. Long-term storage. | Longer-term workloads, sensitive data |

> [!WARNING]
> **Exam trap:** Scratch FSx for Lustre does NOT replicate data. If the underlying server fails, the data is lost. Use **Persistent** deployment for data that cannot be re-created. Scratch is for temporary compute-heavy jobs where the source data lives in S3.

### S3 Data Repository Integration

```
  S3 Bucket (source of truth)
       │
       │  Lazy loading: data loaded from S3 when first accessed
       │
       ▼
  FSx for Lustre
       │
       │  Results written back: hsm_archive command
       │  or auto-export policy
       ▼
  S3 Bucket (results)
```

- **Lazy loading**: Data is loaded from S3 into Lustre only when first accessed by the compute job.
- **Write-back**: Results can be written back to S3 via `hsm_archive` or automatic export.
- This makes FSx for Lustre a **processing acceleration layer** for S3 data.

> [!TIP]
> **Exam Pattern:** "Process S3 data with HPC / ML at high throughput" → **FSx for Lustre** with S3 data repository. "Windows shared drive" → **FSx for Windows**. "Linux shared NFS" → **EFS**.

---

## FSx for NetApp ONTAP

A **fully managed NetApp ONTAP file system** — the most feature-rich FSx variant. It supports NFS, SMB, and iSCSI protocols simultaneously, making it the **universal file system** for mixed workloads.

```
  ┌────────────────────────────────────────────────────────────────┐
  │                  FSx for NetApp ONTAP                          │
  │                                                                │
  │  ┌──────────┐  NFS   ┌──────────────┐                        │
  │  │  Linux   │───────►│              │                        │
  │  └──────────┘        │  ONTAP FS    │  • Auto-tiering        │
  │  ┌──────────┐  SMB   │              │  • SnapMirror          │
  │  │ Windows  │───────►│  NFS + SMB   │  • FlexClone           │
  │  └──────────┘        │  + iSCSI     │  • Compression         │
  │  ┌──────────┐ iSCSI  │              │  • Deduplication       │
  │  │  Apps    │───────►│              │  • Point-in-time       │
  │  └──────────┘        └──────────────┘    snapshots           │
  └────────────────────────────────────────────────────────────────┘
```

| Attribute | Detail |
|---|---|
| **Protocols** | **NFS, SMB, and iSCSI** — all simultaneously |
| **Compatibility** | Works with Linux, Windows, macOS, VMware, and virtually any workload |
| **Features** | Auto-tiering (SSD → capacity pool), SnapMirror replication, FlexClone (instant clones), data compression, deduplication, point-in-time snapshots |
| **Scale** | Scales up and out |
| **Use case** | Migration from on-premises NetApp storage, mixed-OS environments, workloads needing multi-protocol support |

> [!TIP]
> **Exam Pattern:** "Migrate from on-premises NetApp" → **FSx for NetApp ONTAP**. "Need NFS + SMB + iSCSI simultaneously" → **FSx for NetApp ONTAP**. "Multi-protocol file access for mixed OS" → **FSx for NetApp ONTAP**.

---

## FSx for OpenZFS

A **fully managed OpenZFS file system** for workloads already running on ZFS or Linux. Provides high performance with ZFS features like snapshots, cloning, and compression.

| Attribute | Detail |
|---|---|
| **Protocol** | **NFS** (v3, v4, v4.1, v4.2) |
| **Compatibility** | Linux, Windows, macOS — anything that speaks NFS |
| **Performance** | Up to 1 million IOPS, sub-ms latencies |
| **Features** | Point-in-time snapshots, data compression, instant cloning, copy-on-write |
| **Use case** | Migrate on-premises ZFS workloads to AWS, internal databases, CI/CD, dev environments |

> [!NOTE]
> FSx for OpenZFS is ideal for workloads currently running ZFS on Linux. If the exam mentions "ZFS" or "migrate ZFS workloads" → **FSx for OpenZFS**.

---

## FSx Decision Matrix

```
  "I need a file system for..."

  Windows workloads + SMB + Active Directory?
  └──► FSx for Windows File Server

  HPC / ML / massive throughput + S3 integration?
  └──► FSx for Lustre

  Multi-protocol (NFS + SMB + iSCSI) + NetApp features?
  └──► FSx for NetApp ONTAP

  ZFS-based workloads or migrate from ZFS?
  └──► FSx for OpenZFS

  Simple shared NFS for Linux EC2 fleet?
  └──► Amazon EFS (not FSx)
```

---

## Comparison Table

| Feature | FSx Windows | FSx Lustre | FSx ONTAP | FSx OpenZFS | [[Elastic File System (EFS)\|EFS]] |
|---|---|---|---|---|---|
| **Protocol** | SMB | POSIX | NFS + SMB + iSCSI | NFS | NFS |
| **OS Support** | Windows (primary) + Linux | Linux | All | All (via NFS) | Linux |
| **AD Integration** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **S3 Integration** | ❌ | ✅ (data repo) | ❌ | ❌ | ❌ |
| **Multi-AZ** | ✅ (optional) | ❌ | ✅ | ❌ | ✅ |
| **Max Throughput** | 10s GB/s | 100s GB/s | 10s GB/s | 10s GB/s | ~10 GB/s |
| **Primary Use** | Enterprise Windows | HPC / ML | Mixed workloads | ZFS migration | Linux NFS |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → FSx type:**
> - "Windows file server" or "SMB protocol" or "Active Directory" → **FSx for Windows**
> - "HPC" or "Machine Learning" or "high-throughput compute" or "Lustre" → **FSx for Lustre**
> - "Process S3 data with high performance" → **FSx for Lustre** (S3 data repository)
> - "NetApp migration" or "NFS + SMB + iSCSI" → **FSx for NetApp ONTAP**
> - "ZFS migration" or "OpenZFS" → **FSx for OpenZFS**
> - "Linux shared NFS file system" → **[[Elastic File System (EFS)|EFS]]** (not FSx)
>
> **Key traps:**
> - EFS = Linux NFS only. FSx for Windows = Windows SMB only. Don't mix them up.
> - FSx for Lustre Scratch = no replication, data lost on failure. Persistent = replicated within AZ.
> - FSx for Lustre can link to S3 and **lazily load** data — not a full copy, on-demand.
> - FSx for NetApp ONTAP is the only FSx that supports **all three protocols** (NFS + SMB + iSCSI).
> - FSx for Windows supports **Multi-AZ** deployment for HA.
