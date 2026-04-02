# EC2 Secure Provisioning with Key-Based SSH Authentication (AWS CLI)

> **Competency Domain:** AWS Compute | Identity and Access | Linux Systems Administration
> **Environment:** AWS CLI | Region: `us-east-1` | Shell: Bash

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1: Verify AWS Identity](#step-1-verify-aws-identity)
  - [Step 2: Generate SSH Key Pair](#step-2-generate-ssh-key-pair)
  - [Step 3: Retrieve Latest Amazon Linux 2 AMI](#step-3-retrieve-latest-amazon-linux-2-ami)
  - [Step 4: Create Security Group with SSH Ingress Rule](#step-4-create-security-group-with-ssh-ingress-rule)
  - [Step 5: Configure EC2 User-Data Bootstrap Script](#step-5-configure-ec2-user-data-bootstrap-script)
  - [Step 6: Launch EC2 Instance](#step-6-launch-ec2-instance)
  - [Step 7: Retrieve Public IP Address](#step-7-retrieve-public-ip-address)
  - [Step 8: Establish SSH Session](#step-8-establish-ssh-session)
- [Validation](#validation)
- [Key Decisions](#key-decisions)
- [Security Considerations](#security-considerations)
- [Lessons Learned](#lessons-learned)
- [What This Demonstrates](#what-this-demonstrates)

---

## Overview

This project documents the end-to-end provisioning of an Amazon EC2 instance using the AWS CLI exclusively, with no reliance on the AWS Management Console. The workflow covers identity verification, SSH key pair generation, dynamic AMI resolution, security group configuration, EC2 instance bootstrapping via user-data, and validated remote access over SSH.

The goal is to demonstrate hands-on operational fluency across:

- AWS Identity and Access Management (IAM/STS)
- EC2 networking, security groups, and access control
- Linux file permission management and SSH hardening
- Automated instance bootstrapping via user-data scripts
- Secure remote access patterns using key-based authentication

This workflow mirrors production DevOps and Cloud Engineering practices used during infrastructure provisioning, environment standup, and secure access configuration.

---

## Problem Statement

Manual, console-driven EC2 provisioning is not repeatable, auditable, or portable across environments. Engineering teams require a CLI-driven, scriptable workflow that:

- Provisions infrastructure programmatically without console dependency
- Injects SSH credentials securely at boot time via user-data
- Enforces key-based authentication over password-based access
- Produces a fully accessible, remotely manageable instance in a single deployment pass

This lab solves that challenge using only native AWS CLI commands and standard Linux tooling.

---

## Architecture

| Attribute | Value |
|---|---|
| **Cloud Provider** | AWS |
| **Region** | `us-east-1` |
| **AMI** | Amazon Linux 2 (dynamically resolved) |
| **Instance Type** | `t2.micro` |
| **Access Method** | SSH (RSA 2048-bit key-based) |
| **Security Layer** | EC2 Security Group (TCP port 22) |
| **Bootstrap Mechanism** | EC2 user-data shell script |
| **Subnet** | Default VPC subnet (`us-east-1a`) |

---

## Prerequisites

- AWS account with EC2 and IAM permissions
- AWS CLI installed and configured (`aws configure`)
- Linux-based environment with `ssh-keygen` and `ssh` available
- Outbound internet access to reach the EC2 public IP on port 22

---

## Implementation

---

### Step 1: Verify AWS Identity

Before executing any AWS API calls, confirm the active CLI identity to ensure the correct IAM principal, account, and permission scope are in use. This prevents provisioning resources under an unintended identity.

```bash
aws sts get-caller-identity
```

**Expected Output:** A JSON payload containing `UserId`, `Account`, and `Arn`. Verify the account ID and IAM user/role match the intended target environment before proceeding.

> **Operational Note:** In multi-account or assume-role workflows, this step is critical for confirming the correct role is active prior to resource creation.

**Screenshot: STS identity confirmed and SSH directory state verified**

<img width="1034" height="412" alt="AWS STS get-caller-identity output confirming active IAM principal and account ID" src="https://github.com/user-attachments/assets/c100a1b9-b03f-4a5b-a141-66955f94ccd6" />

---

### Step 2: Generate SSH Key Pair

Generate a local RSA 2048-bit SSH key pair. The public key will be injected into the EC2 instance at boot time via user-data. The private key remains on the provisioning host and is used for authentication.

```bash
ssh-keygen -t rsa -b 2048 -f /root/.ssh/id_rsa -N ""
```

**Flag breakdown:**

- `-t rsa` -- RSA algorithm
- `-b 2048` -- 2048-bit key length (minimum recommended for production)
- `-f /root/.ssh/id_rsa` -- explicit output path for key files
- `-N ""` -- empty passphrase (acceptable in lab/automation contexts; use a passphrase in production)

After generation, confirm both key files exist:

```bash
ls /root/.ssh
```

**Expected output:** `id_rsa` (private key) and `id_rsa.pub` (public key) alongside any pre-existing files.

> **Security Note:** In production, always protect the private key with a passphrase and restrict file permissions to `600`. The public key (`id_rsa.pub`) is safe to distribute.

**Screenshot: SSH key pair generated, both key files confirmed in `/root/.ssh`, and public key content displayed**

<img width="1033" height="659" alt="ssh-keygen output showing successful RSA 2048-bit key pair generation and key file listing" src="https://github.com/user-attachments/assets/aa49f081-1ffd-4cc7-8ec5-2b3b954d6b78" />

---

### Step 3: Retrieve Latest Amazon Linux 2 AMI

Dynamically resolve the most recent Amazon Linux 2 AMI ID for the target region. Hardcoding AMI IDs is an anti-pattern in automation pipelines because AMI IDs are region-specific and updated regularly by AWS.

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'Images | sort_by(@, &CreationDate)[-1].ImageId' \
  --output text
```

**Query breakdown:**

- `--owners amazon` -- restrict results to AWS-published AMIs only
- `--filters` -- match Amazon Linux 2 HVM x86_64 GP2 AMIs by name pattern
- `sort_by(@, &CreationDate)[-1]` -- sort by creation date and select the newest image
- `--output text` -- return the AMI ID as plain text for easy variable capture

**Resolved AMI:** `ami-0199fa5fada510433`

> **Best Practice:** In CI/CD pipelines, capture this value in a variable (`AMI_ID=$(aws ec2 describe-images ...)`) and pass it downstream to avoid hardcoded IDs across environments.

**Screenshot: AMI query returning the latest Amazon Linux 2 AMI ID**

<img width="1032" height="777" alt="AWS CLI describe-images output resolving the latest Amazon Linux 2 AMI ID dynamically" src="https://github.com/user-attachments/assets/d475f0f6-56d2-4643-a16f-845435edd201" />

---

### Step 4: Create Security Group with SSH Ingress Rule

Create a dedicated security group to govern inbound network access to the EC2 instance. Then attach an ingress rule permitting SSH traffic (TCP port 22) from any source IP.

**Create the security group:**

```bash
aws ec2 create-security-group \
  --group-name datacenter-sg \
  --description "Allow SSH access"
```

**Add the SSH ingress rule:**

```bash
aws ec2 authorize-security-group-ingress \
  --group-name datacenter-sg \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

**Returned Group ID:** `sg-071a2bc44bda9d1c8`

> **Security Warning:** The CIDR `0.0.0.0/0` opens SSH to the public internet. This is acceptable in lab environments for demonstration purposes. In production, restrict to known IP ranges (e.g., your corporate egress IP or a bastion host CIDR).

> **Operational Note:** The API response includes the `SecurityGroupRuleId`, confirming the rule was applied. `"IsEgress": false` confirms this is an inbound rule.

**Screenshot: Security group created with TCP port 22 ingress rule applied and confirmed**

<img width="1028" height="742" alt="AWS CLI output showing security group creation and SSH ingress rule authorization with rule details" src="https://github.com/user-attachments/assets/9e2924d5-603e-4c39-a6a1-37a71663270a" />

---

### Step 5: Configure EC2 User-Data Bootstrap Script

Compose a shell script that will execute automatically on first boot via the EC2 user-data mechanism. This script handles SSH key injection, permission hardening, root login enablement, and SSH service restart.

```bash
cat <<EOF > /tmp/userdata.sh
#!/bin/bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh
echo "<SSH_PUBLIC_KEY>" > /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
systemctl restart sshd
EOF
```

**Script operation breakdown:**

- `mkdir -p /root/.ssh` -- ensures the `.ssh` directory exists regardless of base image state
- `chmod 700 /root/.ssh` -- restricts directory access to the root user only (required by OpenSSH)
- `echo "<SSH_PUBLIC_KEY>" > /root/.ssh/authorized_keys` -- injects the public key at boot
- `chmod 600 /root/.ssh/authorized_keys` -- enforces strict file permissions (required by OpenSSH)
- `sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/'` -- uncomments the directive if present but commented
- `sed -i 's/^PermitRootLogin.*/PermitRootLogin yes/'` -- overwrites the directive if already uncommented
- `systemctl restart sshd` -- applies the configuration changes immediately

> **Key Decision:** Two `sed` commands handle both commented and uncommented `PermitRootLogin` states. This makes the script idempotent and resilient across different base image configurations without requiring a manual `sshd_config` audit beforehand.

> **Operational Note:** User-data scripts run as root on first boot via `cloud-init`. Failures in user-data do not prevent instance launch but will result in a non-functional SSH configuration. Always validate user-data logic in a test environment first.

**Screenshot: User-data script written to `/tmp/userdata.sh` with public key injected and SSH hardening commands in place**

<img width="1035" height="735" alt="cat heredoc writing the bootstrap shell script to /tmp/userdata.sh with SSH key injection and sshd_config modification" src="https://github.com/user-attachments/assets/bf2e812a-6b10-4e79-9086-e4739113a413" />

---

### Step 6: Launch EC2 Instance

Launch the EC2 instance using the resolved AMI, `t2.micro` instance type, previously created security group, and the user-data script. A name tag is applied to the instance for identification via CLI filters in subsequent steps.

```bash
aws ec2 run-instances \
  --image-id ami-0199fa5fada510433 \
  --instance-type t2.micro \
  --security-group-ids sg-071a2bc44bda9d1c8 \
  --user-data file:///tmp/userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datacenter-ec2}]'
```

**Flag breakdown:**

- `--image-id` -- the dynamically resolved Amazon Linux 2 AMI
- `--instance-type t2.micro` -- Free Tier eligible; sufficient for SSH connectivity validation
- `--security-group-ids` -- attaches the previously created security group by ID
- `--user-data file:///tmp/userdata.sh` -- passes the bootstrap script at launch time
- `--tag-specifications` -- applies a `Name` tag used for CLI-based instance filtering

**Confirmed instance metadata from API response:**

- **Instance ID:** `i-0df4a4c03bd825f10`
- **Initial State:** `pending`
- **AMI:** `ami-0199fa5fada510433`
- **Type:** `t2.micro`
- **Availability Zone:** `us-east-1a`
- **Private IP:** `172.31.28.144`
- **Subnet:** `subnet-01f4df1938f24e9f8`
- **VPC:** `vpc-0574e6a4df0a17974`
- **Security Group:** `sg-071a2bc44bda9d1c8` (`datacenter-sg`)
- **Launch Time:** `2026-02-23T03:54:11.000Z`

> **Operational Note:** The instance state `pending` at launch is expected. The instance transitions to `running` within approximately 30 to 60 seconds. User-data execution occurs during this initialization window. Allow additional time for `cloud-init` to complete SSH configuration before attempting to connect.

**Screenshots: `run-instances` API response confirming launch parameters, network attachment, instance metadata, and state**

<img width="1017" height="855" alt="aws ec2 run-instances command execution and start of JSON response showing ReservationId and NetworkInterfaces" src="https://github.com/user-attachments/assets/97ffb5a1-b8fd-462e-999d-e889b564b3d3" />

<img width="1026" height="858" alt="EC2 run-instances response showing PrivateDnsName, PrivateIpAddress, SubnetId, VpcId, SecurityGroups, and state reason pending" src="https://github.com/user-attachments/assets/b1d26184-c6a6-4369-848e-d557c5330c5c" />

<img width="1013" height="861" alt="EC2 run-instances response showing VirtualizationType, CpuOptions, MetadataOptions, PrivateDnsNameOptions, and MaintenanceOptions" src="https://github.com/user-attachments/assets/0dba9622-4705-4796-82bf-aef693c1ad2d" />

<img width="1032" height="857" alt="EC2 run-instances response showing InstanceId i-0df4a4c03bd825f10, ImageId, State pending, InstanceType t2.micro, LaunchTime, Placement, SubnetId, VpcId, and PrivateIpAddress" src="https://github.com/user-attachments/assets/c0eaea9c-80f8-4d79-9f39-c545dfacef2c" />

---

### Step 7: Retrieve Public IP Address

Query the public IP address of the running instance by filtering on the `Name` tag applied at launch. This approach is preferred over querying by instance ID when working in environments with human-readable tag-based resource naming.

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query "Reservations[*].Instances[*].PublicIpAddress" \
  --output text
```

**Resolved Public IP:** `100.53.178.79`

> **Operational Note:** If the public IP returns empty, the instance may still be in a `pending` state or may have been launched in a subnet without auto-assign public IP enabled. Wait for the instance to reach `running` state and re-query. Alternatively, use `--instance-ids` with the instance ID returned from `run-instances` as a fallback filter.

**Screenshot: Public IP address `100.53.178.79` returned via describe-instances tag filter**

<img width="1031" height="373" alt="aws ec2 describe-instances filtering by Name tag returning public IP address 100.53.178.79" src="https://github.com/user-attachments/assets/0faefc88-a7b1-48cc-bbbc-c32c528e8a8f" />

---

### Step 8: Establish SSH Session

Connect to the EC2 instance as `root` using the private key generated in Step 2. The SSH client will perform a host key fingerprint check on first connection.

```bash
ssh root@100.53.178.79
```

**First-connection host key prompt:**

```
The authenticity of host '100.53.178.79 (100.53.178.79)' can't be established.
ECDSA key fingerprint is SHA256:VtrYD5qUppAhQVfIoi6gggJFOFaYFop+krap5NHZA9k.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '100.53.178.79' (ECDSA) to the list of known hosts.
```

Type `yes` to accept and persist the host key fingerprint in `~/.ssh/known_hosts`.

**Successful login prompt:**

```
[root@ip-172-31-28-144 ~]#
```

The Amazon Linux 2 MOTD confirms:

- AL2 End of Life is `2026-06-30`
- Amazon Linux 2023 is the recommended upgrade path (supported until `2028-03-15`)

> **Upgrade Advisory:** Given the AL2 EOL date, evaluate migrating workloads to Amazon Linux 2023 for continued security patches and AWS support.

**Screenshot: SSH session established as root, Amazon Linux 2 MOTD displayed, shell prompt confirmed**

<img width="1037" height="469" alt="SSH session to 100.53.178.79 as root confirming successful key-based authentication and Amazon Linux 2 welcome banner" src="https://github.com/user-attachments/assets/781abe44-750b-48e9-9009-3cde69f608d5" />

---

## Validation

Successful SSH login as `root` confirms all components of the provisioning chain functioned correctly:

- **IAM identity** was verified prior to provisioning
- **SSH key pair** was generated and the public key was captured for injection
- **AMI** was dynamically resolved using a filter-based query
- **Security group** was created and port 22 ingress was authorized
- **User-data script** executed at boot, injected the authorized key, and restarted the SSH daemon
- **EC2 instance** launched successfully in the default VPC
- **Public IP** was assigned and queried via CLI tag filter
- **SSH authentication** succeeded using the locally generated private key

---

## Key Decisions

| Decision | Rationale |
|---|---|
| Dynamic AMI resolution via `sort_by` | Avoids hardcoded AMI IDs that drift as AWS publishes new images |
| Two-pattern `sed` for `PermitRootLogin` | Handles both commented and active directive states for idempotent configuration |
| Tag-based IP retrieval | Human-readable and portable; avoids reliance on ephemeral instance IDs |
| `file://` prefix for user-data | Ensures correct script delivery without inline escaping issues |
| CIDR `0.0.0.0/0` for SSH | Acceptable in lab contexts; in production, restrict to known CIDR blocks |

---

## Security Considerations

- **Root login is enabled for demonstration purposes only.** This configuration is not suitable for production.
- In production environments, apply the following mitigations:
  - Disable root SSH access (`PermitRootLogin no`)
  - Create a dedicated non-root user with `sudo` privileges
  - Use IAM roles and AWS Systems Manager Session Manager to eliminate the need for open SSH ports entirely
  - Restrict security group ingress to specific CIDR ranges (e.g., VPN egress or bastion host IP)
  - Rotate SSH key pairs regularly and revoke unused keys
  - Enable EC2 Instance Metadata Service v2 (IMDSv2) to prevent SSRF attacks via the metadata endpoint
  - Use AWS CloudTrail to audit all EC2 API calls

---

## Lessons Learned

- **User-data timing matters:** SSH access attempts immediately after launch may fail if `cloud-init` has not yet completed key injection. Wait for the instance to reach `running` state and allow an additional 30 to 60 seconds before connecting.
- **Double `sed` pattern is necessary:** Amazon Linux 2 `sshd_config` ships with `PermitRootLogin` commented out. A single `sed` targeting uncommented lines would silently no-op, leaving root login disabled. Using both patterns ensures the directive is applied regardless of initial state.
- **Tag-based querying is more maintainable than ID-based querying** in documentation and automation, since IDs change with every new deployment while tags persist as a naming convention.
- **`file://` for user-data avoids encoding bugs:** Passing scripts inline with `--user-data` requires proper shell escaping. Using `file://` with a locally written script eliminates this complexity.

---

## What This Demonstrates

- **AWS CLI fluency:** End-to-end infrastructure provisioning without console dependency
- **EC2 lifecycle management:** Instance launch, state transitions, and metadata querying
- **SSH key management:** Key generation, permission hardening, and authorized_keys injection
- **Security group configuration:** Programmatic ingress rule management
- **Linux systems administration:** `sshd_config` manipulation, `chmod`, `systemctl` usage
- **Bootstrapping automation:** user-data scripting with `cloud-init`
- **DevOps-ready troubleshooting mindset:** Idempotent scripting and resilient configuration patterns

