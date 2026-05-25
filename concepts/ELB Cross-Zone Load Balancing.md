---
tags: [concept, networking, high-availability, load-balancing]
aliases: [Cross-Zone Load Balancing, Cross Zone, Cross-AZ Load Balancing]
date: 2026-04-28
---

# ELB — Cross-Zone Load Balancing

**Cross-Zone Load Balancing** controls whether an [[Elastic Load Balancer (ELB)]] node distributes traffic **only to targets in its own [[Availability Zones (AZ)|Availability Zone]]** or **evenly across all targets in all AZs**. It is one of the most heavily tested ELB concepts because it directly impacts how traffic is distributed, application availability, and your AWS bill.

---

## The Problem: Uneven Distribution

When you create a load balancer, AWS deploys a **load balancer node** in each AZ you enable. Each node receives a roughly equal share of incoming traffic (via DNS round-robin). The question is: what does each node do with its share?

### Without Cross-Zone Load Balancing

Each LB node distributes traffic **only to targets registered in its own AZ**. If the number of targets per AZ is unequal, the per-instance load becomes wildly uneven.

```
                        Client Traffic (100%)
                              │
                    ┌─────────┴─────────┐
                    │ DNS round-robin    │
                    │ 50%         50%    │
                    ▼                    ▼
            ┌──────────────┐    ┌──────────────┐
            │  LB Node     │    │  LB Node     │
            │  AZ-a        │    │  AZ-b        │
            └──────┬───────┘    └──────┬───────┘
                   │                   │
          Only AZ-a targets     Only AZ-b targets
                   │                   │
         ┌─────────┼─────────┐         │
         ▼         ▼         ▼         ▼
    ┌─────────┐┌─────────┐┌─────────┐┌─────────┐
    │ i-1     ││ i-2     ││ i-3     ││ i-4     │
    │ 16.7%   ││ 16.7%   ││ 16.7%   ││ 50%  ⚠ │
    └─────────┘└─────────┘└─────────┘└─────────┘
         AZ-a (3 instances)     AZ-b (1 instance)
```

**Result:** Instance `i-4` in AZ-b gets **50%** of all traffic — 3× more than any instance in AZ-a. It's likely to become a bottleneck or crash.

### With Cross-Zone Load Balancing

Each LB node distributes traffic **evenly across all registered targets in all AZs**. The number of targets per AZ no longer matters.

```
                        Client Traffic (100%)
                              │
                    ┌─────────┴─────────┐
                    │ DNS round-robin    │
                    │ 50%         50%    │
                    ▼                    ▼
            ┌──────────────┐    ┌──────────────┐
            │  LB Node     │    │  LB Node     │
            │  AZ-a        │    │  AZ-b        │
            └──────┬───────┘    └──────┬───────┘
                   │                   │
          ALL targets in ALL AZs  ALL targets in ALL AZs
                   │                   │
         ┌─────────┼─────────┬─────────┘
         ▼         ▼         ▼         ▼
    ┌─────────┐┌─────────┐┌─────────┐┌─────────┐
    │ i-1     ││ i-2     ││ i-3     ││ i-4     │
    │ 25%  ✓  ││ 25%  ✓  ││ 25%  ✓  ││ 25%  ✓ │
    └─────────┘└─────────┘└─────────┘└─────────┘
         AZ-a (3 instances)     AZ-b (1 instance)
```

**Result:** Every instance gets exactly **25%** — perfectly balanced, regardless of how instances are distributed across AZs.

---

## Defaults & Costs Per Load Balancer Type

This table is **the most exam-tested fact** about cross-zone load balancing:

| Load Balancer | Default | Can Be Changed? | Inter-AZ Data Charges |
|---|---|---|---|
| **[[Application Load Balancer (ALB)\|ALB]]** | **ON** (always enabled at LB level) | Can be **disabled per Target Group** (since 2023) | **Free** — no charges |
| **[[Network Load Balancer (NLB)\|NLB]]** | **OFF** | Can be enabled per Target Group | **You pay** for inter-AZ data |
| **[[Gateway Load Balancer (GWLB)\|GWLB]]** | **OFF** | Can be enabled per Target Group | **You pay** for inter-AZ data |
| **CLB** | **OFF** | Can be enabled at LB level | **Free** — no charges |

> [!IMPORTANT]
> **Memorize this pattern:**
> - ALB = ON by default, free.
> - NLB & GWLB = OFF by default, costs money.
> - CLB = OFF by default, but free if you turn it on.

---

## The Cost Question — Inter-AZ Data Transfer

When cross-zone is enabled, a LB node in AZ-a sends traffic to targets in AZ-b. This is **inter-AZ data transfer**, which AWS normally charges for (~$0.01/GB per direction in most regions).

### Why ALB Is Free
AWS made a product decision: ALB's cross-zone is always on and they absorb the cost. This makes ALB simpler to reason about — traffic is always evenly distributed.

### Why NLB/GWLB Charge
NLB and GWLB handle **massive volumes** of traffic (millions of RPS, large packet sizes). The inter-AZ data transfer costs at scale would be enormous if absorbed by AWS, so they pass the cost to the customer.

