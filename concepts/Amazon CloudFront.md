---
tags: [concept, networking, cdn, caching, security, performance, global]
aliases: [CloudFront, CF, CDN, Content Delivery Network, AWS CloudFront]
date: 2026-05-15
---

# Amazon CloudFront

**Amazon CloudFront** is a **Content Delivery Network (CDN)** service that distributes content to end users with **low latency** and **high transfer speeds** through a global network of **edge locations** (450+ Points of Presence).

CloudFront **caches content at edge locations** worldwide, so subsequent requests are served from the nearest edge rather than the origin — dramatically reducing latency.

> [!IMPORTANT]
> **Core exam concept:** CloudFront is NOT a web server or load balancer. It is a **caching and distribution layer** that sits in front of your origin (S3, ALB, EC2, HTTP server). It improves performance by serving cached content from locations close to the user.

---

## How CloudFront Works

```
  User (Sydney)                                              Origin (us-east-1)
       │                                                          │
       │  1. Request example.com/image.jpg                       │
       ▼                                                          │
  ┌──────────────────┐                                           │
  │  Edge Location   │── Cache HIT? ──► Return cached copy       │
  │  (Sydney)        │                                           │
  │                  │── Cache MISS ──► Forward to Origin ───────►│
  │                  │◄───────────────── Return response ─────────│
  │                  │   (cache the response for next request)   │
  └──────────────────┘                                           │
       │                                                          
       │  2. Response (low latency — served from edge)
       ▼
  User receives content
```

### Key Components

| Component | Description |
|---|---|
| **Edge Location** | Where content is cached. 450+ locations globally. NOT an AZ or Region. |
| **Regional Edge Cache** | Larger cache between edge locations and origin. Holds less-popular content longer. |
| **Origin** | The source of the original content (S3 bucket, ALB, EC2, custom HTTP server) |
| **Distribution** | The CloudFront configuration — defines origins, cache behaviors, and settings |
| **Behaviors** | Rules that define which path patterns go to which origin, with what cache settings |

```
  Users worldwide
  ┌──────┐ ┌──────┐ ┌──────┐
  │Sydney│ │London│ │Tokyo │
  └──┬───┘ └──┬───┘ └──┬───┘
     │        │        │
     ▼        ▼        ▼
  ┌──────┐ ┌──────┐ ┌──────┐
  │Edge  │ │Edge  │ │Edge  │      ◄── 450+ Edge Locations
  │Loc.  │ │Loc.  │ │Loc.  │
  └──┬───┘ └──┬───┘ └──┬───┘
     │        │        │
     └────────┼────────┘
              ▼
     ┌────────────────┐
     │ Regional Edge  │               ◄── Larger cache, fewer locations
     │ Cache          │
     └───────┬────────┘
             │
             ▼
     ┌────────────────┐
     │    Origin      │               ◄── S3, ALB, EC2, Custom HTTP
     │  (us-east-1)   │
     └────────────────┘
```

---

## CloudFront Origins

### S3 Bucket as Origin

- Distribute and **cache files at the edge**.
- Enhanced security with **Origin Access Control (OAC)** — allows CloudFront to access private S3 buckets.
- CloudFront can also be used as an **ingress** (upload files to S3 via CloudFront).

```
  ┌──────────┐  HTTPS   ┌────────────┐    S3 API    ┌──────────────┐
  │  Browser  │────────►│ CloudFront  │────────────►│  S3 Bucket   │
  │           │◄────────│  (OAC)      │◄────────────│  (private)   │
  └──────────┘          └────────────┘              └──────────────┘
                              │
                        OAC signs requests
                        so S3 trusts CloudFront
```

### Custom Origin (HTTP Backend)

Any HTTP endpoint can be a CloudFront origin:

| Origin Type | Example |
|---|---|
| **Application Load Balancer** | ALB in front of EC2 fleet |
| **EC2 Instance** | Direct EC2 public IP |
| **S3 Static Website** | S3 website endpoint (different from S3 bucket origin!) |
| **Any HTTP server** | On-premises server, API endpoint |

