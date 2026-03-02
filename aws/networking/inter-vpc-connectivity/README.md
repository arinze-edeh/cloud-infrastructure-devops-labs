# 🔗 AWS VPC Peering — Cross-VPC Private Connectivity

> **Enterprise-grade private network connectivity between isolated AWS VPCs using VPC Peering, enabling secure inter-instance communication without traversing the public internet.**

---

![AWS](https://img.shields.io/badge/AWS-VPC%20Peering-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Region](https://img.shields.io/badge/Region-us--east--1-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen?style=for-the-badge)
![IaC](https://img.shields.io/badge/IaC-AWS%20CLI-0078D4?style=for-the-badge&logo=terminal&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Infrastructure Inventory](#infrastructure-inventory)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Phase 1 — Discover Existing Infrastructure](#phase-1--discover-existing-infrastructure)
  - [Phase 2 — Create VPC Peering Connection](#phase-2--create-vpc-peering-connection)
  - [Phase 3 — Configure Route Tables](#phase-3--configure-route-tables)
  - [Phase 4 — Configure Security Groups](#phase-4--configure-security-groups)
  - [Phase 5 — Configure SSH Access](#phase-5--configure-ssh-access)
  - [Phase 6 — Validate End-to-End Connectivity](#phase-6--validate-end-to-end-connectivity)
- [Resolution & Results](#resolution--results)
- [Troubleshooting](#troubleshooting)
- [Key Learnings](#key-learnings)
- [References](#references)

---

## Project Overview

This project demonstrates the configuration of **AWS VPC Peering** to establish private, low-latency network connectivity between a **Default Public VPC** and a **Private VPC** (`nautilus-private-vpc`) within the same AWS account and region. Traffic flows entirely within the AWS backbone — no internet gateway, NAT, or VPN required.

This pattern is foundational for multi-tier application architectures, microservices separation, and environment isolation (e.g., dev/staging/prod) where services must communicate privately and securely.

---

## Problem Statement

### Business Context

The Nautilus DevOps team required a solution to enable **private network communication** between two isolated VPC environments:

- A **public-facing VPC** hosting internet-accessible compute (`nautilus-public-ec2`)
- A **private VPC** (`nautilus-private-vpc`) hosting internal compute (`nautilus-private-ec2`) with no direct internet exposure

### Technical Problem

By default, **VPCs are completely isolated** from one another at the network layer. Without explicit peering and routing:

| Problem | Impact |
|---|---|
| No network path between VPCs | Services cannot communicate across VPC boundaries |
| No shared route table entries | Packets destined for the peer VPC are dropped |
| Default-deny security group rules | Even with peering, ICMP/TCP traffic is blocked |
| No SSH key trust between hosts | Cannot authenticate from the management host to EC2 |

The goal was to resolve all four blockers and **demonstrate end-to-end connectivity** by successfully pinging `nautilus-private-ec2` (10.1.1.197) from `nautilus-public-ec2` across the peering link.

---

## Architecture

### Network Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS us-east-1                            │
│                                                                 │
│  ┌──────────────────────────┐    VPC Peering     ┌──────────────────────────┐  │
│  │   Default VPC            │  pcx-0207198e601a5b32d  │   nautilus-private-vpc   │  │
│  │   172.31.0.0/16          │◄──────────────────►│   10.1.0.0/16            │  │
│  │                          │                    │                          │  │
│  │  ┌────────────────────┐  │                    │  ┌────────────────────┐  │  │
│  │  │ nautilus-public-ec2│  │                    │  │nautilus-private-ec2│  │  │
│  │  │ Public Subnet      │  │  ping 10.1.1.197   │  │nautilus-private-   │  │  │
│  │  │ (internet-facing)  │──┼────────────────────┼─►│subnet 10.1.1.0/24  │  │  │
│  │  └────────────────────┘  │                    │  └────────────────────┘  │  │
│  └──────────────────────────┘                    └──────────────────────────┘  │
│                                                                 │
│  aws-client host ──SSH──► nautilus-public-ec2 ──ping──► nautilus-private-ec2   │
└─────────────────────────────────────────────────────────────────┘
```

### Traffic Flow

```
aws-client (management host)
    │
    │ SSH (port 22) via /root/.ssh/id_rsa
    ▼
nautilus-public-ec2 [172.31.x.x]
    │
    │ ICMP via VPC Peering pcx-0207198e601a5b32d
    │ Route: 10.1.0.0/16 → pcx-*
    ▼
nautilus-private-ec2 [10.1.1.197]
    │
    │ Return traffic
    │ Route: 172.31.0.0/16 → pcx-*
    ▼
nautilus-public-ec2 (reply received)
```

---

## Infrastructure Inventory

| Resource | Name | Value |
|---|---|---|
| **Default VPC** | Default | `vpc-0880ec78a65acdceb` |
| **Default VPC CIDR** | — | `172.31.0.0/16` |
| **Private VPC** | `nautilus-private-vpc` | `vpc-0bea0bc81c2fb1851` |
| **Private VPC CIDR** | — | `10.1.0.0/16` |
| **Private Subnet** | `nautilus-private-subnet` | `10.1.1.0/24` |
| **Peering Connection** | `nautilus-vpc-peering` | `pcx-0207198e601a5b32d` |
| **Public EC2** | `nautilus-public-ec2` | Public IP: `100.55.64.9` |
| **Private EC2** | `nautilus-private-ec2` | Private IP: `10.1.1.197` |
| **Public EC2 SG** | — | `sg-0104b845d741f1ff2` |
| **Private EC2 SG** | — | `sg-072bab54b2b54f706` |
| **Region** | — | `us-east-1` |

---

## Prerequisites

### Required Access

- AWS Console or CLI access with the following IAM permissions:
  - `ec2:CreateVpcPeeringConnection`
  - `ec2:AcceptVpcPeeringConnection`
  - `ec2:CreateRoute`
  - `ec2:AuthorizeSecurityGroupIngress`
  - `ec2-instance-connect:SendSSHPublicKey`

### Required Tools

```bash
# Verify AWS CLI is configured
aws sts get-caller-identity

# Verify EC2 Instance Connect availability
aws ec2-instance-connect help
```

### Environment Setup

```bash
# Set core variables — run these first in every session
DEFAULT_VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

PRIVATE_VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=nautilus-private-vpc" \
  --query "Vpcs[0].VpcId" --output text)

DEFAULT_VPC_CIDR=$(aws ec2 describe-vpcs \
  --vpc-ids $DEFAULT_VPC_ID \
  --query "Vpcs[0].CidrBlock" --output text)

PRIVATE_CIDR="10.1.0.0/16"

echo "Default VPC:  $DEFAULT_VPC_ID ($DEFAULT_VPC_CIDR)"
echo "Private VPC:  $PRIVATE_VPC_ID ($PRIVATE_CIDR)"
```

---

## Implementation

### Phase 1 — Discover Existing Infrastructure

**Objective:** Baseline all existing resources before making any changes.

```bash
# Describe both VPCs side by side
aws ec2 describe-vpcs \
  --filters "Name=vpc-id,Values=$DEFAULT_VPC_ID,$PRIVATE_VPC_ID" \
  --query "Vpcs[*].{VpcId:VpcId,CIDR:CidrBlock,Default:IsDefault,Name:Tags[?Key=='Name'].Value|[0]}" \
  --output table

# List all EC2 instances with network context
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-public-ec2,nautilus-private-ec2" \
  --query "Reservations[*].Instances[*].{Name:Tags[?Key=='Name'].Value|[0],InstanceId:InstanceId,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress,VPC:VpcId,State:State.Name}" \
  --output table
```

***Screenshot: AWS Console → VPC Dashboard showing both VPCs with their CIDR ranges***

---

### Phase 2 — Create VPC Peering Connection

**Objective:** Establish the logical peering link between the two VPCs.

**Problem:** VPCs are network-isolated by default. No peering = no route = dropped packets.

#### Step 2.1 — Create the Peering Request

```bash
PEER_CONN_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $DEFAULT_VPC_ID \
  --peer-vpc-id $PRIVATE_VPC_ID \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=nautilus-vpc-peering}]' \
  --query "VpcPeeringConnection.VpcPeeringConnectionId" \
  --output text)

echo "Created Peering Connection: $PEER_CONN_ID"
```

**Expected output:**
```
Created Peering Connection: pcx-0207198e601a5b32d
```

#### Step 2.2 — Accept the Peering Connection

> ⚠️ **Critical:** Peering connections default to `pending-acceptance`. They must be explicitly accepted — even within the same account.

```bash
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEER_CONN_ID

# Confirm status is 'active'
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids $PEER_CONN_ID \
  --query "VpcPeeringConnections[0].Status"
```

**Expected output:**
```json
{
    "Code": "active",
    "Message": "Active"
}
```

***Screenshot: AWS Console → VPC → Peering Connections showing `nautilus-vpc-peering` with Status = Active***

---

### Phase 3 — Configure Route Tables

**Objective:** Add explicit routes in both VPCs so traffic knows to traverse the peering link.

**Problem:** VPC Peering creates a logical connection, but does **not** automatically add routes. Packets with no matching route are silently dropped.

#### Step 3.1 — Route in Default VPC (→ Private VPC)

```bash
DEFAULT_RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$DEFAULT_VPC_ID" \
  --query "RouteTables[0].RouteTableId" --output text)

aws ec2 create-route \
  --route-table-id $DEFAULT_RT_ID \
  --destination-cidr-block $PRIVATE_CIDR \
  --vpc-peering-connection-id $PEER_CONN_ID
```

**Expected output:**
```json
{ "Return": true }
```

#### Step 3.2 — Route in Private VPC (→ Default VPC)

```bash
PRIVATE_RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$PRIVATE_VPC_ID" \
  --query "RouteTables[0].RouteTableId" --output text)

aws ec2 create-route \
  --route-table-id $PRIVATE_RT_ID \
  --destination-cidr-block $DEFAULT_VPC_CIDR \
  --vpc-peering-connection-id $PEER_CONN_ID
```

**Expected output:**
```json
{ "Return": true }
```

***Screenshot: AWS Console → VPC → Route Tables → Default VPC route table showing the new 10.1.0.0/16 → pcx-* entry***

***Screenshot: AWS Console → VPC → Route Tables → Private VPC route table showing the new 172.31.0.0/16 → pcx-* entry***

---

### Phase 4 — Configure Security Groups

**Objective:** Open the minimum necessary ports to allow ICMP (ping) traffic from the public VPC to the private EC2.

**Problem:** AWS security groups are default-deny. Even with valid routes, all traffic is blocked unless explicitly permitted at the security group layer.

```bash
PRIVATE_SG_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-private-ec2" \
  --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" \
  --output text)

# Allow ALL ICMP from Default VPC CIDR
aws ec2 authorize-security-group-ingress \
  --group-id $PRIVATE_SG_ID \
  --protocol icmp \
  --port -1 \
  --cidr $DEFAULT_VPC_CIDR
```

**Expected output:**
```json
{
    "Return": true,
    "SecurityGroupRules": [{
        "IpProtocol": "icmp",
        "FromPort": -1,
        "ToPort": -1,
        "CidrIpv4": "172.31.0.0/16"
    }]
}
```

***Screenshot: AWS Console → EC2 → Security Groups → `sg-072bab54b2b54f706` Inbound Rules showing ICMP All from 172.31.0.0/16***

---

### Phase 5 — Configure SSH Access

**Objective:** Enable key-based SSH from the `aws-client` management host to `nautilus-public-ec2`.

**Problem:** The `ec2-user` account on `nautilus-public-ec2` had no trusted public keys. Direct `ssh-copy-id` failed because the instance rejected all authentication methods. Password auth is disabled on AWS EC2 by default (`Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`).

**Resolution:** Use **EC2 Instance Connect** to temporarily inject the public key (valid 60 seconds), then SSH in immediately using the corresponding private key to permanently append the key to `authorized_keys`.

#### Step 5.1 — Open SSH Port on Public EC2 Security Group

```bash
PUBLIC_SG_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-public-ec2" \
  --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $PUBLIC_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

#### Step 5.2 — Inject Key via EC2 Instance Connect

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-public-ec2" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

AZ=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text)

PUBLIC_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-public-ec2" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

# Inject key (60-second window)
aws ec2-instance-connect send-ssh-public-key \
  --region us-east-1 \
  --instance-id $INSTANCE_ID \
  --availability-zone $AZ \
  --instance-os-user ec2-user \
  --ssh-public-key file:///root/.ssh/id_rsa.pub
```

**Expected output:**
```json
{
    "RequestId": "b330fdd5-559b-4ed0-98e3-ec5f1877de7f",
    "Success": true
}
```

#### Step 5.3 — Permanently Append Key (within 60 seconds)

```bash
# Run IMMEDIATELY after send-ssh-public-key
ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no ec2-user@$PUBLIC_IP \
  "echo '$(cat /root/.ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys && \
   chmod 600 ~/.ssh/authorized_keys"
```

> 💡 **Why `echo '$(cat ...)'` instead of piping?** The subshell `$(cat ...)` is evaluated **locally on the aws-client** before the SSH session opens. This means the key content is passed as a string argument — nothing is left reading from stdin, which was the root cause of the hanging command in the earlier attempt.

***Screenshot: Terminal output showing `aws ec2-instance-connect send-ssh-public-key` returning `"Success": true`***

---

### Phase 6 — Validate End-to-End Connectivity

**Objective:** Prove the full chain works — SSH from `aws-client` → `nautilus-public-ec2` → ping `nautilus-private-ec2`.

```bash
PRIVATE_EC2_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-private-ec2" \
  --query "Reservations[0].Instances[0].PrivateIpAddress" \
  --output text)

echo "Target: $PRIVATE_EC2_IP"

# The definitive test
ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no ec2-user@$PUBLIC_IP \
  "ping -c 4 $PRIVATE_EC2_IP"
```

---

## Resolution & Results

### ✅ Final Output

```
PING 10.1.1.197 (10.1.1.197) 56(84) bytes of data.
64 bytes from 10.1.1.197: icmp_seq=1 ttl=127 time=2.48 ms
64 bytes from 10.1.1.197: icmp_seq=2 ttl=127 time=0.886 ms
64 bytes from 10.1.1.197: icmp_seq=3 ttl=127 time=1.62 ms
64 bytes from 10.1.1.197: icmp_seq=4 ttl=127 time=1.13 ms

--- 10.1.1.197 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3033ms
rtt min/avg/max/mdev = 0.886/1.529/2.481/0.610 ms
```

***Screenshot: Terminal showing successful 0% packet loss ping from nautilus-public-ec2 to 10.1.1.197***

### ✅ Completion Checklist

| Requirement | Resource | Status |
|---|---|---|
| VPC Peering Connection created | `nautilus-vpc-peering` (`pcx-0207198e601a5b32d`) | ✅ Active |
| Route: Default VPC → `10.1.0.0/16` | Default VPC main route table | ✅ Configured |
| Route: Private VPC → `172.31.0.0/16` | Private VPC main route table | ✅ Configured |
| ICMP allowed from `172.31.0.0/16` | `sg-072bab54b2b54f706` | ✅ Rule added |
| SSH port 22 open on public EC2 | `sg-0104b845d741f1ff2` | ✅ Rule added |
| Public key in `authorized_keys` | `nautilus-public-ec2` / ec2-user | ✅ Appended |
| SSH from `aws-client` → public EC2 | Key-based auth | ✅ Working |
| Ping from public EC2 → private EC2 | ICMP across peering | ✅ **0% packet loss** |

---

## Troubleshooting

### Issue 1: `Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`

**Root Cause:** The target EC2 instance has no record of the connecting host's public key. EC2 instances disable password authentication by default and reject any connection without a pre-trusted key.

**Resolution:**
```bash
# Use EC2 Instance Connect to temporarily inject the key
aws ec2-instance-connect send-ssh-public-key \
  --instance-id $INSTANCE_ID \
  --availability-zone $AZ \
  --instance-os-user ec2-user \
  --ssh-public-key file:///root/.ssh/id_rsa.pub

# Immediately SSH in using the matching private key
ssh -i /root/.ssh/id_rsa ec2-user@$PUBLIC_IP "echo '$(cat /root/.ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys"
```

---

### Issue 2: SSH command hangs indefinitely (`cat >> ~/.ssh/authorized_keys`)

**Root Cause:** The pipe `cat /root/.ssh/id_rsa.pub | ssh ... "cat >> ..."` causes the remote `cat` to block waiting for EOF on stdin. The SSH session never terminates because no EOF is sent.

**Resolution:** Use shell substitution to embed the key as a string before the SSH session opens:
```bash
# ❌ Wrong — remote cat reads from stdin, hangs forever
cat /root/.ssh/id_rsa.pub | ssh ec2-user@$IP "cat >> ~/.ssh/authorized_keys"

# ✅ Correct — key is interpolated locally before SSH executes
ssh -i /root/.ssh/id_rsa ec2-user@$IP \
  "echo '$(cat /root/.ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys"
```

---

### Issue 3: `ssh: connect to host X port 22: Connection timed out`

**Root Cause:** The public EC2's security group had no inbound rule allowing TCP port 22.

**Resolution:**
```bash
aws ec2 authorize-security-group-ingress \
  --group-id $PUBLIC_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

---

### Issue 4: `RouteAlreadyExists` on `create-route`

**Root Cause:** Commands were re-run after the routes were already successfully created in the first pass. Not an error — infrastructure state is correct.

**Resolution:** This is informational. Verify the route exists and proceed:
```bash
aws ec2 describe-route-tables \
  --route-table-ids $DEFAULT_RT_ID \
  --query "RouteTables[0].Routes[?DestinationCidrBlock=='10.1.0.0/16']"
```

---

### Issue 5: Ping fails despite all configuration appearing correct

**Root Cause candidates:** Route table is associated with the wrong subnet, or the EC2 instance is in a non-main route table.

**Resolution:** Find all route tables in the private VPC and verify subnet association:
```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$PRIVATE_VPC_ID" \
  --query "RouteTables[*].{RTB:RouteTableId,Subnet:Associations[0].SubnetId,Main:Associations[0].Main}" \
  --output table
```

---

## Key Learnings

### 1. VPC Peering Requires Three Independent Layers
A common misconception is that creating a peering connection enables traffic flow. In reality, **all three layers must be configured independently**:

```
Layer 1: VPC Peering Connection (logical link — must be accepted)
Layer 2: Route Tables (both VPCs need routes pointing to pcx-*)  
Layer 3: Security Groups (default-deny — ICMP/TCP must be explicitly allowed)
```

Failure at any single layer silently drops traffic, making diagnosis non-obvious.

### 2. EC2 Instance Connect as a Bootstrap Mechanism
When an EC2 instance has no trusted keys and password auth is disabled, EC2 Instance Connect provides a **60-second injection window** that can be used to bootstrap permanent key trust — without requiring console access or a bastion host.

### 3. Shell Substitution vs. Piping for Remote Commands
`echo '$(cat file)'` evaluates the subshell **on the local machine** and passes the result as a string argument to the remote shell. This is fundamentally different from piping, where the remote process reads from stdin and may block indefinitely waiting for EOF.

### 4. `RouteAlreadyExists` is a Success Indicator
When re-running infrastructure scripts idempotently, `RouteAlreadyExists` and `InvalidPermission.Duplicate` errors confirm that the desired state was already achieved in a prior run — not that something went wrong.

---

## References

- [AWS VPC Peering Documentation](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
- [EC2 Instance Connect — AWS Docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Connect-using-EC2-Instance-Connect.html)
- [VPC Route Tables — AWS Docs](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [Security Groups for EC2 — AWS Docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html)
- [AWS CLI EC2 Reference](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/index.html)

---

## Author

**Nautilus DevOps Team**
*Infrastructure & Cloud Engineering*

---

> *This implementation was executed entirely via AWS CLI following infrastructure-as-code principles. All commands are idempotent-safe and reproducible.*


<img width="1040" height="472" alt="image" src="https://github.com/user-attachments/assets/84fb89e1-a9b2-4b51-b414-83b8031416da" />
<img width="1035" height="521" alt="image" src="https://github.com/user-attachments/assets/a7be414a-06bd-4ede-84cb-7fd4ab887ca2" />
<img width="1033" height="627" alt="image" src="https://github.com/user-attachments/assets/f9d41f77-167b-433d-a0e9-2e4ffce9f1b6" />
<img width="1037" height="621" alt="image" src="https://github.com/user-attachments/assets/ecdc3e30-bdf1-415f-a3aa-5d7d1f2f666d" />
<img width="1035" height="871" alt="image" src="https://github.com/user-attachments/assets/8bfdc5c3-f65c-4f25-abdd-6a14942c8600" />
<img width="1037" height="492" alt="image" src="https://github.com/user-attachments/assets/8d8ab7b2-8df5-4f9e-9140-651f26fba9af" />
<img width="1036" height="277" alt="image" src="https://github.com/user-attachments/assets/c1d1e11a-944e-4618-9559-1d7a5d03859d" />
<img width="1031" height="348" alt="image" src="https://github.com/user-attachments/assets/71218b04-9cdc-40e4-ae7f-3ce4a69f1530" />
<img width="1031" height="511" alt="image" src="https://github.com/user-attachments/assets/ad47f237-a861-4b53-bf28-ed0ddeab0d1a" />
<img width="1029" height="358" alt="image" src="https://github.com/user-attachments/assets/839ad23b-a1e7-41df-88b9-2eec8422d536" />
<img width="1035" height="542" alt="image" src="https://github.com/user-attachments/assets/f7ef38ec-e82a-4323-85fb-52e454f0e6c2" />
<img width="1020" height="858" alt="image" src="https://github.com/user-attachments/assets/781df65a-158d-44f6-b27a-4febe30118b1" />
<img width="1035" height="632" alt="image" src="https://github.com/user-attachments/assets/9e6a2443-6124-4e2b-9070-af207846255d" />
<img width="1036" height="504" alt="image" src="https://github.com/user-attachments/assets/084c518c-19dc-44d7-a16d-1e0a4da230b2" />
<img width="1035" height="545" alt="image" src="https://github.com/user-attachments/assets/553bf1e6-2eed-46da-9b47-ce005ce7f0d3" />
<img width="1037" height="562" alt="image" src="https://github.com/user-attachments/assets/a4c191e8-2695-47ec-91c9-50b2e6e6a5b9" />
<img width="1041" height="452" alt="image" src="https://github.com/user-attachments/assets/f132fcc8-9056-4879-bbb3-1fea6cd924fe" />
<img width="1035" height="443" alt="image" src="https://github.com/user-attachments/assets/95689b8c-6cbb-4818-ac18-d5e343646b5e" />
<img width="1039" height="303" alt="image" src="https://github.com/user-attachments/assets/62588c43-1f7e-4a72-af13-b5b9b3aaf8fd" />
<img width="1037" height="400" alt="image" src="https://github.com/user-attachments/assets/dd878ac7-ed0b-4ee8-8ab4-78ec849f64f2" />
<img width="1034" height="439" alt="image" src="https://github.com/user-attachments/assets/951a5b9e-1cde-4753-b282-af595c8ca6ee" />
<img width="1029" height="674" alt="image" src="https://github.com/user-attachments/assets/5d74d6d7-b96b-4459-aca5-cd0794daaa6b" />



