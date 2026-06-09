---
tags: [concept, security, identity, iam, authorization, authentication, policy, federation, advanced]
aliases: [IAM, AWS IAM, IAM Advanced, Identity and Access Management, IAM Policies, IAM Roles, STS, Security Token Service]
date: 2026-06-08
---

# AWS IAM Advanced

**AWS Identity and Access Management (IAM)** is the **foundation of security** in AWS. It controls **who** (authentication) can do **what** (authorization) on **which resources**. This note covers advanced IAM concepts critical for the SAA-C03 exam beyond the basics.

> [!IMPORTANT]
> **Core exam concept:** IAM is a **global service** (not regional). By default, new users have **NO permissions** (implicit deny). Policies are evaluated using the **deny-allow-deny** logic: explicit Deny always wins → then check for explicit Allow → otherwise implicit Deny.

---

## IAM Architecture Overview

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                         AWS IAM (Global)                             │
  │                                                                      │
  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐                │
  │  │  IAM Users  │  │  IAM Groups │  │  IAM Roles   │                │
  │  │  (people)   │  │  (organize  │  │  (temporary  │                │
  │  │             │  │   users)    │  │   assume)    │                │
  │  └──────┬──────┘  └──────┬──────┘  └──────┬───────┘                │
  │         │                │                │                         │
  │         └────────────────┼────────────────┘                         │
  │                          ▼                                          │
  │                 ┌──────────────────┐                                │
  │                 │   IAM Policies   │                                │
  │                 │  (JSON documents │                                │
  │                 │   defining perms)│                                │
  │                 └────────┬─────────┘                                │
  │                          ▼                                          │
  │              ┌──────────────────────┐                               │
  │              │  Policy Evaluation   │                               │
  │              │  Engine              │                               │
  │              │  Deny > Allow > Deny │                               │
  │              └──────────────────────┘                               │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## IAM Policy Types

### Six Policy Types (Evaluation Order)

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  1. SERVICE CONTROL POLICIES (SCPs)         [AWS Organizations] │
  │     • Guardrails on member accounts                             │
  │     • Do NOT grant permissions, only restrict                   │
  │     • Do NOT affect management account                          │
  │                                                                  │
  │  2. RESOURCE-BASED POLICIES                                     │
  │     • Attached to resources (S3 bucket, SQS queue, KMS key)    │
  │     • Specify Principal (who can access)                        │
  │     • Enable CROSS-ACCOUNT access                               │
  │                                                                  │
  │  3. IAM PERMISSIONS BOUNDARIES                                  │
  │     • Maximum permissions an IAM entity CAN have                │
  │     • Used for delegation — let admins create roles safely      │
  │     • Intersect with identity-based policies                    │
  │                                                                  │
  │  4. IDENTITY-BASED POLICIES                                     │
  │     • Attached to users, groups, or roles                       │
  │     │  ├─ AWS Managed (maintained by AWS)                       │
  │     │  ├─ Customer Managed (you create & maintain)              │
  │     │  └─ Inline (embedded directly in one entity)              │
  │                                                                  │
  │  5. SESSION POLICIES                                            │
  │     • Passed when assuming a role or federating                 │
  │     • Further restricts the session's effective permissions     │
  │                                                                  │
  │  6. ACCESS CONTROL LISTS (ACLs)                                 │
  │     • Legacy (S3 & VPC only) — cross-account without IAM       │
  └──────────────────────────────────────────────────────────────────┘
```

### Policy Evaluation Logic

```
  Request arrives
       │
       ▼
  ┌──────────────┐    YES
  │ Explicit DENY │─────────► DENIED (always wins)
  │ anywhere?     │
  └──────┬───────┘
         │ NO
         ▼
  ┌──────────────┐    NO
  │ SCP allows?  │─────────► DENIED
  │ (if in Org)  │
  └──────┬───────┘
         │ YES
         ▼
  ┌──────────────┐    NO
  │ Resource-based│─────────► Check identity policies
  │ policy allows?│
  └──────┬───────┘
         │ YES → ALLOWED (cross-account stops here)
         ▼
  ┌──────────────┐    NO
  │ Identity-based│─────────► DENIED
  │ policy allows?│
  └──────┬───────┘
         │ YES
         ▼
  ┌──────────────┐    NO
  │ Permissions   │─────────► DENIED
  │ boundary      │
  │ allows?       │
  └──────┬───────┘
         │ YES
         ▼
       ALLOWED
