# AWS Private RDS Instance Provisioning with EC2 Integration

![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8.4.5-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![Apache](https://img.shields.io/badge/Apache-2.4-D22128?style=for-the-badge&logo=apache&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-8.1-777BB4?style=for-the-badge&logo=php&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen?style=for-the-badge)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Known Issues and Resolutions](#known-issues-and-resolutions)
- [Step-by-Step Implementation](#step-by-step-implementation)
- [Verification](#verification)
- [Security Considerations](#security-considerations)
- [Lessons Learned](#lessons-learned)
- [Contributing](#contributing)

---

## Overview

This runbook documents the end-to-end provisioning of a **private Amazon RDS (MySQL 8.4.5)** instance and its integration with an existing EC2 web server, including passwordless SSH access configuration and PHP application deployment. It is intended as a repeatable, production-grade reference for DevOps and Cloud Infrastructure teams.

**Business Context:** The Nautilus DevOps team required a private, VPC-scoped MySQL database backend for their web application, with the EC2 instance serving as the only authorized entry point into the database tier.

---

## Architecture

```
                         +-----------------------+
                         |      aws-client        |
                         |  (Admin Bastion Host)  |
                         +----------+------------+
                                    |
                              SSH (Port 22)
                                    |
                    +---------------v---------------+
                    |         nautilus-ec2           |
                    |   Ubuntu 22.04 + Apache2 +PHP  |
                    |   Security Group: sg-045f...   |
                    +---------------+---------------+
                                    |
                            MySQL (Port 3306)
                       Source SG Authorization Only
                                    |
                    +---------------v---------------+
                    |         nautilus-rds           |
                    |   MySQL 8.4.5 / db.t3.micro    |
                    |   Private / No Public Access   |
                    |   Security Group: sg-0b88...   |
                    +-------------------------------+
```

> **Screenshot Placeholder**
> `[SCREENSHOT-01: AWS Console -- VPC Architecture Diagram showing EC2 and RDS in the same VPC]`

---

## Prerequisites

| Requirement | Details |
|---|---|
| AWS CLI | Configured with appropriate IAM permissions |
| IAM Permissions | EC2, RDS, VPC, EC2 Instance Connect |
| Existing Resource | EC2 instance tagged `Name=nautilus-ec2` |
| OS on EC2 | Ubuntu 22.04 LTS with Apache2 and PHP installed |
| Local File | `/root/index.php` on `aws-client` host |
| Region | `us-east-1` |

---

## Known Issues and Resolutions

This section documents every failure encountered during implementation and the exact resolution applied. This is the core operational value of this runbook.

---

### Issue 1: SSH Connection Timeout on Port 22

**Symptom:**
```
ssh: connect to host <EC2_IP> port 22: Connection timed out
```

**Root Cause:**
Port 22 was not open in the EC2 instance security group inbound rules at the time of the SSH attempt.

**Resolution:**
Open port 22 on the EC2 security group **before** any SSH or SCP operation.

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $EC2_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

> **Screenshot Placeholder**
> `[SCREENSHOT-02: AWS Console -- EC2 Security Group Inbound Rules showing ports 22, 80, and 3306]`

---

### Issue 2: SSH Public Key Permission Denied

**Symptom:**
```
root@<EC2_IP>: Permission denied (publickey).
```

**Root Cause:**
The EC2 instance did not have the `aws-client` public key in `/root/.ssh/authorized_keys`. The default instance only accepts the original launch keypair.

**Resolution:**
Use EC2 Instance Connect to inject the key temporarily, then immediately write it permanently to `authorized_keys` within the same chained command to avoid the 60-second expiry window.

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

**Verification:**
```bash
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" root@$EC2_IP \
  "echo 'Passwordless SSH confirmed'"
```

> **Screenshot Placeholder**
> `[SCREENSHOT-03: Terminal output showing "SSH key permanently installed" and "Passwordless SSH confirmed"]`

---

### Issue 3: Grader Fails with "SSH key is not configured for passwordless access"

**Symptom:**
`curl http://$EC2_IP` returned `Connected successfully`, but the automated grader reported:
```
SSH key is not configured for passwordless access to the instance
```

**Root Cause:**
EC2 Instance Connect injects the SSH public key **temporarily** (60-second TTL) into the instance metadata endpoint. The grader performs its own independent SSH check at evaluation time, by which point the key had expired from the session and was never persisted to `authorized_keys`.

**Resolution:**
The key must be written **permanently** to `/root/.ssh/authorized_keys` on the EC2 instance as described in Issue 2. Temporary EC2 Instance Connect sessions alone are insufficient for graded or audited environments.

---

### Issue 4: Apache Serves Default Page Instead of PHP Application

**Symptom:**
```bash
curl http://$EC2_IP
# Returns: Apache2 Ubuntu Default Page
```

**Root Cause:**
Apache's `DirectoryIndex` directive prioritizes `index.html` over `index.php` when both files exist in `/var/www/html/`. The default Ubuntu Apache installation ships with `index.html` in place.

**Resolution:**
Remove `index.html` before or immediately after deploying `index.php`.

```bash
ssh -o "IdentitiesOnly=yes" root@$EC2_IP \
  "rm -f /var/www/html/index.html && systemctl restart apache2"
```

> **Screenshot Placeholder**
> `[SCREENSHOT-04: Browser showing "Connected successfully" after index.html removal]`

---

### Issue 5: sed Placeholder Mismatch -- No Substitution Performed

**Symptom:**
```
PHP Warning: mysqli_connect(): php_network_getaddresses: getaddrinfo for <dbhost> failed
```

**Root Cause:**
The `sed` commands were targeting incorrect placeholder names (`db_host`, `db_user`, etc.) while the actual placeholders in `index.php` used angle-bracket syntax (`<dbhost>`, `<dbuser>`, `<dbpass>`, `<dbname>`). The substitution silently succeeded with zero matches, leaving literal placeholder strings in the deployed file.

**Resolution:**
Always `cat` the file first to inspect exact placeholder syntax before writing any `sed` commands.

```bash
# Step 1: Inspect first
cat /root/index.php

# Step 2: Match EXACTLY what you see
sed -i 's/<dbhost>/'"$RDS_ENDPOINT"'/' /root/index.php
sed -i 's/<dbuser>/nautilus_admin/' /root/index.php
sed -i 's/<dbpass>/YourPassword/' /root/index.php
sed -i 's/<dbname>/nautilus_db/' /root/index.php

# Step 3: Verify no angle brackets remain
cat /root/index.php | grep '<db'
# Expected output: (empty -- no matches)
```

> **Screenshot Placeholder**
> `[SCREENSHOT-05: Terminal showing cat output with all placeholders correctly substituted]`

---

### Issue 6: Bash History Expansion Error with Passwords Containing "!"

**Symptom:**
```
bash: !/g: event not found
```

**Root Cause:**
Bash interprets `!` inside double-quoted strings as a history expansion trigger. Passwords or `sed` replacement strings containing `!` fail when wrapped in double quotes.

**Resolution:**
Always use **single quotes** for passwords and `sed` replacement values containing special characters.

```bash
# Incorrect -- triggers history expansion
sed -i "s/<dbpass>/Password123!/g" /root/index.php

# Correct -- single quotes prevent expansion
sed -i 's/<dbpass>/Password123!/' /root/index.php

# Correct -- for RDS endpoint variable interpolation
sed -i 's/<dbhost>/'"$RDS_ENDPOINT"'/' /root/index.php
```

---

## Step-by-Step Implementation

This is the complete, validated, production-ready execution sequence.

### Step 1: Export Environment Variables

```bash
export VPC_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query 'Reservations[0].Instances[0].VpcId' --output text)

export EC2_SG_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' --output text)

export EC2_INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)

export EC2_IP=$(aws ec2 describe-instances \
  --instance-ids $EC2_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

echo "VPC: $VPC_ID | SG: $EC2_SG_ID | Instance: $EC2_INSTANCE_ID | IP: $EC2_IP"
```

---

### Step 2: Open Required Ports on EC2 Security Group

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $EC2_SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id $EC2_SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
```

---

### Step 3: Create RDS Security Group and Authorize EC2 Access

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

---

### Step 4: Provision the RDS Instance

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
  --master-user-password 'YourSecurePassword!' \
  --vpc-security-group-ids $RDS_SG_ID \
  --no-publicly-accessible

echo "Waiting for RDS to become available (~5 mins)..."
aws rds wait db-instance-available --db-instance-identifier nautilus-rds

export RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier nautilus-rds \
  --query 'DBInstances[0].Endpoint.Address' --output text)

echo "RDS Endpoint: $RDS_ENDPOINT"
```

> **Screenshot Placeholder**
> `[SCREENSHOT-06: AWS Console -- RDS Instance showing "Available" status with endpoint visible]`

---

### Step 5: Generate SSH Keypair (if not present)

```bash
[[ -f /root/.ssh/id_rsa ]] || ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
```

---

### Step 6: Permanently Install SSH Key on EC2

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

**Verify passwordless access works independently:**

```bash
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" root@$EC2_IP \
  "echo 'Passwordless SSH confirmed'"
```

---

### Step 7: Inspect and Patch index.php

```bash
# Always inspect before sed
cat /root/index.php

# Backup original
cp /root/index.php /root/index.php.bak

# Replace placeholders -- single quotes to avoid bash history expansion
sed -i 's/<dbhost>/'"$RDS_ENDPOINT"'/' /root/index.php
sed -i 's/<dbuser>/nautilus_admin/' /root/index.php
sed -i 's/<dbpass>/YourSecurePassword!/' /root/index.php
sed -i 's/<dbname>/nautilus_db/' /root/index.php

# Confirm no placeholders remain
cat /root/index.php
```

---

### Step 8: Deploy Application and Restart Apache

```bash
scp -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" \
    /root/index.php root@$EC2_IP:/var/www/html/index.php && \
ssh -o "StrictHostKeyChecking=no" -o "IdentitiesOnly=yes" root@$EC2_IP \
    "rm -f /var/www/html/index.html && systemctl restart apache2"
```

---

## Verification

```bash
curl -s http://$EC2_IP
# Expected output: Connected successfully
```

> **Screenshot Placeholder**
> `[SCREENSHOT-07: Browser at http://<EC2_PUBLIC_IP> showing "Connected successfully"]`

> **Screenshot Placeholder**
> `[SCREENSHOT-08: Terminal showing curl output "Connected successfully<br />"]`

---

## Security Considerations

| Area | Recommendation |
|---|---|
| Port 22 | Restrict to known IP ranges in production (`--cidr <YOUR_IP>/32`) |
| Port 80 | Use HTTPS with ACM certificate and ALB in production |
| RDS Password | Store in AWS Secrets Manager, not hardcoded in scripts |
| RDS Access | Port 3306 is scoped to EC2 security group only, never `0.0.0.0/0` |
| SSH Keys | Rotate regularly and remove temporary `0.0.0.0/0` SSH rules post-setup |
| RDS Encryption | Enable `--storage-encrypted` for production workloads |

---

## Lessons Learned

| # | Lesson | Impact |
|---|---|---|
| 1 | Open port 22 before attempting any SSH operations | Prevents connection timeout failures |
| 2 | Always `cat` config files before writing `sed` commands | Prevents silent no-op substitutions |
| 3 | Use single quotes for passwords with special characters | Prevents bash history expansion errors |
| 4 | Chain EC2 Instance Connect with SSH/SCP using `&&` | Prevents key expiry within the 60s window |
| 5 | Permanently write to `authorized_keys`, do not rely on Instance Connect TTL | Required for graders, audits, and automation |
| 6 | Remove `index.html` before or alongside deploying `index.php` | Prevents Apache default page from overriding the app |

---

## Contributing

Pull requests are welcome. For significant changes, open an issue first to discuss what you would like to change. Ensure all runbook steps are tested end-to-end in a sandbox environment before submitting.

---

**Region:** `us-east-1`

<img width="1038" height="429" alt="image" src="https://github.com/user-attachments/assets/cf0031c0-8882-41dc-8543-7184550e39bf" />
<img width="1031" height="730" alt="image" src="https://github.com/user-attachments/assets/ad5e76c5-fb1d-4e5a-b815-a378e3199a1c" />
<img width="1031" height="679" alt="image" src="https://github.com/user-attachments/assets/7d4737e9-4bcf-4ac4-9a49-8cd087516b7d" />
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/63a12373-20c3-4c00-bca9-8dc6b94050eb" />
<img width="1026" height="859" alt="image" src="https://github.com/user-attachments/assets/11fce700-c597-428f-b4b9-514eed0afaf9" />
<img width="1030" height="859" alt="image" src="https://github.com/user-attachments/assets/15023acb-a407-4bb3-99c5-6ca457a287b9" />
<img width="1036" height="860" alt="image" src="https://github.com/user-attachments/assets/01e2b08b-0274-412e-8096-960a6d43ea82" />
<img width="1033" height="418" alt="image" src="https://github.com/user-attachments/assets/ef65791d-14de-406f-9416-20f0510800aa" />
<img width="1029" height="728" alt="image" src="https://github.com/user-attachments/assets/01d5d68b-2691-45c9-8f55-9f2589df263d" />
<img width="1030" height="449" alt="image" src="https://github.com/user-attachments/assets/6c429d8d-0883-459c-9d53-12dcbd564287" />
<img width="1032" height="579" alt="image" src="https://github.com/user-attachments/assets/2fd9f59b-bb2c-4a0b-a9b2-7c0fcc996696" />
<img width="1031" height="630" alt="image" src="https://github.com/user-attachments/assets/5c82eb1b-7d20-478a-89ee-b81610569b59" />
<img width="1025" height="630" alt="image" src="https://github.com/user-attachments/assets/cfb94ff3-0aba-4b46-8c29-8e4612a7e41c" />
<img width="1032" height="618" alt="image" src="https://github.com/user-attachments/assets/df87903d-db06-43d9-8d03-43df43c56093" />
<img width="1035" height="209" alt="image" src="https://github.com/user-attachments/assets/81e9ef3c-820a-4f81-871e-7717d0fa1f7c" />



