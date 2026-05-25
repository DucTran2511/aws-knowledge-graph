---
tags: [concept, storage, data-migration, edge-computing, hybrid, offline]
aliases: [Snow Family, Snowball, Snowcone, Snowmobile, AWS Snow, Snowball Edge]
date: 2026-05-21
---

# AWS Snow Family

The **AWS Snow Family** is a collection of **physical devices** used for two main purposes:

1. **Data Migration** — Move massive amounts of data into or out of AWS when network transfer is too slow, too expensive, or impractical.
2. **Edge Computing** — Run compute and storage at locations with limited or no internet connectivity.

> [!IMPORTANT]
> **Core exam concept:** The Snow Family solves the problem: "How do I move petabytes/exabytes of data to AWS when a network transfer would take weeks, months, or years?" The answer is to **physically ship the data on a device**.

---

## Why Not Just Use the Network?

```
  Data Transfer Time Over 1 Gbps Network:

  ┌──────────────┬─────────────────────┐
  │  Data Size   │  Transfer Time      │
  ├──────────────┼─────────────────────┤
  │  10 TB       │  ~1 day             │
  │  100 TB      │  ~12 days           │
  │  1 PB        │  ~4 months          │
  │  10 PB       │  ~3.5 years ⚠️      │
  └──────────────┴─────────────────────┘

  Rule of thumb:
  If network transfer takes MORE THAN 1 WEEK →
  consider using an AWS Snow device instead.
```

> [!TIP]
> **Exam Pattern:** "Migrate 80 TB of data to AWS" or "limited bandwidth" or "data center migration" → Think **Snow Family**. If the question mentions petabytes or exabytes → think **Snowmobile**.

---

## The Three Devices

### AWS Snowcone

The **smallest and most portable** member of the Snow Family. Designed for edge locations where space and power are constrained.

```
  ┌──────────────────────────────────────────┐
  │             AWS Snowcone                  │
  │                                          │
  │    Weight: 4.5 lbs (2.1 kg)             │
  │    Size: Small, ruggedized               │
  │    Power: USB-C or battery optional      │
  │                                          │
  │    ┌──────────────────┐                  │
  │    │  Snowcone         │  8 TB HDD       │
  │    │  Snowcone SSD     │  14 TB SSD      │
  │    └──────────────────┘                  │
  │                                          │
  │    Compute: 2 vCPUs, 4 GB RAM            │
  │    Can run EC2 instances + Lambda        │
  │    Supports AWS IoT Greengrass           │
  └──────────────────────────────────────────┘
```

| Attribute | Detail |
|---|---|
| **Storage** | 8 TB HDD (Snowcone) or 14 TB SSD (Snowcone SSD) |
| **Compute** | 2 vCPUs, 4 GB RAM |
| **Use cases** | Edge computing in harsh environments, small data collection, IoT, migration of small datasets |
| **Data transfer** | Ship device back to AWS OR use **AWS DataSync** over the network |
| **Weight** | 4.5 lbs (2.1 kg) — fits in a backpack |

> [!NOTE]
> Snowcone is the only Snow device that supports **AWS DataSync** agent pre-installed — you can send data back to AWS over the network if connectivity is available, instead of shipping the device.

---

### AWS Snowball Edge

The **mid-range workhorse** of the Snow Family. Available in two flavors optimized for different use cases.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    AWS Snowball Edge                          │
  │                                                              │
  │  ┌───────────────────────────┐  ┌───────────────────────┐   │
  │  │  Storage Optimized        │  │  Compute Optimized    │   │
  │  │                           │  │                       │   │
  │  │  80 TB usable HDD         │  │  42 TB usable HDD    │   │
  │  │  + 1 TB SSD (block vol)  │  │  + 7.68 TB NVMe SSD  │   │
  │  │                           │  │                       │   │
  │  │  40 vCPUs, 80 GB RAM     │  │  104 vCPUs, 416 GB   │   │
  │  │                           │  │  RAM                  │   │
  │  │  Best for: Large-scale   │  │  Optional GPU         │   │
  │  │  data migration          │  │                       │   │
  │  └───────────────────────────┘  │  Best for: HPC,      │   │
  │                                  │  ML inference        │   │
  │                                  └───────────────────────┘   │
  └──────────────────────────────────────────────────────────────┘

  Both run:
  • EC2 instances (AMIs)
  • AWS Lambda functions
  • AWS IoT Greengrass
  • Clustering (up to 5-15 devices)
```

| Variant | Storage | Compute | GPU | Best For |
|---|---|---|---|---|
| **Storage Optimized** | 80 TB HDD + 1 TB SSD | 40 vCPUs, 80 GB RAM | ❌ | Data migration, local storage |
| **Compute Optimized** | 42 TB HDD + 7.68 TB NVMe | 104 vCPUs, 416 GB RAM | Optional | ML inference, video processing |

> [!TIP]
> **Exam Pattern:** "Move 50 TB to AWS" → **Snowball Edge Storage Optimized**. "Run ML inference at the edge with no internet" → **Snowball Edge Compute Optimized**.

---

### AWS Snowmobile

A **45-foot ruggedized shipping container**, pulled by a semi-trailer truck. Designed for **exabyte-scale** data migration.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                     AWS Snowmobile                            │
  │                                                              │
  │    ┌──────────────────────────────────────────────┐          │
  │    │                                              │          │
  │    │          45-foot shipping container           │          │
  │    │          on a semi-trailer truck              │          │
  │    │                                              │          │
  │    │          100 PB per Snowmobile                │          │
  │    │                                              │          │
  │    │          • GPS tracking                       │          │
  │    │          • 24/7 video surveillance            │          │
  │    │          • Escort security vehicle            │          │
  │    │          • Dedicated AWS security personnel   │          │
  │    └──────────────────────────────────────────────┘          │
  │                                                              │
  │    Transfer 1 EB = 10 Snowmobiles                            │
  │    Better than Snowball if > 10 PB of data                   │
  └──────────────────────────────────────────────────────────────┘
```

