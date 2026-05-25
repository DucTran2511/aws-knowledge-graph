---
tags: [concept, networking, dns, route53, routing-policy]
aliases: [Multi-Value Routing, Multi-Value Answer Routing, Multivalue Routing]
date: 2026-05-04
---

# Route 53 Multi-Value Routing

*Parent article: [[Amazon Route 53]]*

**Multi-Value Answer Routing** returns **multiple healthy records** (up to 8) in response to a DNS query. Each record can have an associated health check, and Route 53 only returns records that are currently **healthy**. This provides client-side load balancing with DNS-level health filtering.

---

## How It Works

1. You create **multiple records** with the same name (up to 8).
2. Each record points to a different resource and has its own **health check**.
3. When a client queries, Route 53 returns **up to 8 healthy records**.
4. The client picks one of the returned records (typically at random).
5. If a resource's health check fails, Route 53 **removes it** from the response.

```
Client ──▶ Route 53 (Multi-Value)
              │
              │ Checks health of all records:
              │
              │  10.0.0.1 ✅ healthy → included
              │  10.0.0.2 ✅ healthy → included
              │  10.0.0.3 ❌ unhealthy → excluded
              │  10.0.0.4 ✅ healthy → included
              │
              │  Returns: [10.0.0.1, 10.0.0.2, 10.0.0.4]
              │
              ▼
         Client picks ONE at random from healthy set
```

---

## Key Characteristics

| Feature | Detail |
|---|---|
| **Max records returned** | Up to **8** healthy records per query |
| **Health checks** | ✅ Supported (and recommended — the entire point of this policy) |
| **Unhealthy records** | ❌ **Not returned** in the response |
| **No health check attached** | Record is **always** considered healthy and always returned |
| **Alias record** | ✅ Supported |
| **Record Set ID** | Each multi-value record needs a unique Set ID |

---

## Multi-Value vs Simple Routing

This is the **most common exam comparison** for this policy:

| Feature | Simple | Multi-Value |
|---|---|---|
| Multiple values | ✅ One record with multiple values | ✅ Multiple records (one per resource) |
| Health checks | ❌ **Not supported** | ✅ **Supported** |
| Unhealthy resources | ✅ Still returned (all values always returned) | ❌ **Filtered out** |
| Max returned | Unlimited (all values in the record) | Up to **8** |
| Record structure | 1 record, multiple values | Multiple records, 1 value each |

```
Simple Routing:                          Multi-Value Routing:
┌─────────────────────┐                  ┌─────────────────────┐
│ Record: app.example  │                  │ Record 1: app.example│ ← Health Check ✅
│ Values:              │                  │ Value: 10.0.0.1      │
│   10.0.0.1           │                  ├─────────────────────┤
│   10.0.0.2 (down!)   │                  │ Record 2: app.example│ ← Health Check ❌
│   10.0.0.3           │                  │ Value: 10.0.0.2      │ ← NOT returned
│                      │                  ├─────────────────────┤
│ Returns ALL 3 ❌      │                  │ Record 3: app.example│ ← Health Check ✅
└─────────────────────┘                  │ Value: 10.0.0.3      │
                                         │                     │
                                         │ Returns 1 & 3 only ✅│
                                         └─────────────────────┘
```

> [!CAUTION]
> **Exam Trap:** Multi-Value Answer is **NOT a substitute for a load balancer.** It provides client-side DNS-based selection — not true load balancing. The ELB provides real-time traffic distribution, health checks at the connection level, and features like sticky sessions. Multi-Value is a lightweight DNS-level alternative when you don't need (or can't use) an ELB.

---

## Use Cases

### 1. Simple Multi-Server Setup Without ELB

If you have a few [[EC2]] instances and don't want the cost/complexity of an [[Elastic Load Balancer (ELB)|ELB]], Multi-Value gives basic distribution with health filtering.

### 2. Hybrid Cloud Load Distribution

Distribute traffic across AWS and on-premises servers by IP, with health checks ensuring only reachable endpoints are returned.

### 3. DNS-Level Resilience

When combined with low TTL, provides fast DNS-level failover without dedicated failover infrastructure.

---

## Limitations

- **Not real load balancing** — clients choose randomly, no consideration of load or capacity.
- **DNS caching** — clients may cache an IP and keep using it even after the health status changes (mitigated with low TTL).
- **Max 8 records** — cannot distribute across more than 8 endpoints per DNS name.
- **No traffic weighting** — all healthy records are returned equally (use [[Route 53 Weighted Routing|Weighted]] for proportional control).

> [!TIP]
> **Exam Pattern:** "Return multiple healthy IPs", "client-side load balancing with health checks", "DNS-level health filtering" → **Multi-Value Answer Routing**. If the question simply says "multiple values" without mentioning health checks → that could be **Simple Routing** (check carefully!).
