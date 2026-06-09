---
tags: [concept, security, governance, organizations, multi-account, scp, consolidated-billing]
aliases: [AWS Organizations, Organizations, SCPs, Service Control Policies, OUs, Organizational Units, Consolidated Billing]
date: 2026-06-09
---

# AWS Organizations

**AWS Organizations** is the **multi-account management** service. It enables you to centrally govern multiple AWS accounts — consolidate billing, apply security guardrails (SCPs), automate account creation, and share resources across accounts. Think of it as the **corporate headquarters** governing all branch offices.

> [!IMPORTANT]
> **Core exam concept:** Organizations = **centralized governance** for multi-account environments. SCPs = **guardrails** that restrict maximum permissions (they do NOT grant permissions). The **management account** is never affected by SCPs.

---

## Architecture Overview

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                     AWS Organization                              │
  │                                                                   │
  │  ┌──────────────────────────────────────────────────────────┐    │
  │  │                     Root (top-level)                      │    │
  │  │                                                           │    │
  │  │  Management Account (payer)                               │    │
  │  │  • Creates Organization       • Invites accounts          │    │
  │  │  • Manages SCPs               • Pays all bills            │    │
  │  │  • NOT affected by SCPs       • Should be minimal-use     │    │
  │  └──────────────┬────────────────────────────┬───────────────┘    │
  │                 │                            │                    │
  │      ┌──────────▼──────────┐     ┌──────────▼──────────┐        │
  │      │   OU: Production    │     │   OU: Development   │        │
  │      │                     │     │                     │        │
  │      │  ┌──────┐ ┌──────┐ │     │  ┌──────┐ ┌──────┐ │        │
  │      │  │Acct A│ │Acct B│ │     │  │Acct C│ │Acct D│ │        │
  │      │  └──────┘ └──────┘ │     │  └──────┘ └──────┘ │        │
  │      │                     │     │                     │        │
  │      │  SCP: Deny delete   │     │  SCP: Allow all    │        │
  │      │  CloudTrail         │     │  (less restrictive) │        │
  │      └─────────────────────┘     └─────────────────────┘        │
  │                                                                   │
  │  Features:                                                        │
  │  ✅ Consolidated Billing (single payer, volume discounts)        │
  │  ✅ Service Control Policies (SCPs) — permission guardrails      │
  │  ✅ Automated account creation via API                           │
  │  ✅ Resource sharing via AWS RAM                                  │
  │  ✅ Centralized logging ([[AWS CloudTrail]], [[AWS Config]])      │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### Organization Hierarchy

```
  Root
  ├── Management Account (payer, admin — always 1)
  ├── OU: Infrastructure
  │   ├── Shared Services Account
  │   └── Networking Account
  ├── OU: Security
  │   ├── Log Archive Account
  │   └── Security Tooling Account
  ├── OU: Workloads
  │   ├── OU: Production
  │   │   ├── App A Prod Account
  │   │   └── App B Prod Account
  │   └── OU: Development
  │       ├── App A Dev Account
  │       └── App B Dev Account
  └── OU: Sandbox
      └── Experimentation Account

  Rules:
  • An account belongs to exactly ONE OU
  • OUs can be nested (up to 5 levels deep)
  • SCPs are inherited down the tree
  • An account can be moved between OUs
```

### Key Terminology

| Term | Description |
|---|---|
| **Management Account** | The account that creates the Organization. Pays all bills. Cannot be restricted by SCPs. |
| **Member Account** | Any account joined/created within the Organization. Subject to SCPs. |
| **Organizational Unit (OU)** | Logical grouping of accounts. SCPs and policies attach here. |
| **Root** | Top-level container for all accounts and OUs. |
| **Invitation** | Existing accounts join by invitation from the management account. |
| **Account Factory** | Automated account creation (via API or [[AWS Control Tower]]). |

> [!WARNING]
> **The management account should be used ONLY for Organization management** — never run workloads in it. It cannot be restricted by SCPs, making it a high-privilege target. Enable MFA on root, use minimal IAM users.

---

## Organization Feature Sets

| Feature Set | Consolidated Billing Only | All Features (recommended) |
|---|---|---|
| **Billing** | ✅ Single payer | ✅ Single payer |
| **Volume discounts** | ✅ Aggregated usage | ✅ Aggregated usage |
| **SCPs** | ❌ | ✅ |
| **Tag policies** | ❌ | ✅ |
| **Backup policies** | ❌ | ✅ |
| **AI services opt-out** | ❌ | ✅ |
| **AWS Config/CloudTrail org-wide** | ❌ | ✅ |