```

> [!CAUTION]
> **Exam critical:** For **same-account** access, identity-based OR resource-based policy Allow is sufficient (union). For **cross-account** access, BOTH the resource-based policy on the target AND identity-based policy on the caller must Allow (intersection).

---

## IAM Policy Structure (JSON)

```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowS3ReadOnly",
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:ListBucket"
        ],
        "Resource": [
          "arn:aws:s3:::my-bucket",
          "arn:aws:s3:::my-bucket/*"
        ],
        "Condition": {
          "IpAddress": {
            "aws:SourceIp": "203.0.113.0/24"
          }
        }
      }
    ]
  }
```

### Key Policy Elements

| Element | Required | Description |
|---|---|---|
| **Version** | ✅ | Always `"2012-10-17"` |
| **Statement** | ✅ | Array of permission blocks |
| **Effect** | ✅ | `Allow` or `Deny` |
| **Action** | ✅ | API actions (`s3:GetObject`, `ec2:*`) |
| **Resource** | ✅ | ARN of target resource(s) |
| **Condition** | ❌ | Optional constraints (IP, MFA, tags, time, etc.) |
| **Principal** | ❌ | WHO (only in resource-based policies) |

---

## IAM Conditions — Deep Dive

### Common Condition Keys

| Condition Key | Use Case |
|---|---|
| `aws:SourceIp` | Restrict by IP range (does NOT work inside VPC — use VPC Endpoint policies) |
| `aws:RequestedRegion` | Restrict API calls to specific regions |
| `aws:PrincipalTag` | Check tags on the calling principal |
| `aws:ResourceTag` | Check tags on the target resource |
| `aws:MultiFactorAuthPresent` | Require MFA for sensitive operations |
| `aws:PrincipalOrgID` | Restrict to members of an AWS Organization |
| `aws:SecureTransport` | Require HTTPS (deny HTTP) |
| `s3:x-amz-server-side-encryption` | Enforce encryption on upload |
| `ec2:ResourceTag/Environment` | Tag-based access control for EC2 |

### Condition Operators

| Operator | Example |
|---|---|
| `StringEquals` | Exact match (case sensitive) |
| `StringLike` | Wildcard match (`*`, `?`) |
| `IpAddress` / `NotIpAddress` | CIDR range matching |
| `DateGreaterThan` | Time-based policies |
| `ArnLike` | ARN pattern matching |
| `Bool` | Boolean check (`aws:SecureTransport`) |
| `Null` | Check if key exists |

> [!TIP]
> **Exam Pattern:** "Deny access unless using MFA" → Condition: `"Bool": {"aws:MultiFactorAuthPresent": "false"}` with Effect: Deny. "Restrict to specific regions" → `aws:RequestedRegion`. "Allow only from corporate IP" → `aws:SourceIp`.

---

## IAM Roles — Advanced Concepts

### Role Trust Policy vs Permissions Policy

```
  ┌──────────────────────────────────────────────────────────────┐
  │                      IAM Role                                │
  │                                                              │
  │  Trust Policy (WHO can assume this role):                    │
  │  ┌─────────────────────────────────────────────────────────┐│
  │  │ {                                                       ││
  │  │   "Principal": {                                        ││
  │  │     "Service": "ec2.amazonaws.com"     ← EC2 instances  ││
  │  │     "AWS": "arn:aws:iam::111:root"     ← cross-account ││
  │  │     "Federated": "cognito-identity..." ← federation    ││
  │  │   }                                                     ││
  │  │ }                                                       ││
  │  └─────────────────────────────────────────────────────────┘│
  │                                                              │
  │  Permissions Policy (WHAT can this role do):                 │
  │  ┌─────────────────────────────────────────────────────────┐│
  │  │ {                                                       ││
  │  │   "Effect": "Allow",                                    ││
  │  │   "Action": "s3:*",                                     ││
  │  │   "Resource": "*"                                       ││
  │  │ }                                                       ││
  │  └─────────────────────────────────────────────────────────┘│
  └──────────────────────────────────────────────────────────────┘
