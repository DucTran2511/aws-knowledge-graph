---
tags: [concept, storage, s3, cost-optimization, archival]
aliases: [S3 Storage Classes, S3 Tiers, Glacier, S3 Intelligent-Tiering]
date: 2026-05-08
---

# S3 Storage Classes

[[Amazon S3]] offers **7 storage classes** designed for different access patterns. All share the same **11 nines (99.999999999%) durability**, but differ in availability, retrieval cost, and minimum storage duration.

> [!IMPORTANT]
> **Core exam concept:** Choosing the right storage class is a cost optimization question. You must match the **access frequency** and **retrieval urgency** to the correct class.

---

## Storage Class Deep Dive

### S3 Standard (General Purpose)

| Attribute | Value |
|---|---|
| **Availability** | 99.99% (4 nines) |
| **AZs** | ≥ 3 |
| **Min storage duration** | None |
| **Retrieval fee** | None |
| **Use case** | Frequently accessed data, big data analytics, content distribution |

- **Default** storage class.
- Sustains the loss of 2 concurrent AZs.
- No minimum storage duration or retrieval charges → most flexible.

---

### S3 Intelligent-Tiering

```
  ┌──────────────────────────────────────────────────────────────┐
  │              S3 Intelligent-Tiering (Auto-Tiering)           │
  │                                                              │
  │  ┌──────────────────┐  No access    ┌─────────────────────┐ │
  │  │ Frequent Access   │──── 30d ────►│ Infrequent Access    │ │
  │  │ (automatic)       │◄────────────│ (automatic)          │ │
  │  └──────────────────┘  Accessed     └──────────┬──────────┘ │
  │                                                 │            │
  │                                       90 days   │            │
  │                                       no access │            │
  │                                                 ▼            │
  │                                    ┌─────────────────────┐   │
  │                                    │ Archive Instant     │   │
  │                                    │ Access (automatic)  │   │
  │                                    └──────────┬──────────┘   │
  │                                               │ (opt-in)    │
  │                                    ┌──────────▼──────────┐   │
  │                                    │ Archive Access      │   │
  │                                    │ (90-730 days)       │   │
  │                                    └──────────┬──────────┘   │
  │                                               │ (opt-in)    │
  │                                    ┌──────────▼──────────┐   │
  │                                    │ Deep Archive Access │   │
  │                                    │ (180-730 days)      │   │
  │                                    └─────────────────────┘   │
  └──────────────────────────────────────────────────────────────┘
```

| Attribute | Value |
|---|---|
| **Availability** | 99.9% |
| **Monitoring fee** | Small per-object monitoring & automation fee |
| **Retrieval fee** | None (for Frequent and Infrequent tiers) |
| **Min storage duration** | None |
| **Use case** | Unknown or changing access patterns |

- **No retrieval charges** — unlike IA classes.
- Automatically moves objects between tiers based on access patterns.
- Archive tiers are **opt-in** and configurable.
- Best for when you **cannot predict** access patterns.

> [!TIP]
> **Exam Pattern:** "Access pattern is unknown" or "unpredictable workload" → **S3 Intelligent-Tiering**. It's the "set and forget" class.

---

### S3 Standard-IA (Infrequent Access)

| Attribute | Value |
|---|---|
| **Availability** | 99.9% (3 nines) |
| **AZs** | ≥ 3 |
| **Min storage duration** | 30 days |
| **Min object size** | 128 KB (charged minimum) |
| **Retrieval fee** | Per-GB retrieval charge |
| **Use case** | Disaster recovery, backups, data accessed less frequently but needs fast access |

> [!WARNING]
> **Exam trap:** Standard-IA has a **30-day minimum** charge. If you store an object for 1 day and delete it, you're still charged for 30 days. Same for 128 KB minimum size.

---

### S3 One Zone-IA

| Attribute | Value |
|---|---|
| **Availability** | 99.5% (lower!) |
| **AZs** | **1** (single AZ) |
| **Min storage duration** | 30 days |
| **Retrieval fee** | Per-GB |
| **Use case** | Secondary backup copies, re-creatable data (thumbnails) |

- **20% cheaper** than Standard-IA.
- Data **lost if AZ is destroyed** (still 11 nines durability within that AZ).

> [!CAUTION]
> **Never use One Zone-IA** for data that cannot be recreated. If the AZ goes down, data is gone. Use for thumbnails, re-processable data, or secondary backup copies only.

---

### S3 Glacier Instant Retrieval

| Attribute | Value |
|---|---|
| **Availability** | 99.9% |
| **Min storage duration** | **90 days** |
| **Retrieval speed** | **Milliseconds** (same as Standard) |
| **Retrieval fee** | Per-GB (higher than IA) |
| **Use case** | Archive data that needs instant access (e.g., medical images, news media) |

