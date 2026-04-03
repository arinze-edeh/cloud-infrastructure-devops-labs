# AWS VPC Peering - Cross-VPC Private Connectivity

> **Enterprise-grade private network connectivity between isolated AWS VPCs using VPC Peering, enabling secure inter-instance communication without traversing the public internet.**

---

![AWS](https://img.shields.io/badge/AWS-VPC%20Peering-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Region](https://img.shields.io/badge/Region-us--east--1-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![IaC](https://img.shields.io/badge/IaC-AWS%20CLI-0078D4?style=for-the-badge&logo=terminal&logoColor=white)


---

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Infrastructure Inventory](#infrastructure-inventory)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Phase 1; Initialize Variables](#phase-1--initialize-variables)
  - [Phase 2: Create & Accept VPC Peering Connection](#phase-2--create--accept-vpc-peering-connection)
  - [Phase 3: Configure Route Tables](#phase-3--configure-route-tables)
  - [Phase 4: Configure Security Groups](#phase-4--configure-security-groups)
  - [Phase 5: Configure SSH Access via EC2 Instance Connect](#phase-5--configure-ssh-access-via-ec2-instance-connect)
  - [Phase 6: Validate End-to-End Connectivity](#phase-6--validate-end-to-end-connectivity)
- [Resolution & Results](#resolution--results)
- [Troubleshooting](#troubleshooting)
- [Key Learnings](#key-learnings)
- [References](#references)

---

## Project Overview

This project demonstrates the configuration of **AWS VPC Peering** to establish private, low-latency network connectivity between a **Default Public VPC** (`172.31.0.0/16`) and a **Private VPC** (`nautilus-private-vpc` - `10.1.0.0/16`) within the same AWS account and region (`us-east-1`). Traffic flows entirely within the AWS backbone - no internet gateway, NAT gateway, or VPN required.

This pattern is foundational for multi-tier application architectures, microservices separation, and environment isolation (e.g., dev/staging/prod) where services must communicate privately and securely.

---

## Problem Statement

### Business Context

The Nautilus DevOps team required a solution to enable **private network communication** between two isolated VPC environments:

- A **public-facing VPC** (Default VPC) hosting internet-accessible compute (`nautilus-public-ec2`)
- A **private VPC** (`nautilus-private-vpc`) hosting internal compute (`nautilus-private-ec2`) with no direct internet exposure

### Technical Problem

By default, **VPCs are completely isolated** from one another at the network layer. Without explicit configuration across three independent layers, all cross-VPC traffic is dropped:

| Layer | Problem | Symptom |
|---|---|---|
| **VPC Peering** | No logical link between VPCs | No network path exists |
| **Route Tables** | No routes pointing to peer VPC | Packets silently dropped |
| **Security Groups** | Default-deny on all inbound traffic | ICMP/TCP blocked even with valid route |
| **SSH Key Trust** | No public key on target EC2 | `Permission denied (publickey)` |

**Goal:** Resolve all four blockers and demonstrate end-to-end connectivity by successfully pinging `nautilus-private-ec2` (`10.1.1.197`) from `nautilus-public-ec2` across the peering link - with **0% packet loss**.

---

## Architecture

### Network Topology

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            AWS  us-east-1                                │
│                                                                          │
│   ┌─────────────────────────┐  pcx-0207198e601a5b32d  ┌──────────────────────────┐  │
│   │     Default VPC          │◄────────────────────────►│  nautilus-private-vpc    │  │
│   │     172.31.0.0/16        │    nautilus-vpc-peering  │  10.1.0.0/16             │  │
│   │                          │                          │                          │  │
│   │  ┌──────────────────┐   │                          │  ┌──────────────────┐   │  │
│   │  │nautilus-public-ec2│  │   ping 10.1.1.197 ──────►│  │nautilus-private- │   │  │
│   │  │100.55.64.9 (pub)  │──┼──────────────────────────┼─►│ec2  10.1.1.197   │   │  │
│   │  │sg-0104b845d741f1  │  │                          │  │sg-072bab54b2b54f │   │  │
│   │  └──────────────────┘  │                          │  └──────────────────┘   │  │
│   └─────────────────────────┘                          └──────────────────────────┘  │
│                                                                          │
│   aws-client ──SSH(/root/.ssh/id_rsa)──► public-ec2 ──ICMP──► private-ec2           │
└──────────────────────────────────────────────────────────────────────────┘
```

### Traffic Flow

```
aws-client (management host)
    │
    │  SSH port 22  ·  key: /root/.ssh/id_rsa
    ▼
nautilus-public-ec2  [172.31.x.x private / 100.55.64.9 public]
    │
    │  ICMP via VPC Peering  ·  pcx-0207198e601a5b32d
    │  Route: 10.1.0.0/16 → pcx-0207198e601a5b32d
    ▼
nautilus-private-ec2  [10.1.1.197]
    │
    │  Return ICMP  ·  Route: 172.31.0.0/16 → pcx-0207198e601a5b32d
    ▼
nautilus-public-ec2  (reply received · 0% packet loss)
```

---

## Infrastructure Inventory

| Resource | Name / Tag | ID / Value |
|---|---|---|
| **Default VPC** | Default | `vpc-0880ec78a65acdceb` |
| **Default VPC CIDR** | — | `172.31.0.0/16` |
| **Private VPC** | `nautilus-private-vpc` | `vpc-0bea0bc81c2fb1851` |
| **Private VPC CIDR** | — | `10.1.0.0/16` |
| **Private Subnet** | `nautilus-private-subnet` | `10.1.1.0/24` |
| **Peering Connection** | `nautilus-vpc-peering` | `pcx-0207198e601a5b32d` |
| **Public EC2** | `nautilus-public-ec2` | Public IP: `100.55.64.9` |
| **Private EC2** | `nautilus-private-ec2` | Private IP: `10.1.1.197` |
| **Public EC2 Security Group** | — | `sg-0104b845d741f1ff2` |
| **Private EC2 Security Group** | — | `sg-072bab54b2b54f706` |
| **ICMP Inbound Rule** | — | `sgr-05dc8286a75a49a87` |
| **SSH Inbound Rule** | — | `sgr-0f4a96f7339598caa` |
| **Region** | — | `us-east-1` |
| **Account ID** | — | `115244785922` |

---

## Prerequisites

### Required IAM Permissions

```
ec2:DescribeVpcs
ec2:CreateVpcPeeringConnection
ec2:AcceptVpcPeeringConnection
ec2:DescribeRouteTables
ec2:CreateRoute
ec2:DescribeInstances
ec2:DescribeSecurityGroups
ec2:AuthorizeSecurityGroupIngress
ec2-instance-connect:SendSSHPublicKey
```

### Required Tools

```bash
# Verify AWS CLI is configured and credentials are active
aws sts get-caller-identity

# Verify EC2 Instance Connect is available
aws ec2-instance-connect help
```

### SSH Key Pair

```bash
# Verify the key pair exists on the management host before starting
ls -la /root/.ssh/id_rsa
ls -la /root/.ssh/id_rsa.pub
```

---

## Implementation

### Phase 1: Initialize Variables

**Objective:** Resolve all resource IDs upfront and store them in shell variables. Every subsequent command references these variables, no hardcoded IDs.

```bash
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
```

**Verify resolution:**

```bash
echo "Default VPC : $DEFAULT_VPC_ID  ($DEFAULT_VPC_CIDR)"
echo "Private VPC : $PRIVATE_VPC_ID  ($PRIVATE_CIDR)"
```

***Screenshot: Terminal output confirming both VPC IDs and CIDRs resolved correctly***
<img width="1035" height="521" alt="image" src="https://github.com/user-attachments/assets/a7be414a-06bd-4ede-84cb-7fd4ab887ca2" />

---

### Phase 2: Create & Accept VPC Peering Connection

**Objective:** Establish the logical peering link between the two VPCs and transition its status to `active`.

**Problem:** VPCs are network-isolated by default. Without a peering connection, no network path exists between them regardless of any routing configuration.

#### Step 2.1: Create the Peering Request

```bash
PEER_CONN_ID=$(aws ec2 create-vpc-peering-connection \
    --vpc-id $DEFAULT_VPC_ID \
    --peer-vpc-id $PRIVATE_VPC_ID \
    --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=nautilus-vpc-peering}]' \
    --query "VpcPeeringConnection.VpcPeeringConnectionId" --output text)

echo "Created Peering Connection: $PEER_CONN_ID"
```

**Actual output:**

```
Created Peering Connection: pcx-0207198e601a5b32d
```

#### Step 2.2: Accept the Peering Connection

> ⚠️ **Critical:** A newly created peering connection enters `pending-acceptance` state. It must be explicitly accepted — even in the same account and region — before any traffic can flow.

```bash
aws ec2 accept-vpc-peering-connection \
    --vpc-peering-connection-id $PEER_CONN_ID
```

**Actual output (key fields):**

```json
{
    "VpcPeeringConnection": {
        "AccepterVpcInfo": {
            "CidrBlock": "10.1.0.0/16",
            "VpcId": "vpc-0bea0bc81c2fb1851"
        },
        "RequesterVpcInfo": {
            "CidrBlock": "172.31.0.0/16",
            "VpcId": "vpc-0880ec78a65acdceb"
        },
        "Status": {
            "Code": "active",
            "Message": "Active"
        },
        "VpcPeeringConnectionId": "pcx-0207198e601a5b32d"
    }
}
```

***Screenshots: AWS Console → VPC → Peering Connections showing `nautilus-vpc-peering` with Status = `Active`***

<img width="1037" height="621" alt="image" src="https://github.com/user-attachments/assets/ecdc3e30-bdf1-415f-a3aa-5d7d1f2f666d" />
<img width="1035" height="871" alt="image" src="https://github.com/user-attachments/assets/8bfdc5c3-f65c-4f25-abdd-6a14942c8600" />

---

### Phase 3: Configure Route Tables

**Objective:** Add explicit routes in both VPCs so traffic destined for the peer CIDR is forwarded through the peering connection rather than dropped.

**Problem:** VPC Peering creates a logical link but does **not** automatically populate route tables. Without matching routes on both sides, packets are silently dropped — ping would fail even with an active peering connection.

#### Step 3.1: Route in Default VPC → `10.1.0.0/16`

```bash
DEFAULT_RT_ID=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$DEFAULT_VPC_ID" \
    --query "RouteTables[0].RouteTableId" --output text)

aws ec2 create-route \
    --route-table-id $DEFAULT_RT_ID \
    --destination-cidr-block $PRIVATE_CIDR \
    --vpc-peering-connection-id $PEER_CONN_ID
```

**Actual output:**

```json
{ "Return": true }
```

#### Step 3.2: Route in Private VPC → `172.31.0.0/16`

```bash
PRIVATE_RT_ID=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$PRIVATE_VPC_ID" \
    --query "RouteTables[0].RouteTableId" --output text)

aws ec2 create-route \
    --route-table-id $PRIVATE_RT_ID \
    --destination-cidr-block $DEFAULT_VPC_CIDR \
    --vpc-peering-connection-id $PEER_CONN_ID
```

**Actual output:**

```json
{ "Return": true }
```

***Screenshot: AWS Console → VPC → Route Tables → Default VPC main RTB showing entry `10.1.0.0/16 → pcx-0207198e601a5b32d`***

<img width="1036" height="277" alt="image" src="https://github.com/user-attachments/assets/c1d1e11a-944e-4618-9559-1d7a5d03859d" />

***Screenshot: AWS Console → VPC → Route Tables → Private VPC main RTB showing entry `172.31.0.0/16 → pcx-0207198e601a5b32d`***

<img width="1031" height="511" alt="image" src="https://github.com/user-attachments/assets/ad47f237-a861-4b53-bf28-ed0ddeab0d1a" />

---

### Phase 4: Configure Security Groups

**Objective:** Open ICMP on the private EC2's security group to allow ping traffic sourced from the Default VPC CIDR.

**Problem:** AWS security groups are **default-deny**. Even with a valid peering connection and correct routes, all inbound traffic to `nautilus-private-ec2` is blocked at the instance level until an explicit allow rule exists.

```bash
PRIVATE_SG_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=nautilus-private-ec2" \
    --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)

aws ec2 authorize-security-group-ingress \
    --group-id $PRIVATE_SG_ID \
    --protocol icmp \
    --port -1 \
    --cidr $DEFAULT_VPC_CIDR
```

**Actual output:**

```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-05dc8286a75a49a87",
            "GroupId": "sg-072bab54b2b54f706",
            "IsEgress": false,
            "IpProtocol": "icmp",
            "FromPort": -1,
            "ToPort": -1,
            "CidrIpv4": "172.31.0.0/16"
        }
    ]
}
```

***Screenshot: AWS Console → EC2 → Security Groups → `sg-072bab54b2b54f706` → Inbound Rules tab showing `All ICMP - IPv4` from source `172.31.0.0/16`***

<img width="1035" height="542" alt="image" src="https://github.com/user-attachments/assets/f7ef38ec-e82a-4323-85fb-52e454f0e6c2" />

---

### Phase 5: Configure SSH Access via EC2 Instance Connect

**Objective:** Establish key-based SSH trust between the `aws-client` management host and `nautilus-public-ec2` so commands can be executed remotely.

**Problem:** This phase encountered three sequential blockers before resolution.

---

#### Blocker 1: TCP Port 22 Not Open

**Symptom:**
```
ssh: connect to host 100.55.64.9 port 22: Connection timed out
```

**Root Cause:** The public EC2 security group (`sg-0104b845d741f1ff2`) had no inbound rule for TCP port 22. The SYN packet was dropped at the security group layer, the instance was unreachable.

**Resolution:**

```bash
PUBLIC_SG_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=nautilus-public-ec2" \
    --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)

aws ec2 authorize-security-group-ingress \
    --group-id $PUBLIC_SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0
```

**Actual output:**

```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0f4a96f7339598caa",
            "GroupId": "sg-0104b845d741f1ff2",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
```
***Screenshot:***
<img width="1035" height="545" alt="image" src="https://github.com/user-attachments/assets/553bf1e6-2eed-46da-9b47-ce005ce7f0d3" />

---

#### Blocker 2: No Trusted Public Key on Instance

**Symptom:**
```
ec2-user@100.55.64.9: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

**Root Cause:** Port 22 was now reachable, but the instance had no record of `aws-client`'s public key in `~/.ssh/authorized_keys`. EC2 Amazon Linux 2 instances disable password authentication by default — without a pre-trusted key, all authentication methods fail.

**Resolution: EC2 Instance Connect bootstrap pattern:**

EC2 Instance Connect injects the provided public key into the instance's metadata for a **60-second window**. That window is used to SSH in with the matching private key and permanently append the key to `authorized_keys`.

```bash
# Resolve instance metadata
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-public-ec2" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

AZ=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text)

PUBLIC_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-public-ec2" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

# Step 1 — Inject public key (60-second window opens)
aws ec2-instance-connect send-ssh-public-key \
  --region us-east-1 \
  --instance-id $INSTANCE_ID \
  --availability-zone $AZ \
  --instance-os-user ec2-user \
  --ssh-public-key file:///root/.ssh/id_rsa.pub
```

**Actual output:**

```json
{
    "RequestId": "b330fdd5-559b-4ed0-98e3-ec5f1877de7f",
    "Success": true
}
```

```bash
# Step 2 — SSH IMMEDIATELY and permanently append key (run within 60 seconds)
ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no ec2-user@$PUBLIC_IP \
  "echo '$(cat /root/.ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys && \
   chmod 600 ~/.ssh/authorized_keys"
```
***Screenshots:***
<img width="1041" height="452" alt="image" src="https://github.com/user-attachments/assets/f132fcc8-9056-4879-bbb3-1fea6cd924fe" />
<img width="1035" height="443" alt="image" src="https://github.com/user-attachments/assets/95689b8c-6cbb-4818-ac18-d5e343646b5e" />
<img width="1039" height="303" alt="image" src="https://github.com/user-attachments/assets/62588c43-1f7e-4a72-af13-b5b9b3aaf8fd" />

---

#### Blocker 3: SSH Command Hangs on `cat >> authorized_keys`

**Symptom:**
```bash
ssh -o StrictHostKeyChecking=no ec2-user@$PUBLIC_IP \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
^C   ← had to manually interrupt
```

**Root Cause:** The remote `cat >> ~/.ssh/authorized_keys` reads from **stdin**. With no pipe feeding it data, it blocks indefinitely waiting for EOF. The session never progresses.

**Resolution:** Use shell substitution to expand the key content **locally** before the SSH session opens — passing it as a string argument rather than piping it:

```bash
# ❌ Wrong — remote cat blocks on stdin, hangs until ^C
cat /root/.ssh/id_rsa.pub | ssh ec2-user@$IP "cat >> ~/.ssh/authorized_keys"

# ✅ Correct — $(cat ...) expands locally; key passed as a literal string
ssh -i /root/.ssh/id_rsa ec2-user@$IP \
  "echo '$(cat /root/.ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys"
```

***Screenshot: Terminal showing `send-ssh-public-key` returning `"Success": true`, followed by the silent successful SSH command with no error output***
<img width="1037" height="400" alt="image" src="https://github.com/user-attachments/assets/dd878ac7-ed0b-4ee8-8ab4-78ec849f64f2" />

---

### Phase 6: Validate End-to-End Connectivity

**Objective:** Execute the definitive test — SSH from `aws-client` to `nautilus-public-ec2` and ping `nautilus-private-ec2` across the VPC peering link.

```bash
PRIVATE_EC2_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-private-ec2" \
  --query "Reservations[0].Instances[0].PrivateIpAddress" --output text)

echo "Private EC2 IP: $PRIVATE_EC2_IP"
```

**Output:**
```
Private EC2 IP: 10.1.1.197
```

```bash
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

***Screenshot: Terminal showing the complete ping output — 4/4 packets received, **0% packet loss**, avg RTT 1.529ms***
<img width="1029" height="674" alt="image" src="https://github.com/user-attachments/assets/5d74d6d7-b96b-4459-aca5-cd0794daaa6b" />

### ✅ Completion Checklist

| Requirement | Resource | Status |
|---|---|---|
| VPC Peering Connection created with correct name tag | `nautilus-vpc-peering` · `pcx-0207198e601a5b32d` | ✅ |
| Peering connection accepted — status: active | `pcx-0207198e601a5b32d` | ✅ |
| Route: Default VPC → `10.1.0.0/16` via peering | Default VPC main route table | ✅ |
| Route: Private VPC → `172.31.0.0/16` via peering | Private VPC main route table | ✅ |
| ICMP allowed from `172.31.0.0/16` on private EC2 | `sg-072bab54b2b54f706` · `sgr-05dc8286a75a49a87` | ✅ |
| TCP 22 open on public EC2 security group | `sg-0104b845d741f1ff2` · `sgr-0f4a96f7339598caa` | ✅ |
| Public key appended to `authorized_keys` on public EC2 | `ec2-user@nautilus-public-ec2` | ✅ |
| SSH from `aws-client` → `nautilus-public-ec2` | Key-based auth · `/root/.ssh/id_rsa` | ✅ |
| **Ping `nautilus-private-ec2` from public EC2** | **ICMP across VPC peering · 10.1.1.197** | ✅ **0% packet loss** |

---

## Troubleshooting

### Issue 1: `Connection timed out` on Port 22

**Symptom:**
```
ssh: connect to host 100.55.64.9 port 22: Connection timed out
```

**Root Cause:** The public EC2 security group had no inbound TCP port 22 rule. The TCP SYN packet was dropped at the security group layer before reaching the OS.

**Resolution:**
```bash
aws ec2 authorize-security-group-ingress \
    --group-id $PUBLIC_SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0
```

---

### Issue 2: `Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`

**Symptom:**
```
ec2-user@100.55.64.9: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

**Root Cause:** TCP port 22 was reachable but the instance had no trusted key in `~/.ssh/authorized_keys`. EC2 Amazon Linux 2 disables password authentication by default.

**Resolution:** Bootstrap key trust via EC2 Instance Connect:
```bash
aws ec2-instance-connect send-ssh-public-key \
  --region us-east-1 \
  --instance-id $INSTANCE_ID \
  --availability-zone $AZ \
  --instance-os-user ec2-user \
  --ssh-public-key file:///root/.ssh/id_rsa.pub

# Run within 60 seconds:
ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no ec2-user@$PUBLIC_IP \
  "echo '$(cat /root/.ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

---

### Issue 3: SSH Command Hangs Indefinitely (`^C` required)

**Symptom:**
```bash
ssh ec2-user@$PUBLIC_IP "cat >> ~/.ssh/authorized_keys"
# ← never returns, requires ^C to interrupt
```

**Root Cause:** The remote `cat` process reads from stdin. Without piped input and without an EOF signal, it blocks forever.

**Resolution:** Expand file contents locally with `$(cat ...)` before the SSH session opens:

```bash
# ❌ Broken — remote cat blocks on stdin
cat /root/.ssh/id_rsa.pub | ssh ec2-user@$IP "cat >> ~/.ssh/authorized_keys"

# ✅ Fixed — key content embedded as string argument before SSH executes
ssh -i /root/.ssh/id_rsa ec2-user@$IP \
  "echo '$(cat /root/.ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys"
```

---

### Issue 4: `RouteAlreadyExists` / `InvalidPermission.Duplicate`

**Symptom:**
```
An error occurred (RouteAlreadyExists) when calling the CreateRoute operation:
The route identified by 10.1.0.0/16 already exists.

An error occurred (InvalidPermission.Duplicate) when calling the
AuthorizeSecurityGroupIngress operation: the specified rule already exists.
```

**Root Cause:** Commands were re-executed after the desired state was already achieved on the first pass. These are idempotency signals, not failures.

**Resolution:** No action required. Confirm existing state:
```bash
# Verify route exists
aws ec2 describe-route-tables \
  --route-table-ids $DEFAULT_RT_ID \
  --query "RouteTables[0].Routes[?DestinationCidrBlock=='10.1.0.0/16']"

# Verify ICMP rule exists
aws ec2 describe-security-groups \
  --group-ids $PRIVATE_SG_ID \
  --query "SecurityGroups[0].IpPermissions[?IpProtocol=='icmp']"
```
***Screenshots of Issues***
<img width="1036" height="504" alt="image" src="https://github.com/user-attachments/assets/084c518c-19dc-44d7-a16d-1e0a4da230b2" />

<img width="1037" height="562" alt="image" src="https://github.com/user-attachments/assets/a4c191e8-2695-47ec-91c9-50b2e6e6a5b9" />

---

## Key Learnings

### 1. VPC Peering Requires Three Independent Configuration Layers

Creating a VPC Peering connection alone does not enable traffic flow. All three layers below must be explicitly configured — failure at any single layer silently drops traffic:

```
Layer 1 - VPC Peering Connection   →  logical link (must be accepted, not just created)
Layer 2 - Route Tables             →  both VPCs need bidirectional routes via pcx-*
Layer 3 - Security Groups          →  default-deny; ICMP/TCP must be explicitly allowed
```

### 2. EC2 Instance Connect as a Key Bootstrap Mechanism

When an EC2 instance has no trusted keys and password auth is disabled, **EC2 Instance Connect** (`send-ssh-public-key`) provides a 60-second injection window to bootstrap permanent SSH trust, without console access, a bastion host, or instance reboot. The pattern: inject → immediately SSH with matching private key → permanently append key to `authorized_keys`.

### 3. Shell Substitution vs. Piping for Remote File Writes

`echo '$(cat file)'` evaluates the subshell **on the local machine** before the SSH session is established, passing the result as a string literal. Piping (`cat file | ssh "cat >> file"`) leaves the remote process reading from stdin, which blocks indefinitely without an explicit EOF. One character of difference in the command produces completely opposite behavior.

### 4. `RouteAlreadyExists` and `InvalidPermission.Duplicate` Are Success Signals

When running infrastructure CLI commands across sessions, these errors confirm the desired state was achieved in a prior execution. Treat them as idempotency receipts and verify with a describe command rather than treating them as blockers.

### 5. Route Table Scope - Main vs. Subnet-Specific

`RouteTables[0]` retrieves the first route table returned for a VPC filter, which is the main route table in most cases. However, if a subnet has a custom route table that overrides the main one, instances in that subnet follow the subnet-level routes. Always verify subnet-level route table associations when debugging unexpected routing behavior after peering is confirmed active.

---

## References

- [AWS VPC Peering — Official Documentation](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
- [EC2 Instance Connect — AWS Docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Connect-using-EC2-Instance-Connect.html)
- [VPC Route Tables — AWS Docs](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [Security Groups for EC2 — AWS Docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html)
- [AWS CLI EC2 Reference](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/index.html)

---

> *All commands, IDs, IPs, and output blocks in this document reflect the exact values produced during the live implementation run. No values have been approximated or substituted.*
