---
tags: [concept, compute, high-availability, scaling]
aliases: [ASG, Auto Scaling Group, Auto Scaling, EC2 Auto Scaling]
date: 2026-04-28
---

# Auto Scaling Group (ASG)

An **Auto Scaling Group (ASG)** is an AWS service that automatically manages a fleet of [[EC2]] instances. It **launches** instances when demand increases, **terminates** them when demand decreases, and **replaces** unhealthy instances — all without manual intervention. Combined with an [[Elastic Load Balancer (ELB)]], an ASG is the foundation of **horizontal scaling** and **high availability** in AWS.

---

## Why Auto Scaling Exists

In the real world, website traffic is unpredictable:

```
Traffic Pattern (typical day):

Load ▲
     │                    ╱╲
     │                   ╱  ╲
     │         ╱╲       ╱    ╲
     │        ╱  ╲     ╱      ╲
     │   ╱╲  ╱    ╲   ╱        ╲
     │  ╱  ╲╱      ╲ ╱          ╲
     │ ╱            ╲╱            ╲
     │╱                            ╲
     └──────────────────────────────────► Time
     6am    10am    2pm    6pm    10pm

Without ASG: You provision for peak → waste money 80% of the day
With ASG:    Instances scale with demand → pay only for what you use
```

### Core Goals
1. **Scale out** (add instances) to match increased load.
2. **Scale in** (remove instances) to match decreased load — save money.
3. **Ensure a minimum/maximum** number of instances are always running.
4. **Automatically replace** unhealthy instances.
5. **Automatically register** new instances with an [[Elastic Load Balancer (ELB)|ELB]].

---

## ASG Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │           Auto Scaling Group                │
                    │                                             │
                    │   Minimum: 2    Desired: 3    Maximum: 6   │
                    │                                             │
                    │   ┌──────┐  ┌──────┐  ┌──────┐             │
                    │   │ i-1  │  │ i-2  │  │ i-3  │             │
                    │   │ AZ-a │  │ AZ-b │  │ AZ-a │             │
                    │   └──┬───┘  └──┬───┘  └──┬───┘             │
                    │      │        │        │                   │
                    └──────┼────────┼────────┼───────────────────┘
                           │        │        │
                    ┌──────▼────────▼────────▼───────┐
                    │     Application Load Balancer   │
                    │     (distributes traffic)       │
                    └────────────────┬────────────────┘
                                    │
                                 Internet
```

---

## Key Concepts

### Capacity Settings

| Setting | Description | Example |
|---|---|---|
| **Minimum** | The ASG will **never** have fewer instances than this | 2 (always at least 2 for HA) |
| **Desired** | The **target** number of instances the ASG maintains right now | 3 (current workload needs 3) |
| **Maximum** | The ASG will **never** launch more instances than this | 6 (cost protection cap) |

```
│◄─── Minimum ───►│◄─── Desired ───────►│◄─── Maximum ──────────►│
│                  │                     │                        │
│     2            │       3             │          6             │
│  (always run)    │  (current target)   │    (never exceed)      │
```

- **Scale out:** Desired increases (up to Maximum). New instances are launched.
- **Scale in:** Desired decreases (down to Minimum). Excess instances are terminated.
- You can manually set Desired, or let **scaling policies** adjust it automatically.

> [!WARNING]
> If you set Minimum = Maximum = Desired (e.g., all set to 3), the ASG becomes a **fixed-size fleet** with no scaling. It still provides self-healing (replacing unhealthy instances), but no elasticity.

---

## Launch Template (Required)

A **Launch Template** tells the ASG **how** to launch new instances. It contains:

| Setting | Description |
|---|---|
| **AMI** | The [[Amazon Machine Images (AMI)|AMI]] to use |
| **Instance Type** | e.g., `t3.micro`, `m5.large` |
| **Key Pair** | SSH key for access |
| **Security Groups** | Firewall rules |
| **User Data** | Bootstrap script (install software, configure the app) |
| **IAM Role** | Instance profile for AWS API permissions |
| **EBS Volumes** | Storage configuration |
| **Network** | [[Subnets]], [[Elastic Network Interface (ENI)|ENI]] settings |
| **Purchase Options** | On-Demand, Spot, or mixed fleet |

### Launch Template vs Launch Configuration

| Feature | Launch Template | Launch Configuration |
|---|---|---|
| **Status** | **Current** (recommended) | **Legacy** (deprecated) |
| **Versioning** | ✅ Supports versions | ❌ Immutable (create new each time) |
| **Mixed instances** | ✅ On-Demand + Spot mix | ❌ Single instance type only |
| **Inheritance** | ✅ Can inherit from a parent template | ❌ No |

> [!WARNING]
> **Launch Configurations are deprecated.** AWS recommends migrating to Launch Templates. The exam may still mention Launch Configurations, but the correct modern answer is always Launch Template.

---

## Scaling Policies

Scaling policies define **when and how** the ASG should add or remove instances.

### 1. Manual Scaling
- You manually change the **Desired Capacity**.
- Use case: You know traffic will spike (e.g., product launch) and pre-scale.

### 2. Dynamic Scaling — Target Tracking (Most Common)

You set a **target metric value**, and the ASG automatically adjusts capacity to maintain it. This is the **simplest and most recommended** policy.

| Target Metric | Example Target | What Happens |
|---|---|---|
| **Average CPU Utilization** | 40% | ASG adds instances when avg CPU > 40%, removes when < 40% |
| **Request Count per Target** | 1000 req/target | ASG ensures each instance handles ~1000 requests |
| **Average Network In** | 500 MB/s | ASG scales based on inbound network traffic |
| **Average Network Out** | 500 MB/s | ASG scales based on outbound network traffic |
| **Custom CloudWatch Metric** | Any metric | e.g., queue depth, memory usage, active connections |

```
Example: Target Tracking at 50% CPU

