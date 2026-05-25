---
tags: [concept, networking, dns, route53, routing-policy]
aliases: [Geolocation Routing, Geolocation Routing Policy]
date: 2026-05-04
---

# Route 53 Geolocation Routing

*Parent article: [[Amazon Route 53]]*

**Geolocation Routing** routes traffic based on the **geographic location of the user** (determined by the source IP of the DNS query). Unlike [[Route 53 Latency Routing|Latency Routing]], this is a **hard rule** — users in a specified location will **always** be directed to the associated resource, regardless of latency.

---

## How It Works

1. You create DNS records and map each record to a **geographic location**: continent, country, or US state.
2. When a DNS query arrives, Route 53 determines the user's location from their source IP.
3. Route 53 returns the record that matches the **most specific** location (state → country → continent → default).
4. If no match is found, the **Default** record is returned.

```
                     Route 53 (Geolocation)
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   Location: Europe   Location: Asia    Location: Default
        │                  │                  │
        ▼                  ▼                  ▼
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │  ALB EU   │      │  ALB AP   │      │  ALB US   │
   │ eu-west-1 │      │ap-south-1 │      │us-east-1  │
   └──────────┘      └──────────┘      └──────────┘

   User in France  → ALB EU ✅ (Europe match)
   User in Japan   → ALB AP ✅ (Asia match)
   User in Brazil  → ALB US ✅ (Default — no South America rule)
```

---

## Location Hierarchy (Most Specific Wins)

Route 53 evaluates from **most specific to least specific**:

```
1. US State (e.g., "California")     ← most specific
2. Country  (e.g., "United States")
3. Continent (e.g., "North America")
4. Default                           ← least specific (catch-all)
```

**Example:** If you have rules for "California" and "North America," a user in California matches the state rule. A user in Texas matches the continent rule.

---

## Key Characteristics

| Feature | Detail |
|---|---|
| **Location granularity** | Continent, Country, or US State |
| **Default record** | ⚠️ **Must** create a Default record (catch-all for unmatched locations) |
| **Health checks** | ✅ Can associate health checks |
| **Overlapping rules** | Most specific location wins |
| **IP-based detection** | Uses source IP of DNS resolver (not always the user's actual IP) |
| **Alias record** | ✅ Supported |

---

## The Default Record — Why It's Required

> [!CAUTION]
> **You MUST create a Default record.** Without it, users whose location cannot be determined (some IPs can't be geolocated) or whose location doesn't match any rule will receive **no DNS response** (effectively a DNS failure). The Default record ensures every user gets a valid response.

---

## Use Cases

### 1. Content Localization

Serve different language versions of your website based on user location:

| Location | Endpoint | Content |
|---|---|---|
| France | `eu-west-3` (Paris) | French language site |
| Germany | `eu-central-1` (Frankfurt) | German language site |
| Japan | `ap-northeast-1` (Tokyo) | Japanese language site |
| Default | `us-east-1` (Virginia) | English language site |

### 2. Regulatory Compliance / Data Sovereignty

Ensure users in specific countries **always** hit servers in their country/region (GDPR, data residency laws).

```
EU users → eu-west-1 (data stays in EU) ✅
US users → us-east-1 (data stays in US) ✅
```

### 3. Content Restriction / Geo-Blocking

Restrict or customize content based on the user's country (licensing, legal requirements).

### 4. Load Distribution by Geography

Spread load across regions by assigning continents to specific endpoints.

---

## Geolocation vs Latency vs Geoproximity

| Feature | Geolocation | Latency | Geoproximity |
|---|---|---|---|
| **Routing basis** | User's country/continent (hard rule) | Network latency (ms) | Physical distance + Bias |
| **Goal** | Compliance, localization | Best performance | Shift traffic by geography |
| **Can user override?** | No (always goes to mapped location) | No (always picks lowest latency) | Yes (Bias adjusts reach) |
| **Measures performance?** | ❌ | ✅ | ❌ |
| **Location granularity** | Continent / Country / US State | AWS Region | Lat/Long or AWS Region |
| **Default record needed?** | ✅ Yes (mandatory) | ❌ No | ❌ No |
| **Requires Traffic Flow** | ❌ | ❌ | ✅ |

> [!WARNING]
> **Geolocation ≠ Latency.** A user in France with a Geolocation rule mapping to `eu-west-1` will **always** go to `eu-west-1`, even if `us-east-1` has lower latency at that moment. Geolocation is a compliance tool, not a performance tool.

---

## Health Check Behavior

If the health check for a geolocation record **fails**:

1. Route 53 tries the **next less-specific** geolocation match.
   - State fails → tries Country → tries Continent → tries Default.
2. If all matches are unhealthy, Route 53 returns the **Default** record (even if unhealthy).

```
Location "France" → unhealthy ❌
  └─ Falls back to "Europe" → healthy ✅ (serves this record)
```

> [!TIP]
> **Exam Pattern:** "Users in a specific country MUST hit a specific server", "content localization by region", "data sovereignty", "regulatory compliance", "GDPR" → **Geolocation Routing**. If it says "fastest" or "lowest latency" → that's [[Route 53 Latency Routing|Latency]].
