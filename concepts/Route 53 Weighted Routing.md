---
tags: [concept, networking, dns, route53, routing-policy]
aliases: [Weighted Routing, Weighted Routing Policy]
date: 2026-05-04
---

# Route 53 Weighted Routing

*Parent article: [[Amazon Route 53]]*

**Weighted Routing** allows you to control the **percentage of DNS requests** that are routed to each resource. You assign a relative **weight** to each record, and Route 53 responds to queries proportionally based on those weights.

---

## How It Works

1. You create **multiple records** with the **same name and type** (e.g., multiple A records for `app.example.com`).
2. Each record points to a different resource and has a **weight** value assigned.
3. Route 53 returns a record based on the proportion: `traffic % = weight / sum of all weights`.

```
                    Route 53 (Weighted)
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   Weight: 70        Weight: 20        Weight: 10
   (70%)              (20%)              (10%)
        │                 │                 │
        ▼                 ▼                 ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │  EC2 #1  │      │  EC2 #2  │      │  EC2 #3  │
   │us-east-1 │      │eu-west-1 │      │ap-south-1│
   └─────────┘      └─────────┘      └─────────┘
```

---

## Weight Math

| Record | Weight | Traffic % |
|---|---|---|
| A → `10.0.0.1` | 70 | 70 / (70+20+10) = **70%** |
| A → `10.0.0.2` | 20 | 20 / (70+20+10) = **20%** |
| A → `10.0.0.3` | 10 | 10 / (70+20+10) = **10%** |

- Weights **do not** need to sum to 100. Route 53 calculates proportions automatically.
- Weights can be any integer from **0 to 255**.

---

## Key Characteristics

| Feature | Detail |
|---|---|
| **Health checks** | ✅ Can associate health checks with each record |
| **Weight = 0** | Stops sending traffic to that resource |
| **All weights = 0** | All records are returned equally (special case) |
| **Alias record** | ✅ Supported |
| **Same name required** | ✅ All weighted records must share the same record name and type |
| **Record Set ID** | Each weighted record needs a unique **Set ID** to differentiate them |

---

## Special Weight Behaviors

### Weight = 0

Setting a resource's weight to **0** completely stops traffic to it. Route 53 will never return that record.

### All Weights = 0

If **every** record for a name has weight 0, Route 53 treats them as **equally weighted** — all records are returned with equal probability. This is a special edge case.

### Health Check Integration

If a weighted record's health check fails, Route 53 **removes it from consideration** and redistributes traffic proportionally among the remaining healthy records.

```
Before failure:                    After EC2 #3 unhealthy:
  EC2 #1: 70% (weight 70)          EC2 #1: 70/(70+20) = 77.8%
  EC2 #2: 20% (weight 20)          EC2 #2: 20/(70+20) = 22.2%
  EC2 #3: 10% (weight 10)          EC2 #3: ❌ removed
```

---

## Use Cases

### 1. Canary Deployments

Send a small percentage of traffic (e.g., 5%) to a new application version to validate before full rollout.

```
Production (v1):  Weight = 95  →  95% of traffic
Canary (v2):      Weight = 5   →   5% of traffic
```

### 2. A/B Testing

Split traffic between different application versions for experimentation.

### 3. Gradual Region Migration

Incrementally shift traffic from one region to another:

```
Day 1:   us-east-1 = 100,  eu-west-1 = 0
Day 2:   us-east-1 = 80,   eu-west-1 = 20
Day 3:   us-east-1 = 50,   eu-west-1 = 50
Day 7:   us-east-1 = 0,    eu-west-1 = 100
```

### 4. Blue/Green Deployments

Route traffic between blue (current) and green (new) environments with instant weight adjustments.

---

## Weighted vs Other Policies

| Scenario | Best Policy |
|---|---|
| "Send 10% traffic to new version" | **Weighted** ✅ |
| "Route to fastest region" | [[Route 53 Latency Routing|Latency]] |
| "Users in France → EU server" | [[Route 53 Geolocation Routing|Geolocation]] |
| "Shift traffic to a new region gradually using geographic zones" | [[Route 53 Geoproximity Routing|Geoproximity]] |

> [!TIP]
> **Exam Pattern:** Any question mentioning "percentage of traffic", "canary deployment", "A/B testing", "split traffic between versions", or "gradual migration" → **Weighted Routing**.

> [!WARNING]
> **Weighted ≠ Geoproximity.** Weighted splits by percentage globally. Geoproximity shifts traffic based on geographic proximity with bias. If the question mentions "regions" and "shifting traffic geographically," think Geoproximity. If it mentions "percentages" or "versions," think Weighted.
