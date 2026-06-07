---
tags: [concept, serverless, identity, authentication, authorization, cognito, federation]
aliases: [Cognito, Amazon Cognito, Cognito User Pools, Cognito Identity Pools, CUP, CIP, User Pools, Identity Pools]
date: 2026-05-29
---

# Amazon Cognito

**Amazon Cognito** provides **authentication**, **authorization**, and **user management** for web and mobile applications. It has two main components: **User Pools** (authentication — who are you?) and **Identity Pools** (authorization — what AWS resources can you access?).

> [!IMPORTANT]
> **Core exam concept:** Cognito User Pools = **authentication** (sign-up, sign-in → JWT tokens). Cognito Identity Pools = **authorization** (exchange tokens → temporary AWS credentials). They can be used independently or together.

---

## Cognito User Pools (CUP) — Authentication

```
  ┌──────────┐    sign-up/     ┌────────────────────────────────────────┐
  │   User   │    sign-in      │         Cognito User Pool              │
  │  (web /  │────────────────►│                                        │
  │  mobile) │                 │  ┌──────────────────────────────────┐  │
  └──────────┘                 │  │  User Directory                  │  │
                               │  │  • Username/email/phone          │  │
  Also supports:               │  │  • Password policies             │  │
  • Social IdP:                │  │  • Email/phone verification      │  │
    Google, Facebook,          │  │  • MFA (SMS, TOTP)              │  │
    Apple, Amazon              │  │  • Account recovery             │  │
  • Enterprise federation:     │  │  • Custom attributes            │  │
    SAML 2.0, OIDC             │  └──────────────────────────────────┘  │
                               │                                        │
                               │  Output: JWT Tokens                    │
                               │  ┌──────────┬──────────┬────────────┐ │
                               │  │ ID Token │ Access   │ Refresh    │ │
                               │  │ (user    │ Token    │ Token      │ │
                               │  │  info)   │ (authz)  │ (renew)    │ │
                               │  └──────────┴──────────┴────────────┘ │
                               └────────────────────────────────────────┘
```

### User Pool Features

| Feature | Description |
|---|---|
| **User directory** | Built-in user database — sign-up, sign-in, account recovery |
| **Federation** | Sign in via Google, Facebook, Apple, Amazon, SAML 2.0, OIDC providers |
| **MFA** | SMS-based or TOTP (authenticator app). Adaptive authentication (risk-based). |
| **Password policies** | Min length, require uppercase/lowercase/numbers/special chars |
| **Email/Phone verification** | Verify via code sent to email or SMS |
| **Hosted UI** | Pre-built, customizable login/signup page (OAuth 2.0 flows) |
| **Lambda Triggers** | Run custom logic at key events in the auth flow |
| **Groups** | Organize users into groups with IAM role mappings |
| **Custom attributes** | Add application-specific user attributes |

### Lambda Triggers

```
  Authentication Flow:
  ┌─────────────┐   ┌──────────────┐   ┌──────────────┐   ┌─────────────┐
  │ Pre Sign-Up │──►│ Post Confirm │──►│ Pre Auth     │──►│ Post Auth   │
  │ (validate,  │   │ (welcome     │   │ (custom      │   │ (log event, │
  │  auto-      │   │  email,      │   │  validation) │   │  analytics) │
  │  confirm)   │   │  provision)  │   │              │   │             │
  └─────────────┘   └──────────────┘   └──────────────┘   └─────────────┘

  Token Generation:
  ┌─────────────────┐
  │ Pre Token Gen   │ → Customize/suppress claims in JWT
  │ (add custom     │
  │  claims)        │
  └─────────────────┘

  Custom messaging:
  ┌─────────────────┐
  │ Custom Message  │ → Customize verification/MFA messages
  └─────────────────┘

  Migration:
  ┌─────────────────┐
  │ User Migration  │ → Migrate users from legacy system on first sign-in
  └─────────────────┘
```

