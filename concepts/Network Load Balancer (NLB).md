---
tags: [concept, networking, high-availability, load-balancing]
aliases: [NLB, Network Load Balancer]
date: 2026-04-28
---

# Network Load Balancer (NLB)

The **Network Load Balancer** is a Layer 4 (Transport Layer) load balancer in the [[Elastic Load Balancer (ELB)]] family. It is purpose-built for **extreme performance**, handling **millions of requests per second** at **ultra-low latency** (~100 microseconds). If ALB is the "smart" load balancer, NLB is the "fast" one.

## Why NLB Exists

[[Elastic Load Balancer (ELB)|ALB]] operates at Layer 7 and must parse HTTP headers, which adds latency. Many workloads don't use HTTP at all — they use raw TCP, UDP, or TLS. NLB forwards traffic at the connection level without ever reading the payload, making it dramatically faster.

---

## Core Characteristics

### Layer 4 — TCP / UDP / TLS
NLB operates purely at the transport layer. It makes routing decisions based on:
- **IP protocol data** (source/destination IP + port)
- It does **not** inspect HTTP headers, paths, cookies, or query strings — that's [[Elastic Load Balancer (ELB)|ALB]]'s job.

### Static IP & Elastic IP Support
This is NLB's **single most important differentiator** from ALB:
- NLB provisions **one static IP address per [[Availability Zones (AZ)|Availability Zone]]**.
- You can optionally assign your own **[[AWS IP Addressing|Elastic IP]]** to each AZ.
- This gives your load balancer a **fixed, predictable IP address** that never changes.

> [!IMPORTANT]
> **Exam Trigger:** Any question that mentions "whitelist by IP address," "firewall rules require a static IP," or "must provide a fixed IP to a partner" → the answer is **NLB**. ALB only exposes a DNS hostname whose underlying IPs can change at any time.

### Client IP Preservation
Unlike ALB (which puts the real client IP in `X-Forwarded-For`), NLB **preserves the original source IP** of the client all the way to the target. Your backend sees the actual client IP address in the network packet.

> [!WARNING]
> Because the target sees the real client IP, your target's **Security Group must allow traffic from the client IP ranges**, not from the NLB's IP. This is a common gotcha when migrating from ALB to NLB.

---

## Performance Numbers

| Metric | NLB | ALB (for comparison) |
|---|---|---|
| Latency | ~100 **microseconds** | ~400 **milliseconds** |
| Throughput | Millions of requests/sec | Thousands of requests/sec |
| Connection handling | Optimized for long-lived connections | Optimized for HTTP request/response |

NLB can handle **sudden and volatile traffic patterns** without needing to "warm up" — unlike ALB, which may need pre-warming for traffic spikes.

---

## Target Groups

NLB routes traffic to **Target Groups**. Each target group contains one type of target:

### 1. EC2 Instances
- Targets are specified by **Instance ID**.
- NLB sends traffic to the instance's primary private IP.
- The most common target type.

### 2. IP Addresses
- Targets are specified by IP address (must be private IPs).
- Useful for routing to on-premises servers (via [[VPC]] peering, Direct Connect, or VPN), other [[VPC]]s, or specific container IPs.
- The IP addresses **must be from specific CIDR ranges**: RFC 1918 (private), RFC 6598 (shared address space / CGNAT `100.64.0.0/10`), or RFC 5765.

### 3. Application Load Balancer (ALB)
- You can put an NLB **in front of an ALB**.
- **Why?** To get the best of both worlds: NLB's static IP / Elastic IP + ALB's Layer 7 routing intelligence.
- This is a real-world pattern for when partners require a fixed IP but your application needs path-based routing.

```
Client → NLB (Static IP: 52.1.2.3)
            → ALB (path-based routing)
                → /api   → API Target Group
                → /web   → Web Target Group
```

> [!TIP]
> **Exam Pattern:** "The application requires path-based routing AND a static IP" → use **NLB in front of ALB**.

---

## Health Checks

NLB supports three protocols for health checks:

| Protocol | How It Works |
|---|---|
| **TCP** | NLB opens a TCP connection to the target. If the handshake succeeds → healthy. |
| **HTTP** | NLB sends an HTTP GET request to a path you configure (e.g., `/health`). Expects a `200-399` response. |
| **HTTPS** | Same as HTTP but over TLS. |

- Health checks run on a **port** and **protocol** you configure (can differ from the traffic port).
- **Unhealthy threshold**: Number of consecutive failed checks before marking a target as unhealthy (default: 3).
- **Healthy threshold**: Number of consecutive successful checks before marking a target as healthy (default: 3, and for NLB this **must equal** the unhealthy threshold).
- **Interval**: 10 or 30 seconds.

