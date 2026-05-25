---
tags: [concept, networking, security, load-balancing]
aliases: [GWLB, Gateway Load Balancer]
date: 2026-04-28
---

# Gateway Load Balancer (GWLB)

The **Gateway Load Balancer** is a Layer 3 (Network Layer) load balancer in the [[Elastic Load Balancer (ELB)]] family. It was launched in 2020 to solve a very specific problem: how do you deploy, scale, and manage a fleet of **third-party network virtual appliances** (firewalls, IDS/IPS, deep packet inspection) transparently in your [[VPC]] without changing your application architecture?

If [[Network Load Balancer (NLB)|NLB]] is the "fast" load balancer and [[Elastic Load Balancer (ELB)|ALB]] is the "smart" one, GWLB is the **"transparent inline inspector."**

![[gwlb-architecture.png]]

---

## The Problem GWLB Solves

Imagine your company requires that **every packet** entering or leaving your [[VPC]] must pass through a security appliance (firewall, intrusion detection system, etc.) before reaching your application. Without GWLB, you would need to:

1. Manually insert appliances into the traffic path using complex routing.
2. Handle HA and scaling of those appliances yourself.
3. Rewrite route tables every time an appliance fails.

GWLB does all of this automatically. It acts as a **transparent network gateway** (a single entry/exit point for all traffic) AND a **load balancer** that distributes traffic across your virtual appliance fleet.

---

## How GWLB Works — The Traffic Flow

This is the most important concept for the exam. The key word is **"bump-in-the-wire"** — GWLB inserts appliances into the traffic path without the source or destination knowing.

```
                    INBOUND TRAFFIC FLOW
                    ════════════════════

    ┌──────────┐
    │ Internet │
    └────┬─────┘
         │
    ┌────▼──────────────┐
    │  Internet Gateway  │
    └────┬──────────────┘
         │
         │  Route table sends traffic to GWLB Endpoint
         │
    ┌────▼──────────────────────────────────────────┐
    │  GWLB Endpoint (in Application VPC)            │
    │  (behaves like a VPC Interface Endpoint)       │
    └────┬──────────────────────────────────────────┘
         │
         │  Tunneled via GENEVE to Security VPC
         │
    ┌────▼──────────────────────────────────────────┐
    │  Gateway Load Balancer (in Security VPC)       │
    │  Distributes across appliance fleet            │
    └────┬────────────┬─────────────┬───────────────┘
         │            │             │
    ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
    │Firewall │  │Firewall │  │  IDS    │
    │   #1    │  │   #2    │  │  #3     │
    └────┬────┘  └────┬────┘  └────┬────┘
         │            │             │
         └────────────┼─────────────┘
                      │
              Traffic approved ✓
                      │
                      ▼
              Back through GWLB
                      │
              Back through GWLB Endpoint
                      │
              ┌───────▼───────┐
              │  Application  │
              │  (EC2, etc.)  │
              └───────────────┘
```

