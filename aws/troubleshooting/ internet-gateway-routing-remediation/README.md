# VPC Internet Egress Restoration via Internet Gateway Attachment and Subnet Reachability Remediation

### Restoring Public Internet Access to an EC2-Hosted Nginx Application via VPC Internet Gateway Attachment

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Reference](#environment-reference)
- [Resolution Walkthrough](#resolution-walkthrough)
  - [Pre-Flight: Verify AWS CLI Access](#pre-flight-verify-aws-cli-access)
  - [Phase 1: Collect All Resource IDs](#phase-1-collect-all-resource-ids)
  - [Phase 2: Diagnose and Fix the Internet Gateway](#phase-2-diagnose-and-fix-the-internet-gateway)
  - [Phase 3: Verify the Route Table](#phase-3-verify-the-route-table)
  - [Phase 4: Verify Subnet Public IP Configuration](#phase-4-verify-subnet-public-ip-configuration)
  - [Phase 5: Verify Security Group Rules](#phase-5-verify-security-group-rules)
  - [Phase 6: Verify Nginx on the EC2 Instance](#phase-6-verify-nginx-on-the-ec2-instance)
  - [Phase 7: End-to-End Validation](#phase-7-end-to-end-validation)
  - [Phase 8: Final State Snapshot](#phase-8-final-state-snapshot)
- [Root Cause Summary](#root-cause-summary)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This runbook documents the full investigation and resolution of a production incident in which an EC2-hosted Nginx web application was inaccessible from the internet despite a correctly configured security group and a running application server. The root cause was identified as a detached Internet Gateway on the target VPC, combined with a secondary misconfiguration in the subnet public IP assignment attribute.

**Outcome:** Full public HTTP access restored. `HTTP 200 OK` confirmed via external curl validation.

---

## Problem Statement

The Nautilus Development Team deployed a new web application on an EC2 instance (`xfusion-ec2`) within a public VPC (`xfusion-vpc`) in the `us-east-1` region. The application runs on an Nginx server and is expected to be reachable from the internet on **port 80**.

Despite the following conditions being met:

* Security group `xfusion-sg` configured to allow inbound TCP traffic on port 80
* EC2 instance confirmed in a `running` state with a public IP assigned
* Nginx service confirmed enabled on the instance

**The application remained inaccessible from the internet.**

The DevOps team was engaged to perform a structured, layer-by-layer investigation and restore full internet accessibility.

---

## Architecture

```
Internet
    |
    v
Internet Gateway (igw-00848032487db327d)
    |
    v
VPC: xfusion-vpc (vpc-09eed66dedff99616) -- CIDR: 10.0.0.0/16
    |
    v
Route Table: rtb-03a995c5648181f42
  0.0.0.0/0  -->  igw-00848032487db327d
  10.0.0.0/16 --> local
    |
    v
Public Subnet: subnet-07d943b6368f60dd6 (us-east-1a) -- CIDR: 10.0.1.0/24
    |
    v
Security Group: xfusion-sg (sg-0376eb30fc74b9181)
  Inbound: TCP 80  0.0.0.0/0
  Inbound: TCP 22  0.0.0.0/0
    |
    v
EC2 Instance: xfusion-ec2 (i-054d7d24c49753341)
  Private IP : 10.0.1.108
  Public IP  : 44.211.138.65
  OS         : Amazon Linux 2023
  App        : Nginx (active/running on 0.0.0.0:80)
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| AWS CLI | Installed and configured |
| IAM Permissions | EC2 read/write, VPC read/write |
| Region | `us-east-1` |
| Authentication | IAM user with programmatic access |
| Shell | Bash (Linux / macOS / WSL) |

---

## Environment Reference

All resource IDs confirmed and used throughout this runbook:

| Resource | Name | ID |
|---|---|---|
| VPC | `xfusion-vpc` | `vpc-09eed66dedff99616` |
| Internet Gateway | `xfusion-igw` | `igw-00848032487db327d` |
| Route Table | | `rtb-03a995c5648181f42` |
| Subnet | | `subnet-07d943b6368f60dd6` |
| EC2 Instance | `xfusion-ec2` | `i-054d7d24c49753341` |
| Public IP | | `44.211.138.65` |
| Security Group | `xfusion-sg` | `sg-0376eb30fc74b9181` |
| AWS Account | | `613837775091` |
| IAM User | `kk_labs_user_235523` | |

---

## Resolution Walkthrough

---

### Pre-Flight: Verify AWS CLI Access

Before touching any resource, confirm CLI authentication and region targeting.

**Verify identity:**

```bash
aws sts get-caller-identity
```

**Expected output:**

```json
{
    "UserId": "AIDAY524VEDZUUEP3S27U",
    "Account": "613837775091",
    "Arn": "arn:aws:iam::613837775091:user/kk_labs_user_235523"
}
```

**Verify region:**

```bash
aws configure get region
```

**Expected output:**

```
us-east-1
```

> **SCREENSHOT**

<img width="1028" height="495" alt="Image" src="https://github.com/user-attachments/assets/e09ec41b-c629-4e24-99ff-6644cd82fd54" />

> *Shows: `aws sts get-caller-identity` output confirming account and ARN, and `aws configure get region` returning `us-east-1`*

---

### Phase 1: Collect All Resource IDs

> Never hardcode assumptions. Pull every resource ID dynamically before making any changes.

---

#### Step 1.1 -- Get the VPC ID for `xfusion-vpc`

```bash
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=xfusion-vpc" \
  --query "Vpcs[*].{VpcId:VpcId,CidrBlock:CidrBlock,State:State}" \
  --output table
```

**Result:**

```
-------------------------------------------------------
|                    DescribeVpcs                     |
+--------------+------------+-------------------------+
|   CidrBlock  |   State    |          VpcId          |
+--------------+------------+-------------------------+
|  10.0.0.0/16 |  available |  vpc-09eed66dedff99616  |
+--------------+------------+-------------------------+
```

**Set as variable:**

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=xfusion-vpc" \
  --query "Vpcs[0].VpcId" \
  --output text)

echo "VPC ID: $VPC_ID"
```

**Output:** `VPC ID: vpc-09eed66dedff99616`

> **SCREENSHOT**

<img width="1032" height="854" alt="Image" src="https://github.com/user-attachments/assets/1ce698be-2bfc-4c39-aca7-8db4ac6e7fb3" />

> *Shows: VPC describe table output with `vpc-09eed66dedff99616`, state `available`, CIDR `10.0.0.0/16` and variable echo confirmation*

---

#### Step 1.2 -- Get EC2 Instance Details for `xfusion-ec2`

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query "Reservations[*].Instances[*].{InstanceId:InstanceId,State:State.Name,PublicIP:PublicIpAddress,PrivateIP:PrivateIpAddress,SubnetId:SubnetId,VpcId:VpcId}" \
  --output table
```

**Result:**

```
------------------------------------------------------------------------------------------------------------------------
|                                                   DescribeInstances                                                  |
+---------------------+-------------+----------------+----------+----------------------------+-------------------------+
|     InstanceId      |  PrivateIP  |   PublicIP     |  State   |         SubnetId           |          VpcId          |
+---------------------+-------------+----------------+----------+----------------------------+-------------------------+
|  i-054d7d24c49753341|  10.0.1.108 |  44.211.138.65 |  running |  subnet-07d943b6368f60dd6  |  vpc-09eed66dedff99616  |
+---------------------+-------------+----------------+----------+----------------------------+-------------------------+
```

**Set all EC2 variables:**

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

SUBNET_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query "Reservations[0].Instances[0].SubnetId" \
  --output text)

PUBLIC_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

echo "Instance ID : $INSTANCE_ID"
echo "Subnet ID   : $SUBNET_ID"
echo "Public IP   : $PUBLIC_IP"
```

**Output:**

```
Instance ID : i-054d7d24c49753341
Subnet ID   : subnet-07d943b6368f60dd6
Public IP   : 44.211.138.65
```

> **SCREENSHOT**
<img width="1031" height="866" alt="Image" src="https://github.com/user-attachments/assets/825b2644-a8f1-4054-ab89-e09d03268ff4" />

> *Shows: Instance describe table with all fields populated including `running` state and `44.211.138.65` public IP, followed by variable echo confirmation*

---

#### Step 1.3 -- Get the Security Group ID for `xfusion-sg`

```bash
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=xfusion-sg" \
  --query "SecurityGroups[0].GroupId" \
  --output text)

echo "Security Group ID: $SG_ID"
```

**Output:** `Security Group ID: sg-0376eb30fc74b9181`

> **SCREENSHOT**

<img width="1034" height="864" alt="Image" src="https://github.com/user-attachments/assets/7dc36308-5447-46f1-aca9-8de604677ea3" />

> *Shows: Variable echo confirming `sg-0376eb30fc74b9181`*

---

### Phase 2: Diagnose and Fix the Internet Gateway

---

#### Step 2.1 -- Check if an IGW is Attached to the VPC

```bash
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --query "InternetGateways[*].{IGW_Id:InternetGatewayId,State:Attachments[0].State,VpcId:Attachments[0].VpcId}" \
  --output table
```

**Result:** *Empty output -- no IGW attached to `xfusion-vpc`.*

> **SCREENSHOT**

<img width="1032" height="714" alt="Image" src="https://github.com/user-attachments/assets/c9b0fa65-1539-4909-8a6b-ad43e927e02b" />

> *Shows: `describe-internet-gateways` filtered by VPC ID returning empty output -- confirming no IGW is currently attached*

**Root cause identified.** No Internet Gateway is attached to `xfusion-vpc`. All inbound internet traffic has no entry point into the VPC regardless of security group rules.

---

#### Step 2.2 -- Find the Unattached IGW

```bash
aws ec2 describe-internet-gateways \
  --query "InternetGateways[*].{IGW_Id:InternetGatewayId,Attachments:Attachments[0].State,VpcId:Attachments[0].VpcId}" \
  --output table
```

**Result:**

```
-------------------------------------------------------------------
|                    DescribeInternetGateways                     |
+-------------+-------------------------+-------------------------+
| Attachments |         IGW_Id          |          VpcId          |
+-------------+-------------------------+-------------------------+
|  None       |  igw-00848032487db327d  |  None                   |
|  available  |  igw-0c7be0b8b0d458cb5  |  vpc-0212f100dac794cc8  |
+-------------+-------------------------+-------------------------+
```

**Analysis:**

| IGW ID | Status | Action |
|---|---|---|
| `igw-00848032487db327d` | Detached (`None`) | **Attach to `xfusion-vpc`** |
| `igw-0c7be0b8b0d458cb5` | Attached to another VPC | Do not touch |

> **SCREENSHOT**

<img width="1027" height="528" alt="Image" src="https://github.com/user-attachments/assets/aea25ee7-c39a-431f-9759-b103f7e18515" />

> *Shows: Full IGW listing with `igw-00848032487db327d` showing `None` attachment state and `igw-0c7be0b8b0d458cb5` showing `available` on a different VPC*

---

#### Step 2.3 -- Set IGW Variable and Attach to VPC

```bash
IGW_ID=igw-00848032487db327d

echo "IGW ID: $IGW_ID"
```

**Output:** `IGW ID: igw-00848032487db327d`

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
```

> No output from this command confirms success.

**Verify the attachment:**

```bash
aws ec2 describe-internet-gateways \
  --internet-gateway-ids $IGW_ID \
  --query "InternetGateways[0].Attachments" \
  --output table
```

**Result:**

```
----------------------------------------
|       DescribeInternetGateways       |
+------------+-------------------------+
|    State   |          VpcId          |
+------------+-------------------------+
|  available |  vpc-09eed66dedff99616  |
+------------+-------------------------+
```

> **SCREENSHOT**

<img width="1033" height="773" alt="Image" src="https://github.com/user-attachments/assets/a672048b-41af-4c0f-9219-238f3deb49eb" />

> *Shows: IGW variable set, `attach-internet-gateway` command with no error output, and verification table showing state `available` with `vpc-09eed66dedff99616`*

---

### Phase 3: Verify the Route Table

---

#### Step 3.1 -- Find the Route Table for the EC2 Subnet

```bash
RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
  --query "RouteTables[0].RouteTableId" \
  --output text)

echo "Route Table ID: $RT_ID"
```

**Output:** `Route Table ID: rtb-03a995c5648181f42`

---

#### Step 3.2 -- Verify the Default Route Exists

```bash
aws ec2 describe-route-tables \
  --route-table-ids $RT_ID \
  --query "RouteTables[0].Routes[*].{Destination:DestinationCidrBlock,Target:GatewayId,State:State}" \
  --output table
```

**Result:**

```
----------------------------------------------------
|                DescribeRouteTables               |
+--------------+---------+-------------------------+
|  Destination |  State  |         Target          |
+--------------+---------+-------------------------+
|  10.0.0.0/16 |  active |  local                  |
|  0.0.0.0/0   |  active |  igw-00848032487db327d  |
+--------------+---------+-------------------------+
```

The `0.0.0.0/0` route pointing to `igw-00848032487db327d` was auto-populated when the IGW was attached. No manual route creation was required.

> **SCREENSHOT**

<img width="1019" height="841" alt="Image" src="https://github.com/user-attachments/assets/7a93064d-fc69-4b26-aa9e-41ff0b919152" />

> *Shows: Route table query output with both routes active -- `10.0.0.0/16` to local and `0.0.0.0/0` to `igw-00848032487db327d`*

---

### Phase 4: Verify Subnet Public IP Configuration

---

#### Step 4.1 -- Check `MapPublicIpOnLaunch` Attribute

```bash
aws ec2 describe-subnets \
  --subnet-ids $SUBNET_ID \
  --query "Subnets[*].{SubnetId:SubnetId,MapPublicIpOnLaunch:MapPublicIpOnLaunch,AvailabilityZone:AvailabilityZone,CidrBlock:CidrBlock}" \
  --output table
```

**Result:**

```
----------------------------------------------------------------------------------------
|                                    DescribeSubnets                                   |
+------------------+--------------+-----------------------+----------------------------+
| AvailabilityZone |  CidrBlock   |  MapPublicIpOnLaunch  |         SubnetId           |
+------------------+--------------+-----------------------+----------------------------+
|  us-east-1a      |  10.0.1.0/24 |  False                |  subnet-07d943b6368f60dd6  |
+------------------+--------------+-----------------------+----------------------------+
```

**Issue found.** `MapPublicIpOnLaunch` is `False`. New instances launched in this subnet will not automatically receive a public IP address, which misclassifies it as a private subnet.

> **SCREENSHOT**

<img width="1033" height="691" alt="Image" src="https://github.com/user-attachments/assets/44c88a70-b027-4005-a52c-05262bc211d1" />

> *Shows: Subnet describe table with `MapPublicIpOnLaunch` column showing `False`*

---

#### Step 4.2 -- Enable Auto-Assign Public IP

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id $SUBNET_ID \
  --map-public-ip-on-launch
```

> No output confirms success.

**Verify the fix:**

```bash
aws ec2 describe-subnets \
  --subnet-ids $SUBNET_ID \
  --query "Subnets[*].{SubnetId:SubnetId,MapPublicIpOnLaunch:MapPublicIpOnLaunch,AvailabilityZone:AvailabilityZone,CidrBlock:CidrBlock}" \
  --output table
```

**Result:**

```
----------------------------------------------------------------------------------------
|                                    DescribeSubnets                                   |
+------------------+--------------+-----------------------+----------------------------+
| AvailabilityZone |  CidrBlock   |  MapPublicIpOnLaunch  |         SubnetId           |
+------------------+--------------+-----------------------+----------------------------+
|  us-east-1a      |  10.0.1.0/24 |  True                 |  subnet-07d943b6368f60dd6  |
+------------------+--------------+-----------------------+----------------------------+
```

> **SCREENSHOT**

<img width="1035" height="617" alt="Image" src="https://github.com/user-attachments/assets/7b7cb12f-ec05-4fc4-b1dc-1c8de6d5a588" />

> *Shows: Before and after subnet describe table -- `False` becoming `True` for `MapPublicIpOnLaunch`*

---

### Phase 5: Verify Security Group Rules

---

#### Step 5.1 -- Audit Inbound Rules on `xfusion-sg`

```bash
aws ec2 describe-security-groups \
  --group-ids $SG_ID \
  --query "SecurityGroups[0].IpPermissions[*].{Protocol:IpProtocol,FromPort:FromPort,ToPort:ToPort,CIDR:IpRanges[0].CidrIp}" \
  --output table
```

**Result:**

```
-------------------------------------------------
|            DescribeSecurityGroups             |
+-----------+------------+------------+---------+
|   CIDR    | FromPort   | Protocol   | ToPort  |
+-----------+------------+------------+---------+
|  0.0.0.0/0|  80        |  tcp       |  80     |
+-----------+------------+------------+---------+
```

Port 80 is correctly open to `0.0.0.0/0`. No change required.

> **SCREENSHOT**

<img width="1030" height="847" alt="Image" src="https://github.com/user-attachments/assets/171b6bf6-e3ab-4728-99a8-26e14b1f2bb3" />

> *Shows: Security group inbound rules table with port 80 TCP open to `0.0.0.0/0`*

---

#### Step 5.2 -- Confirm Security Group is Attached to the Instance

```bash
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].SecurityGroups[*].{GroupId:GroupId,GroupName:GroupName}" \
  --output table
```

**Result:**

```
----------------------------------------
|           DescribeInstances          |
+-----------------------+--------------+
|        GroupId        |  GroupName   |
+-----------------------+--------------+
|  sg-0376eb30fc74b9181 |  xfusion-sg  |
+-----------------------+--------------+
```

`xfusion-sg` is correctly attached to `xfusion-ec2`. No change required.

> **SCREENSHOT**

<img width="1035" height="825" alt="Image" src="https://github.com/user-attachments/assets/0b0e813d-effb-4186-9bda-8296cf066165" />

> *Shows: Instance security group query returning `sg-0376eb30fc74b9181` and `xfusion-sg`*

---

### Phase 6: Verify Nginx on the EC2 Instance

---

#### Step 6.1 -- Attempt SSM Session (Failed)

```bash
aws ssm start-session \
  --target $INSTANCE_ID
```

**Result:**

```
An error occurred (TargetNotConnected) when calling the StartSession operation:
i-054d7d24c49753341 is not connected.
```

SSM agent is not running on this instance. Switching to EC2 Instance Connect via SSH.

> **SCREENSHOT**

<img width="1032" height="378" alt="Image" src="https://github.com/user-attachments/assets/905e3849-b3af-4e3d-ba97-f15ba5bdf4e8" />

> *Shows: `start-session` command returning `TargetNotConnected` error for instance `i-054d7d24c49753341`*

---

#### Step 6.2 -- Retrieve Availability Zone

```bash
AZ=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" \
  --output text)

echo "Availability Zone: $AZ"
```

**Output:** `Availability Zone: us-east-1a`

---

#### Step 6.3 -- Check for Existing SSH Keys

```bash
ls ~/.ssh/
```

**Output:** `agent-environment  authorized_keys`

No RSA key pair exists. A new key pair must be generated.

> **SCREENSHOT**

<img width="1033" height="256" alt="Image" src="https://github.com/user-attachments/assets/408d85ff-2271-4814-9849-ada7dd260f16" />

> *Shows: AZ variable echo confirming `us-east-1a` and `ls ~/.ssh/` output showing no `id_rsa` keypair present*

---

#### Step 6.4 -- Generate SSH Key Pair

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
```

**Output:**

```
Generating public/private rsa key pair.
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:5MFvrU4dGkIScM56lxdztmeuLLw4+hCGzcgDNR6UFGw root@aws-client
```

---

#### Step 6.5 -- Add SSH Inbound Rule to Security Group

Initial SSH connection attempt timed out. Port 22 was not open in `xfusion-sg`.

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

**Result:**

```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0c09db13e3c7d8170",
            "GroupId": "sg-0376eb30fc74b9181",
            "GroupOwnerId": "613837775091",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
```

**Verify both rules are now present:**

```bash
aws ec2 describe-security-groups \
  --group-ids $SG_ID \
  --query "SecurityGroups[0].IpPermissions[*].{Protocol:IpProtocol,FromPort:FromPort,ToPort:ToPort,CIDR:IpRanges[0].CidrIp}" \
  --output table
```

**Result:**

```
-------------------------------------------------
|            DescribeSecurityGroups             |
+-----------+------------+------------+---------+
|   CIDR    | FromPort   | Protocol   | ToPort  |
+-----------+------------+------------+---------+
|  0.0.0.0/0|  80        |  tcp       |  80     |
|  0.0.0.0/0|  22        |  tcp       |  22     |
+-----------+------------+------------+---------+
```

> **SCREENSHOT**

<img width="1034" height="663" alt="Image" src="https://github.com/user-attachments/assets/8a871168-2837-4c81-9a41-5f7ae3820f30" />
<img width="1037" height="290" alt="Image" src="https://github.com/user-attachments/assets/d43fd185-26ba-4e68-bb9c-d687d0965750" />
<img width="1036" height="698" alt="Image" src="https://github.com/user-attachments/assets/070ff343-e82c-4315-a5b7-3d0d94eacacb" />

> *Shows: `authorize-security-group-ingress` success JSON response and updated security group table with both port 22 and port 80 rules*

---

#### Step 6.6 -- Connect via EC2 Instance Connect

Push the public key and SSH in a single chained command to avoid key expiry (60-second TTL):

```bash
aws ec2-instance-connect send-ssh-public-key \
  --instance-id $INSTANCE_ID \
  --availability-zone $AZ \
  --instance-os-user ec2-user \
  --ssh-public-key file://~/.ssh/id_rsa.pub && \
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no ec2-user@$PUBLIC_IP
```

**Result:**

```json
{
    "RequestId": "37caec48-f1f2-465c-98fa-834f96dfe018",
    "Success": true
}
```

```
Warning: Permanently added '44.211.138.65' (ECDSA) to the list of known hosts.
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023

[ec2-user@ip-10-0-1-108 ~]$
```

> **SCREENSHOT**

<img width="1037" height="678" alt="Image" src="https://github.com/user-attachments/assets/dd3b7cfd-570b-4c4d-8513-9d6cff585863" />

> *Shows: `send-ssh-public-key` returning `Success: true`, followed by successful SSH connection to `ip-10-0-1-108` with Amazon Linux 2023 banner and `ec2-user` prompt*

---

#### Step 6.7 -- Verify Nginx Service Status

```bash
sudo systemctl status nginx
```

**Result:**

```
* nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Sun 2026-03-15 00:51:18 UTC; 40min ago
    Process: 4178 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 4186 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 4263 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 4309 (nginx)
      Tasks: 2 (limit: 1114)
     Memory: 5.0M
        CPU: 58ms
     CGroup: /system.slice/nginx.service
             |--4309 "nginx: master process /usr/sbin/nginx"
             `--4311 "nginx: worker process"

Mar 15 00:51:18 ... nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Mar 15 00:51:18 ... nginx: configuration file /etc/nginx/nginx.conf test is successful
Mar 15 00:51:18 ... systemd[1]: Started nginx.service - The nginx HTTP and reverse proxy server.
```

Nginx is `active (running)`. Config syntax valid. No changes required.

> **SCREENSHOT**

<img width="1033" height="852" alt="Image" src="https://github.com/user-attachments/assets/1737c570-b8a3-47a6-ae61-1a96b9de889a" />

> *Shows: `systemctl status nginx` output with green `active (running)` status, `enabled` load state, both master and worker processes, and config test success logs*

---

#### Step 6.8 -- Confirm Nginx is Listening on Port 80

```bash
sudo ss -tlnp | grep :80
```

**Result:**

```
LISTEN 0  511  0.0.0.0:80  0.0.0.0:*  users:(("nginx",pid=4311,fd=8),("nginx",pid=4309,fd=8))
LISTEN 0  511     [::]:80     [::]:*  users:(("nginx",pid=4311,fd=9),("nginx",pid=4309,fd=9))
```

Nginx is listening on all IPv4 and IPv6 interfaces on port 80. Exit the instance.

```bash
exit
```

> **SCREENSHOT**

<img width="1036" height="540" alt="Image" src="https://github.com/user-attachments/assets/883e3f05-e0a4-469f-9b2d-c52c8b66ec9a" />

> *Shows: `ss -tlnp` output with two LISTEN entries on `0.0.0.0:80` and `[::]:80` both bound to nginx PIDs 4309 and 4311, followed by `exit` and connection closed message*

---

### Phase 7: End-to-End Validation

From the local terminal, confirm external HTTP access to the public IP:

```bash
curl --connect-timeout 10 --max-time 15 -o /dev/null -s -w \
  "HTTP Code: %{http_code}\nTime to connect: %{time_connect}s\nTime total: %{time_total}s\n" \
  http://$PUBLIC_IP
```

**Result:**

```
HTTP Code: 200
Time to connect: 0.099760s
Time total: 0.197838s
```

**HTTP 200 confirmed.** The Nginx application is publicly accessible from the internet.

> **SCREENSHOT**

<img width="1028" height="667" alt="Image" src="https://github.com/user-attachments/assets/7a7d0d77-151a-42bd-98c5-db5912fc16e0" />

> *Shows: `curl` command with full flags targeting `$PUBLIC_IP` returning `HTTP Code: 200`, `Time to connect: 0.099760s`, `Time total: 0.197838s`*

---

### Phase 8: Final State Snapshot

```bash
echo "======= FINAL STATE SNAPSHOT ======="
echo "VPC ID       : $VPC_ID"
echo "IGW ID       : $IGW_ID"
echo "Route Table  : $RT_ID"
echo "Subnet ID    : $SUBNET_ID"
echo "Instance ID  : $INSTANCE_ID"
echo "Public IP    : $PUBLIC_IP"
echo "Security GRP : $SG_ID"
echo "====================================="
```

**Output:**

```
======= FINAL STATE SNAPSHOT =======
VPC ID       : vpc-09eed66dedff99616
IGW ID       : igw-00848032487db327d
Route Table  : rtb-03a995c5648181f42
Subnet ID    : subnet-07d943b6368f60dd6
Instance ID  : i-054d7d24c49753341
Public IP    : 44.211.138.65
Security GRP : sg-0376eb30fc74b9181
=====================================
```

> **SCREENSHOT**

<img width="1035" height="553" alt="Image" src="https://github.com/user-attachments/assets/5e6455db-74e3-4064-a64e-77a7bebc0124" />

> *Shows: All eight echo commands producing the complete final state snapshot with all confirmed resource IDs*

---

## Root Cause Summary

| Phase | Layer | Finding | Status | Action Taken |
|---|---|---|---|---|
| Phase 2 | Internet Gateway | `igw-00848032487db327d` existed but was detached from `xfusion-vpc` | **ROOT CAUSE** | Attached IGW to VPC |
| Phase 3 | Route Table | `0.0.0.0/0` route auto-populated on IGW attachment | No action required | Verified only |
| Phase 4 | Subnet | `MapPublicIpOnLaunch` was `False` | **SECONDARY ISSUE** | Enabled auto-assign public IP |
| Phase 5 | Security Group | Port 80 TCP open to `0.0.0.0/0`, correctly attached to instance | No action required | Verified only |
| Phase 5b | Security Group | Port 22 TCP missing -- blocked SSH access during investigation | **OPERATIONAL BLOCKER** | Added TCP 22 inbound rule |
| Phase 6 | Application | Nginx `active (running)`, listening on `0.0.0.0:80`, config valid | No action required | Verified only |

**Primary root cause:** The Internet Gateway `igw-00848032487db327d` existed in the account but was not attached to `xfusion-vpc`. Without an attached IGW, all inbound and outbound internet traffic is blocked at the VPC boundary regardless of any downstream configuration.

---

## Lessons Learned

**1. IGW attachment is not automatic.**
Creating an Internet Gateway does not attach it to a VPC. Attachment is a separate explicit action. Infrastructure provisioning pipelines must verify IGW attachment state, not just IGW existence.

**2. Subnet `MapPublicIpOnLaunch` must be explicitly enabled.**
Subnets intended to be public require this attribute to be set to `True`. The default is `False`. Without it, new instances launched into the subnet will not receive a public IP, defeating the public subnet design.

**3. Security group port 22 is not open by default.**
SSH access requires an explicit inbound rule. In production environments where SSM is unavailable, port 22 must be pre-provisioned in the security group or an alternative access method (bastion, VPN) must be established before deployment.

**4. SSM is the preferred access method.**
EC2 Instance Connect with a 60-second key TTL is workable but operationally fragile. SSM Session Manager eliminates SSH dependency entirely. The SSM agent should be pre-installed and the instance role should include `AmazonSSMManagedInstanceCore`.

**5. Diagnose before changing.**
All resource IDs were collected and verified before any change was made. This prevents incorrect targeting and creates a clean audit trail.

---

## References

* [AWS Documentation: Internet Gateways](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
* [AWS Documentation: Route Tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
* [AWS Documentation: EC2 Instance Connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Connect-using-EC2-Instance-Connect.html)
* [AWS Documentation: Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
* [AWS Documentation: Subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)
* [AWS Documentation: SSM Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)

---

*Region: us-east-1*
*Resolution Time: Under 60 minutes*
*Final Validation: HTTP 200 OK at `44.211.138.65`*