CPU ▲
    │
70% │     ╱╲              ← ASG launches instances to bring CPU down
    │    ╱  ╲
50% │───╱────╲─────────── ← Target line
    │  ╱      ╲
30% │ ╱        ╲          ← ASG terminates instances (underutilized)
    │╱          ╲
    └──────────────────► Time
```

> [!TIP]
> **Exam Pattern:** "Maintain average CPU at X%" → **Target Tracking** scaling policy.

### 3. Dynamic Scaling — Simple / Step Scaling

You create **CloudWatch Alarms** and define actions:

**Simple Scaling:**
- Alarm triggers → perform one action → wait for cooldown → re-evaluate.
- e.g., "If CPU > 70% for 5 minutes → add 2 instances."

**Step Scaling (improved):**
- Alarm triggers → perform actions proportional to how far the metric exceeds the threshold.
- e.g., "If CPU 60-70% → add 1. If CPU 70-80% → add 2. If CPU > 80% → add 3."
- **No cooldown period** — continuously adjusts, making it more responsive than Simple Scaling.

```
Step Scaling Example:

CPU Range       Action
─────────────────────────
0%  - 30%       Remove 2 instances
30% - 50%       Remove 1 instance
50% - 70%       Do nothing
70% - 80%       Add 1 instance
80% - 90%       Add 2 instances
90% - 100%      Add 3 instances
```

### 4. Scheduled Scaling

Scale based on **known, predictable patterns** using a cron-like schedule.

```
Example:
- Every Monday 8:00 AM  → Set Desired to 10  (workday starts)
- Every Friday 6:00 PM  → Set Desired to 2   (weekend)
- Every Black Friday     → Set Desired to 50  (annual sale)
```

Use case: Known traffic patterns (business hours, seasonal events, marketing campaigns).

### 5. Predictive Scaling

AWS uses **machine learning** to analyze historical traffic patterns and **proactively scale** before traffic arrives.

```
Traditional Reactive Scaling:          Predictive Scaling:
Traffic spike → alarm → scale out     ML forecasts spike → scale out BEFORE spike
(users experience slow response        (capacity ready when traffic arrives)
 during the scaling lag)
