---
tags: [concept, networking, global, high-availability, performance]
aliases: [Global Accelerator, GA, AWS Global Accelerator, AGA]
date: 2026-05-18
---

# AWS Global Accelerator

**AWS Global Accelerator** is a networking service that improves the **availability**, **performance**, and **security** of your applications by directing user traffic through the **AWS global network** instead of the public internet. It provides **two static Anycast IP addresses** that serve as a fixed entry point to your application, regardless of the AWS Region(s) where your endpoints reside.

> [!IMPORTANT]
> **Core exam concept:** Global Accelerator is NOT a CDN — it does NOT cache content. It is a **network-layer optimization** service that routes traffic over the AWS private backbone network to the optimal healthy endpoint, reducing latency, jitter, and packet loss compared to the public internet.

---

## The Problem Global Accelerator Solves

Without Global Accelerator, user traffic traverses the **public internet** through many hops (ISPs, peering points, etc.), which introduces latency, jitter, and unreliability — especially for global users.

```
  WITHOUT Global Accelerator:

  User (Australia) ──► ISP ──► Peering ──► ISP ──► Peering ──► ... ──► ALB (us-east-1)
                       │          │          │          │
                    Many hops over public internet = variable latency, packet loss
                    
  WITH Global Accelerator:

  User (Australia) ──► Nearest Edge Location ══════════════════════► ALB (us-east-1)
                       │                                             │
                    Short public hop        AWS Private Backbone     Endpoint
                    to nearest edge      (fast, reliable, optimized)
```

The key insight: users only travel over the **public internet** for a short hop to the **nearest AWS edge location** (same 400+ edge locations used by CloudFront). From there, traffic flows over AWS's **private global network**, which is faster, more reliable, and less congested.

> [!TIP]
> **Exam Pattern:** "Improve performance for global users accessing an application in a single region" → **Global Accelerator**. It reduces latency by leveraging the AWS private backbone network instead of the public internet.

---

## How Global Accelerator Works

```
                          AWS Global Network (Private Backbone)
                    ┌──────────────────────────────────────────────┐
                    │                                              │
  User (Sydney)     │    Edge Location       ═══► Endpoint Group   │
  ──────────►  ┌────┤    (Sydney)                 (us-east-1)     │
               │    │        │                     ├── ALB         │
  User (Tokyo) │    │    Edge Location       ═══► │  ├── EC2       │
  ──────────►  ┤    │    (Tokyo)                   │  └── EIP      │
               │    │        │                     │               │
  User (Paris) │    │    Edge Location       ═══► Endpoint Group   │
  ──────────►  └────┤    (Paris)                  (eu-west-1)     │
                    │                              ├── ALB         │
                    │                              └── NLB         │
                    └──────────────────────────────────────────────┘
                    
  1. Users connect to nearest edge location via Anycast IPs
  2. Traffic enters AWS private backbone at the edge
  3. Global Accelerator routes to the optimal endpoint group (Region)
  4. Traffic is delivered to the healthiest endpoint in that group
```

### Key Components

| Component | Description |
|---|---|
| **Accelerator** | The Global Accelerator resource. Directs traffic to optimal endpoints. Provides 2 static Anycast IPs. |
| **Static Anycast IPs** | Two fixed IPv4 addresses that are advertised from multiple AWS edge locations simultaneously. Clients connect to the nearest edge. |
| **Listener** | Processes inbound connections based on **port** and **protocol** (TCP, UDP, or both). One accelerator can have multiple listeners. |
| **Endpoint Group** | Associated with a specific **AWS Region**. Contains one or more endpoints. You configure traffic dials (0–100%) per group. |
| **Endpoint** | The actual resource receiving traffic: **ALB**, **NLB**, **EC2 instance**, or **Elastic IP address**. |

---

## Static Anycast IP Addresses

Global Accelerator provides **2 static IPv4 addresses** that act as a single, fixed entry point for your application globally.

```
  Traditional Setup:              Global Accelerator:

  Region 1: ALB → 54.23.45.67    Anycast IP 1: 75.2.1.10   ──► ALB (us-east-1)
  Region 2: ALB → 52.11.22.33    Anycast IP 2: 99.83.2.20  ──► ALB (eu-west-1)
  Region 3: ALB → 13.44.55.66                               ──► ALB (ap-southeast-1)
  
  Problem: IPs change if you       Benefit: IPs NEVER change.
  replace an ALB or add regions.    Works across all regions.
  Clients must update DNS.          Zero client-side changes.
```

