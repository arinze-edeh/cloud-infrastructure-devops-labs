# Provisioning an EC2 Instance via AWS CLI: A Production-Grade Infrastructure Workflow

> **Discipline:** Cloud Infrastructure | DevOps Engineering
> **Domain:** AWS EC2 | CLI Automation | Foundational Provisioning
> **Environment:** us-east-1 | Amazon Linux 2023 | t2.micro

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Summary](#architecture-summary)
- [Environment Configuration](#environment-configuration)
- [Tools and Services Used](#tools-and-services-used)
- [Step 1: Confirm AWS CLI Region Configuration](#step-1-confirm-aws-cli-region-configuration)
- [Step 2: Create the EC2 Key Pair](#step-2-create-the-ec2-key-pair)
- [Step 3: Secure the Private Key](#step-3-secure-the-private-key)
- [Step 4: Identify the Latest Amazon Linux 2023 AMI](#step-4-identify-the-latest-amazon-linux-2023-ami)
- [Step 5: Identify the Default VPC](#step-5-identify-the-default-vpc)
- [Step 6: Identify a Subnet in the Default VPC](#step-6-identify-a-subnet-in-the-default-vpc)
- [Step 7: Retrieve the Default Security Group ID](#step-7-retrieve-the-default-security-group-id)
- [Step 8: Launch the EC2 Instance](#step-8-launch-the-ec2-instance)
- [Step 9: Verify the Instance ID](#step-9-verify-the-instance-id)
- [Step 10: Confirm the Instance is Running](#step-10-confirm-the-instance-is-running)
- [Final Result](#final-result)
- [Security and Operational Best Practices](#security-and-operational-best-practices)
- [Troubleshooting and Edge Cases](#troubleshooting-and-edge-cases)
- [Real-World Relevance](#real-world-relevance)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This project documents the complete, end-to-end provisioning of an Amazon EC2 instance using the **AWS Command Line Interface (CLI)**. Every step follows the exact sequence executed in the terminal, from confirming CLI region configuration through to validating that the instance reached a `running` state.

The workflow demonstrates hands-on competence with programmatic resource discovery, secure key management, and instance lifecycle management. It follows a **problem-solution-implementation-validation** structure throughout and is suitable for onboarding engineers, production handoff documentation, and portfolio reference.

---

## Problem Statement

Infrastructure teams require a repeatable, auditable, and automation-compatible approach to provisioning compute resources. Console-based provisioning is error-prone, non-scriptable, and cannot be integrated into CI/CD pipelines. The CLI-first approach addresses this by:

- Producing **reproducible, scriptable** provisioning sequences.
- Enabling **programmatic validation** of resource state at each stage.
- Establishing a direct path toward **infrastructure-as-code** tooling such as Terraform and CloudFormation.
- Providing a **full audit trail** through CloudTrail logging of every API call.

---

## Architecture Summary

```
IAM Credentials (configured via AWS CLI)
        |
        v
Default VPC: vpc-047eb809070ac5824  (us-east-1)
        |
        +---> Default Subnet:         subnet-03c3f0cfab32f1741
        |
        +---> Default Security Group: sg-010a9fc12de2386b2
        |
        +---> EC2 Key Pair:           devops-kp (RSA, devops-kp.pem)
        |
        v
EC2 Instance: devops-ec2
  Instance ID:   i-07fbce84650910c86
  AMI:           ami-0e3008cbd8722baf0 (Amazon Linux 2023)
  Instance Type: t2.micro
  State:         running
```

---

## Environment Configuration

| Parameter | Value |
|---|---|
| **Region** | us-east-1 |
| **Instance Name** | devops-ec2 |
| **Instance Type** | t2.micro |
| **AMI** | ami-0e3008cbd8722baf0 (Amazon Linux 2023, x86_64) |
| **Key Pair Name** | devops-kp |
| **Key Type** | RSA |
| **Local Key File** | devops-kp.pem |
| **VPC** | vpc-047eb809070ac5824 (Default) |
| **Subnet** | subnet-03c3f0cfab32f1741 |
| **Security Group** | sg-010a9fc12de2386b2 (Default) |
| **Instance ID** | i-07fbce84650910c86 |

---

## Tools and Services Used

- **AWS EC2** - compute provisioning and instance lifecycle management
- **AWS CLI** - programmatic interface for all AWS resource operations
- **Amazon Linux 2023 AMI** - AWS-maintained, hardened base image for x86_64
- **Default VPC and Security Group** - foundational network layer for the region
- **Linux Shell** - command execution, key management, and permission enforcement

---

## Step 1: Confirm AWS CLI Region Configuration

**Intent:** Verify that the AWS CLI is targeting the correct region before executing any resource operations. All subsequent API calls inherit this region context, making verification the mandatory first gate of any provisioning workflow.

**Command:**

```bash
aws configure get region
```

**Output:**

```
us-east-1
```

**Validation:** The command must return `us-east-1`. If the region is incorrect or empty, update it before proceeding:

```bash
aws configure set region us-east-1
aws configure get region   # Re-verify
```

> **Operational Note:** In role-based or CI/CD environments, the region is typically injected via the `AWS_DEFAULT_REGION` environment variable rather than stored in a named profile. Always verify the active region explicitly before any provisioning run to prevent cross-region resource sprawl and unexpected billing.

**Screenshot: AWS CLI region confirmed as us-east-1**

<img width="1034" height="696" alt="image" src="https://github.com/user-attachments/assets/64b72aaf-dbc2-4e27-b98e-2ab1bfbbee5c" />

---

## Step 2: Create the EC2 Key Pair

**Intent:** Generate an RSA key pair named `devops-kp` and capture the private key locally as `devops-kp.pem`. AWS returns the private key **only once**, at creation time. Failure to save it here means permanent loss of key-based SSH access to any instance using this pair.

**Command:**

```bash
aws ec2 create-key-pair \
  --key-name devops-kp \
  --key-type rsa \
  --query 'KeyMaterial' \
  --output text > devops-kp.pem
```

**Expected Behavior:**
- No terminal output is printed. The private key material is redirected directly into `devops-kp.pem`.
- The key pair `devops-kp` is registered in AWS EC2 for the `us-east-1` region.

**Validation:**

```bash
ls -lh devops-kp.pem
head -1 devops-kp.pem   # Must return: -----BEGIN RSA PRIVATE KEY-----
```

> **Risk:** Loss of `devops-kp.pem` means permanent loss of SSH access to any instance using this key pair. Recovery requires replacing the key via AWS Systems Manager Session Manager or re-imaging from a snapshot.

> **Operational Note:** Never commit `.pem` files to version control. Add `*.pem` to your global `.gitignore`. For team environments, store private keys in AWS Secrets Manager or HashiCorp Vault with fine-grained IAM access controls and full audit trails.

**Screenshot: Key pair creation command executed and private key written to devops-kp.pem**

<img width="1026" height="720" alt="image" src="https://github.com/user-attachments/assets/bf048f62-7cd0-48dc-8b8e-80df98149cb3" />

---

## Step 3: Secure the Private Key

**Intent:** Restrict the private key file permissions to owner-read-only (`400`). The SSH client enforces this requirement and will refuse to use a key file with broader permissions, raising an `UNPROTECTED PRIVATE KEY FILE` error and aborting the connection.

**Commands:**

```bash
chmod 400 devops-kp.pem
ls -l devops-kp.pem
```

**Output:**

```
-r-------- 1 root root 1675 Feb  1 16:39 devops-kp.pem
```

**Validation:** The permission string must be exactly `-r--------`. Any writable bit indicates an insecure state that must be corrected before attempting SSH access.

> **Operational Note:** On shared or multi-user systems, also verify file ownership with `stat devops-kp.pem` to confirm no other system user can access the key via group or world permissions. For production environments, AWS Systems Manager Session Manager eliminates the need for key-based SSH entirely and provides IAM-controlled, audited shell access without opening port 22.

**Screenshot: chmod 400 applied and file permissions verified as -r--------**

<img width="1026" height="720" alt="image" src="https://github.com/user-attachments/assets/bf048f62-7cd0-48dc-8b8e-80df98149cb3" />

---

## Step 4: Identify the Latest Amazon Linux 2023 AMI

**Intent:** Dynamically retrieve the most recent Amazon Linux 2023 AMI ID for the `x86_64` architecture in `us-east-1`. Hardcoding AMI IDs is an operational anti-pattern because AMIs are region-specific, architecture-specific, and periodically superseded by security-patched and feature-updated releases.

**Command:**

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
  --query "Images | sort_by(@, &CreationDate)[-1].ImageId" \
  --output text
```

**Output:**

```
ami-0e3008cbd8722baf0
```

**Validation:** Confirm the AMI is in `available` state before referencing it in a launch command:

```bash
aws ec2 describe-images \
  --image-ids ami-0e3008cbd8722baf0 \
  --query "Images[0].[State,Name,CreationDate]" \
  --output table
```

> **Operational Note:** The `sort_by(@, &CreationDate)[-1]` JMESPath expression selects the newest matching AMI by creation date. For production deployments, pin to a validated, tested AMI ID after qualification to prevent uncontrolled drift from automatic image updates. AWS Systems Manager Parameter Store path `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64` provides a managed, always-current AMI reference for use in automation.

**Screenshot: Latest Amazon Linux 2023 AMI ID dynamically retrieved as ami-0e3008cbd8722baf0**

<img width="1026" height="720" alt="image" src="https://github.com/user-attachments/assets/bf048f62-7cd0-48dc-8b8e-80df98149cb3" />

---

## Step 5: Identify the Default VPC

**Intent:** Retrieve the VPC ID of the default VPC in `us-east-1`. The default VPC is pre-configured with a set of public subnets, a main route table with internet gateway attachment, and a default security group. Its ID is required to scope both the subnet and security group discovery steps that follow.

**Command:**

```bash
aws ec2 describe-vpcs \
  --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" \
  --output text
```

**Output:**

```
vpc-047eb809070ac5824
```

**Validation:** The returned value must begin with `vpc-`. A `None` response means no default VPC exists and one must be created:

```bash
aws ec2 create-default-vpc
```

> **Operational Note:** Enterprise environments typically delete the default VPC as a governance control and replace it with custom VPCs featuring private/public subnet tiers, NAT gateways, VPC Flow Logs, and strict network ACLs enforced via AWS Organizations Service Control Policies. This lab uses the default VPC to keep the focus on the EC2 provisioning workflow itself.

**Screenshot: Default VPC ID vpc-047eb809070ac5824 returned**

<img width="1026" height="720" alt="image" src="https://github.com/user-attachments/assets/bf048f62-7cd0-48dc-8b8e-80df98149cb3" />

---

## Step 6: Identify a Subnet in the Default VPC

**Intent:** Retrieve a valid subnet ID within the default VPC. The subnet determines the Availability Zone where the instance will be placed and must be within the same VPC as the security group used in the launch command.

**Command:**

```bash
aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=vpc-047eb809070ac5824 \
  --query "Subnets[0].SubnetId" \
  --output text
```

**Output:**

```
subnet-03c3f0cfab32f1741
```

**Validation:** List all available subnets in the VPC with their AZ and state to confirm placement options:

```bash
aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=vpc-047eb809070ac5824 \
  --query "Subnets[].[SubnetId,AvailabilityZone,State]" \
  --output table
```

> **Operational Note:** In production, subnet selection must be intentional by Availability Zone for high-availability designs. Auto Scaling Groups distribute instances across multiple subnets in different AZs by default, which is the standard pattern for resilient, fault-tolerant workloads. Subnets should also be classified as public (with a route to an internet gateway) or private (with a route to a NAT gateway) based on the intended exposure of the workload.

**Screenshot: Subnet ID subnet-03c3f0cfab32f1741 returned within the default VPC**

![Step 6 - Subnet Identified](screenshots/img-05-keypair-images-vpc-subnet.png)

---

## Step 7: Retrieve the Default Security Group ID

**Intent:** Retrieve the ID of the default security group associated with the default VPC. The default security group permits all inbound traffic from other members of the same group and all outbound traffic. Its ID is required as a parameter for the `run-instances` command.

**Command:**

```bash
aws ec2 describe-security-groups \
  --filters Name=vpc-id,Values=vpc-047eb809070ac5824 Name=group-name,Values=default \
  --query "SecurityGroups[0].GroupId" \
  --output text
```

**Output:**

```
sg-010a9fc12de2386b2
```

**Validation:** Confirm the security group VPC association and review its rules:

```bash
aws ec2 describe-security-groups \
  --group-ids sg-010a9fc12de2386b2 \
  --query "SecurityGroups[0].[GroupName,VpcId,Description]" \
  --output table
```

> **Operational Note:** The default security group is appropriate only for foundational lab use. Production instances must use purpose-built security groups with explicit, least-privilege ingress rules. Port 22 (SSH) must never be open to `0.0.0.0/0` in any production context. Prefer AWS Systems Manager Session Manager to eliminate the need for SSH-based access entirely, removing the port 22 attack surface from the equation.

**Screenshot: Default security group ID sg-010a9fc12de2386b2 retrieved, scoped to the default VPC**

![Step 7 - Security Group Retrieved](screenshots/img-06-subnet-secgroup.png)

---

## Step 8: Launch the EC2 Instance

**Intent:** Provision the EC2 instance by combining all previously collected identifiers into a single atomic `run-instances` command. This step creates the compute resource with the specified image, size, network placement, security group, key pair, and name tag.

**Command:**

```bash
aws ec2 run-instances \
  --image-id ami-0e3008cbd8722baf0 \
  --instance-type t2.micro \
  --key-name devops-kp \
  --security-group-ids sg-010a9fc12de2386b2 \
  --subnet-id subnet-03c3f0cfab32f1741 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-ec2}]'
```

**Expected Output:** AWS returns a JSON object immediately. Key fields to note and record:

| JSON Field | Value | Purpose |
|---|---|---|
| `InstanceId` | `i-07fbce84650910c86` | Primary identifier for all subsequent operations |
| `State.Name` | `pending` | Initial launch state; instance is being allocated |
| `ImageId` | `ami-0e3008cbd8722baf0` | Confirms correct AMI was used |
| `InstanceType` | `t2.micro` | Confirms correct instance sizing |
| `KeyName` | `devops-kp` | Confirms SSH key association |
| `SubnetId` | `subnet-03c3f0cfab32f1741` | Confirms correct network placement |
| `VpcId` | `vpc-047eb809070ac5824` | Confirms correct VPC scope |

> **Operational Note:** For production workloads, extend this command with `--user-data` to run bootstrap scripts at first boot (agent installation, service configuration, registration with configuration management), and `--iam-instance-profile` to attach an IAM role so the instance can access AWS services without storing credentials on disk.

**Screenshot Part 1: run-instances command executed with full JSON response, instance in pending state**

![Step 8 Part 1 - Run Instances JSON](screenshots/img-10-run-instances-json.png)

**Screenshot Part 2: Continued JSON response confirming InstanceId, State, SubnetId, VpcId, and PrivateIpAddress**

![Step 8 Part 2 - Run Instances Metadata](screenshots/img-11-verify-running.png)

---

## Step 9: Verify the Instance ID

**Intent:** Programmatically confirm the Instance ID of the newly launched instance by querying against the `Name` tag applied at launch. This validates that the tag was applied correctly and retrieves the canonical identifier for use in all subsequent operational commands.

**Command:**

```bash
aws ec2 describe-instances \
  --filters Name=tag:Name,Values=devops-ec2 \
  --query "Reservations[].Instances[].InstanceId" \
  --output text
```

**Output:**

```
i-07fbce84650910c86
```

**Validation:** The returned Instance ID must match the `InstanceId` field from the `run-instances` JSON response in Step 8. A mismatch or empty result indicates a tagging error or a filter typo.

> **Operational Note:** Tag-based querying is the standard operational pattern for managing resources at scale. It enables discovery and automation without maintaining external state or hardcoding Instance IDs in scripts. Consistent tagging across all resources is a prerequisite for cost attribution, automated compliance checks, and operational runbook execution.

---

## Step 10: Confirm the Instance is Running

**Intent:** Confirm that the EC2 instance has fully transitioned from `pending` to `running`. This is the final validation gate of the provisioning workflow. An instance in the `running` state has completed hardware allocation, network interface attachment, and initial boot sequence.

**Command:**

```bash
aws ec2 describe-instances \
  --filters Name=tag:Name,Values=devops-ec2 \
  --query "Reservations[].Instances[].State.Name" \
  --output text
```

**Output:**

```
running
```

**Alternative: Block until running using the built-in waiter**

```bash
aws ec2 wait instance-running \
  --instance-ids i-07fbce84650910c86
echo "Instance is now running."
```

**Connect via SSH once the instance is running:**

```bash
# Retrieve the public IP address
aws ec2 describe-instances \
  --filters Name=tag:Name,Values=devops-ec2 \
  --query "Reservations[].Instances[].PublicIpAddress" \
  --output text

# Connect via SSH
ssh -i devops-kp.pem ec2-user@<PUBLIC_IP>
```

> **Operational Note:** `aws ec2 wait instance-running` polls the EC2 API every 15 seconds and exits cleanly when `running` state is confirmed, making it ideal for pipeline gate steps. For full SSH readiness validation (system and instance health checks passed), use `aws ec2 wait instance-status-ok` instead.

**Screenshot: Instance ID i-07fbce84650910c86 retrieved and state confirmed as running**

![Step 9 and 10 - Instance ID and Running State Confirmed](screenshots/img-11-verify-running.png)

---

## Final Result

| Outcome | Detail | Status |
|---|---|---|
| EC2 instance provisioned via AWS CLI | Instance ID: `i-07fbce84650910c86` | Confirmed |
| Correct AMI used | `ami-0e3008cbd8722baf0` (Amazon Linux 2023) | Confirmed |
| Instance type correct | `t2.micro` | Confirmed |
| Key pair created and secured | `devops-kp` / `devops-kp.pem` at `chmod 400` | Confirmed |
| VPC and subnet correct | `vpc-047eb809070ac5824` / `subnet-03c3f0cfab32f1741` | Confirmed |
| Security group applied | `sg-010a9fc12de2386b2` (Default) | Confirmed |
| Resource tagged | `Name=devops-ec2` | Confirmed |
| Instance state | `running` | Confirmed |

---

## Security and Operational Best Practices

**Key Management**
- Never commit `.pem` files to version control. Add `*.pem` to `.gitignore` globally and enforce this at the repository policy level.
- Store private keys in AWS Secrets Manager or HashiCorp Vault for team-shared access with full audit trails and automatic rotation capability.
- Prefer AWS Systems Manager Session Manager over key-based SSH in production. Session Manager requires no open port 22, records all session activity to CloudTrail and S3, and enforces access through IAM policies.

**Network Security**
- The default security group is appropriate only for foundational labs. Production instances must use dedicated security groups with explicit, least-privilege ingress rules.
- Never expose port 22 to `0.0.0.0/0`. Restrict SSH source CIDRs to known IP ranges or eliminate key-based SSH entirely via Session Manager.
- Enable VPC Flow Logs to capture all accepted and rejected traffic for audit and forensic purposes.

**Resource Governance**
- Apply consistent tags (`Name`, `Environment`, `Owner`, `Project`, `CostCenter`) to all resources at launch time for cost attribution, operational traceability, and automated policy enforcement.
- Enable AWS Cost Explorer and configure billing alerts to catch runaway costs from forgotten running instances.

**Automation Readiness**
- This CLI workflow maps directly to Terraform `aws_instance` resource definitions, CloudFormation `AWS::EC2::Instance` templates, and Ansible `amazon.aws.ec2_instance` playbooks.
- Use `--dry-run` with supported AWS CLI commands to validate IAM permissions before executing provisioning or destructive operations.

---

## Troubleshooting and Edge Cases

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `UnauthorizedOperation` | IAM user or role lacks `ec2:RunInstances` or related permissions | Attach the required IAM policy or escalate to an administrator |
| `InvalidKeyPair.NotFound` | Key pair name in the command does not match the registered name | Verify with `aws ec2 describe-key-pairs --key-names devops-kp` |
| `InvalidAMIID.NotFound` | AMI ID is not available in the target region | Re-run the `describe-images` query scoped to `us-east-1` |
| `VPCIdNotSpecified` | No default VPC exists in the region | Run `aws ec2 create-default-vpc` |
| `UNPROTECTED PRIVATE KEY FILE` on SSH | Key file permissions are broader than `400` | Run `chmod 400 devops-kp.pem` and retry |
| Instance stuck in `pending` | Insufficient On-Demand capacity for the instance type in the AZ | Retry with a subnet in a different AZ or use an alternative instance type |
| SSH connection times out | Security group does not allow inbound TCP port 22 | Add a security group inbound rule for port 22 from your source IP |
| `describe-instances` returns empty | Tag filter value mismatch or typo | Verify tag directly with `aws ec2 describe-instances --instance-ids i-07fbce84650910c86` |

---

## Real-World Relevance

This workflow directly mirrors the operational patterns used by DevOps and Cloud Engineers at scale:

- **CLI-first provisioning** is the baseline competency for scripted infrastructure workflows, GitOps pipelines, and infrastructure-as-code migration projects.
- **Dynamic resource discovery** (VPC, subnet, AMI, security group) via CLI replaces error-prone console navigation and produces environment-agnostic, reusable provisioning scripts.
- **Tag-based state validation** establishes the foundation for health checks, readiness gates, and deployment pipeline integration.

Teams operating at scale extend this exact pattern with infrastructure tooling (Terraform, Pulumi), configuration management (Ansible, Chef), and immutable image pipelines (Packer) to achieve repeatable, auditable, and zero-touch deployments across hundreds or thousands of instances.

---

## Skills Demonstrated

- **AWS CLI proficiency** - multi-service command execution, JMESPath querying, and output format control
- **EC2 provisioning fundamentals** - complete instance lifecycle from resource discovery to running-state validation
- **Cloud resource discovery** - programmatic retrieval of VPC, subnet, security group, and AMI identifiers
- **Secure key pair management** - RSA key generation, safe local storage, and AWS key registration
- **Linux permissions enforcement** - `chmod 400` applied and verified for SSH client compliance
- **Operational discipline** - resource tagging, step-by-step validation, and documentation standards consistent with production engineering expectations

























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