---

## Service Control Policies (SCPs) — Deep Dive

### What SCPs Are (and Are Not)

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                    Service Control Policies                       │
  │                                                                   │
  │  SCPs define the MAXIMUM permissions boundary for accounts       │
  │                                                                   │
  │  ✅ SCPs DO:                      ❌ SCPs DO NOT:                │
  │  • Restrict what actions are      • Grant permissions             │
  │    allowed in member accounts     • Affect management account     │
  │  • Apply to ALL users & roles     • Affect service-linked roles  │
  │    in the account (incl. root)    • Restrict resource-based       │
  │  • Inherit down the OU tree         policies for cross-account   │
  │  • Filter available permissions                                   │
  │                                                                   │
  │  Think of SCPs as a FILTER, not a GRANT:                         │
  │                                                                   │
  │  IAM Policy (Allow s3:*)                                          │
  │       │                                                           │
  │       ▼                                                           │
  │  SCP Filter (Allow s3:Get*, s3:List*)                            │
  │       │                                                           │
  │       ▼                                                           │
  │  Effective: s3:Get*, s3:List* ONLY                               │
  │  (s3:Put*, s3:Delete* blocked by SCP)                            │
  └──────────────────────────────────────────────────────────────────┘
```

### SCP Inheritance

```
  Root ── SCP: FullAWSAccess (default)
    │
    ├── OU: Production ── SCP: DenyDeleteTrail, DenyLeaveOrg
    │   │
    │   ├── Account A: Effective SCP = FullAWSAccess
    │   │   MINUS DenyDeleteTrail, DenyLeaveOrg
    │   │
    │   └── Sub-OU: Critical ── SCP: DenyEC2Stop
    │       │
    │       └── Account B: Effective SCP = FullAWSAccess
    │           MINUS DenyDeleteTrail, DenyLeaveOrg, DenyEC2Stop
    │
    └── OU: Sandbox ── SCP: AllowOnlyApprovedServices
        │
        └── Account C: Can ONLY use services in the allow-list

  Rules:
  • SCPs at Root apply to ALL member accounts
  • Child OUs inherit parent SCPs (intersection)
  • An account's effective permissions = intersection of ALL SCPs
    from Root down to its OU
  • Explicit Deny in any SCP overrides Allow at any level
```

### SCP Strategies

```
  Strategy 1: DENY LIST (recommended)
  ┌─────────────────────────────────────────┐
  │  Default: FullAWSAccess (AWS managed)   │
  │  + Add explicit Deny statements         │
  │                                          │
  │  Example: Deny specific dangerous APIs  │
  │  "Effect": "Deny",                      │
  │  "Action": [                            │
  │    "organizations:LeaveOrganization",   │
  │    "cloudtrail:StopLogging",            │
  │    "cloudtrail:DeleteTrail",            │
  │    "ec2:DisableEbsEncryptionByDefault"  │
  │  ]                                      │
  │                                          │
  │  ✅ Easy to manage — block what's bad   │
  │  ✅ New services are allowed by default │
  └─────────────────────────────────────────┘

  Strategy 2: ALLOW LIST
  ┌─────────────────────────────────────────┐
  │  Remove: FullAWSAccess                  │
  │  + Add explicit Allow only for approved │
  │    services                              │
  │                                          │
  │  Example: Allow only EC2, S3, RDS       │
  │  "Effect": "Allow",                     │
  │  "Action": [                            │
  │    "ec2:*", "s3:*", "rds:*"            │
  │  ]                                      │
  │                                          │
  │  ⚠️ Very restrictive                    │
  │  ⚠️ New services blocked by default     │
  │  ⚠️ Must explicitly allow each service  │
  └─────────────────────────────────────────┘
