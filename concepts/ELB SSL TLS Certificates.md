---
tags: [concept, networking, security, load-balancing]
aliases: [ELB SSL, ELB TLS, SSL Termination, TLS Termination, SNI, Server Name Indication, ELB HTTPS]
date: 2026-04-28
---

# ELB — SSL/TLS Certificates

**SSL/TLS Certificates** on [[Elastic Load Balancer (ELB)]] enable encrypted HTTPS connections between clients and your load balancer. This is one of the most critical security concepts for AWS — nearly every production workload serves traffic over HTTPS, and the load balancer is where encryption is managed.

---

## SSL vs TLS — Quick Clarification

| Term | What It Is |
|---|---|
| **SSL** (Secure Sockets Layer) | The original encryption protocol. **Deprecated** — all versions (SSLv2, SSLv3) have known vulnerabilities. |
| **TLS** (Transport Layer Security) | The modern successor to SSL. Current versions: TLS 1.2 and **TLS 1.3** (recommended). |

In practice, everyone still says "SSL certificate" even though the actual protocol in use is TLS. AWS documentation and the exam use the terms interchangeably.

> [!NOTE]
> When you hear "SSL certificate" on the exam, it really means a **TLS certificate**. The underlying protocol is always TLS.

---

## How SSL/TLS Termination Works

**SSL Termination** (also called **TLS Offloading**) means the load balancer handles the computationally expensive encryption/decryption, so your backend servers don't have to.

```
                    ENCRYPTED                    UNENCRYPTED
                    (HTTPS)                      (HTTP)

    ┌────────┐     TLS 1.3      ┌──────────┐     HTTP        ┌──────────┐
    │ Client │ ◄══════════════► │   ELB    │ ──────────────► │ Backend  │
    │        │   port 443       │          │   port 80       │ (EC2)    │
    └────────┘                  └──────────┘                 └──────────┘
                                     │
                                Uses SSL/TLS
                                certificate
                                to decrypt
```

### Why Terminate at the Load Balancer?
1. **Offload CPU work** — TLS handshakes and encryption are expensive. Your backend servers can focus on application logic.
2. **Centralized certificate management** — One place to manage certs, not every server.
3. **Simplified backend** — Backend servers just serve HTTP. No cert rotation, no key management.
4. **Still secure** — Traffic between the ELB and backends is within the [[VPC]] (private network). AWS's internal network is encrypted at the physical layer.

### End-to-End Encryption (Optional)
If your compliance requirements demand encryption all the way to the backend:

```
    Client ══TLS══► ELB ══TLS══► Backend
              443         443
```

- The ELB re-encrypts traffic to the backend using a **separate TLS connection**.
- The backend needs its own SSL certificate (can be self-signed).
- This is called **end-to-end encryption** or **TLS re-encryption**.

---

## AWS Certificate Manager (ACM)

**ACM** is the AWS service for managing SSL/TLS certificates. It is the **primary** way to get certificates for your ELB.

### Key Features
| Feature | Description |
|---|---|
| **Free public certificates** | ACM provides SSL/TLS certificates at **no cost** for AWS services (ELB, [[CloudFront]], API Gateway) |
| **Automatic renewal** | ACM automatically renews certificates **before they expire** — no manual work |
| **Validation methods** | **DNS validation** (recommended — add a CNAME record) or **Email validation** |
| **Wildcard support** | `*.example.com` covers all subdomains |
| **Multi-domain (SAN)** | A single cert can cover multiple domains (e.g., `example.com` + `api.example.com` + `shop.example.com`) |

### What ACM Does NOT Do
- ACM public certificates **cannot be exported** — you can't download the private key and use them outside AWS.
- ACM **cannot be used with [[EC2]] directly** — it only works with integrated services (ELB, CloudFront, API Gateway).
- For EC2 or on-premises, you need to **upload your own certificate** to ACM or IAM.

> [!TIP]
> **Exam Pattern:** "Need a free, auto-renewing SSL certificate for your load balancer" → **ACM**. Always choose ACM over manual certificate management.

---

## Uploading Your Own Certificates

If you can't use ACM (e.g., you have a certificate from a third-party CA), you can upload it to:

| Location | Use Case |
|---|---|
| **ACM (imported)** | Preferred. Upload to ACM and reference it from the ELB listener. Note: **imported certs are NOT auto-renewed** — you must rotate manually. |
| **IAM Certificate Store** | Legacy method. Only use if ACM is not available in your region (rare). |