```

### Common Role Use Cases

| Role Type | Trust Principal | Use Case |
|---|---|---|
| **EC2 Instance Role** | `ec2.amazonaws.com` | EC2 accessing S3, DynamoDB (via Instance Profile) |
| **Lambda Execution Role** | `lambda.amazonaws.com` | Lambda accessing AWS resources |
| **Cross-Account Role** | `arn:aws:iam::OTHER_ACCT:root` | Access resources in another account |
| **Service-Linked Role** | Various AWS services | AWS-managed, pre-defined permissions |
| **Federation Role** | `cognito-identity.amazonaws.com` | Federated users accessing AWS |

> [!IMPORTANT]
> **EC2 Instance Profiles** are the wrapper that delivers IAM role credentials to EC2 instances via the **Instance Metadata Service (IMDS)**. Never embed access keys in EC2 — always use instance roles. IMDSv2 (token-based) is recommended over IMDSv1 for security.

---

## AWS Security Token Service (STS)

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                    AWS STS (Global + Regional)                    │
  │                                                                   │
  │  Provides TEMPORARY security credentials:                        │
  │  • Access Key ID                                                  │
  │  • Secret Access Key                                              │
  │  • Session Token                                                  │
  │  • Expiration (configurable: 15 min → 12 hours)                  │
  │                                                                   │
  │  Key APIs:                                                        │
  │  ┌───────────────────────────┬───────────────────────────────┐   │
  │  │ AssumeRole               │ Cross-account, same-account   │   │
  │  │                          │ role assumption                │   │
  │  ├───────────────────────────┼───────────────────────────────┤   │
  │  │ AssumeRoleWithSAML       │ SAML 2.0 federation           │   │
  │  ├───────────────────────────┼───────────────────────────────┤   │
  │  │ AssumeRoleWithWebIdentity│ Web IdP (use Cognito instead) │   │
  │  ├───────────────────────────┼───────────────────────────────┤   │
  │  │ GetSessionToken          │ MFA-protected API access       │   │
  │  ├───────────────────────────┼───────────────────────────────┤   │
  │  │ GetFederationToken       │ Federated user temp creds      │   │
  │  └───────────────────────────┴───────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────┘
```

### Cross-Account Access Flow

```
  Account A (Trusting)                 Account B (Trusted)
  ┌──────────────────────┐            ┌──────────────────────┐
  │                      │            │                      │
  │  S3 Bucket           │            │  IAM User "Alice"    │
  │                      │            │  (has permission to  │
  │  IAM Role:           │  AssumeRole│   sts:AssumeRole)    │
  │  "CrossAccountRole"  │◄───────────│                      │
  │                      │            │                      │
  │  Trust: Account B    │  Returns   │  Receives temp       │
  │  Perms: s3:Get*      │───────────►│  credentials         │
  │                      │  temp creds│                      │
  └──────────────────────┘            │  Accesses S3 in      │
                                      │  Account A           │
                                      └──────────────────────┘
```

> [!TIP]
> **Exam Pattern:** "Allow Account B to access resources in Account A" → Create IAM Role in Account A with trust policy for Account B + permissions for the resource. User in Account B calls `sts:AssumeRole`. This is preferred over sharing long-term access keys.

---

## Permissions Boundaries

```
  ┌────────────────────────────────────────────────────────────────┐
  │  Permissions Boundary = Maximum permissions ceiling             │
  │                                                                 │
  │  Identity Policy (what you request):    Boundary (max allowed):│
  │  ┌─────────────────────────┐   ┌─────────────────────────┐    │
  │  │  s3:*                   │   │  s3:GetObject            │    │
  │  │  ec2:*                  │   │  s3:PutObject            │    │
  │  │  iam:*                  │   │  ec2:Describe*           │    │
  │  └─────────────────────────┘   └─────────────────────────┘    │
  │                                                                 │
  │  Effective = INTERSECTION:                                      │
  │  ┌─────────────────────────┐                                   │
  │  │  s3:GetObject ✅         │                                   │
  │  │  s3:PutObject ✅         │                                   │
  │  │  ec2:Describe* ✅        │                                   │
  │  │  ec2:RunInstances ❌     │  ← not in boundary              │
  │  │  iam:* ❌                │  ← not in boundary              │
  │  └─────────────────────────┘                                   │
  └────────────────────────────────────────────────────────────────┘
```