### Why Anycast Matters

- **Anycast** = the same IP address is advertised from **multiple locations** simultaneously.
- The internet's BGP routing automatically sends each user to the **nearest** location advertising that IP.
- Users **always** reach the closest AWS edge location — no DNS resolution delay, no DNS caching issues.

> [!IMPORTANT]
> The 2 static IPs are **globally unique** and provided by AWS. You can also **bring your own IP addresses (BYOIP)** if you need to use IPs already whitelisted by your customers.

---

## Listeners

A listener processes inbound connections from clients to your accelerator based on port and protocol configuration.

| Setting | Options |
|---|---|
| **Ports** | One or more ports (1–65535). Can map port ranges. |
| **Protocol** | TCP, UDP, or TCP + UDP |
| **Client Affinity** | None (default) or Source IP (sticky — same client always goes to same endpoint) |

You can configure **multiple listeners** on a single accelerator to handle different ports or protocols.

```
  Accelerator (75.2.1.10, 99.83.2.20)
  │
  ├── Listener 1: TCP Port 80, 443  ──► Endpoint Group (us-east-1)
  │                                     └── ALB (web traffic)
  │
  └── Listener 2: UDP Port 9000     ──► Endpoint Group (us-east-1)
                                        └── EC2 (game server)
```

> [!NOTE]
> **Client Affinity:** By default, Global Accelerator distributes traffic across endpoints. Set client affinity to **Source IP** when you need a specific client to always reach the same endpoint (e.g., stateful applications, gaming).

---

## Endpoint Groups

Each listener has one or more endpoint groups, and **each endpoint group is associated with a specific AWS Region**.

### Traffic Dial

The **traffic dial** (0–100%) controls the **percentage of traffic** directed to an endpoint group. This is useful for:
- **Blue/green deployments** — shift traffic gradually to a new region.
- **Performance testing** — send a small percentage to a new region.
- **Failover testing** — dial down one region to simulate failure.

```
  Listener (TCP 443)
  │
  ├── Endpoint Group: us-east-1  (Traffic Dial: 70%)
  │   └── ALB
  │
  └── Endpoint Group: eu-west-1  (Traffic Dial: 30%)
      └── ALB

  Gradually shift from 70/30 → 50/50 → 0/100 for migration
```

### Health Check Settings

Each endpoint group has configurable health checks:

| Setting | Default |
|---|---|
| **Health check port** | The listener port |
| **Health check protocol** | TCP, HTTP, or HTTPS |
| **Health check path** | `/` (for HTTP/HTTPS) |
| **Health check interval** | 30 seconds |
| **Threshold** | 3 consecutive checks to mark healthy/unhealthy |

---

## Endpoints

Endpoints are the AWS resources that receive traffic from Global Accelerator.

### Supported Endpoint Types

| Endpoint Type | Notes |
|---|---|
| **Application Load Balancer (ALB)** | Can be internet-facing or internal |
| **Network Load Balancer (NLB)** | Can be internet-facing or internal |
| **EC2 Instance** | Must have a public or Elastic IP |
| **Elastic IP Address** | Static IP for EC2, NLB, or other resources |

### Endpoint Weights

Each endpoint has a **weight** (0–255) that determines the proportion of traffic it receives within its endpoint group.

```
  Endpoint Group (us-east-1)
  │
  ├── ALB-1  (Weight: 200)  ──► 200/255 ≈ 78% of traffic
  └── ALB-2  (Weight: 55)   ──► 55/255  ≈ 22% of traffic
```

A weight of **0** means no traffic is sent to that endpoint.

> [!TIP]
> **Traffic Dial** controls traffic **between regions** (endpoint groups). **Endpoint Weights** control traffic **within a region** (between endpoints in the same group).

---

## Health Checking & Failover

Global Accelerator performs **automatic health checks** on all endpoints and routes traffic only to healthy ones. This provides automatic failover with **less than 1 minute** failover time.

