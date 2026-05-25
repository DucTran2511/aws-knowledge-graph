---
tags: [concept, networking, high-availability, load-balancing]
aliases: [ALB, Application Load Balancer]
date: 2026-04-28
---

# Application Load Balancer (ALB)

The **Application Load Balancer** is a Layer 7 (Application Layer) load balancer in the [[Elastic Load Balancer (ELB)]] family. It is the **most feature-rich** load balancer AWS offers and is purpose-built for **HTTP/HTTPS workloads**. If [[Network Load Balancer (NLB)|NLB]] is the "fast" load balancer, ALB is the **"smart"** one — it can read HTTP headers, paths, query strings, and make intelligent routing decisions.

ALB is the **default choice** for modern web applications, microservices, and container-based architectures.

---

## Why ALB Exists

Traditional load balancers (CLB) could only forward traffic to a single application. Modern architectures run **multiple independent services** behind a single domain — e.g., `/api` goes to one service, `/web` goes to another, `mobile.example.com` goes to a third. ALB was built to route traffic to **different Target Groups** based on the content of the HTTP request.

---

## Core Characteristics

### Layer 7 — HTTP / HTTPS / gRPC / WebSocket
ALB understands the **application protocol**. It fully parses the HTTP request, which enables:
- Content-based routing (path, host, headers, query strings)
- HTTP/2 and WebSocket native support
- gRPC support (via HTTP/2)
- Redirects and fixed responses at the LB level

### DNS-Based Endpoint (No Static IP)
- ALB exposes a **fixed DNS hostname**: `xxx.region.elb.amazonaws.com`
- The underlying IP addresses **can change at any time** — you must always use the DNS name, never hardcode IPs.

> [!WARNING]
> ALB does **not** support static IPs or Elastic IPs. If you need a fixed IP, use [[Network Load Balancer (NLB)|NLB]] in front of ALB (the NLB → ALB pattern).

---

## Routing Rules — ALB's Superpower

ALB listeners support **rules** with **conditions** and **actions**. Each rule evaluates a condition against the incoming request and routes it to the appropriate Target Group.

### Routing Conditions

| Condition | Example | Use Case |
|---|---|---|
| **Path** | `/api/*`, `/images/*` | Microservices — each path is a different service |
| **Host header** | `api.example.com`, `admin.example.com` | Multi-tenant or multi-subdomain apps |
| **HTTP header** | `X-Custom-Header: mobile` | Route by custom metadata |
| **HTTP method** | `GET`, `POST` | Separate read vs. write traffic |
| **Query string** | `?platform=mobile` | Route mobile vs. desktop clients |
| **Source IP** | `203.0.113.0/24` | Route traffic from specific IP ranges |

### Routing Actions

| Action | Description |
|---|---|
| **Forward** | Send to a Target Group (the most common action) |
| **Redirect** | Return a 301/302 redirect (e.g., HTTP → HTTPS) |
| **Fixed response** | Return a static response (e.g., 503 maintenance page) |
| **Authenticate** | Trigger OIDC / Amazon Cognito authentication before forwarding |

### Rule Priority
- Rules are evaluated **in priority order** (lowest number = highest priority).
- The **default rule** (catch-all) is evaluated last and has no conditions.
- Each listener can have up to **100 rules** (excluding the default).

```
Listener :443 (HTTPS)

Rule 1 (Priority 1):  IF path = /api/*        → Forward to API Target Group
Rule 2 (Priority 2):  IF host = admin.example  → Forward to Admin Target Group
Rule 3 (Priority 3):  IF query = ?mobile=true  → Forward to Mobile Target Group
Default Rule:          No condition             → Forward to Default Target Group
```

> [!TIP]
> **Exam Pattern:** Any question mentioning "route based on URL path," "route based on hostname," or "route based on HTTP headers" → the answer is **ALB**.

---

## Target Groups

ALB routes to **Target Groups**. Each target group contains one type of target and has its own health check configuration.

### Target Types

| Target Type | Description | Example |
|---|---|---|
| **EC2 Instances** | Registered by Instance ID | Auto Scaling Group of web servers |
| **ECS Tasks** | Dynamic port mapping | Containerized microservices on [[ECS]] |
| **Lambda Functions** | HTTP request is converted to a JSON event | Serverless API endpoints |
| **IP Addresses** | Must be private IPs | On-premises servers via VPN/Direct Connect, peered VPCs |

