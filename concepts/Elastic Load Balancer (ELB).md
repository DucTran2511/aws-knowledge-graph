---
tags: [concept, networking, high-availability, load-balancing]
aliases: [ELB, Load Balancer, ALB, NLB, CLB, GWLB, Application Load Balancer, Network Load Balancer, Classic Load Balancer, Gateway Load Balancer]
date: 2026-04-28
---

# Elastic Load Balancer (ELB)

An **Elastic Load Balancer (ELB)** is a managed AWS service that automatically distributes incoming traffic across multiple downstream targets (e.g., [[EC2]] instances, containers, Lambda functions, IP addresses). It is a core building block for achieving **high availability**, **fault tolerance**, and **horizontal scaling** in AWS architectures.

## Why Use a Load Balancer?

- **Spread load** across multiple downstream instances.
- **Expose a single point of access** (DNS) to your application.
- **Seamlessly handle failures** of downstream instances via health checks.
- **Provide SSL/TLS termination** (HTTPS) for your websites.
- **Enforce stickiness** with cookies so a user always hits the same backend.
- **High availability across [[Availability Zones (AZ)|Availability Zones]].**
- **Separate public traffic from private traffic** in your architecture.

AWS manages the load balancer infrastructure entirely — it guarantees it will be working, handles upgrades, maintenance, and provides high availability out of the box. You only configure its behavior.

---

## Types of Load Balancers

AWS offers **four** generations of load balancers. The exam focuses heavily on ALB and NLB.

### 1. Classic Load Balancer (CLB) — v1, 2009

- The **legacy** load balancer. AWS considers it deprecated (not recommended for new architectures).
- Supports HTTP, HTTPS, TCP, and SSL (TCP with TLS).
- Operates at **both** Layer 4 and Layer 7 but in a rudimentary way.
- Health checks are TCP or HTTP-based.
- Provides a **fixed hostname**: `xxx.region.elb.amazonaws.com`.

> [!WARNING]
> CLB is deprecated. AWS strongly recommends migrating to ALB or NLB. On the exam, if you see "Classic Load Balancer," the correct answer is almost always to replace it with a newer generation.

---

### 2. Application Load Balancer (ALB) — v2, 2016

- Operates at **Layer 7** (HTTP/HTTPS/WebSocket).
- This is the **most feature-rich** load balancer and the **exam favorite** for web applications.

#### Routing Capabilities
ALB supports advanced **request-based routing** to different Target Groups based on:
- **URL Path** — e.g., `/users` → Target Group A, `/orders` → Target Group B.
- **Hostname** — e.g., `api.example.com` vs. `admin.example.com`.
- **Query String & Headers** — e.g., `?platform=mobile` → mobile Target Group.

This makes ALB ideal for **microservices** and **container-based architectures** (e.g., [[ECS]], [[EKS]]) where different URL paths map to entirely different backend services.

#### Target Groups
ALB routes to **Target Groups**, which can contain:
- [[EC2]] instances (managed by an Auto Scaling Group).
- [[ECS]] tasks.
- [[Lambda]] functions (HTTP request is translated to a JSON event).
- Private IP addresses (for on-premises servers via Direct Connect / VPN).

#### Key Features
- **Fixed hostname**: `xxx.region.elb.amazonaws.com`.
- The application servers **do not see the client's IP directly**. The client's real IP is inserted into the header `X-Forwarded-For`. The port goes into `X-Forwarded-Port` and the protocol into `X-Forwarded-Proto`.
- Supports **HTTP/2** and **WebSocket**.
- Supports **redirects** (e.g., HTTP → HTTPS at the load balancer level).
- Supports **authentication** via OIDC / Cognito integration.

> [!TIP]
> **Exam Pattern:** Whenever a question mentions routing based on URL path, hostname, or headers — the answer is **ALB**.

---

### 3. Network Load Balancer (NLB) — v2, 2017