```
  Normal Operation:                     After Failover:

  User ──► Edge ══► us-east-1 (healthy) User ──► Edge ══► eu-west-1 (healthy)
                    └── ALB ✓                              └── ALB ✓
                                         
           ══► eu-west-1 (healthy)                 ✗ us-east-1 (unhealthy)
               └── ALB ✓                              └── ALB ✗
                                         
  Traffic goes to nearest healthy       Traffic automatically rerouted
  region by default                     to next best healthy region
```

### Failover Behavior

1. Global Accelerator **continuously monitors** endpoint health.
2. If all endpoints in an endpoint group become unhealthy → traffic **automatically fails over** to the next closest healthy endpoint group.
3. When the original endpoint recovers → traffic **automatically fails back** (no manual intervention needed).
4. **Failover time:** Typically **< 30 seconds** for health check detection + rerouting. Often cited as instantaneous due to Anycast.

> [!IMPORTANT]
> Global Accelerator failover is **DNS-independent** — it doesn't rely on DNS TTL expiration. Because it uses Anycast IPs, traffic rerouting happens at the **network level** and takes effect in seconds, not minutes. This is a major advantage over DNS-based failover (Route 53).

---

## Global Accelerator vs CloudFront

This is a **critical exam comparison**. Both use the AWS edge network but serve fundamentally different purposes.

| Feature | Global Accelerator | CloudFront |
|---|---|---|
| **Type** | Network-layer acceleration | Content Delivery Network (CDN) |
| **Caching** | ❌ NO caching | ✅ Caches content at edge |
| **Static IPs** | ✅ 2 static Anycast IPs | ❌ No static IPs (uses DNS) |
| **Protocols** | TCP and UDP | HTTP and HTTPS only |
| **Use case** | Non-HTTP (gaming, IoT, VoIP), TCP/UDP apps, static IP requirements | Static content caching, web acceleration |
| **How it works** | Proxies traffic over AWS backbone to the endpoint | Caches content at edge, serves from cache |
| **Health checks** | ✅ Built-in endpoint health checks + auto-failover | Via Origin Groups (failover between origins) |
| **Edge behavior** | Routes/proxies packets | Caches and serves content |
| **Best for** | Dynamic content, gaming, financial trading, IoT | Static content, video streaming, API acceleration |

```
  CloudFront:                           Global Accelerator:

  User ──► Edge ──► Cache HIT ──► User  User ──► Edge ════════════► Endpoint
                │                                (no caching)
                └── Cache MISS ──► Origin
                    (fetches, caches)

  CDN = caches content at edge          GA = routes traffic through backbone
  Best for: static, cacheable content   Best for: dynamic, non-cacheable, TCP/UDP
```

> [!CAUTION]
> **Exam Trap:** If the question mentions **caching**, **static content**, or **HTTP/HTTPS** → answer is likely **CloudFront**. If it mentions **static IPs**, **UDP/TCP**, **gaming**, **IoT**, **VoIP**, or **non-HTTP traffic** → answer is **Global Accelerator**.

### When to Use Each (or Both)

| Scenario | Service |
|---|---|
| Serve static images to global users | **CloudFront** |
| Low-latency gaming server (UDP) | **Global Accelerator** |
| API with dynamic responses, global users | **Either** (CloudFront with TTL=0, or GA) |
| Application requires fixed IPs for whitelisting | **Global Accelerator** |
| DDoS protection for web application | **CloudFront + Shield + WAF** |
| DDoS protection for TCP/UDP application | **Global Accelerator + Shield Advanced** |
| Multi-region active-active with instant failover | **Global Accelerator** |
| Content localization with edge compute | **CloudFront + Lambda@Edge** |

---

## Global Accelerator vs Route 53 (DNS-Based Failover)

| Feature | Global Accelerator | Route 53 Failover |
|---|---|---|
| **Failover speed** | Seconds (network-level, Anycast) | Minutes (depends on DNS TTL) |
| **Static IPs** | ✅ Yes (2 Anycast IPs) | ❌ No (IPs change with DNS) |
| **Protocol support** | TCP, UDP | Any (DNS-based, protocol agnostic) |
| **Client caching** | No impact (Anycast routing) | TTL-dependent (cached DNS answers cause delay) |
| **Health checks** | Built-in, continuous | Route 53 health checks |
| **Cost** | Higher (per-accelerator + data transfer) | Lower (DNS query charges only) |

