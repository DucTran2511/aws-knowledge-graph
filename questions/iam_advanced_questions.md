---
tags: [questions, security, iam, identity, authorization, exam-prep]
date: 2026-06-08
---

# IAM Advanced — Practice Questions (SAA-C03)

---

### Q1. Cross-Account S3 Access

A company has two AWS accounts: Account A (production) contains an S3 bucket with critical data, and Account B (analytics) has data scientists who need read access. The data scientists must also retain access to DynamoDB tables in Account B during the same operations. What is the BEST approach?

**A)** Create IAM users in Account A for each data scientist
**B)** Share long-term access keys from Account A with Account B users
**C)** Add a resource-based policy (bucket policy) on the S3 bucket in Account A granting access to Account B principals
**D)** Create an IAM role in Account A and have Account B users assume it

<details><summary>Answer</summary>

**C)** Resource-based policy on the S3 bucket.

**Why:** When using resource-based policies, the caller **retains their original permissions** (DynamoDB in Account B) while gaining access to the resource in Account A. With AssumeRole (D), users would **give up** their Account B permissions. (A) doesn't scale. (B) is a security anti-pattern.

</details>

---

### Q2. Permissions Boundary Delegation

A company wants to allow team leads to create IAM roles for their developers, but wants to ensure those roles can never have more than S3 and DynamoDB permissions. What should the cloud architect implement?

**A)** Service Control Policies (SCPs) on the team lead's OU
**B)** Permissions boundaries that restrict to S3 and DynamoDB, and require team leads to attach the boundary when creating roles
**C)** Inline deny policies on all developer roles
**D)** Use IAM Access Analyzer to monitor role creation

<details><summary>Answer</summary>

**B)** Permissions boundaries.

**Why:** Permissions boundaries set the **maximum permissions** a role can have. By requiring team leads to attach a specific boundary when creating roles, you enable delegation while preventing privilege escalation. SCPs (A) apply at the account/OU level, not per-role. (C) is manual and error-prone. (D) is detective, not preventive.

</details>

---

### Q3. MFA-Protected API Calls

A security team requires that all IAM users must use MFA before performing any EC2 termination actions. Which IAM policy approach achieves this?

**A)** Attach a policy with `"Effect": "Allow"` on `ec2:TerminateInstances` with condition `aws:MultiFactorAuthPresent: true`
**B)** Attach a policy with `"Effect": "Deny"` on `ec2:TerminateInstances` with condition `"Bool": {"aws:MultiFactorAuthPresent": "false"}`
**C)** Enable MFA on the root account only
**D)** Use an SCP to require MFA for all API calls

<details><summary>Answer</summary>

**B)** Deny when MFA is NOT present.

**Why:** The deny-when-false pattern is more secure because explicit Deny always wins. Option A only allows with MFA but doesn't deny without it — other policies could still allow the action. Option B ensures TerminateInstances is always denied unless MFA is active. (C) only protects root. (D) is too broad.

</details>

---

### Q4. EC2 Instance Accessing DynamoDB

A developer deployed an application on an EC2 instance that needs to read from a DynamoDB table. The developer stored AWS access keys in the application's config file. What should the solutions architect recommend?

**A)** Encrypt the config file using KMS
**B)** Store the access keys in AWS Secrets Manager
**C)** Remove the access keys and attach an IAM role with DynamoDB read permissions to the EC2 instance
**D)** Use environment variables instead of a config file

<details><summary>Answer</summary>

**C)** Use an IAM role with an instance profile.

**Why:** IAM roles provide **temporary credentials** that rotate automatically via the Instance Metadata Service. Embedding long-term access keys (even encrypted or in Secrets Manager) is an anti-pattern for EC2. The role should follow least privilege — only `dynamodb:GetItem`, `dynamodb:Query` etc.

</details>

---

### Q5. Enterprise SSO Multi-Account

A large enterprise with 50 AWS accounts in an Organization wants employees to sign in once with their corporate Active Directory credentials and access the appropriate AWS accounts based on their job function. What is the recommended solution?

**A)** Create IAM users in each account mapped to AD users
**B)** Configure SAML 2.0 federation in each account
**C)** Use AWS IAM Identity Center with Active Directory as the identity source
**D)** Use Amazon Cognito User Pools for enterprise SSO

<details><summary>Answer</summary>

**C)** IAM Identity Center.