> [!NOTE]
> If health checks fail but your application appears to work, check that the target's **Security Group allows health check traffic** from the NLB's private IPs (or from the client IP ranges if using IP-based targets).

---

## Listener Rules

NLB listeners are simpler than ALB's — you configure:
1. **Protocol**: TCP, TLS, UDP, or TCP_UDP.
2. **Port**: The port the NLB listens on.
3. **Default action**: Forward to a Target Group.

There is **no path-based or host-based routing** — every connection on a given port goes to the same target group (or you use multiple listeners on different ports).

### TLS Listeners (SSL Offloading)
- NLB can terminate TLS connections using an SSL certificate from **AWS Certificate Manager (ACM)**.
- Supports **SNI (Server Name Indication)** for hosting multiple TLS certificates.
- After termination, NLB forwards unencrypted TCP to the backend.
- Alternatively, you can use **TCP passthrough** to let the backend handle TLS directly.

---

## Cross-Zone Load Balancing

| Setting | Default | Cost |
|---|---|---|
| **Cross-Zone** | **Disabled** | You **pay** for inter-AZ data transfer if enabled |

When disabled (default), each NLB node only distributes traffic to targets in its own AZ. This can cause **uneven distribution** if AZs have different numbers of targets.

```
Cross-Zone DISABLED (default):          Cross-Zone ENABLED:
AZ-a node → only AZ-a targets          AZ-a node → all targets in all AZs
AZ-b node → only AZ-b targets          AZ-b node → all targets in all AZs
```

> [!WARNING]
> Enabling cross-zone load balancing on NLB incurs **inter-AZ data transfer charges**. This is different from ALB, where cross-zone is always on and free.

---

## Sticky Sessions

NLB supports stickiness, but it works differently from ALB:
- NLB stickiness is **source IP based** — not cookie-based.
- All connections from the same source IP are routed to the same target for the lifetime of the session.
- Configured at the **Target Group** level.

---

## Connection Draining (Deregistration Delay)

When a target is deregistered or fails a health check:
- NLB **stops sending new connections** to the target.
- Existing connections are allowed to complete within the **deregistration delay** period.
- Default: **300 seconds** (configurable: 0–3600 seconds).
- Set to **0** to terminate connections immediately (useful for stateless backends).

---

## Security Groups

As of 2023, NLB **supports Security Groups** (this was a major update — previously NLB did not have Security Groups).

- You can now attach a Security Group directly to the NLB.
- If no Security Group is attached, the NLB is "open" and the target's Security Group must handle all filtering.
- If a Security Group **is** attached, target Security Groups can reference the **NLB's Security Group** as a source — just like with ALB.

> [!NOTE]
> **Before 2023:** You had to allow all client IPs in the target's Security Group because NLB preserved the source IP and had no SG of its own. Now you can simplify by attaching a SG to the NLB and referencing it in the target's SG.

---

## Common Use Cases

| Use Case | Why NLB? |
|---|---|
| **Gaming servers** | Ultra-low latency, UDP support |
| **IoT backends** | Millions of simultaneous TCP connections |
| **Financial trading platforms** | Microsecond latency matters |
| **VoIP / Media streaming** | UDP protocol support |
| **Static IP requirement** | Partners/firewalls need a fixed IP to whitelist |
| **Non-HTTP protocols** | MQTT, custom TCP, gRPC (without HTTP/2) |
| **NLB → ALB pattern** | Need both static IP AND Layer 7 routing |
| **AWS PrivateLink** | NLB is **required** to expose a service via [[VPC]] endpoint (PrivateLink) |

---

## NLB vs ALB — Decision Cheat Sheet

| Question | ALB | NLB |
|---|---|---|
| Do you need HTTP path/host/header routing? | ✅ | ❌ |
| Do you need a static/Elastic IP? | ❌ | ✅ |
| Is your protocol TCP/UDP (non-HTTP)? | ❌ | ✅ |
| Need extreme performance (millions RPS)? | ❌ | ✅ |
| Need to preserve client source IP natively? | ❌ (uses X-Forwarded-For) | ✅ |
| Need WebSocket support? | ✅ | ✅ (TCP) |
| Need AWS PrivateLink? | ❌ | ✅ (required) |
| Free tier eligible? | ✅ | ❌ |

> [!TIP]
> **PrivateLink Exam Trigger:** "Expose a service to thousands of VPCs without VPC peering" → you need an **NLB** in front of the service, exposed via a VPC Endpoint Service (PrivateLink).