---

## HTTPS Listener Configuration

To enable HTTPS on your ELB, you configure an **HTTPS listener**:

```
Listener Configuration:
├── Protocol: HTTPS
├── Port: 443
├── Default SSL Certificate: arn:aws:acm:...:certificate/abc-123
├── Optional Additional Certificates (for SNI)
├── SSL/TLS Security Policy: ELBSecurityPolicy-TLS13-1-2-2021-06
└── Default Action: Forward to Target Group
```

### Default Certificate
Every HTTPS listener must have exactly **one default certificate**. This is used when:
- The client doesn't support SNI.
- No other certificate matches the requested hostname.

### Additional Certificates
You can attach **additional certificates** to support multiple domains on the same listener. The correct cert is selected via **SNI** (see below).

---

## Server Name Indication (SNI)

**SNI** is the protocol extension that allows a single load balancer to serve **multiple SSL certificates** for different domains on the same IP address and port.

### The Problem SNI Solves

Without SNI, one HTTPS listener = one SSL certificate = one domain. If you wanted to serve `api.example.com` and `shop.example.com` on the same load balancer, you needed **two load balancers** (or a multi-domain SAN certificate).

### How SNI Works

```
Step 1: Client initiates TLS handshake
        Client sends: "I want to connect to shop.example.com"
                       ↑ This is the SNI extension in the ClientHello

Step 2: ELB reads the hostname from the SNI extension

Step 3: ELB looks through its certificate list:
        - Default cert: *.example.com
        - Cert #2: shop.example.com    ← match!
        - Cert #3: api.example.com

Step 4: ELB uses shop.example.com's certificate for the TLS handshake

Step 5: Encrypted connection established with the correct cert ✓
```

### SNI Support by Load Balancer

| Load Balancer | SNI Support | Multiple Certs? |
|---|---|---|
| **[[Application Load Balancer (ALB)\|ALB]]** | ✅ Yes | Up to **25 certs** per listener (more via API) |
| **[[Network Load Balancer (NLB)\|NLB]]** | ✅ Yes | Up to **25 certs** per listener (more via API) |
| **CLB** | ❌ **No** | Only **1 cert** per CLB |
| **[[Gateway Load Balancer (GWLB)\|GWLB]]** | N/A | GWLB operates at Layer 3 — no TLS termination |

> [!IMPORTANT]
> **Exam Trigger:** "Multiple domains with different SSL certificates on the same load balancer" → you need **ALB or NLB with SNI**. CLB cannot do this — you'd need one CLB per domain.

> [!WARNING]
> CLB does **not** support SNI. If a question mentions CLB + multiple HTTPS domains, the answer is either:
> 1. **Migrate to ALB** (preferred), or
> 2. Deploy **one CLB per domain** (wasteful, but technically works).

---

## SSL/TLS Security Policies

A **Security Policy** is a predefined set of TLS protocols and ciphers that the ELB uses during the TLS handshake. It determines:
- Which **TLS versions** are allowed (TLS 1.0, 1.1, 1.2, 1.3).
- Which **cipher suites** are allowed (the algorithms for encryption).

### Common Policies

| Policy Name | TLS Versions | Use Case |
|---|---|---|
| `ELBSecurityPolicy-TLS13-1-2-2021-06` | TLS 1.2 + 1.3 | **Recommended** — modern and secure |
| `ELBSecurityPolicy-TLS-1-2-2017-01` | TLS 1.2 only | Strict — no TLS 1.3 |
| `ELBSecurityPolicy-2016-08` | TLS 1.0, 1.1, 1.2 | Legacy compatibility (avoid if possible) |
| `ELBSecurityPolicy-TLS13-1-3-2021-06` | TLS 1.3 only | Maximum security, limited client support |

### How to Choose
- **Default:** Use the latest recommended policy (`TLS13-1-2`).
- **Legacy clients:** If you must support old browsers/devices, use a policy that includes TLS 1.0/1.1 (but only as a last resort).
- **Compliance (PCI DSS, HIPAA):** Requires at least TLS 1.2. Disable TLS 1.0 and 1.1.

> [!TIP]
> **Exam Tip:** If a question asks how to disable TLS 1.0 on your load balancer → change the **Security Policy** to one that only supports TLS 1.2+.

---

## SSL by Load Balancer Type — Comparison