**Why:** IAM Identity Center (successor to AWS SSO) is purpose-built for **multi-account SSO** within AWS Organizations. It supports AD integration, permission sets per account, and a single sign-on portal. SAML federation (B) works but requires per-account setup. Cognito (D) is for application users, not workforce. (A) doesn't scale.

</details>

---

### Q6. IAM Access Analyzer — Least Privilege

A security engineer wants to generate IAM policies that grant only the permissions an application actually uses, based on historical API activity. Which service should they use?

**A)** IAM Policy Simulator
**B)** AWS CloudTrail Lake with custom SQL queries
**C)** IAM Access Analyzer policy generation
**D)** IAM Access Advisor

<details><summary>Answer</summary>

**C)** IAM Access Analyzer policy generation.

**Why:** IAM Access Analyzer can analyze [[AWS CloudTrail]] logs and automatically **generate least-privilege policies** based on actual API usage. Access Advisor (D) shows which services were accessed but doesn't generate policies. Policy Simulator (A) tests existing policies. CloudTrail Lake (B) provides raw event data but doesn't generate policies.

</details>

---

### Q7. Restricting API Calls to Specific Regions

A compliance requirement mandates that all AWS resources must be created only in `eu-west-1` and `eu-central-1`. The company uses AWS Organizations. What is the MOST effective solution?

**A)** IAM policy on each user with `aws:RequestedRegion` condition
**B)** SCP on the OU with a Deny for all actions NOT in the approved regions using `aws:RequestedRegion`
**C)** AWS Config rule to detect non-compliant resources
**D)** VPC configuration to restrict region access

<details><summary>Answer</summary>

**B)** SCP with region restriction.

**Why:** SCPs are the **preventive guardrail** at the Organization level. A Deny SCP for actions outside approved regions using `aws:RequestedRegion` condition prevents ANY user (including admins) in member accounts from creating resources in unauthorized regions. (A) requires per-user management. (C) is detective, not preventive. (D) is unrelated.

</details>

---

### Q8. Tag-Based Access Control (ABAC)

A company with multiple development teams shares a single AWS account. Each team's EC2 instances are tagged with `Team=<team-name>`. The architect needs to ensure each team can only manage their own instances without creating separate policies per team. What approach should they use?

**A)** Create separate IAM groups with hardcoded resource ARNs
**B)** Use ABAC with `aws:ResourceTag/Team` matching `aws:PrincipalTag/Team` in IAM policies
**C)** Create separate VPCs per team
**D)** Use AWS Organizations with separate accounts per team

<details><summary>Answer</summary>

**B)** ABAC with tag matching.

**Why:** Attribute-Based Access Control using `aws:ResourceTag/Team` equals `${aws:PrincipalTag/Team}` scales automatically. When a new team is added, just tag users and resources — no new policies needed. (A) requires constant policy updates. (C) and (D) are over-engineered for this scenario.

</details>

---

### Q9. STS Session Duration

A partner company needs to assume a cross-account IAM role to run batch processing jobs that take approximately 6 hours. They report that their credentials expire after 1 hour. How should the solutions architect resolve this?

**A)** Create an IAM user with long-term access keys instead
**B)** Increase the role's maximum session duration to 6+ hours and pass `--duration-seconds` in the AssumeRole call
**C)** Have the application re-authenticate every hour
**D)** Use AWS IAM Identity Center instead

<details><summary>Answer</summary>

**B)** Increase maximum session duration.

**Why:** IAM roles have a default session duration of 1 hour but can be configured up to **12 hours**. The caller must also request the longer duration via `--duration-seconds` in the `sts:AssumeRole` call. (A) is less secure than temp credentials. (C) adds unnecessary complexity. (D) is for workforce SSO, not cross-account batch processing.

</details>

---

### Q10. Finding Externally Shared Resources

A security auditor needs to identify all S3 buckets, IAM roles, and KMS keys in an AWS account that are shared with external AWS accounts or the public. What is the MOST efficient approach?

**A)** Manually review all resource policies
**B)** Use AWS Config rules to check each resource type
**C)** Enable IAM Access Analyzer with the account as the zone of trust
**D)** Run AWS Trusted Advisor security checks

<details><summary>Answer</summary>

**C)** IAM Access Analyzer.

**Why:** IAM Access Analyzer automatically identifies resources with policies that grant access to **external principals** outside the defined zone of trust (account or Organization). It supports S3, IAM roles, KMS, Lambda, SQS, and Secrets Manager. (A) doesn't scale. (B) requires separate rules per resource type. (D) has limited coverage.

</details>

