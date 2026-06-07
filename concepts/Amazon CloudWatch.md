---
tags: [concept, monitoring, observability, cloudwatch, metrics, alarms, logs, dashboards]
aliases: [CloudWatch, Amazon CloudWatch, CW, CloudWatch Metrics, CloudWatch Alarms, CloudWatch Logs, CloudWatch Agent]
date: 2026-06-06
---

# Amazon CloudWatch

**Amazon CloudWatch** is AWS's **monitoring and observability** service. It answers the question: **"What is happening right now?"** — collecting metrics, logs, and events from virtually every AWS service to give you unified visibility into resource utilization, application performance, and operational health.

> [!IMPORTANT]
> **Core exam concept:** CloudWatch = **performance monitoring** (metrics + alarms + logs). CloudWatch does NOT track "who did what" (that's [[AWS CloudTrail]]) or "is the configuration compliant?" (that's [[AWS Config]]).

---

## Architecture Overview

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                      Amazon CloudWatch                              │
  │                                                                     │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐│
  │  │   Metrics   │  │    Logs     │  │   Alarms    │  │ Dashboards││
  │  │             │  │             │  │             │  │           ││
  │  │ • Default   │  │ • Log Groups│  │ • Metric    │  │ • Cross-  ││
  │  │ • Custom    │  │ • Log       │  │ • Composite │  │   region  ││
  │  │ • Detailed  │  │   Streams   │  │ • Anomaly   │  │ • Cross-  ││
  │  │ • High-Res  │  │ • Insights  │  │   Detection │  │   account ││
  │  └─────────────┘  └─────────────┘  └─────────────┘  └───────────┘│
  │                                                                     │
  │  Sources:                          Actions:                         │
  │  ┌──────────┐ ┌──────────┐        ┌──────────┐ ┌──────────┐       │
  │  │ EC2, RDS │ │ Lambda,  │        │ SNS      │ │ Auto     │       │
  │  │ ELB, S3  │ │ API GW,  │        │ notify   │ │ Scaling  │       │
  │  │ DynamoDB │ │ On-prem  │        │          │ │ policy   │       │
  │  └──────────┘ └──────────┘        ┌──────────┐ ┌──────────┐       │
  │                                    │ EC2 stop/│ │ Lambda   │       │
  │                                    │ reboot/  │ │ invoke   │       │
  │                                    │ recover  │ │          │       │
  │                                    └──────────┘ └──────────┘       │
  └─────────────────────────────────────────────────────────────────────┘
```

---

## CloudWatch Metrics

### Metric Anatomy

| Concept | Description |
|---|---|
| **Namespace** | Container for metrics (e.g., `AWS/EC2`, `AWS/RDS`, `AWS/Lambda`). Custom namespaces for your own metrics. |
| **Metric** | Time-ordered set of data points (e.g., `CPUUtilization`) |
| **Dimension** | Name/value pair to filter metrics (e.g., `InstanceId=i-1234`) — up to **30 dimensions per metric** |
| **Statistic** | Aggregation over a period: `Average`, `Sum`, `Min`, `Max`, `SampleCount`, `pNN` (percentile) |
| **Period** | Length of time to aggregate data: 1 second → 1 day. Default = **5 minutes** |
| **Resolution** | **Standard** = 60 seconds. **High-resolution** = 1 second (custom metrics only) |

### Basic vs Detailed Monitoring

```
  Basic Monitoring (default, free):
  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
  │    │    │    │    │ ●  │    │    │    │    │ ●  │  ← data point every 5 min
  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  0    1    2    3    4    5    6    7    8    9   10 min

  Detailed Monitoring (enabled per-resource, paid):
  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
  │ ●  │ ●  │ ●  │ ●  │ ●  │ ●  │ ●  │ ●  │ ●  │ ●  │  ← data point every 1 min
  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  0    1    2    3    4    5    6    7    8    9   10 min