| Feature | ALB | NLB | CLB | GWLB |
|---|---|---|---|---|
| **HTTPS/TLS termination** | ✅ (HTTPS listener) | ✅ (TLS listener) | ✅ (HTTPS/SSL listener) | ❌ (Layer 3) |
| **SNI** | ✅ | ✅ | ❌ | N/A |
| **Multiple certs** | ✅ (25+) | ✅ (25+) | ❌ (1 per CLB) | N/A |
| **TLS passthrough** | ❌ (always terminates) | ✅ (TCP listener) | ❌ | N/A |
| **ACM integration** | ✅ | ✅ | ✅ | N/A |
| **Security Policies** | ✅ | ✅ | ✅ | N/A |
| **End-to-end encryption** | ✅ (re-encrypt to backend) | ✅ (passthrough or re-encrypt) | ✅ (re-encrypt) | N/A |

### NLB: TLS Termination vs TCP Passthrough
NLB gives you a unique choice that ALB does not:

| Mode | Listener Protocol | What Happens |
|---|---|---|
| **TLS Termination** | TLS | NLB decrypts, forwards unencrypted TCP to backend |
| **TCP Passthrough** | TCP | NLB forwards the encrypted traffic as-is — the **backend** handles TLS |

**Use case for passthrough:** Your backend needs to see the raw TLS connection (e.g., mutual TLS / mTLS, custom certificate validation).

---

## Mutual TLS (mTLS)

**Mutual TLS** (also called **two-way TLS** or **client certificate authentication**) adds an extra layer: not only does the server prove its identity to the client, but the **client also proves its identity to the server** using a client certificate.

### ALB mTLS Support (2023+)
ALB now supports mTLS in two modes:

| Mode | Description |
|---|---|
| **Passthrough** | ALB sends the client certificate to the backend in the `X-Amzn-Mtls-Clientcert` header. Your application validates it. |
| **Verify** | ALB validates the client certificate against a **Trust Store** (a CA bundle you upload). Only clients with valid certs are allowed through. |

### Trust Store
- A collection of **CA certificates** (root and intermediate) that ALB uses to validate client certificates.
- Created and managed in the ELB console or API.
- Supports **Certificate Revocation Lists (CRLs)** to reject compromised client certs.

> [!NOTE]
> mTLS is an advanced topic. For the SAA-C03 exam, the key fact is: **ALB supports mTLS** for scenarios where you need to authenticate clients using certificates (common in B2B APIs, financial services, IoT).

---

## Common Architecture Patterns

### Pattern 1: Simple HTTPS Termination (Most Common)
```
Client ──HTTPS──► ALB ──HTTP──► EC2 (port 80)
                    │
              ACM certificate
              for example.com
```

### Pattern 2: Multi-Domain with SNI
```
Client ──HTTPS──► ALB ──HTTP──► Target Group A (api.example.com)
                    │
                    ├──HTTP──► Target Group B (shop.example.com)
                    │
              3 certificates:
              - api.example.com
              - shop.example.com
              - *.example.com (default)
```

### Pattern 3: HTTP → HTTPS Redirect
```
Listener :80  → Redirect to HTTPS (301)
Listener :443 → Forward to Target Group (with ACM cert)
```

This is the **standard production pattern**. All HTTP traffic is automatically redirected to HTTPS.

### Pattern 4: End-to-End Encryption
```
Client ──HTTPS──► ALB ──HTTPS──► EC2 (port 443, self-signed cert)
                    │
              ACM certificate         Backend certificate
              (public-facing)         (internal, can be self-signed)
```

### Pattern 5: NLB TLS Passthrough
```
Client ──TLS──► NLB (TCP listener, no termination) ──TLS──► Backend handles everything
```

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → SSL/TLS on ELB:**
> - "HTTPS on the load balancer" → Configure HTTPS listener + ACM certificate
> - "Free, auto-renewing certificate" → **ACM**
> - "Multiple domains on one LB with different certs" → **ALB/NLB + SNI**
> - "CLB + multiple HTTPS domains" → Migrate to ALB, or one CLB per domain
> - "Disable TLS 1.0" → Change the **Security Policy**
> - "End-to-end encryption" → Backend also needs a cert, ELB re-encrypts
> - "Client must prove identity with a certificate" → **mTLS on ALB**
> - "TLS passthrough to backend" → **NLB with TCP listener**
> - "Cannot export the certificate" → That's an **ACM public certificate** (by design)