**Use case:** Allow a team lead to create IAM users/roles but only with a specific permissions boundary attached — prevents privilege escalation.

---

## Identity Federation

### Federation Options Comparison

| Method | Protocol | Use Case |
|---|---|---|
| **[[Amazon Cognito]] Identity Pools** | OIDC/SAML/Social | Mobile/web apps accessing AWS (recommended) |
| **IAM SAML 2.0 Federation** | SAML 2.0 | Enterprise SSO (Active Directory → AWS Console/API) |
| **IAM Web Identity Federation** | OIDC | Deprecated — use Cognito instead |
| **AWS IAM Identity Center (SSO)** | SAML 2.0 / SCIM | Multi-account SSO for Organizations (recommended) |
| **Custom Identity Broker** | STS API | Legacy systems without SAML support |

### SAML 2.0 Federation Flow

```
  ┌──────┐  1. Auth   ┌──────────┐  2. SAML     ┌──────┐  3. AssumeRole
  │ User │──────────►│ Corp IdP  │  Assertion   │ AWS  │  WithSAML
  │      │           │ (AD FS)   │─────────────►│ STS  │──────────┐
  │      │◄──────────│           │              │      │          │
  │      │  SAML     └──────────┘              └──────┘          │
  │      │  assertion                                             │
  │      │                                                        │
  │      │◄───────────────────────────────────────────────────────┘
  │      │  4. Temp AWS credentials
  │      │
  │      │  5. Access AWS Console or API
  └──────┘
```

---

## AWS IAM Identity Center (successor to AWS SSO)

```
  ┌──────────────────────────────────────────────────────────────┐
  │              IAM Identity Center                              │
  │                                                               │
  │  ┌──────────────┐     ┌──────────────────────────┐           │
  │  │ Identity     │     │ Permission Sets          │           │
  │  │ Sources:     │     │ (collections of policies │           │
  │  │ • Built-in   │     │  assigned to users/groups │           │
  │  │ • Active Dir │     │  per account)             │           │
  │  │ • External   │     └──────────────────────────┘           │
  │  │   SAML IdP   │                                             │
  │  └──────────────┘     ┌──────────────────────────┐           │
  │                       │ Multi-Account Access:     │           │
  │  Single sign-on ────► │ Account A → Admin         │           │
  │  portal for ALL       │ Account B → ReadOnly      │           │
  │  AWS accounts         │ Account C → Developer     │           │
  │                       └──────────────────────────┘           │
  │                                                               │
  │  ✅ Recommended for Organizations multi-account SSO          │
  │  ✅ Supports ABAC (Attribute-Based Access Control)           │
  │  ✅ Integration with AD, Okta, Azure AD, etc.               │
  └──────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Exam Pattern:** "Single sign-on across multiple AWS accounts in an Organization" → **IAM Identity Center**. "Enterprise employees access AWS Console with corporate credentials" → **IAM Identity Center** (if multi-account) or **SAML 2.0 Federation** (single account).

---

## Resource-Based Policies vs IAM Roles (Cross-Account)

| Feature | Resource-Based Policy | IAM Role (AssumeRole) |
|---|---|---|
| **How it works** | Policy on resource grants access to external principal | Caller assumes role, gets temp credentials |
| **Original permissions** | Caller **keeps** their own permissions | Caller **gives up** original permissions |
| **Supported by** | S3, SQS, SNS, KMS, Lambda, API Gateway, etc. | Any service |
| **Use when** | Need to retain caller's own permissions | Universal cross-account pattern |

> [!CAUTION]
> **Key difference:** When you assume a role, you **give up your original permissions** and take only the role's permissions. With resource-based policies, you **keep your own permissions** plus gain access. This matters when a user in Account A needs to both scan DynamoDB in Account A AND write to S3 in Account B in the same operation.

---

## Tag-Based Access Control (ABAC)

```
  ABAC = Attribute-Based Access Control (using Tags)

  Example: Allow users to manage only EC2 instances with matching Department tag

  {
    "Effect": "Allow",
    "Action": "ec2:*",
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "aws:ResourceTag/Department": "${aws:PrincipalTag/Department}"
      }
    }
  }

  User Tag: Department=Engineering
       │
       ├──► ec2 (Department=Engineering) → ✅ ALLOWED
       └──► ec2 (Department=Finance)     → ❌ DENIED
