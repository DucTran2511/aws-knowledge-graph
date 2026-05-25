---
tags: [concept, networking, dns, route53, routing-policy, high-availability]
aliases: [Failover Routing, Failover Routing Policy, Active-Passive Failover]
date: 2026-05-04
---

# Route 53 Failover Routing

*Parent article: [[Amazon Route 53]]*

**Failover Routing** configures an **active-passive** setup where Route 53 automatically switches traffic from a primary resource to a secondary (disaster recovery) resource when the primary becomes unhealthy.

---

## How It Works

1. You create **two records** with the same name — one designated **Primary**, one designated **Secondary**.
2. A **health check** is attached to the **Primary** record (mandatory).
3. Under normal conditions, Route 53 returns only the Primary record.
4. If the Primary's health check **fails**, Route 53 automatically returns the Secondary record instead.

```
                ┌──────────────────────┐
                │    Route 53          │
                │  Failover Policy     │
                └───────┬──────┬───────┘
                        │      │
              Health    │      │
              Check ────┤      │
                        │      │
                   Primary   Secondary
                   (active)  (passive / DR)
                        │      │
                        ▼      ▼
                  ┌──────┐  ┌──────┐
                  │ALB #1│  │ALB #2│
                  │US-E-1│  │EU-W-1│
                  └──────┘  └──────┘

Normal:    DNS → Primary (ALB #1)
Failover:  DNS → Secondary (ALB #2) ← when Primary fails health check
```

---

## Key Characteristics

| Feature | Detail |
|---|---|
| **Health check on Primary** | ✅ **Mandatory** — the failover trigger |
| **Health check on Secondary** | ✅ Optional but **recommended** |
| **Record types** | Exactly **one Primary** and **one Secondary** record per name |
| **Alias record** | ✅ Supported (with Evaluate Target Health) |
| **Failback** | Automatic — when Primary becomes healthy again, Route 53 switches back |

---

## Health Check Scenarios

| Primary | Secondary | Route 53 Returns |
|---|---|---|
| ✅ Healthy | ✅ Healthy | **Primary** |
| ❌ Unhealthy | ✅ Healthy | **Secondary** (failover!) |
| ✅ Healthy | ❌ Unhealthy | **Primary** |
| ❌ Unhealthy | ❌ Unhealthy | **Secondary** (returns it anyway as a last resort) |

> [!WARNING]
> If **both** Primary and Secondary are unhealthy, Route 53 still returns the **Secondary** record. It does NOT return an empty response. This is the "last resort" behavior.

---

## Failover with Alias Records

When using **Alias records** pointing to AWS resources (e.g., [[Elastic Load Balancer (ELB)|ELB]]), you can enable **Evaluate Target Health** instead of creating a separate health check.

- Route 53 automatically monitors the health of the underlying AWS resource.
- If the ALB reports unhealthy (all targets unhealthy), Route 53 triggers failover.
- This eliminates the need for a manually configured health check.

---

## Architecture Patterns

### 1. Active-Passive with S3 Static Site

A common exam pattern — failover to a static "sorry" page hosted on [[S3]]:

```
Primary:    ALB → EC2 instances (full application)
Secondary:  S3 Website Bucket (static maintenance page)

When ALB fails → users see "We're down for maintenance" on S3
```

### 2. Active-Passive Cross-Region DR

```
Primary:    us-east-1 (ALB + EC2 + RDS Primary)
Secondary:  eu-west-1 (ALB + EC2 + RDS Read Replica → promoted on failover)
```

### 3. Failover with Calculated Health Checks

Use [[Route 53 Health Checks|Calculated Health Checks]] to combine multiple health checks (web server + database + cache) into one composite check. Failover triggers only when the composite check fails.

---

## Failover vs Other High-Availability Patterns

| Pattern | Mechanism | Speed |
|---|---|---|
| **Route 53 Failover** | DNS-based (TTL-dependent) | Seconds to minutes (depends on TTL + health check interval) |
| **ELB Health Checks** | Real-time target removal | Seconds |
| **[[Auto Scaling Group (ASG)\|ASG]]** | Instance replacement | Minutes |
| **RDS Multi-AZ** | Database engine failover | 60-120 seconds |

> [!IMPORTANT]
> DNS failover is **not instantaneous**. It depends on:
> 1. Health check interval (default 30s, fast 10s)
> 2. Failure threshold (default 3 consecutive failures)
> 3. Client TTL caching (old DNS answer may be cached)
>
> Minimum failover time ≈ 30s × 3 = **~90 seconds** (with default settings) + TTL propagation.

---

## Combining with Other Policies

Failover can be the **outer layer** with other routing policies nested inside:

```
Route 53 Failover
    │
    ├── Primary: Latency Routing → multi-region active-active
    │       ├── us-east-1
    │       └── eu-west-1
    │
    └── Secondary: S3 static "under maintenance" page
```

This pattern provides latency-based routing under normal conditions, with a static fallback if all regions fail. This requires **Route 53 Traffic Flow** to configure nested policies.

> [!TIP]
> **Exam Pattern:** "Active-passive DR", "automatically switch to backup", "disaster recovery with DNS" → **Failover Routing**. If the question mentions "active-active" across regions, consider [[Route 53 Latency Routing|Latency]] or [[Route 53 Weighted Routing|Weighted]] instead.