> [!TIP]
> **Exam Pattern:** "Custom logic during authentication" → **Cognito Lambda Triggers**. "Migrate users from existing database" → **User Migration Lambda Trigger** (migrates on first sign-in, no bulk migration needed). "Customize JWT claims" → **Pre Token Generation trigger**.

---

## Cognito Identity Pools (CIP) — Authorization / Federation

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                     Cognito Identity Pool                        │
  │                                                                  │
  │  Input: Authentication tokens from ANY identity provider         │
  │                                                                  │
  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐        │
  │  │ Cognito      │   │ Social IdP   │   │ SAML/OIDC    │        │
  │  │ User Pool    │   │ (Google,     │   │ Enterprise   │        │
  │  │ (JWT)        │   │  Facebook)   │   │ (JWT/SAML)   │        │
  │  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘        │
  │         │                  │                   │                 │
  │         └──────────────────┼───────────────────┘                │
  │                            ▼                                     │
  │                  ┌──────────────────┐                            │
  │                  │  Identity Pool   │                            │
  │                  │  exchanges token │                            │
  │                  │  for temporary   │                            │
  │                  │  AWS credentials │                            │
  │                  │  (via STS)       │                            │
  │                  └────────┬─────────┘                            │
  │                           │                                      │
  │              ┌────────────┼────────────┐                         │
  │              ▼            ▼            ▼                         │
  │         ┌────────┐  ┌────────┐  ┌────────┐                     │
  │         │   S3   │  │DynamoDB│  │ Lambda │                     │
  │         │        │  │        │  │        │                     │
  │         └────────┘  └────────┘  └────────┘                     │
  │                                                                  │
  │  Also supports: Guest/Unauthenticated access (limited perms)    │
  └──────────────────────────────────────────────────────────────────┘
```

### Identity Pool Key Concepts

| Concept | Description |
|---|---|
| **Authenticated role** | IAM role assumed by users who have signed in |
| **Unauthenticated role** | IAM role for guest users (limited permissions) |
| **Role mapping** | Map different IdPs or user groups to different IAM roles |
| **Policy variables** | Use `${cognito-identity.amazonaws.com:sub}` in IAM policies for per-user access |

### Per-User Access Example (S3)

```json
  {
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": [
      "arn:aws:s3:::my-bucket/users/${cognito-identity.amazonaws.com:sub}/*"
    ]
  }

  // Each user can only access their OWN folder in S3
  // user-abc → s3://my-bucket/users/user-abc/*
  // user-xyz → s3://my-bucket/users/user-xyz/*
```

> [!TIP]
> **Exam Pattern:** "Users need to upload files to their own S3 folder" or "fine-grained per-user access to AWS resources" → **Cognito Identity Pool** with IAM policy variables (`${cognito-identity.amazonaws.com:sub}`). "Guest access with limited permissions" → **Unauthenticated role** in Identity Pool.

---

## User Pools vs Identity Pools

| Feature | User Pools (CUP) | Identity Pools (CIP) |
|---|---|---|
| **Purpose** | **Authentication** (who are you?) | **Authorization** (what can you access?) |
| **Output** | **JWT tokens** (ID, Access, Refresh) | **Temporary AWS credentials** (Access Key, Secret Key, Session Token) |
| **User directory** | ✅ Built-in | ❌ No user management |
| **Federation** | ✅ (Google, Facebook, SAML, OIDC) | ✅ (accepts tokens from CUP, social IdPs, SAML/OIDC) |
| **MFA** | ✅ | ❌ (handled by the IdP) |
| **Access to AWS services** | ❌ (JWT is for your app, not AWS) | ✅ (STS credentials for S3, DynamoDB, etc.) |
| **Guest access** | ❌ | ✅ (unauthenticated role) |
| **Use case** | "Sign in to my web/mobile app" | "Access S3/DynamoDB from mobile app" |

---

## Complete Flow: User Pools + Identity Pools Together

```
  ┌──────┐  1. Sign in   ┌──────────┐  2. JWT tokens
  │ User │──────────────►│ Cognito  │───────────────┐
  │      │               │User Pool │               │
  │      │◄──────────────│          │               │
  │      │  JWT tokens   └──────────┘               │
  │      │                                          │
  │      │  3. Exchange JWT for AWS credentials     │
  │      │─────────────────────────────────────────►│
  │      │                                          │
  │      │               ┌──────────┐               │
  │      │               │ Cognito  │◄──────────────┘
  │      │               │Identity  │  JWT
  │      │               │Pool      │
  │      │◄──────────────│          │
  │      │  4. Temp AWS  │ (calls   │
  │      │  credentials  │  STS)    │
  │      │  (via STS)    └──────────┘
  │      │
  │      │  5. Access AWS resources directly
  │      │─────────────────────────────────►┌──────────┐
  │      │                                  │ S3       │
  │      │                                  │ DynamoDB │
  └──────┘                                  │ Lambda   │
                                            └──────────┘