> [!TIP]
> **Exam Pattern:** "Instant failover without DNS TTL issues" → **Global Accelerator**. "Cost-effective DNS failover is acceptable" → **Route 53 Failover Routing**.

---

## Security

### DDoS Protection

- **AWS Shield Standard** is automatically applied to Global Accelerator (free).
- **AWS Shield Advanced** can be enabled for enhanced DDoS protection with 24/7 DRT (DDoS Response Team) support.
- Because traffic enters the AWS network at the edge, DDoS attacks are mitigated close to the source.

### Client IP Preservation

Global Accelerator can **preserve the original client IP address** for certain endpoint types:

| Endpoint Type | Client IP Preserved? |
|---|---|
| **ALB** | ✅ Yes (in `X-Forwarded-For` header) |
| **NLB** | ✅ Yes (via Proxy Protocol v2 or client IP preservation) |
| **EC2** | ✅ Yes |
| **Elastic IP** | ❌ No (source IP = Global Accelerator) |

> [!NOTE]
> Client IP preservation is important for logging, analytics, security rules, and compliance. When enabled, your application sees the **real client IP** instead of the Global Accelerator IP.

### Security Groups and NACLs

- Endpoints behind ALB/NLB use the standard security group and NACL rules.
- For EC2 endpoints: security groups must allow traffic from Global Accelerator's IP ranges.

---

## Architecture Patterns

### Pattern 1: Multi-Region Active-Active

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    Global Accelerator                        │
  │              Anycast IPs: 75.2.1.10, 99.83.2.20            │
  └─────────┬──────────────────────┬─────────────────┬──────────┘
            │                      │                 │
            ▼                      ▼                 ▼
  ┌──────────────┐      ┌──────────────┐   ┌──────────────┐
  │  us-east-1   │      │  eu-west-1   │   │ap-southeast-1│
  │  ┌────────┐  │      │  ┌────────┐  │   │  ┌────────┐  │
  │  │  ALB   │  │      │  │  ALB   │  │   │  │  ALB   │  │
  │  └───┬────┘  │      │  └───┬────┘  │   │  └───┬────┘  │
  │  ┌───▼────┐  │      │  ┌───▼────┐  │   │  ┌───▼────┐  │
  │  │  EC2   │  │      │  │  EC2   │  │   │  │  EC2   │  │
  │  │ fleet  │  │      │  │ fleet  │  │   │  │ fleet  │  │
  │  └────────┘  │      │  └────────┘  │   │  └────────┘  │
  └──────────────┘      └──────────────┘   └──────────────┘

  → Users routed to nearest healthy region automatically
  → Failover in seconds if a region goes down
  → Single pair of static IPs for all regions
```

### Pattern 2: Blue/Green Deployment Across Regions

```
  Global Accelerator
  │
  ├── Endpoint Group: us-east-1 (Blue)  ─── Traffic Dial: 90%
  │   └── ALB (current production)
  │
  └── Endpoint Group: us-west-2 (Green) ─── Traffic Dial: 10%
      └── ALB (new version)

  Gradually shift traffic dial: 90/10 → 70/30 → 50/50 → 0/100
```

### Pattern 3: Static IP for Whitelisting

```
  Corporate Firewall Rules:
  ┌──────────────────────────┐
  │ Allow outbound to:       │
  │  • 75.2.1.10            │    ◄── Fixed! Never changes.
  │  • 99.83.2.20           │        Even if you add regions,
  └──────────────────────────┘        change ALBs, or failover.
            │
            ▼
  Global Accelerator ══► ALB (any region)

  Without GA: Firewall rules must be updated every time
  the ALB IP changes (ALBs have dynamic IPs!)