> [!WARNING]
> **Exam trap:** S3 bucket origin vs S3 website endpoint are different origins in CloudFront. For **static website hosting**, use the **S3 website endpoint** as a custom origin (not the S3 bucket). For file distribution with OAC, use the **S3 bucket** origin.

---

## Origin Access Control (OAC)

OAC is the **recommended** way to restrict S3 bucket access so that content is only accessible through CloudFront (replaces the legacy Origin Access Identity — OAI).

```
  Direct S3 access: BLOCKED ✗
  ┌──────┐ ──── s3://bucket/file.jpg ────► ┌──────┐
  │ User │                                 │  S3  │  ← Bucket policy
  └──────┘                                 └──────┘    denies direct access

  Via CloudFront: ALLOWED ✓
  ┌──────┐ ──► ┌────────────┐ ──► ┌──────┐
  │ User │     │ CloudFront │     │  S3  │  ← Bucket policy allows
  └──────┘     │   (OAC)    │     └──────┘    CloudFront service principal
               └────────────┘
```

The S3 bucket policy grants access to the **CloudFront service principal**:

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::account-id:distribution/EDFDVBD6EXAMPLE"
      }
    }
  }]
}
```

> [!IMPORTANT]
> **OAC vs OAI:** OAC is the modern replacement. OAC supports **all S3 buckets in all regions**, **SSE-KMS** encryption, and **dynamic requests** (PUT/DELETE). OAI is legacy — exam questions will increasingly reference OAC.

---

## CloudFront vs S3 Cross-Region Replication

| Feature | CloudFront | S3 Cross-Region Replication |
|---|---|---|
| **Architecture** | Global edge network (450+ locations) | Specific target regions you choose |
| **Content** | Caches static AND dynamic content | Replicates entire objects |
| **TTL** | Cached for a TTL (typically hours/day) | Near real-time, read-only replicas |
| **Updates** | Serves stale until TTL expires | Updated in near real-time |
| **Use case** | Static content for global audience | Dynamic content in specific regions, low-latency reads |
| **Setup** | One distribution, automatic global reach | Must configure each target region |

> [!TIP]
> **Exam Pattern:** "Serve static content globally with low latency" → **CloudFront**. "Real-time replicated data available in specific regions" → **S3 CRR**.

---

## CloudFront Caching

### Cache Key

The **cache key** is the unique identifier for each cached object. By default it includes:
- **Hostname** + **Resource path** (URL path)

You can customize the cache key to include:
- **HTTP Headers** (e.g., `Accept-Language`, `Authorization`)
- **Query Strings** (e.g., `?size=large&color=red`)
- **Cookies** (e.g., session cookies)

```
  Default Cache Key:         Custom Cache Key:

  example.com/image.jpg      example.com/image.jpg
                              + Header: Accept-Language
                              + Query: ?size=large
                              + Cookie: session_id

  More items in cache key = MORE cache misses = LESS caching efficiency
  Fewer items in cache key = MORE cache hits = BETTER performance
```

> [!WARNING]
> **Exam trap:** Adding more elements to the cache key REDUCES cache hit ratio. Only include values that actually affect the response. Use **Cache Policies** to control what's in the cache key and **Origin Request Policies** to forward values to the origin WITHOUT adding them to the cache key.

### Cache Policies vs Origin Request Policies

| Policy | Purpose | Affects Cache Key? |
|---|---|---|
| **Cache Policy** | Define what's included in the cache key (headers, cookies, query strings) + TTL settings | ✅ Yes |
| **Origin Request Policy** | Forward additional values to the origin that are NOT part of the cache key | ❌ No |

```
  Client Request
       │
       ▼
  ┌─────────────────────────────┐
  │       CloudFront            │
  │                             │
  │  Cache Policy: cache key    │──► Lookup in cache
  │  includes ?lang parameter   │
  │                             │
  │  Cache MISS ──────────────────────────────────┐
  │                             │                  │
  │  Origin Request Policy:     │                  │
  │  forward User-Agent header  │──► Origin gets   │
  │  (NOT in cache key)         │   lang + UA      │
  └─────────────────────────────┘                  │
                                                   ▼
                                              ┌─────────┐
                                              │ Origin  │
                                              └─────────┘
