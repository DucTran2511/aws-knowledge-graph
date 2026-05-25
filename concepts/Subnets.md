---
tags: [concept, networking, vpc]
aliases: [Subnet, Public Subnet, Private Subnet]
date: 2026-04-14
---

# Subnets (AWS VPC)

A **Subnet** is a logical subdivision of an IP network. In AWS, you use subnets to divide your [[VPC]]'s massive IP address range (CIDR block) into smaller, manageable chunks where you securely deploy resources like [[EC2]] instances.

## Key Rules & Characteristics
- **AZ Boundaries:** A subnet is strictly locked to **one and only one [[Availability Zones (AZ)|Availability Zone (AZ)]]**. You absolutely cannot stretch a single subnet across multiple AZs. 
- **AWS Reserved IPs:** In *every* subnet you create, AWS automatically reserves exactly **5 IP addresses** for its own internal networking. If you create a `/28` subnet (16 IPs total), only 11 are usable.
  - `.0` - Network address.
  - `.1` - Reserved for the VPC Router.
  - `.2` - Reserved for mapping to Amazon-provided DNS.
  - `.3` - Reserved for future use.
  - `.255` - Network broadcast address (even though AWS doesn't support broadcast, they still reserve it).

## Public vs. Private Subnets

There is no technical "switch" or flag that makes a subnet public or private. It is entirely determined by **routing**.

### Public Subnet
- **Definition:** A subnet whose Route Table sends internet-bound traffic (`0.0.0.0/0`) directly to an **Internet Gateway (IGW)**.
- **Access:** Resources inside (provided they have a Public IP or Elastic IP) can communicate with the open internet, and the internet can communicate back.
- **Common Use Cases:** Public-facing web servers, Application Load Balancers, Bastion Hosts (Jump Boxes), NAT Gateways.

### Private Subnet
- **Definition:** A subnet whose Route Table does *not* have a route to an Internet Gateway.
- **Access:** Resources inside cannot be reached from the internet. If they need to reach *out* to the internet (e.g., to download OS patches or software updates), their traffic must be routed through a **NAT Gateway** (which must be placed in a Public Subnet).
- **Common Use Cases:** Database servers, application backend servers, internal APIs, cache clusters (memcached/redis).

> [!TIP]
> The AWS Well-Architected Framework highly recommends putting all compute and data resources into **Private Subnets** by default, exposing only the absolute minimum (like Load Balancers) inside Public Subnets.