### Step-by-Step
1. Traffic enters the [[VPC]] via the Internet Gateway.
2. The **route table** redirects traffic to a **GWLB Endpoint** (a VPC endpoint in the application's VPC/subnet).
3. The GWLB Endpoint tunnels the traffic to the **GWLB** itself, which may live in a completely different VPC (the "Security VPC").
4. GWLB distributes the traffic across the **virtual appliance fleet** (Target Group).
5. The appliance **inspects** the packet. If approved, the appliance sends it back to the GWLB.
6. GWLB returns the traffic through the GWLB Endpoint back into the application's VPC.
7. The packet reaches the application [[EC2]] instance as if nothing happened.

> [!IMPORTANT]
> The application and the client are **completely unaware** that a security appliance inspected the traffic. GWLB is transparent — it doesn't modify the packet payload, source, or destination IP.

---

## GENEVE Protocol — The Technical Glue

GWLB uses the **GENEVE** (Generic Network Virtualization Encapsulation) tunneling protocol on **port 6081**.

### Why GENEVE?
When GWLB sends a packet to a firewall appliance, the appliance needs to inspect the packet and then send it **back** to the GWLB so it can be forwarded to the original destination. GENEVE encapsulation preserves the **entire original packet** (including all headers) inside a tunnel. This means:

- The appliance sees the **original source/destination IPs** for inspection.
- The appliance doesn't need to know the routing — it just sends the packet back through the GENEVE tunnel.
- GWLB can reconstruct the original flow and route accordingly.

> [!NOTE]
> You don't need to configure GENEVE yourself. The virtual appliance vendor (e.g., Palo Alto, Fortinet, Check Point) builds GENEVE support into their AMI. You just deploy their appliance from the **AWS Marketplace**.

---

## Key Components

### 1. Gateway Load Balancer (the LB itself)
- Lives in the **Security VPC** (or same VPC as the appliances).
- Distributes traffic across the virtual appliance Target Group.
- Operates at Layer 3 — forwards **IP packets**, not TCP connections or HTTP requests.

### 2. GWLB Endpoint (GWLBe)
- A **VPC Endpoint** (powered by AWS PrivateLink) that you create in the **application's VPC**.
- Acts as the entry/exit point for traffic that needs to be inspected.
- You configure **route tables** to point traffic at the GWLB Endpoint.
- You can have multiple GWLB Endpoints across different [[Availability Zones (AZ)|AZs]] and [[Subnets]] for high availability.

### 3. Target Group (the appliances)
- Contains your fleet of virtual network appliances.
- Target types: **[[EC2]] instances** or **IP addresses**.
- GWLB performs health checks on appliances and only sends traffic to healthy ones.

---

## Target Groups

| Target Type | Description |
|---|---|
| **EC2 Instances** | Virtual appliances running on EC2 (e.g., Palo Alto VM-Series AMI from Marketplace) |
| **IP Addresses** | Private IPs — useful for appliances in peered VPCs or on-premises via Direct Connect |

> [!WARNING]
> GWLB target groups do **not** support Lambda functions or ALB as targets — only instances and IPs.

---

## Health Checks

GWLB health checks work similarly to [[Network Load Balancer (NLB)|NLB]]:

| Protocol | Notes |
|---|---|
| **TCP** | Default. Checks if the appliance port is accepting connections. |
| **HTTP** | Sends GET to a health check path. Expects `200` response. |
| **HTTPS** | Same as HTTP but over TLS. |

- If an appliance fails health checks, GWLB **stops routing traffic** to it and redistributes across remaining healthy appliances.
- **Critical:** If all appliances are unhealthy, traffic is **dropped** (your application becomes unreachable). This is by design — it's better to block traffic than to let uninspected traffic through in a security architecture.

---

## Cross-Zone Load Balancing

| Setting | Default | Cost |
|---|---|---|
| **Cross-Zone** | **Disabled** | You **pay** for inter-AZ data transfer if enabled |

Same behavior as [[Network Load Balancer (NLB)|NLB]]:
- When disabled, each GWLB node only sends traffic to appliances in its own AZ.
- When enabled, traffic is distributed evenly across all appliances in all AZs.

---

## Routing Configuration

This is where GWLB differs from all other load balancers. You don't point your application at the GWLB — you **modify route tables** so traffic is transparently redirected.

### Ingress Route Table (attached to Internet Gateway)
```
Destination          Target
10.0.1.0/24          gwlbe-xxxxxxxxx   ← GWLB Endpoint
```

### Application Subnet Route Table
```
Destination          Target
0.0.0.0/0            gwlbe-xxxxxxxxx   ← GWLB Endpoint (for outbound)
```

This means **all traffic** entering or leaving the application subnet is forced through the GWLB Endpoint → GWLB → Appliances, transparently.

> [!TIP]
> **Exam Pattern:** If you see a diagram or question where traffic is being routed through a "security inspection VPC" before reaching the application VPC, the answer involves **GWLB + GWLB Endpoints + route table modifications**.

---

## Common Use Cases

| Use Case | Example Appliance |
|---|---|
| **Next-Gen Firewalls** | Palo Alto VM-Series, Fortinet FortiGate |
| **Intrusion Detection / Prevention (IDS/IPS)** | Suricata, Snort on EC2 |
| **Deep Packet Inspection (DPI)** | Custom or vendor appliances |
| **Data Loss Prevention (DLP)** | Symantec, McAfee |
| **Network Monitoring / Analytics** | Traffic mirroring to analytics appliances |
| **Compliance Enforcement** | All traffic must pass through approved inspection |

---

## GWLB vs Other Load Balancers

| Feature | ALB | NLB | GWLB |
|---|---|---|---|
| **OSI Layer** | 7 (HTTP) | 4 (TCP/UDP) | 3 (IP) |
| **Use Case** | Web app routing | High-perf TCP/UDP | Transparent traffic inspection |
| **Targets** | EC2, ECS, Lambda, IPs | EC2, IPs, ALB | EC2, IPs |
| **Protocol** | HTTP, HTTPS, gRPC | TCP, UDP, TLS | IP (GENEVE on 6081) |
| **Client sees it?** | Yes (DNS endpoint) | Yes (static IP) | **No** (transparent, route-based) |
| **Modifies traffic?** | Yes (terminates HTTP) | Minimal (L4 forwarding) | **No** (passes original packets) |
| **Cross-Zone default** | On (free) | Off (paid) | Off (paid) |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → GWLB:**
> - "Inspect all network traffic before it reaches the application"
> - "Third-party virtual appliances" or "network virtual appliances"
> - "Firewalls in a centralized security VPC"
> - "Deep packet inspection" or "intrusion detection"
> - "Transparently route traffic through appliances"
> - "GENEVE protocol"
> - "Bump-in-the-wire"
> - "All traffic must be inspected for compliance"
