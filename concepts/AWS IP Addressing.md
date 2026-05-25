---
tags: [concept, networking, compute]
aliases: [Private IP, Public IP, Elastic IP, EIP]
date: 2026-04-14
---

# AWS IP Addressing (Private vs Public vs Elastic)

When you deploy a resource like an [[EC2]] instance in AWS, it requires an IP address to communicate across networks. AWS provides three primary types of IPv4 addressing behaviors for instances:

### 1. Private IP
- **Scope:** Internal to the AWS network (your VPC - Virtual Private Cloud).
- **Persistence:** **Static** for the life of the instance. It never changes as long as the instance exists, even if you stop and start the machine.
- **Cost:** Always FREE.
- **Use Case:** Secure communication between resources inside the same VPC (e.g., App server talking to Database server).

### 2. Public IP
- **Scope:** Routable over the open Internet.
- **Persistence:** **Ephemeral (Temporary)**. If you Stop/Start your [[EC2]] instance, the Public IP address is released back into the AWS pool, and you will receive a new/different Public IP upon starting.
- **Cost:** Incurs an hourly charge ($0.005/hr for all public IPv4).
- **Use Case:** Initial internet access or temporary workloads. It is **not** suitable for DNS A-records since the IP will change upon a restart.

### 3. Elastic IP (EIP)
- **Scope:** Routable over the open Internet.
- **Persistence:** **Static (Permanent)**. This is a fixed public IP address that you allocate to your AWS account. It persists even if the attached instance is stopped or terminated, until you actively release it.
- **Cost:** Incurs an hourly charge. *Historically*, AWS charged you only if you hoarded them (didn't attach them to a running instance). However, under recent AWS pricing changes, **all public IPv4 addresses (including attached Elastic IPs) incur charges**.
- **Use Case:** Workloads requiring a fixed public IP address, such as an entry point for a static DNS record, or masking an instance failure by rapidly remaping the EIP to a healthy standby instance.

> [!TIP]
> You can rapidly move an Elastic IP from a failed instance to a healthy instance, allowing you to bypass DNS propagation delays in High Availability architectures.
