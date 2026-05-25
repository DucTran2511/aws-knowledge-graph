---
tags: [concept, networking, dns, route53, routing-policy]
aliases: [Geoproximity Routing, Geoproximity Routing Policy, Traffic Flow]
date: 2026-05-04
---

# Route 53 Geoproximity Routing

*Parent article: [[Amazon Route 53]]*

**Geoproximity Routing** routes traffic based on the **geographic location of both the user and the resource**, with the ability to **shift traffic** between resources using a **Bias** value. It is the most flexible geographic routing policy and the only one that lets you control the "reach" of each endpoint.

> [!IMPORTANT]
> Geoproximity routing **requires Route 53 Traffic Flow**. You cannot configure it with simple record creation — you must use the Traffic Flow visual policy editor.

---

## How It Works

1. You define resources with their **location** (AWS Region or latitude/longitude for non-AWS resources).
2. Route 53 calculates the **geographic proximity** between each user and each resource.
3. By default, traffic goes to the **closest** resource.
4. You apply a **Bias** value to expand or shrink a resource's "reach" — shifting the geographic boundary.

```
Default (Bias = 0):                  Bias = +50 on us-east-1:
                                     
     ┌────────┬────────┐                 ┌──────────────┬───┐
     │        │        │                 │              │   │
     │  US    │   EU   │                 │     US       │EU │
     │ east   │  west  │                 │    east      │wst│
     │  50%   │  50%   │                 │    ~75%      │25%│
     │        │        │                 │              │   │
     └────────┴────────┘                 └──────────────┴───┘
     
     Equal geographic split              US "reaches further east"
                                         → steals traffic from EU
```

---

## The Bias Value

The Bias is the key differentiator of Geoproximity routing. It ranges from **-99 to +99**.

| Bias | Effect | Use Case |
|---|---|---|
| **0** (default) | Route to geographically closest resource | Normal operation |
| **+1 to +99** | **Expand** the resource's reach — attracts more traffic | "Pull" traffic toward this region |
| **-1 to -99** | **Shrink** the resource's reach — pushes traffic away | "Push" traffic away from this region |

### Bias in Practice

Imagine a world map split between `us-east-1` and `eu-west-1`. The Bias shifts the **dividing line**:

```
Bias: -25 on us-east-1, +25 on eu-west-1

Before (Bias 0):           After:
  US ◄──── line ────► EU    US ◄── line ──────────► EU
  50%          50%           30%              70%
                             
  The line moves WEST → EU "reaches further" into the Atlantic
  More users now route to EU
```

Higher absolute bias values produce **larger shifts**. A bias of ±99 produces a dramatic shift.

---

## Key Characteristics

| Feature | Detail |
|---|---|
| **Requires Traffic Flow** | ✅ **Mandatory** — cannot use without Traffic Flow ($50/month per policy record) |
| **Health checks** | ✅ Supported |
| **AWS resources** | Specify the **AWS Region** — Route 53 uses the region's geographic center |
| **Non-AWS resources** | Specify **latitude and longitude** coordinates |
| **Bias range** | -99 to +99 |
| **Default record** | Not required (geographic proximity always finds the closest) |
| **Alias record** | ✅ Supported |

---

## Route 53 Traffic Flow

Traffic Flow is the **visual policy editor** required for Geoproximity routing.

### Key Features

- **Visual editor** — drag-and-drop policy builder in the AWS console.
- **Policy versioning** — save, version, and rollback traffic policies.
- **Chaining** — combine multiple routing policies (e.g., Geoproximity → Failover → Weighted).
- **Traffic policy records** — apply a policy to a hosted zone record.

### Cost

- **$50/month per policy record** (not per policy — per record that uses the policy).
- This is in addition to standard Route 53 hosted zone and query charges.

---

## Use Cases

### 1. Gradual Regional Expansion

You're expanding from `us-east-1` to `eu-west-1`. Start with a small bias and increase over time:

| Phase | us-east-1 Bias | eu-west-1 Bias | Effect |
|---|---|---|---|
| Week 1 | 0 | +10 | EU attracts a little nearby traffic |
| Week 2 | 0 | +30 | EU serves most of Europe |
| Week 4 | -20 | +50 | EU handles most Atlantic traffic |
| Full | 0 | 0 | Settle at natural geographic split |

### 2. Region Evacuation / Maintenance

Set a large **negative bias** on the region under maintenance to push all traffic away:

```
us-east-1: Bias = -99  → Almost no traffic
eu-west-1: Bias = 0    → Absorbs nearly all traffic
```

### 3. Cost Optimization

Shift traffic toward regions with lower infrastructure or data transfer costs.

### 4. Unequal Capacity Distribution

If `us-east-1` has 3× the capacity of `eu-west-1`, use bias to send proportionally more traffic to the US.

---

## Geoproximity vs Geolocation vs Weighted

| Scenario | Best Policy |
|---|---|
| "Users in France must always go to EU server" (hard rule) | [[Route 53 Geolocation Routing|Geolocation]] |
| "Shift 70% traffic to US, 30% to EU" (exact percentages) | [[Route 53 Weighted Routing|Weighted]] |
| "Gradually expand EU presence by shifting the geographic boundary" | **Geoproximity** ✅ |
| "Route to closest region, but shift some border traffic" | **Geoproximity** ✅ |
| "Drain traffic from a region for maintenance" | **Geoproximity** ✅ (negative bias) |

> [!TIP]
> **Exam Pattern:** "Shift traffic from one region to another", "expand region reach", "bias", "gradually move traffic geographically", "Traffic Flow" → **Geoproximity Routing**. The word **"bias"** in a question almost always means Geoproximity.

> [!WARNING]
> **Geoproximity ≠ Weighted.** Weighted splits globally by percentage (any user anywhere can hit any endpoint). Geoproximity shifts the **geographic boundary** — users near the boundary change, but users deep in a region's territory don't. A user in Tokyo will always go to `ap-northeast-1` regardless of Bias between US and EU regions.
