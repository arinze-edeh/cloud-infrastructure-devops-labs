# Nautilus AWS Infrastructure: EC2 to RDS Private Connectivity

![Project Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen)
![AWS](https://img.shields.io/badge/Cloud-AWS-orange)
![MySQL](https://img.shields.io/badge/Database-MySQL%208.4.5-blue)
![Apache](https://img.shields.io/badge/Web%20Server-Apache2-red)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Infrastructure Components](#infrastructure-components)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: Environment Bootstrap](#phase-1-environment-bootstrap)
  - [Phase 2: Security Group Configuration](#phase-2-security-group-configuration)
  - [Phase 3: RDS Instance Provisioning](#phase-3-rds-instance-provisioning)
  - [Phase 4: Passwordless SSH Setup](#phase-4-passwordless-ssh-setup)
  - [Phase 5: Application Deployment](#phase-5-application-deployment)
  - [Phase 6: Validation](#phase-6-validation)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

---

## Overview

This project documents the end-to-end provisioning of a **private Amazon RDS MySQL instance** and its secure integration with an existing EC2-hosted PHP web application. The solution enforces least-privilege networking by scoping database access exclusively to the application tier, eliminating public database exposure while preserving full application connectivity.

> **Outcome:** A browser request to the EC2 public IP returns `Connected successfully`, confirming an authenticated, end-to-end application-to-database connection over a private VPC network path.

---

## Problem Statement

### Context

The Nautilus DevOps team required a managed relational database backend for their web application running on an existing EC2 instance (`nautilus-ec2`). The key engineering constraints were:

| Constraint | Requirement |
|---|---|
| Database Exposure | Private only, no public endpoint |
| Network Access | EC2 security group as sole ingress source |
| Authentication | Dedicated master credentials, not default |
| Web Access | Port 80 open to the internet |
| SSH Access | Key-based passwordless authentication |
| Validation | Browser-level connectivity confirmation |

### Root Cause of Gap

The pre-existing EC2 instance had no database backend. The `index.php` application file contained placeholder values (`<dbhost>`, `<dbuser>`, `<dbpass>`, `<dbname>`) with no live database to connect to, rendering the application non-functional.

---

## Architecture

```
                        Internet
                            |
                     [ Port 80 / 22 ]
                            |
                 +---------------------+
                 |    nautilus-ec2     |
                 |   (Apache + PHP)    |
                 |  sg-045f295ffdb9b5  |
                 +---------------------+
                            |
                     [ Port 3306 ]
                   (Source SG only)
                            |
                 +---------------------+
                 |    nautilus-rds     |
                 |  MySQL 8.4.5        |
                 |  db.t3.micro        |
                 |  sg-0b883608cbdb5   |
                 +---------------------+
                            |
                    [ VPC Private Subnets ]
                    vpc-0b8bf1011ce50fff5
```
---

## Prerequisites

Before executing this runbook, confirm the following:

- AWS CLI v2 configured with sufficient IAM permissions (`ec2:*`, `rds:*`)
- An existing EC2 instance tagged `Name=nautilus-ec2` in `us-east-1`
- `ssh-keygen`, `scp`, and `curl` available on the client host (`aws-client`)
- EC2 instance reachable from the client host
- `index.php` present at `/root/index.php` on `aws-client`

---

## Infrastructure Components

| Resource | Identifier | Description |
|---|---|---|
| VPC | `vpc-0b8bf1011ce50fff5` | Existing VPC hosting all resources |
| EC2 Instance | `i-0eaf224b03992b6e9` | Application server running Apache2 + PHP |
| EC2 Security Group | `sg-045f295ffdb9b5107` | Controls inbound traffic to EC2 |
| RDS Security Group | `sg-0b883608cbdb55f93` | Controls inbound traffic to RDS (EC2 SG as source) |
| RDS Instance | `nautilus-rds` | Private MySQL 8.4.5 on `db.t3.micro`, 5 GiB gp2 |
| RDS Endpoint | `nautilus-rds.czigwwy0ygu9.us-east-1.rds.amazonaws.com` | Private DNS endpoint |
| Database Name | `nautilus_db` | Application database |
| Master Username | `nautilus_admin` | RDS master user |

---

## Implementation Guide

### Phase 1: Environment Bootstrap

Export all required environment variables to avoid hardcoded values across subsequent commands.

```bash
export VPC_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=nautilus-ec2" \
    --query 'Reservations[0].Instances[0].VpcId' \
    --output text)

export EC2_SG_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=nautilus-ec2" \
    --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
    --output text)

export EC2_INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=nautilus-ec2" \
    --query 'Reservations[0].Instances[0].InstanceId' \
    --output text)

export EC2_IP=$(aws ec2 describe-instances \
    --instance-ids $EC2_INSTANCE_ID \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text)

echo "VPC: $VPC_ID | SG: $EC2_SG_ID | Instance: $EC2_INSTANCE_ID | IP: $EC2_IP"
```

**Expected Output:**
```
VPC: vpc-0b8bf1011ce50fff5 | SG: sg-045f295ffdb9b5107 | Instance: i-0eaf224b03992b6e9 | IP: 44.204.190.242
```

---

### Phase 2: Security Group Configuration

#### 2a. Open EC2 Inbound Ports

Allow SSH (port 22) for remote management and HTTP (port 80) for web traffic.

```bash
aws ec2 authorize-security-group-ingress \
    --group-id $EC2_SG_ID \
    --protocol tcp --port 22 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id $EC2_SG_ID \
    --protocol tcp --port 80 --cidr 0.0.0.0/0
```

#### 2b. Create and Configure the RDS Security Group

Create a dedicated security group for the RDS instance. The ingress rule references the **EC2 security group ID** as the source, not a CIDR block. This ensures only traffic originating from the EC2 instance is permitted on port 3306.

```bash
export RDS_SG_ID=$(aws ec2 create-security-group \
    --group-name nautilus-rds-sg \
    --description "RDS access for nautilus-ec2" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
    --group-id $RDS_SG_ID \
    --protocol tcp --port 3306 \
    --source-group $EC2_SG_ID

echo "RDS SG: $RDS_SG_ID"
```

> **Security Note:** Using a source security group instead of a CIDR range eliminates the risk of lateral movement from other resources in the VPC subnet range. Only instances attached to `EC2_SG_ID` can reach the RDS port.

### Screenshots

<img width="1038" height="429" alt="image" src="https://github.com/user-attachments/assets/cf0031c0-8882-41dc-8543-7184550e39bf" />
<img width="1031" height="730" alt="image" src="https://github.com/user-attachments/assets/ad5e76c5-fb1d-4e5a-b815-a378e3199a1c" />
<img width="1031" height="679" alt="image" src="https://github.com/user-attachments/assets/7d4737e9-4bcf-4ac4-9a49-8cd087516b7d" />

---

### Phase 3: RDS Instance Provisioning

Create the MySQL RDS instance with `--no-publicly-accessible` enforced. The instance is attached to the RDS-specific security group created in Phase 2.

```bash
aws rds create-db-instance \
    --db-instance-identifier nautilus-rds \
    --engine mysql \
    --engine-version 8.4.5 \
    --db-instance-class db.t3.micro \
    --allocated-storage 5 \
    --storage-type gp2 \
    --db-name nautilus_db \
    --master-username nautilus_admin \
    --master-user-password 'Secure#Pass2026!' \
    --vpc-security-group-ids $RDS_SG_ID \
    --no-publicly-accessible
```

Wait for the instance to reach `available` state and capture the endpoint:

```bash
echo "Waiting for RDS (~5 mins)..."
aws rds wait db-instance-available --db-instance-identifier nautilus-rds

export RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier nautilus-rds \
    --query 'DBInstances[0].Endpoint.Address' --output text)

echo "RDS Endpoint: $RDS_ENDPOINT"
```

**Expected Output:**
```
RDS Endpoint: nautilus-rds.czigwwy0ygu9.us-east-1.rds.amazonaws.com
```

### Screenshots

<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/63a12373-20c3-4c00-bca9-8dc6b94050eb" />
<img width="1026" height="859" alt="image" src="https://github.com/user-attachments/assets/11fce700-c597-428f-b4b9-514eed0afaf9" />
<img width="1030" height="859" alt="image" src="https://github.com/user-attachments/assets/15023acb-a407-4bb3-99c5-6ca457a287b9" />
<img width="1036" height="860" alt="image" src="https://github.com/user-attachments/assets/01e2b08b-0274-412e-8096-960a6d43ea82" />

---

### Phase 4: Passwordless SSH Setup

#### 4a. Generate RSA Key Pair (if not present)

```bash
[ -f /root/.ssh/id_rsa ] || ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
```

#### 4b. Inject Public Key via EC2 Instance Connect

Use EC2 Instance Connect for the initial key delivery, then permanently install the public key in the EC2 instance's `authorized_keys`.

```bash
aws ec2-instance-connect send-ssh-public-key \
    --instance-id $EC2_INSTANCE_ID \
    --instance-os-user root \
    --ssh-public-key file:///root/.ssh/id_rsa.pub && \
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" root@$EC2_IP \
    "mkdir -p /root/.ssh && \
     echo '$(cat /root/.ssh/id_rsa.pub)' >> /root/.ssh/authorized_keys && \
     chmod 700 /root/.ssh && \
     chmod 600 /root/.ssh/authorized_keys && \
     echo 'SSH key permanently installed'"
```

#### 4c. Confirm Passwordless Access

```bash
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" root@$EC2_IP \
    "echo 'Passwordless SSH confirmed'"
```

**Expected Output:**
```
Passwordless SSH confirmed
```

### Screenshots

<img width="1033" height="418" alt="image" src="https://github.com/user-attachments/assets/ef65791d-14de-406f-9416-20f0510800aa" />
<img width="1029" height="728" alt="image" src="https://github.com/user-attachments/assets/01d5d68b-2691-45c9-8f55-9f2589df263d" />
<img width="1030" height="449" alt="image" src="https://github.com/user-attachments/assets/6c429d8d-0883-459c-9d53-12dcbd564287" />

---

### Phase 5: Application Deployment

#### 5a. Populate Database Credentials in index.php

Back up the original file, then use `sed` to inject the live RDS credentials in-place.

```bash
cp /root/index.php /root/index.php.bak

sed -i 's/<dbhost>/'"$RDS_ENDPOINT"'/' /root/index.php
sed -i 's/<dbuser>/nautilus_admin/' /root/index.php
sed -i 's/<dbpass>/Secure#Pass2026!/' /root/index.php
sed -i 's/<dbname>/nautilus_db/' /root/index.php
```

#### 5b. Verify Substitution

```bash
cat /root/index.php
```

**Expected Output (credentials populated):**
```php
<?php
$dbname = 'nautilus_db';
$dbuser = 'nautilus_admin';
$dbpass = 'Secure#Pass2026!';
$dbhost = 'nautilus-rds.czigwwy0ygu9.us-east-1.rds.amazonaws.com';
...
```

#### 5c. Deploy to EC2 and Restart Apache

Transfer the file to the EC2 web root, remove the default Apache placeholder, and restart the service.

```bash
scp -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" \
    /root/index.php root@$EC2_IP:/var/www/html/index.php && \
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" root@$EC2_IP \
    "rm -f /var/www/html/index.html && systemctl restart apache2"
```

### Screenshots

<img width="1032" height="579" alt="image" src="https://github.com/user-attachments/assets/2fd9f59b-bb2c-4a0b-a9b2-7c0fcc996696" />
<img width="1031" height="630" alt="image" src="https://github.com/user-attachments/assets/5c82eb1b-7d20-478a-89ee-b81610569b59" />
<img width="1025" height="630" alt="image" src="https://github.com/user-attachments/assets/cfb94ff3-0aba-4b46-8c29-8e4612a7e41c" />
<img width="1032" height="618" alt="image" src="https://github.com/user-attachments/assets/df87903d-db06-43d9-8d03-43df43c56093" />

---

### Phase 6: Validation

Perform a final end-to-end HTTP connectivity test against the EC2 public IP.

```bash
curl -s http://$EC2_IP
```

**Expected Output:**
```
Connected successfully
```

### Screenshot 

<img width="1035" height="209" alt="image" src="https://github.com/user-attachments/assets/81e9ef3c-820a-4f81-871e-7717d0fa1f7c" />

---

## Security Considerations

### What Was Done Right

- **Private RDS endpoint:** `--no-publicly-accessible` prevents any direct internet route to the database.
- **Security group chaining:** The RDS ingress rule references the EC2 security group, not a CIDR, scoping access to the application tier only.
- **Key-based SSH only:** Password authentication is bypassed entirely via RSA key injection.
- **Credential injection at runtime:** Placeholder values in `index.php` are substituted at deploy time, not hardcoded in source.

### Recommended Hardening for Production

| Area | Current State | Recommended Improvement |
|---|---|---|
| SSH CIDR | `0.0.0.0/0` on port 22 | Restrict to known operator IP ranges or use AWS Systems Manager Session Manager |
| Credential Storage | Inline in PHP file | Migrate to AWS Secrets Manager with IAM-based retrieval |
| Storage Encryption | Disabled | Enable `--storage-encrypted` with a KMS CMK |
| Backup Retention | 1 day | Increase to 7 or more days for production workloads |
| Multi-AZ | Disabled | Enable `--multi-az` for high availability |
| TLS in Transit | Not enforced | Configure `require_secure_transport=ON` in the RDS parameter group |

---

## Troubleshooting

### curl returns "Unable to Connect to"

**Cause:** The EC2 instance cannot reach the RDS endpoint on port 3306.

**Resolution Steps:**
1. Confirm `$RDS_SG_ID` inbound rule lists `EC2_SG_ID` as source on port 3306.
2. Confirm the RDS instance status is `available` via `aws rds describe-db-instances`.
3. Confirm both resources are in the same VPC (`$VPC_ID`).

```bash
aws rds describe-db-instances \
    --db-instance-identifier nautilus-rds \
    --query 'DBInstances[0].DBInstanceStatus'
```

---

### curl returns "Could not open the db"

**Cause:** The database name `nautilus_db` does not match the one provisioned, or the `sed` substitution did not apply correctly.

**Resolution Steps:**

```bash
grep dbname /root/index.php
# Must show: $dbname = 'nautilus_db';
```

---

### SSH hangs or times out

**Cause:** Port 22 is not open on the EC2 security group, or the EC2 Instance Connect key delivery window (60 seconds) expired before the SSH command executed.

**Resolution Steps:**
1. Verify port 22 is open: `aws ec2 describe-security-groups --group-ids $EC2_SG_ID`
2. Re-run the `send-ssh-public-key` command immediately followed by the SSH command in the same shell session.

---

### Screenshot Placeholder

> **[SCREENSHOT 6: AWS EC2 Console - Security Groups tab for nautilus-ec2 showing inbound rules for ports 22 and 80 from 0.0.0.0/0]**

---

*Region: us-east-1.*
