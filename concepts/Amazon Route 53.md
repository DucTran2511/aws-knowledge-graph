---
tags: [concept, networking, dns, high-availability, routing]
aliases: [Route 53, R53, DNS, Domain Name System]
date: 2026-05-03
---

# Amazon Route 53

**Amazon Route 53** is a highly available, scalable, fully managed and **authoritative** DNS (Domain Name System) web service. "Authoritative" means **you** (the customer) can update the DNS records. It is also a **Domain Registrar** — you can purchase and manage domain names directly through Route 53.

The name "Route 53" is a reference to **TCP/UDP port 53**, the standard port for DNS traffic.

Route 53 is the only AWS service that provides a **100% availability SLA**.

---

## DNS Fundamentals

DNS is the system that translates human-readable domain names (e.g., `www.example.com`) into machine-readable IP addresses (e.g., `93.184.216.34`). It is the backbone of the internet.

### Key DNS Terminology

| Term | Definition |
|---|---|
| **Domain Registrar** | Where you register/purchase domain names (e.g., Route 53, GoDaddy, Namecheap) |
| **DNS Records** | Instructions stored in a hosted zone that tell DNS how to route traffic (A, AAAA, CNAME, NS, etc.) |
| **Hosted Zone** | A container for DNS records for a specific domain |
| **Name Server (NS)** | The servers that respond to DNS queries for your hosted zone |
| **Top-Level Domain (TLD)** | The last part of a domain: `.com`, `.org`, `.gov`, `.io` |
| **Second-Level Domain (SLD)** | The part before the TLD: `amazon` in `amazon.com` |
| **FQDN** | Fully Qualified Domain Name — the complete domain: `api.www.example.com.` |
| **TTL** | Time to Live — how long a DNS resolver caches the answer before querying again |

### How DNS Works (Simplified)

```
Web Browser                Local DNS           Root DNS          TLD DNS (.com)       Route 53
(client)                   Resolver            Server            Server               (SLD NS)
    │                         │                   │                  │                    │
    │─── example.com? ──────▶│                   │                  │                    │
    │                         │─── .com NS? ────▶│                  │                    │
    │                         │◀── NS: x.tld ────│                  │                    │
    │                         │─── example.com NS? ──────────────▶│                    │
    │                         │◀── NS: ns1.route53 ──────────────│                    │
    │                         │─── A record for example.com? ──────────────────────▶│
    │                         │◀── IP: 9.10.11.12, TTL 300 ────────────────────────│
    │◀── 9.10.11.12 ─────────│                   │                  │                    │
    │    (cached for 300s)    │                   │                  │                    │
```

---

## Hosted Zones

A **Hosted Zone** is a container for records that define how to route traffic for a domain and its subdomains. There are two types:

### Public Hosted Zone

- Contains records that specify how to route traffic **on the internet** (public domain names).
- Example: resolves `app.mypublicdomain.com` to a public IP `54.22.33.44`.

### Private Hosted Zone

- Contains records that specify how to route traffic **within one or more [[VPC]]s** (private domain names).
- Example: resolves `api.internal.company.com` to a private IP `10.0.0.25`.
- The private hosted zone must be **associated** with the VPC(s) that need to resolve these names.

> [!IMPORTANT]
> Public hosted zones cost **$0.50/month** per hosted zone. Private hosted zones also cost $0.50/month. You also pay $0.40 per million queries (first billion) for standard queries.

---

## DNS Record Types

You must know the following record types for the exam:

| Record Type | What It Maps | Example |
|---|---|---|
| **A** | Domain → **IPv4** address | `example.com` → `1.2.3.4` |
| **AAAA** | Domain → **IPv6** address | `example.com` → `2001:db8::1` |
| **CNAME** | Domain → **another domain name** | `app.example.com` → `blabla.anything.com` |
| **NS** | **Name Servers** for the hosted zone | Controls which servers answer DNS queries |
| **Alias** | Domain → **AWS resource** (Route 53 proprietary) | `example.com` → ALB DNS name |

Other record types (less tested): **MX** (mail), **TXT** (verification), **SRV** (service locator), **SOA** (start of authority).

---

## CNAME vs Alias Records

