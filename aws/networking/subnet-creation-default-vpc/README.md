# AWS VPC Subnet Provisioning via CLI: Network Segmentation for Cloud Workloads

**Author:** DevOps / Cloud Infrastructure Engineer
**Cloud Provider:** AWS
**Region:** us-east-1
**Tooling:** AWS CLI, Linux Shell, IAM

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [Environment Details](#environment-details)
- [Step 1: Identify the Default VPC](#step-1-identify-the-default-vpc)
- [Step 2: Inspect Existing Subnet CIDRs](#step-2-inspect-existing-subnet-cidrs)
- [Step 3: Create the New Subnet](#step-3-create-the-new-subnet)
- [Step 4: Tag the Subnet](#step-4-tag-the-subnet)
- [Step 5: Verify Subnet Provisioning](#step-5-verify-subnet-provisioning)
- [Results Summary](#results-summary)
- [Operational Considerations](#operational-considerations)
- [Troubleshooting Reference](#troubleshooting-reference)
- [Real-World Relevance](#real-world-relevance)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This document covers the end-to-end process of provisioning a new subnet inside an existing AWS default VPC using the AWS CLI. Every action is executed programmatically, with no reliance on the AWS Management Console, which ensures repeatability and integration into automation pipelines.

Network segmentation is a foundational step before deploying any compute, database, or container workload in AWS. Provisioning subnets with correct CIDR planning and tagging prevents address conflicts, enables workload isolation, and supports scalable infrastructure growth.

---

## Problem Statement

During incremental cloud migrations, engineering teams frequently encounter the need to extend network segmentation before deploying additional workloads. A default VPC, while convenient, is often under-segmented for production use. Without an additional, purpose-built subnet, teams cannot enforce isolation, apply targeted routing policies, or maintain least-privilege network access between workloads.

**The objective is to:**

- Identify the existing default VPC in us-east-1
- Audit all existing subnet CIDR allocations to prevent overlap
- Provision a new, non-overlapping subnet in a specified Availability Zone
- Apply standardized resource tags for identification and lifecycle management
- Validate the complete provisioning outcome using structured CLI queries

---

## Architecture Context

```
Default VPC: 172.31.0.0/16
│
├── 172.31.64.0/20   (existing)
├── 172.31.16.0/20   (existing)
├── 172.31.48.0/20   (existing)
├── 172.31.80.0/20   (existing)
├── 172.31.32.0/20   (existing)
├── 172.31.0.0/20    (existing)
│
└── 172.31.96.0/20   <-- NEW: devops-subnet | AZ: us-east-1a
```

All existing subnets use /20 blocks. The new subnet follows the same prefix length to maintain consistency across the VPC's addressing scheme.

---

## Prerequisites

Before executing any of the steps below, ensure the following conditions are met:

- AWS CLI v2 is installed and configured (`aws configure`)
- IAM credentials are active with the following minimum permissions:
  - `ec2:DescribeVpcs`
  - `ec2:DescribeSubnets`
  - `ec2:CreateSubnet`
  - `ec2:CreateTags`
- The target region is set to `us-east-1` in your CLI profile or via `AWS_DEFAULT_REGION`
- You have confirmed there are no organizational SCPs blocking subnet creation in the target account

---

## Environment Details

| Parameter | Value |
|---|---|
| Cloud Provider | AWS |
| Region | us-east-1 |
| VPC Type | Default VPC |
| VPC CIDR | 172.31.0.0/16 |
| New Subnet CIDR | 172.31.96.0/20 |
| Availability Zone | us-east-1a |
| Subnet Name Tag | devops-subnet |

---

## Step 1: Identify the Default VPC

**Intent:** Retrieve all VPCs in the region and isolate the one flagged as the default. This provides the `VpcId` required for all subsequent operations.

```bash
aws ec2 describe-vpcs \
  --query "Vpcs[*].{VpcId:VpcId,Cidr:CidrBlock,IsDefault:IsDefault}" \
  --output table
```

**Expected output fields:**

- `Cidr` - the VPC's address space
- `IsDefault` - `True` confirms this is the default VPC
- `VpcId` - the identifier to be used in downstream commands

**Result from execution:**

| Cidr | IsDefault | VpcId |
|---|---|---|
| 172.31.0.0/16 | True | vpc-04b343a0138113c8a |

**Screenshot: Default VPC identification**

![Default VPC identified via AWS CLI describe-vpcs with CIDR, IsDefault, and VpcId fields displayed in table format](https://github.com/user-attachments/assets/68cb9cd9-87cd-40cd-9900-a5645d558212)

> **Operational Note:** In multi-VPC environments, always explicitly filter for `IsDefault=true` or store the VpcId as a shell variable immediately to prevent accidental operations against the wrong network boundary.

---

## Step 2: Inspect Existing Subnet CIDRs

**Intent:** Enumerate all subnets currently allocated within the default VPC. This audit is a mandatory prerequisite to CIDR selection for the new subnet. Overlapping CIDR blocks will cause the `create-subnet` call to fail with an `InvalidSubnet.Conflict` error.

```bash
aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=vpc-04b343a0138113c8a \
  --query "Subnets[*].CidrBlock" \
  --output table
```

**Existing subnet allocations identified:**

- 172.31.64.0/20
- 172.31.16.0/20
- 172.31.48.0/20
- 172.31.80.0/20
- 172.31.32.0/20
- 172.31.0.0/20

**Screenshot: Existing subnet CIDR inventory**

![Output of describe-subnets showing all six existing /20 subnet CIDRs within the default VPC](https://github.com/user-attachments/assets/b0e78cff-d0a8-4919-b356-7463cdeb7f4d)

> **CIDR Planning Note:** The next available /20 block within 172.31.0.0/16 after the existing allocations is `172.31.96.0/20`. A /20 subnet provides 4,096 addresses (4,091 usable after AWS reserves 5), which is appropriate for workload groupings at this tier.

> **Edge Case:** Always validate against both allocated and planned CIDRs if multiple engineers are operating concurrently. Use infrastructure-as-code state files or a CMDB to maintain a single source of truth for address space allocation.

---

## Step 3: Create the New Subnet

**Intent:** Provision the new subnet using the confirmed non-overlapping CIDR block `172.31.96.0/20` within the `us-east-1a` Availability Zone. Specifying the Availability Zone is required to control placement for workloads with AZ affinity requirements, such as EC2 instances paired with EBS volumes or AZ-scoped RDS deployments.

```bash
aws ec2 create-subnet \
  --vpc-id vpc-04b343a0138113c8a \
  --cidr-block 172.31.96.0/20 \
  --availability-zone us-east-1a
```

**Key fields in the creation response:**

- `SubnetId` - `subnet-0c7cf0c38dc347f1a` - the unique identifier for subsequent operations
- `State` - `available` - confirms the subnet is immediately usable
- `AvailableIpAddressCount` - `4091` - confirms the expected address space
- `CidrBlock` - `172.31.96.0/20` - matches the intended allocation
- `AvailabilityZone` - `us-east-1a` - confirms correct AZ placement
- `DefaultForAz` - `false` - this is a custom subnet, not an AZ default
- `MapPublicIpOnLaunch` - `false` - public IP auto-assignment is disabled by default; enable only if this subnet is intended to be public-facing

**Screenshot: Subnet creation API response**

![Full JSON output from create-subnet showing SubnetId, State available, CidrBlock, AvailabilityZone, and AvailableIpAddressCount](https://github.com/user-attachments/assets/046f5460-03f7-404e-bd87-8adb6de5a423)

> **Security Note:** `MapPublicIpOnLaunch` defaults to `false`, which is the correct posture for any subnet that is not explicitly intended to host public-facing resources. For private workloads such as application servers, databases, or internal services, this setting must remain `false`.

> **Risk:** Do not create subnets with overlapping CIDRs in VPCs with VPC peering, Transit Gateway, or VPN connections active. Overlapping CIDR blocks will cause routing conflicts across the entire network fabric.

---

## Step 4: Tag the Subnet

**Intent:** Apply a `Name` tag to the newly created subnet. Tags are essential for cost allocation, access control via tag-based IAM policies, resource discovery, and lifecycle automation. Untagged subnets in large environments are operationally indistinguishable and represent a governance risk.

```bash
aws ec2 create-tags \
  --resources subnet-0c7cf0c38dc347f1a \
  --tags Key=Name,Value=devops-subnet
```

This command produces no output on success, which is the expected behavior for `create-tags`.

**Screenshot: Tag applied to subnet with no error output confirming success**

![Terminal output showing create-tags command executed successfully with no error, confirming the devops-subnet Name tag was applied](https://github.com/user-attachments/assets/fdc3b988-379d-4df3-98f7-70f2684f1db0)

> **Tagging Best Practice:** A minimal tag set for production subnets should include: `Name`, `Environment` (e.g., dev / staging / prod), `Owner`, `Project`, and `CostCenter`. Enforcing this through Tag Policies at the AWS Organizations level prevents tag drift at scale.

---

## Step 5: Verify Subnet Provisioning

**Intent:** Perform a final structured query to confirm the subnet exists, is correctly addressed, is placed in the intended Availability Zone, and is discoverable by its `Name` tag. This step closes the provisioning loop and provides auditable proof of successful deployment.

```bash
aws ec2 describe-subnets \
  --filters Name=tag:Name,Values=devops-subnet \
  --query "Subnets[*].{SubnetId:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone}" \
  --output table
```

**Verification output:**

| AZ | CIDR | SubnetId |
|---|---|---|
| us-east-1a | 172.31.96.0/20 | subnet-0c7cf0c38dc347f1a |

**Screenshot: Final verification confirming subnet AZ, CIDR, and SubnetId**

![Verification table output displaying the devops-subnet in us-east-1a with CIDR 172.31.96.0/20 and confirmed SubnetId](https://github.com/user-attachments/assets/0c970cc3-0f01-488a-93cd-2d0c8c3036c7)

All three key attributes are confirmed:

- **Subnet ID** matches the resource returned at creation
- **CIDR** is correctly set to `172.31.96.0/20`
- **Availability Zone** is confirmed as `us-east-1a`

---

## Results Summary

| Objective | Status |
|---|---|
| Default VPC identified | Confirmed (vpc-04b343a0138113c8a) |
| Existing CIDR conflicts audited | No conflicts found |
| New subnet provisioned | subnet-0c7cf0c38dc347f1a |
| CIDR block assigned | 172.31.96.0/20 |
| Availability Zone placement | us-east-1a |
| Resource tagged | Key=Name, Value=devops-subnet |
| Provisioning verified | All attributes confirmed |

The network layer is provisioned, tagged, and validated. The subnet is ready to receive EC2 instances, RDS deployments, ECS tasks, or any other compute workload that requires this network segment.

---

## Operational Considerations

- **CIDR immutability:** Once a subnet is created, its CIDR block cannot be modified. Plan address space carefully before provisioning.
- **Multi-AZ deployments:** For high availability, provision matching subnets in at least two Availability Zones (e.g., `us-east-1a` and `us-east-1b`). Single-AZ subnets are a single point of failure for AZ-bound resources.
- **Route table association:** A newly created subnet inherits the VPC's main route table by default. For production workloads requiring distinct routing (e.g., private subnets using a NAT Gateway), explicitly associate a custom route table after subnet creation.
- **Network ACL association:** New subnets are automatically associated with the default Network ACL, which allows all inbound and outbound traffic. Apply a restrictive custom NACL for subnets hosting sensitive workloads.
- **IPv6:** If the VPC has an IPv6 CIDR block associated, consider enabling IPv6 on the subnet at creation time using `--ipv6-cidr-block` to avoid a separate modification step.

---

## Troubleshooting Reference

| Error | Likely Cause | Resolution |
|---|---|---|
| `InvalidSubnet.Conflict` | CIDR overlaps with an existing subnet | Re-audit existing CIDRs and select a non-overlapping block |
| `InvalidVpcID.NotFound` | Incorrect VpcId passed | Re-run `describe-vpcs` to retrieve the current VpcId |
| `AuthFailure` | IAM credentials expired or insufficient permissions | Refresh credentials; verify `ec2:CreateSubnet` is allowed |
| `InvalidAvailabilityZone` | AZ name is malformed or not available | Run `aws ec2 describe-availability-zones` to list valid AZs |
| `TagLimitExceeded` | Resource has reached the 50-tag limit | Remove unused tags before applying new ones |

---

## Real-World Relevance

This process directly mirrors network provisioning patterns in production cloud migration projects. Before any compute resource (EC2, RDS, EKS node group, Lambda VPC attachment) can be deployed into a new network segment, the underlying subnet must exist, be correctly addressed, and carry the appropriate metadata tags.

CIDR planning errors at the subnet level compound quickly in environments using VPC Peering, AWS Transit Gateway, or Direct Connect. A single overlapping CIDR block can silently break connectivity across an entire hub-and-spoke network topology. Treating subnet provisioning as a deliberate, auditable engineering task rather than an ad-hoc console operation is a defining characteristic of mature cloud infrastructure practice.

---

## Skills Demonstrated

- AWS VPC architecture and subnet lifecycle management
- CIDR block planning and conflict avoidance
- AWS CLI proficiency with structured `--query` filtering and `--output table` formatting
- Resource tagging strategy and operational governance
- Cloud networking fundamentals: subnets, Availability Zones, route tables, NACLs
- Infrastructure validation and audit-ready documentation practices

























# AWS Networking – Subnet Creation in Default VPC (AWS CLI)

## Overview
- This lab demonstrates how to provision a subnet
-  inside an existing AWS default VPC using AWS CLI.
- Subnet creation is a foundational networking task
- required for workload isolation and scalability.

---

## Scenario
-  As part of an incremental cloud migration, a DevOps team needs to prepare network segmentation within AWS before deploying compute resources.
-  A new subnet is required under the default VPC.

---

## Objectives
- Identify the default VPC using AWS CLI
- Inspect existing CIDR allocations
- Create a non-overlapping subnet
- Apply resource tagging
- Verify successful provisioning

---

## Tools & Services Used
- AWS EC2
- AWS VPC
- AWS CLI
- Linux Shell
- IAM (temporary lab credentials)

---

## Environment Details
- Cloud Provider: AWS
- Region: us-east-1
- VPC Type: Default VPC
- Subnet CIDR: /24
- Availability Zone: us-east-1a

---

## Step 1: Verify Active AWS Region
- Ensure all resources are created in the intended region

aws configure get region

Expected Output:
- us-east-1

📸 Screenshot: Region verification
<img width="1035" height="861" alt="image" src="https://github.com/user-attachments/assets/68cb9cd9-87cd-40cd-9900-a5645d558212" />
---

## Step 2: Identify the Default VPC
- Query all VPCs and filter for the default VPC

aws ec2 describe-vpcs \
  --query "Vpcs[*].{VpcId:VpcId,Cidr:CidrBlock,IsDefault:IsDefault}" \
  --output table

// Store the VPC ID where IsDefault = true

📸 Screenshot: Default VPC details
<img width="1035" height="861" alt="image" src="https://github.com/user-attachments/assets/68cb9cd9-87cd-40cd-9900-a5645d558212" />

---

## Step 3: Inspect Existing Subnets
- Prevent CIDR conflicts by reviewing existing subnets

aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=<DEFAULT_VPC_ID> \
  --query "Subnets[*].CidrBlock" \
  --output table

- Select an unused CIDR block within the VPC range

📸 Screenshot: Existing subnet CIDRs
<img width="1035" height="865" alt="image" src="https://github.com/user-attachments/assets/b0e78cff-d0a8-4919-b356-7463cdeb7f4d" />
---

## Step 4: Create the Subnet
- Provision a new subnet using a valid CIDR block

aws ec2 create-subnet \
  --vpc-id <DEFAULT_VPC_ID> \
  --cidr-block 172.31.96.0/20 \
  --availability-zone us-east-1a

📸 Screenshot: Subnet creation output
<img width="1036" height="772" alt="image" src="https://github.com/user-attachments/assets/046f5460-03f7-404e-bd87-8adb6de5a423" />
---

## Step 5: Tag the Subnet
- Assign a Name tag for identification and lifecycle management

aws ec2 create-tags \
  --resources <SUBNET_ID> \
  --tags Key=Name,Value=devops-subnet

📸 Screenshot: Subnet tagging confirmation
<img width="1038" height="692" alt="image" src="https://github.com/user-attachments/assets/fdc3b988-379d-4df3-98f7-70f2684f1db0" />
---

## Step 6: Final Verification
- Confirm subnet existence, CIDR block, and AZ placement

aws ec2 describe-subnets \
  --filters Name=tag:Name,Values=devops-subnet \
  --query "Subnets[*].{SubnetId:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone}" \
  --output table

📸 Screenshot: Final verification output
<img width="1034" height="861" alt="Screenshot 2026-02-01 045806" src="https://github.com/user-attachments/assets/0c970cc3-0f01-488a-93cd-2d0c8c3036c7" />

---

## Result
- Default VPC successfully identified
- CIDR conflicts avoided through validation
- Subnet created and tagged correctly
- Network layer ready for compute deployment

---

## Real-World Relevance
- This task mirrors real-world cloud migrations
- where networking must be established
- before EC2, RDS, or container workloads.
- Proper CIDR planning prevents production outages.

---

## Skills Demonstrated
- AWS VPC & subnet management
- CIDR planning and validation
- AWS CLI proficiency
- Cloud networking fundamentals

