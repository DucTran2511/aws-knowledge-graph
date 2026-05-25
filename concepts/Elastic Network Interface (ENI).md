---
tags: [concept, networking, compute]
aliases: [ENI, Elastic Network Interfaces]
date: 2026-04-14
---

# Elastic Network Interface (ENI)

An **Elastic Network Interface (ENI)** is a logical networking component in AWS that represents a virtual network card. It is the fundamental mechanism that allows an [[EC2]] instance to connect to your Virtual Private Cloud (VPC).

### Key Attributes
Every ENI can be configured with the following properties:
- A **primary private IPv4 address** from the subnet's CIDR range.
- One or more secondary private IPv4 addresses.
- One **Elastic IP** (static public IP) per private IPv4 address (see: [[AWS IP Addressing]]).
- One ephemeral public IPv4 address.
- One or more **Security Groups** attached to it (note: firewalls are attached to the ENI, not the EC2 instance itself).
- A MAC address.

### The Architecture Trick: Secondary ENIs
While your primary network card (eth0) is glued to the server, you can create Secondary ENIs (eth1, eth2, etc.) that are completely independent. You can attach and detach these secondary network cards to running EC2 instances on the fly.

Here are the two main reasons you do this:

1. The "Management Network" (Multi-Homed Instances)
Imagine you have an EC2 web server in a public subnet facing the internet. But your company has a strict security policy: all administrators must SSH into servers using a completely separate, highly secure private subnet.

Solution: You attach two ENIs to the instance. ENI #1 is in the public subnet (handling web traffic). ENI #2 is in the private management subnet (handling SSH traffic). The server now has a foot in two different networks simultaneously.

2. Rapid Failover & MAC-tied Licensing (The Exam Favorite)
If Instance A crashes, you could spin up Instance B and update your DNS records or Route Tables to point to the new server. But that takes time.

Solution: You put all of your important network identity (the Elastic IP, the Private IP) onto a Secondary ENI. If Instance A crashes, you simply detach the Secondary ENI from the broken server and instantly plug it into Instance B.

Bonus: Because the ENI brings its MAC address with it, this is the only way to run expensive enterprise software (like legacy databases) whose licenses are permanently locked to a specific hardware MAC address!

### Default vs. Secondary ENIs
1. **Primary ENI (eth0):** Every EC2 instance is created with a default primary network interface. It is permanently attached and **cannot be detached** while the instance is alive.
2. **Secondary ENIs:** You can create and attach additional ENIs to an EC2 instance. Unlike the primary ENI, secondary ENIs are independent resources that **can be detached and attached to different instances on the fly**.

### Common Use Cases
- **Failover / High Availability:** If an active instance fails, you can detach its secondary ENI and quickly attach it to a standby instance. Since the ENI retains its IP addresses and MAC address, traffic is instantly rerouted to the standby machine without waiting for DNS propagation.
- **Management vs. Data Networks:** By attaching multiple ENIs, you can place an instance in multiple subnets simultaneously (e.g., `eth0` in a private subnet for secure backend traffic, and `eth1` in a public subnet for management/SSH traffic).
- **MAC-Bound Licensing:** Some legacy software relies on hardware MAC addresses for license verification. Because an ENI has a fixed MAC address that persists even if you move it between instances, it is highly useful for managing these licenses in the cloud.

> [!WARNING]
> An ENI acts as a physical network card localized to a specific data center. Therefore, an ENI is strictly bound to a single **Availability Zone (AZ)**. You cannot detach an ENI in AZ `us-east-1a` and attach it to an instance in `us-east-1b`.