---

### Q11. Resource-Based vs Identity-Based (Cross-Account)

An application in Account A needs to invoke a Lambda function in Account B. The Lambda function's resource-based policy allows Account A's role. However, the IAM role in Account A does not have `lambda:InvokeFunction` permission. Will the invocation succeed?

**A)** Yes — the resource-based policy on the Lambda function is sufficient for cross-account access
**B)** No — for cross-account access, both the resource-based policy AND the identity-based policy must Allow
**C)** Yes — resource-based policies always override identity-based policies
**D)** No — Lambda does not support resource-based policies

<details><summary>Answer</summary>

**A)** Yes — for cross-account access with resource-based policies, if the resource-based policy explicitly allows the external principal, the caller does NOT need an identity-based policy.

**Why:** This is a nuance. When a resource-based policy in Account B specifies the **exact ARN** of the principal in Account A (not just the account), the identity-based policy in Account A is not required. The resource-based policy alone is sufficient. This differs from the "both must allow" rule which applies when only the account is specified.

> **Note:** This is a subtle edge case. The general SAA-C03 guidance is that cross-account requires both sides to allow, but resource-based policies that specify the exact principal ARN can grant access independently.

</details>

---

### Q12. IMDSv1 vs IMDSv2

A security assessment flagged that EC2 instances are using Instance Metadata Service version 1 (IMDSv1), which is vulnerable to SSRF attacks. What should the solutions architect do?

**A)** Disable the Instance Metadata Service entirely
**B)** Configure instances to require IMDSv2 (HttpTokens = required) which uses session-oriented token-based requests
**C)** Move to IAM users with access keys instead of instance roles
**D)** Use a VPN to access the metadata endpoint

<details><summary>Answer</summary>

**B)** Require IMDSv2.

**Why:** IMDSv2 requires a **PUT request to get a session token** before accessing metadata, which mitigates SSRF attacks (the token requires specific HTTP headers that SSRF exploits typically can't set). Set `HttpTokens=required` on instances to enforce IMDSv2. (A) would break applications needing instance role credentials. (C) is less secure. (D) is irrelevant.

</details>

---

### Q13. SCP vs IAM Policy

A company uses AWS Organizations. The management account admin wants to prevent anyone in a member account from deleting CloudTrail trails. The member account has an IAM admin user with `AdministratorAccess`. What should they do?

**A)** Remove `AdministratorAccess` from the member account admin
**B)** Apply an SCP to the OU that denies `cloudtrail:DeleteTrail` and `cloudtrail:StopLogging`
**C)** Use an IAM permissions boundary on the admin user
**D)** Enable MFA delete on CloudTrail

<details><summary>Answer</summary>

**B)** SCP denying CloudTrail deletion.

**Why:** SCPs are **guardrails that override IAM permissions** in member accounts. Even `AdministratorAccess` cannot override an SCP Deny. SCPs don't affect the management account. (A) removes too much access. (C) requires per-user setup. (D) doesn't exist for CloudTrail (it's an S3 feature).

</details>

---

### Q14. Credential Report vs Access Advisor

A compliance audit requires the security team to identify: (1) all IAM users who haven't rotated their access keys in 90+ days, and (2) which services each user has accessed in the last year. Which tools should they use?

**A)** IAM Access Analyzer for both requirements
**B)** IAM Credentials Report for (1) and IAM Access Advisor for (2)
**C)** AWS CloudTrail for both requirements
**D)** IAM Policy Simulator for (1) and Access Analyzer for (2)

<details><summary>Answer</summary>

**B)** Credentials Report + Access Advisor.

**Why:** **IAM Credentials Report** is an account-level CSV showing all users, password age, access key age, MFA status — perfect for identifying stale keys. **IAM Access Advisor** shows per-user which services were last accessed — perfect for identifying unused permissions. CloudTrail (C) has raw API logs but isn't designed for this reporting.

</details>

---

### Q15. Inline vs Managed Policies

A solutions architect is designing IAM policies for a team of 20 developers. All developers need identical permissions. Which policy approach is recommended?

**A)** Create inline policies on each developer's IAM user
**B)** Create a customer managed policy and attach it to an IAM group containing all developers
**C)** Use AWS managed policies directly on each user
**D)** Create 20 separate customer managed policies, one per user

<details><summary>Answer</summary>

**B)** Customer managed policy on an IAM group.

