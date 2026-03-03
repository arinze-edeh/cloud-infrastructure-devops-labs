# AWS NAT Instance: Private EC2 Internet Access

> **Enterprise DevOps Implementation** | Enabling outbound internet connectivity for private subnet workloads using a cost-optimized NAT Instance architecture on AWS.

---

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Implementation](#implementation)
  * [Phase 1: Credential Verification and Resource Discovery](#phase-1-credential-verification-and-resource-discovery)
  * [Phase 2: Public Subnet and Routing Infrastructure](#phase-2-public-subnet-and-routing-infrastructure)
  * [Phase 3: NAT Instance Security Group](#phase-3-nat-instance-security-group)
  * [Phase 4: NAT Instance Provisioning](#phase-4-nat-instance-provisioning)
  * [Phase 5: NAT Instance OS Configuration](#phase-5-nat-instance-os-configuration)
  * [Phase 6: Private Route Table Update](#phase-6-private-route-table-update)
  * [Phase 7: Verification](#phase-7-verification)
* [Resource Summary](#resource-summary)
* [Troubleshooting](#troubleshooting)
* [Cost Considerations](#cost-considerations)
* [Security Considerations](#security-considerations)
* [References](#references)

---

## Overview

This project documents the end-to-end implementation of a **NAT Instance** on AWS to provide internet access to a private EC2 workload. The solution avoids the ongoing cost of a managed NAT Gateway by using a self-managed Amazon Linux 2 EC2 instance configured as a network address translator.

The private EC2 instance runs a cron job that uploads a test file to an S3 bucket every minute. The task is considered complete when that file appears in the bucket, confirming successful outbound internet routing through the NAT Instance.

### Key Technologies

* **AWS EC2** -- NAT Instance and private workload host
* **AWS VPC** -- Subnets, route tables, Internet Gateway
* **AWS S3** -- Upload destination for connectivity verification
* **iptables** -- Kernel-level NAT masquerading on Amazon Linux 2
* **AWS EC2 Instance Connect** -- Keyless SSH access for instance configuration

---

## Problem Statement

### Context

The Nautilus DevOps team manages a private EC2 instance (`devops-priv-ec2`) running inside a private subnet (`devops-priv-subnet`) within a dedicated VPC (`devops-priv-vpc`). The instance is pre-configured with a cron job that attempts to upload `devops-test.txt` to S3 bucket `devops-nat-17499` every minute.

### Challenge

The private subnet had no outbound internet route. The VPC also lacked an Internet Gateway attachment, making outbound traffic from the private instance impossible. A NAT Gateway would solve this but introduces significant hourly and data-processing costs that are unnecessary for this workload.

### Constraints

* No pre-existing Internet Gateway attached to the target VPC
* No SSH key pair available on the bastion host at launch time
* NAT Gateway explicitly excluded due to cost requirements
* Must use Amazon Linux 2 AMI for the NAT Instance

### Resolution

Deploy a self-managed **NAT Instance** in a new public subnet within the same VPC and Availability Zone. Configure Linux IP forwarding and iptables masquerading on the instance, then update the private subnet route table to direct all outbound traffic through the NAT Instance.

---

## Architecture

```
                          devops-priv-vpc (10.1.0.0/16)
  +------------------------------------------------------------+
  |                                                            |
  |   devops-priv-subnet (10.1.1.0/24)  us-east-1a            |
  |   +---------------------------+                            |
  |   |  devops-priv-ec2          |                            |
  |   |  (cron: upload to S3)     |                            |
  |   |  Route: 0.0.0.0/0 ------->|---+                        |
  |   +---------------------------+   |                        |
  |                                   v                        |
  |   devops-pub-subnet (10.1.2.0/24) us-east-1a               |
  |   +---------------------------+   |                        |
  |   |  devops-nat-instance      |<--+                        |
  |   |  44.211.188.254 (public)  |                            |
  |   |  iptables MASQUERADE      |                            |
  |   |  Route: 0.0.0.0/0 ------->|---+                        |
  |   +---------------------------+   |                        |
  |                                   v                        |
  +------------------------------> devops-igw ----------------->  Internet / S3
```

### Traffic Flow

1. Private EC2 cron job initiates outbound S3 upload request
2. Private subnet route table directs `0.0.0.0/0` to NAT Instance ENI
3. NAT Instance applies iptables MASQUERADE, replacing source IP with its own public IP
4. Traffic exits via the Internet Gateway to the internet
5. Response packets return to the NAT Instance, which forwards them back to the private EC2

---

## Prerequisites

| Requirement | Detail |
|---|---|
| AWS CLI | Configured with valid credentials |
| IAM Permissions | EC2 full access, S3 read, VPC management |
| Existing VPC | `devops-priv-vpc` with private subnet and private EC2 |
| Region | `us-east-1` |
| Bastion/Client Host | `aws-client` with EC2 Instance Connect CLI available |

### Verify Credentials

```bash
aws sts get-caller-identity
```

**Screenshot: Credential verification output**

> ![Credential Verification](screenshots/01-credentials-verified.png)

---

## Implementation

### Phase 1: Credential Verification and Resource Discovery

Collect all existing resource IDs before creating anything. Every subsequent command depends on these values.

```bash
# VPC ID
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=devops-priv-vpc" \
  --query 'Vpcs[0].VpcId' \
  --output text

# VPC CIDR
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=devops-priv-vpc" \
  --query 'Vpcs[0].CidrBlock' \
  --output text

# Private subnet ID
aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=devops-priv-subnet" \
  --query 'Subnets[0].SubnetId' \
  --output text

# Private subnet CIDR
aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=devops-priv-subnet" \
  --query 'Subnets[0].CidrBlock' \
  --output text

# Private subnet Availability Zone
aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=devops-priv-subnet" \
  --query 'Subnets[0].AvailabilityZone' \
  --output text

# Private subnet route table
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=<PRIV_SUBNET_ID>" \
  --query 'RouteTables[0].RouteTableId' \
  --output text
```

#### Discovered Values

| Resource | Value |
|---|---|
| VPC ID | `vpc-01b75a856c993de09` |
| VPC CIDR | `10.1.0.0/16` |
| Private Subnet ID | `subnet-05adbff3594106e34` |
| Private Subnet CIDR | `10.1.1.0/24` |
| Availability Zone | `us-east-1a` |
| Private Route Table | `rtb-0ec8f8b9133961a6e` |

**Screenshot: Resource discovery commands and outputs**

> ![Resource Discovery](screenshots/02-resource-discovery.png)

#### Problem: No Internet Gateway Attached to VPC

Running `describe-internet-gateways` with the VPC filter returned `None`. A full account-level query revealed the only existing IGW was attached to a different VPC.

```bash
aws ec2 describe-internet-gateways \
  --query 'InternetGateways[*].{ID:InternetGatewayId,State:Attachments[0].State,VPC:Attachments[0].VpcId}' \
  --output table
```

**Resolution:** Create and attach a new Internet Gateway to the target VPC.

```bash
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=devops-igw}]' \
  --query 'InternetGateway.InternetGatewayId' \
  --output text

aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0270baf826dd1daf0 \
  --vpc-id vpc-01b75a856c993de09
```

**Screenshot: IGW creation and attachment confirmation**

> ![IGW Creation](screenshots/03-igw-created-attached.png)

---

### Phase 2: Public Subnet and Routing Infrastructure

Create the public subnet in the same Availability Zone as the private subnet, then build a dedicated route table with an internet route.

```bash
# Create public subnet
aws ec2 create-subnet \
  --vpc-id vpc-01b75a856c993de09 \
  --cidr-block 10.1.2.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=devops-pub-subnet}]' \
  --query 'Subnet.SubnetId' \
  --output text

# Enable auto-assign public IP
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-0a689b36076f85b9d \
  --map-public-ip-on-launch

# Create route table
aws ec2 create-route-table \
  --vpc-id vpc-01b75a856c993de09 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=devops-pub-rt}]' \
  --query 'RouteTable.RouteTableId' \
  --output text

# Add internet route
aws ec2 create-route \
  --route-table-id rtb-0627debf18341eb11 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0270baf826dd1daf0

# Associate route table with public subnet
aws ec2 associate-route-table \
  --route-table-id rtb-0627debf18341eb11 \
  --subnet-id subnet-0a689b36076f85b9d
```

#### Outputs

| Resource | Value |
|---|---|
| Public Subnet ID | `subnet-0a689b36076f85b9d` |
| Public Route Table | `rtb-0627debf18341eb11` |
| RT Association ID | `rtbassoc-0e531c8b89f31fcf4` |

**Screenshot: Public subnet creation and route table association**

> ![Public Subnet](screenshots/04-public-subnet-routing.png)

---

### Phase 3: NAT Instance Security Group

Create a dedicated security group that allows all inbound traffic from the private subnet CIDR and SSH access for management.

```bash
# Create security group
aws ec2 create-security-group \
  --group-name devops-nat-sg \
  --description "NAT instance SG - allows forwarding from private subnet" \
  --vpc-id vpc-01b75a856c993de09 \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=devops-nat-sg}]' \
  --query 'GroupId' \
  --output text

# Allow all traffic from private subnet
aws ec2 authorize-security-group-ingress \
  --group-id sg-0ebad42cff3626d27 \
  --protocol all \
  --cidr 10.1.1.0/24

# Allow SSH from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-0ebad42cff3626d27 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

#### Security Group Rules Summary

| Direction | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| Inbound | All | All | `10.1.1.0/24` | Allow private EC2 traffic to route through NAT |
| Inbound | TCP | 22 | `0.0.0.0/0` | SSH management access |
| Outbound | All | All | `0.0.0.0/0` | Allow NAT to forward to internet (default rule) |

**Screenshot: Security group creation and ingress rules**

> ![Security Group](screenshots/05-security-group-rules.png)

---

### Phase 4: NAT Instance Provisioning

Find the latest Amazon Linux 2 AMI and launch the NAT instance in the public subnet.

```bash
# Get latest Amazon Linux 2 AMI
aws ec2 describe-images \
  --owners 137112412989 \
  --filters \
    'Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2' \
    'Name=state,Values=available' \
    'Name=virtualization-type,Values=hvm' \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text

# Launch NAT instance
aws ec2 run-instances \
  --image-id ami-0199fa5fada510433 \
  --instance-type t2.micro \
  --subnet-id subnet-0a689b36076f85b9d \
  --security-group-ids sg-0ebad42cff3626d27 \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-nat-instance}]' \
  --count 1 \
  --query 'Instances[0].InstanceId' \
  --output text

# Wait for running state
aws ec2 wait instance-running \
  --instance-ids i-08b3c8cae76332623
```

#### Instance Details

| Attribute | Value |
|---|---|
| Instance ID | `i-08b3c8cae76332623` |
| AMI | `ami-0199fa5fada510433` (Amazon Linux 2) |
| Type | `t2.micro` |
| Private IP | `10.1.2.164` |
| Public IP | `44.211.188.254` |
| State | `running` |

**Screenshot: NAT instance launch and running state confirmation**

> ![NAT Instance Running](screenshots/06-nat-instance-running.png)

---

### Phase 5: NAT Instance OS Configuration

Two mandatory configuration steps: disable the AWS-level Source/Destination Check, then configure the OS for IP forwarding and NAT masquerading.

#### Step 5.1: Disable Source/Destination Check

By default, EC2 drops packets not addressed directly to the instance. This must be disabled for NAT forwarding to work.

```bash
aws ec2 modify-instance-attribute \
  --instance-id i-08b3c8cae76332623 \
  --no-source-dest-check

# Verify -- must return false
aws ec2 describe-instances \
  --instance-ids i-08b3c8cae76332623 \
  --query 'Reservations[0].Instances[0].SourceDestCheck'
```

**Screenshot: Source/Destination Check disabled**

> ![Source Dest Check](screenshots/07-source-dest-check-disabled.png)

#### Step 5.2: Access the Instance

No key pair was attached at launch. SSM Agent was not registered on the instance. Resolution: generate a temporary key pair and inject it using **EC2 Instance Connect** (valid for 60 seconds).

```bash
# Generate temporary key pair
ssh-keygen -t rsa -f /tmp/nat-temp-key -N ""

# Push public key via EC2 Instance Connect
aws ec2-instance-connect send-ssh-public-key \
  --instance-id i-08b3c8cae76332623 \
  --availability-zone us-east-1a \
  --instance-os-user ec2-user \
  --ssh-public-key file:///tmp/nat-temp-key.pub

# SSH in immediately (within 60 seconds)
ssh -i /tmp/nat-temp-key \
  -o StrictHostKeyChecking=no \
  ec2-user@44.211.188.254
```

**Screenshot: EC2 Instance Connect key injection and SSH session**

> ![EC2 Instance Connect](screenshots/08-ec2-instance-connect-ssh.png)

#### Step 5.3: Configure IP Forwarding and iptables

```bash
# Enable IP forwarding immediately
sudo sysctl -w net.ipv4.ip_forward=1

# Make persistent across reboots
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf

# Verify -- must return 1
cat /proc/sys/net/ipv4/ip_forward

# Add NAT masquerade rule on eth0
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Allow all forwarded traffic
sudo iptables -A FORWARD -j ACCEPT

# Save rules (Amazon Linux 2 uses iptables-save, not service iptables save)
sudo iptables-save | sudo tee /etc/sysconfig/iptables

# Verify MASQUERADE rule
sudo iptables -t nat -L POSTROUTING -n -v

exit
```

> **Note:** `sudo service iptables save` fails on Amazon Linux 2 because it uses `systemctl`. Use `iptables-save | tee` instead to persist rules.

**Screenshot: IP forwarding enabled and iptables MASQUERADE rule confirmed**

> ![iptables Configuration](screenshots/09-iptables-masquerade-configured.png)

---

### Phase 6: Private Route Table Update

Direct all outbound traffic from the private subnet through the NAT Instance.

```bash
# Verify no existing 0.0.0.0/0 route (only local VPC route present)
aws ec2 describe-route-tables \
  --route-table-ids rtb-0ec8f8b9133961a6e \
  --query 'RouteTables[0].Routes'

# Add default route pointing to NAT instance
aws ec2 create-route \
  --route-table-id rtb-0ec8f8b9133961a6e \
  --destination-cidr-block 0.0.0.0/0 \
  --instance-id i-08b3c8cae76332623

# Verify route is active
aws ec2 describe-route-tables \
  --route-table-ids rtb-0ec8f8b9133961a6e \
  --query 'RouteTables[0].Routes[?DestinationCidrBlock==`0.0.0.0/0`]'
```

#### Expected Output

```json
[
    {
        "DestinationCidrBlock": "0.0.0.0/0",
        "InstanceId": "i-08b3c8cae76332623",
        "NetworkInterfaceId": "eni-082840082a896ca3b",
        "Origin": "CreateRoute",
        "State": "active"
    }
]
```

**Screenshot: Private route table updated with NAT instance route**

> ![Private Route Table](screenshots/10-private-route-table-updated.png)

---

### Phase 7: Verification

Wait up to 90 seconds for the private EC2 cron job to execute, then confirm the file appears in S3.

```bash
aws s3 ls s3://devops-nat-17499/
```

#### Result

```
2026-03-03 22:57:03          0 devops-test.txt
```

**`devops-test.txt` is present in the S3 bucket. Task complete.**

**Screenshot: S3 bucket showing devops-test.txt upload confirmation**

> ![S3 Verification](screenshots/11-s3-file-verified.png)

---

## Resource Summary

| Resource | Name | ID |
|---|---|---|
| Internet Gateway | `devops-igw` | `igw-0270baf826dd1daf0` |
| Public Subnet | `devops-pub-subnet` | `subnet-0a689b36076f85b9d` |
| Public Route Table | `devops-pub-rt` | `rtb-0627debf18341eb11` |
| Security Group | `devops-nat-sg` | `sg-0ebad42cff3626d27` |
| NAT Instance | `devops-nat-instance` | `i-08b3c8cae76332623` |
| Private Route | `0.0.0.0/0` | `-> i-08b3c8cae76332623 (active)` |

---

## Troubleshooting

### devops-test.txt does not appear in S3 after 3+ minutes

Work through each check in order:

**Check 1: NAT instance is running**
```bash
aws ec2 describe-instances \
  --instance-ids i-08b3c8cae76332623 \
  --query 'Reservations[0].Instances[0].State.Name'
# Must return: running
```

**Check 2: Source/Destination Check is disabled**
```bash
aws ec2 describe-instances \
  --instance-ids i-08b3c8cae76332623 \
  --query 'Reservations[0].Instances[0].SourceDestCheck'
# Must return: false
```

**Check 3: Private route table has active 0.0.0.0/0 route**
```bash
aws ec2 describe-route-tables \
  --route-table-ids rtb-0ec8f8b9133961a6e \
  --query 'RouteTables[0].Routes[?DestinationCidrBlock==`0.0.0.0/0`]'
# Must show State=active and InstanceId=i-08b3c8cae76332623
```

**Check 4: IP forwarding and iptables inside NAT instance**
```bash
# Re-inject key and SSH in
aws ec2-instance-connect send-ssh-public-key \
  --instance-id i-08b3c8cae76332623 \
  --availability-zone us-east-1a \
  --instance-os-user ec2-user \
  --ssh-public-key file:///tmp/nat-temp-key.pub

ssh -i /tmp/nat-temp-key -o StrictHostKeyChecking=no ec2-user@44.211.188.254

cat /proc/sys/net/ipv4/ip_forward          # Must return 1
sudo iptables -t nat -L POSTROUTING -n -v  # Must show MASQUERADE rule on eth0
```

### Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Source/Dest Check not disabled | Packets dropped silently at NAT instance | `modify-instance-attribute --no-source-dest-check` |
| `service iptables save` used on AL2 | Rules lost after reboot | Use `iptables-save | tee /etc/sysconfig/iptables` |
| Public subnet in different AZ than private | Routing asymmetry issues | Recreate subnet in matching AZ |
| No IGW attached to VPC | No outbound path from public subnet | Create and attach IGW before creating route |
| Used `replace-route` when `create-route` needed | Error or no-op | Check existing routes first with `describe-route-tables` |

---

## Cost Considerations

| Component | NAT Gateway | NAT Instance (this solution) |
|---|---|---|
| Hourly charge | ~$0.045/hr | EC2 t2.micro (~$0.0116/hr or free tier) |
| Data processing | $0.045/GB | None |
| Availability | Managed, highly available | Single instance (SPOF) |
| Management overhead | None | OS patching, iptables management |

For low-traffic, non-critical private workloads, a NAT Instance provides significant cost savings over a NAT Gateway.

---

## Security Considerations

* **Restrict SSH inbound** -- The SG currently allows SSH from `0.0.0.0/0`. In production, restrict to a known bastion IP or use EC2 Instance Connect exclusively and remove the SSH rule entirely.
* **Key pair hygiene** -- The temporary key at `/tmp/nat-temp-key` should be deleted after use. EC2 Instance Connect keys expire after 60 seconds server-side.
* **NAT Instance as SPOF** -- A single NAT Instance has no built-in redundancy. For production, consider a second instance in another AZ or auto-recovery via CloudWatch alarms.
* **iptables persistence** -- Rules are saved to `/etc/sysconfig/iptables`. Verify they reload correctly after any instance stop/start cycle.
* **Source/Destination Check** -- Disabling this is intentional and required for NAT. Do not re-enable it.

---

## References

* [AWS: NAT Instances](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_NAT_Instance.html)
* [AWS: EC2 Instance Connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Connect-using-EC2-Instance-Connect.html)
* [AWS: VPC Route Tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
* [Amazon Linux 2 iptables persistence](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-best-practices.html)

---


<img width="1029" height="368" alt="image" src="https://github.com/user-attachments/assets/17a31e91-1cf5-4400-bb16-8a71339ee61d" />
<img width="1034" height="563" alt="image" src="https://github.com/user-attachments/assets/024499c3-a7ca-4f8f-afbd-173c30705290" />
<img width="1039" height="372" alt="image" src="https://github.com/user-attachments/assets/aeb6b58b-ac93-47e9-bbdc-b8dc882ba036" />
<img width="1041" height="342" alt="image" src="https://github.com/user-attachments/assets/4c759efb-b3db-46d4-b517-790a1232f3eb" />
<img width="1031" height="461" alt="image" src="https://github.com/user-attachments/assets/92d20c8b-2416-41be-8fca-6e48eba926c2" />
<img width="1028" height="515" alt="image" src="https://github.com/user-attachments/assets/c137af92-d568-4b3d-a2f1-d7afae6a80db" />
<img width="1039" height="379" alt="image" src="https://github.com/user-attachments/assets/b4bce762-c0f4-45b0-8a77-f1039e50c865" />
<img width="1027" height="578" alt="image" src="https://github.com/user-attachments/assets/cb9d3203-c041-4a2a-9a0c-d0d811dd7916" />
<img width="1037" height="159" alt="image" src="https://github.com/user-attachments/assets/dfbf7bbb-84e2-45f6-a751-1d32dcf33e2c" />
<img width="1027" height="376" alt="image" src="https://github.com/user-attachments/assets/081a9d35-3fe7-4d98-885d-cd79b3f7b4e1" />
<img width="1039" height="371" alt="image" src="https://github.com/user-attachments/assets/4ef32740-796f-4706-80bb-513a7e81ce40" />
<img width="1036" height="416" alt="image" src="https://github.com/user-attachments/assets/5c5d5273-f36b-4e44-860e-5505c9ec0226" />
<img width="1035" height="849" alt="image" src="https://github.com/user-attachments/assets/8aebb185-4027-4f59-8d8c-a1906e0264c5" />
<img width="1039" height="330" alt="image" src="https://github.com/user-attachments/assets/dc163e1f-9b3c-4ef9-be15-2b8090343377" />
<img width="1036" height="251" alt="image" src="https://github.com/user-attachments/assets/e1f17fee-a0f6-4781-9e55-8697be80b553" />
<img width="1028" height="542" alt="image" src="https://github.com/user-attachments/assets/8e66ba25-8a28-4afe-b54a-bd2784fe7c7c" />
<img width="1028" height="578" alt="image" src="https://github.com/user-attachments/assets/e75c381d-edc8-45da-a913-80b6a8447c17" />
<img width="1034" height="515" alt="image" src="https://github.com/user-attachments/assets/67c638d8-20a6-45f8-8909-d7e36a523af9" />
<img width="1030" height="624" alt="image" src="https://github.com/user-attachments/assets/a41fbd96-f861-4b57-a314-b6455075e62f" />
<img width="1037" height="861" alt="image" src="https://github.com/user-attachments/assets/a8fda850-c06e-4ae7-a88d-996a912a77b5" />
<img width="1041" height="647" alt="image" src="https://github.com/user-attachments/assets/cba12543-ed69-4f09-8fe3-aa49e0a409b4" />
<img width="1035" height="461" alt="image" src="https://github.com/user-attachments/assets/b10862b0-2d97-4058-b020-2c5bd74683fd" />
<img width="1031" height="522" alt="image" src="https://github.com/user-attachments/assets/b8995f7b-e8a7-4e37-866f-ec549f1e9792" />
<img width="1034" height="744" alt="image" src="https://github.com/user-attachments/assets/95904de7-4ade-4a28-a209-c306ba8b89c5" />
<img width="1034" height="433" alt="image" src="https://github.com/user-attachments/assets/99dd37dc-b00c-4924-a44b-e2df870edc64" />
<img width="1035" height="300" alt="image" src="https://github.com/user-attachments/assets/da46efac-e725-4875-93fc-d15a9313cbb6" />
<img width="1035" height="578" alt="image" src="https://github.com/user-attachments/assets/a21d455f-e4ac-4bb1-975b-df4e60f5ba88" />
<img width="1027" height="600" alt="image" src="https://github.com/user-attachments/assets/7f0c80a0-2c27-4c2d-8441-fe0e46a8d856" />
<img width="1033" height="516" alt="image" src="https://github.com/user-attachments/assets/5a340d99-ef5f-4852-9517-754d96082427" />
<img width="1036" height="603" alt="image" src="https://github.com/user-attachments/assets/d2e76360-712b-4716-9f8f-e308c03b6217" />
<img width="1039" height="672" alt="image" src="https://github.com/user-attachments/assets/ba69f54e-566b-494d-8f9c-5bd2ecf39725" />
<img width="1033" height="746" alt="image" src="https://github.com/user-attachments/assets/b5e30675-05de-44f4-b0b6-cbe74ab06778" />
<img width="1036" height="836" alt="image" src="https://github.com/user-attachments/assets/25c19ed0-8f4a-4c7e-9152-724cf38a897d" />
<img width="1031" height="691" alt="image" src="https://github.com/user-attachments/assets/3ccd7d78-1827-49cd-9093-f62e17291172" />
<img width="1034" height="523" alt="image" src="https://github.com/user-attachments/assets/e01bd26b-50c7-41c2-be98-2a520f3b45c0" />

