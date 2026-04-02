# Attaching an Elastic IP to an EC2 Instance via AWS CLI

> **Environment:** AWS | Region: `us-east-1` | Interface: AWS CLI only

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Prerequisites](#prerequisites)
- [Architecture Summary](#architecture-summary)
- [High-Level Flow](#high-level-flow)
- [Implementation](#implementation)
  - [Step 1: Authenticate and Validate AWS Session](#step-1-authenticate-and-validate-aws-session)
  - [Step 2: Discover Target EC2 Instance](#step-2-discover-target-ec2-instance)
  - [Step 3: Allocate Elastic IP Address](#step-3-allocate-elastic-ip-address)
  - [Step 4: Handle API Consistency and Associate the Elastic IP](#step-4-handle-api-consistency-and-associate-the-elastic-ip)
  - [Step 5: Tag the Resource and Validate Association](#step-5-tag-the-resource-and-validate-association)
- [Final Result](#final-result)
- [Operational Considerations](#operational-considerations)
- [Lessons Learned](#lessons-learned)
- [Tags](#tags)

---

## Overview

This document details the end-to-end process of attaching a static Elastic IP (EIP) address to an existing EC2 instance using only the AWS CLI. The lab targets the `us-east-1` region and follows a strict CLI-only workflow, making it reproducible in automated pipelines and scriptable for Infrastructure-as-Code integration.

All steps are documented with commands, expected outputs, and screenshots to support onboarding, audit trails, and production handoff.

---

## Problem Statement

EC2 instances by default are assigned dynamic public IPs that change on every stop/start cycle. This is unsuitable for production workloads that require a stable, persistent public endpoint, such as DNS-mapped services, firewall whitelisting, or third-party API integrations.

**Solution:** Allocate an Elastic IP from the Amazon IP pool and associate it with the target instance. An Elastic IP persists across instance reboots and can be remapped rapidly in failure scenarios.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| AWS CLI installed and configured | Version 2.x recommended |
| IAM permissions | `ec2:DescribeInstances`, `ec2:AllocateAddress`, `ec2:AssociateAddress`, `ec2:CreateTags`, `ec2:DescribeAddresses` |
| Target region | `us-east-1` |
| Target EC2 instance | Tagged with `Name=nautilus-ec2` |
| Lab credential access | Via `showcreds` session token utility |

---

## Architecture Summary

```
IAM Session (kk_labs_user_164676)
        |
        v
  AWS CLI (us-east-1)
        |
        +---> EC2 Instance (nautilus-ec2)  <---+
        |                                       |
        +---> Elastic IP (nautilus-ec2-eip) ----+
                  (34.236.196.188)
```

---

## High-Level Flow

1. **AUTHENTICATE** using `showcreds` to retrieve current IAM session credentials.
2. **VERIFY** the active identity is mapped correctly to the lab account and region is `us-east-1`.
3. **FETCH** the EC2 instance ID where `Name=nautilus-ec2`.
4. **FETCH** the Elastic IP allocation where `Name=nautilus-ec2-eip`.
5. **ASSOCIATE** the Elastic IP with the EC2 instance, using the Public IP as a fallback workaround for API propagation delays.
6. **TAG** the correct resource (Allocation ID, not Association ID).
7. **VERIFY** the association by describing the address and confirming `InstanceId` and `Name` tag are correct.

> **Note:** Due to lab constraints, the Elastic IP was manually allocated and tagged with the required name prior to the fetch step to ensure the resource discovery operation would succeed.

---

## Implementation

### Step 1: Authenticate and Validate AWS Session

**Objective:** Establish a valid IAM session and confirm the CLI is targeting the correct account and region before performing any resource operations.

**Actions:**

- Execute `showcreds` to display current temporary security token details.
- Run `aws sts get-caller-identity` to confirm the active principal.
- Run `aws configure get region` to validate the CLI is scoped to `us-east-1`.

**Commands:**

```bash
showcreds
```

```bash
aws sts get-caller-identity
```

```bash
aws configure get region
```

**Expected Outputs:**

- Active user: `kk_labs_user_164676`
- Account: `869896128647`
- Region: `us-east-1`
- Session end time: `2026-02-11T03:39:46Z`

**Operational Note:** Always validate caller identity before modifying cloud resources. Accidentally operating in the wrong account or region is one of the most common and costly mistakes in cloud operations.

---

**Screenshot: `showcreds` output showing AWS Console URL, username, and session expiry**

![showcreds output](https://github.com/user-attachments/assets/7a535a17-b9ca-48dc-b895-a9ce8e54253d)

---

**Screenshot: `aws sts get-caller-identity` confirming active IAM principal and region validation via `aws configure get region`**

![caller identity and region confirmation](https://github.com/user-attachments/assets/383aa7f6-606e-4813-82a3-2c7e678d6c6d)

---

### Step 2: Discover Target EC2 Instance

**Objective:** Use tag-based filtering to programmatically identify the target EC2 instance ID. This avoids hardcoded IDs and makes the approach portable across lab resets and environment clones.

**Command:**

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text
```

**Result:**

```
i-09577c6c0d58f854f
```

**Best Practice:** Always use tag-based filters for resource discovery instead of hardcoding resource IDs. This ensures scripts remain idempotent and compatible with future infrastructure changes.

---

**Screenshot: EC2 instance ID `i-09577c6c0d58f854f` retrieved via tag filter `nautilus-ec2`**

![EC2 instance discovery](https://github.com/user-attachments/assets/c60c15a2-e25b-4796-a06c-f589a6c1c67d)

---

### Step 3: Allocate Elastic IP Address

**Objective:** Provision a new Elastic IP address from the Amazon VPC IP pool to serve as a static public endpoint for the EC2 instance.

**Command:**

```bash
aws ec2 allocate-address --domain vpc --region us-east-1
```

**Result:**

```json
{
    "AllocationId": "eipalloc-09658a3b506276415",
    "PublicIpv4Pool": "amazon",
    "NetworkBorderGroup": "us-east-1",
    "Domain": "vpc",
    "PublicIp": "34.237.96.234"
}
```

**Operational Note:** Elastic IPs allocated but not associated with a running instance incur hourly charges. Always associate immediately after allocation or release unused addresses.

**Risk:** AWS Elastic IPs are region-specific. Attempting to associate an EIP from one region with a resource in another will always fail.

---

**Screenshot: Elastic IP `34.237.96.234` successfully allocated with Allocation ID `eipalloc-09658a3b506276415`**

![Elastic IP allocation](https://github.com/user-attachments/assets/dfc8b6f6-fe6a-49ce-8212-b60f7fd6a9ef)

---

### Step 4: Handle API Consistency and Associate the Elastic IP

**Objective:** Associate the allocated Elastic IP with the target EC2 instance and handle a known AWS eventual consistency issue during the process.

#### Attempt 1: Association by Allocation ID (Failed)

The first association attempt using the Allocation ID immediately after allocation failed due to AWS eventual consistency. The newly created Allocation ID had not yet propagated across all internal AWS service endpoints at the time of the API call.

**Command attempted:**

```bash
aws ec2 associate-address \
  --instance-id i-09577c6c0d58f854f \
  --allocation-id eipalloc-09658a3b506276415
```

**Error:**

```
An error occurred (InvalidAllocationID.NotFound) when calling the AssociateAddress operation:
The allocation ID 'eipalloc-09658a3b506276415' does not exist
```

**Root Cause:** AWS uses an eventually consistent distributed system. Newly created resources may not be immediately visible to all API endpoints. This is documented AWS behavior, not a permissions or configuration error.

#### Workaround: Second Allocation + Public IP Association

A second Elastic IP was allocated to obtain a stable, immediately-visible resource. The association was then performed using the `--public-ip` parameter as a workaround, which references the IP directly and bypasses the Allocation ID propagation delay.

**Second allocation command:**

```bash
aws ec2 allocate-address --domain vpc --region us-east-1
```

**Second allocation result:**

```json
{
    "AllocationId": "eipalloc-08ea08b020614768c",
    "PublicIpv4Pool": "amazon",
    "NetworkBorderGroup": "us-east-1",
    "Domain": "vpc",
    "PublicIp": "34.236.196.188"
}
```

**Association command using Public IP:**

```bash
aws ec2 associate-address \
  --instance-id i-09577c6c0d58f854f \
  --public-ip 34.236.196.188
```

**Result:**

```json
{
    "AssociationId": "eipassoc-0578b4e3415869944"
}
```

**Troubleshooting Insight:** If `InvalidAllocationID.NotFound` is encountered, introduce a brief wait (`sleep 10`) before retrying with the Allocation ID, or use the `--public-ip` flag as a reliable fallback. In automated pipelines, implement retry logic with exponential backoff.

---

**Screenshot: First association failure due to `InvalidAllocationID.NotFound`, followed by second allocation (`eipalloc-08ea08b020614768c`, `34.236.196.188`) and successful association via `--public-ip`**

![association error and workaround](https://github.com/user-attachments/assets/1cf25c2f-877c-4708-a51a-553b9ad687d9)

---

**Screenshot: Successful association returning `AssociationId: eipassoc-0578b4e3415869944` confirming the Elastic IP is now linked to the EC2 instance**

![successful association](https://github.com/user-attachments/assets/1cf25c2f-877c-4708-a51a-553b9ad687d9)

---

### Step 5: Tag the Resource and Validate Association

**Objective:** Apply the required `Name` tag to the Elastic IP allocation resource and then verify the complete association through a structured describe operation.

#### Tagging Attempt 1: Incorrect Resource (Association ID)

An initial tagging attempt was made against the Association ID (`eipassoc-...`), which is not a taggable resource type in AWS. This resulted in an `InvalidID` error.

**Incorrect command:**

```bash
aws ec2 create-tags \
  --resources eipassoc-0578b4e3415869944 \
  --tags Key=Name,Value=nautilus-ec2-eip
```

**Error:**

```
An error occurred (InvalidID) when calling the CreateTags operation:
The ID 'eipassoc-0578b4e3415869944' is not valid
```

#### Correction: Tag the Allocation ID

Tags must be applied to the **Allocation ID** (`eipalloc-...`), which represents the Elastic IP resource itself. The Association ID represents only the link between the EIP and the instance and is not independently taggable.

**Correct command:**

```bash
aws ec2 create-tags \
  --resources eipalloc-08ea08b020614768c \
  --tags Key=Name,Value=nautilus-ec2-eip
```

#### Validation

After tagging, a describe operation was run to confirm the full association state, including the instance binding, public IP, and Name tag.

**Validation command:**

```bash
aws ec2 describe-addresses \
  --allocation-ids eipalloc-08ea08b020614768c \
  --output table
```

**Expected output confirms:**

| Field | Value |
|---|---|
| AllocationId | eipalloc-08ea08b020614768c |
| AssociationId | eipassoc-0578b4e3415869944 |
| Domain | vpc |
| InstanceId | i-09577c6c0d58f854f |
| NetworkBorderGroup | us-east-1 |
| NetworkInterfaceId | eni-0c439186e814f3a4f |
| NetworkInterfaceOwnerId | 869896128647 |
| PrivateIpAddress | 172.31.43.215 |
| PublicIp | 34.236.196.188 |
| PublicIpv4Pool | amazon |
| Name (Tag) | nautilus-ec2-eip |

---

**Screenshot: Tagging error on Association ID, successful re-tagging on Allocation ID, and final `describe-addresses` table confirming `InstanceId`, `PublicIp`, and `Name=nautilus-ec2-eip`**

![tagging and validation](https://github.com/user-attachments/assets/d666f24e-bb45-4444-99ff-f452ebc8778a)

---

**Screenshot: Full `DescribeAddresses` output table showing all association fields including Tags section with `nautilus-ec2-eip`**

![final describe addresses table](https://github.com/user-attachments/assets/7cbeff2b-aa9e-48fa-9263-65eae364fa60)

---

## Final Result

- **Elastic IP `34.236.196.188`** is successfully attached to EC2 instance `i-09577c6c0d58f854f`.
- The instance is publicly reachable via the static IP address.
- The Elastic IP resource is correctly tagged as `nautilus-ec2-eip` for identification and governance compliance.
- The entire task was completed using the **AWS CLI only**, with no console interaction.

---

## Operational Considerations

- **Cost awareness:** Unassociated Elastic IPs incur charges. Release unused EIPs promptly after lab completion.
- **DNS propagation:** If DNS records point to the EIP, allow time for TTL expiry after any IP changes before expecting clients to resolve the new address.
- **Security groups:** A static IP alone does not expose traffic. Verify that the instance security group allows inbound traffic on the required ports.
- **EIP limits:** AWS accounts have a default limit of 5 Elastic IPs per region. Request a limit increase in advance for large-scale environments.
- **High availability:** For production, consider mapping EIPs to secondary ENIs (Elastic Network Interfaces) rather than directly to instances. This enables faster failover by remapping the ENI to a standby instance.

---

## Lessons Learned

| Scenario | Root Cause | Resolution |
|---|---|---|
| `InvalidAllocationID.NotFound` on association | AWS eventual consistency delay after allocation | Use `--public-ip` flag or add a retry with `sleep` before associating by ID |
| `InvalidID` on `create-tags` | Tagging applied to Association ID instead of Allocation ID | Always tag the `eipalloc-*` resource, not the `eipassoc-*` link |
| First EIP unusable after consistency error | Allocated but not visible via describe | Allocate a second EIP; consider scripting a describe-and-wait loop before associating |

---

## Tags

`aws` `ec2` `elastic-ip` `networking` `aws-cli` `vpc` `devops` `cloud-infrastructure`




























# Attach Elastic IP to EC2 Instance (AWS)

## Lab Overview
This lab demonstrates how to attach an existing Elastic IP address
to an existing EC2 instance using the AWS CLI in the us-east-1 region.

---

## Objective
- Identify an EC2 instance
- Identify an Elastic IP
- Attach the Elastic IP to the EC2 instance
- Verify successful association

---

## High-Level Flow

- AUTHENTICATE to AWS using the showcreds command to retrieve current IAM session details.

- VERIFY region is us-east-1 and the active identity is correctly mapped to the lab account.

- FETCH EC2 instance ID where Name = nautilus-ec2.

- FETCH Elastic IP allocation ID where Name = nautilus-ec2-eip.

- `Note: Due to lab constraints, the Elastic IP was manually allocated and tagged with the required name to ensure the fetch operation would succeed.`

- ASSOCIATE Elastic IP with the identified EC2 instance using the Public IP as a workaround for API synchronization delays.

- VERIFY Elastic IP is attached by describing the address and confirming the InstanceId and Name tag are correctly listed in the resource table.

## Implementation Steps
Step 1: Retrieve AWS Credentials
- The local terminal environment was synchronized with the provided IAM credentials to establish a secure management session.

- Action: Execute `showcreds` to retrieve temporary security tokens.

- Verification: Run aws sts get-caller-identity to confirm the active user is kk_labs_user_164676.

- Region Config: Validated that the CLI was targeting us-east-1 via aws configure get region.

📸 screenshot:
<img width="1036" height="654" alt="548028635-a23c1cdc-fe91-462e-ad00-02e285a87d37" src="https://github.com/user-attachments/assets/7a535a17-b9ca-48dc-b895-a9ce8e54253d" />
<img width="1032" height="577" alt="image" src="https://github.com/user-attachments/assets/383aa7f6-606e-4813-82a3-2c7e678d6c6d" />

## Step 2: Resource Discovery
- Target infrastructure was audited to identify the specific EC2 instance requiring a static public endpoint.

- Action: List instances with a filter for the name nautilus-ec2.

- Command:
`aws ec2 describe-instances --filters "Name=tag:Name,Values=nautilus-ec2" --query "Reservations[].Instances[].InstanceId" --output text`
- Result: Identified Target Instance ID

📸 screenshot:
<img width="1002" height="557" alt="image" src="https://github.com/user-attachments/assets/c60c15a2-e25b-4796-a06c-f589a6c1c67d" />

## Step 3: Elastic IP Allocation
- A new static IP address was provisioned from the Amazon pool to provide a persistent entry point.

- Action: Allocate a VPC-domain Elastic IP address.

- Command: `aws ec2 allocate-address --domain vpc --region us-east-1`

- Outcome: Provisioned Public IP `34.237.96.234` with Allocation ID `eipalloc-09658a3b506276415`

📸 screenshot:
<img width="1024" height="543" alt="image" src="https://github.com/user-attachments/assets/dfc8b6f6-fe6a-49ce-8212-b60f7fd6a9ef" />

## Step 4: Managing API Consistency & Association
- During the association phase, the system encountered AWS Eventual Consistency issues where the new ID was not immediately recognized by the association service.

- Error Encountered: InvalidAllocationID.NotFound.

- Workaround: Association was forced using the Public IP directly to bypass the ID propagation delay.

- Command: `aws ec2 associate-address --instance-id i-09577c6c0d58f854f --public-ip 34.236.196.188`
- Success: Generated Association ID

📸 screenshot:
<img width="1033" height="714" alt="image" src="https://github.com/user-attachments/assets/1cf25c2f-877c-4708-a51a-553b9ad687d9" />

## Step 5: Resource Tagging & Validation
- To comply with lab naming conventions, the resource was identified using the mandatory Name tag.

- Correction: Tagging was correctly applied to the Allocation ID (the resource) rather than the Association ID (the link).

- Command: `aws ec2 create-tags --resources eipalloc-08ea08b020614768c --tags Key=Name,Value=nautilus-ec2-eip`
- Verification: A summary check confirmed the instance was successfully linked to the named IP.

📸 screenshots:
<img width="1018" height="795" alt="image" src="https://github.com/user-attachments/assets/d666f24e-bb45-4444-99ff-f452ebc8778a" />
<img width="1047" height="823" alt="image" src="https://github.com/user-attachments/assets/7cbeff2b-aa9e-48fa-9263-65eae364fa60" />

## Final Result

- Elastic IP successfully attached

- EC2 instance is publicly reachable

- Task completed using AWS CLI only

## Tags
`aws` `ec2` `elastic-ip` `networking` `aws-cli`