- Operates at **Layer 4** (TCP, TLS, UDP).
- Designed for **extreme performance**: handles **millions of requests per second** with ultra-low latency (~100ms vs ~400ms for ALB).

#### Key Differentiators
- NLB exposes **one static IP per AZ** and supports assigning an **[[AWS IP Addressing|Elastic IP]]** to each AZ. This is critical for whitelisting by IP address.
- **Preserves the client's source IP** — the backend sees the actual client IP (unlike ALB which uses `X-Forwarded-For`).
- **Not included in the AWS Free Tier.**

#### Target Groups
NLB can route to:
- [[EC2]] instances.
- Private IP addresses.
- An **ALB** in front of it (NLB → ALB pattern for static IP + Layer 7 routing).

#### Health Checks
NLB supports three health check protocols: **TCP**, **HTTP**, and **HTTPS**.

> [!TIP]
> **Exam Pattern:** Whenever a question mentions static IP, whitelisting IP, extreme performance, UDP, or non-HTTP protocols — the answer is **NLB**.

---

### 4. Gateway Load Balancer (GWLB) — 2020

- Operates at **Layer 3** (Network Layer — IP Packets).
- Used to deploy, scale, and manage a fleet of **third-party network virtual appliances** in AWS.
- Examples: Firewalls, Intrusion Detection/Prevention Systems (IDS/IPS), Deep Packet Inspection, payload manipulation.

#### How It Works
All traffic into your [[VPC]] is first routed through the GWLB. The GWLB distributes the traffic across your virtual appliances (e.g., firewalls) for inspection. If the appliance approves the traffic, it is forwarded to the destination application.

```
                        ┌─────────────────────┐
                        │   Internet Gateway   │
                        └────────┬────────────┘
                                 │
                        ┌────────▼────────────┐
                        │  Gateway Load        │
                        │  Balancer (GWLB)     │
                        └────────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                   ▼
        ┌───────────┐    ┌───────────┐    ┌───────────────┐
        │ Firewall  │    │ Firewall  │    │  IDS/IPS      │
        │ Appliance │    │ Appliance │    │  Appliance    │
        └─────┬─────┘    └─────┬─────┘    └──────┬────────┘
              └────────────────┼─────────────────┘
                               ▼
                    ┌──────────────────────┐
                    │  Your Application    │
                    │  (EC2, etc.)         │
                    └──────────────────────┘
```

#### Key Technical Detail
- Uses the **GENEVE protocol** on port **6081**.
- Target Groups: [[EC2]] instances or IP addresses.

> [!TIP]
> **Exam Pattern:** Any question about inspecting network traffic, third-party security appliances, or deep packet inspection → the answer is **GWLB**.

---

## Health Checks

Health checks are **critical** — they allow the ELB to know if a target instance is healthy and able to receive traffic.

- The health check is done on a **port** and a **route** (e.g., `/health` on port 4567).
- If the target does not respond with `200 OK`, the ELB marks it as **unhealthy** and stops sending traffic to it.
- Once the instance starts responding again, it is marked **healthy** and traffic resumes.

---

## Sticky Sessions (Session Affinity)

*Main article: [[ELB Sticky Sessions]]*

By default, an ELB distributes every request to a different target. **Stickiness** ensures that the same client is always redirected to the same target behind the load balancer.

- Works for **CLB**, **ALB**, and **NLB**.
- Uses a **cookie** with an expiration date that you control.
- Use case: ensure the user doesn't lose their session data (if sessions are stored on the instance rather than in a centralized store like [[ElastiCache]] or [[DynamoDB]]).

### Cookie Types
| Type | Name | Who Controls It | Notes |
|---|---|---|---|
| **Application-based (Custom)** | Custom name | Your application generates the cookie | Must not use `AWSALB`, `AWSALBAPP`, or `AWSALBTG` as names |
| **Application-based (ALB-generated)** | `AWSALBAPP` | ALB generates it | — |
| **Duration-based** | `AWSALB` (ALB) or `AWSELB` (CLB) | The load balancer generates it | Expiration is set by you in the LB configuration |

