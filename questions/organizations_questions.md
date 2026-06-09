---
tags: [questions, security, organizations, scp, multi-account, governance, exam-prep]
date: 2026-06-09
---

# AWS Organizations — Practice Questions (SAA-C03)

---

### Q1. SCP Region Restriction

A company with AWS Organizations requires all member accounts to create resources only in `us-east-1` and `eu-west-1`. An SCP is applied with a Deny for all actions outside these regions, but IAM operations start failing globally. What is the issue?

**A)** SCPs cannot restrict regions
**B)** The SCP should use `NotAction` to exclude global services like IAM, STS, and Organizations
**C)** The management account needs a separate region policy
**D)** SCPs only work with allow-list strategies

<details><summary>Answer</summary>

**B)** Use `NotAction` to exclude global services.

**Why:** Global services (IAM, STS, Organizations, Support, CloudFront) operate in `us-east-1` regardless of region. A blanket Deny on all actions outside approved regions will break these global services. Using `NotAction` with these services ensures they remain functional while restricting regional services.

</details>

---

### Q2. SCP vs Management Account

A security admin applies an SCP to the Root of the Organization that denies `ec2:TerminateInstances`. An IAM admin in the management account tries to terminate an EC2 instance. What happens?

**A)** The termination is denied — SCPs apply to all accounts
**B)** The termination succeeds — SCPs do NOT affect the management account
**C)** The termination is denied only if the admin lacks IAM permissions
**D)** The SCP must be applied directly to the management account to take effect

<details><summary>Answer</summary>

**B)** SCPs never affect the management account.

**Why:** The management account is always exempt from SCPs, even those attached at the Root. This is why AWS recommends using the management account only for Organization management — not running workloads — since it cannot be restricted by SCPs.

</details>

---

### Q3. Consolidated Billing RI Sharing

Company A has 3 AWS accounts in an Organization. Account 1 purchased 5 `m5.large` Reserved Instances but only uses 3. Account 2 runs 4 `m5.large` On-Demand instances. How does billing work?

**A)** Account 2 pays full On-Demand for all 4 instances
**B)** Account 2 benefits from Account 1's unused RIs — 2 instances at RI rate, 2 at On-Demand
**C)** RI sharing must be manually configured per account
**D)** RIs cannot be shared across accounts

<details><summary>Answer</summary>

**B)** Account 2 gets 2 instances at the RI rate.

**Why:** RI sharing is **enabled by default** in Organizations with Consolidated Billing. Account 1's 2 unused `m5.large` RIs automatically apply to Account 2's matching instances. The remaining 2 instances in Account 2 are billed at On-Demand rates. RI sharing can be disabled per account if needed.

</details>

---

### Q4. Control Tower vs Manual Setup

A startup is creating its first multi-account AWS environment. They need centralized logging, SSO for employees, preventive and detective guardrails, and automated account provisioning. The team has limited AWS experience. What should they use?

**A)** Manually configure Organizations, SCPs, CloudTrail, Config, and IAM Identity Center
**B)** Use AWS Control Tower to set up a Landing Zone with pre-configured guardrails
**C)** Use a single account with IAM groups for isolation
**D)** Deploy a third-party governance tool

<details><summary>Answer</summary>

**B)** AWS Control Tower.

**Why:** Control Tower automates the setup of a governed multi-account environment (Landing Zone) with pre-configured Log Archive and Audit accounts, guardrails (SCPs + Config Rules), Account Factory for provisioning, and IAM Identity Center for SSO. It's the "easy button" — perfect for teams with limited experience. Manual setup (A) achieves the same but requires significant expertise.

</details>

---

### Q5. Preventing CloudTrail Tampering

A CISO wants to ensure that no administrator in ANY member account can disable CloudTrail logging or delete trails. What is the MOST effective approach?

**A)** Remove IAM permissions for CloudTrail from all member accounts
**B)** Apply an SCP denying `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail`, and `cloudtrail:UpdateTrail`
**C)** Use [[AWS Config]] rules to detect disabled trails
**D)** Enable MFA on all CloudTrail configurations

<details><summary>Answer</summary>

**B)** SCP denying CloudTrail modification.