```

> [!TIP]
> **Exam Pattern:** "Need faster scaling response" or "1-minute granularity for EC2" → **Enable Detailed Monitoring**. Required for [[Auto Scaling Group (ASG)|ASG]] scaling policies that need 1-minute data.

### Default EC2 Metrics vs Custom Metrics

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                    EC2 Instance                                   │
  │  ┌────────────────────────────────────────────────────────────┐  │
  │  │  Guest OS (INVISIBLE to hypervisor)                       │  │
  │  │                                                            │  │
  │  │  ❌ Memory (RAM) utilization                              │  │
  │  │  ❌ Disk space utilization (% used)                       │  │
  │  │  ❌ Swap utilization                                      │  │
  │  │  ❌ Number of processes / threads                         │  │
  │  │                                                            │  │
  │  │  → Requires CloudWatch Agent to collect!                  │  │
  │  └────────────────────────────────────────────────────────────┘  │
  │                                                                   │
  │  Hypervisor level (DEFAULT — no agent needed):                   │
  │  ✅ CPU Utilization          ✅ Network In/Out                   │
  │  ✅ Disk Read/Write Ops      ✅ Disk Read/Write Bytes            │
  │  ✅ Status Checks (System + Instance)                            │
  │  ✅ Network Packets In/Out                                       │
  └──────────────────────────────────────────────────────────────────┘
```

> [!CAUTION]
> **Exam critical:** "Monitor **memory utilization** on EC2" → **CloudWatch Agent** required (custom metric). Memory/RAM is NOT a default CloudWatch metric. This is one of the most frequently tested facts on SAA-C03.

### Custom Metrics

- Published via `PutMetricData` API or **CloudWatch Agent**
- Support **high-resolution**: down to **1-second** intervals (standard = 60s)
- Examples: memory usage, disk space, application queue depth, active connections
- `StorageResolution` parameter: `1` = high-res, `60` = standard

---

## CloudWatch Unified Agent

```
  ┌──────────────────────────────────────────────────────────────┐
  │                      EC2 Instance / On-Premises Server       │
  │                                                              │
  │  ┌──────────────────────────────────────┐                   │
  │  │     CloudWatch Unified Agent         │                   │
  │  │                                      │                   │
  │  │  Collects:                           │                   │
  │  │  📊 System Metrics                   │──► CloudWatch     │
  │  │     • Memory (mem_used_percent)      │    Metrics        │
  │  │     • Disk  (disk_used_percent)      │                   │
  │  │     • Swap  (swap_used_percent)      │                   │
  │  │     • CPU   (cpu_usage_idle, etc.)   │                   │
  │  │     • Netstat, Processes             │                   │
  │  │                                      │                   │
  │  │  📄 Logs                             │──► CloudWatch     │
  │  │     • /var/log/messages              │    Logs           │
  │  │     • /var/log/httpd/access_log      │                   │
  │  │     • Application logs               │                   │
  │  └──────────────────────────────────────┘                   │
  │                                                              │
  │  Requires: IAM Role with CloudWatchAgentServerPolicy         │
  │  Config:   Stored in SSM Parameter Store (recommended)       │
  └──────────────────────────────────────────────────────────────┘
```

| Feature | Old CW Logs Agent | Unified CW Agent |
|---|---|---|
| **Metrics** | ❌ | ✅ (memory, disk, swap, processes) |
| **Logs** | ✅ | ✅ |
| **OS support** | Linux only | Linux + Windows |
| **Config storage** | Local file | SSM Parameter Store (recommended) |
| **Status** | Legacy (deprecated) | ✅ Current — use this |

> [!NOTE]
> The **Unified Agent** replaces the older CloudWatch Logs Agent. It can collect both metrics AND logs. Configuration is centrally managed via **SSM Parameter Store**.

---

