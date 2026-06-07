---
tags: [concept, security, compliance, governance, config, audit, remediation]
aliases: [Config, AWS Config, Configuration Recorder, Config Rules, Conformance Packs]
date: 2026-06-06
---

# AWS Config

**AWS Config** is the **compliance and configuration auditing** service for AWS. It answers the question: **"Is my resource configuration compliant?"** — continuously recording and evaluating the configuration of your AWS resources against desired rules. Think of it as the **compliance auditor** that watches every resource change.

> [!IMPORTANT]
> **Core exam concept:** AWS Config = **configuration compliance** (record changes, evaluate rules, remediate). It does NOT monitor performance (that's [[Amazon CloudWatch]]) or track who made API calls (that's [[AWS CloudTrail]]). Config tells you **WHAT changed** and **if it's compliant**. CloudTrail tells you **WHO changed it**.

---

## Architecture Overview

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                          AWS Config                                  │
  │                                                                      │
  │  ┌────────────────────┐    ┌────────────────────┐                   │
  │  │ Configuration      │    │  Config Rules       │                   │
  │  │ Recorder           │    │  (evaluate)         │                   │
  │  │                    │    │                      │                   │
  │  │ Records every      │───►│ AWS Managed Rules   │                   │
  │  │ configuration      │    │ Custom Rules (Lambda)│                   │
  │  │ change as a        │    │                      │                   │
  │  │ Configuration Item │    │ Result:              │                   │
  │  │ (CI)               │    │ ✅ COMPLIANT         │                   │
  │  └────────────────────┘    │ ❌ NON_COMPLIANT     │                   │
  │                            └──────────┬───────────┘                   │
  │                                       │                               │
  │  ┌────────────────────┐    ┌──────────▼───────────┐                  │
  │  │ Delivery Channel   │    │  Remediation          │                  │
  │  │                    │    │  (auto-fix)            │                  │
  │  │ ──► S3 Bucket      │    │                        │                  │
  │  │     (config history)│    │  SSM Automation        │                  │
  │  │ ──► SNS Topic      │    │  Document               │                  │
  │  │     (notifications)│    │  (e.g., stop instance, │                  │
  │  └────────────────────┘    │   enable encryption)   │                  │
  │                            └────────────────────────┘                  │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Configuration Recorder

```
  Resource Change Detected:
  ┌──────────────────┐     ┌─────────────────────────────────────┐
  │  EC2 Instance    │     │  Configuration Item (CI)             │
  │  i-abc123        │────►│                                      │
  │                  │     │  Resource Type: AWS::EC2::Instance   │
  │  Security Group  │     │  Resource ID:   i-abc123             │
  │  changed!        │     │  Configuration: {                    │
  │                  │     │    "instanceType": "t3.micro",       │
  │                  │     │    "securityGroups": ["sg-open"],     │
  │                  │     │    "subnetId": "subnet-abc"          │
  │                  │     │  }                                    │
  └──────────────────┘     │  Relationships: [VPC, Subnet, SG]   │
                           │  Timestamp: 2026-06-06T10:30:00Z     │
                           └─────────────────────────────────────┘

  • Records EVERY configuration change
  • Must be enabled per-region
  • Stores Configuration Items (CIs) in S3
  • Creates a timeline of changes for each resource
```

### 2. Config Rules

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  AWS Managed Rules (pre-built, 250+ available)                  │
  │                                                                  │
  │  Examples:                                                       │
  │  • s3-bucket-public-read-prohibited                             │
  │  • restricted-ssh (no 0.0.0.0/0 on port 22)                    │
  │  • ec2-instance-no-public-ip                                    │
  │  • rds-instance-public-access-check                             │
  │  • encrypted-volumes (EBS encryption check)                     │
  │  • iam-root-access-key-check                                    │
  │  • required-tags                                                 │
  │  • cloudtrail-enabled                                            │
  │  • rds-multi-az-support                                          │
  │  • ebs-snapshot-public-restorable-check                         │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  Custom Rules (your own logic)                                   │
  │                                                                  │
  │  ┌────────────────┐     ┌──────────────────────────────────┐   │
  │  │ Config Rule    │────►│ Lambda Function                   │   │
  │  │ (trigger)      │     │ (evaluation logic)                │   │
  │  └────────────────┘     │                                    │   │
  │                         │ def evaluate(event):               │   │
  │                         │   if port_22_open_to_world:        │   │
  │                         │     return "NON_COMPLIANT"         │   │
  │                         │   return "COMPLIANT"               │   │
  │                         └──────────────────────────────────┘   │
  │                                                                  │
  │  Also: Custom Rules using Guard (declarative, no Lambda)        │
  └──────────────────────────────────────────────────────────────────┘