| Attribute | Detail |
|---|---|
| **Storage** | **100 PB** per Snowmobile |
| **Security** | GPS tracking, 24/7 video surveillance, alarm monitoring, escort vehicle, optional armed security |
| **Temperature** | Climate-controlled |
| **Network** | High-speed network connection to your data center |
| **Use case** | Data center shutdown / massive migration (> 10 PB) |

> [!CAUTION]
> **Snowmobile is for extreme scenarios only.** If you have less than 10 PB, multiple Snowball Edge devices are typically more practical. Snowmobile requires AWS to literally **drive a truck** to your data center.

---

## Device Selection Guide

```
  How much data do you need to move?

  < 8 TB?
  └──► AWS Snowcone (or just use the network)

  8 TB – 80 TB?
  └──► AWS Snowball Edge (Storage Optimized)

  80 TB – 10 PB?
  └──► Multiple Snowball Edge devices

  > 10 PB (up to exabytes)?
  └──► AWS Snowmobile
```

| Device | Storage Capacity | Compute | Best For |
|---|---|---|---|
| **Snowcone** | 8–14 TB | 2 vCPUs, 4 GB | Small edge deployments, constrained environments |
| **Snowball Edge SO** | 80 TB | 40 vCPUs, 80 GB | Large data migrations |
| **Snowball Edge CO** | 42 TB | 104 vCPUs, 416 GB | Edge compute + ML |
| **Snowmobile** | 100 PB | N/A | Exabyte-scale data center migration |

---

## Snow Family: Edge Computing

Beyond data migration, Snow devices run **compute at the edge** — locations with limited or no internet connectivity.

### Edge Computing Use Cases

- **Process data as it's created** — on a ship, at a mine, on a truck
- **Pre-process data** before shipping to AWS
- **Machine Learning inference** at the edge
- **Transcode media** in remote locations

### What Can Run on Snow Devices?

| Capability | Snowcone | Snowball Edge |
|---|---|---|
| **EC2 instances** | ✅ (small) | ✅ |
| **Lambda functions** | ✅ (via IoT Greengrass) | ✅ |
| **IoT Greengrass** | ✅ | ✅ |
| **Clustering** | ❌ | ✅ (up to 5–15 devices) |
| **AWS OpsHub** | ✅ | ✅ |

> [!NOTE]
> **AWS OpsHub** is a graphical application (installed on your laptop) that provides an easy-to-use GUI to manage Snow devices — transfer data, launch EC2 instances, and monitor device status. Replaces the old CLI-only approach.

---

## Snow Family: Data Transfer Workflow

```
  1. REQUEST                           2. RECEIVE
  ┌──────────┐                        ┌──────────┐
  │  AWS      │  AWS ships device     │  You     │
  │  Console  │──────────────────────►│  receive │
  │  Order    │                       │  device  │
  └──────────┘                        └─────┬────┘
                                            │
  4. AWS LOADS DATA                    3. LOAD DATA
  ┌──────────┐                        ┌─────▼────┐
  │  AWS      │◄─────────────────────│  Copy    │
  │  imports  │  You ship device     │  data to │
  │  to S3    │  back to AWS         │  device  │
  └──────────┘                        └──────────┘

  All data is encrypted with 256-bit encryption
  before leaving your premises.
```

### Key Security Features

- All data is **encrypted at rest** using 256-bit encryption keys managed through [[AWS KMS]].
- **Tamper-resistant** enclosures with **Trusted Platform Module (TPM)**.
- After transfer, AWS performs a **software erasure** of the device following NIST guidelines.

---

## Snow Family + Other Services

| Integration | Description |
|---|---|
| **S3** | Snow devices transfer data into S3 buckets |
| **Glacier** | Cannot import directly to Glacier — import to S3 first, then use lifecycle rules |
| **EC2** | Run EC2 AMIs on Snowball Edge for edge compute |
| **DataSync** | Pre-installed on Snowcone for online data transfer |
| **OpsHub** | GUI management tool for all Snow devices |

> [!WARNING]
> **Exam trap:** You **cannot import data directly into Glacier** via Snow devices. Data goes to S3 first, then you transition to Glacier using **S3 Lifecycle Rules**.

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Snow Family:**
> - "Transfer terabytes/petabytes offline" → **Snowball Edge**
> - "Limited internet bandwidth for large migration" → **Snow Family**
> - "Edge computing in remote location" → **Snowball Edge** or **Snowcone**
> - "Data center migration (exabytes)" → **Snowmobile**
> - "Small, portable edge device" → **Snowcone**
> - "ML inference at the edge" → **Snowball Edge Compute Optimized**
> - "Move 10 PB+" → **Snowmobile**
> - "Cannot transfer to Glacier directly" → Import to S3, then lifecycle rule
>
> **Key facts:**
> - Snowcone: 8/14 TB, 2 vCPUs — smallest, supports DataSync
> - Snowball Edge Storage Optimized: 80 TB, 40 vCPUs — bulk migration
> - Snowball Edge Compute Optimized: 42 TB, 104 vCPUs — edge compute/ML
> - Snowmobile: 100 PB — exabyte-scale, literal truck
> - All devices are encrypted (256-bit, KMS-managed keys)
> - Data → S3 first, THEN lifecycle to Glacier (no direct Glacier import)
> - Rule of thumb: > 1 week network transfer → consider Snow Family