```

---

## Cognito + API Gateway

### Pattern 1: Cognito User Pool Authorizer (Recommended)

```
  ┌──────┐  1. Auth   ┌──────────┐  2. JWT    ┌───────────┐  3. Invoke  ┌──────────┐
  │ User │──────────►│ Cognito  │──────────►│ API       │───────────►│ Lambda   │
  │      │           │ User Pool│           │ Gateway   │            │ Function │
  │      │◄──────────│          │           │           │            │          │
  │      │  JWT      └──────────┘           │ Cognito   │            └──────────┘
  └──────┘  tokens                          │ Authorizer│
                                            │ validates │
                                            │ JWT token │
                                            └───────────┘

  ✅ Fully managed — no custom code for token validation
  ✅ Least operational overhead
  ✅ API Gateway validates JWT automatically
```

### Pattern 2: Cognito + Identity Pools + API Gateway (IAM Auth)

```
  ┌──────┐  1. Auth    ┌──────────┐  2. JWT    ┌──────────┐  3. STS creds
  │ User │───────────►│ Cognito  │──────────►│ Cognito  │──────────────┐
  │      │            │User Pool │           │Identity  │              │
  │      │            └──────────┘           │Pool      │              │
  │      │                                   └──────────┘              │
  │      │  4. SigV4 signed request                                    │
  │      │────────────────────────────────►┌───────────┐              │
  │      │                                 │ API GW    │              │
  │      │                                 │ (IAM Auth)│              │
  └──────┘                                 └───────────┘
                                                │
  ✅ Fine-grained IAM policy control            ▼
  ✅ User-specific resource access          ┌──────────┐
                                            │ Lambda   │
                                            └──────────┘
```

> [!CAUTION]
> **Exam critical:** "Authenticate API calls with **least operational overhead**" → **Cognito User Pool + API Gateway Cognito Authorizer** (Pattern 1). "Users need to **directly access AWS resources** (S3, DynamoDB) from mobile app" → **Cognito User Pool + Identity Pool** (Pattern 2). Use **[[Amazon API Gateway|Lambda Authorizer]]** only for custom/3rd-party auth logic.

---

## Cognito Hosted UI

```
  ┌──────────────────────────────────────────────────┐
  │  Cognito Hosted UI                                │
  │                                                   │
  │  ┌──────────────────────────────────────────────┐│
  │  │                                              ││
  │  │  🔐 Sign In                                 ││
  │  │                                              ││
  │  │  Email:    [________________]                ││
  │  │  Password: [________________]                ││
  │  │                                              ││
  │  │  [        Sign In         ]                  ││
  │  │                                              ││
  │  │  ─── OR ───                                  ││
  │  │                                              ││
  │  │  [🔵 Sign in with Google ]                   ││
  │  │  [🔵 Sign in with Facebook]                  ││
  │  │  [⬛ Sign in with Apple  ]                   ││
  │  │                                              ││
  │  │  Don't have an account? Sign up              ││
  │  └──────────────────────────────────────────────┘│
  │                                                   │
  │  ✅ Pre-built, customizable with CSS/logo         │
  │  ✅ Supports OAuth 2.0 / OIDC flows               │
  │  ✅ Custom domain (auth.yourdomain.com)           │
  │  ✅ No frontend code for auth flows needed        │
  └──────────────────────────────────────────────────┘
