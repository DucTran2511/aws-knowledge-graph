---
tags: [concept, networking, high-availability, load-balancing]
aliases: [Connection Draining, Deregistration Delay, ELB Draining, ELB Deregistration]
date: 2026-04-28
---

# ELB — Connection Draining

**Connection Draining** (called **Deregistration Delay** on ALB/NLB) is an [[Elastic Load Balancer (ELB)]] feature that gives **in-flight requests time to complete** before a target is fully removed from service. Without it, active connections are abruptly terminated — users see errors, downloads break, and transactions fail mid-way.

---

## The Problem: Abrupt Termination

When a target needs to be removed (scaling down, deploying new code, failing health checks), there are likely **active connections** still being processed:

```
WITHOUT Connection Draining:

Client A ──request──► ALB ──► Instance i-1 (processing a 30-second report...)
                                    │
                              Instance deregistered ✕
                                    │
                              Connection KILLED ← Client A gets 502 Bad Gateway
```

```
WITH Connection Draining:

Client A ──request──► ALB ──► Instance i-1 (processing a 30-second report...)
                                    │
                              Instance set to "draining" state
                              ├── New requests → go to OTHER instances
                              └── Client A's request → allowed to finish ✓
                                    │
                              30 seconds later: report completes, response sent
                                    │
                              Instance fully deregistered ✓
```

---

## How It Works — Step by Step

### 1. Trigger
Connection draining activates when a target enters one of these states:

| Trigger | Description |
|---|---|
| **Manual deregistration** | You remove a target from the Target Group (console, CLI, API) |
| **Auto Scaling scale-in** | An [[EC2]] Auto Scaling Group terminates an instance to reduce capacity |
| **Failed health checks** | The target fails consecutive health checks and is marked **unhealthy** |
| **Rolling deployment** | A deployment tool (CodeDeploy, ECS) replaces old instances with new ones |

### 2. Target Enters "Draining" State
- The ELB marks the target as **draining** (visible in the console as `draining` status).
- **New requests** are **no longer routed** to this target.
- **Existing in-flight requests** continue to be processed.

### 3. Countdown Timer Starts
- The **deregistration delay timer** begins counting down (default: 300 seconds).
- During this window, the target can finish processing all active connections.

### 4. Outcome
| What Happens | Result |
|---|---|
| All connections complete **before** the timer expires | Target is deregistered immediately (no need to wait the full duration) |
| Timer expires with connections **still active** | Remaining connections are **forcefully closed** — the target is deregistered regardless |

```
Timeline:

t=0s        Target marked "draining"
            ├── No new requests sent
            ├── 5 in-flight requests still processing
            │
t=10s       3 requests complete → connections closed normally
t=25s       1 request completes → connection closed normally
t=45s       Last request completes → connection closed normally
            │
t=45s       All connections done ← Target deregistered EARLY (didn't wait 300s)
```

```
Worst Case:

t=0s        Target marked "draining"
            ├── 1 very long request still processing
            │
t=300s      Timer expires → connection FORCEFULLY CLOSED
            │
t=300s      Target deregistered (request may have failed)
```

---

## Naming by Load Balancer Type

AWS uses different names for the same concept:

| Load Balancer | Name | Where to Configure |
|---|---|---|
| **CLB** | **Connection Draining** | Load Balancer settings |
| **[[Application Load Balancer (ALB)\|ALB]]** | **Deregistration Delay** | Target Group → Attributes |
| **[[Network Load Balancer (NLB)\|NLB]]** | **Deregistration Delay** | Target Group → Attributes |
| **[[Gateway Load Balancer (GWLB)\|GWLB]]** | **Deregistration Delay** | Target Group → Attributes |

> [!NOTE]
> The exam may use either term. "Connection Draining" = CLB terminology. "Deregistration Delay" = ALB/NLB/GWLB terminology. They are the **same feature**.

---

## Configuration

| Setting | Value |
|---|---|
| **Default** | **300 seconds** (5 minutes) |
| **Minimum** | **0 seconds** (disabled — connections are terminated immediately) |
| **Maximum** | **3600 seconds** (1 hour) |
| **Configured at** | Target Group level (ALB/NLB/GWLB) or LB level (CLB) |

### AWS CLI Example
```bash
# Set deregistration delay to 60 seconds for a Target Group
aws elbv2 modify-target-group-attributes \
    --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-tg/abc123 \
    --attributes Key=deregistration_delay.timeout_seconds,Value=60
```

---

## How to Choose the Right Value

The deregistration delay should match the **longest request your application handles**:

