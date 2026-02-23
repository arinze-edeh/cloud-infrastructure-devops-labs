# Secure EC2 Provisioning with SSH Access (AWS CLI)

## Overview
- This project demonstrates **end-to-end provisioning of an Amazon EC2 instance** using the AWS CLI, including secure SSH access via key-based authentication and automated bootstrapping with **user-data**.

The goal is to validate hands-on knowledge of:
- AWS Identity & Access Management
- EC2 networking and security
- Linux system configuration
- Secure remote access patterns

This workflow mirrors **real-world DevOps and Cloud Engineering practices**.

---

## Architecture
- **Cloud Provider:** `AWS`
- **Region:** `us-east-1`
- **AMI:** `Amazon Linux 2`
- **Instance Type:** `t2.micro`
- **Access Method:** `SSH (Key-based`)
- **Security Layer:** `EC2 Security Group (Port 22)`

---

## Prerequisites
- AWS account with EC2 permissions
- AWS CLI configured
- Linux-based environment
- Open outbound SSH access

---

## Implementation Steps

## Step 1️: Verify AWS Identity

- `aws sts get-caller-identity`

📸 Screenshot:
<img width="1034" height="412" alt="image" src="https://github.com/user-attachments/assets/c100a1b9-b03f-4a5b-a141-66955f94ccd6" />

## Step 2️: Generate SSH Key Pair
- `ssh-keygen -t rsa -b 2048 -f /root/.ssh/id_rsa -N ""`

📸 Screenshot:
<img width="1033" height="659" alt="image" src="https://github.com/user-attachments/assets/aa49f081-1ffd-4cc7-8ec5-2b3b954d6b78" />

## Step 3️: Retrieve Latest Amazon Linux 2 AMI
- `aws ec2 describe-images \`
 - `--owners amazon \`
- `--filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \`
 - `--query 'Images | sort_by(@, &CreationDate)[-1].ImageId' \`
 - `--output text`

📸 Screenshot:
<img width="1032" height="777" alt="image" src="https://github.com/user-attachments/assets/d475f0f6-56d2-4643-a16f-845435edd201" />

## Step 4️: Create Security Group (SSH Access)
- `aws ec2 create-security-group \`
 - `--group-name datacenter-sg \`
 - `--description "Allow SSH access"`
- `aws ec2 authorize-security-group-ingress \`
 - `--group-name datacenter-sg \`
 - `--protocol tcp \`
 - `--port 22 \`
 - `--cidr 0.0.0.0/0`

📸 Screenshot:
<img width="1028" height="742" alt="image" src="https://github.com/user-attachments/assets/9e2924d5-603e-4c39-a6a1-37a71663270a" />

## Step 5️: Configure EC2 User-Data (Bootstrapping)

The user-data script:

- Injects SSH public key

- Sets correct file permissions

- Enables root login

- Restarts SSH service

- `cat <<EOF > /tmp/userdata.sh`
- `#!/bin/bash`
- `mkdir -p /root/.ssh`
- `chmod 700 /root/.ssh`
- `echo "<SSH_PUBLIC_KEY>" > /root/.ssh/authorized_keys`
- `chmod 600 /root/.ssh/authorized_keys`
- `sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config`
- `sed -i 's/^PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config`
- `systemctl restart sshd`
- `EOF`

📸 Screenshot:
<img width="1035" height="735" alt="image" src="https://github.com/user-attachments/assets/bf2e812a-6b10-4e79-9086-e4739113a413" />

## Step 6️: Launch EC2 Instance
- `aws ec2 run-instances \`
 - `--image-id ami-0199fa5fada510433 \`
 - `--instance-type t2.micro \`
 - `--security-group-ids sg-071a2bc44bda9d1c8 \`
 - `--user-data file:///tmp/userdata.sh \`
 - `--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datacenter-ec2}]'`

📸 Screenshots:
<img width="1017" height="855" alt="image" src="https://github.com/user-attachments/assets/97ffb5a1-b8fd-462e-999d-e889b564b3d3" />
<img width="1026" height="858" alt="image" src="https://github.com/user-attachments/assets/b1d26184-c6a6-4369-848e-d557c5330c5c" />
<img width="1013" height="861" alt="image" src="https://github.com/user-attachments/assets/0dba9622-4705-4796-82bf-aef693c1ad2d" />
<img width="1032" height="857" alt="image" src="https://github.com/user-attachments/assets/c0eaea9c-80f8-4d79-9f39-c545dfacef2c" />

## Step 7️: Retrieve Public IP
- `aws ec2 describe-instances \`
 - `--filters "Name=tag:Name,Values=datacenter-ec2" \`
 - `--query "Reservations[*].Instances[*].PublicIpAddress" \`
 - `--output text`

📸 Screenshot:
<img width="1031" height="373" alt="image" src="https://github.com/user-attachments/assets/0faefc88-a7b1-48cc-bbbc-c32c528e8a8f" />

## Step 8️: SSH into the Instance
- `ssh root@<PUBLIC_IP>`

📸 Screenshot:
<img width="1037" height="469" alt="image" src="https://github.com/user-attachments/assets/781abe44-750b-48e9-9009-3cde69f608d5" />

## Verification

Successful SSH login confirms:

- Security group correctly configured

- User-data executed successfully

- SSH key authentication functional

- Instance reachable from public internet

## Security Notes

- `⚠️ Root login is enabled for demonstration purposes only.`

In production environments:

- Disable root SSH access

- Use IAM roles + SSM Session Manager

- Restrict SSH CIDR ranges

- Rotate SSH keys regularly

## What This Demonstrates

- Cloud infrastructure provisioning

- Secure access automation

- Linux permissions management

- AWS CLI proficiency

- DevOps-ready troubleshooting mindset