> [!TIP]
> **Cost optimization tip:** If your NLB targets are evenly distributed across AZs and you're concerned about inter-AZ costs, you can leave cross-zone **disabled**. The distribution will be roughly even anyway. Only enable it when target counts per AZ are significantly unbalanced.

---

## How Cross-Zone Works Under the Hood

### Step 1: DNS Resolution
When a client resolves the LB's DNS name, Route 53 returns the IP addresses of the LB nodes — one per enabled AZ. The client picks one (usually round-robin or nearest).

### Step 2: LB Node Receives Traffic
The selected LB node receives the request.

### Step 3: Target Selection
- **Without Cross-Zone:** The node has a list of **only** the targets in its AZ. It picks one using the routing algorithm (round-robin, least outstanding requests, etc.).
- **With Cross-Zone:** The node has a list of **all** targets in **all** AZs. It picks one, potentially sending the request to a different AZ.

### Step 4: Inter-AZ Hop (if applicable)
If the selected target is in a different AZ, the traffic crosses AZ boundaries through AWS's internal network. This adds:
- ~0.5ms of additional latency (negligible for most apps).
- Inter-AZ data transfer cost (for NLB/GWLB).

---

## ALB: Target Group Level Control

Since 2023, ALB allows you to control cross-zone at the **Target Group level**, not just the LB level:

| Target Group Setting | Behavior |
|---|---|
| **Use load balancer setting** (default) | Inherits the LB-level setting (which is always on) |
| **On** | Cross-zone enabled for this Target Group |
| **Off** | Cross-zone disabled for this Target Group — LB node only routes to targets in its own AZ |

**Use case for disabling per Target Group:** You have latency-sensitive targets that should only receive traffic from the local AZ to avoid the ~0.5ms inter-AZ hop, while other Target Groups can be cross-zone.

---

## Common Scenarios (Exam Style)

### Scenario 1: Uneven Instance Distribution
> "You have an ALB with 10 instances in AZ-a and 2 instances in AZ-b. Users in AZ-b report slow response times."

**Without Cross-Zone:** Each AZ-b instance handles 25% of traffic (50% ÷ 2), while each AZ-a instance handles 5% (50% ÷ 10). AZ-b instances are overloaded.

**With Cross-Zone (ALB default):** Each instance handles ~8.3% (100% ÷ 12). Problem solved.

→ **Answer:** ALB has cross-zone on by default, so this shouldn't happen unless it was disabled at the Target Group level. Check Target Group settings.

### Scenario 2: Cost Optimization with NLB
> "Your NLB serves 10 TB/day of traffic across 3 AZs. You enabled cross-zone and your bill increased significantly."

**Why:** Cross-zone on NLB causes inter-AZ data transfer. At ~$0.01/GB, 10 TB/day × 30 days × ~33% cross-zone traffic ≈ additional cost.

→ **Answer:** Ensure targets are **evenly distributed** across AZs, then disable cross-zone. If targets are even, the per-AZ distribution will be naturally balanced.

### Scenario 3: AZ Failure
> "AZ-a goes down. All instances in AZ-a are unreachable."

**Without Cross-Zone:** The LB node in AZ-a has no healthy targets. Clients routed to AZ-a's LB node get errors. Only clients routed to AZ-b's LB node succeed.

**With Cross-Zone:** Both LB nodes (AZ-a and AZ-b) can route to targets in AZ-b. But wait — clients still need to reach the LB node in AZ-a, and if the entire AZ is down, the LB node itself may be unreachable.

→ **Answer:** Cross-zone doesn't protect against **LB node failure** — it protects against **uneven target distribution**. For AZ failure, the LB's DNS automatically removes the failed AZ's LB node IP, and all traffic goes to the surviving AZ's LB node, which routes to surviving targets.

---

## Interaction with Other Features

### Cross-Zone + Sticky Sessions
- With cross-zone enabled, [[ELB Sticky Sessions|sticky sessions]] can pin a client to a target in **any AZ**, not just the AZ their LB node is in.
- This means stickiness persists even if the initial LB node changes (e.g., DNS rotation).

### Cross-Zone + Auto Scaling
- [[EC2]] Auto Scaling Groups can be configured to maintain an **even instance count across AZs** (AZ rebalancing).
- If your ASG keeps instances balanced, cross-zone has less impact because the per-AZ distribution is already even.
- **Best practice:** Use ASG AZ rebalancing **and** cross-zone for double protection.

### Cross-Zone + Weighted Target Groups
- With [[Application Load Balancer (ALB)|ALB weighted target groups]], cross-zone applies **within** each target group's share. The weight split is honored first, then cross-zone distributes within each group.

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Cross-Zone Load Balancing:**
> - "Uneven number of instances across AZs"
> - "Some instances receive more traffic than others"
> - "Imbalanced load distribution"
> - "Inter-AZ data transfer charges increased after enabling..."
> - "One AZ has more targets than another"
>
> **Defaults to memorize:**
> - ALB = **ON**, free
> - NLB = **OFF**, paid
> - GWLB = **OFF**, paid
> - CLB = **OFF**, free