| Application Type | Recommended Value | Reasoning |
|---|---|---|
| **Stateless API** (< 1s responses) | **30 seconds** | Requests complete almost instantly; drain fast for quick deployments |
| **Web application** (< 5s responses) | **60–120 seconds** | Most pages load within seconds; buffer for slow queries |
| **File uploads / downloads** | **300–600 seconds** | Large file transfers can take minutes |
| **Long-polling / WebSocket** | **300–3600 seconds** | Connections can be very long-lived |
| **Batch processing / Reports** | **600–3600 seconds** | Report generation can take many minutes |
| **Stateless + fast deploys** | **0 seconds** (disabled) | No in-flight state to preserve; instant recycling |

> [!TIP]
> **General rule:** Set the deregistration delay to **slightly longer** than your slowest request. Too short = dropped connections. Too long = slow deployments (instances sit in "draining" for minutes doing nothing).

---

## Impact on Deployments & Auto Scaling

### Rolling Deployments
During a rolling deployment (e.g., via CodeDeploy or ECS), old instances are deregistered and new ones registered. The deregistration delay directly impacts **deployment speed**:

```
Deployment with 300s deregistration delay (default):

t=0s      Old instance deregistered → enters "draining" (300s timer)
t=0s      New instance registered → needs to pass health checks (~30s)
t=300s    Old instance fully removed ← this is the bottleneck!

Total time per instance swap: ~300 seconds
For 10 instances rolling 1-at-a-time: ~50 minutes!
```

```
Deployment with 30s deregistration delay:

t=0s      Old instance deregistered → enters "draining" (30s timer)
t=0s      New instance registered → needs to pass health checks (~30s)
t=30s     Old instance fully removed ✓

Total time per instance swap: ~30 seconds
For 10 instances rolling 1-at-a-time: ~5 minutes!
```

> [!IMPORTANT]
> **The #1 cause of slow rolling deployments** is a deregistration delay that is too high. If your app handles only short requests, reduce it to 30–60 seconds.

### Auto Scaling Scale-In
When an Auto Scaling Group terminates an instance (scale-in), it first deregisters the instance from the ELB. The instance is **not terminated** until the draining period completes (or the ASG's own termination wait period expires).

This means:
- You **pay** for the instance during the entire draining period.
- With a 300s delay, a scale-in event keeps the instance running for 5 extra minutes.

---

## What the User Sees During Draining

| User Type | Experience |
|---|---|
| **Existing user** (in-flight request to draining target) | Request completes normally — they notice nothing |
| **New user** (arriving during draining) | Routed to a healthy target — they notice nothing |
| **Existing user** (new request after their current one finishes) | Routed to a healthy target — they notice nothing |
| **User with [[ELB Sticky Sessions\|sticky session]]** to draining target | Stickiness is **broken** — next request goes to a different target |

> [!WARNING]
> Draining **breaks sticky sessions**. If a user was sticky to a draining instance, their next request goes to a different target. This is another reason to externalize session state to [[ElastiCache]] or [[DynamoDB]].

---

## Connection Draining vs Connection Idle Timeout

These are **two different features** — don't confuse them:

| Feature | Connection Draining | Idle Timeout |
|---|---|---|
| **When it applies** | Target is being **removed** | Connection is **idle** (no data flowing) |
| **What it does** | Allows in-flight requests to finish | Closes connections that have been silent too long |
| **Default** | 300 seconds | 60 seconds (ALB) |
| **Purpose** | Graceful shutdown | Resource cleanup |
| **Configured at** | Target Group attributes | Load Balancer attributes |

---

## NLB-Specific Behavior

NLB handles connection draining slightly differently because it operates at Layer 4 (TCP):

- NLB draining applies to **TCP connections**, not HTTP requests.
- A single TCP connection can carry many HTTP requests (HTTP keep-alive).
- If a TCP connection is open when draining starts, NLB waits for the connection to close or the timer to expire.
- NLB also supports a **connection termination delay** — after the deregistration delay expires, NLB sends a TCP RST to forcefully close remaining connections.

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Connection Draining:**
> - "Instances are deregistered and users see 502 errors"
> - "In-flight requests are being dropped during deployments"
> - "Rolling deployment is too slow" → reduce deregistration delay
> - "How to gracefully remove an instance from the load balancer"
> - "Instance is in 'draining' state"
>
> **Key facts to memorize:**
> - CLB = "Connection Draining", ALB/NLB = "Deregistration Delay"
> - Default: **300 seconds**
> - Range: **0–3600 seconds**
> - Set to **0** to disable (instant removal)
> - Affects deployment speed and Auto Scaling cost