```

### Rule Trigger Types

| Trigger | When Evaluated | Use Case |
|---|---|---|
| **Configuration change** | When a tracked resource changes | "Check encryption whenever a new EBS volume is created" |
| **Periodic** | On a schedule (1h, 3h, 6h, 12h, 24h) | "Every 24 hours, check all S3 buckets are private" |

> [!TIP]
> **Exam Pattern:** "Ensure all S3 buckets have encryption enabled" → **AWS Config Rule** (`s3-bucket-server-side-encryption-enabled`). "Check all security groups don't allow SSH from 0.0.0.0/0" → **Config Rule** (`restricted-ssh`). "Custom compliance check" → **Custom Config Rule** with Lambda.

### 3. Remediation

```
  ┌───────────────┐     ┌────────────────────┐     ┌──────────────────────┐
  │ Config Rule   │────►│ NON_COMPLIANT      │────►│ Remediation Action   │
  │ evaluation    │     │ resource detected  │     │                      │
  └───────────────┘     └────────────────────┘     │ SSM Automation Doc:  │
                                                    │ • Enable encryption  │
                                                    │ • Close SSH port     │
                                                    │ • Enable versioning  │
                                                    │ • Add required tags  │
                                                    └──────────────────────┘

  Remediation Types:
  ┌─────────────────────────────────────────────────────────────────┐
  │ Automatic Remediation    │ Manual Remediation                  │
  │                          │                                      │
  │ Runs SSM doc immediately │ You review and click "Remediate"    │
  │ when non-compliant       │ in the console (or CLI/API)         │
  │                          │                                      │
  │ ⚠️ Can set retry         │ ✅ Safer for critical resources     │
  │   attempts (up to 5)     │                                      │
  └─────────────────────────────────────────────────────────────────┘
```

> [!CAUTION]
> **Exam critical:** Config Rules are **detective, not preventive**. They evaluate AFTER a change happens and flag non-compliance. They do NOT prevent the change from occurring. For prevention → use **IAM policies**, **SCPs**, or **AWS Organizations**. Config can then **auto-remediate** after detection.

---

## Configuration Timeline

```
  Resource: sg-abc123 (Security Group)

  Time ──────────────────────────────────────────────────────────────►

  T1: Created              T2: Rule added         T3: Rule removed
  port 443 (HTTPS)         port 22 (SSH)           port 22 removed
  ┌──────────┐            ┌──────────┐            ┌──────────┐
  │ CI #1    │            │ CI #2    │            │ CI #3    │
  │          │            │          │            │          │
  │ Ingress: │            │ Ingress: │            │ Ingress: │
  │ 443/tcp  │            │ 443/tcp  │            │ 443/tcp  │
  │          │            │ 22/tcp   │            │          │
  │ ✅ COMP  │            │ ❌ NON   │            │ ✅ COMP  │
  └──────────┘            └──────────┘            └──────────┘
                                │
                                ▼
                          Who changed it?
                          → Check CloudTrail!
                          (Config + CloudTrail together)
```

> [!NOTE]
> **Config + CloudTrail synergy:** AWS Config records **WHAT** changed (configuration state). [[AWS CloudTrail]] records **WHO** changed it (API call). Together, they provide a complete audit trail: what happened, who did it, and was it compliant?

---

## AWS Config Aggregator

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                   Aggregator Account                             │
  │                   (Central Security Account)                     │
  │                                                                  │
  │  ┌────────────────────────────────────────────────────────────┐ │
  │  │              Config Aggregator (READ-ONLY)                 │ │
  │  │                                                            │ │
  │  │  Aggregates from:                                          │ │
  │  │  • Individual accounts (with authorization)                │ │
  │  │  • OR entire AWS Organization (no auth needed)             │ │
  │  │  • Multiple regions                                        │ │
  │  │                                                            │ │
  │  │  Provides:                                                 │ │
  │  │  ✅ Unified compliance dashboard                           │ │
  │  │  ✅ Advanced SQL-like queries across all accounts          │ │
  │  │  ✅ Resource relationships across organization             │ │
  │  │  ❌ Cannot deploy rules (read-only view)                  │ │
  │  └────────────────────────────────────────────────────────────┘ │
  │                                                                  │
  │  ◄──── Account A (us-east-1, eu-west-1)                        │
  │  ◄──── Account B (us-east-1, ap-south-1)                       │
  │  ◄──── Account C (all regions)                                  │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Conformance Packs

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  Conformance Pack: "PCI-DSS-Operational-Best-Practices"         │
  │                                                                  │
  │  YAML Template containing:                                       │
  │  ┌─────────────────────────────────────────────────────────┐    │
  │  │ Rule 1: encrypted-volumes                               │    │
  │  │ Rule 2: s3-bucket-server-side-encryption-enabled        │    │
  │  │ Rule 3: rds-storage-encrypted                           │    │
  │  │ Rule 4: cloudtrail-enabled                               │    │
  │  │ Rule 5: iam-root-access-key-check                       │    │
  │  │ Rule 6: restricted-ssh                                   │    │
  │  │ ...                                                      │    │
  │  │ Remediation: auto-enable encryption (SSM doc)           │    │
  │  └─────────────────────────────────────────────────────────┘    │
  │                                                                  │
  │  ✅ Deploy as a single entity across accounts                   │
  │  ✅ Pack-level compliance score                                  │
  │  ✅ Pre-built templates for HIPAA, PCI-DSS, NIST, CIS          │
  │  ✅ Deploy via AWS Organizations (StackSets)                    │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Notifications and Automation

```
  Resource changes / compliance changes:

  AWS Config ──► EventBridge ──► Lambda (auto-remediate)
                             ──► SNS (alert security team)
                             ──► SQS (queue for processing)
                             ──► SSM Automation (run remediation)

  AWS Config ──► SNS Topic (delivery channel notification)
                 • Configuration item changes
                 • Config rule compliance changes
                 • Configuration snapshot delivery

  AWS Config ──► S3 Bucket (configuration history + snapshots)
