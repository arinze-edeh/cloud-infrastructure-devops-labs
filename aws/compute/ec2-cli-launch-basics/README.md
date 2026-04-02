# Provisioning EC2 Instances via AWS CLI: A Production-Style Infrastructure Workflow

> **Discipline:** Cloud Infrastructure | DevOps Engineering
> **Domain:** AWS EC2 | CLI Automation | Foundational Provisioning
> **Complexity:** Foundational to Intermediate
> **Environment:** us-east-1 | Amazon Linux | t2.micro

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Summary](#architecture-summary)
- [Environment Configuration](#environment-configuration)
- [Tools and Services Used](#tools-and-services-used)
- [Step 1: Confirm AWS CLI Configuration](#step-1-confirm-aws-cli-configuration)
- [Step 2: Identify the Default VPC](#step-2-identify-the-default-vpc)
- [Step 3: Identify a Subnet in the Default VPC](#step-3-identify-a-subnet-in-the-default-vpc)
- [Step 4: Create the EC2 Key Pair](#step-4-create-the-ec2-key-pair)
- [Step 5: Secure the Private Key](#step-5-secure-the-private-key)
- [Step 6: Identify the Amazon Linux AMI](#step-6-identify-the-amazon-linux-ami)
- [Step 7: Launch the EC2 Instance](#step-7-launch-the-ec2-instance)
- [Step 8: Verify EC2 Instance Status](#step-8-verify-ec2-instance-status)
- [Final Result](#final-result)
- [Security and Operational Best Practices](#security-and-operational-best-practices)
- [Troubleshooting and Edge Cases](#troubleshooting-and-edge-cases)
- [Real-World Relevance](#real-world-relevance)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This project documents the end-to-end provisioning of an Amazon EC2 instance using the **AWS Command Line Interface (CLI)**. The workflow covers network resource discovery, key pair creation, AMI selection, instance launch, and programmatic state validation.

This documentation follows a **problem-solution-implementation-validation** structure and is intended as a reference for Cloud and DevOps engineers operating in CLI-driven or automation-first environments.

---

## Problem Statement

Infrastructure teams frequently need to provision compute resources rapidly, repeatably, and without relying on the AWS Management Console. Console-based provisioning introduces human error, lacks auditability, and cannot be integrated into CI/CD pipelines. The CLI-first approach solves this by:

- Enabling **scriptable, reproducible** provisioning workflows.
- Providing **programmatic validation** of resource state.
- Supporting **infrastructure-as-code** readiness through command composability.
- Establishing a foundation for tools like **Terraform**, **Ansible**, and **AWS CloudFormation**.

---

## Architecture Summary

```
IAM Credentials (configured via AWS CLI)
        |
        v
Default VPC (us-east-1)
        |
        +---> Default Subnet (AZ: us-east-1x)
        |
        +---> Default Security Group
        |
        +---> EC2 Key Pair (RSA, devops-kp)
        |
        v
EC2 Instance: devops-ec2
  - AMI:           Amazon Linux 2023
  - Instance Type: t2.micro
  - State:         running
```

---

## Environment Configuration

| Parameter | Value |
|---|---|
| **Region** | us-east-1 |
| **Instance Name** | devops-ec2 |
| **Instance Type** | t2.micro |
| **AMI** | Amazon Linux 2023 (al2023-ami-*-x86_64) |
| **Key Pair** | devops-kp (RSA) |
| **Security Group** | Default (VPC-scoped) |
| **Network** | Default VPC and Default Subnet |

---

## Tools and Services Used

- **AWS EC2** - compute provisioning and instance management
- **AWS CLI** - programmatic interface for all resource operations
- **Amazon Linux 2023 AMI** - hardened, AWS-optimized base image
- **Default VPC and Security Group** - foundational network layer
- **Linux Shell** - key management, permissions enforcement, and command execution

---

## Step 1: Confirm AWS CLI Configuration

**Intent:** Verify that the AWS CLI is properly configured with valid credentials and the correct target region before executing any provisioning commands. Misconfigured credentials are the most common cause of silent failures or unintended cross-region deployments.

**Command:**

```bash
aws configure list
# Alternatively, to confirm only the active region:
aws configure get region
```

**Expected Output:**
- Active credentials are present (access key and secret key configured).
- Region is set to `us-east-1`.

**Validation:**
If the region is not `us-east-1`, update it with:
```bash
aws configure set region us-east-1
```

> **Operational Note:** In CI/CD pipelines and IAM role-based environments, credentials are injected via environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`) rather than static profiles. Always prefer IAM roles over long-lived access keys in production.

**Screenshot: AWS CLI region configuration confirmed as us-east-1**

<img width="1034" height="696" alt="image" src="https://github.com/user-attachments/assets/64b72aaf-dbc2-4e27-b98e-2ab1bfbbee5c" />

---

## Step 2: Identify the Default VPC

**Intent:** Retrieve the VPC ID of the default VPC in the target region. The default VPC provides pre-configured networking (subnets, route tables, internet gateway) and is suitable for non-production and foundational workloads.

**Command:**

```bash
aws ec2 describe-vpcs \
  --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" \
  --output text
```

**Expected Output:**
```
vpc-047eb809070ac5824
```

**Validation:** The returned value must begin with `vpc-`. An empty response or `None` indicates no default VPC exists in the region and one must be created:

```bash
aws ec2 create-default-vpc
```

> **Operational Note:** In enterprise environments, the default VPC is typically deleted and replaced with a custom VPC featuring private/public subnet tiers, NAT gateways, and strict network ACLs. For this foundational lab, the default VPC is appropriate.

**Screenshot: Default VPC ID successfully retrieved via AWS CLI**

![Step 2 - Describe VPCs](screenshots/step2-describe-vpcs.png)

---

## Step 3: Identify a Subnet in the Default VPC

**Intent:** Retrieve a valid subnet ID within the default VPC. The subnet defines the Availability Zone (AZ) where the instance will be placed and must be reachable from within the VPC.

**Command:**

```bash
aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=<DEFAULT_VPC_ID> \
  --query "Subnets[0].SubnetId" \
  --output text
```

Replace `<DEFAULT_VPC_ID>` with the value returned in Step 2. For example:

```bash
aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=vpc-047eb809070ac5824 \
  --query "Subnets[0].SubnetId" \
  --output text
```

**Expected Output:**
```
subnet-03c3f0cfab32f1741
```

**Validation:** Confirm the subnet is in a healthy state and associated with the correct VPC:

```bash
aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=vpc-047eb809070ac5824 \
  --query "Subnets[].[SubnetId,AvailabilityZone,State]" \
  --output table
```

> **Operational Note:** In production, prefer explicit subnet targeting by AZ for high-availability architectures. Deploying across multiple AZs using Auto Scaling Groups (ASGs) is the standard pattern for resilient workloads.

**Screenshot Part 1: Subnet query command execution**

![Step 3a - Describe Subnets](screenshots/step3a-describe-subnets.png)

**Screenshot Part 2: Subnet ID returned for the default VPC**

![Step 3b - Describe Subnets Output](screenshots/step3b-describe-subnets.png)

---

## Step 4: Create the EC2 Key Pair

**Intent:** Generate an RSA key pair to enable SSH access to the EC2 instance. The private key material is returned once at creation time and must be saved immediately. It cannot be retrieved again from AWS.

**Command:**

```bash
aws ec2 create-key-pair \
  --key-name devops-kp \
  --key-type rsa \
  --query 'KeyMaterial' \
  --output text > devops-kp.pem
```

**Expected Output:**
- No terminal output (the private key is written directly to `devops-kp.pem`).
- File `devops-kp.pem` is created in the working directory.

**Validation:** Confirm the file exists and contains valid PEM content:

```bash
ls -lh devops-kp.pem
head -1 devops-kp.pem  # Should output: -----BEGIN RSA PRIVATE KEY-----
```

> **Risk:** If `devops-kp.pem` is lost, SSH access to the instance is permanently unavailable. The only recovery path is to create a new key pair and replace it via the instance's user data or Systems Manager Session Manager.

> **Operational Note:** Never commit `.pem` files to version control. Add `*.pem` to `.gitignore` as a standing rule. For team environments, store private keys in a secrets manager such as AWS Secrets Manager or HashiCorp Vault.

**Screenshot: Key pair created and private key saved to devops-kp.pem**

![Step 4 - Create Key Pair](screenshots/step4-create-keypair.png)

---

## Step 5: Secure the Private Key

**Intent:** Restrict file system permissions on the private key to owner-read-only (`400`). SSH clients enforce this requirement and will reject keys with overly permissive permissions with an `UNPROTECTED PRIVATE KEY FILE` error.

**Commands:**

```bash
chmod 400 devops-kp.pem
ls -l devops-kp.pem
```

**Expected Output:**
```
-r-------- 1 root root 1675 Feb  1 16:39 devops-kp.pem
```

**Validation:** The permission string must read `-r--------`. Any writable bits indicate an insecure state.

> **Operational Note:** On shared or multi-user systems, also verify the file ownership (`chown`) to ensure the key is not accessible by other system users. In containerized or ephemeral environments, consider using AWS Systems Manager Session Manager to eliminate the need for key-based SSH access entirely.

**Screenshot: Private key permissions hardened to read-only (chmod 400 verified)**

![Step 5 - Secure Key Permissions](screenshots/step5-secure-key.png)

---

## Step 6: Identify the Amazon Linux AMI

**Intent:** Dynamically retrieve the latest Amazon Linux 2023 AMI ID for the `x86_64` architecture in the target region. Hardcoding AMI IDs is an anti-pattern because AMIs are region-specific and periodically superseded by updated releases.

**Command:**

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
  --query "Images | sort_by(@, &CreationDate)[-1].ImageId" \
  --output text
```

**Expected Output:**
```
ami-0e3008cbd8722baf0
```

**Validation:** Confirm the AMI is available and not deprecated:

```bash
aws ec2 describe-images \
  --image-ids ami-0e3008cbd8722baf0 \
  --query "Images[0].[State,Name,CreationDate]" \
  --output table
```

> **Operational Note:** For production workloads, consider pinning to a specific AMI ID after validation and testing, rather than using the latest dynamically. This ensures environment consistency across deployments. Use AWS Systems Manager Parameter Store (`/aws/service/ami-amazon-linux-latest/`) as a managed source of AMI IDs in automation scripts.

**Screenshot: Latest Amazon Linux 2023 AMI ID dynamically retrieved**

![Step 6 - Describe Images](screenshots/step6-describe-images.png)

---

## Step 7: Launch the EC2 Instance

**Intent:** Provision the EC2 instance with all pre-collected resource identifiers. This single command brings together the AMI, instance type, key pair, network placement, security group, and resource tagging in one atomic operation.

**Command:**

```bash
aws ec2 run-instances \
  --image-id ami-0e3008cbd8722baf0 \
  --instance-type t2.micro \
  --key-name devops-kp \
  --subnet-id subnet-03c3f0cfab32f1741 \
  --security-group-ids sg-010a9fc12de2386b2 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-ec2}]'
```

**Expected Output:**
- A JSON response containing the `InstanceId` and initial state `"Name": "pending"`.

**Key response fields to note:**

| Field | Value | Purpose |
|---|---|---|
| `InstanceId` | `i-07fbce84650910c86` | Unique identifier for all future operations |
| `State.Name` | `pending` | Initial launch state (transitions to `running`) |
| `InstanceType` | `t2.micro` | Confirms correct sizing |
| `KeyName` | `devops-kp` | Confirms SSH key association |
| `SubnetId` | `subnet-03c3f0cfab32f1741` | Confirms correct network placement |

> **Operational Note:** For production workloads, extend this command with `--user-data` to run bootstrap scripts at launch (e.g., installing agents, configuring services), and `--iam-instance-profile` to attach an IAM role for secure, credential-free access to other AWS services.

**Screenshot Part 1: run-instances command execution and JSON response**

![Step 7 - Run Instances Part 1](screenshots/step7-run-instances-pt1.png)

**Screenshot Part 2: Instance metadata confirming launch parameters and pending state**

![Step 7 - Run Instances Part 2](screenshots/step7-run-instances-pt2.png)

---

## Step 8: Verify EC2 Instance Status

**Intent:** Programmatically confirm that the provisioned instance has reached the `running` state. Validation is a non-negotiable step in any provisioning workflow; it confirms the resource is operational and ready for use.

**Commands:**

```bash
# Retrieve Instance ID by name tag
aws ec2 describe-instances \
  --filters Name=tag:Name,Values=devops-ec2 \
  --query "Reservations[].Instances[].[InstanceId]" \
  --output text

# Confirm instance is in running state
aws ec2 describe-instances \
  --filters Name=tag:Name,Values=devops-ec2 \
  --query "Reservations[].Instances[].[InstanceId,State.Name]" \
  --output table
```

**Expected Output:**
```
-------------------------------------
|        DescribeInstances          |
+----------------------+------------+
|  i-07fbce84650910c86 |  running   |
+----------------------+------------+
```

**Alternative: Wait for running state with built-in waiter**

```bash
aws ec2 wait instance-running \
  --instance-ids i-07fbce84650910c86
echo "Instance is now running."
```

> **Operational Note:** The `aws ec2 wait` family of commands are ideal for scripted pipelines where subsequent steps depend on instance readiness. They poll the API on a defined interval and block until the desired state is reached or a timeout occurs. For SSH readiness specifically, use `aws ec2 wait instance-status-ok`.

**Screenshot Part 1: Instance state verification showing running status**

![Step 8 - Verify Status Part 1](screenshots/step8-verify-status-pt1.png)

**Screenshot Part 2: Final confirmation of instance ID and running state in tabular output**

![Step 8 - Verify Status Part 2](screenshots/step8-verify-status-pt2.png)

---

## Final Result

| Outcome | Status |
|---|---|
| EC2 instance provisioned via AWS CLI | Confirmed |
| Instance named `devops-ec2` in us-east-1 | Confirmed |
| Key pair `devops-kp` created and secured | Confirmed |
| Instance state verified as `running` | Confirmed |
| Resource tagging applied | Confirmed |
| Private key permissions hardened to `400` | Confirmed |

The instance `i-07fbce84650910c86` is operational and accessible via SSH using:

```bash
ssh -i devops-kp.pem ec2-user@<PUBLIC_IP>
```

Retrieve the public IP address with:

```bash
aws ec2 describe-instances \
  --filters Name=tag:Name,Values=devops-ec2 \
  --query "Reservations[].Instances[].PublicIpAddress" \
  --output text
```

---

## Security and Operational Best Practices

**Key Management**
- Private key files (`.pem`) must never be committed to version control. Add `*.pem` to `.gitignore` globally.
- In team environments, centralize key storage in AWS Secrets Manager or HashiCorp Vault.
- Prefer AWS Systems Manager Session Manager over key-based SSH for production access. Session Manager eliminates the need to expose port 22 and provides full audit logging via CloudTrail.

**Network Security**
- The default security group permits all outbound traffic and inbound traffic from the same security group. For production, define explicit ingress rules with the minimum required ports and source IP ranges.
- Never use `0.0.0.0/0` as an SSH source CIDR in security group rules. Restrict to known IP ranges or use a bastion host pattern.

**Resource Governance**
- Apply consistent tagging (`Name`, `Environment`, `Owner`, `Project`) to all resources for cost attribution and operational traceability.
- Enable AWS Cost Explorer and set billing alerts to prevent runaway costs from forgotten instances.

**Automation Readiness**
- This CLI workflow is the direct predecessor to infrastructure-as-code (IaC) tooling. The same parameters map directly to Terraform `aws_instance` resource blocks or AWS CloudFormation templates.
- Use `--dry-run` flag with supported CLI commands to validate permissions before executing destructive or cost-incurring operations.

---

## Troubleshooting and Edge Cases

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `UnauthorizedOperation` error | IAM user lacks `ec2:RunInstances` permission | Attach appropriate IAM policy or escalate to admin |
| `InvalidKeyPair.NotFound` | Key pair name mismatch | Verify with `aws ec2 describe-key-pairs` |
| `InvalidAMIID.NotFound` | AMI not available in region | Re-run the `describe-images` query for the target region |
| `VPCIdNotSpecified` | No default VPC exists | Create one with `aws ec2 create-default-vpc` |
| `UNPROTECTED PRIVATE KEY FILE` on SSH | Key file permissions too open | Run `chmod 400 devops-kp.pem` |
| Instance stuck in `pending` | Insufficient capacity for instance type | Retry with a different AZ or instance type |
| SSH connection timeout | Security group blocks port 22 | Add inbound rule for TCP port 22 from your IP |

---

## Real-World Relevance

This workflow directly mirrors the operational patterns used by DevOps and Cloud Engineers at scale:

- **CLI-first provisioning** is the baseline competency for scripted infrastructure workflows, GitOps pipelines, and IaC migration projects.
- **Programmatic resource discovery** (VPC, subnet, AMI) replaces error-prone console navigation and enables dynamic, environment-agnostic scripts.
- **State validation via CLI** establishes the foundation for health checks, readiness gates, and deployment pipeline integration.

Teams operating at FAANG scale extend this pattern with infrastructure tooling (Terraform, Pulumi), configuration management (Ansible, Chef), and immutable image pipelines (Packer) to achieve repeatable, auditable, and zero-touch deployments.

---

## Skills Demonstrated

- **AWS CLI proficiency** - multi-service command execution, JMESPath querying, and output formatting
- **EC2 provisioning fundamentals** - full instance lifecycle from resource discovery to state validation
- **Cloud resource discovery** - dynamic retrieval of VPC, subnet, security group, and AMI identifiers
- **Secure key pair management** - RSA key generation, secure storage, and Linux permission enforcement
- **Linux permissions handling** - `chmod` enforcement aligned with SSH client security requirements
- **Operational discipline** - resource tagging, validation steps, and documentation standards consistent with production engineering expectations



























# EC2 Instance Launch via AWS CLI (Foundational Provisioning)

## Overview
This project demonstrates how to provision a basic Amazon EC2 instance using the AWS CLI.
The lab focuses on understanding the **end-to-end EC2 launch workflow**, including key pair creation,
AMI selection, instance configuration, and validation.

The objective is to showcase **hands-on AWS CLI competence**, which is a core expectation for
Cloud and DevOps roles.

---

## Project Scope
- Launch an EC2 instance using AWS CLI
- Use Amazon Linux AMI
- Configure instance type, key pair, and security group
- Verify successful instance creation
- Follow AWS operational best practices

---

## Tools & Services Used
- AWS EC2
- AWS CLI
- Amazon Linux AMI
- Default VPC & Security Group
- Linux Shell Environment

---

## Environment Details
- REGION = us-east-1
- INSTANCE NAME = devops-ec2
- INSTANCE TYPE = t2.micro
- AMI = Amazon Linux
- KEY PAIR = devops-kp (RSA)
- SECURITY GRP = Default


---

## Step 1: Confirm AWS CLI Configuration

COMMAND:
aws configure list


EXPECTED:
- Credentials configured
- Region set to us-east-1

📸 Screenshot:
<img width="1034" height="696" alt="image" src="https://github.com/user-attachments/assets/64b72aaf-dbc2-4e27-b98e-2ab1bfbbee5c" />

---

## Step 2: Identify Default VPC

COMMAND:
- aws ec2 describe-vpcs
 - --filters Name=isDefault,Values=true
 - --query "Vpcs[0].VpcId"
 - --output text


EXPECTED:
- Default VPC ID returned

📸 Screenshot:
<img width="1026" height="720" alt="image" src="https://github.com/user-attachments/assets/bf048f62-7cd0-48dc-8b8e-80df98149cb3" />

---

## Step 3: Identify a Subnet in the Default VPC

COMMAND:
- aws ec2 describe-subnets
- --filters Name=vpc-id,Values=<DEFAULT_VPC_ID>
- --query "Subnets[0].SubnetId"
- --output text


EXPECTED:
- Subnet ID returned

📸 Screenshot:
<img width="1031" height="814" alt="image" src="https://github.com/user-attachments/assets/4288311b-ecd2-4e54-902c-849c361203c5" />
<img width="1023" height="839" alt="image" src="https://github.com/user-attachments/assets/fbacab15-f778-4cd4-bb7d-5d354a59f312" />

---

## Step 4: Create EC2 Key Pair

COMMAND:
- aws ec2 create-key-pair
- --key-name devops-kp
- --key-type rsa
- --query 'KeyMaterial'
- --output text > devops-kp.pem


EXPECTED:
- devops-kp.pem created locally

📸 Screenshot:
<img width="1033" height="813" alt="image" src="https://github.com/user-attachments/assets/6ac265fe-8a00-4a7c-92ad-3d328dd22c22" />

---

## Step 5: Secure the Private Key

COMMAND:
chmod 400 devops-kp.pem


VERIFY:
ls -l devops-kp.pem


EXPECTED:
- File permission: -r--------

📸 Screenshot:
<img width="1029" height="804" alt="image" src="https://github.com/user-attachments/assets/5ae60d3a-b9ab-458e-a216-8e0040f3a111" />

---

## Step 6: Identify Amazon Linux AMI

COMMAND:
- aws ec2 describe-images
- --owners amazon
- --filters Name=name,Values="amzn2-ami-hvm-*-x86_64-gp2"
- --query "Images | sort_by(@, &CreationDate)[-1].ImageId"
- --output text


EXPECTED:
- Latest Amazon Linux AMI ID returned

📸 Screenshot:
<img width="1035" height="740" alt="image" src="https://github.com/user-attachments/assets/3b694c7e-ee09-49fe-bbef-dec008669969" />

---

## Step 7: Launch EC2 Instance

COMMAND:
- aws ec2 run-instances
- --image-id <AMI_ID>
- --instance-type t2.micro
- --key-name devops-kp
- --subnet-id <SUBNET_ID>
- --security-group-ids <DEFAULT_SG_ID>
- --tag-specifications
- 'ResourceType=instance,Tags=[{Key=Name,Value=devops-ec2}]'


EXPECTED:
- Instance ID returned
- Instance state = pending → running

📸 Screenshot:
<img width="1030" height="859" alt="image" src="https://github.com/user-attachments/assets/e3083489-1a78-45f6-be54-0ae227667f25" />

---

## Step 8: Verify EC2 Instance Status

COMMAND:
- aws ec2 describe-instances
- --filters Name=tag:Name,Values=devops-ec2
- --query "Reservations[].Instances[].[InstanceId,State.Name]"
- --output table


EXPECTED:
- Instance state = running

📸 Screenshot:
<img width="1035" height="860" alt="image" src="https://github.com/user-attachments/assets/75b1ca3c-b273-41ba-accc-e38bf2dddab8" />

---

## Final Result
- EC2 instance successfully launched via AWS CLI
- Key pair securely created and managed
- Instance correctly tagged and verified
- All resources deployed in us-east-1

---

## Security & Operational Best Practices
- Private keys are never committed to version control
- Least-privilege default security group used
- Resource tagging applied for traceability
- CLI-based provisioning supports automation readiness

---

## Real-World Relevance
This workflow mirrors how DevOps and Cloud Engineers:
- Provision infrastructure using CLI tools
- Validate resources programmatically
- Operate efficiently without relying on the AWS Console

---

## Skills Demonstrated
- AWS CLI proficiency
- EC2 provisioning fundamentals
- Cloud resource discovery
- Secure key pair management
- Linux permissions handling