**Why:** SCPs override all IAM permissions in member accounts, including root user access. Even an `AdministratorAccess` user cannot override an SCP Deny. This is a **preventive** control. Config rules (C) are detective — they detect after the fact but don't prevent. (A) is impractical to maintain across all users/roles.

</details>

---

### Q6. AWS RAM VPC Sharing

A company wants multiple AWS accounts to launch resources into the same VPC subnets without setting up VPC peering or Transit Gateway. How should they architect this?

**A)** Create the VPC in one account and share subnets via AWS RAM across the Organization
**B)** Create identical VPCs in each account and peer them
**C)** Use a VPN connection between accounts
**D)** Deploy all workloads in a single account

<details><summary>Answer</summary>

**A)** Share VPC subnets via AWS RAM.

**Why:** AWS Resource Access Manager (RAM) can share VPC subnets across Organization accounts. Each account launches its own resources (EC2, RDS, Lambda) into the shared subnet, managing them independently. No VPC peering is required. The networking account owns the VPC; workload accounts use the subnets. This simplifies network management significantly.

</details>

---

### Q7. SCP Allow-List Strategy

A highly regulated account should ONLY be allowed to use S3, EC2, and RDS — no other services. How should the SCP be configured?

**A)** Attach Deny policies for every other AWS service
**B)** Remove the default `FullAWSAccess` SCP, then attach a custom SCP that Allows only `s3:*`, `ec2:*`, `rds:*` (plus global services)
**C)** Use IAM policies to restrict services instead
**D)** Use AWS Config to detect unauthorized service usage

<details><summary>Answer</summary>

**B)** Allow-list strategy: remove `FullAWSAccess` and allow only approved services.

**Why:** The allow-list approach removes the default `FullAWSAccess` SCP and replaces it with explicit Allows for only the approved services. This blocks ALL unapproved services by default. Remember to include global services (IAM, STS, Organizations). (A) is unmanageable — you'd need to list hundreds of services. (C) requires per-user management and doesn't cover the root user.

</details>

---

### Q8. Tag Policies Enforcement

A finance team requires all EC2 instances across the Organization to have a `CostCenter` tag with values from an approved list. They implement Tag Policies. A developer launches an EC2 instance without the tag. What happens?

**A)** The instance launch is blocked
**B)** The instance launches successfully but is flagged as non-compliant
**C)** The instance is automatically terminated
**D)** The developer receives an error and must add the tag

<details><summary>Answer</summary>

**B)** Launches successfully but flagged as non-compliant.

**Why:** Tag Policies are **detective**, not preventive. They flag non-compliant resources but do NOT block resource creation. To prevent launches without required tags, combine Tag Policies with an SCP that uses a `Condition` to deny `ec2:RunInstances` when the `CostCenter` tag is missing.

</details>

---

### Q9. Delegated Administrator

The security team wants to manage GuardDuty across all Organization accounts, but they don't want to use the management account for daily operations. What's the recommended approach?

**A)** Give the security team IAM users in the management account
**B)** Designate the Security Tooling account as a delegated administrator for GuardDuty
**C)** Enable GuardDuty independently in each account
**D)** Use cross-account IAM roles from the management account

<details><summary>Answer</summary>

**B)** Delegated administrator.

**Why:** Organizations supports **delegated administrators** — member accounts designated to manage specific AWS services on behalf of the Organization. This avoids using the management account for daily operations (best practice: keep management account minimal). The Security Tooling account can manage GuardDuty, Security Hub, etc., for all accounts.

</details>

---

### Q10. Account Leaving Organization

A member account needs to leave the Organization to become standalone. What must be true before the account can leave?

**A)** The management account must approve the departure
**B)** The member account must have its own payment method (credit card or billing agreement) configured
**C)** All resources in the account must be deleted first
**D)** SCPs must be removed from the account first

<details><summary>Answer</summary>

**B)** The account must have its own payment method.

**Why:** When an account leaves an Organization, it becomes a standalone account and must pay its own bills. It needs a valid payment method before it can leave. Additionally, if an SCP denies `organizations:LeaveOrganization`, the account cannot leave — this SCP must be removed first or applied differently. Resources do not need to be deleted.