```

### Common SCP Examples

```json
  // 1. Prevent leaving the Organization
  {
    "Effect": "Deny",
    "Action": "organizations:LeaveOrganization",
    "Resource": "*"
  }

  // 2. Restrict to specific regions
  {
    "Effect": "Deny",
    "NotAction": [
      "iam:*", "sts:*", "organizations:*",
      "support:*", "budgets:*"
    ],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:RequestedRegion": ["eu-west-1", "eu-central-1"]
      }
    }
  }

  // 3. Require encryption on S3
  {
    "Effect": "Deny",
    "Action": "s3:PutObject",
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "s3:x-amz-server-side-encryption": "aws:kms"
      }
    }
  }

  // 4. Prevent disabling CloudTrail
  {
    "Effect": "Deny",
    "Action": [
      "cloudtrail:StopLogging",
      "cloudtrail:DeleteTrail",
      "cloudtrail:UpdateTrail"
    ],
    "Resource": "*"
  }

  // 5. Prevent root user access (except for root-only tasks)
  {
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "StringLike": {
        "aws:PrincipalArn": "arn:aws:iam::*:root"
      }
    }
  }
```

> [!CAUTION]
> **Exam critical:** When restricting regions with SCPs, you MUST use `NotAction` to exclude **global services** (IAM, STS, Organizations, Support, Billing) — otherwise you'll break account functionality. Global services always run in `us-east-1` regardless of your region.

---

## Consolidated Billing

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                     Consolidated Billing                          │
  │                                                                   │
  │  Management Account (Payer)                                       │
  │  ┌─────────────────────────────────────────────────────────┐     │
  │  │  Single invoice for ALL accounts                        │     │
  │  │                                                          │     │
  │  │  Account A: 5 TB S3 ─┐                                  │     │
  │  │  Account B: 3 TB S3 ─┼─► Aggregated: 10 TB S3          │     │
  │  │  Account C: 2 TB S3 ─┘   (higher volume = lower rate)  │     │
  │  │                                                          │     │
  │  │  Account A: 3 Reserved Instances ─┐                     │     │
  │  │  Account B: 0 instances (uses RI) ─┤  RI sharing across│     │
  │  │  Account C: 2 On-Demand           ─┘  accounts          │     │
  │  └─────────────────────────────────────────────────────────┘     │
  │                                                                   │
  │  Benefits:                                                        │
  │  ✅ Volume pricing discounts (S3, EC2, data transfer)            │
  │  ✅ Reserved Instance / Savings Plan sharing across accounts     │
  │  ✅ Single payment method                                        │
  │  ✅ Track costs per account with Cost Explorer tags              │
  │                                                                   │
  │  RI Sharing:                                                      │
  │  • Can be disabled per account if needed                         │
  │  • Unused RI capacity in Account A can be used by Account B     │
  │  • Savings Plans also shared across Organization                 │
  └──────────────────────────────────────────────────────────────────┘
```

> [!TIP]
> **Exam Pattern:** "Reduce overall AWS costs across multiple accounts" → **Consolidated Billing** (volume discounts + RI/Savings Plan sharing). "One account has unused Reserved Instances" → RI sharing across the Organization (enabled by default).

---

## Multi-Account Strategies

### Why Multi-Account?

| Reason | Explanation |
|---|---|
| **Blast radius isolation** | A security breach in dev doesn't affect production |
| **Billing isolation** | Track costs per team/project/environment |
| **Service limit isolation** | Each account has independent service quotas |
| **Compliance** | Separate accounts for PCI, HIPAA workloads |
| **Team autonomy** | Teams manage their own account with guardrails |

### Recommended OU Structure (AWS Best Practice)

```
  Root
  ├── OU: Security
  │   ├── Log Archive (centralized [[AWS CloudTrail]], [[AWS Config]])
  │   └── Security Tooling (GuardDuty delegated admin, Security Hub)
  │
  ├── OU: Infrastructure
  │   ├── Shared Services (Active Directory, DNS, CI/CD)
  │   └── Network (Transit Gateway, Direct Connect, VPN)
  │
  ├── OU: Workloads
  │   ├── OU: Production
  │   │   └── App accounts (strict SCPs)
  │   └── OU: Non-Production
  │       ├── OU: Staging
  │       └── OU: Development (lenient SCPs)
  │
  ├── OU: Sandbox
  │   └── Experimentation (budget limits, auto-cleanup)
  │
  ├── OU: Policy Staging
  │   └── Test new SCPs before applying to Workloads
  │
  └── OU: Suspended
      └── Accounts pending closure
```

---