This is a **heavily tested** topic. Understanding the difference is critical.

### CNAME Record

- Points a hostname to **any other hostname** (`app.mydomain.com` → `blabla.anything.com`).
- ⚠️ **CANNOT be used for the zone apex** (root domain). You cannot create a CNAME for `mydomain.com`, only for `something.mydomain.com`.
- Standard DNS record — works with any DNS provider.
- **DNS queries are charged**.
- You set the **TTL** yourself.

### Alias Record (Route 53 Proprietary)

- Points a hostname to a **specific AWS resource** (`mydomain.com` → `d111.cloudfront.net`).
- ✅ **CAN be used for the zone apex** (root domain). This is the critical differentiator!
- **Free of charge** for DNS queries to AWS resources.
- **Native health check** integration — automatically recognizes the health of the target resource.
- TTL is **automatically set by Route 53** — you cannot override it.

### Alias Targets (What You Can Point To)

| ✅ Valid Alias Targets | ❌ NOT a Valid Alias Target |
|---|---|
| [[Elastic Load Balancer (ELB)]] | **[[EC2]] DNS name** (you cannot alias to an EC2 instance!) |
| [[Amazon CloudFront\|CloudFront]] Distribution | |
| [[Amazon API Gateway\|API Gateway]] | |
| [[Elastic Beanstalk]] environment | |
| [[S3]] Website endpoint | |
| [[VPC]] Interface Endpoint | |
| Global Accelerator | |
| Route 53 record **in the same hosted zone** | |

> [!CAUTION]
> **Exam Trap:** You **cannot** set an Alias record for an EC2 DNS name (e.g., `ec2-1-2-3-4.compute.amazonaws.com`). Use an **A record** pointing to the EC2 IP, or put an ELB/ALB in front of it.

> [!TIP]
> **Exam Pattern:** Whenever a question asks about pointing a root/apex domain (`example.com`) to an AWS resource — the answer is **Alias record**, never CNAME.

---

## TTL (Time to Live)

TTL defines how long a DNS response is **cached** at the client or resolver before a new query is made to Route 53.

```
                    TTL = 300s (5 min)
                    
Client ──▶ Route 53 ──▶ "A record: 1.2.3.4, TTL 300"
  │                                  │
  │◀─────────── cached ─────────────│
  │                                  
  │ (next 300 seconds: uses cached IP, NO new DNS query)
  │
  │ (after 300 seconds: cache expires, new DNS query)
  │──▶ Route 53 ──▶ "A record: 5.6.7.8, TTL 300"
```

| TTL Value | Pros | Cons |
|---|---|---|
| **High** (e.g., 24h) | Less DNS traffic, less cost, faster for clients | Records are stale longer if you change them |
| **Low** (e.g., 60s) | Records update quickly, easy to change | More DNS queries = more cost, more traffic to Route 53 |

> [!TIP]
> **Best Practice:** Before a DNS record change (e.g., migrating to a new IP), lower TTL to 60 seconds 24-48 hours in advance. After the change propagates, raise TTL back up.

> [!WARNING]
> Alias records do **not** let you set TTL — it is automatically managed by Route 53.

---

## Routing Policies

*Detailed articles: [[Route 53 Simple Routing]], [[Route 53 Weighted Routing]], [[Route 53 Latency Routing]], [[Route 53 Failover Routing]], [[Route 53 Geolocation Routing]], [[Route 53 Geoproximity Routing]], [[Route 53 Multi-Value Routing]], [[Route 53 IP-Based Routing]]*

Routing policies define **how Route 53 responds to DNS queries**. They do NOT route traffic — they answer DNS queries. The client then uses the DNS answer to connect.

Route 53 supports **8 routing policies**:

### 1. Simple

- Route traffic to **a single resource**.
- Can specify **multiple values** in the same record — the client chooses one at random (client-side load balancing).
- **Cannot** associate health checks with Simple routing.
- If multiple values are returned, the client picks a random one.

### 2. Weighted

- Control the **percentage of traffic** sent to each resource.
- Assign a **weight** to each record. Traffic proportion = `weight / sum of all weights`.
- Weights don't need to sum to 100. Example: weights 70, 20, 10 → 70%, 20%, 10%.
- A weight of **0** stops sending traffic to a resource. If **all** weights are 0, records are returned equally.
- **Can** associate health checks.
- **Use case:** Canary deployments, A/B testing, gradual migration between regions.