## CloudWatch Logs

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                     CloudWatch Logs                               │
  │                                                                   │
  │  ┌─────────────────────────────────────────────────────────────┐ │
  │  │  Log Group: /aws/lambda/my-function                        │ │
  │  │  (retention: 30 days)                                      │ │
  │  │                                                             │ │
  │  │  ┌──────────────────────────────────────────┐              │ │
  │  │  │  Log Stream: 2026/06/06/[$LATEST]abc123  │              │ │
  │  │  │  → log event: "START RequestId: ..."     │              │ │
  │  │  │  → log event: "Processing order #456"    │              │ │
  │  │  │  → log event: "END RequestId: ..."       │              │ │
  │  │  └──────────────────────────────────────────┘              │ │
  │  │  ┌──────────────────────────────────────────┐              │ │
  │  │  │  Log Stream: 2026/06/06/[$LATEST]def456  │              │ │
  │  │  │  → log event: "START RequestId: ..."     │              │ │
  │  │  └──────────────────────────────────────────┘              │ │
  │  └─────────────────────────────────────────────────────────────┘ │
  │                                                                   │
  │  Sources: Lambda, ECS, API Gateway, CloudTrail, Route 53,       │
  │           VPC Flow Logs, Elastic Beanstalk, CW Agent, custom     │
  └──────────────────────────────────────────────────────────────────┘
```

### Key Log Features

| Feature | Description |
|---|---|
| **Log Groups** | Container for log streams. Retention set here (1 day → 10 years, or never expire) |
| **Log Streams** | Sequence of log events from a single source (instance, function invocation) |
| **Metric Filters** | Extract metric values from log text → create CloudWatch metrics → trigger alarms |
| **Subscription Filters** | Stream log events in real-time to [[Amazon Kinesis|Kinesis Data Streams]], Kinesis Firehose, or [[AWS Lambda|Lambda]] |
| **Logs Insights** | Interactive SQL-like query engine for log analysis (`fields`, `filter`, `stats`, `sort`) |
| **Contributor Insights** | Top-N reports — find "heavy hitters" (top IPs, top error codes, top users) |
| **Export to S3** | Batch export (not real-time). For real-time → use Subscription Filters |
| **Cross-account** | Aggregate logs from multiple accounts into a central monitoring account |

### Log Data Flow

```
  Log Events ──► Metric Filters ──► CloudWatch Metric ──► CloudWatch Alarm ──► SNS / Lambda
                                                              │
  Log Events ──► Subscription Filters ──► Kinesis Firehose ──► S3 (near real-time)
                                      ──► Kinesis Data Streams ──► Lambda processing
                                      ──► Lambda ──► Elasticsearch / OpenSearch

  Log Events ──► Export to S3 (batch, up to 12 hours delay)
```

> [!TIP]
> **Exam Pattern:** "Real-time log processing" → **Subscription Filter** to Kinesis/Lambda. "Export logs to S3 for archival" → **Export task** (batch, NOT real-time). "Find top contributing IPs" → **Contributor Insights**. "Query across log groups" → **Logs Insights**.

---

## CloudWatch Alarms

### Alarm States

```
        ┌────────────┐          ┌────────────┐          ┌────────────────────┐
        │     OK     │◄────────►│   ALARM    │◄────────►│ INSUFFICIENT_DATA  │
        │            │          │            │          │                    │
        │ Within     │          │ Threshold  │          │ Not enough data    │
        │ threshold  │          │ breached   │          │ or alarm just      │
        └────────────┘          └────────────┘          │ started            │
                                     │                  └────────────────────┘
                                     │
                              Triggers Actions:
                              • SNS notification
                              • Auto Scaling policy
                              • EC2 action (stop/terminate/reboot/recover)
                              • Lambda function
                              • Systems Manager action
