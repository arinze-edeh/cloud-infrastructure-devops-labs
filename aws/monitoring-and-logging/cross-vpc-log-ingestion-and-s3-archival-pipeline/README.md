# AWS Secure Log Aggregation Pipeline: Private VPC to S3 via VPC Peering

> **Enterprise DevOps | AWS Infrastructure | Log Pipeline Automation**
> Securely aggregate system logs from a private EC2 instance across VPC boundaries into a centralized S3 bucket using VPC Peering, IAM roles, and automated cron-based pipelines.

---

## Table of Contents

* [Project Overview](#project-overview)
* [Architecture Diagram](#architecture-diagram)
* [Problem Statement](#problem-statement)
* [Solution Summary](#solution-summary)
* [Prerequisites](#prerequisites)
* [Infrastructure Provisioned](#infrastructure-provisioned)
* [Step-by-Step Implementation](#step-by-step-implementation)
  * [Step 1: Provision the Public VPC, Subnet, and Route Table](#step-1-provision-the-public-vpc-subnet-and-route-table)
  * [Step 2: Attach Internet Gateway and Enable Public Routing](#step-2-attach-internet-gateway-and-enable-public-routing)
  * [Step 3: Launch the Public EC2 Instance](#step-3-launch-the-public-ec2-instance)
  * [Step 4: Create and Attach IAM Role with S3 Permissions](#step-4-create-and-attach-iam-role-with-s3-permissions)
  * [Step 5: Create the Private S3 Log Bucket](#step-5-create-the-private-s3-log-bucket)
  * [Step 6: Establish VPC Peering Between Public and Private VPCs](#step-6-establish-vpc-peering-between-public-and-private-vpcs)
  * [Step 7: Update Security Groups and Retrieve Instance IPs](#step-7-update-security-groups-and-retrieve-instance-ips)
  * [Step 8: Distribute SSH Keys Across Instances](#step-8-distribute-ssh-keys-across-instances)
  * [Step 9: Install AWS CLI on the Public EC2 Instance](#step-9-install-aws-cli-on-the-public-ec2-instance)
  * [Step 10: Verify Log Files on the Private EC2 Instance](#step-10-verify-log-files-on-the-private-ec2-instance)
  * [Step 11: Configure Cron Job on Private EC2 to Push Logs to Public EC2](#step-11-configure-cron-job-on-private-ec2-to-push-logs-to-public-ec2)
  * [Step 12: Configure Cron Job on Public EC2 to Push Logs to S3](#step-12-configure-cron-job-on-public-ec2-to-push-logs-to-s3)
  * [Step 13: Validate VPC Peering Status](#step-13-validate-vpc-peering-status)
  * [Step 14: Confirm Log Delivery to S3](#step-14-confirm-log-delivery-to-s3)
* [Resource Reference Table](#resource-reference-table)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)
* [Troubleshooting](#troubleshooting)

---

## Project Overview

This project demonstrates a production-grade, secure log aggregation pipeline built entirely on AWS. A private EC2 instance (with no internet access) periodically ships its boot log (`/var/log/boots.log`) across a VPC Peering connection to a bastion-style public EC2 instance, which then relays it to a centralized, private S3 bucket using an IAM instance profile. The entire pipeline is automated with cron jobs and requires zero manual intervention after initial setup.

**Region:** `us-east-1`
**Key Pair:** `devops-key`
**OS:** Ubuntu 22.04 LTS (Jammy) on both instances

---

## Architecture Diagram

```
+-----------------------------+          VPC Peering (pcx-07ff722f053d64b7b)          +-------------------------------+
|      devops-priv-vpc        |<-------------------------------------------------->   |       devops-pub-vpc          |
|   CIDR: 10.10.0.0/16        |                                                        |    CIDR: 10.1.0.0/16          |
|                             |                                                        |                               |
|  +----------------------+   |   SCP /var/log/boots.log every 1 min                  |  +------------------------+   |
|  |  devops-priv-ec2     |---+------------------------------------------------------->|  devops-pub-ec2          |   |
|  |  10.10.1.235 (priv)  |   |                                                        |  10.1.1.85 (priv)        |   |
|  |  No Internet Access  |   |                                                        |  3.235.170.246 (pub)     |   |
|  +----------------------+   |                                                        |  IAM: devops-s3-role     |   |
|                             |                                                        |  +----+                  |   |
+-----------------------------+                                                        |  | aws s3 cp every 1 min |   |
                                                                                       |  +----+                  |   |
                                                                                       +-------------------------------+
                                                                                                     |
                                                                                                     v
                                                                              +----------------------------------+
                                                                              |   devops-s3-logs-25406           |
                                                                              |   Path: devops-priv-vpc/boot/    |
                                                                              |         boots.log                |
                                                                              |   Access: Private (no public ACL)|
                                                                              +----------------------------------+
```

---

## Problem Statement

The Nautilus DevOps team required a secure, automated mechanism to centralize system logs from an isolated private EC2 instance (`devops-priv-ec2`) that has no direct internet access. The log data needed to flow:

1. Out of the private VPC securely (no NAT Gateway required)
2. Through a controlled, routed channel to a public EC2 instance
3. Into a private S3 bucket for long-term storage and audit trail

The solution had to be cost-efficient, operationally automated, and follow AWS security best practices.

---

## Solution Summary

| Layer | Component | Purpose |
|---|---|---|
| Networking | VPC Peering | Private-to-public VPC communication |
| Compute | Public EC2 (devops-pub-ec2) | Relay/bastion node for log forwarding |
| IAM | devops-s3-role | Grants EC2 PutObject access to S3 |
| Storage | S3 (devops-s3-logs-25406) | Centralized, private log archive |
| Automation | Cron (both EC2s) | Minute-level log shipping pipeline |
| Security | SG rules + no public S3 ACL | Least-privilege access enforcement |

---

## Prerequisites

* AWS CLI configured on the client host (`aws-client`) with sufficient IAM permissions
* Existing private infrastructure:
  * VPC: `devops-priv-vpc` (CIDR: `10.10.0.0/16`)
  * Subnet: `devops-priv-subnet`
  * Route Table: `devops-priv-rt`
  * EC2: `devops-priv-ec2` (Ubuntu, tagged accordingly)
  * Key Pair: `devops-key` (PEM file at `/root/.ssh/devops-key.pem`)
* Target AWS Region: `us-east-1`
* The private EC2 instance must have `/var/log/boots.log` present

---

## Infrastructure Provisioned

| Resource | Name / ID | Value |
|---|---|---|
| Public VPC | devops-pub-vpc | `vpc-00c2cdb99aaab238e` / `10.1.0.0/16` |
| Public Subnet | devops-pub-subnet | `subnet-0d8b57cf9b154a022` / `10.1.1.0/24` |
| Public Route Table | devops-pub-rt | `rtb-0bb48479c56abb39d` |
| Internet Gateway | -- | `igw-0ec6e010f95525cfd` |
| Public Security Group | devops-pub-sg | `sg-0e7dff2b2dbff34fd` |
| Public EC2 | devops-pub-ec2 | `i-05a7403454c019495` |
| Public EC2 Public IP | -- | `3.235.170.246` |
| Public EC2 Private IP | -- | `10.1.1.85` |
| Private EC2 Private IP | devops-priv-ec2 | `10.10.1.235` |
| IAM Role | devops-s3-role | `arn:aws:iam::284304506227:role/devops-s3-role` |
| S3 Bucket | devops-s3-logs-25406 | `us-east-1`, private |
| VPC Peering | devops-vpc-peering | `pcx-07ff722f053d64b7b` |
| Private VPC | devops-priv-vpc | `vpc-09f0f241ccc7851e0` / `10.10.0.0/16` |

---

## Step-by-Step Implementation

---

### Step 1: Provision the Public VPC, Subnet, and Route Table

Create the public VPC with CIDR `10.1.0.0/16`, a subnet (`10.1.1.0/24`), and a dedicated route table. Associate the route table with the subnet immediately.

```bash
PUB_VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.1.0.0/16 \
  --region us-east-1 \
  --query 'Vpc.VpcId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_VPC_ID \
  --tags Key=Name,Value=devops-pub-vpc \
  --region us-east-1 && \
PUB_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id $PUB_VPC_ID \
  --cidr-block 10.1.1.0/24 \
  --region us-east-1 \
  --query 'Subnet.SubnetId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_SUBNET_ID \
  --tags Key=Name,Value=devops-pub-subnet \
  --region us-east-1 && \
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $PUB_VPC_ID \
  --region us-east-1 \
  --query 'RouteTable.RouteTableId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_RT_ID \
  --tags Key=Name,Value=devops-pub-rt \
  --region us-east-1 && \
aws ec2 associate-route-table \
  --route-table-id $PUB_RT_ID \
  --subnet-id $PUB_SUBNET_ID \
  --region us-east-1 && \
echo "PUB_VPC_ID=$PUB_VPC_ID  PUB_SUBNET_ID=$PUB_SUBNET_ID  PUB_RT_ID=$PUB_RT_ID"
```

**Expected Output:**
```
{
    "AssociationId": "rtbassoc-0523e7ee3745af09b",
    "AssociationState": { "State": "associated" }
}
PUB_VPC_ID=vpc-00c2cdb99aaab238e  PUB_SUBNET_ID=subnet-0d8b57cf9b154a022  PUB_RT_ID=rtb-0bb48479c56abb39d
```

> **Screenshot**

<img width="1032" height="837" alt="image" src="https://github.com/user-attachments/assets/391530a9-3e3d-4247-96d0-0038cbc48673" />

> `AWS Console > VPC Dashboard showing devops-pub-vpc, devops-pub-subnet, and devops-pub-rt created with correct CIDRs and association state`

---

### Step 2: Attach Internet Gateway and Enable Public Routing

Create an Internet Gateway, attach it to the public VPC, add a default route (`0.0.0.0/0`) pointing to the IGW in the public route table, and enable auto-assign public IPs on the public subnet.

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --region us-east-1 \
  --query 'InternetGateway.InternetGatewayId' \
  --output text) && \
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $PUB_VPC_ID \
  --region us-east-1 && \
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID \
  --region us-east-1 && \
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_ID \
  --map-public-ip-on-launch \
  --region us-east-1 && \
echo "IGW_ID=$IGW_ID"
```

**Expected Output:**
```
{ "Return": true }
IGW_ID=igw-0ec6e010f95525cfd
```

> **Screenshot**

<img width="1028" height="866" alt="image" src="https://github.com/user-attachments/assets/8d064d34-1dd3-407b-8919-f8c779b4406c" />

---

### Step 3: Launch the Public EC2 Instance

Retrieve the key pair name from the existing private instance, fetch the latest Ubuntu 22.04 AMI, create a security group permitting SSH (port 22) from anywhere, launch a `t2.micro` instance into the public subnet, tag it, and wait until it is in the `running` state.

```bash
KEY_NAME=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-priv-ec2" \
  --query 'Reservations[0].Instances[0].KeyName' \
  --output text \
  --region us-east-1) && \
UBUNTU_AMI=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' \
  --output text \
  --region us-east-1) && \
PUB_SG_ID=$(aws ec2 create-security-group \
  --group-name devops-pub-sg \
  --description "SG for devops-pub-ec2" \
  --vpc-id $PUB_VPC_ID \
  --region us-east-1 \
  --query 'GroupId' \
  --output text) && \
aws ec2 authorize-security-group-ingress \
  --group-id $PUB_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0 \
  --region us-east-1 && \
PUB_EC2_ID=$(aws ec2 run-instances \
  --image-id $UBUNTU_AMI \
  --instance-type t2.micro \
  --key-name "$KEY_NAME" \
  --subnet-id $PUB_SUBNET_ID \
  --security-group-ids $PUB_SG_ID \
  --region us-east-1 \
  --query 'Instances[0].InstanceId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_EC2_ID \
  --tags Key=Name,Value=devops-pub-ec2 \
  --region us-east-1 && \
aws ec2 wait instance-running \
  --instance-ids $PUB_EC2_ID \
  --region us-east-1 && \
echo "KEY_NAME=$KEY_NAME  UBUNTU_AMI=$UBUNTU_AMI  PUB_SG_ID=$PUB_SG_ID  PUB_EC2_ID=$PUB_EC2_ID"
```

**Expected Output:**
```
KEY_NAME=devops-key  UBUNTU_AMI=ami-00de3875b03809ec5  PUB_SG_ID=sg-0e7dff2b2dbff34fd  PUB_EC2_ID=i-05a7403454c019495
```

> **Screenshots**

<img width="1031" height="841" alt="image" src="https://github.com/user-attachments/assets/fd60add0-c99c-4d51-91d9-eefe5a917e52" />
<img width="1034" height="865" alt="image" src="https://github.com/user-attachments/assets/d21d4f15-e399-4b28-bf2f-530141faef9d" />

---

### Step 4: Create and Attach IAM Role with S3 Permissions

Create an IAM role with an EC2 trust policy, attach the `AmazonS3FullAccess` managed policy, create an instance profile, add the role to the profile, and associate it with the public EC2 instance. A 15-second sleep is included to allow IAM propagation before association.

```bash
aws iam create-role \
  --role-name devops-s3-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}' && \
aws iam attach-role-policy \
  --role-name devops-s3-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess && \
aws iam create-instance-profile \
  --instance-profile-name devops-s3-role && \
aws iam add-role-to-instance-profile \
  --instance-profile-name devops-s3-role \
  --role-name devops-s3-role && \
sleep 15 && \
aws ec2 associate-iam-instance-profile \
  --instance-id $PUB_EC2_ID \
  --iam-instance-profile Name=devops-s3-role \
  --region us-east-1 && \
echo "IAM role attached"
```

**Expected Output:**
```json
{
    "IamInstanceProfileAssociation": {
        "AssociationId": "iip-assoc-06957e99034e284e9",
        "InstanceId": "i-05a7403454c019495",
        "State": "associating"
    }
}
IAM role attached
```

> **Screenshots**

<img width="1030" height="853" alt="image" src="https://github.com/user-attachments/assets/08a8cd81-0f55-4f52-88da-0cb6113bf1f7" />
<img width="1028" height="867" alt="image" src="https://github.com/user-attachments/assets/ea8e1305-e144-4e9c-8cc2-24626e759ca2" />

---

### Step 5: Create the Private S3 Log Bucket

Create the S3 bucket `devops-s3-logs-25406` in `us-east-1` and immediately block all public access to enforce private-only access via IAM.

```bash
aws s3api create-bucket \
  --bucket devops-s3-logs-25406 \
  --region us-east-1 && \
aws s3api put-public-access-block \
  --bucket devops-s3-logs-25406 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true && \
echo "S3 bucket ready: devops-s3-logs-25406"
```

**Expected Output:**
```
{ "Location": "/devops-s3-logs-25406" }
S3 bucket ready: devops-s3-logs-25406
```

> **Screenshot**

<img width="1024" height="862" alt="image" src="https://github.com/user-attachments/assets/e3ce40aa-58e1-460e-b7c1-fe17febc16f8" />


---

### Step 6: Establish VPC Peering Between Public and Private VPCs

Retrieve the private VPC ID and CIDR, initiate a peering connection from the public VPC to the private VPC, accept it, then add bidirectional routes: the private route table routes `10.1.0.0/16` through the peering connection, and the public route table routes `10.10.0.0/16` through it as well.

```bash
PRIV_VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=devops-priv-vpc" \
  --query 'Vpcs[0].VpcId' \
  --output text \
  --region us-east-1) && \
PRIV_VPC_CIDR=$(aws ec2 describe-vpcs \
  --vpc-ids $PRIV_VPC_ID \
  --query 'Vpcs[0].CidrBlock' \
  --output text \
  --region us-east-1) && \
PEERING_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $PUB_VPC_ID \
  --peer-vpc-id $PRIV_VPC_ID \
  --region us-east-1 \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PEERING_ID \
  --tags Key=Name,Value=devops-vpc-peering \
  --region us-east-1 && \
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1 && \
PRIV_RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=devops-priv-rt" \
  --query 'RouteTables[0].RouteTableId' \
  --output text \
  --region us-east-1) && \
aws ec2 create-route \
  --route-table-id $PRIV_RT_ID \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1 && \
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block $PRIV_VPC_CIDR \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1 && \
echo "PRIV_VPC_ID=$PRIV_VPC_ID  PRIV_VPC_CIDR=$PRIV_VPC_CIDR  PEERING_ID=$PEERING_ID  PRIV_RT_ID=$PRIV_RT_ID"
```

**Expected Output:**
```
PRIV_VPC_ID=vpc-09f0f241ccc7851e0  PRIV_VPC_CIDR=10.10.0.0/16  PEERING_ID=pcx-07ff722f053d64b7b  PRIV_RT_ID=rtb-0b2873c0c9a0cb96c
```

> **Screenshots**

<img width="1028" height="844" alt="image" src="https://github.com/user-attachments/assets/ab08b5b2-a0a5-4bd6-a988-fcd8e93738fb" />
<img width="1032" height="860" alt="image" src="https://github.com/user-attachments/assets/1bc4f0b6-0558-4b19-bedd-d8f9a6c4d28c" />
<img width="1040" height="871" alt="image" src="https://github.com/user-attachments/assets/3517b524-fde7-432b-a6b9-3834f614cee6" />

---

### Step 7: Update Security Groups and Retrieve Instance IPs

Add an inbound SSH rule to the private EC2 security group allowing traffic only from the public VPC CIDR (`10.1.0.0/16`). Then capture the public and private IPs of the public EC2 and the private IP of the private EC2.

```bash
PRIV_SG_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-priv-ec2" \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
  --output text \
  --region us-east-1) && \
aws ec2 authorize-security-group-ingress \
  --group-id $PRIV_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 10.1.0.0/16 \
  --region us-east-1 && \
PUB_EC2_PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $PUB_EC2_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text \
  --region us-east-1) && \
PUB_EC2_PRIVATE_IP=$(aws ec2 describe-instances \
  --instance-ids $PUB_EC2_ID \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text \
  --region us-east-1) && \
PRIV_EC2_PRIVATE_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-priv-ec2" \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text \
  --region us-east-1) && \
echo "PUB_PUBLIC=$PUB_EC2_PUBLIC_IP  PUB_PRIVATE=$PUB_EC2_PRIVATE_IP  PRIV_PRIVATE=$PRIV_EC2_PRIVATE_IP"
```

**Expected Output:**
```
PUB_PUBLIC=3.235.170.246  PUB_PRIVATE=10.1.1.85  PRIV_PRIVATE=10.10.1.235
```

> **Screenshot Placeholder**
> `[SCREENSHOT: AWS Console > EC2 > Security Groups > devops-priv-ec2 SG showing inbound rule: TCP 22 from 10.1.0.0/16 only]`

---

### Step 8: Distribute SSH Keys Across Instances

Copy the PEM key to the public EC2, then use SSH agent forwarding to relay the key to the private EC2. Also add the public key derived from the PEM to `authorized_keys` on the public instance to enable SCP from the private EC2 back to it.

```bash
scp -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  /root/.ssh/devops-key.pem \
  ubuntu@$PUB_EC2_PUBLIC_IP:/home/ubuntu/.ssh/devops-key.pem && \
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP \
  "chmod 600 /home/ubuntu/.ssh/devops-key.pem" && \
eval $(ssh-agent) && \
ssh-add /root/.ssh/devops-key.pem && \
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  -A ubuntu@$PUB_EC2_PUBLIC_IP \
  "scp -o StrictHostKeyChecking=no /home/ubuntu/.ssh/devops-key.pem ubuntu@$PRIV_EC2_PRIVATE_IP:/home/ubuntu/.ssh/devops-key.pem && \
  ssh -o StrictHostKeyChecking=no ubuntu@$PRIV_EC2_PRIVATE_IP 'chmod 600 /home/ubuntu/.ssh/devops-key.pem'" && \
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP \
  "cat /home/ubuntu/.ssh/devops-key.pem | ssh-keygen -y -f /dev/stdin >> /home/ubuntu/.ssh/authorized_keys && echo 'Public key added'" && \
echo "Keys distributed"
```

**Expected Output:**
```
devops-key.pem  100% 1675   16.6KB/s   00:00
Agent pid 1534
Identity added: /root/.ssh/devops-key.pem
Warning: Permanently added '10.10.1.235' (ED25519) to the list of known hosts.
Public key added
Keys distributed
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output showing successful SCP of devops-key.pem to both public and private EC2, and "Keys distributed" confirmation]`

---

### Step 9: Install AWS CLI on the Public EC2 Instance

SSH into the public EC2 and install the AWS CLI v2 using the official installer. The public instance needs AWS CLI to execute `aws s3 cp` from the cron job in Step 12.

```bash
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP \
  "sudo apt-get update -y -qq && \
  sudo apt-get install -y unzip curl -qq && \
  curl -s https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip && \
  unzip -q awscliv2.zip && \
  sudo ./aws/install && \
  /usr/local/bin/aws --version && \
  echo 'AWS CLI installed'"
```

**Expected Output:**
```
aws-cli/2.34.16 Python/3.14.3 Linux/6.8.0-1050-aws exe/x86_64.ubuntu.22
AWS CLI installed
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing aws --version output on devops-pub-ec2 confirming aws-cli/2.x installation]`

---

### Step 10: Verify Log Files on the Private EC2 Instance

SSH from the client through the public EC2 into the private EC2 (jump host pattern using `-A` agent forwarding) and verify that the target log files (`/var/log/boots.log`, `/var/log/syslog`, `/var/log/kern.log`, `/var/log/auth.log`) exist and are non-empty.

```bash
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  -A ubuntu@$PUB_EC2_PUBLIC_IP \
  "ssh -o StrictHostKeyChecking=no ubuntu@$PRIV_EC2_PRIVATE_IP \
  'ls -lh /var/log/boot* /var/log/syslog /var/log/kern.log /var/log/auth.log 2>&1 && \
  echo \"=== SYSLOG HEAD ===\" && sudo head -3 /var/log/syslog 2>&1 && \
  echo \"=== KERN HEAD ===\" && sudo head -3 /var/log/kern.log 2>&1 && \
  echo \"=== BOOTS.LOG check ===\" && ls -la /var/log/boots.log 2>&1'"
```

**Expected Output:**
```
-rw-r----- 1 syslog adm   5.2K Mar 25 11:49 /var/log/auth.log
-rw-r--r-- 1 root   root    23 Mar 25 11:36 /var/log/boots.log
-rw-r----- 1 syslog adm   54K Mar 25 11:35 /var/log/kern.log
-rw-r----- 1 syslog adm  135K Mar 25 11:49 /var/log/syslog
=== SYSLOG HEAD ===
Mar 25 11:35:41 ip-10-10-1-235 systemd[1]: Mounted Huge Pages File System.
...
=== BOOTS.LOG check ===
-rw-r--r-- 1 root root 23 Mar 25 11:36 /var/log/boots.log
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing ls -lh output from devops-priv-ec2 confirming /var/log/boots.log exists at 23 bytes with correct permissions]`

---

### Step 11: Configure Cron Job on Private EC2 to Push Logs to Public EC2

Install a crontab entry on the private EC2 that runs every minute, using `scp` with the key pair to copy `/var/log/boots.log` over the peering connection to the public EC2's home directory.

```bash
ssh -i /root/.ssh/devops-key.pem -o StrictHostKeyChecking=no -A ubuntu@$PUB_EC2_PUBLIC_IP \
  "ssh -o StrictHostKeyChecking=no ubuntu@$PRIV_EC2_PRIVATE_IP \
  'echo \"* * * * * scp -i /home/ubuntu/.ssh/devops-key.pem -o StrictHostKeyChecking=no /var/log/boots.log ubuntu@$PUB_EC2_PRIVATE_IP:/home/ubuntu/boots.log\" | crontab -'"
```

**What this does:**

* Cron schedule: `* * * * *` (every minute)
* Source: `/var/log/boots.log` on `devops-priv-ec2` (`10.10.1.235`)
* Destination: `/home/ubuntu/boots.log` on `devops-pub-ec2` (`10.1.1.85`)
* Transport: SCP over VPC Peering (no internet traversal)

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing crontab -l output on devops-priv-ec2 confirming the SCP cron entry is active]`

---

### Step 12: Configure Cron Job on Public EC2 to Push Logs to S3

Install a crontab entry on the public EC2 that runs every minute, using the AWS CLI (backed by the attached IAM role) to sync the received `boots.log` to the target S3 path.

```bash
ssh -i /root/.ssh/devops-key.pem -o StrictHostKeyChecking=no ubuntu@$PUB_EC2_PUBLIC_IP \
  "echo \"* * * * * /usr/local/bin/aws s3 cp /home/ubuntu/boots.log s3://devops-s3-logs-25406/devops-priv-vpc/boot/boots.log\" | crontab -"
```

**What this does:**

* Cron schedule: `* * * * *` (every minute)
* Source: `/home/ubuntu/boots.log` on `devops-pub-ec2`
* Destination: `s3://devops-s3-logs-25406/devops-priv-vpc/boot/boots.log`
* Auth: IAM instance profile `devops-s3-role` (no static credentials required)

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing crontab -l output on devops-pub-ec2 confirming the aws s3 cp cron entry is active with full S3 path]`

---

### Step 13: Validate VPC Peering Status

Confirm that the VPC peering connection is in `active` state before declaring the pipeline operational.

```bash
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids $PEERING_ID \
  --query 'VpcPeeringConnections[0].Status.Code'
```

**Expected Output:**
```
"active"
```

> **Screenshot Placeholder**
> `[SCREENSHOT: AWS Console > VPC > Peering Connections > devops-vpc-peering showing Status: Active with both VPC IDs and CIDRs visible]`

---

### Step 14: Confirm Log Delivery to S3

Wait approximately 1 minute for the cron pipeline to complete a full cycle, then list the S3 bucket path to confirm the file has landed at the correct location.

```bash
aws s3 ls s3://devops-s3-logs-25406/devops-priv-vpc/boot/
```

**Expected Output:**
```
2026-03-25 12:02:03         23 boots.log
```

**Pipeline confirmed operational.** The 23-byte `boots.log` file from the private instance is now centrally stored in S3 under the path `devops-priv-vpc/boot/boots.log`.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing aws s3 ls output with boots.log (23 bytes) at s3://devops-s3-logs-25406/devops-priv-vpc/boot/]`

> **Screenshot Placeholder**
> `[SCREENSHOT: AWS Console > S3 > devops-s3-logs-25406 > devops-priv-vpc/boot/ showing boots.log object with Last Modified timestamp]`

---

## Resource Reference Table

| Variable | Value |
|---|---|
| `PUB_VPC_ID` | `vpc-00c2cdb99aaab238e` |
| `PUB_SUBNET_ID` | `subnet-0d8b57cf9b154a022` |
| `PUB_RT_ID` | `rtb-0bb48479c56abb39d` |
| `IGW_ID` | `igw-0ec6e010f95525cfd` |
| `PUB_SG_ID` | `sg-0e7dff2b2dbff34fd` |
| `PUB_EC2_ID` | `i-05a7403454c019495` |
| `PUB_EC2_PUBLIC_IP` | `3.235.170.246` |
| `PUB_EC2_PRIVATE_IP` | `10.1.1.85` |
| `PRIV_EC2_PRIVATE_IP` | `10.10.1.235` |
| `PRIV_VPC_ID` | `vpc-09f0f241ccc7851e0` |
| `PRIV_VPC_CIDR` | `10.10.0.0/16` |
| `PRIV_RT_ID` | `rtb-0b2873c0c9a0cb96c` |
| `PRIV_SG_ID` | `sg-034057aef886496b9` |
| `PEERING_ID` | `pcx-07ff722f053d64b7b` |
| `KEY_NAME` | `devops-key` |
| `UBUNTU_AMI` | `ami-00de3875b03809ec5` |
| S3 Bucket | `devops-s3-logs-25406` |
| S3 Log Path | `devops-priv-vpc/boot/boots.log` |

---

## Best Practices

### IAM and Security

* **Never use static AWS credentials on EC2.** This project uses an IAM instance profile (`devops-s3-role`) to grant S3 access. The AWS CLI on the public EC2 automatically uses the instance metadata endpoint to retrieve temporary credentials. This is the gold standard for EC2-to-S3 communication.
* **Scope security group rules tightly.** The private EC2 security group allows SSH only from the public VPC CIDR (`10.1.0.0/16`), not from the public internet. The public EC2 SG allows SSH from anywhere for demo purposes; in production, restrict to a known management CIDR or use AWS Systems Manager Session Manager instead.
* **Block all public access on S3 buckets by default.** The `put-public-access-block` call in Step 5 ensures that even if someone accidentally applies a public ACL, it will be ignored.
* **Use `AmazonS3FullAccess` only for labs.** In production, replace with a custom policy scoped to a specific bucket and `s3:PutObject` action only. Follow least-privilege rigorously.

### Networking

* **VPC CIDR ranges must not overlap.** `10.1.0.0/16` (public) and `10.10.0.0/16` (private) are non-overlapping, which is a hard requirement for VPC Peering to function.
* **Both route tables must be updated for bidirectional peering.** A common mistake is adding only one side. This project adds the return route to `devops-priv-rt` and the forward route to `devops-pub-rt`.
* **VPC Peering does not support transitive routing.** If you later add a third VPC, it cannot route through this peering. Use AWS Transit Gateway for hub-and-spoke topologies.

### SSH Key Management

* **Use `chmod 600` on all PEM files immediately.** SSH clients will reject keys with open permissions. This is enforced in Step 8.
* **Use SSH agent forwarding (`-A`) instead of copying the key everywhere.** For production jump-host patterns, prefer agent forwarding to avoid key material sprawl. In this lab, the key is distributed to the private instance for use in the cron SCP job, which is acceptable for a controlled environment.
* **`StrictHostKeyChecking=no` is acceptable for ephemeral lab VMs.** In production, pre-populate `known_hosts` or use certificate-based SSH with AWS EC2 Instance Connect.

### Automation and Reliability

* **Use absolute paths in cron jobs.** The cron environment is minimal and does not inherit your shell PATH. This project correctly uses `/usr/local/bin/aws` instead of just `aws`.
* **Install AWS CLI v2 before setting cron.** Always verify the tool is present before the cron job runs for the first time. This project validates with `aws --version` in Step 9.
* **Add IAM propagation delay (`sleep 15`).** IAM changes are eventually consistent. The 15-second sleep before `associate-iam-instance-profile` prevents race conditions.
* **Tag all resources consistently.** Every resource created in this project is tagged with a `Name` so it can be identified in the console and filtered via CLI.

---

## Lessons Learned

* **VPC Peering requires route table updates on BOTH sides.** It is a common operational error to accept the peering and add only one route. The private instance will be unreachable from the public instance until the reverse route (`10.10.0.0/16` via the peering) is added to the public route table as well.

* **IAM roles propagate asynchronously.** The `sleep 15` guard is not optional in automation scripts. Without it, `associate-iam-instance-profile` may fail with a resource not found error because the role has not yet been fully registered by the IAM service.

* **The cron environment requires absolute binary paths.** A cron entry that calls `aws s3 cp` will silently fail because `aws` is not in the cron PATH. Using `/usr/local/bin/aws` resolved this entirely.

* **`authorized_keys` must be pre-populated for cron SCP to succeed.** The SCP job on the private instance runs non-interactively. If the public EC2's `authorized_keys` does not contain the corresponding public key, the SCP will fail with a host authentication error and produce no visible output. Step 8 handles this by deriving and appending the public key from the PEM file.

* **`StrictHostKeyChecking=no` is required for automated SSH/SCP.** Cron jobs cannot interact with interactive prompts. Without this flag, the first SCP attempt to a new host would hang indefinitely waiting for a user to confirm the host key.

* **S3 bucket names must be globally unique.** The numeric suffix (`-25406`) ensures uniqueness. In production, use a deterministic naming convention incorporating account ID or environment name.

* **VPC Peering does not replace NAT Gateway for internet access.** The private instance still has no internet access in this design. Peering only enables instance-to-instance communication between the two VPCs. If the private instance ever needs to reach external services, a NAT Gateway in the public VPC would be required.

* **The IAM instance profile (not just the role) must be created and attached.** A frequent mistake is creating the role and attaching the policy but forgetting to create the instance profile and add the role to it. EC2 does not accept roles directly; it requires an instance profile as the wrapper.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `aws s3 cp` fails on public EC2 | IAM role not attached or not yet propagated | Verify `aws sts get-caller-identity` on the EC2 returns the instance role ARN |
| SCP from private EC2 times out | Peering route missing or SG not updated | Check both route tables have peering routes; verify private SG allows TCP 22 from `10.1.0.0/16` |
| `boots.log` never appears in S3 | Cron not running or wrong binary path | Run `crontab -l` on public EC2; manually run the `aws s3 cp` command to test |
| SSH to private EC2 fails via jump host | `authorized_keys` not populated | Re-run the key distribution step from Step 8 |
| IAM `associate-iam-instance-profile` fails | Role not yet propagated | Add `sleep 15` or verify the instance profile exists with `aws iam get-instance-profile` |
| Peering status stuck in `pending-acceptance` | `accept-vpc-peering-connection` not called | Explicitly call `aws ec2 accept-vpc-peering-connection` as shown in Step 6 |

---








<img width="1080" height="861" alt="image" src="https://github.com/user-attachments/assets/9dd6b873-178c-4852-8bf6-48ce371c9fc7" />
<img width="1082" height="871" alt="image" src="https://github.com/user-attachments/assets/283a6b21-a826-4953-ad22-aac8aa760ed0" />
<img width="1081" height="848" alt="image" src="https://github.com/user-attachments/assets/19976361-e237-4ea2-ac39-e513eb673cd6" />
<img width="1079" height="863" alt="image" src="https://github.com/user-attachments/assets/a7e60128-588a-4a9d-9638-59fe16aff1aa" />
<img width="1078" height="862" alt="image" src="https://github.com/user-attachments/assets/f4c8b422-25f7-4a49-82ee-f6c8f5f348cc" />
<img width="1134" height="855" alt="image" src="https://github.com/user-attachments/assets/5fca679d-b771-451d-b568-60fc2f9cdd57" />
<img width="1130" height="862" alt="image" src="https://github.com/user-attachments/assets/8f75199b-87e5-403a-9111-2a6d0bacbaac" />
<img width="1049" height="857" alt="image" src="https://github.com/user-attachments/assets/7c8adb28-5c53-49eb-b170-f2e3aad26b2a" />









<img width="654" height="882" alt="image" src="https://github.com/user-attachments/assets/8779ceee-db81-4b33-a573-00e7c44a4e67" />


~ on ☁️  (us-east-1) ➜  PUB_VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.1.0.0/16 \
  --region us-east-1 \
  --query 'Vpc.VpcId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_VPC_ID \
  --tags Key=Name,Value=devops-pub-vpc \
  --region us-east-1 && \
PUB_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id $PUB_VPC_ID \
  --cidr-block 10.1.1.0/24 \
  --region us-east-1 \
  --query 'Subnet.SubnetId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_SUBNET_ID \
  --tags Key=Name,Value=devops-pub-subnet \
  --region us-east-1 && \
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $PUB_VPC_ID \
  --region us-east-1 \
  --query 'RouteTable.RouteTableId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_RT_ID \
  --tags Key=Name,Value=devops-pub-rt \
  --region us-east-1 && \
aws ec2 associate-route-table \
  --route-table-id $PUB_RT_ID \
  --subnet-id $PUB_SUBNET_ID \
  --region us-east-1 && \
echo "PUB_VPC_ID=$PUB_VPC_ID  PUB_SUBNET_ID=$PUB_SUBNET_ID  PUB_RT_ID=$PUB_RT_ID"
{
    "AssociationId": "rtbassoc-0523e7ee3745af09b",
    "AssociationState": {
        "State": "associated"
    }
}
PUB_VPC_ID=vpc-00c2cdb99aaab238e  PUB_SUBNET_ID=subnet-0d8b57cf9b154a022  PUB_RT_ID=rtb-0bb48479c56abb39d

~ on ☁️  (us-east-1) ➜  IGW_ID=$(aws ec2 create-internet-gateway \
  --region us-east-1 \
  --query 'InternetGateway.InternetGatewayId' \
  --output text) && \
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $PUB_VPC_ID \
  --region us-east-1 && \
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID \
  --region us-east-1 && \
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_ID \
  --map-public-ip-on-launch \
  --region us-east-1 && \
echo "IGW_ID=$IGW_ID"
{
    "Return": true
}
IGW_ID=igw-0ec6e010f95525cfd

~ on ☁️  (us-east-1) ➜  KEY_NAME=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-priv-ec2" \
  --query 'Reservations[0].Instances[0].KeyName' \
  --output text \
  --region us-east-1) && \
UBUNTU_AMI=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' \
  --output text \
  --region us-east-1) && \
PUB_SG_ID=$(aws ec2 create-security-group \
  --group-name devops-pub-sg \
  --description "SG for devops-pub-ec2" \
  --vpc-id $PUB_VPC_ID \
  --region us-east-1 \
  --query 'GroupId' \
  --output text) && \
aws ec2 authorize-security-group-ingress \
  --group-id $PUB_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0 \
  --region us-east-1 && \
PUB_EC2_ID=$(aws ec2 run-instances \
  --image-id $UBUNTU_AMI \
  --instance-type t2.micro \
  --key-name "$KEY_NAME" \
  --subnet-id $PUB_SUBNET_ID \
  --security-group-ids $PUB_SG_ID \
  --region us-east-1 \
  --query 'Instances[0].InstanceId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PUB_EC2_ID \
  --tags Key=Name,Value=devops-pub-ec2 \
  --region us-east-1 && \
aws ec2 wait instance-running \
  --instance-ids $PUB_EC2_ID \
  --region us-east-1 && \
echo "KEY_NAME=$KEY_NAME  UBUNTU_AMI=$UBUNTU_AMI  PUB_SG_ID=$PUB_SG_ID  PUB_EC2_ID=$PUB_EC2_ID"
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0eb36676e79043eb5",
            "GroupId": "sg-0e7dff2b2dbff34fd",
            "GroupOwnerId": "284304506227",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0",
            "SecurityGroupRuleArn": "arn:aws:ec2:us-east-1:284304506227:security-group-rule/sgr-0eb36676e79043eb5"
        }
    ]
}
KEY_NAME=devops-key  UBUNTU_AMI=ami-00de3875b03809ec5  PUB_SG_ID=sg-0e7dff2b2dbff34fd  PUB_EC2_ID=i-05a7403454c019495

~ on ☁️  (us-east-1) ➜  aws iam create-role \
  --role-name devops-s3-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}' && \
aws iam attach-role-policy \
  --role-name devops-s3-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess && \
aws iam create-instance-profile \
  --instance-profile-name devops-s3-role && \
aws iam add-role-to-instance-profile \
  --instance-profile-name devops-s3-role \
  --role-name devops-s3-role && \
sleep 15 && \
aws ec2 associate-iam-instance-profile \
  --instance-id $PUB_EC2_ID \
  --iam-instance-profile Name=devops-s3-role \
  --region us-east-1 && \
echo "IAM role attached"
{
    "Role": {
        "Path": "/",
        "RoleName": "devops-s3-role",
        "RoleId": "AROAUEMO6PVZ56J3PEPSG",
        "Arn": "arn:aws:iam::284304506227:role/devops-s3-role",
        "CreateDate": "2026-03-25T11:42:19Z",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "ec2.amazonaws.com"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}
{
    "InstanceProfile": {
        "Path": "/",
        "InstanceProfileName": "devops-s3-role",
        "InstanceProfileId": "AIPAUEMO6PVZ2GMD66KZC",
        "Arn": "arn:aws:iam::284304506227:instance-profile/devops-s3-role",
        "CreateDate": "2026-03-25T11:42:21Z",
        "Roles": []
    }
}
{
    "IamInstanceProfileAssociation": {
        "AssociationId": "iip-assoc-06957e99034e284e9",
        "InstanceId": "i-05a7403454c019495",
        "IamInstanceProfile": {
            "Arn": "arn:aws:iam::284304506227:instance-profile/devops-s3-role",
            "Id": "AIPAUEMO6PVZ2GMD66KZC"
        },
        "State": "associating"
    }
}
IAM role attached

~ on ☁️  (us-east-1) ➜  aws s3api create-bucket \
  --bucket devops-s3-logs-25406 \
  --region us-east-1 && \
aws s3api put-public-access-block \
  --bucket devops-s3-logs-25406 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true && \
echo "S3 bucket ready: devops-s3-logs-25406"
{
    "Location": "/devops-s3-logs-25406"
}
S3 bucket ready: devops-s3-logs-25406

~ on ☁️  (us-east-1) ➜  PRIV_VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=devops-priv-vpc" \
  --query 'Vpcs[0].VpcId' \
  --output text \
  --region us-east-1) && \
PRIV_VPC_CIDR=$(aws ec2 describe-vpcs \
  --vpc-ids $PRIV_VPC_ID \
  --query 'Vpcs[0].CidrBlock' \
  --output text \
  --region us-east-1) && \
PEERING_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $PUB_VPC_ID \
  --peer-vpc-id $PRIV_VPC_ID \
  --region us-east-1 \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
  --output text) && \
aws ec2 create-tags \
  --resources $PEERING_ID \
  --tags Key=Name,Value=devops-vpc-peering \
  --region us-east-1 && \
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1 && \
PRIV_RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=devops-priv-rt" \
  --query 'RouteTables[0].RouteTableId' \
  --output text \
  --region us-east-1) && \
aws ec2 create-route \
  --route-table-id $PRIV_RT_ID \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1 && \
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block $PRIV_VPC_CIDR \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1 && \
echo "PRIV_VPC_ID=$PRIV_VPC_ID  PRIV_VPC_CIDR=$PRIV_VPC_CIDR  PEERING_ID=$PEERING_ID  PRIV_RT_ID=$PRIV_RT_ID"
{
    "VpcPeeringConnection": {
        "AccepterVpcInfo": {
            "CidrBlock": "10.10.0.0/16",
            "CidrBlockSet": [
                {
                    "CidrBlock": "10.10.0.0/16"
                }
            ],
            "OwnerId": "284304506227",
            "PeeringOptions": {
                "AllowDnsResolutionFromRemoteVpc": false,
                "AllowEgressFromLocalClassicLinkToRemoteVpc": false,
                "AllowEgressFromLocalVpcToRemoteClassicLink": false
            },
            "VpcId": "vpc-09f0f241ccc7851e0",
            "Region": "us-east-1"
        },
        "RequesterVpcInfo": {
            "CidrBlock": "10.1.0.0/16",
            "CidrBlockSet": [
                {
                    "CidrBlock": "10.1.0.0/16"
                }
            ],
            "OwnerId": "284304506227",
            "PeeringOptions": {
                "AllowDnsResolutionFromRemoteVpc": false,
                "AllowEgressFromLocalClassicLinkToRemoteVpc": false,
                "AllowEgressFromLocalVpcToRemoteClassicLink": false
            },
            "VpcId": "vpc-00c2cdb99aaab238e",
            "Region": "us-east-1"
        },
        "Status": {
            "Code": "provisioning",
            "Message": "Provisioning"
        },
        "Tags": [],
        "VpcPeeringConnectionId": "pcx-07ff722f053d64b7b"
    }
}
{
    "Return": true
}
{
    "Return": true
}
PRIV_VPC_ID=vpc-09f0f241ccc7851e0  PRIV_VPC_CIDR=10.10.0.0/16  PEERING_ID=pcx-07ff722f053d64b7b  PRIV_RT_ID=rtb-0b2873c0c9a0cb96c

~ on ☁️  (us-east-1) ➜  PRIV_SG_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-priv-ec2" \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
  --output text \
  --region us-east-1) && \
aws ec2 authorize-security-group-ingress \
  --group-id $PRIV_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 10.1.0.0/16 \
  --region us-east-1 && \
PUB_EC2_PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $PUB_EC2_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text \
  --region us-east-1) && \
PUB_EC2_PRIVATE_IP=$(aws ec2 describe-instances \
  --instance-ids $PUB_EC2_ID \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text \
  --region us-east-1) && \
PRIV_EC2_PRIVATE_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-priv-ec2" \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text \
  --region us-east-1) && \
echo "PUB_PUBLIC=$PUB_EC2_PUBLIC_IP  PUB_PRIVATE=$PUB_EC2_PRIVATE_IP  PRIV_PRIVATE=$PRIV_EC2_PRIVATE_IP"
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0c77b4fb2d99146f5",
            "GroupId": "sg-034057aef886496b9",
            "GroupOwnerId": "284304506227",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "10.1.0.0/16",
            "SecurityGroupRuleArn": "arn:aws:ec2:us-east-1:284304506227:security-group-rule/sgr-0c77b4fb2d99146f5"
        }
    ]
}
PUB_PUBLIC=3.235.170.246  PUB_PRIVATE=10.1.1.85  PRIV_PRIVATE=10.10.1.235

~ on ☁️  (us-east-1) ➜  scp -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  /root/.ssh/devops-key.pem \
  ubuntu@$PUB_EC2_PUBLIC_IP:/home/ubuntu/.ssh/devops-key.pem && \
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP \
  "chmod 600 /home/ubuntu/.ssh/devops-key.pem" && \
eval $(ssh-agent) && \
ssh-add /root/.ssh/devops-key.pem && \
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  -A ubuntu@$PUB_EC2_PUBLIC_IP \
  "scp -o StrictHostKeyChecking=no /home/ubuntu/.ssh/devops-key.pem ubuntu@$PRIV_EC2_PRIVATE_IP:/home/ubuntu/.ssh/devops-key.pem && \ 
  ssh -o StrictHostKeyChecking=no ubuntu@$PRIV_EC2_PRIVATE_IP 'chmod 600 /home/ubuntu/.ssh/devops-key.pem'" && \
ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP \
  "cat /home/ubuntu/.ssh/devops-key.pem | ssh-keygen -y -f /dev/stdin >> /home/ubuntu/.ssh/authorized_keys && echo 'Public key added'" && \
echo "Keys distributed"
Warning: Permanently added '3.235.170.246' (ECDSA) to the list of known hosts.
devops-key.pem                                                                                     100% 1675    16.6KB/s   00:00    
Agent pid 1534
Identity added: /root/.ssh/devops-key.pem (/root/.ssh/devops-key.pem)
Warning: Permanently added '10.10.1.235' (ED25519) to the list of known hosts.
Public key added
Keys distributed

~ on ☁️  (us-east-1) ➜  ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP \
  "sudo apt-get update -y -qq && \
  sudo apt-get install -y unzip curl -qq && \
  curl -s https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip && \
  unzip -q awscliv2.zip && \
  sudo ./aws/install && \
  /usr/local/bin/aws --version && \
  echo 'AWS CLI installed'"
debconf: unable to initialize frontend: Dialog
debconf: (Dialog frontend will not work on a dumb terminal, an emacs shell buffer, or without a controlling terminal.)
debconf: falling back to frontend: Readline
debconf: unable to initialize frontend: Readline
debconf: (This frontend requires a controlling tty.)
debconf: falling back to frontend: Teletype
dpkg-preconfigure: unable to re-open stdin: 
Selecting previously unselected package unzip.
(Reading database ... 66073 files and directories currently installed.)
Preparing to unpack .../unzip_6.0-26ubuntu3.2_amd64.deb ...
Unpacking unzip (6.0-26ubuntu3.2) ...
Setting up unzip (6.0-26ubuntu3.2) ...
Processing triggers for man-db (2.10.2-1) ...

Running kernel seems to be up-to-date.

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
You can now run: /usr/local/bin/aws --version
aws-cli/2.34.16 Python/3.14.3 Linux/6.8.0-1050-aws exe/x86_64.ubuntu.22
AWS CLI installed

~ on ☁️  (us-east-1) ➜  ssh -i /root/.ssh/devops-key.pem \
  -o StrictHostKeyChecking=no \
  -A ubuntu@$PUB_EC2_PUBLIC_IP \
  "ssh -o StrictHostKeyChecking=no ubuntu@$PRIV_EC2_PRIVATE_IP \
  'ls -lh /var/log/boot* /var/log/syslog /var/log/kern.log /var/log/auth.log 2>&1 && \
  echo \"=== SYSLOG HEAD ===\" && sudo head -3 /var/log/syslog 2>&1 && \
  echo \"=== KERN HEAD ===\" && sudo head -3 /var/log/kern.log 2>&1 && \
  echo \"=== BOOTS.LOG check ===\" && ls -la /var/log/boots.log 2>&1'"
-rw-r----- 1 syslog adm  5.2K Mar 25 11:49 /var/log/auth.log
-rw-r--r-- 1 root   root   23 Mar 25 11:36 /var/log/boots.log
-rw-r----- 1 syslog adm   54K Mar 25 11:35 /var/log/kern.log
-rw-r----- 1 syslog adm  135K Mar 25 11:49 /var/log/syslog
=== SYSLOG HEAD ===
Mar 25 11:35:41 ip-10-10-1-235 systemd[1]: Mounted Huge Pages File System.
Mar 25 11:35:41 ip-10-10-1-235 systemd[1]: Mounted POSIX Message Queue File System.
Mar 25 11:35:41 ip-10-10-1-235 systemd[1]: Mounted Kernel Debug File System.
=== KERN HEAD ===
Mar 25 11:35:41 ip-10-10-1-235 kernel: [    0.000000] Linux version 5.15.0-1084-aws (buildd@lcy02-amd64-055) (gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, GNU ld (GNU Binutils for Ubuntu) 2.34) #91~20.04.1-Ubuntu SMP Fri May 2 06:59:36 UTC 2025 (Ubuntu 5.15.0-1084.91~20.04.1-aws 5.15.179)
Mar 25 11:35:41 ip-10-10-1-235 kernel: [    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-5.15.0-1084-aws root=PARTUUID=cfbcdd53-02ad-4c2e-8163-3cd5c66e640a ro console=tty1 console=ttyS0 nvme_core.io_timeout=4294967295 panic=-1
Mar 25 11:35:41 ip-10-10-1-235 kernel: [    0.000000] KERNEL supported cpus:
=== BOOTS.LOG check ===
-rw-r--r-- 1 root root 23 Mar 25 11:36 /var/log/boots.log

~ on ☁️  (us-east-1) ➜  ssh -i /root/.ssh/devops-key.pem -o StrictHostKeyChecking=no -A ubuntu@$PUB_EC2_PUBLIC_IP \
  "ssh -o StrictHostKeyChecking=no ubuntu@$PRIV_EC2_PRIVATE_IP \
  'echo \"* * * * * scp -i /home/ubuntu/.ssh/devops-key.pem -o StrictHostKeyChecking=no /var/log/boots.log ubuntu@$PUB_EC2_PRIVATE_IP:/home/ubuntu/boots.log\" | crontab -'"

~ on ☁️  (us-east-1) ➜  ssh -i /root/.ssh/devops-key.pem -o StrictHostKeyChecking=no ubuntu@$PUB_EC2_PUBLIC_IP \
  "echo \"* * * * * /usr/local/bin/aws s3 cp /home/ubuntu/boots.log s3://devops-s3-logs-25406/devops-priv-vpc/boot/boots.log\" | crontab -"

~ on ☁️  (us-east-1) ➜  aws ec2 describe-vpc-peering-connections \
--vpc-peering-connection-ids $PEERING_ID \
--query 'VpcPeeringConnections[0].Status.Code'
"active"

~ on ☁️  (us-east-1) ➜  aws s3 ls s3://devops-s3-logs-25406/devops-priv-vpc/boot/
2026-03-25 12:02:03         23 boots.log

~ on ☁️  (us-east-1) ➜  