**Why:** Customer managed policies are **reusable, versionable, and centrally managed**. Attaching to a group means changes apply to all members instantly. Inline policies (A) require updating 20 users individually. AWS managed policies (C) may not match exact requirements. (D) creates unnecessary duplication.

</details>

---

### Q16. aws:PrincipalOrgID Condition

A company shares an S3 bucket with multiple AWS accounts in their Organization. They want to ensure ONLY accounts within their Organization can access the bucket, even if new accounts are added later. What's the MOST scalable approach?

**A)** List every account ID in the bucket policy Principal field
**B)** Use `aws:PrincipalOrgID` condition in the bucket policy matching their Organization ID
**C)** Create cross-account roles for each member account
**D)** Use VPC endpoints to restrict access

<details><summary>Answer</summary>

**B)** `aws:PrincipalOrgID` condition.

**Why:** The `aws:PrincipalOrgID` condition key automatically matches **all current and future** accounts in the Organization — no policy updates needed when accounts are added or removed. (A) requires manual updates. (C) doesn't scale. (D) restricts by network, not identity.

</details>

---

### Q17. Service-Linked Roles

A developer is trying to modify the permissions of a role named `AWSServiceRoleForElasticLoadBalancing` but receives an access denied error, even though they have `iam:*` permissions. Why?

**A)** The role requires MFA to modify
**B)** The role is a service-linked role managed by AWS and cannot be modified by customers
**C)** The developer needs root account access
**D)** There is an SCP blocking the modification

<details><summary>Answer</summary>

**B)** Service-linked roles are managed by AWS.

**Why:** **Service-linked roles** are pre-defined by AWS services with exact permissions the service needs. They are created and managed by the service and **cannot be modified or deleted** by users (unless the service is no longer in use). Their trust policy only allows the specific AWS service to assume them.

</details>

---

### Q18. Session Policies

A company's admin assumes a cross-account role with broad S3 permissions. For a specific task, they want to further restrict the session to only `s3:GetObject` on a single bucket. What mechanism should they use?

**A)** Modify the role's permissions policy temporarily
**B)** Pass a session policy during the `AssumeRole` call that restricts to `s3:GetObject` on the specific bucket
**C)** Use a permissions boundary
**D)** Create a separate role with limited permissions

<details><summary>Answer</summary>

**B)** Session policy.

**Why:** Session policies are optional policies passed during `sts:AssumeRole` that **further restrict** the effective permissions for that specific session. The effective permissions are the intersection of the role's policy and the session policy. (A) affects all users of the role. (C) boundaries are set at entity creation, not per-session. (D) creates unnecessary role proliferation.

</details>

---

### Q19. aws:SourceIp Limitation

A security policy requires that IAM users can only make API calls from the corporate network (IP range 10.0.0.0/8). The policy works for CLI access from corporate laptops, but fails for an EC2-based application using the same IAM user's credentials from within a VPC. Why?

**A)** EC2 instances cannot use IAM user credentials
**B)** `aws:SourceIp` checks the public IP; EC2 instances in a VPC use private IPs that don't match
**C)** VPC doesn't support IAM conditions
**D)** The security group is blocking API calls

<details><summary>Answer</summary>

**B)** `aws:SourceIp` uses the request's source IP.

**Why:** For API calls from within AWS (EC2, Lambda, etc.), the `aws:SourceIp` condition sees the **instance's public/NAT Gateway IP**, not the VPC private IP. If the corporate range is a private range (10.0.0.0/8), it won't match. For VPC-originated calls, use **VPC Endpoint policies** or `aws:SourceVpc`/`aws:SourceVpce` conditions instead.

</details>

---

### Q20. Confused Deputy Problem

A third-party SaaS provider needs access to your S3 bucket via a cross-account IAM role. How do you prevent the "confused deputy" problem where other customers of the SaaS provider could use the same role to access your bucket?

**A)** Use an IP-based condition to restrict the SaaS provider's source IP
**B)** Add an `aws:ExternalId` condition to the role's trust policy, using a unique ID shared only with the SaaS provider
**C)** Encrypt the S3 bucket with a customer-managed KMS key
**D)** Use a VPC endpoint to restrict access

<details><summary>Answer</summary>

**B)** External ID in the trust policy.

**Why:** The **External ID** is a unique secret between you and the third-party. It's included in the trust policy's `sts:ExternalId` condition. The SaaS provider must pass this ID when calling AssumeRole. Other customers of the same SaaS provider have different External IDs, so they can't use the role to access your resources. This is the standard solution for the confused deputy problem.

</details>