```

- AWS analyzes the last **14 days** of CloudWatch metrics.
- Creates a **forecast** and pre-scales capacity.
- Works best when traffic has **recurring patterns** (daily, weekly).
- Can be combined with other scaling policies.

> [!TIP]
> **Exam Pattern:** "Traffic is predictable and recurring" → **Predictive Scaling**. "Traffic is unpredictable" → **Target Tracking**.

---

## Cooldown Period

After a scaling activity, the ASG enters a **cooldown period** (default: **300 seconds**) during which it ignores further scaling triggers. This prevents the ASG from launching or terminating instances in rapid succession before the previous scaling action takes effect.

```
t=0s      CPU hits 80% → Scale Out (launch 2 instances)
t=0-300s  COOLDOWN — ignore all scaling triggers
t=300s    Cooldown ends → re-evaluate metrics
t=300s    CPU now at 45% → no action needed ✓
```

- Default: **300 seconds**.
- You can customize per scaling policy.
- **Tip:** Use a short cooldown (60s) with **Target Tracking** policies, which are already self-regulating.
- **Tip:** Create a dedicated scaling policy for scale-in with a shorter cooldown to react faster to decreasing load.

> [!NOTE]
> Target Tracking and Step Scaling have built-in intelligence that makes cooldowns less critical. Simple Scaling relies heavily on cooldowns.

---

## ASG + ELB Integration

ASGs integrate seamlessly with [[Elastic Load Balancer (ELB)]]:

### Automatic Registration
When the ASG launches a new instance, it **automatically registers** the instance with the configured ELB Target Group. When an instance is terminated, it is **automatically deregistered**.

### ELB Health Checks
By default, the ASG only uses **EC2 status checks** (is the VM running?). You can enable **ELB health checks**, which are much more thorough:

| Health Check Type | What It Checks | Default |
|---|---|---|
| **EC2** | Is the VM running? (system + instance status) | ✅ Always on |
| **ELB** | Is the application responding to HTTP health checks? | ❌ Must enable |

> [!IMPORTANT]
> **Always enable ELB health checks** on your ASG. Without them, the ASG might consider an instance "healthy" even if the application has crashed — the VM is still running, but the app returns 500 errors. With ELB health checks, the ASG replaces it.

### Health Check Grace Period
After launching a new instance, the ASG waits for the **Health Check Grace Period** (default: **300 seconds**) before checking health. This gives the instance time to boot, install software (User Data), and start the application.

If the grace period is too short, the ASG may terminate instances that are still booting → launching replacements → infinite loop.

---

## Scaling Termination Policy

When the ASG needs to scale in (terminate instances), it follows a **Termination Policy** to decide which instance to kill:

### Default Termination Policy (Order)
1. **Find the AZ with the most instances** (to maintain balance across AZs).
2. Within that AZ, terminate the instance with the **oldest Launch Configuration / Launch Template**.
3. If tied, terminate the instance **closest to the next billing hour** (to maximize value).

### Other Termination Policies

| Policy | Behavior |
|---|---|
| **Default** | AZ balance → oldest config → closest to billing hour |
| **OldestInstance** | Terminate the oldest instance (by launch time) |
| **NewestInstance** | Terminate the newest instance |
| **OldestLaunchConfiguration** | Terminate instance with the oldest Launch Config |
| **OldestLaunchTemplate** | Terminate instance with the oldest Launch Template version |
| **ClosestToNextInstanceHour** | Terminate the instance closest to the next billing hour |
| **AllocationStrategy** | For mixed fleets — aligns with Spot allocation strategy |

### Instance Protection
You can **protect specific instances** from scale-in:
- **Instance Scale-In Protection:** Prevents the ASG from terminating a specific instance during scale-in.
- Useful for instances running critical batch jobs or leader election processes.
- Protected instances **can still be terminated** manually or due to health check failures.

---

## Lifecycle Hooks

**Lifecycle Hooks** let you pause an instance during launch or termination to perform custom actions.

```
Normal Lifecycle:
Pending → InService → Terminating → Terminated

With Lifecycle Hooks:
Pending → [PENDING:WAIT] → Pending:Proceed → InService
                ↑
        Your custom action:
        - Install software
        - Pull config from S3
        - Register with monitoring
        - Run validation tests

InService → Terminating → [TERMINATING:WAIT] → Terminating:Proceed → Terminated
                                ↑
                        Your custom action:
                        - Upload logs to S3
                        - Deregister from service discovery
                        - Take a final EBS snapshot
                        - Send notification
```

### How Hooks Work
1. ASG launches/terminates an instance.
2. Instance enters the **wait state** (e.g., `Pending:Wait`).
3. ASG sends a notification (to **SNS**, **SQS**, **EventBridge**, or invokes a **Lambda**).
4. Your custom code runs.
5. Your code signals **CONTINUE** (proceed) or **ABANDON** (cancel and terminate/keep).
6. If no signal is received within the **heartbeat timeout** (default: 3600 seconds, max: 48 hours), the default action is taken.

> [!TIP]
> **Exam Pattern:** "Need to perform custom setup on new instances before they receive traffic" or "need to extract logs before instance termination" → **Lifecycle Hooks**.

---

## Instance Refresh

**Instance Refresh** allows you to update all instances in an ASG in a rolling fashion (e.g., when you update the Launch Template with a new AMI).

### How It Works
1. You trigger an Instance Refresh (console, CLI, or API).
2. ASG terminates instances in batches and launches replacements with the **new Launch Template**.
3. You configure a **Minimum Healthy Percentage** (e.g., 90%) — the ASG ensures at least 90% of capacity is always available during the refresh.

```
Instance Refresh (min healthy = 90%, 10 instances):

