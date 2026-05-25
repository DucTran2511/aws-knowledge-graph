---
tags: [concept, networking, dns, route53, health-checks, high-availability]
aliases: [Route 53 Health Check, DNS Health Check, Calculated Health Check]
date: 2026-05-04
---

# Route 53 Health Checks

*Parent article: [[Amazon Route 53]]*

**Route 53 Health Checks** monitor the health of your endpoints and integrate with routing policies to automatically stop routing traffic to unhealthy resources. They are essential for building highly available DNS architectures.

---

## Types of Health Checks

### 1. Endpoint Health Checks

Monitor a specific **IP address** or **domain name** by sending requests to it.

| Setting | Default | Options |
|---|---|---|
| **Protocol** | HTTP | HTTP, HTTPS, TCP |
| **Interval** | 30 seconds | 30s (standard) or 10s (fast — extra cost) |
| **Failure threshold** | 3 | 1–10 consecutive failures to mark unhealthy |
| **Health checkers** | ~15 global locations | Cannot customize locations |
| **Healthy threshold** | 18% | Resource is healthy if ≥ 18% of checkers report healthy |

#### HTTP/HTTPS Health Checks

- Route 53 sends a `GET` request to your endpoint.
- The health check **passes** if the endpoint returns a `2xx` or `3xx` HTTP status code.
- **String matching (optional):** Route 53 can check if the response body contains a specific text string. It inspects the **first 5,120 bytes** of the response body.

```
Route 53 Health Checker ──▶ GET http://10.0.0.1/health
                           ◀── 200 OK + body: "status: healthy"
                           
                           ✅ Pass (status 2xx + string found in first 5120 bytes)
```

#### TCP Health Checks

- Route 53 attempts to establish a **TCP connection** on the specified port.
- If the connection succeeds within 10 seconds → healthy.
- No HTTP request is sent — just a TCP handshake.

> [!WARNING]
> **Firewall Requirement:** Your security groups and network ACLs **must allow inbound traffic** from Route 53 health checker IP addresses. AWS publishes these IP ranges. If your firewall blocks them, health checks will always fail.

---

### 2. Calculated Health Checks

Combine **multiple child health checks** into a single parent health check using boolean logic.

```
              Calculated Health Check (Parent)
              "Healthy if ≥ 2 of 3 children are healthy"
                          │
            ┌─────────────┼─────────────┐
            │             │             │
      Child HC #1    Child HC #2    Child HC #3
      (Web Server)   (Database)    (Cache)
         ✅              ✅            ❌
         
      2 of 3 healthy ≥ threshold of 2 → Parent: ✅ HEALTHY
```

#### Configuration Options

| Setting | Values |
|---|---|
| **Combine with** | AND, OR, or NOT |
| **Threshold** | "At least X of Y child checks must be healthy" |
| **Max children** | Up to **256** child health checks |

#### Use Cases

- **Composite application health:** Only fail over when multiple components are down (not just one).
- **Maintenance windows:** Temporarily disable individual child health checks without triggering a full failover.
- **Complex architectures:** Combine web server, database, and cache health into one check.

---

### 3. CloudWatch Alarm Health Checks

Monitor the state of a **CloudWatch Alarm** instead of directly checking an endpoint. The health check mirrors the alarm state.

| CloudWatch Alarm State | Health Check Status |
|---|---|
| **OK** | ✅ Healthy |
| **ALARM** | ❌ Unhealthy |
| **INSUFFICIENT_DATA** | Configurable (healthy or unhealthy) |

#### Why This Exists — The Private Resource Problem

> [!CAUTION]
> Route 53 health checkers are **public internet servers**. They **cannot** access resources in private [[VPC]] subnets, private IPs, or on-premises servers behind a firewall that blocks external access.

**The workaround:**

```
Private Resource        CloudWatch            CloudWatch           Route 53
(private subnet)        Metric                Alarm                Health Check
       │                    │                    │                     │
       │─── push metric ──▶│                    │                     │
       │                    │─── triggers ─────▶│                     │
       │                    │                    │─── state change ──▶│
       │                    │                    │                     │
       │                    │              ALARM state = ❌ Unhealthy  │
```

**Steps:**
1. Create a **CloudWatch Metric** on the private resource (e.g., CPU, custom health metric).
2. Create a **CloudWatch Alarm** on that metric.
3. Create a Route 53 **Health Check** that monitors the CloudWatch Alarm.
4. Associate the health check with your DNS routing policy.

> [!TIP]
> **Exam Pattern:** "Monitor health of a private resource for DNS failover" or "health check for resource in private subnet" → **CloudWatch Alarm Health Check**. This is the ONLY way to health-check private resources with Route 53.

---

## Health Check Integration with Routing Policies

| Routing Policy | Health Check Support | Behavior When Unhealthy |
|---|---|---|
| **[[Route 53 Simple Routing\|Simple]]** | ❌ Not supported | Unhealthy targets still returned |
| **[[Route 53 Weighted Routing\|Weighted]]** | ✅ | Traffic redistributed to healthy records |
| **[[Route 53 Latency Routing\|Latency]]** | ✅ | Next lowest-latency healthy region |
| **[[Route 53 Failover Routing\|Failover]]** | ✅ (mandatory on primary) | Switches to Secondary |
| **[[Route 53 Geolocation Routing\|Geolocation]]** | ✅ | Falls back through location hierarchy → Default |
| **[[Route 53 Geoproximity Routing\|Geoproximity]]** | ✅ | Next closest healthy resource |
| **[[Route 53 Multi-Value Routing\|Multi-Value]]** | ✅ | Removed from DNS response |
| **[[Route 53 IP-Based Routing\|IP-Based]]** | ✅ | Default location record |

---

## Health Check Timing

Understanding the timing is important for estimating failover speed:

```
Detection Time = Health Check Interval × Failure Threshold

Standard: 30s × 3 = 90 seconds to detect failure
Fast:     10s × 3 = 30 seconds to detect failure

Total Failover Time = Detection Time + DNS TTL propagation
```

> [!IMPORTANT]
> **Failover is NOT instantaneous.** Even after Route 53 detects a failure, clients may still use the **cached DNS response** until the TTL expires. To minimize failover time:
> 1. Use **Fast health checks** (10s interval — additional cost).
> 2. Set a **low failure threshold** (e.g., 2 instead of 3).
> 3. Use a **low TTL** on DNS records (e.g., 60 seconds).

---

## Monitoring Health Checks

- Health check status is visible in the **Route 53 console**.
- Route 53 publishes health check metrics to **CloudWatch** (you can alarm on them).
- Health check status changes can trigger **SNS notifications** for alerting.

---

## Quick Reference: Exam Patterns

| Scenario | Answer |
|---|---|
| Health check a public web server | **Endpoint Health Check** (HTTP/HTTPS) |
| Health check a private database in VPC | **CloudWatch Alarm Health Check** |
| Fail over only when multiple services are down | **Calculated Health Check** (threshold) |
| Perform maintenance without triggering failover | **Calculated Health Check** (disable one child) |
| Check if response contains specific text | **Endpoint Health Check with string matching** (first 5120 bytes) |
| Allow Route 53 health checkers through firewall | Update **Security Groups / NACLs** with Route 53 IP ranges |