```

---

## Common Config Rule Examples

| Rule Name | What It Checks |
|---|---|
| `s3-bucket-public-read-prohibited` | No public read access on S3 buckets |
| `s3-bucket-server-side-encryption-enabled` | S3 bucket has encryption |
| `restricted-ssh` | No security group allows SSH from 0.0.0.0/0 |
| `ec2-instance-no-public-ip` | EC2 instances don't have public IPs |
| `encrypted-volumes` | All EBS volumes are encrypted |
| `rds-instance-public-access-check` | RDS is not publicly accessible |
| `rds-multi-az-support` | RDS has Multi-AZ enabled |
| `cloudtrail-enabled` | CloudTrail is logging |
| `iam-root-access-key-check` | Root account has no access keys |
| `required-tags` | Resources have mandatory tags |
| `eip-attached` | All EIPs are attached to resources |
| `vpc-flow-logs-enabled` | VPC has flow logs enabled |
| `approved-amis-by-id` | EC2 uses only approved AMIs |

---

## AWS Config vs CloudTrail vs CloudWatch

| Question | Service |
|---|---|
| "What is the **current performance**?" | [[Amazon CloudWatch]] |
| "**Who** made this API call?" | [[AWS CloudTrail]] |
| "**Is this resource compliant** with our rules?" | **AWS Config** |
| "**What was the configuration** at time T?" | **AWS Config** (timeline) |
| "**Alert** when CPU > 80%" | [[Amazon CloudWatch]] Alarm |
| "**Alert** when someone logs in as root" | [[AWS CloudTrail]] → CloudWatch Logs |
| "**Alert** when a resource becomes non-compliant" | **AWS Config** → EventBridge → SNS |
| "**Auto-fix** a non-compliant resource" | **AWS Config** Remediation (SSM) |

---

## Key AWS Config Limits

| Parameter | Value |
|---|---|
| **Config rules per account per region** | 400 (default) |
| **Conformance packs per account** | 50 |
| **Config Aggregators per account** | 50 |
| **Accounts per aggregator** | 10,000 |
| **Configuration recorder** | 1 per region (must enable per region) |
| **Supported resource types** | 300+ |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → AWS Config:**
> - "Ensure all S3 buckets are encrypted" → **Config Rule** (`s3-bucket-server-side-encryption-enabled`)
> - "Check no security group allows unrestricted SSH" → **Config Rule** (`restricted-ssh`)
> - "Automatically fix non-compliant resources" → **Config Remediation** (SSM Automation)
> - "Compliance dashboard across multiple accounts" → **Config Aggregator**
> - "Deploy compliance rules at scale (PCI, HIPAA)" → **Conformance Packs**
> - "History of configuration changes for a resource" → **Config Timeline** (Configuration Items)
> - "What was the config of this resource 3 months ago?" → **AWS Config** (historical CIs)
> - "Evaluate resource on every change" → Config Rule with **configuration change trigger**
> - "Check compliance periodically (daily)" → Config Rule with **periodic trigger**
> - "Custom compliance logic" → **Custom Config Rule** (Lambda or Guard)
>
> **Key facts:**
> - Config is **regional** — must enable per region. Use Aggregator for cross-region/cross-account.
> - Config Rules are **detective** (evaluate after change), NOT **preventive** (don't block changes).
> - Remediation uses **SSM Automation Documents** — can be automatic or manual.
> - Conformance Packs = collection of rules + remediation as a single YAML template.
> - Aggregator is **read-only** — view compliance but can't push rules from aggregator.
> - Config + CloudTrail = complete audit: **what changed** (Config) + **who changed it** (CloudTrail).
> - Config is NOT free — charges per configuration item recorded + rule evaluation.
> - 250+ AWS Managed Rules available for common compliance checks.