```

### Alarm Evaluation

```
  Period = 5 min, Evaluation Periods = 3, Datapoints to Alarm = 2
  Threshold: CPU > 80%

  ┌───────┬───────┬───────┬───────┬───────┬───────┐
  │ 85%   │ 72%   │ 90%   │ 65%   │ 88%   │ 92%   │
  │ BREACH│  ok   │ BREACH│  ok   │ BREACH│ BREACH│
  └───────┴───────┴───────┴───────┴───────┴───────┘
          ▲               ▲               ▲
          │               │               │
   Window 1: 2/3 breach  Window 2: 1/3   Window 3: 2/3 breach
   → ALARM!              → OK            → ALARM!
```

### EC2 Alarm Actions

| Action | Description |
|---|---|
| **Stop** | Stop the instance (EBS-backed only) |
| **Terminate** | Terminate the instance permanently |
| **Reboot** | Reboot the instance |
| **Recover** | Migrate to new hardware (same instance ID, IP, EBS, metadata). Uses `StatusCheckFailed_System` |

> [!IMPORTANT]
> **EC2 Recovery** action: Only works for instances with **EBS-backed root volumes** (not instance store). Preserves instance ID, private IP, Elastic IP, and all EBS attachments. Triggered by **system status check failure** (hardware issue).

### Composite Alarms

```
  Individual Alarms:          Composite Alarm:
  ┌──────────────────┐       ┌────────────────────────────────────┐
  │ Alarm A: CPU     │──────►│                                    │
  │ CPUUtilization   │       │  Composite Alarm                   │
  │ > 80%            │       │  Rule: A AND B                     │
  └──────────────────┘       │                                    │──► SNS → PagerDuty
  ┌──────────────────┐       │  Only triggers when BOTH           │
  │ Alarm B: IOPS    │──────►│  CPU AND IOPS are breaching        │
  │ VolumeReadOps    │       │                                    │
  │ > 1000           │       │  Reduces alarm noise!              │
  └──────────────────┘       └────────────────────────────────────┘
```

> [!TIP]
> **Exam Pattern:** "Reduce alarm noise" or "only alert when multiple conditions are met" → **Composite Alarms** (AND/OR logic across child alarms).

---

## CloudWatch Anomaly Detection

- Uses **machine learning** to analyze historical metric data
- Automatically creates a **band** of expected values (accounts for seasonality, trends)
- Create alarms that trigger when metric goes **outside the expected band**
- Better than static thresholds for metrics with variable patterns (e.g., traffic spikes on weekdays)

---

## Metric Streams

```
  CloudWatch Metrics ──► Amazon Data Firehose ──► S3 / Datadog / Splunk / New Relic
                         (near real-time)

  • Continuous stream (not polling)
  • Near-real-time delivery
  • Filter by namespace or metric name
  • OpenTelemetry 0.7.0 or JSON format
```

---

## CloudWatch Dashboards

- **Global** — can include metrics from multiple regions and accounts
- Auto-refresh: 10s, 1m, 2m, 5m, 15m intervals
- Can be **shared** with users who don't have AWS accounts (public dashboards)
- Support: metrics, logs, alarms, and text widgets
- **Pricing**: 3 dashboards (up to 50 metrics each) = free. Then $3/dashboard/month

---

## CloudWatch Synthetics (Canaries)

```
  ┌──────────────────────────────────────────────────────────────┐
  │  CloudWatch Synthetics Canary                                │
  │                                                              │
  │  Configurable script (Node.js / Python)                      │
  │  that runs on a schedule to monitor:                         │
  │                                                              │
  │  🌐 Endpoint availability                                   │
  │  ⏱️  Latency of APIs                                        │
  │  🔗 Broken links                                             │
  │  📸 Screenshots of UI                                        │
  │  📋 Multi-step workflows (e.g., login → checkout)            │
  │                                                              │
  │  → Integrates with CloudWatch Alarms for alerting            │
  └──────────────────────────────────────────────────────────────┘