### 3. Latency-Based

- Redirect to the resource that has the **lowest network latency** relative to the user.
- Latency is measured between the user and the **AWS Region** where the resource resides.
- **Can** associate health checks (has failover capability).
- **Use case:** Global applications where response time matters most.

> [!TIP]
> **Exam Pattern:** "Minimize latency for global users" → **Latency-based routing**. It measures actual network performance, NOT geographic distance.

### 4. Failover (Active-Passive)

- Uses a **primary** and **secondary** (disaster recovery) resource.
- Route 53 performs a health check on the primary. If unhealthy → automatically routes to secondary.
- **Mandatory** health check on the primary resource.
- **Use case:** Active-passive DR setup.

```
         Health Check
              │
    ┌─────────▼──────────┐
    │     Route 53        │
    │  Failover Policy    │
    └────┬──────────┬─────┘
         │          │
    Primary    Secondary
   (healthy)   (standby)
         │          
    ◀─── traffic ──┘ (only if primary unhealthy)
```

### 5. Geolocation

- Route traffic based on **where the user is located** (continent, country, or US state).
- You create records mapped to specific locations.
- **Must define a "Default" record** for users whose location cannot be determined or doesn't match any specific record.
- **Use case:** Content localization (language), restrict content distribution, regulatory compliance.

> [!WARNING]
> Geolocation is NOT the same as Latency-based. Geolocation is a **hard rule** based on the user's location. A user in France will always go to the EU resource even if a US resource is faster.

### 6. Geoproximity (with Traffic Flow)

- Route traffic based on the **geographic location of users AND resources**.
- Uses a **Bias** value to shift traffic toward or away from a resource:
  - **Positive bias (+1 to +99):** Expands the resource's reach — attracts more traffic.
  - **Negative bias (-1 to -99):** Shrinks the resource's reach — pushes traffic away.
- **Requires Route 53 Traffic Flow** to use.
- Resources can be AWS resources (specify AWS Region) or non-AWS resources (specify latitude/longitude).

```
Bias = 0 (default)                    Bias = +25 on us-east-1
┌─────────┬─────────┐                 ┌──────────────┬────────┐
│         │         │                 │              │        │
│us-east-1│eu-west-1│                 │  us-east-1   │eu-west │
│  50%    │  50%    │                 │    70%       │  30%   │
│         │         │                 │              │        │
└─────────┴─────────┘                 └──────────────┴────────┘
   Equal split                         us-east-1 "reaches further"
```

> [!TIP]
> **Exam Pattern:** "Shift traffic from one region to another" or "gradually expand to a new region" → **Geoproximity with Bias**.

### 7. Multi-Value Answer

- Route traffic to **multiple resources** (up to 8 healthy records returned).
- **Can** associate health checks — only healthy resources are returned.
- It is **NOT a substitute** for a load balancer, but provides client-side load balancing with health checking.
- **Use case:** Return multiple IPs and let the client choose, while filtering out unhealthy targets.

> [!IMPORTANT]
> **Simple vs Multi-Value:** Both can return multiple values. The key difference is that Multi-Value supports **health checks** and only returns healthy records. Simple routing returns all values regardless of health.

### 8. IP-Based

- Route traffic based on the **client's source IP address**.
- You define a list of **CIDR blocks** mapped to specific endpoints/locations.
- **Use case:** Route traffic from a known ISP's IP range to a specific endpoint, optimize costs, restrict access by IP.

---

## Routing Policy Comparison Cheat Sheet

| Policy | Health Check? | Use Case Trigger Phrase |
|---|---|---|
| **Simple** | ❌ | Single resource, no special routing |
| **Weighted** | ✅ | "Split traffic", "canary", "percentage" |
| **Latency** | ✅ | "Lowest latency", "fastest response" |
| **Failover** | ✅ (mandatory on primary) | "Active-passive", "disaster recovery" |
| **Geolocation** | ✅ | "Users in France → EU server", "compliance" |
| **Geoproximity** | ✅ | "Shift traffic", "bias", "expand region" |
| **Multi-Value** | ✅ | "Multiple IPs with health checking" |
| **IP-Based** | ✅ | "Route by client CIDR", "ISP-based routing" |

