---
tags: [concept, security, audit, governance, cloudtrail, logging, compliance]
aliases: [CloudTrail, AWS CloudTrail, CT, Trail, API Logging, Audit Trail]
date: 2026-06-06
---

# AWS CloudTrail

**AWS CloudTrail** is the **audit and governance** service for AWS. It answers the question: **"Who did what, when, and from where?"** — recording every API call made in your AWS account by users, roles, and services. Think of it as the **CCTV camera** for your AWS environment.

> [!IMPORTANT]
> **Core exam concept:** CloudTrail = **API activity logging** (audit trail). It records WHO made a call, WHAT they called, WHEN it happened, and WHERE (source IP). It does NOT monitor performance (that's [[Amazon CloudWatch]]) or check compliance rules (that's [[AWS Config]]).

---

## Architecture Overview

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                        AWS CloudTrail                                │
  │                                                                      │
  │   WHO?              WHAT?              WHEN?           WHERE?        │
  │   IAM User/Role     API Call           Timestamp       Source IP     │
  │   AWS Service       (e.g., RunInstances) (UTC)         User Agent   │
  │                                                                      │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │                    Event History                                │  │
  │  │  • Last 90 days of management events (free, always-on)        │  │
  │  │  • Viewable in Console, searchable by user/event/resource     │  │
  │  │  • Read-only, cannot be deleted                                │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  │                                                                      │
  │  ┌────────────────────────────────────────────────────────────────┐  │
  │  │                    Trail (custom config)                        │  │
  │  │                                                                 │  │
  │  │  ──► S3 Bucket (long-term storage, encrypted)                  │  │
  │  │  ──► CloudWatch Logs (real-time monitoring + alarms)           │  │
  │  │  ──► EventBridge (automated reactions)                         │  │
  │  │  ──► SNS Topic (notifications)                                 │  │
  │  └────────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Types of CloudTrail Events

### Three Event Categories

```
  ┌────────────────────────────────────────────────────────────────────┐
  │                                                                    │
  │  1. MANAGEMENT EVENTS (Control Plane)              ✅ Default ON  │
  │  ──────────────────────────────────────                            │
  │  "Operations performed ON resources"                              │
  │                                                                    │
  │  Examples:                                                         │
  │  • CreateBucket, DeleteBucket                                      │
  │  • CreateUser, AttachRolePolicy                                    │
  │  • RunInstances, CreateVPC                                         │
  │  • ConfigureTrail, CreateTrail                                     │
  │                                                                    │
  │  Subtypes:                                                         │
  │  • Read events  (Describe*, List*, Get*)                          │
  │  • Write events (Create*, Delete*, Put*, Update*)                 │
  │                                                                    │
  ├────────────────────────────────────────────────────────────────────┤
  │                                                                    │
  │  2. DATA EVENTS (Data Plane)                       ❌ Default OFF │
  │  ──────────────────────────────────                                │
  │  "Operations performed ON data within resources"                  │
  │                                                                    │
  │  Examples:                                                         │
  │  • S3: GetObject, PutObject, DeleteObject                         │
  │  • Lambda: Invoke function                                         │
  │  • DynamoDB: GetItem, PutItem, DeleteItem                         │
  │                                                                    │
  │  ⚠️ HIGH VOLUME → extra charges apply                             │
  │  Must be explicitly enabled per resource type                     │
  │                                                                    │
  ├────────────────────────────────────────────────────────────────────┤
  │                                                                    │
  │  3. INSIGHTS EVENTS                                ❌ Default OFF │
  │  ──────────────────────────────────                                │
  │  "Detect unusual API activity patterns"                           │
  │                                                                    │
  │  Examples:                                                         │
  │  • Spike in TerminateInstances calls                               │
  │  • Burst in S3 deletions (potential attack)                        │
  │  • Unusual IAM activity                                            │
  │                                                                    │
  │  Uses ML baseline → detects anomalies                             │
  └────────────────────────────────────────────────────────────────────┘
```

### Event Comparison Table

| Feature | Management Events | Data Events | Insights Events |
|---|---|---|---|
| **Default logged?** | ✅ Yes | ❌ No | ❌ No |
| **Volume** | Low/Medium | **High** | Low |
| **Cost** | First copy free | Extra charges | Extra charges |
| **Examples** | CreateBucket, RunInstances | GetObject, Invoke | Anomaly detection |
| **Scope** | Control plane (resource ops) | Data plane (object ops) | Pattern analysis |

> [!CAUTION]
> **Exam critical:** Data events are **OFF by default** because they are high-volume (e.g., every S3 GetObject). "Track who accessed S3 objects" → enable **Data Events** on the trail. "Track who created/deleted the S3 bucket" → **Management Events** (already on by default).

