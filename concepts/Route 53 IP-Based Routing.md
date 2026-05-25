---
tags: [concept, networking, dns, route53]
aliases: [IP-Based Routing]
date: 2026-05-04
---

# Route 53 IP-Based Routing

**IP-Based Routing** allows you to route traffic based on the source IP address of the client (usually the user's ISP or corporate network) making the DNS query.

## How It Works

- You provide Route 53 with a list of user IP CIDR blocks (a collection of IP address ranges).
- You map these CIDR blocks to specific locations/endpoints.
- When a DNS query comes in, Route 53 evaluates the source IP. If it falls within one of your defined CIDR blocks, Route 53 returns the associated record.
- You define a **Default** location for queries that do not match any specified CIDR blocks.

## Use Cases
- **ISP Optimization:** You have special peering agreements or dedicated network links with a specific Internet Service Provider (ISP). You want to route all users from that ISP to a specific, optimized endpoint.
- **Cost Optimization:** Route specific corporate IP ranges over a cheaper network path.
- **Granular Control:** When Geolocation routing is too broad (e.g., you need to route traffic differently within the same city or state based on the specific network).

> [!TIP]
> **Exam Tip:** If a question asks about routing traffic based on the user's specific ISP, CIDR block, or precise network pathing, choose **IP-Based Routing**.
