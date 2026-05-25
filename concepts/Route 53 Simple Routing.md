---
tags: [concept, networking, dns, route53, routing-policy]
aliases: [Simple Routing, Simple Routing Policy]
date: 2026-05-04
---

# Route 53 Simple Routing

*Parent article: [[Amazon Route 53]]*

**Simple Routing** is the most basic [[Amazon Route 53]] routing policy. Use it when you have a **single resource** that performs a given function for your domain (e.g., one web server serving content for `example.com`).

---

## How It Works

1. You create a DNS record (e.g., A record) for your domain.
2. You specify **one or more IP addresses** (values) in that single record.
3. When a client queries the domain, Route 53 returns **all values** in the record.
4. The **client** picks one of the returned values at random (client-side selection).

```
Client ──▶ Route 53 (Simple)
              │
              │  Returns ALL values:
              │  ┌──────────────┐
              │  │ 11.22.33.44  │
              │  │ 55.66.77.88  │
              │  │ 99.00.11.22  │
              │  └──────────────┘
              │
              ▼
         Client picks ONE at random
```

---

## Key Characteristics

| Feature | Detail |
|---|---|
| **Multiple values** | ✅ Can specify multiple IPs/values in one record |
| **Health checks** | ❌ **Cannot** associate health checks |
| **Alias record** | ✅ Can use Alias — but only to **one** AWS resource |
| **Multiple records** | ❌ Only **one** record per name (cannot create multiple Simple records for the same name) |
| **Failover** | ❌ No automatic failover — unhealthy targets are still returned |

---

## When to Use

- You have a **single resource** (one [[EC2]] instance, one [[Elastic Load Balancer (ELB)|ELB]]).
- You want the **simplest** DNS configuration with no routing logic.
- Health checking is **not required** at the DNS level (handled elsewhere, e.g., by an ELB).

---

## When NOT to Use

- You need health checks at the DNS level → use **[[Route 53 Multi-Value Routing|Multi-Value]]** instead.
- You need traffic distribution control → use **[[Route 53 Weighted Routing|Weighted]]**.
- You need failover → use **[[Route 53 Failover Routing|Failover]]**.

---

## Simple vs Multi-Value Answer

This is a common exam comparison:

| Feature | Simple | Multi-Value |
|---|---|---|
| Returns multiple values | ✅ Yes (all values) | ✅ Yes (up to 8) |
| Health checks | ❌ No | ✅ Yes |
| Unhealthy targets returned? | ✅ Yes (all values always returned) | ❌ No (only healthy) |
| Multiple records for same name | ❌ No (one record) | ✅ Yes (one record per resource) |
| Use case | Basic, single-resource | Client-side LB with health filtering |

> [!CAUTION]
> **Exam Trap:** Simple routing with multiple values is NOT the same as Multi-Value routing. Simple returns **all** values including unhealthy ones. Multi-Value only returns **healthy** records. If the question mentions health checks, the answer is Multi-Value, not Simple.

> [!TIP]
> **Exam Pattern:** "Single web server", "no health check needed at DNS", "simplest configuration" → **Simple Routing**.
