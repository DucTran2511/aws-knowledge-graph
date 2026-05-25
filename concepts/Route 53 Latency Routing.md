---
tags: [concept, networking, dns, route53, routing-policy]
aliases: [Latency Routing, Latency-Based Routing, Latency Routing Policy]
date: 2026-05-04
---

# Route 53 Latency Routing

*Parent article: [[Amazon Route 53]]*

**Latency-Based Routing** directs users to the AWS resource that provides the **lowest network latency** (fastest response time). This is the best routing policy for global applications where performance matters most.

---

## How It Works

1. You deploy resources in **multiple AWS Regions** (e.g., [[EC2]] in `us-east-1` and `eu-west-1`).
2. You create a **Latency record** for each resource, specifying which **Region** the resource is in.
3. When a client queries the domain, Route 53 evaluates latency data between the user's location and each Region.
4. Route 53 returns the record for the Region with the **lowest latency** for that user.

```
                    Route 53 (Latency)
                          │
          Measures latency from user to each Region
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
  us-east-1          eu-west-1          ap-southeast-1
  (latency: 45ms)    (latency: 12ms)    (latency: 180ms)
       │                  │                  │
       │              ◀── WINNER ──▶         │
       │            (lowest latency)         │
       │                  │                  │
       ▼                  ▼                  ▼
  ┌──────────┐     ┌──────────┐      ┌──────────┐
  │  ALB #1   │     │  ALB #2   │      │  ALB #3   │
  │  US East  │     │  EU West  │      │  AP SE    │
  └──────────┘     └──────────┘      └──────────┘

  User in Paris → eu-west-1 (12ms) ✅
```

---

## Key Characteristics

| Feature | Detail |
|---|---|
| **Latency measurement** | Based on actual **network latency** between users and AWS Regions, NOT geographic distance |
| **Health checks** | ✅ Can associate health checks — provides automatic failover to next-lowest-latency region |
| **Region-based** | You specify the **AWS Region** for each record; Route 53 maintains latency tables |
| **Dynamic** | Latency data is updated as network conditions change |
| **Alias record** | ✅ Supported |
| **Record Set ID** | Each latency record needs a unique Set ID |

---

## Latency ≠ Geographic Distance

This is a **critical exam concept**. Latency routing does NOT use physical distance — it uses actual measured network performance.

| User Location | Closest Region (geographic) | Lowest Latency Region | Why? |
|---|---|---|---|
| São Paulo, Brazil | sa-east-1 (São Paulo) | **us-east-1 (Virginia)** | Better network peering, more direct routes |
| Mumbai, India | ap-south-1 (Mumbai) | **ap-south-1 (Mumbai)** | Usually matches, but not always |

> [!WARNING]
> **Exam Trap:** Latency is NOT always proportional to geographic distance. Network peering, routing paths, and congestion all affect latency. A geographically further region can have lower latency than a closer one.

---

## Health Check + Failover Behavior

When you associate health checks with Latency records, Route 53 automatically fails over to the **next lowest-latency** healthy resource.

```
Normal operation:                      ALB #2 unhealthy:
  User → eu-west-1 (12ms) ✅           User → us-east-1 (45ms) ✅
                                       eu-west-1 ❌ (removed)
```

This provides **implicit failover** without needing a separate [[Route 53 Failover Routing|Failover]] policy.

---

## Use Cases

### 1. Global Application Performance

Deploy your application in multiple regions and let Route 53 automatically route each user to the fastest endpoint.

### 2. Multi-Region Active-Active Architecture

Combine with health checks for an active-active setup where all regions serve traffic, and unhealthy regions are automatically bypassed.

### 3. Database Read Replicas

Route read traffic to the [[RDS Read Replicas|Read Replica]] in the region closest (by latency) to the user for fastest query response.

---

## Latency vs Geolocation vs Geoproximity

| Feature | Latency | Geolocation | Geoproximity |
|---|---|---|---|
| **Routing basis** | Network latency (ms) | User's country/continent | Physical distance + Bias |
| **Goal** | Fastest response | Compliance / content localization | Shift traffic by geography |
| **Can override?** | No (always picks lowest latency) | Yes (hard rule per location) | Yes (Bias -99 to +99) |
| **Measures real performance?** | ✅ Yes | ❌ No | ❌ No |
| **Health checks** | ✅ | ✅ | ✅ |
| **Requires Traffic Flow** | ❌ | ❌ | ✅ |

> [!TIP]
> **Exam Pattern:** "Minimize latency", "fastest response time for global users", "best performance across regions" → **Latency-Based Routing**. If the question instead says "users in a specific country MUST go to a specific server" → that's [[Route 53 Geolocation Routing|Geolocation]] (compliance, not performance).

> [!IMPORTANT]
> Latency routing requires you to **specify the AWS Region** for each record. Route 53 uses this Region information combined with its internal latency database to make routing decisions.