```

---

## Pricing

| Charge | Rate (approx.) |
|---|---|
| **Fixed fee** | ~$0.025/hour per accelerator (~$18/month) |
| **Data Transfer Premium** | DT-Premium per GB (varies by source-destination regions). On top of standard EC2/ALB data transfer. |

> [!NOTE]
> Global Accelerator adds a **premium** on top of normal data transfer costs. It is not free. Factor this into cost-optimization questions — if the question emphasizes cost savings and latency improvement for static content, **CloudFront** is usually cheaper.

---

## Integration with Other Services

| Service | Integration |
|---|---|
| **[[Amazon Route 53\|Route 53]]** | Alias records can point to Global Accelerator (zone apex support) |
| **[[Amazon CloudFront\|CloudFront]]** | Can be used together — GA for TCP/UDP, CloudFront for HTTP caching |
| **[[Elastic Load Balancer (ELB)\|ELB]]** | ALB and NLB are supported endpoints |
| **AWS Shield** | Shield Standard included; Shield Advanced available |
| **AWS CloudFormation** | Full support for infrastructure-as-code provisioning |
| **AWS CloudWatch** | Metrics for data flow, health check status, connection counts |

---

## Custom Routing Accelerators

In addition to the standard accelerator, Global Accelerator supports **Custom Routing Accelerators** for use cases where you need to **deterministically route traffic to specific EC2 instances** within a VPC subnet.

| Feature | Standard Accelerator | Custom Routing Accelerator |
|---|---|---|
| **Routing logic** | Automatic (nearest healthy endpoint) | Deterministic (map user to specific instance) |
| **Use case** | General apps, APIs, web traffic | Gaming (match users to game servers), VoIP |
| **Endpoints** | ALB, NLB, EC2, EIP | VPC subnets (EC2 instances within) |
| **Port mapping** | Listener ports map to endpoint ports | Unique port mapping per EC2 instance |

```
  Custom Routing:

  Player A ──► GA (port 19001) ══► EC2 Instance 1 (Game Server A)
  Player B ──► GA (port 19002) ══► EC2 Instance 2 (Game Server B)
  Player C ──► GA (port 19003) ══► EC2 Instance 3 (Game Server C)

  Each player is routed to a SPECIFIC instance, not load-balanced
```

> [!TIP]
> **Exam Pattern:** "Route users to specific backend instances" (e.g., multiplayer gaming matchmaking) → **Custom Routing Accelerator**.

---

## Quick Reference: Exam Patterns

> [!TIP]
> **Trigger phrases → Global Accelerator:**
> - "Improve performance for global users (non-cacheable content)" → **Global Accelerator**
> - "Static/fixed IP addresses for application" → **Global Accelerator**
> - "Instant failover without DNS TTL delays" → **Global Accelerator**
> - "UDP or TCP traffic optimization" → **Global Accelerator**
> - "Gaming, IoT, VoIP with low latency" → **Global Accelerator**
> - "Whitelist a fixed IP for a multi-region app" → **Global Accelerator**
> - "Blue/green deployment with traffic shifting across regions" → **Global Accelerator Traffic Dial**
> - "Route each player to a specific game server" → **Custom Routing Accelerator**
> - "Reduce jitter and packet loss for global traffic" → **Global Accelerator**
> - "DDoS protection for non-HTTP application" → **Global Accelerator + Shield Advanced**
>
> **Key facts:**
> - Provides **2 static Anycast IPv4 addresses** (or bring your own — BYOIP).
> - Uses the **AWS global network** (400+ edge locations — same as CloudFront).
> - **Does NOT cache** — it is NOT a CDN.
> - Supports **TCP and UDP** protocols.
> - Endpoint types: **ALB, NLB, EC2, Elastic IP**.
> - Failover time: **< 30 seconds** (network-level, DNS-independent).
> - **Traffic Dial** = control traffic between regions (0–100%).
> - **Endpoint Weight** = control traffic within a region (0–255).
> - **Client IP preservation** supported for ALB, NLB, and EC2 endpoints.
> - AWS Shield Standard is **included free**.
> - Route 53 supports **Alias records** pointing to Global Accelerator.
> - Cost: fixed hourly fee + data transfer premium (more expensive than CloudFront for cacheable content).

| Scenario | Answer |
|---|---|
| Static content globally with caching | **CloudFront** (not GA) |
| Non-cacheable API for global users + need static IPs | **Global Accelerator** |
| UDP gaming server, low latency worldwide | **Global Accelerator** |
| Instant multi-region failover without DNS delays | **Global Accelerator** |
| Firewall whitelisting for a multi-region app | **Global Accelerator** (fixed IPs) |
| Reduce latency for HTTP content by caching at edge | **CloudFront** (not GA) |
| Blue/green across AWS regions | **Global Accelerator Traffic Dial** |
| Route players to specific game server instances | **Custom Routing Accelerator** |
| 100% availability SLA for DNS | **Route 53** (not GA) |
| DDoS protection for UDP app | **Global Accelerator + Shield Advanced** |