## AWS Control Tower

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                     AWS Control Tower                              │
  │                                                                   │
  │  "Easy button" for setting up a governed multi-account env       │
  │                                                                   │
  │  Built on top of:                                                 │
  │  • AWS Organizations (account structure)                         │
  │  • SCPs (preventive guardrails)                                  │
  │  • AWS Config Rules (detective guardrails)                       │
  │  • AWS CloudFormation StackSets (baseline config)                │
  │  • AWS IAM Identity Center (SSO)                                 │
  │                                                                   │
  │  ┌────────────────────────────────────────────────────────┐      │
  │  │  Landing Zone (auto-configured)                        │      │
  │  │                                                         │      │
  │  │  • Management Account                                  │      │
  │  │  • Log Archive Account (centralized logs)              │      │
  │  │  • Audit Account (cross-account security access)       │      │
  │  │  • Pre-configured OUs (Security, Sandbox)              │      │
  │  │  • Guardrails (mandatory + elective + strongly rec.)   │      │
  │  │  • Account Factory (self-service account provisioning) │      │
  │  │  • Dashboard (compliance overview)                     │      │
  │  └────────────────────────────────────────────────────────┘      │
  │                                                                   │
  │  Guardrail Types:                                                 │
  │  ┌──────────────────┬──────────────────┬──────────────────┐      │
  │  │  Preventive      │  Detective       │  Proactive       │      │
  │  │  (SCPs)          │  (Config Rules)  │  (CFN Hooks)     │      │
  │  │  Block actions   │  Flag violations │  Pre-deploy check│      │
  │  │  e.g., deny      │  e.g., detect    │  e.g., validate  │      │
  │  │  public S3       │  unencrypted EBS │  CFN template    │      │
  │  └──────────────────┴──────────────────┴──────────────────┘      │
  └──────────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Exam Pattern:** "Set up a secure multi-account environment with LEAST effort" → **AWS Control Tower**. "Automated account provisioning with pre-configured guardrails" → **Control Tower Account Factory**. Control Tower automates what you'd otherwise build manually with Organizations + SCPs + Config + CloudFormation.

---

## Organization-Wide Services Integration

| Service | Organization Feature |
|---|---|
| **[[AWS CloudTrail]]** | Organization Trail — logs ALL accounts to one S3 bucket |
| **[[AWS Config]]** | Aggregator — compliance view across all accounts |
| **GuardDuty** | Delegated admin — threat detection across all accounts |
| **Security Hub** | Aggregated security findings from all accounts |
| **AWS RAM** | Share resources (subnets, Transit GW, License Manager) across accounts |
| **AWS Backup** | Organization-wide backup policies |
| **Tag Policies** | Enforce consistent tagging across accounts |
| **SSO / [[AWS IAM Advanced\|IAM Identity Center]]** | Single sign-on portal for all accounts |
| **Cost Explorer** | Cost analysis broken down by account/OU |
| **Budgets** | Set budgets per account or linked account |

### AWS Resource Access Manager (RAM)

```
  ┌──────────────────────────────────────────────────────────────┐
  │  AWS RAM — Share resources across Organization accounts       │
  │                                                               │
  │  Shareable resources:                                        │
  │  • VPC Subnets (share network across accounts)               │
  │  • Transit Gateway                                            │
  │  • Route 53 Resolver Rules                                   │
  │  • License Manager Configurations                            │
  │  • Aurora DB Clusters                                        │
  │  • AWS Glue Catalog                                          │
  │  • EC2 Capacity Reservations                                 │
  │                                                               │
  │  Account A (Network)          Account B (Workload)           │
  │  ┌────────────────┐          ┌────────────────┐              │
  │  │ VPC            │  shared  │ Launches EC2   │              │
  │  │ ├── Subnet-1  ─┼─────────┤ in Subnet-1   │              │
  │  │ └── Subnet-2   │  via RAM │ (Account B's   │              │
  │  └────────────────┘          │  own instances) │              │
  │                               └────────────────┘              │
  │                                                               │
  │  ✅ No VPC peering needed                                    │
  │  ✅ Resources stay in owner account                          │
  │  ✅ Each account manages its own resources in shared subnet  │
  └──────────────────────────────────────────────────────────────┘
```

---

## Tag Policies

```
  Tag Policies enforce consistent tagging across the Organization

  Example: Enforce "CostCenter" tag with specific values
  {
    "tags": {
      "CostCenter": {
        "tag_key": { "@@assign": "CostCenter" },
        "tag_value": {
          "@@assign": ["100", "200", "300"]
        },
        "enforced_for": {
          "@@assign": ["ec2:instance", "s3:bucket"]
        }
      }
    }
  }

  • Standardize tag key capitalization
  • Restrict tag values to approved list
  • Enforce on specific resource types
  • NON-COMPLIANT resources are flagged (not blocked)
```

