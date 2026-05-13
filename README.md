# AWS Networking Lab — Multi-VPC Architecture

# Student name: Maxim Raikinakh

> **Project 1 — Full Step-by-Step Guide**

A hands-on AWS networking lab where you build a complete multi-VPC enterprise network from scratch — including Transit Gateway, NAT Gateway, EC2 Connect Endpoints, and Transit Gateway Peering.

---

## Overview

This lab teaches you how to design and build a real AWS network the way large companies do it in production. You will create multiple isolated networks (VPCs), connect them using a Transit Gateway, provide internet access through a NAT Gateway, install a web server on private servers, and finally extend the architecture using **Transit Gateway Peering**.

---

## Architecture Summary

| Component | Details |
|---|---|
| VPCs | 3 VPCs: `10.1.0.0/16`, `10.2.0.0/16`, `10.3.0.0/16` |
| Subnets | 4 subnets per VPC (2 regular + 2 TGW Attachment subnets) |
| Transit Gateway | 1 TGW connecting all 3 VPCs |
| Ubuntu Servers | 2 servers — one in VPC-1, one in VPC-2 |
| EC2 Connect Endpoints | 2 endpoints for secure private access |
| NAT Gateway | 1 NAT Gateway in VPC-3 (shared internet access) |
| Apache2 | Installed on both servers to prove internet connectivity |
| TGW Peering | 1 additional VPC (VPC-4) + TGW-2, peered with the original TGW |

---

## Lab Stages

### Stage 1 — Create 3 VPCs

A VPC (Virtual Private Cloud) is your own private network inside AWS. Each VPC must have a unique, non-overlapping CIDR block.

**IP Allocation Plan:**

| VPC | CIDR Block | First IP | Last IP |
|---|---|---|---|
| VPC-1 | 10.1.0.0/16 | 10.1.0.1 | 10.1.255.254 |
| VPC-2 | 10.2.0.0/16 | 10.2.0.1 | 10.2.255.254 |
| VPC-3 | 10.3.0.0/16 | 10.3.0.1 | 10.3.255.254 |

> 💡 VPC-1 and VPC-2 represent different teams/apps. VPC-3 is the shared services VPC (NAT Gateway).

**Review Questions:**
- Q1: What is a VPC and why do we need one?
- Q2: Why must each VPC have a different CIDR block?
- Q3: What happens if two VPCs have overlapping IP ranges?
- Q4: What is the difference between a public and private subnet?

---

### Stage 2 — Create 4 Subnets in Each VPC

Each VPC gets 4 subnets across 2 Availability Zones:

- **Subnet A** (e.g. `10.1.0.0/20`) — regular subnet in AZ-A, for servers
- **Subnet B** (e.g. `10.1.16.0/20`) — regular subnet in AZ-B, for servers
- **TGW Attachment Subnet A** (e.g. `10.1.32.0/28`) — dedicated for Transit Gateway in AZ-A
- **TGW Attachment Subnet B** (e.g. `10.1.32.16/28`) — dedicated for Transit Gateway in AZ-B

> 💡 TGW Attachment Subnets are `/28` (only 11 usable IPs) because they only need a few IPs for the TGW connection — no servers live there.

**Review Questions:**
- Q1: What is the difference between a /20 and /28 subnet?
- Q2: Why do we use /28 for TGW Attachment subnets specifically?
- Q3: What is an Availability Zone and why does it matter?
- Q4: How many usable IP addresses does a /28 subnet have?

---

### Stage 3 — Create a Transit Gateway (TGW)

A Transit Gateway is a central hub that connects multiple VPCs. Without it, you'd need a direct VPC Peering connection between every pair of VPCs.

| Feature | VPC Peering | Transit Gateway |
|---|---|---|
| Connections needed | N × (N-1) / 2 | Just 1 TGW |
| Scales well | No | Yes |
| Central management | No | Yes |
| Cost | Lower for 2 VPCs | Better for 3+ VPCs |

> 💡 After creating the TGW and attachments, you **must** also update the route tables in each VPC — without routes, traffic goes nowhere.

**Review Questions:**
- Q1: What is a Transit Gateway and what problem does it solve?
- Q2: Why do we attach the TGW to the TGW Attachment subnets specifically?
- Q3: After creating the TGW attachment, what else must you configure for traffic to flow?
- Q4: How many connections would you need with VPC Peering to connect 5 VPCs to each other?