- Great for data accessed **once per quarter** but needing instant retrieval.
- Cheaper storage than Standard-IA, but more expensive retrieval.

---

### S3 Glacier Flexible Retrieval

| Attribute | Value |
|---|---|
| **Availability** | 99.99% |
| **Min storage duration** | **90 days** |
| **Retrieval options** | Expedited (1–5 min), Standard (3–5 hrs), Bulk (5–12 hrs) |
| **Use case** | Archival data with flexible retrieval needs |

```
  Retrieval Tiers:
  ┌──────────────┬──────────────┬───────────────┐
  │  Expedited   │   Standard   │     Bulk      │
  │  1–5 min     │   3–5 hours  │   5–12 hours  │
  │  $$$         │   $$         │   $           │
  │  Urgent      │   Normal     │   Large batch │
  └──────────────┴──────────────┴───────────────┘
```

- Formerly called "S3 Glacier" (renamed for clarity).

---

### S3 Glacier Deep Archive

| Attribute | Value |
|---|---|
| **Availability** | 99.99% |
| **Min storage duration** | **180 days** |
| **Retrieval options** | Standard (12 hours), Bulk (48 hours) |
| **Use case** | Long-term compliance archives, regulatory retention |

- **Cheapest** storage class in S3.
- Designed for data accessed **once or twice per year**.
- Retrieval: 12–48 hours.

> [!IMPORTANT]
> **Exam Pattern:** "Store data for 7 years for compliance at lowest cost" → **S3 Glacier Deep Archive**. "Archive with occasional same-day access" → **Glacier Flexible (Expedited)**. "Archive but must access in milliseconds" → **Glacier Instant Retrieval**.

---

## Cost Comparison (Relative)

```
  Storage Cost (per GB/month)          Retrieval Cost (per GB)
  ──────────────────────────           ──────────────────────
  HIGH │ Standard                      HIGH │ Deep Archive
       │ Intelligent-Tiering                │ Glacier Flexible
       │ Standard-IA                        │ Glacier Instant
       │ One Zone-IA                        │ One Zone-IA
       │ Glacier Instant                    │ Standard-IA
       │ Glacier Flexible                   │ Intelligent-Tiering
  LOW  │ Glacier Deep Archive          LOW  │ Standard
       ▼                                    ▼

  💡 Inverse relationship: cheaper storage = more expensive retrieval
```

---

## Lifecycle Transitions — Waterfall Rule

Objects can only transition **downward** (toward cheaper storage):

```
                    ┌───────────────────┐
                    │   S3 Standard     │
                    └─────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
  ┌───────────────┐ ┌─────────────────┐ ┌───────────────┐
  │ Standard-IA   │ │ Intelligent-    │ │ One Zone-IA   │
  │               │ │ Tiering         │ │               │
  └───────┬───────┘ └─────────────────┘ └───────┬───────┘
          │                                     │
          └──────────────┬──────────────────────┘
                         ▼
              ┌──────────────────────┐
              │ Glacier Instant      │
              └──────────┬───────────┘
                         ▼
              ┌──────────────────────┐
              │ Glacier Flexible     │
              └──────────┬───────────┘
                         ▼
              ┌──────────────────────┐
              │ Glacier Deep Archive │
              └──────────────────────┘
```

> [!NOTE]
> Objects must stay in a class for the **minimum storage duration** before transitioning. You cannot transition from Standard to Standard-IA before 30 days, or from Standard-IA to Glacier before 30 additional days.

---

## Exam Cheat Sheet

> [!TIP]
> **Quick decision guide:**
> - Frequent access, no retrieval cost → **Standard**
> - Unknown pattern → **Intelligent-Tiering**
> - Infrequent, multi-AZ, fast access → **Standard-IA**
> - Infrequent, re-creatable, single AZ → **One Zone-IA**
> - Archive, instant retrieval → **Glacier Instant**
> - Archive, hours OK → **Glacier Flexible**
> - Archive, 12-48 hours OK, cheapest → **Glacier Deep Archive**
>
> **Key traps:**
> - All classes have **11 nines durability** — durability is NOT a differentiator
> - **Availability** varies: Standard (99.99%) > IA (99.9%) > One Zone-IA (99.5%)
> - Minimum storage charges: IA/One Zone-IA = 30 days, Glacier Instant/Flexible = 90 days, Deep Archive = 180 days
> - Intelligent-Tiering has a small monitoring fee but **no retrieval fee**
> - One Zone-IA: data lost if AZ fails (use only for re-creatable data)
