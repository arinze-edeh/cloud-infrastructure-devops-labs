# Provisioning an EC2 Instance with a Static Elastic IP via AWS CLI

> **Enterprise-grade walkthrough** for provisioning, networking, and validating an Amazon EC2 instance with a persistent Elastic IP address using the AWS CLI exclusively. Designed for DevOps engineers, cloud practitioners, and onboarding teams operating in production or lab environments.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Architecture Summary](#architecture-summary)
- [Prerequisites](#prerequisites)
- [Step 1: Verify AWS Identity](#step-1-verify-aws-identity)
- [Step 2: Launch EC2 Instance](#step-2-launch-ec2-instance)
- [Step 3: Allocate Elastic IP](#step-3-allocate-elastic-ip)
- [Step 4: Associate Elastic IP with EC2](#step-4-associate-elastic-ip-with-ec2)
- [Step 5: Validate Elastic IP Association](#step-5-validate-elastic-ip-association)
- [Final Outcome](#final-outcome)
- [Operational Considerations and Best Practices](#operational-considerations-and-best-practices)
- [Risks, Edge Cases, and Troubleshooting](#risks-edge-cases-and-troubleshooting)
- [Tools and Services Used](#tools-and-services-used)

---

## Project Overview

This project demonstrates an end-to-end, CLI-driven workflow for provisioning an Amazon EC2 instance and associating a static Elastic IP (EIP) to establish a persistent, reliable public endpoint. Every operation is performed via the **AWS CLI** in the **us-east-1** region, with no reliance on the AWS Management Console.

This approach reflects production-grade DevOps standards: reproducible, auditable, and scriptable infrastructure operations that integrate naturally into CI/CD pipelines, runbooks, and IaC workflows.

---

## Problem Statement

Dynamic public IP addresses assigned to EC2 instances at launch are ephemeral. Every time an instance is stopped and started, its public IP changes. This creates operational fragility for:

- DNS records pointing to a specific IP
- Firewall allowlists requiring a stable source or destination IP
- Client-side configurations depending on a fixed endpoint

**Solution:** Allocate a VPC-scoped Elastic IP (static public IPv4) and associate it permanently with the EC2 instance, decoupling the instance lifecycle from the public IP address.

---

## Architecture Summary

| Component | Detail |
|---|---|
| **Compute** | Amazon EC2 (t2.micro) |
| **Networking** | Elastic IP (VPC scope) |
| **Region** | us-east-1 |
| **Availability Zone** | us-east-1b (auto-selected) |
| **Provisioning Method** | AWS CLI |
| **AMI** | ami-0c7217cdde317cfec (Amazon Linux) |
| **Instance Tag** | Name = datacenter-ec2 |
| **EIP Tag** | Name = datacenter-eip |

---

## Prerequisites

Before executing any CLI commands, ensure the following are in place:

- **AWS CLI v2** installed and configured (`aws configure`)
- **IAM permissions** covering: `ec2:RunInstances`, `ec2:AllocateAddress`, `ec2:AssociateAddress`, `ec2:DescribeAddresses`, `ec2:DescribeInstances`, `sts:GetCallerIdentity`
- **Active shell session** in the correct AWS account and region (`us-east-1`)
- **Default VPC** available in the target region (this lab uses the default VPC and subnet)

> **Best Practice:** Use least-privilege IAM credentials scoped to only the permissions required for this workflow. Avoid using root credentials.

---

## Step 1: Verify AWS Identity

**Intent:** Confirm the active IAM identity, account context, and permission scope before provisioning any resources. This is a critical first step in any CLI-driven workflow to prevent accidental resource creation in the wrong account or region.

```bash
aws sts get-caller-identity
```

**Expected output fields:**

- `UserId` - The unique identifier of the IAM principal
- `Account` - The AWS account ID in use
- `Arn` - The full ARN of the authenticated identity, confirming the IAM user or role context

> **Operational Tip:** Always run this command at the start of any AWS CLI session. If the output does not match the expected account and user, stop and reconfigure your CLI environment before proceeding.

**Screenshot: `aws sts get-caller-identity` output confirming IAM user, account ID, and ARN**

![aws sts get-caller-identity output](https://github.com/user-attachments/assets/262ad477-209a-44b9-80ad-e53a4f02e154)

---

## Step 2: Launch EC2 Instance

**Intent:** Provision a t2.micro EC2 instance using a known Amazon Linux AMI, with a descriptive tag applied at launch time to enable resource tracking and cost attribution from the start.

```bash
aws ec2 run-instances \
  --image-id ami-0c7217cdde317cfec \
  --count 1 \
  --instance-type t2.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datacenter-ec2}]' \
  --region us-east-1
```

**Key parameters:**

- `--image-id` - Specifies the Amazon Linux AMI to use as the base image
- `--count 1` - Launches exactly one instance
- `--instance-type t2.micro` - Free-tier eligible general-purpose compute
- `--tag-specifications` - Tags are applied at launch to ensure the instance is identifiable immediately in billing, Cost Explorer, and resource inventories
- `--region us-east-1` - Explicitly scoped to avoid unintended cross-region provisioning

**Key values returned in the response:**

- `InstanceId` - `i-0f6a39c883a6e3cb1` (required for subsequent association steps)
- `AvailabilityZone` - `us-east-1b` (auto-selected from the default VPC)
- `PrivateIpAddress` - `172.31.22.163`
- `State.Name` - Initially `pending`, transitioning to `running`
- `VpcId` - `vpc-01c56808b78d3cf00`
- `SubnetId` - `subnet-00e2a4a9eb1143975`

> **Best Practice:** Capture the `InstanceId` from the command output immediately. It is required for EIP association and subsequent validation queries. In automation pipelines, parse and store this value using `--query` and `--output text` flags.

**Screenshot: `aws ec2 run-instances` command and initial JSON response showing reservation and network interface details**

![EC2 run-instances output - part 1](https://github.com/user-attachments/assets/6c93ab9a-9705-45db-9646-dbe35c38f702)

**Screenshot: Continued JSON output showing network interface configuration, private IP, and subnet binding**

![EC2 run-instances output - part 2](https://github.com/user-attachments/assets/8d6014df-ffca-45bf-afc5-f42ef7376f4c)

**Screenshot: Further output confirming VPC ID, security group assignment, and instance tag `datacenter-ec2`**

![EC2 run-instances output - part 3](https://github.com/user-attachments/assets/a67d9cbc-3908-463c-9dc0-b434e8c9009d)

**Screenshot: Final section of the launch response confirming instance ID `i-0f6a39c883a6e3cb1`, instance type `t2.micro`, and initial state `pending`**

![EC2 run-instances output - part 4](https://github.com/user-attachments/assets/0b41dcbe-c63b-4fce-b906-e665566acc8b)

---

## Step 3: Allocate Elastic IP

**Intent:** Reserve a static public IPv4 address from Amazon's address pool in VPC scope. This address persists independently of the instance lifecycle and will not change on stop/start cycles once associated.

```bash
aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=datacenter-eip}]' \
  --region us-east-1
```

**Key parameters:**

- `--domain vpc` - Allocates the EIP in VPC scope, which is required for association with instances running in a VPC. Classic scope is deprecated.
- `--tag-specifications` - Tags the EIP at allocation time, enabling identification and preventing orphaned IP charges

**Key values returned:**

- `AllocationId` - `eipalloc-0cb4e844e5a931a3e` (required for association and release operations)
- `PublicIp` - `98.85.1.0` (the static IP that will be permanently assigned)
- `Domain` - `vpc`
- `NetworkBorderGroup` - `us-east-1`

> **Cost Warning:** Elastic IPs incur charges when allocated but **not** associated with a running instance. Always release EIPs that are no longer in use to avoid unnecessary costs. AWS charges approximately $0.005/hour for unassociated EIPs.

> **Best Practice:** Tag EIPs with the owning team, project, and environment at allocation time. Untagged EIPs in large accounts become difficult to audit.

**Screenshot: `aws ec2 allocate-address` output confirming `AllocationId`, public IP `98.85.1.0`, and VPC domain scope**

![Elastic IP allocation output](https://github.com/user-attachments/assets/a98b9a32-1f6d-4305-9180-e9aa93108cc4)

---

## Step 4: Associate Elastic IP with EC2

**Intent:** Bind the allocated Elastic IP to the running EC2 instance. This operation replaces any previously assigned dynamic public IP with the static EIP, making the public endpoint persistent across instance stop/start cycles.

```bash
aws ec2 associate-address \
  --allocation-id eipalloc-0cb4e844e5a931a3e \
  --instance-id i-0f6a39c883a6e3cb1 \
  --region us-east-1
```

**Key parameters:**

- `--allocation-id` - References the EIP allocated in Step 3
- `--instance-id` - Targets the EC2 instance launched in Step 2

**Expected response:**

- `AssociationId` - `eipassoc-03eb755344e9d7826` - Confirms successful binding between the EIP and the instance

> **Operational Note:** The association is immediate. The instance's public IP address changes from the ephemeral IP assigned at launch to the static EIP (`98.85.1.0`) as soon as this command completes successfully.

> **Edge Case:** If the instance was previously assigned a dynamic public IP (from auto-assign public IP settings on the subnet), that IP is released back to the pool upon EIP association. It cannot be recovered.

**Screenshot: `aws ec2 associate-address` command and response confirming `AssociationId` `eipassoc-03eb755344e9d7826`**

![Elastic IP association success](https://github.com/user-attachments/assets/eea56650-352f-4efd-8899-0aa8722a5733)

---

## Step 5: Validate Elastic IP Association

**Intent:** Independently verify, using two separate describe commands, that the Elastic IP is correctly bound to the target EC2 instance and that the instance is in a healthy running state with the expected public IP.

### 5a. Verify Elastic IP Address Record

```bash
aws ec2 describe-addresses \
  --allocation-ids eipalloc-0cb4e844e5a931a3e \
  --region us-east-1
```

**Expected fields in response:**

- `AllocationId` - Matches the EIP allocated in Step 3
- `AssociationId` - Confirms the EIP is actively associated
- `InstanceId` - Confirms association to the correct instance `i-0f6a39c883a6e3cb1`
- `PublicIp` - `98.85.1.0`
- `PrivateIpAddress` - `172.31.22.163` (the instance's private IP within the VPC)
- `Tags` - Confirms `Name=datacenter-eip` is present

### 5b. Verify EC2 Instance Public IP

```bash
aws ec2 describe-instances \
  --instance-ids i-0f6a39c883a6e3cb1 \
  --query "Reservations[*].Instances[*].{ID:InstanceId,State:State.Name,PublicIP:PublicIpAddress,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table \
  --region us-east-1
```

**Expected table output confirms:**

- `ID` - `i-0f6a39c883a6e3cb1`
- `Name` - `datacenter-ec2`
- `PublicIP` - `98.85.1.0` (matches the allocated EIP)
- `State` - `running`

> **Validation Rule:** The `PublicIP` value in the instance describe output **must match** the `PublicIp` value in the EIP describe output. A mismatch indicates the association did not complete successfully.

> **Best Practice:** Incorporate both validation commands into post-deployment smoke tests or CI/CD pipeline health checks. These queries are lightweight, read-only, and safe to run repeatedly.

**Screenshot: `aws ec2 describe-addresses` output confirming EIP `98.85.1.0` is associated with instance `i-0f6a39c883a6e3cb1` and tagged `datacenter-eip`**

![describe-addresses output confirming EIP association details](https://github.com/user-attachments/assets/b2422731-d84c-4e35-bffd-ac8b1ef6ef23)

**Screenshot: `aws ec2 describe-instances` formatted table output confirming instance `datacenter-ec2` is in `running` state with public IP `98.85.1.0`**

![describe-instances table output confirming running state and EIP](https://github.com/user-attachments/assets/633344bf-73d4-413f-843d-6c5978f03798)

---

## Final Outcome

All provisioning, networking, and validation steps completed successfully. The deployed environment reflects the following confirmed state:

- **EC2 instance** `i-0f6a39c883a6e3cb1` is in `running` state in `us-east-1b`
- **Elastic IP** `98.85.1.0` is allocated in VPC scope and permanently associated with the instance
- **Public endpoint** `98.85.1.0` is static and will persist across instance stop/start cycles
- **Both resources** are tagged with descriptive names (`datacenter-ec2`, `datacenter-eip`) for governance and cost attribution
- **Validation** confirmed via independent `describe-addresses` and `describe-instances` queries with matching public IP values

---

## Operational Considerations and Best Practices

- **Tag everything at creation time.** Applying tags via `--tag-specifications` at launch and allocation, rather than as a secondary operation, ensures resources are identifiable even if subsequent tagging steps fail or are skipped.
- **Explicitly scope all commands to a region.** Using `--region us-east-1` in every command prevents accidental provisioning in the default region configured in the CLI profile, which may differ from the intended target.
- **Capture and store resource IDs immediately.** The `InstanceId` and `AllocationId` returned at creation are critical for all subsequent operations. In automated pipelines, parse these with `--query` and `--output text` and store them as environment variables or pipeline artifacts.
- **Release unused Elastic IPs.** EIPs not associated with a running instance incur hourly charges. Build EIP release steps into teardown runbooks and infrastructure cleanup scripts.
- **Use the `--allow-reassociation` flag** if re-associating an EIP that is already bound to another instance or network interface. Without it, the command will fail if the EIP is currently in use.
- **Validate with read-only describe commands** after every state-changing operation. This pattern catches partial failures and confirms the intended end state before proceeding to the next step.
- **Prefer IMDSv2** for instance metadata access in production environments. This can be enforced at launch using `--metadata-options HttpTokens=required` in the `run-instances` command.

---

## Risks, Edge Cases, and Troubleshooting

| Scenario | Risk / Symptom | Resolution |
|---|---|---|
| Wrong account or region active | Resources provisioned in unintended account | Always run `aws sts get-caller-identity` before any provisioning command |
| EIP limit reached | `AddressLimitExceeded` error on `allocate-address` | Request a limit increase via AWS Support or release unused EIPs |
| Instance not yet running when associating EIP | Association may succeed but connectivity may not be immediate | Wait for instance status checks to pass before testing connectivity |
| EIP associated but PublicIP mismatch in describe | Stale describe cache or eventual consistency lag | Re-run describe commands after a brief delay |
| Orphaned EIP after instance termination | Ongoing hourly charges with no associated resource | Implement automated EIP audit using `describe-addresses` and alert on unassociated EIPs |
| No key pair specified at launch | Instance is inaccessible via SSH | Add `--key-name <key-pair-name>` to the `run-instances` command for SSH access requirements |
| Default security group blocks inbound traffic | SSH or application traffic fails | Review and update the default security group inbound rules as needed |

---

## Tools and Services Used

- **AWS CLI v2** - Primary provisioning and validation interface
- **Amazon EC2** - Compute layer (t2.micro, Amazon Linux AMI)
- **Amazon Elastic IP (VPC)** - Static public IPv4 addressing
- **AWS IAM** - User-based authentication and permission enforcement
- **AWS STS** - Identity verification via `get-caller-identity`






























# AWS EC2 Instance with Elastic IP (CLI Provisioning)

## Project Overview
- This project demonstrates how to provision an Amazon EC2 instance using the AWS CLI and associate a static Elastic IP (EIP) to ensure a consistent public endpoint.  
- The setup is performed entirely via CLI commands in the **us-east-1** region, following best practices for reproducibility and validation.

- This lab reflects real-world DevOps workflows including identity verification, resource provisioning, networking configuration, and post-deployment validation.

---

## Architecture Summary

- **Compute**: Amazon EC2 (t2.micro)
- **Networking**: Elastic IP (VPC scope)
- **Region**: us-east-1
- **Provisioning Method**: AWS CLI
- **AMI**: Amazon Linux–based AMI

---

## Step 1: Verify AWS Identity
- Before provisioning resources, confirm the active AWS identity and account context.

`aws sts get-caller-identity`

Expected output confirms:

- IAM User

- AWS Account ID

- Correct permissions context

📸 Screenshot: `aws sts get-caller-identity output`
<img width="1035" height="416" alt="image" src="https://github.com/user-attachments/assets/262ad477-209a-44b9-80ad-e53a4f02e154" />

## Step 2: Launch EC2 Instance

- An EC2 instance is launched using the AWS CLI with tagging applied at creation time.

`aws ec2 run-instances \
  --image-id ami-0c7217cdde317cfec \
  --count 1 \
  --instance-type t2.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datacenter-ec2}]' \
  --region us-east-1`

Key details:

- Instance Type: `t2.micro`

- Tag: `Name=datacenter-ec2`

- Availability Zone: `auto-selected`

📸 Screenshots: `[EC2 run-instances command]`
<img width="952" height="860" alt="image" src="https://github.com/user-attachments/assets/6c93ab9a-9705-45db-9646-dbe35c38f702" />
<img width="1014" height="864" alt="image" src="https://github.com/user-attachments/assets/8d6014df-ffca-45bf-afc5-f42ef7376f4c" />
<img width="956" height="859" alt="image" src="https://github.com/user-attachments/assets/a67d9cbc-3908-463c-9dc0-b434e8c9009d" />
<img width="1030" height="858" alt="image" src="https://github.com/user-attachments/assets/0b41dcbe-c63b-4fce-b906-e665566acc8b" />

## Step 3: Allocate Elastic IP

- A VPC-scoped Elastic IP is allocated and tagged for identification.

`aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=datacenter-eip}]' \
  --region us-east-1`

This returns:

- `AllocationId`

- `Public IP address`

📸 Screenshot: `Elastic IP allocation output`
<img width="1023" height="864" alt="image" src="https://github.com/user-attachments/assets/a98b9a32-1f6d-4305-9180-e9aa93108cc4" />

## Step 4: Associate Elastic IP with EC2

- The allocated Elastic IP is associated with the EC2 instance.

`aws ec2 associate-address \
  --allocation-id eipalloc-0cb4e844e5a931a3e \
  --instance-id i-0f6a39c883a6e3cb1 \
  --region us-east-1`

- Successful execution returns an AssociationId.

📸 Screenshot: `Elastic IP association success`
<img width="1024" height="858" alt="image" src="https://github.com/user-attachments/assets/eea56650-352f-4efd-8899-0aa8722a5733" />

##Step 5: Validate Elastic IP Association

- Confirm that the Elastic IP is correctly attached to the instance.

- Verify Elastic IP
`aws ec2 describe-addresses \
  --allocation-ids eipalloc-0cb4e844e5a931a3e \
  --region us-east-1`

- Verify EC2 Public IP
`aws ec2 describe-instances \
  --instance-ids i-0f6a39c883a6e3cb1 \
  --query "Reservations[*].Instances[*].{ID:InstanceId,State:State.Name,PublicIP:PublicIpAddress,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table \
  --region us-east-1`

Expected result:

- Instance state: `running`

- Public IP matches Elastic IP

📸 Screenshots: `[describe-addresses output]` 
<img width="1027" height="855" alt="image" src="https://github.com/user-attachments/assets/b2422731-d84c-4e35-bffd-ac8b1ef6ef23" />

`[describe-instances table output]`
<img width="1032" height="805" alt="image" src="https://github.com/user-attachments/assets/633344bf-73d4-413f-843d-6c5978f03798" />

## Final Outcome

- EC2 instance successfully provisioned via CLI

- Static Elastic IP allocated and associated

- Public endpoint validated and persistent

- Resources correctly tagged and region-scoped

## Tools & Services Used

- AWS CLI

- Amazon EC2

- Elastic IP (VPC)

- IAM (User-based authentication)