---

## Account Management

### Creating vs Inviting Accounts

| Method | Behavior |
|---|---|
| **Create (API/Console)** | New account created inside Organization. Automatically joined. |
| **Invite** | Existing account receives invitation. Must accept. |
| **Remove/Leave** | Account becomes standalone. Must have payment method configured. |

### Closing Accounts

- Account enters **90-day suspended** period after close request
- During suspension: no new resources, existing resources still billed
- After 90 days: permanently closed, resources deleted
- Move to **Suspended OU** during this period (best practice)

---

## Organizations + IAM Policy Evaluation

```
  API Request in a Member Account:
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  Step 1: SCP Check (Organization level)                     │
  │  ┌────────────────────────────────────────────────┐         │
  │  │ Is the action allowed by ALL SCPs from Root    │         │
  │  │ down to this account's OU?                     │         │
  │  │                                                 │         │
  │  │ NO → DENIED (SCP blocks it)                    │         │
  │  │ YES → Continue to Step 2                       │         │
  │  └────────────────────────────────────────────────┘         │
  │                                                              │
  │  Step 2: IAM Policy Check (Account level)                   │
  │  ┌────────────────────────────────────────────────┐         │
  │  │ Standard [[AWS IAM Advanced|IAM evaluation]]:  │         │
  │  │ Explicit Deny → Resource-based Allow →         │         │
  │  │ Identity-based Allow → Permissions Boundary → │         │
  │  │ Session Policy                                 │         │
  │  └────────────────────────────────────────────────┘         │
  │                                                              │
  │  Both SCP AND IAM must allow = action proceeds              │
  └──────────────────────────────────────────────────────────────┘
```

> [!TIP]
> **Remember:** SCPs are like a **ceiling** — they set the maximum. IAM policies are the actual **grant**. You need BOTH to allow. SCP Allow + IAM Deny = Denied. SCP Deny + IAM Allow = Denied. Only SCP Allow + IAM Allow = Allowed.

---

## Key Limits

| Parameter | Value |
|---|---|
| **Accounts per Organization** | Default 10 (can increase to thousands) |
| **OUs per Organization** | 1,000 |
| **OU nesting depth** | 5 levels (+ root) |
| **SCPs per Organization** | 1,000 |
| **SCPs per OU/account** | 5 attached |
| **SCP document size** | 5,120 characters |
| **Tag policies per Organization** | 1,000 |
| **Delegated administrators** | Up to 3 per service |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Organizations:**
> - "Centrally manage multiple AWS accounts" → **AWS Organizations**
> - "Single bill, volume discounts" → **Consolidated Billing**
> - "Restrict services/regions across all accounts" → **SCPs**
> - "Prevent member accounts from leaving" → SCP: `Deny organizations:LeaveOrganization`
> - "Prevent disabling CloudTrail in any account" → SCP: `Deny cloudtrail:StopLogging`
> - "Set up multi-account with least effort" → **AWS Control Tower**
> - "Share VPC subnets across accounts" → **AWS RAM**
> - "Enforce consistent tags across accounts" → **Tag Policies**
> - "Automated account provisioning with guardrails" → **Control Tower Account Factory**
> - "Preventive guardrails" → **SCPs**. "Detective guardrails" → **AWS Config Rules**
> - "Unused Reserved Instances in one account, use in another" → **RI sharing** (default on)
> - "Centralized audit logging for all accounts" → **Organization Trail** ([[AWS CloudTrail]])
> - "Restrict to EU regions only for compliance" → SCP with `aws:RequestedRegion` + `NotAction` for global services
>
> **Key facts:**
> - SCPs do NOT grant permissions — they only restrict (filter).
> - SCPs do NOT affect the management account — ever.
> - SCPs affect ALL users and roles in member accounts, including root user.
> - SCPs do NOT affect service-linked roles.
> - Default SCP: `FullAWSAccess` (allow all) — remove this for allow-list strategy.
> - Region restriction SCPs must exclude global services via `NotAction` (IAM, STS, etc.).
> - Consolidated Billing aggregates usage for volume discounts and shares RI/Savings Plans.
> - Control Tower = Organizations + SCPs + Config + CloudFormation + IAM Identity Center.
> - AWS RAM shares resources (VPC subnets, Transit GW) WITHOUT VPC peering.
> - Tag Policies flag non-compliant resources but do NOT block resource creation.