```

> [!NOTE]
> The Hosted UI is useful for quick prototyping or when you want a complete auth flow without building custom login pages. It supports **Authorization Code Grant** (recommended for server-side apps) and **Implicit Grant** (for SPAs, less secure).

---

## Cognito Sync & AppSync

| Feature | Cognito Sync (Legacy) | AWS AppSync (Modern) |
|---|---|---|
| **Purpose** | Sync user data across devices | Real-time GraphQL API with offline support |
| **Status** | **Deprecated** — use AppSync instead | ✅ Active, recommended |
| **Offline** | ✅ | ✅ |
| **Conflict resolution** | Basic (last writer wins) | Advanced (auto-merge, custom Lambda) |

---

## Cognito Security Features

| Feature | Description |
|---|---|
| **Adaptive Authentication** | Risk-based auth — blocks suspicious sign-ins, requires additional verification |
| **Advanced Security** | Compromised credentials detection, IP-based risk scoring |
| **WAF Integration** | Protect User Pool endpoints with [[Amazon API Gateway|AWS WAF]] rules |
| **Encryption** | Data encrypted at rest and in transit |
| **Compliance** | HIPAA eligible, SOC, PCI DSS |

---

## Key Cognito Limits

| Parameter | Value |
|---|---|
| **User Pools per account** | 1,000 per region |
| **Users per User Pool** | 40,000,000 (40 million) |
| **Custom attributes** | 50 per User Pool |
| **Identity Pools per account** | 1,000 per region |
| **Groups per User Pool** | 10,000 |
| **Identity providers per User Pool** | 300 |
| **App clients per User Pool** | 1,000 |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Cognito:**
> - "User sign-up/sign-in for web/mobile app" → **Cognito User Pools**
> - "Authentication with Google/Facebook/Apple" → **Cognito User Pools** (social federation)
> - "Enterprise SSO with SAML" → **Cognito User Pools** (SAML 2.0 federation)
> - "Temporary AWS credentials for app users" → **Cognito Identity Pools**
> - "Users upload to their own S3 folder" → **Identity Pool + IAM policy variables**
> - "Guest access to AWS resources" → **Identity Pool unauthenticated role**
> - "Auth for API Gateway, least overhead" → **Cognito User Pool Authorizer**
> - "Custom auth logic / 3rd-party tokens" → **Lambda Authorizer** (not Cognito)
> - "Migrate users from legacy system" → **User Migration Lambda Trigger**
> - "MFA for web app users" → **Cognito User Pools** (SMS or TOTP MFA)
>
> **Key facts:**
> - User Pools = authentication → JWT tokens. Identity Pools = authorization → AWS credentials (STS).
> - User Pools support: local users, social IdP, SAML 2.0, OIDC federation.
> - Identity Pools accept tokens from: User Pools, social IdPs, SAML/OIDC, or custom auth.
> - Identity Pools support unauthenticated (guest) access with limited IAM role.
> - Lambda Triggers enable custom logic at every step of the auth flow.
> - Hosted UI provides pre-built login/signup pages with OAuth 2.0 flows.
> - Per-user S3 access: use `${cognito-identity.amazonaws.com:sub}` in IAM policy.
> - Cognito Authorizer on API Gateway validates JWT — no custom code needed.
> - Cognito Sync is deprecated — use **AWS AppSync** for cross-device data sync.