</details>

---

### Q11. SCP + IAM Interaction

An OU has an SCP that allows `s3:*` and `ec2:*` only (allow-list strategy). A user in that OU's account has an IAM policy granting `rds:*`. Can the user create an RDS database?

**A)** Yes — IAM policies override SCPs
**B)** No — the SCP does not allow `rds:*`, so it's blocked regardless of IAM permissions
**C)** Yes — because there's no explicit Deny on RDS
**D)** It depends on whether a permissions boundary is attached

<details><summary>Answer</summary>

**B)** No — SCP blocks it.

**Why:** With an allow-list SCP strategy, only explicitly allowed actions pass the SCP filter. Since `rds:*` is not in the SCP's Allow list, it's implicitly denied at the Organization level. The user's IAM `rds:*` policy is irrelevant — SCPs set the **maximum ceiling**. Both SCP AND IAM must Allow for access to be granted.

</details>

---

### Q12. Organization Trail

A security team needs to centralize API audit logs from all 50 accounts in their Organization into a single location for analysis. What's the MOST operationally efficient solution?

**A)** Create individual CloudTrail trails in each account pointing to a shared S3 bucket
**B)** Create an Organization Trail in the management account
**C)** Use CloudWatch Logs aggregation across all accounts
**D)** Export CloudTrail Event History from each account manually

<details><summary>Answer</summary>

**B)** Organization Trail.

**Why:** An [[AWS CloudTrail]] **Organization Trail** is created once in the management account and automatically logs API events from ALL member accounts across ALL regions to a single S3 bucket. Member accounts can see the trail but cannot modify or delete it. This is far more efficient than managing 50 individual trails (A).

</details>

---

### Q13. Preventing Public S3 Buckets Across Organization

A company needs to ensure NO member account can create public S3 buckets. Which TWO controls should they implement together?

**A)** SCP denying `s3:PutBucketPublicAccessBlock` with "BlockPublicAcls=false" + AWS Config rule for detection
**B)** SCP denying `s3:PutBucketPolicy` for any policy with Principal "*" + S3 Block Public Access at the account level
**C)** Only IAM policies restricting S3 public access
**D)** Tag policies requiring a "Private" tag on all S3 buckets

<details><summary>Answer</summary>

**B)** SCP + S3 Block Public Access.

**Why:** A layered approach: (1) SCP prevents disabling S3 Block Public Access settings at the account level, and (2) Enable S3 Block Public Access at the account level in each account. The SCP ensures administrators can't weaken the setting. This is **preventive**. Add [[AWS Config]] rules as a **detective** layer for additional assurance.

</details>

---

### Q14. Moving Accounts Between OUs

A development account needs to be promoted to production. It's currently in `OU:Development` with lenient SCPs. What happens when it's moved to `OU:Production` with strict SCPs?

**A)** The account retains Development SCPs and gains Production SCPs
**B)** The account immediately inherits Production SCPs and loses Development SCPs
**C)** The move requires account recreation
**D)** SCPs take 24 hours to propagate after the move

<details><summary>Answer</summary>

**B)** Immediately inherits new OU's SCPs.

**Why:** When an account moves between OUs, it **immediately** inherits the SCPs of its new OU (and ancestors) and loses the SCPs of its old OU. There's no propagation delay. This means workloads in the account must already comply with the Production SCPs or they may be disrupted.

</details>

---

### Q15. Service-Linked Roles and SCPs

An SCP is applied that denies all EC2 actions. However, Auto Scaling (which uses a service-linked role) continues to launch EC2 instances. Why?

**A)** Auto Scaling ignores SCPs
**B)** Service-linked roles are NOT affected by SCPs
**C)** The SCP has a bug
**D)** Auto Scaling uses the management account's permissions

<details><summary>Answer</summary>

**B)** Service-linked roles are not affected by SCPs.

**Why:** SCPs do NOT restrict **service-linked roles**. These roles are used by AWS services to perform actions on your behalf, and blocking them could break core service functionality. This is an important exception to remember: SCPs affect all IAM users and roles EXCEPT the management account and service-linked roles.

</details>