> [!WARNING]
> Stickiness can cause **imbalanced load**. If one user makes vastly more requests, their "stuck" instance may become overloaded while others idle.

---

## Cross-Zone Load Balancing

*Main article: [[ELB Cross-Zone Load Balancing]]*

Without Cross-Zone Load Balancing, traffic is distributed only within the instances in the same AZ as the load balancer node.

With **Cross-Zone Load Balancing** enabled, each load balancer node distributes traffic **evenly across all registered instances in all AZs**.

```
Without Cross-Zone:                    With Cross-Zone:
AZ-1 (2 instances) gets 50%           AZ-1 (2 instances) gets 20% each
AZ-2 (8 instances) gets 50%           AZ-2 (8 instances) gets 10% each
→ Each AZ-1 instance: 25%             → Every instance: 10%
→ Each AZ-2 instance: 6.25%           → Perfectly even!
```

### Defaults by LB Type
| Load Balancer | Cross-Zone Default | Cost for Inter-AZ Data |
|---|---|---|
| **ALB** | **Always on** (can be disabled at Target Group level) | No charges |
| **NLB & GWLB** | **Disabled** by default | You **pay** for inter-AZ data if enabled |
| **CLB** | **Disabled** by default | No charges if enabled |

---

## SSL/TLS Termination

*Main article: [[ELB SSL TLS Certificates]]*

An ELB can **terminate SSL/TLS** connections. This means the client connects to the LB via HTTPS, and the LB forwards unencrypted HTTP to your backend (within the [[VPC]], so it's still private).

- The SSL certificate is managed using **AWS Certificate Manager (ACM)**.
- You can also upload your own certificates.
- The HTTPS listener must specify a **default SSL certificate** and can optionally list multiple certs to support **multiple domains** using **Server Name Indication (SNI)**.

### Server Name Indication (SNI)

SNI solves the problem of loading **multiple SSL certificates onto one web server** (to serve multiple websites).

- It's a newer protocol that requires the **client to indicate the hostname** of the target server in the initial SSL handshake.
- The server then finds the correct certificate and returns it.
- **Only works with ALB, NLB, and CloudFront.** Does **not** work with CLB (need one CLB per hostname).

---

## Connection Draining / Deregistration Delay

*Main article: [[ELB Connection Draining]]*

When an instance is being deregistered or marked unhealthy, **Connection Draining** gives in-flight requests time to complete before the instance is fully taken out of service.

- Called **Connection Draining** for CLB.
- Called **Deregistration Delay** for ALB & NLB.
- Default: **300 seconds** (configurable from 1 to 3600 seconds).
- Can be **disabled** by setting the value to 0.
- During draining, the ELB **stops sending new requests** to the deregistering instance, but allows existing connections to finish.

> [!TIP]
> If your requests are short (< 1 second), set a low deregistration delay value (e.g., 30 seconds) so instances are recycled faster during deployments.

---

## Security Groups & ELB

Load balancers have their own **Security Groups**. A common architecture pattern:

1. **ELB Security Group**: Allow inbound HTTP (80) and HTTPS (443) from `0.0.0.0/0` (the internet).
2. **EC2 Security Group**: Allow inbound traffic **only from the ELB's Security Group** — not from the internet directly.

This ensures your [[EC2]] instances are never directly exposed to the public internet. All traffic must pass through the load balancer first.

---

## Quick Reference: Choosing the Right Load Balancer

| Requirement | Answer |
|---|---|
| HTTP/HTTPS routing by path, host, headers | **ALB** |
| Microservices / Container routing | **ALB** |
| WebSocket support | **ALB** |
| Static IP / Elastic IP needed | **NLB** |
| Extreme performance / millions of RPS | **NLB** |
| TCP/UDP/TLS (non-HTTP) protocol | **NLB** |
| Third-party network appliances (firewalls, IDS) | **GWLB** |
| Deep packet inspection | **GWLB** |
| Legacy application (avoid if possible) | **CLB** |