```

---

## Specialized Monitoring

| Feature | Purpose | Use For |
|---|---|---|
| **Container Insights** | Metrics/logs for ECS, EKS, Kubernetes | CPU, memory, disk, network at cluster/node/pod/task level |
| **Lambda Insights** | Enhanced monitoring for [[AWS Lambda]] | Cold starts, memory usage, function duration |
| **Application Insights** | Automated dashboards for .NET / SQL Server / IIS | Problem detection for common app stacks |
| **Contributor Insights** | Top-N contributor analysis | VPC Flow Logs top talkers, API Gateway top callers |
| **ServiceLens** | End-to-end service map (integrates with X-Ray) | Visualize dependencies, trace requests across microservices |

---

## EventBridge Integration

```
  CloudWatch Alarm ──► state change event ──► EventBridge Rule ──► Target
  (ALARM/OK)                                                       │
                                                           ┌───────┴───────┐
                                                           │ Lambda        │
                                                           │ SNS           │
                                                           │ SQS           │
                                                           │ SSM Automation│
                                                           │ Step Functions│
                                                           └───────────────┘
```

> [!NOTE]
> **EventBridge** (formerly CloudWatch Events) is the preferred method for reacting to operational changes. Alarm state changes are automatically sent to EventBridge. Use EventBridge rules for automated remediation workflows.

---

## Key CloudWatch Limits

| Parameter | Value |
|---|---|
| **Metrics per dashboard** | 2,500 |
| **Alarms per account per region** | 5,000 (default) |
| **Dimensions per metric** | 30 |
| **Custom metric resolution** | 1 second (high-res) to 60 seconds (standard) |
| **Metric retention** | 3 hours (1-sec), 15 days (1-min), 63 days (5-min), 455 days (1-hour) |
| **Log retention** | 1 day → 10 years, or never expire |
| **PutMetricData** | 1,000 metrics per API call |
| **Dashboards (free tier)** | 3 dashboards, 50 metrics each |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → CloudWatch:**
> - "Monitor CPU, network, disk I/O" → **CloudWatch default metrics** (no agent)
> - "Monitor **memory** or **disk space** on EC2" → **CloudWatch Agent** (custom metric)
> - "Alert when metric breaches threshold" → **CloudWatch Alarm** → SNS / Auto Scaling
> - "Reduce alarm noise, multiple conditions" → **Composite Alarms** (AND/OR logic)
> - "Auto-recover EC2 from hardware failure" → **EC2 Recovery alarm action** (StatusCheckFailed_System)
> - "Real-time log streaming to S3/Splunk" → **Subscription Filter** → Kinesis Firehose
> - "Analyze logs interactively" → **CloudWatch Logs Insights** (SQL-like queries)
> - "Stream metrics to 3rd-party (Datadog)" → **Metric Streams** via Firehose
> - "Monitor container metrics (ECS/EKS)" → **Container Insights**
> - "Trace requests across microservices" → **ServiceLens** + AWS X-Ray
> - "Test website/API endpoint availability" → **CloudWatch Synthetics Canary**
> - "1-minute EC2 metrics for faster scaling" → **Enable Detailed Monitoring**
> - "Centralized dashboard, multi-region" → **CloudWatch Dashboard** (global)
>
> **Key facts:**
> - Default EC2 metrics: CPU, Network, Disk I/O, Status Checks. **NOT** memory/disk space/swap.
> - Basic Monitoring = 5 min. Detailed Monitoring = 1 min. Custom high-res = 1 sec.
> - Alarm states: OK, ALARM, INSUFFICIENT_DATA.
> - Alarm actions: SNS, Auto Scaling, EC2 (stop/terminate/reboot/recover), Lambda.
> - Metric data retention scales with resolution: 1s→3h, 1m→15d, 5m→63d, 1h→455d.
> - CloudWatch Agent config stored in **SSM Parameter Store**.
> - Logs export to S3 is batch (up to 12h delay). For real-time → Subscription Filters.
> - CloudWatch ≠ [[AWS CloudTrail]] (audit API calls) ≠ [[AWS Config]] (compliance).
