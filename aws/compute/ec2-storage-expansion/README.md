# [Provisioning Amazon EBS Storage via AWS CLI](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)

> **Domain:** AWS Storage | **Service:** Amazon EC2 / EBS | **Interface:** AWS CLI | **Region:** us-east-1

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Summary](#solution-summary)
- [Architecture Context](#architecture-context)
- [Environment Details](#environment-details)
- [Tools and Services Used](#tools-and-services-used)
- [Implementation](#implementation)
  - [Step 1: Verify AWS Region Configuration](#step-1-verify-aws-region-configuration)
  - [Step 2: Identify Available Availability Zones](#step-2-identify-available-availability-zones)
  - [Step 3: Create the EBS Volume](#step-3-create-the-ebs-volume)
  - [Step 4: Verify Volume State](#step-4-verify-volume-state)
  - [Step 5: Validate Volume Properties and Tags](#step-5-validate-volume-properties-and-tags)
- [Validation Summary](#validation-summary)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices Demonstrated](#best-practices-demonstrated)
- [Lessons Learned](#lessons-learned)
- [Real-World Relevance](#real-world-relevance)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This project documents the end-to-end provisioning of an Amazon Elastic Block Store (EBS) volume using the AWS CLI. The exercise reflects a production-grade storage expansion workflow where block storage is provisioned independently of compute resources to support incremental infrastructure scaling.

The process covers region verification, Availability Zone selection, volume creation with tagging, state confirmation, and multi-attribute validation, following the operational patterns expected in enterprise AWS environments.

---

## Problem Statement

The Nautilus DevOps team is executing a phased migration of on-premises infrastructure to AWS. During the initial phase, persistent block storage must be provisioned ahead of EC2 instance attachment to support a decoupled, scalable architecture. The challenge is to:

- Provision a correctly sized and typed EBS volume in the right Availability Zone
- Apply consistent resource tags for cost attribution and operational traceability
- Validate the volume is in an `available` state before any downstream attachment

---

## Solution Summary

A 2 GiB **gp3** EBS volume named `nautilus-volume` was created programmatically using the AWS CLI in the `us-east-1a` Availability Zone. All properties, tags, and state were confirmed through targeted CLI queries before the volume was marked ready for attachment.

---

## Architecture Context

In a phased cloud migration, storage provisioning is often decoupled from compute provisioning. This allows teams to:

- Pre-allocate storage capacity before EC2 instances are sized or launched
- Maintain separation of concerns between storage, compute, and networking layers
- Reduce blast radius when modifying infrastructure components individually

EBS volumes are Availability Zone-scoped resources. A volume created in `us-east-1a` can only be attached to EC2 instances running in `us-east-1a`. This AZ affinity is a critical design constraint in multi-AZ architectures.

---

## Environment Details

| Parameter | Value |
|---|---|
| AWS Region | `us-east-1` |
| Availability Zone | `us-east-1a` |
| Volume Name | `nautilus-volume` |
| Volume Type | `gp3` |
| Volume Size | `2 GiB` |
| Volume ID | `vol-07ae583017f2c57d8` |
| Encryption | `false` (lab environment) |
| IOPS | `3000` (gp3 baseline) |
| Throughput | `125 MiB/s` (gp3 baseline) |

---

## Tools and Services Used

- **Amazon EC2 / EBS** - Block storage provisioning and management
- **AWS CLI** - Programmatic resource creation, querying, and validation
- **Linux Shell** - Command execution environment
- **JMESPath Query Language** - Targeted CLI output filtering via `--query`

---

## Implementation

### Step 1: Verify AWS Region Configuration

Before provisioning any resource, confirm the AWS CLI is configured against the correct target region. Mismatched region configuration is a common root cause of resource provisioning errors and cost overruns.

```bash
aws configure get region
```

**Expected output:**

```
us-east-1
```

This confirms the CLI is targeting `us-east-1`, the region where all subsequent resources will be created.

> **Operational Note:** In production pipelines, region configuration is often passed explicitly via `--region` flags or environment variables (`AWS_DEFAULT_REGION`) to prevent environment-level misconfigurations from affecting automation.

**Screenshot: AWS CLI active region confirmed as `us-east-1`**

<img width="1034" height="788" alt="AWS CLI region verification output showing us-east-1" src="https://github.com/user-attachments/assets/ce3eb6b3-550d-4fa0-9e32-91eba7e61551" />

---

### Step 2: Identify Available Availability Zones

EBS volumes are Availability Zone-scoped. The target AZ must be confirmed before creation to ensure the volume can be attached to instances in the same zone. Query the available AZs in the region:

```bash
aws ec2 describe-availability-zones \
  --query "AvailabilityZones[*].ZoneName" \
  --output table
```

**Expected output:**

```
---------------------------
|DescribeAvailabilityZones|
+-------------------------+
|  us-east-1a             |
|  us-east-1b             |
|  us-east-1c             |
|  us-east-1d             |
|  us-east-1e             |
|  us-east-1f             |
+-------------------------+
```

**Selected AZ:** `us-east-1a`

> **Key Design Consideration:** In production, AZ selection should align with where the target EC2 instance resides. Creating a volume in the wrong AZ requires snapshotting and re-provisioning, which introduces downtime risk and additional cost. Always confirm instance AZ placement before provisioning attached storage.

**Screenshot: Available Availability Zones in `us-east-1` returned by the CLI**

<img width="1028" height="827" alt="AWS CLI output listing all Availability Zones in us-east-1" src="https://github.com/user-attachments/assets/762ff340-06d7-4518-9f6a-4f32ff800e03" />

---

### Step 3: Create the EBS Volume

With the target AZ confirmed, provision the EBS volume with the required specifications. The `--tag-specifications` flag applies a `Name` tag at creation time, eliminating the need for a separate tagging step.

```bash
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 2 \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=nautilus-volume}]'
```

**Key parameters explained:**

- `--availability-zone us-east-1a` - Pins the volume to the selected AZ for EC2 attachment compatibility
- `--size 2` - Allocates 2 GiB of block storage
- `--volume-type gp3` - Selects the General Purpose SSD v3 type, which provides 3000 IOPS and 125 MiB/s throughput at no additional cost over the base price
- `--tag-specifications` - Applies the `Name` tag inline at creation, ensuring the resource is identifiable immediately

**Returned volume metadata:**

```json
{
    "Iops": 3000,
    "Tags": [{ "Key": "Name", "Value": "nautilus-volume" }],
    "VolumeType": "gp3",
    "MultiAttachEnabled": false,
    "Throughput": 125,
    "VolumeId": "vol-07ae583017f2c57d8",
    "Size": 2,
    "SnapshotId": "",
    "AvailabilityZone": "us-east-1a",
    "State": "creating",
    "CreateTime": "2026-02-01T06:41:52.000Z",
    "Encrypted": false
}
```

> **Operational Note:** The initial `State: creating` is expected. AWS provisions EBS volumes asynchronously. The volume will transition to `available` within seconds. Do not attempt attachment until the state is `available`.

> **Security Note:** In production environments, EBS volumes storing sensitive workload data should be created with `--encrypted` and a KMS key specified via `--kms-key-id`. Unencrypted volumes are acceptable only in non-production or sandboxed environments.

**Screenshot: AWS CLI output confirming EBS volume creation with Volume ID `vol-07ae583017f2c57d8`**

<img width="1041" height="863" alt="AWS CLI JSON response from create-volume showing VolumeId, State, and tags" src="https://github.com/user-attachments/assets/5a9883ce-41f5-498a-9c2d-cfa27142e94e" />

---

### Step 4: Verify Volume State

After creation, verify the volume has transitioned from `creating` to `available`. Use a filtered describe query to isolate the target volume by its `Name` tag:

```bash
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=nautilus-volume" \
  --query "Volumes[*].[VolumeId,Size,VolumeType,State,AvailabilityZone]" \
  --output table
```

**Expected output:**

```
--------------------------------------------------------------
|                      DescribeVolumes                       |
+----------------------+---+------+-----------+-----------+
|  vol-07ae583017f2c57d8 | 2 | gp3  | available | us-east-1a |
+----------------------+---+------+-----------+-----------+
```

**State: `available`** confirms the volume is fully provisioned and ready for EC2 attachment.

> **Troubleshooting:** If the volume remains in `creating` for more than 60 seconds, check for AWS service health events in the target region. Volumes stuck in `creating` are rare but can occur during regional degradation events.

**Screenshot: Volume status table showing `vol-07ae583017f2c57d8` in `available` state**

<img width="1034" height="852" alt="Filtered describe-volumes output confirming volume is available in us-east-1a" src="https://github.com/user-attachments/assets/244fbcbc-d3a4-428d-b014-32d807ad0982" />

---

### Step 5: Validate Volume Properties and Tags

Perform a final tag-level validation to confirm the `Name` tag was correctly applied and is associated with the correct resource type and ID. This step ensures the volume is correctly discoverable by resource management tooling, cost allocation reports, and infrastructure automation.

```bash
aws ec2 describe-tags \
  --filters "Name=resource-id,Values=vol-07ae583017f2c57d8"
```

**Expected output:**

```json
{
    "Tags": [
        {
            "Key": "Name",
            "ResourceId": "vol-07ae583017f2c57d8",
            "ResourceType": "volume",
            "Value": "nautilus-volume"
        }
    ]
}
```

All three tag attributes confirm correct configuration:

- `ResourceId` matches the provisioned volume ID
- `ResourceType` is `volume`, confirming the tag is not orphaned on another resource type
- `Value` matches the intended name `nautilus-volume`

**Screenshot: Tag validation output confirming `Name=nautilus-volume` is correctly applied to the volume**

<img width="1034" height="864" alt="describe-tags output confirming ResourceId, ResourceType, and Value for the EBS volume" src="https://github.com/user-attachments/assets/594bca28-6d05-451a-a45e-87bf70c62099" />

---

## Validation Summary

| Check | Expected | Actual | Result |
|---|---|---|---|
| Region | `us-east-1` | `us-east-1` | PASS |
| Availability Zone | `us-east-1a` | `us-east-1a` | PASS |
| Volume ID | Non-null | `vol-07ae583017f2c57d8` | PASS |
| Volume Size | `2 GiB` | `2 GiB` | PASS |
| Volume Type | `gp3` | `gp3` | PASS |
| Volume State | `available` | `available` | PASS |
| Tag Key | `Name` | `Name` | PASS |
| Tag Value | `nautilus-volume` | `nautilus-volume` | PASS |

**Overall Result: SUCCESS**

---

## Key Decisions

- **gp3 over gp2:** gp3 was selected because it decouples IOPS and throughput from volume size. Unlike gp2 (which scales IOPS proportionally with size), gp3 delivers 3000 IOPS and 125 MiB/s baseline on any volume regardless of size, making it the cost-optimal choice for small workloads. For a 2 GiB volume, gp3 provides 15x more IOPS than gp2 at the same or lower cost.

- **Tag at creation, not after:** Applying the `Name` tag via `--tag-specifications` during `create-volume` ensures the resource is tagged from the moment it exists. Post-creation tagging introduces a window during which the resource is untagged, which can cause issues with tag-based IAM policies, cost allocation filters, and automated compliance checks.

- **Independent storage provisioning:** Decoupling EBS provisioning from EC2 launch gives teams flexibility to pre-allocate storage, modify volume specs before attachment, and version control storage configuration independently from compute.

- **CLI over Console:** Using the AWS CLI enables reproducibility, version control, and integration with CI/CD pipelines. Console-based provisioning cannot be peer-reviewed, diffed, or automated.

---

## Errors and Resolutions

No errors were encountered during this implementation. The following are known edge cases to watch for in similar workflows:

**Issue: Volume stuck in `creating` state**
- Cause: Rare AWS-side provisioning delays, often during service events
- Resolution: Check the AWS Service Health Dashboard. If no active event, delete and recreate the volume. Volumes that fail to become `available` within a few minutes should be treated as failed and reprovisioned.

**Issue: `InvalidParameterValue` on volume type**
- Cause: Specifying a volume type not supported in the target region or AZ (e.g., `io2` in certain regions)
- Resolution: Run `aws ec2 describe-volume-types` to confirm supported types in the target region before provisioning.

**Issue: Volume created in wrong AZ**
- Cause: AZ not explicitly specified or defaulted incorrectly
- Resolution: Create a snapshot of the volume, then restore it into the correct AZ using `aws ec2 create-volume --snapshot-id`. Delete the original volume.

**Issue: Tag not visible in Cost Explorer**
- Cause: Cost allocation tags must be activated in Billing settings before they appear in cost reports
- Resolution: Navigate to AWS Billing > Cost allocation tags and activate the `Name` key.

---

## Best Practices Demonstrated

- **Region verification before provisioning** - Eliminates silent misconfigurations that create resources in unintended regions
- **AZ-aware storage planning** - Ensures volume and compute placement are aligned before any attachment attempt
- **Inline resource tagging** - Tags applied at creation prevent untagged resource windows and enforce governance from day one
- **CLI-based targeted queries** - Using `--filters` and `--query` returns minimal, actionable output rather than full API payloads, reducing cognitive overhead
- **Multi-attribute state validation** - Confirming volume ID, size, type, state, and tag independently closes the loop on all provisioning parameters
- **gp3 as default volume type** - Selecting gp3 over gp2 reflects cost-efficiency best practices for general-purpose workloads

---

## Lessons Learned

- EBS volume state transitions from `creating` to `available` are asynchronous. Automation scripts must implement a polling or waiter pattern (e.g., `aws ec2 wait volume-available`) rather than assuming immediate availability.
- The `--query` parameter with JMESPath expressions is a powerful pattern for extracting specific fields from large JSON payloads. Building familiarity with JMESPath syntax significantly reduces the need for external tools like `jq` in CLI-only environments.
- Resource tagging strategy should be defined at the organizational level before provisioning begins. Retroactively tagging large numbers of resources is operationally expensive and error-prone.

---

## Real-World Relevance

This workflow mirrors production storage provisioning patterns at scale. DevOps and cloud infrastructure teams routinely provision EBS volumes independently of EC2 instances to support:

- Pre-staged storage for scheduled instance launches
- Data volume lifecycle management (snapshot, modify, reattach)
- Infrastructure-as-Code modules where storage and compute are managed in separate Terraform resources
- Disaster recovery pre-provisioning where replacement volumes are kept on standby

The CLI-first approach demonstrated here maps directly to Terraform's `aws_ebs_volume` resource and is the foundation for any automated storage management pipeline.

---

## Skills Demonstrated

- AWS EC2 storage architecture and EBS volume lifecycle management
- AWS CLI proficiency including targeted JMESPath queries and table output formatting
- Resource tagging strategy and cost allocation best practices
- AZ-aware infrastructure design and placement constraints
- State-based validation and readiness checking for provisioned resources
- Production-style documentation and operational handoff standards