### Multiple Target Groups
A single ALB can route to **many different Target Groups** simultaneously using listener rules. This is what makes ALB ideal for microservices:

```
                         ALB (single DNS entry)
                              │
              ┌───────────────┼───────────────┐
              │               │               │
         /api/*          /auth/*          /web/*
              │               │               │
    ┌─────────▼──────┐ ┌─────▼──────┐ ┌──────▼─────────┐
    │ API Target     │ │ Auth Target│ │ Web Target     │
    │ Group          │ │ Group      │ │ Group          │
    │ (ECS Fargate)  │ │ (Lambda)   │ │ (EC2 ASG)      │
    └────────────────┘ └────────────┘ └────────────────┘
```

### Weighted Target Groups
ALB supports **weighted routing** — you can split traffic across multiple Target Groups by percentage. This enables:
- **Blue/Green deployments**: 90% to v1, 10% to v2.
- **Canary releases**: Gradually shift traffic to the new version.

```
Rule: IF path = /api/*
  → Forward to:
      API-v1 Target Group (weight: 90)
      API-v2 Target Group (weight: 10)
```

---

## Health Checks

ALB performs health checks at the **Target Group** level.

| Setting | Default | Description |
|---|---|---|
| **Protocol** | HTTP | Can be HTTP or HTTPS |
| **Path** | `/` | The URL path to check (e.g., `/health`) |
| **Port** | traffic-port | The port to health check on |
| **Healthy threshold** | 5 | Consecutive successes to mark healthy |
| **Unhealthy threshold** | 2 | Consecutive failures to mark unhealthy |
| **Timeout** | 5 seconds | Time to wait for a response |
| **Interval** | 30 seconds | Time between checks |
| **Success codes** | `200` | HTTP status codes that mean healthy (e.g., `200-299`) |

> [!TIP]
> Always set up a dedicated `/health` endpoint that returns `200 OK` with minimal computation. Don't health-check your homepage — it's slow and can cause false unhealthy statuses.

---

## Sticky Sessions (Session Affinity)

ALB supports **cookie-based stickiness** to ensure a user always hits the same target.

### Cookie Types

| Cookie | Name | Generated By | Duration |
|---|---|---|---|
| **Duration-based** | `AWSALB` | ALB | Configured in LB (1s–7 days) |
| **Application-based (LB)** | `AWSALBAPP` | ALB | Configured in LB |
| **Application-based (Custom)** | Your choice | Your application | Your application controls it |

> [!WARNING]
> Stickiness can cause **imbalanced load distribution**. One instance may get overloaded if a "sticky" user generates heavy traffic. Prefer **externalized session storage** ([[ElastiCache]], [[DynamoDB]]) over sticky sessions when possible.

### Reserved Cookie Names
Your application must **not** use these names for custom cookies:
- `AWSALB`
- `AWSALBAPP`
- `AWSALBTG`

---

## SSL/TLS Termination

ALB can **terminate HTTPS** connections, handling the encryption/decryption overhead so your backend doesn't have to.