```

**Advantages over RBAC:** Scales without creating new policies per resource. Add a tag → access is automatic.

---

## IAM Access Analyzer

| Feature | Description |
|---|---|
| **Purpose** | Identify resources shared with external entities |
| **Zone of Trust** | Your AWS account or Organization |
| **Findings** | Resources with policies granting access outside the zone of trust |
| **Supports** | S3, IAM Roles, KMS, Lambda, SQS, Secrets Manager |
| **Policy Validation** | Validates policies against best practices and grammar |
| **Policy Generation** | Generates least-privilege policies from [[AWS CloudTrail]] access logs |

> [!TIP]
> **Exam Pattern:** "Generate least-privilege IAM policies based on actual usage" → **IAM Access Analyzer policy generation** (uses CloudTrail logs). "Find resources shared externally" → **IAM Access Analyzer findings**.

---

## IAM Best Practices & Security Tools

### Best Practices

| Practice | Why |
|---|---|
| **Use roles, not long-term keys** | Temp credentials rotate automatically |
| **Enforce MFA** | Especially for privileged operations and console access |
| **Least privilege** | Start with zero permissions, grant only what's needed |
| **Use groups** | Attach policies to groups, not individual users |
| **Rotate credentials** | Use Credential Report to audit |
| **Use IAM Identity Center** | For workforce access to multiple accounts |
| **Never use root** | Lock away root access keys, enable MFA on root |
| **Use Permissions Boundaries** | For delegated admin scenarios |

### IAM Security Tools

| Tool | Description |
|---|---|
| **IAM Credentials Report** | Account-level CSV: all users, passwords, access keys, MFA status |
| **IAM Access Advisor** | User-level: shows service permissions granted vs last accessed |
| **IAM Access Analyzer** | Identifies resources shared externally, generates policies |
| **IAM Policy Simulator** | Test and debug IAM policies without making real API calls |

---

## Key IAM Limits

| Parameter | Value |
|---|---|
| **IAM Users per account** | 5,000 |
| **IAM Groups per account** | 300 |
| **Groups per user** | 10 |
| **Managed policies per user/role** | 10 |
| **Inline policy size** | 2,048 characters (user), 10,240 (role/group) |
| **Managed policy size** | 6,144 characters |
| **Roles per account** | 1,000 (can increase) |
| **Instance profiles per account** | 1,000 |
| **SAML providers per account** | 100 |
| **STS session duration** | 15 min → 12 hours (default 1 hour) |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → IAM:**
> - "Least privilege policy based on actual usage" → **IAM Access Analyzer** (CloudTrail-based policy generation)
> - "EC2 needs to access S3" → **IAM Role + Instance Profile** (never access keys)
> - "Cross-account access" → **IAM Role with trust policy** or **resource-based policy**
> - "Restrict max permissions for delegated admins" → **Permissions Boundaries**
> - "Enterprise SSO across multiple accounts" → **IAM Identity Center**
> - "Corporate AD users access AWS Console" → **SAML 2.0 Federation** or **IAM Identity Center**
> - "Temporary credentials" → **STS AssumeRole**
> - "Force MFA for API calls" → **IAM Condition: aws:MultiFactorAuthPresent**
> - "Restrict to specific regions" → **IAM Condition: aws:RequestedRegion**
> - "Find externally shared resources" → **IAM Access Analyzer**
> - "Audit all users' credential status" → **IAM Credentials Report**
> - "Check which services a user actually uses" → **IAM Access Advisor**
> - "Limit to specific AWS Organization" → **Condition: aws:PrincipalOrgID**
> - "Prevent privilege escalation" → **Permissions Boundaries** + SCPs
>
> **Key facts:**
> - IAM is global. Policies are JSON. Explicit Deny always wins.
> - Same-account: identity OR resource policy Allow = access. Cross-account: BOTH must Allow.
> - AssumeRole = give up original permissions. Resource-based policy = keep original permissions.
> - 5,000 users max → use federation/Identity Center for large workforces.
> - IMDSv2 (token-based) recommended over IMDSv1 for instance metadata security.
> - SCPs don't affect the management account and don't grant permissions.
> - Service-linked roles are managed by AWS and cannot be modified.
> - Access Analyzer zone of trust = account or Organization.