---

### Stage 4 — Create EC2 Connect Endpoints

EC2 Connect Endpoints let you SSH into servers that have **no public IP** — keeping them private and secure. You only need one EC2 Connect Endpoint per VPC to access any server in that VPC.

**Review Questions:**
- Q1: Why do we use EC2 Connect Endpoints instead of giving servers public IPs?
- Q2: Can one EC2 Connect Endpoint be used to connect to multiple servers in the same VPC?
- Q3: What security benefit does removing public IPs from servers provide?
- Q4: In which subnet do we place the EC2 Connect Endpoint?

---

### Stage 5 — Deploy Ubuntu Servers

Ubuntu servers are placed in the **regular subnets** (not TGW Attachment subnets). TGW Attachment subnets are reserved for TGW connections only.

**Review Questions:**
- Q1: Why are the Ubuntu servers placed in the regular subnets and not the TGW Attachment subnets?
- Q2: The servers have no public IP — how will they eventually get internet access?
- Q3: What instance type is used and why is it sufficient for this lab?

---

### Stage 6 — Ping Test Between VPCs

A successful ping proves:
- The Transit Gateway is correctly configured
- The route tables are correctly set up
- The Security Groups allow ICMP traffic between the servers

**Routes needed for VPC-1 ↔ VPC-2 traffic:**
- VPC-1 subnet route table → `10.2.0.0/16` via TGW
- VPC-2 subnet route table → `10.1.0.0/16` via TGW
- TGW route table → knows about both VPCs

> 💡 Think of route tables as GPS directions — without them, traffic has no idea where to go even if the road (TGW) exists.

**Review Questions:**
- Q1: What does a successful ping test prove about your network configuration?
- Q2: What 3 things must be correctly configured for a ping to succeed between VPCs?
- Q3: If the ping fails, what would you check first?
- Q4: What protocol does ping use and why must the Security Group allow it?

---

### Stage 7 — Set Up NAT Gateway in VPC-3

A NAT Gateway allows servers in private subnets to reach the internet — without exposing them to inbound internet traffic.

**Traffic flow through the NAT Gateway:**
1. Server in VPC-1 wants to reach the internet (e.g. install Apache2)
2. Traffic goes through TGW to VPC-3
3. VPC-3 route table sends traffic to the NAT Gateway
4. NAT Gateway translates the private IP to its own public IP
5. Request goes out to the internet; response comes back
6. NAT Gateway sends the response back to the original server

> 💡 NAT Gateway is **one-way** — it allows outbound traffic only. Servers behind it are still protected from inbound internet traffic.

**Review Questions:**
- Q1: What is the difference between an Internet Gateway and a NAT Gateway?
- Q2: Why is the NAT Gateway placed in VPC-3 and not in VPC-1 or VPC-2?
- Q3: What route must be added in VPC-1 and VPC-2 to use the NAT Gateway in VPC-3?
- Q4: Can traffic from the internet initiate a connection to a server behind a NAT Gateway?

---

### Stage 8 — Install Apache2

Once internet access is confirmed (`ping 8.8.8.8` succeeds), run:

```bash
sudo apt update && sudo apt install apache2 -y
```

Installing Apache2 proves two things:
- Internet access works (packages can only be downloaded if the server can reach the internet)
- The servers are now functional web servers — a real production use case

**Review Questions:**
- Q1: Why does installing Apache2 prove that internet access works?
- Q2: What command verifies that Apache2 is running after installation?
- Q3: What port does Apache2 listen on by default?
- Q4: Why do we test `ping 8.8.8.8` before installing Apache2?

---

### Stage 9 — Transit Gateway Peering

Transit Gateway Peering connects two separate Transit Gateways together — either in the same AWS region or across different regions. In this stage you create a brand new VPC and a brand new TGW, then peer them with your existing TGW so traffic can flow across all networks.

#### Step 1 — Create a New VPC (VPC-4)

Create a fourth VPC representing a separate isolated network (e.g. a different department or partner environment). Use a non-overlapping CIDR block:

- Go to **VPC → Your VPCs → Create VPC**
- Name: `VPC-4` | CIDR: `10.4.0.0/16`
- Create at least 2 subnets: one regular subnet (`10.4.0.0/20`) and one TGW Attachment subnet (`10.4.32.0/28`)