---

## Trail Configuration

### Single-Region vs Multi-Region

```
  Multi-Region Trail (recommended, console default):
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  us-east-1 ──┐                                              │
  │  us-west-2 ──┤                                              │
  │  eu-west-1 ──┼──► Single S3 Bucket ──► CloudWatch Logs     │
  │  ap-south-1 ─┤                                              │
  │  sa-east-1 ──┘                                              │
  │                                                             │
  │  ✅ Captures events from ALL regions                        │
  │  ✅ Includes events from NEW regions automatically          │
  │  ✅ AWS best practice for security                          │
  └─────────────────────────────────────────────────────────────┘

  Single-Region Trail (CLI/API default):
  ┌─────────────────────────────┐
  │  us-east-1 only ──► S3     │
  │                             │
  │  ❌ Misses other regions    │
  └─────────────────────────────┘
```

### Organization Trail

```
  ┌──────────────────────────────────────────────────────────────┐
  │                  AWS Organization                            │
  │                                                              │
  │  Management Account                                          │
  │  ┌────────────────────────────────┐                          │
  │  │  Organization Trail           │                          │
  │  │  (created in mgmt account)    │                          │
  │  └────────────┬───────────────────┘                          │
  │               │                                              │
  │  ┌────────────▼───────────────────┐                          │
  │  │  Logs ALL member accounts     │                          │
  │  │  across ALL regions            │                          │
  │  │  into ONE S3 bucket            │                          │
  │  └────────────────────────────────┘                          │
  │                                                              │
  │  Member Account A ───► logged                               │
  │  Member Account B ───► logged                               │
  │  Member Account C ───► logged                               │
  │                                                              │
  │  ✅ Centralized audit for entire organization               │
  │  ✅ Member accounts can see trail but cannot modify/delete  │
  └──────────────────────────────────────────────────────────────┘
```

---

## CloudTrail Log Delivery & Storage

```
  API Call ──► CloudTrail ──► S3 Bucket (delivered within ~5-15 min)
                    │
                    ├──► CloudWatch Logs (for real-time monitoring)
                    │         │
                    │         ├──► Metric Filter: "AccessDenied"
                    │         │         │
                    │         │         └──► CloudWatch Alarm ──► SNS
                    │         │
                    │         └──► Metric Filter: "ConsoleLogin without MFA"
                    │                   │
                    │                   └──► CloudWatch Alarm ──► SNS
                    │
                    ├──► EventBridge (automated response)
                    │         │
                    │         └──► Rule: DeleteTrail ──► Lambda (re-enable)
                    │
                    └──► SNS Topic (raw notifications)
```

### Log File Integrity Validation

| Feature | Description |
|---|---|
| **Digest files** | CloudTrail creates a hash digest every hour |
| **Purpose** | Detect if log files were modified, deleted, or forged after delivery |
| **How** | SHA-256 hashing + digital signing with AWS private key |
| **Validation** | Use `aws cloudtrail validate-logs` CLI command |
| **Exam tip** | "Ensure log files haven't been tampered with" → **Enable Log File Integrity Validation** |

### S3 Integration Best Practices

- **SSE-S3** or **SSE-KMS** encryption (default: SSE-S3)
- **S3 Object Lock** — prevent deletion (WORM compliance)
- **S3 Lifecycle Rules** — transition old logs to Glacier
- **S3 Bucket Policy** — restrict access to security team only
- **S3 Access Logging** — audit who accessed the CloudTrail logs themselves

> [!TIP]
> **Exam Pattern:** "Prevent CloudTrail logs from being deleted or modified" → **S3 Object Lock** (Compliance mode) + **Log File Integrity Validation** + **MFA Delete** on the S3 bucket.

---

## CloudTrail + CloudWatch Logs Integration

```
  Use Case: Real-time alerting on suspicious activity

  Step 1: Trail → delivers to CloudWatch Log Group
  Step 2: Metric Filter → searches for pattern (e.g., "UnauthorizedAccess")
  Step 3: CloudWatch Alarm → triggers when count > threshold
  Step 4: SNS → emails security team

  Example Metric Filters:
  ┌──────────────────────────────────────────────────────────────┐
  │ Filter Pattern                    │ Alert On                │
  │───────────────────────────────────│─────────────────────────│
  │ { $.errorCode = "AccessDenied" }  │ Unauthorized API calls  │
  │ { $.eventName = "ConsoleLogin"    │ Root account login      │
  │   && $.userIdentity.type="Root" } │                         │
  │ { $.eventName = "StopLogging" }   │ Someone disabled trail  │
  │ { $.eventName = "DeleteTrail" }   │ Someone deleted trail   │
  │ { $.eventName = "AuthorizeSecurityGroupIngress" }           │
  │                                   │ Security group changed  │
  └──────────────────────────────────────────────────────────────┘
```