### How It Works
1. Client connects to ALB over **HTTPS** (port 443).
2. ALB decrypts the request using the SSL certificate.
3. ALB forwards **plain HTTP** to the backend (within the [[VPC]], so it's still private).

### Certificate Management
- Managed via **AWS Certificate Manager (ACM)** — free, auto-renewing SSL certs.
- You can also upload your own certificates.
- A single listener can hold **multiple certificates** (up to 25 per listener + additional via API).

### Server Name Indication (SNI)
- ALB supports **SNI**, allowing it to serve **multiple domains** with different SSL certs on the same listener.
- The client indicates the target hostname in the TLS handshake → ALB picks the correct cert.
- Example: `api.example.com` and `shop.example.com` both on the same ALB, each with their own certificate.

> [!NOTE]
> SNI works on ALB, [[Network Load Balancer (NLB)|NLB]], and [[CloudFront]], but **not** on Classic Load Balancer (CLB needs one LB per hostname).

---

## Client IP & Headers

ALB **terminates the connection** from the client and opens a **new connection** to the backend. This means the backend does not see the client's real IP address in the network packet.

ALB injects the following **HTTP headers** for the backend to use:

| Header | Contains |
|---|---|
| `X-Forwarded-For` | The **client's real IP address** (e.g., `203.0.113.50`) |
| `X-Forwarded-Port` | The port the client connected on (e.g., `443`) |
| `X-Forwarded-Proto` | The protocol the client used (e.g., `https`) |

```
Client (203.0.113.50)
    │
    │ HTTPS
    ▼
   ALB (terminates connection)
    │
    │ HTTP (new connection from ALB's IP)
    │ Headers added:
    │   X-Forwarded-For: 203.0.113.50
    │   X-Forwarded-Port: 443
    │   X-Forwarded-Proto: https
    ▼
  EC2 Backend
```

> [!IMPORTANT]
> If your application needs the real client IP (for logging, rate limiting, geo-blocking), you **must** read it from `X-Forwarded-For`, not from the network socket. This is an exam favorite.

---

## Authentication

ALB has **built-in authentication** support — it can authenticate users before the request even reaches your application.

### Supported Identity Providers
- **Amazon Cognito** — User pools for managed sign-up/sign-in.
- **Any OIDC-compliant IdP** — Google, Facebook, Azure AD, Okta, Auth0, etc.

### How It Works
1. User hits the ALB.
2. ALB redirects the user to the Identity Provider's login page.
3. User authenticates and gets a token.
4. ALB validates the token and forwards the request to the backend with **user claims** in HTTP headers.

This offloads authentication from your application entirely.

---

## Cross-Zone Load Balancing

| Setting | Default | Cost |
|---|---|---|
| **Cross-Zone** | **Always on** (can be disabled at Target Group level) | **Free** — no inter-AZ data charges |

This is different from [[Network Load Balancer (NLB)|NLB]] and [[Gateway Load Balancer (GWLB)|GWLB]], which are disabled by default and charge for cross-zone data.

---

## Connection Draining (Deregistration Delay)

When a target is deregistered or fails health checks:
- ALB **stops sending new requests** to the target.
- In-flight requests are given time to complete.
- Default: **300 seconds** (configurable: 0–3600 seconds).
- Set to **0** for instant deregistration (stateless apps, fast deployments).

> [!TIP]
> For rapid deployments with short-lived requests, set deregistration delay to **30 seconds** or lower. The default 300 seconds will slow down rolling updates significantly.

---

## Slow Start Mode

By default, when a new target is registered, ALB sends it the full share of traffic immediately. **Slow Start Mode** gradually increases the traffic to a newly registered target over a configurable period (30–900 seconds).

This is useful for applications that need warm-up time (e.g., loading caches, initializing connections to databases).

---

## Security Groups

ALB has its own **Security Group**. The standard architecture pattern:

```
Internet
    │
    ▼
┌─────────────────────────────────┐
│ ALB Security Group              │
│ Inbound: 80, 443 from 0.0.0.0/0│
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│ EC2 Security Group              │
│ Inbound: 80 from ALB-SG        │  ← Reference the ALB's SG, NOT 0.0.0.0/0
└─────────────────────────────────┘
```

> [!IMPORTANT]
> The EC2 instances should **only** allow inbound traffic from the ALB's Security Group. This ensures no one can bypass the load balancer and hit the instances directly.

---

## ALB vs Other Load Balancers — Quick Decision

| Question | ALB | NLB | GWLB |
|---|---|---|---|
| HTTP path/host/header routing? | ✅ | ❌ | ❌ |
| WebSocket? | ✅ | ✅ (TCP) | ❌ |
| gRPC? | ✅ | ❌ | ❌ |
| Lambda targets? | ✅ | ❌ | ❌ |
| Built-in authentication? | ✅ | ❌ | ❌ |
| Static IP / Elastic IP? | ❌ | ✅ | ❌ |
| Non-HTTP protocols? | ❌ | ✅ | ✅ |
| Weighted target groups? | ✅ | ❌ | ❌ |
| Free tier? | ✅ | ❌ | ❌ |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → ALB:**
> - "Route based on URL path / hostname / HTTP headers / query string"
> - "Microservices" or "container-based architecture"
> - "Multiple applications behind a single load balancer"
> - "Lambda function as a target"
> - "Blue/Green deployment with weighted routing"
> - "Authenticate users at the load balancer" or "Cognito + load balancer"
> - "HTTP to HTTPS redirect"
> - "Fixed response / maintenance page from the LB"
> - "gRPC"
> - "`X-Forwarded-For`" (need client's real IP behind a load balancer)