```

### Cache Invalidation

Force CloudFront to remove cached content before TTL expires:

| Method | Example | Cost |
|---|---|---|
| **Invalidate specific file** | `/images/logo.png` | First 1,000 paths/month free |
| **Invalidate with wildcard** | `/images/*` | Counts as 1 invalidation path |
| **Invalidate everything** | `/*` | Counts as 1 invalidation path |

> [!TIP]
> **Exam Pattern:** "Updated S3 files but CloudFront still serves old version" → **Create a Cache Invalidation** or use **file versioning** in the URL (e.g., `/images/logo_v2.png`). File versioning is the best practice — no invalidation cost, instant update.

---

## CloudFront Cache Behaviors

Cache Behaviors let you route different URL path patterns to **different origins** with **different cache settings**.

```
  CloudFront Distribution
  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  Path Pattern          Origin              Cache TTL   │
  │  ─────────────         ──────              ─────────   │
  │  /api/*          →     ALB (dynamic)       0 (no cache)│
  │  /images/*       →     S3 Bucket (static)  1 day       │
  │  /static/*       →     S3 Bucket (static)  7 days      │
  │  Default (*)     →     ALB (app server)    5 minutes   │
  │                                                        │
  └────────────────────────────────────────────────────────┘
```

- Behaviors are evaluated in **order** (first match wins). Default behavior (`*`) is always last.
- Each behavior can have its own cache policy, origin request policy, viewer protocol policy, and function associations.

---

## CloudFront Security

### Viewer Protocol Policy

Controls the protocol between the **viewer (client) and CloudFront**:

| Policy | Behavior |
|---|---|
| **HTTP and HTTPS** | Accept both |
| **Redirect HTTP to HTTPS** | Automatically redirect HTTP → HTTPS (most common) |
| **HTTPS Only** | Reject HTTP requests |

### Origin Protocol Policy

Controls the protocol between **CloudFront and the origin**:

| Policy | Behavior |
|---|---|
| **HTTP Only** | CloudFront connects to origin via HTTP |
| **HTTPS Only** | CloudFront connects to origin via HTTPS |
| **Match Viewer** | CloudFront uses whatever protocol the viewer used |

> [!NOTE]
> For **S3 bucket origins**, the origin protocol is always **HTTPS** (managed by AWS). For **S3 website endpoints**, it's always **HTTP only** (S3 websites don't support HTTPS natively).

### Geo Restriction (Geo-Blocking)

Restrict access to your distribution based on the **country** of the viewer:

| Type | Description |
|---|---|
| **Allowlist** | Only users from approved countries can access |
| **Blocklist** | Users from specified countries are blocked |

Uses a 3rd-party Geo-IP database to determine the user's country.

> [!TIP]
> **Exam Pattern:** "Block users from specific countries from accessing content" → CloudFront **Geo Restriction**. For more granular location-based routing (not blocking) → [[Amazon Route 53|Route 53]] **Geolocation Routing**.

---

## CloudFront Signed URLs & Signed Cookies

Provide **time-limited access** to private CloudFront content.

### Signed URL vs Signed Cookie

| Feature | Signed URL | Signed Cookie |
|---|---|---|
| **Scope** | Access to **one individual file** | Access to **multiple files** (entire path/domain) |
| **URL change** | URL is modified with signature | URL stays the same; cookie is set in browser |
| **Use case** | Download a specific premium video | Access an entire premium section of a website |

### CloudFront Signed URL vs S3 Pre-Signed URL

| Feature | CloudFront Signed URL | S3 Pre-Signed URL |
|---|---|---|
| **Access path** | Through CloudFront → cached at edge | Direct to S3 (bypasses CloudFront) |
| **Scope** | Can restrict by IP, path, date/time | Inherits IAM user's permissions |
| **Caching** | ✅ Benefits from edge caching | ❌ No caching |
| **Key type** | CloudFront key pair (root account) or Trusted Key Group | IAM credentials of the signer |
| **Use case** | Distribute private content globally with caching | Quick, temporary, direct S3 access |

```
  CloudFront Signed URL:                   S3 Pre-Signed URL:

  User ──► Edge Location ──► Origin        User ──► S3 (directly)
           (cached content)                        (no caching)
           
  ✓ Global, low-latency                   ✓ Simple, direct
  ✓ IP restriction, path restriction       ✓ IAM-based permissions
  ✓ Works with any origin                  ✗ S3 only
```

> [!IMPORTANT]
> **Exam Pattern:** "Distribute private content globally via CDN" → **CloudFront Signed URL/Cookie**. "Give temporary direct access to an S3 object" → **S3 Pre-Signed URL**.

### Trusted Key Groups (Recommended)

- Create **CloudFront Key Groups** using your own public/private key pairs.
- Assign the key group to the distribution's **Trusted Signers**.
- Use the **private key** in your application to sign URLs/cookies.
- Can manage keys via **IAM API** — no need for root account.

> [!NOTE]
> The legacy method used a **CloudFront Key Pair** created by the root account. **Trusted Key Groups** are the recommended approach — they support IAM-based management and key rotation.

---

## CloudFront Functions vs Lambda@Edge

CloudFront supports running code at the edge for request/response manipulation:

### CloudFront Functions

| Aspect | Detail |
|---|---|
| **Runtime** | JavaScript only |
| **Execution** | At **Edge Locations** (450+) |
| **Scale** | Millions of requests per second |
| **Max execution time** | < 1 ms |
| **Max memory** | 2 MB |
| **Triggers** | Viewer Request, Viewer Response only |
| **Network/file access** | ❌ No |
| **Use cases** | URL rewrites/redirects, header manipulation, cache key normalization, JWT validation |

### Lambda@Edge

| Aspect | Detail |
|---|---|
| **Runtime** | Node.js, Python |
| **Execution** | At **Regional Edge Caches** (fewer locations) |
| **Scale** | Thousands of requests per second |
| **Max execution time** | 5–30 seconds (depending on trigger) |
| **Max memory** | 128–10,240 MB |
| **Triggers** | Viewer Request, Viewer Response, Origin Request, Origin Response |
| **Network/file access** | ✅ Yes |
| **Use cases** | A/B testing, dynamic content based on device, user authentication, API calls to external services |

```
  Request Flow:

  Viewer ──► [Viewer Request] ──► CloudFront Cache ──► [Origin Request] ──► Origin
                   │                                         │
           CloudFront Functions                       Lambda@Edge only
           OR Lambda@Edge                             
                                                            
  Origin ──► [Origin Response] ──► CloudFront Cache ──► [Viewer Response] ──► Viewer
                   │                                         │
           Lambda@Edge only                           CloudFront Functions
                                                      OR Lambda@Edge
```

### Comparison

| Feature | CloudFront Functions | Lambda@Edge |
|---|---|---|
| **Trigger points** | Viewer only (2) | All 4 (Viewer + Origin) |
| **Speed** | Sub-millisecond | Milliseconds to seconds |
| **Scale** | Millions/sec | Thousands/sec |
| **Body access** | ❌ | ✅ |
| **External calls** | ❌ | ✅ |
| **Cost** | Very cheap (1/6th of Lambda@Edge) | More expensive |
| **Deploy location** | All edge locations | Regional edge caches |

> [!TIP]
> **Exam Pattern:** "Simple URL rewrite or header manipulation at massive scale" → **CloudFront Functions**. "Need to call an external API, access the request body, or perform complex logic" → **Lambda@Edge**.

---

## CloudFront with ALB or EC2

### EC2 as Origin

- EC2 instance **must be public** (CloudFront connects over the public internet).
- Security Group must allow **CloudFront edge IP ranges** (published by AWS as a public IP list).

### ALB as Origin

- ALB **must be public** (CloudFront connects over the public internet).
- EC2 instances behind ALB can be **private**.
- ALB Security Group must allow CloudFront edge IPs.

```
  CloudFront with ALB + Private EC2:

  ┌──────┐     ┌────────────┐     ┌──────────┐     ┌──────────┐
  │ User │────►│ CloudFront │────►│   ALB    │────►│   EC2    │
  └──────┘     └────────────┘     │ (public) │     │ (private)│
                                  └──────────┘     └──────────┘
                Public Internet        │          Private Subnet
                                  SG: Allow         SG: Allow
                                  CF edge IPs       ALB SG
```

> [!WARNING]
> **Exam trap:** EC2 instances used as CloudFront origins must be **public** with a public IP. But EC2 behind an ALB can be **private** — the ALB acts as the public-facing origin.

---

## HTTPS & SSL/TLS Certificates

### Default Domain

Every CloudFront distribution gets a default domain: `d111111abcdef8.cloudfront.net` with a default AWS-managed SSL certificate.

### Custom Domain (CNAME)

To use a custom domain (e.g., `cdn.example.com`):

1. Add the custom domain as a **CNAME** (Alternate Domain Name) in CloudFront.
2. Provision an **SSL certificate in ACM** (AWS Certificate Manager).
3. ⚠️ The certificate **MUST be in us-east-1** (N. Virginia) — regardless of where your origin is.
4. Create a DNS record ([[Amazon Route 53|Route 53]] Alias or CNAME) pointing to CloudFront.

> [!CAUTION]
> **Exam critical:** ACM certificates for CloudFront **must** be in **us-east-1**. This is a frequent exam question. Even if your origin is in eu-west-1, the CloudFront SSL cert must be in us-east-1.

### SNI (Server Name Indication)

| Method | Description | Cost |
|---|---|---|
| **SNI** (default) | Multiple SSL certs on one IP — client sends hostname in TLS handshake | Free |
| **Dedicated IP** | Dedicated IP at each edge location for clients that don't support SNI | $600/month per edge location |

> [!TIP]
> Only very old browsers (pre-2006) don't support SNI. The exam answer is almost always **SNI** unless the question specifically mentions legacy clients.

---

## CloudFront Price Classes

CloudFront edge locations span the globe, but not all regions cost the same. **Price Classes** let you reduce cost by limiting which edge locations are used:

| Price Class | Edge Locations Included |
|---|---|
| **Price Class All** | All edge locations — best performance |
| **Price Class 200** | Most regions, excludes the most expensive (e.g., South America, Australia) |
| **Price Class 100** | Only the least expensive regions (North America + Europe) |

```
  ┌──────────────────────────────────────────────────────┐
  │                  Price Classes                       │
  │                                                      │
  │  All:  NA + EU + Asia + SA + Africa + Oceania       │
  │  200:  NA + EU + Asia + some others (no SA)         │
  │  100:  NA + EU only                                 │
  │                                                      │
  │  Cost:   All > 200 > 100                            │
  │  Perf:   All > 200 > 100                            │
  └──────────────────────────────────────────────────────┘
```

---

## CloudFront + S3 Static Website (HTTPS)

The most common architecture for hosting a secure static website:

```
  ┌──────────┐  HTTPS   ┌────────────┐  S3 API   ┌──────────────┐
  │  Browser  │────────►│ CloudFront  │──────────►│  S3 Bucket   │
  │           │◄────────│  + ACM cert │◄──────────│  (private)   │
  └──────────┘          │  + OAC      │           └──────────────┘
                        └────────────┘
                              │
                        ┌────────────┐
                        │  Route 53  │
                        │  Alias     │
                        │  Record    │
                        └────────────┘

  Components:
  1. S3 Bucket (private, Block Public Access ON)
  2. CloudFront Distribution with OAC
  3. S3 Bucket Policy → allow CloudFront service principal
  4. ACM Certificate (us-east-1) for custom domain
  5. Route 53 Alias record → CloudFront distribution
```

> [!TIP]
> **Exam Pattern:** "Secure static website with custom domain and HTTPS" → **CloudFront + S3 (OAC) + ACM (us-east-1) + Route 53 Alias**.

---

## CloudFront Origin Groups (High Availability)

Origin Groups provide **failover** for origins — if the primary origin fails, CloudFront automatically routes to a secondary origin.

```
  ┌────────────┐
  │ CloudFront │
  └─────┬──────┘
        │
  ┌─────▼──────────────────────┐
  │     Origin Group           │
  │  ┌────────────────────┐    │
  │  │  Primary Origin    │────│──► S3 Bucket (us-east-1)
  │  │  (if 5xx or 4xx)   │    │
  │  └────────┬───────────┘    │
  │           │ failover       │
  │  ┌────────▼───────────┐    │
  │  │  Secondary Origin  │────│──► S3 Bucket (eu-west-1)
  │  └────────────────────┘    │
  └────────────────────────────┘
```

- Define failover criteria: specific HTTP status codes (e.g., 500, 502, 503, 504, 404, 403).
- Combined with **S3 CRR**, provides a fully redundant static content architecture.

---

## AWS WAF + CloudFront

**AWS WAF** (Web Application Firewall) can be attached to CloudFront to protect against web exploits:

- Block/allow requests based on **IP addresses**, **geographic location**, **request size**, **string matching**, **SQL injection**, **XSS**.
- Define **Web ACLs** (Access Control Lists) with **rules** and **rule groups**.
- WAF evaluates requests at the **edge** — malicious requests are blocked before reaching your origin.

> [!TIP]
> **Exam Pattern:** "Protect web application from SQL injection and XSS" → **AWS WAF**. "Block specific IPs at the CDN level" → **AWS WAF + CloudFront**.

---

## AWS Shield + CloudFront

All CloudFront distributions are automatically protected by **AWS Shield Standard** (free) against DDoS attacks.

| Shield Tier | Protection | Cost |
|---|---|---|
| **Shield Standard** | Automatic protection against L3/L4 DDoS attacks | Free (included) |
| **Shield Advanced** | Enhanced DDoS protection, 24/7 DRT support, cost protection | $3,000/month |

---

## Field-Level Encryption

Protect **specific sensitive fields** (e.g., credit card numbers) in POST requests through the entire request chain:

```
  User ──► Edge Location ──► Origin ──► Application
             │
             │ Encrypts specific fields using
             │ a public key you provide
             │
             ▼
  Only the application with the private key
  can decrypt those specific fields
```

- Adds an extra layer of encryption on top of HTTPS.
- Ensures intermediate systems (caches, proxies) cannot read sensitive data.

---

## Real-Time Logs

CloudFront can send **real-time, request-level logs** to **Amazon Kinesis Data Streams** for analysis:

- Choose which **fields** to log, which **cache behaviors** to log, and what **sampling rate**.
- Use downstream services (Kinesis Data Firehose → S3 → Athena) for analysis.

Standard **access logs** can also be delivered to an **S3 bucket** (not real-time, but comprehensive).

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → CloudFront:**
> - "Low latency content delivery globally" → **CloudFront**
> - "HTTPS for S3 static website" → **CloudFront + ACM**
> - "Cache static content at the edge" → **CloudFront**
> - "Serve private content via CDN" → **CloudFront Signed URL / Signed Cookie**
> - "Signed URL for one file" → **CloudFront Signed URL**
> - "Signed cookie for multiple files" → **CloudFront Signed Cookie**
> - "Simple URL rewrite at the edge" → **CloudFront Functions**
> - "Complex logic at the edge (API calls, body access)" → **Lambda@Edge**
> - "Restrict content by country" → **CloudFront Geo Restriction**
> - "Reduce CloudFront costs" → **Price Class 100 or 200**
> - "Protect against DDoS" → **Shield Standard** (free with CF) or **Shield Advanced**
> - "Block SQL injection at CDN" → **AWS WAF + CloudFront**
> - "ACM certificate for CloudFront" → **Must be in us-east-1**
> - "Origin failover for CloudFront" → **Origin Groups**
> - "Restrict S3 access to CloudFront only" → **OAC** (Origin Access Control)
>
> **Key facts:**
> - CloudFront has **450+ edge locations** globally.
> - ACM cert for CloudFront must be in **us-east-1** (N. Virginia).
> - **OAC** replaces the legacy **OAI** for S3 origins.
> - CloudFront Functions: Viewer events only, < 1 ms, JavaScript, millions/sec.
> - Lambda@Edge: All 4 events, seconds, Node.js/Python, thousands/sec.
> - Default TTL is **24 hours** (86400 seconds).
> - Cache invalidation: first 1,000 paths/month free, then $0.005 per path.
> - EC2 origin must be **public**. ALB origin must be **public** (EC2 behind it can be private).
> - CloudFront Signed URL = 1 file. Signed Cookie = multiple files.
> - **SNI** is the default (free) for serving HTTPS with custom SSL certs.