---

## CloudTrail + EventBridge

```
  Every API call recorded by CloudTrail → event on default EventBridge bus

  Example rules:
  ┌─────────────────────────────────────────────────────────────────────┐
  │ Event Pattern                       │ Target                       │
  │─────────────────────────────────────│──────────────────────────────│
  │ source: "aws.iam"                   │ Lambda: revoke permission   │
  │ detail: CreateUser                  │ SNS: alert security team    │
  │                                     │                              │
  │ source: "aws.s3"                    │ Lambda: re-enable encryption│
  │ detail: DeleteBucketEncryption      │                              │
  │                                     │                              │
  │ source: "aws.ec2"                   │ SNS: alert + SSM remediate  │
  │ detail: AuthorizeSecurityGroupIngress                              │
  └─────────────────────────────────────────────────────────────────────┘
```

> [!NOTE]
> **EventBridge** reacts to CloudTrail events in near-real-time (seconds). This is faster than the S3 delivery path (5-15 min). Use EventBridge for **automated remediation** and **real-time security response**.

---

## CloudTrail Event Record Structure

```json
  {
    "eventVersion": "1.08",
    "userIdentity": {
      "type": "IAMUser",
      "arn": "arn:aws:iam::123456789012:user/Alice",
      "accountId": "123456789012"
    },
    "eventTime": "2026-06-06T10:30:00Z",
    "eventSource": "ec2.amazonaws.com",
    "eventName": "RunInstances",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "203.0.113.50",
    "userAgent": "aws-cli/2.15.0",
    "requestParameters": { "instanceType": "t3.micro" },
    "responseElements": { "instanceId": "i-0abc123def456" },
    "readOnly": false
  }
```

---

## CloudTrail Lake (Advanced)

| Feature | Description |
|---|---|
| **Purpose** | Managed data lake for CloudTrail events |
| **Query** | SQL-based queries directly on events (no S3 + Athena setup needed) |
| **Retention** | Up to 7 years (configurable) |
| **Scope** | Can aggregate events from multiple accounts + organizations |
| **Use case** | Advanced security analytics, long-term compliance queries |

---

## Event History vs Trail vs CloudTrail Lake

| Feature | Event History | Trail | CloudTrail Lake |
|---|---|---|---|
| **Setup required** | None (always on) | Create trail | Create event data store |
| **Retention** | 90 days | Unlimited (S3) | Up to 2,555 days (7 years) |
| **Event types** | Management only | Management + Data + Insights | Management + Data + Insights |
| **Delivery** | Console only | S3, CloudWatch Logs, EventBridge | Built-in SQL query |
| **Cost** | Free | First mgmt copy free | Pay for ingestion + storage |
| **Multi-account** | ❌ | ✅ (Organization Trail) | ✅ (Organization data store) |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → CloudTrail:**
> - "Who deleted the S3 bucket / EC2 instance?" → **CloudTrail** (Management Events)
> - "Who accessed S3 objects (GetObject)?" → **CloudTrail Data Events** (must enable)
> - "Log all API calls across all regions" → **Multi-region trail** (best practice)
> - "Centralized audit for AWS Organization" → **Organization Trail** (mgmt account)
> - "Ensure logs are tamper-proof" → **Log File Integrity Validation** + S3 Object Lock
> - "Real-time alert on suspicious API activity" → CloudTrail → **CloudWatch Logs** → Metric Filter → Alarm
> - "Automated response to security events" → CloudTrail → **EventBridge** → Lambda
> - "Detect unusual API patterns/spikes" → **CloudTrail Insights**
> - "SQL queries on CloudTrail events" → **CloudTrail Lake** or S3 + Athena
> - "Root account login alert" → CloudTrail → CloudWatch Logs → Metric Filter → Alarm → SNS
>
> **Key facts:**
> - Event History: free, 90 days, management events only, always on.
> - Management Events logged by default. Data Events and Insights are OFF by default.
> - Console creates multi-region trails. CLI/API creates single-region by default.
> - Log delivery to S3 takes **5-15 minutes** (not real-time).
> - For real-time: use CloudWatch Logs integration or EventBridge.
> - First copy of management events to S3 is **free**. Additional copies/data events cost extra.
> - CloudTrail ≠ [[Amazon CloudWatch]] (performance) ≠ [[AWS Config]] (compliance rules).