---

## Health Checks

*Main article: [[Route 53 Health Checks]]*

Route 53 health checks are critical for high-availability architectures. They monitor endpoints and integrate with routing policies to automatically avoid unhealthy resources.

### Types of Health Checks

#### 1. Endpoint Health Checks

- Monitor an **IP address** or **domain name** endpoint.
- ~15 global health checkers test the endpoint.
- Configurable: interval (default 30s, fast 10s costs more), threshold (default 3), protocol (HTTP, HTTPS, TCP).
- Healthy if ≥ **18%** of checkers report healthy.
- For HTTP/HTTPS: can optionally check if the response body contains a specific string (first **5120 bytes**).
- ⚠️ Must ensure your firewall/security group allows inbound traffic from Route 53 health checker IPs.

#### 2. Calculated Health Checks

- Combine the results of **multiple child health checks** using AND, OR, or NOT logic.
- Define a threshold: "healthy if at least X of Y child checks are healthy."
- **Use case:** Perform maintenance on one endpoint without triggering a full failover.

#### 3. CloudWatch Alarm Health Checks

- Monitor a **CloudWatch Alarm** state (useful for private resources).
- The health check status mirrors the CloudWatch alarm state.
- **Use case:** Monitor health of resources in **private [[VPC]]s** — since Route 53 health checkers are external and can't access private IPs, use CloudWatch metrics + alarm → Route 53 health check.

> [!WARNING]
> **Private Resources:** Route 53 health checkers live on the public internet. They **cannot** directly access endpoints in private VPCs or private subnets. The workaround is: create a **CloudWatch Metric** → **CloudWatch Alarm** → **Route 53 Health Check** that monitors the alarm.

---

## Domain Registration with Route 53

Route 53 is also a **Domain Registrar**. You can:

1. **Register a new domain** directly in Route 53.
2. **Transfer a domain** from another registrar (e.g., GoDaddy → Route 53).

### Using Route 53 as DNS with a Third-Party Registrar

If your domain is registered with a third-party registrar (e.g., GoDaddy), you can still use Route 53 as your DNS service:

1. Create a **Public Hosted Zone** in Route 53 for your domain.
2. Copy the **NS (Name Server) records** from the hosted zone.
3. Update the **name servers** at your third-party registrar to point to Route 53's NS records.

> [!IMPORTANT]
> **Domain Registrar ≠ DNS Service.** You can buy a domain from GoDaddy and use Route 53 for DNS. The registrar just stores your NS records. The NS records tell the world which DNS servers are authoritative for your domain.

---

## Route 53 Traffic Flow

**Traffic Flow** is a visual editor for creating complex routing configurations (traffic policies).

- Supports **versioning** — you can roll back to previous policy versions.
- Creates a **Traffic Flow Policy** which is applied to a hosted zone via a **Policy Record**.
- Required for **Geoproximity routing**.
- Allows **chaining** of routing policies (e.g., Geolocation → Failover → Weighted).
- Traffic Flow policies cost **$50/month** per policy record.

---

## Quick Reference: Exam Patterns

| Scenario | Answer |
|---|---|
| Point root domain to an ALB | **Alias record** |
| Point `www.example.com` to another hostname | **CNAME** (or Alias if AWS resource) |
| Canary deployment — send 5% to new version | **Weighted routing** (5 vs 95) |
| Active-passive DR | **Failover routing** |
| Fastest response time for global users | **Latency-based routing** |
| Users in Germany must hit EU servers (compliance) | **Geolocation routing** |
| Shift more traffic to a new region gradually | **Geoproximity routing with Bias** |
| Return multiple healthy IPs to the client | **Multi-Value routing** |
| Monitor private resource health for DNS failover | **CloudWatch Alarm → Route 53 Health Check** |
| Domain bought on GoDaddy, DNS on Route 53 | Update NS records at GoDaddy to point to Route 53 |
| 100% availability SLA | **Route 53** (only AWS service with this SLA) |