Step 1: Terminate 1 old instance (9/10 = 90% healthy ✓)
Step 2: Launch 1 new instance with new AMI
Step 3: Wait for new instance to pass health checks
Step 4: Terminate next old instance
... repeat until all instances are updated
```

### Key Settings
| Setting | Description | Default |
|---|---|---|
| **Min Healthy Percentage** | Minimum % of instances that must remain InService | 90% |
| **Instance Warmup** | Time to wait after launch before considering healthy | Health check grace period |
| **Skip Matching** | Skip instances already running the target Launch Template version | ✅ Enabled |

> [!NOTE]
> Instance Refresh is the modern way to do rolling AMI updates. Before this feature, you had to manually double the ASG size, wait, then scale down — or use external tools.

---

## Multi-AZ & AZ Rebalancing

ASG **automatically distributes instances evenly** across the [[Availability Zones (AZ)|Availability Zones]] you configure.

### AZ Rebalancing
If an AZ imbalance occurs (e.g., an AZ outage recovers and its instances were replaced in other AZs), the ASG will:
1. **Launch** new instances in the underrepresented AZ.
2. **Terminate** excess instances in the overrepresented AZ.

This happens automatically and ensures your fleet is always spread across AZs for maximum availability.

> [!WARNING]
> During rebalancing, the ASG may **temporarily exceed** the Maximum capacity by 10% to avoid dropping below the Desired count. This is expected behavior.

---

## Warm Pools

A **Warm Pool** is a set of **pre-initialized instances** sitting in a stopped (or running) state, ready to be launched into the ASG much faster than cold-starting a new instance.

| State | Description | Cost |
|---|---|---|
| **Stopped** | Instance is created, booted, initialized, then stopped | Pay for EBS only |
| **Running** | Instance is fully running in the pool (not receiving traffic) | Pay for instance + EBS |
| **Hibernated** | Instance is hibernated (RAM saved to EBS) | Pay for EBS only |

### Why Warm Pools?
- Cold-launching an instance can take **5-10 minutes** (AMI boot + User Data + app initialization).
- A warm pool instance is **pre-initialized** and can be launched in **seconds**.
- Ideal for applications with **long initialization times**.

---

## Mixed Instance Policies (Spot + On-Demand)

ASGs can run a **mix of On-Demand and Spot Instances** for cost optimization:

```
Mixed Fleet Example:

ASG (Desired: 10)
├── On-Demand Base: 4 instances (guaranteed capacity)
├── On-Demand % Above Base: 30%
└── Spot %: 70% of remaining capacity

Result:
- 4 On-Demand (base)
- 2 On-Demand (30% of 6 remaining)
- 4 Spot (70% of 6 remaining)
```

### Spot Instance Integration
- Define **multiple instance types** in the Launch Template for Spot diversity.
- ASG automatically picks the cheapest available Spot capacity.
- If Spot is interrupted, ASG launches a replacement (On-Demand or different Spot pool).
- Use **capacity-optimized** allocation strategy for lowest interruption rates.

---

## Scaling Metrics — What to Monitor

| Metric | Good For | Notes |
|---|---|---|
| **CPUUtilization** | Compute-bound apps | Most common target tracking metric |
| **RequestCountPerTarget** | Web apps behind ALB | Ensures even request distribution |
| **NetworkIn / NetworkOut** | Network-bound apps | Upload/download heavy workloads |
| **Custom CloudWatch metric** | Any workload | e.g., queue depth (SQS), memory usage, active DB connections |

> [!TIP]
> **SQS + ASG pattern (exam favorite):** Use the **ApproximateNumberOfMessagesVisible** metric from [[SQS]] as a custom CloudWatch metric. Scale out when the queue grows, scale in when it shrinks. This decouples producers from consumers.

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → ASG:**
> - "Automatically add/remove instances based on demand" → ASG + scaling policy
> - "Maintain CPU at X%" → **Target Tracking**
> - "Known traffic patterns at specific times" → **Scheduled Scaling**
> - "Proactively scale before traffic arrives" → **Predictive Scaling**
> - "Replace unhealthy instances automatically" → ASG + ELB health checks
> - "Run custom scripts before instance receives traffic" → **Lifecycle Hooks**
> - "Update all instances with new AMI" → **Instance Refresh**
> - "Reduce cost with Spot Instances" → **Mixed Instance Policy**
> - "Scale based on SQS queue depth" → **Custom CloudWatch metric + Target Tracking**
> - "Application takes 10 minutes to initialize" → **Warm Pools** or increase Health Check Grace Period
>
> **Key defaults:**
> - Cooldown: **300 seconds**
> - Health Check Grace Period: **300 seconds**
> - Termination Policy: AZ balance → oldest config → closest to billing hour