#### Step 2 — Create a Second Transit Gateway (TGW-2)

- Go to **VPC → Transit Gateways → Create Transit Gateway**
- Name: `TGW-2` | Leave all other settings as default
- Wait for the state to become **Available** before proceeding

#### Step 3 — Attach VPC-4 to TGW-2

- Go to **VPC → Transit Gateway Attachments → Create Transit Gateway Attachment**
- Transit Gateway ID: `TGW-2` | Attachment Type: `VPC` | VPC: `VPC-4`
- Select the TGW Attachment subnet in VPC-4

#### Step 4 — Create the TGW Peering Attachment

This is the core step — creating the peering link between TGW-1 and TGW-2:

- Go to **VPC → Transit Gateway Attachments → Create Transit Gateway Attachment**
- Transit Gateway ID: `TGW-1` | Attachment Type: `Peering Connection`
- Region: same region | Peer Transit Gateway ID: `TGW-2` | Click Create
- The attachment will show **Pending Acceptance** — you must accept it from TGW-2

#### Step 5 — Accept the Peering Request on TGW-2

Like VPC Peering, TGW Peering requires the other side to accept before it becomes active:

- Go to **Transit Gateway Attachments** and find the attachment with state `Pending Acceptance`
- Select it → **Actions → Accept Transit Gateway Peering Attachment**
- Wait for the state to change to **Available**

#### Step 6 — Update TGW Route Tables on Both Sides

After the peering is active, traffic still won't flow — you must add static routes manually:

- **TGW-1 route table:** add a static route for `10.4.0.0/16` → peering attachment
- **TGW-2 route table:** add static routes for `10.1.0.0/16`, `10.2.0.0/16`, `10.3.0.0/16` → peering attachment
- **VPC-4 subnet route table:** add routes for `10.1.0.0/16`, `10.2.0.0/16`, `10.3.0.0/16` via TGW-2

#### Step 7 — Deploy a Server in VPC-4 and Test Connectivity

Deploy an Ubuntu server in VPC-4 (same as Stage 5) and verify end-to-end connectivity:

- Create an EC2 Connect Endpoint in VPC-4 for secure access
- From the VPC-4 server, ping the private IP of the VPC-1 server (`10.1.x.x`) — a successful reply confirms TGW peering is fully working
- Verify Security Groups on both servers allow ICMP from the opposite VPC CIDR

**Review Questions:**
- Q1: What is the difference between TGW Peering and regular TGW attachments?
- Q2: Why does TGW Peering require manual static routes instead of automatic propagation?
- Q3: What happens if you forget to add routes on one side of the peering but not the other?
- Q4: In what real-world scenario would you use TGW Peering instead of just adding more VPCs to the same TGW?

---

## Final Summary

### What You Built

| Component | Purpose |
|---|---|
| 3 VPCs | Isolated networks for different teams/environments |
| 12 Subnets | Network segmentation inside each VPC |
| Transit Gateway | Central hub connecting all VPCs |
| EC2 Connect Endpoints | Secure access to private servers |
| Ubuntu Servers | Application servers in private subnets |
| NAT Gateway | Shared internet access for all VPCs |
| Apache2 | Web server proving the architecture works |
| TGW Peering | Extending the network across a second TGW and VPC |

### Key Lessons Learned

- VPCs are isolated by default — you must explicitly connect them
- Route tables are the GPS of AWS — without routes, traffic goes nowhere
- Transit Gateway scales better than VPC Peering for multiple VPCs
- NAT Gateway allows outbound internet access without exposing servers
- EC2 Connect Endpoints allow secure access without public IPs
- Availability Zones provide high availability and fault tolerance
- TGW Peering requires manual static routes — there is no automatic propagation across peered TGWs

### Tips for Next Time

- Always plan your IP ranges before creating VPCs — overlapping CIDRs cause major problems
- After creating TGW attachments, don't forget to update route tables — this is the most common mistake
- Test connectivity step by step: ping within VPC first, then between VPCs, then to internet
- Security Groups must allow ICMP for ping tests to work
- NAT Gateway needs to be in a public subnet with an Internet Gateway
- For TGW Peering, add static routes on **both** TGW route tables and the VPC subnet route table

---

**Great work completing this lab!** 💪
