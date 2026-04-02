# Attaching an EBS Volume to an EC2 Instance via AWS CLI

## Overview

This document provides a production-style, step-by-step guide for attaching an existing Amazon EBS volume to a running EC2 instance using the AWS CLI. It is intended for DevOps engineers, cloud administrators, and infrastructure teams working in AWS environments.

The task is part of a broader incremental cloud migration workflow executed by the Nautilus DevOps team. All operations are scoped to the **us-east-1** region and follow AWS best practices for block storage management.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Prerequisites](#prerequisites)
- [Architecture Context](#architecture-context)
- [Implementation Steps](#implementation-steps)
  - [Step 1: Verify AWS CLI Region Configuration](#step-1-verify-aws-cli-region-configuration)
  - [Step 2: Identify the Target EC2 Instance](#step-2-identify-the-target-ec2-instance)
  - [Step 3: Identify the Target EBS Volume](#step-3-identify-the-target-ebs-volume)
  - [Step 4: Attach the EBS Volume to the EC2 Instance](#step-4-attach-the-ebs-volume-to-the-ec2-instance)
  - [Step 5: Verify Volume Attachment](#step-5-verify-volume-attachment)
    - [5a: Check Attachment State](#5a-check-attachment-state)
    - [5b: Verify Block Device Visibility at OS Level](#5b-verify-block-device-visibility-at-os-level)
    - [5c: Confirm Full Attachment Details](#5c-confirm-full-attachment-details)
- [Final Outcome](#final-outcome)
- [Lessons Learned and Operational Considerations](#lessons-learned-and-operational-considerations)

---

## Problem Statement

The Nautilus DevOps team required a storage expansion for an existing EC2 instance (`nautilus-ec2`). A pre-provisioned EBS volume (`nautilus-volume`) needed to be discovered, correctly identified, attached to the running instance, and verified as operational, all without any console access or manual intervention.

---

## Prerequisites

Before beginning, confirm the following:

- AWS CLI is installed and configured with the appropriate IAM credentials
- The IAM user or role has the following permissions:
  - `ec2:DescribeInstances`
  - `ec2:DescribeVolumes`
  - `ec2:AttachVolume`
- The EC2 instance (`nautilus-ec2`) is in a **running** state in `us-east-1`
- The EBS volume (`nautilus-volume`) is in an **available** state and located in the same Availability Zone as the instance

---

## Architecture Context

| Resource | Identifier | Region |
|---|---|---|
| EC2 Instance | `nautilus-ec2` | us-east-1 |
| EBS Volume | `nautilus-volume` | us-east-1 |
| Device Name | `/dev/sdb` | N/A |

---

## Implementation Steps

### Step 1: Verify AWS CLI Region Configuration

Before executing any resource operations, confirm the AWS CLI is targeting the correct region. Mismatched regions are a leading cause of "resource not found" errors in multi-region environments.

```bash
aws configure get region
```

**Expected output:**
```
us-east-1
```

> **Best Practice:** Always verify the active region before performing resource operations. For automation scripts, pass `--region us-east-1` explicitly rather than relying on default configuration.

**Screenshot: Region verification output**

![Region verification](https://github.com/user-attachments/assets/aa34b752-7134-4b73-90af-19b23af74e75)

---

### Step 2: Identify the Target EC2 Instance

Use tag-based filtering to dynamically retrieve the Instance ID for `nautilus-ec2`. Storing the ID in a variable eliminates hardcoding and reduces the risk of human error in subsequent commands.

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text)
echo $INSTANCE_ID
```

**Expected output:**
```
i-071b8164197111728
```

> **Operational Note:** If the `INSTANCE_ID` variable is empty, the instance may not exist, the tag may be misconfigured, or the instance may be in a terminated state. Use `--output json` for a full response to diagnose the issue.

> **Edge Case:** If multiple instances share the same `Name` tag, the query will return multiple IDs. Ensure tag uniqueness across the environment, or add additional filters such as `instance-state-name=running`.

**Screenshot: EC2 Instance ID retrieval**

![EC2 instance ID retrieval](https://github.com/user-attachments/assets/093ab8b8-b14e-4273-9852-42afc5f93ce0)

---

### Step 3: Identify the Target EBS Volume

Similarly, retrieve the Volume ID for `nautilus-volume` using tag-based filtering and store it in a variable for reuse.

```bash
VOLUME_ID=$(aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=nautilus-volume" \
  --query "Volumes[].VolumeId" \
  --output text)
echo $VOLUME_ID
```

**Expected output:**
```
vol-0601c88aa49288062
```

> **Best Practice:** Before attaching, confirm the volume state is `available`. A volume in `in-use` state is already attached to another instance and cannot be attached again unless it is a Multi-Attach enabled `io1`/`io2` volume.

> **Troubleshooting:** To check volume state before proceeding:
> ```bash
> aws ec2 describe-volumes --volume-ids $VOLUME_ID --query "Volumes[0].State"
> ```

**Screenshot: EBS Volume ID retrieval**

![EBS volume ID retrieval](https://github.com/user-attachments/assets/2b4385b3-6293-4358-9083-02849ae476a8)

---

### Step 4: Attach the EBS Volume to the EC2 Instance

With both resource identifiers confirmed, attach the volume to the instance using the designated device name `/dev/sdb`.

```bash
aws ec2 attach-volume \
  --volume-id $VOLUME_ID \
  --instance-id $INSTANCE_ID \
  --device /dev/sdb
```

**Expected JSON response:**
```json
{
    "VolumeId": "vol-0601c88aa49288062",
    "InstanceId": "i-071b8164197111728",
    "Device": "/dev/sdb",
    "State": "attaching",
    "AttachTime": "2026-02-13T06:51:59.197Z"
}
```

> **Important:** An initial state of `attaching` is expected and normal. The attachment process is asynchronous. Proceed to Step 5 to confirm the final `attached` state.

> **Device Naming Note:** On modern Nitro-based EC2 instances, the kernel may remap `/dev/sdb` to `/dev/nvme1n1` or similar NVMe device paths. The AWS-assigned device name (`/dev/sdb`) is used for the API call, but the OS-visible path may differ. Use `lsblk` inside the instance to identify the actual device.

> **Risk:** Attaching a volume to the wrong device path or to an instance in a different Availability Zone will result in an error. Always verify that the volume and instance reside in the same AZ before initiating attachment.

**Screenshot: Volume attachment command and initial response**

![Volume attach command output](https://github.com/user-attachments/assets/fad3c318-9cd6-4038-99a4-d58bec0b3c12)

---

### Step 5: Verify Volume Attachment

#### 5a: Check Attachment State

Poll the volume state to confirm it has transitioned from `attaching` to `attached`.

```bash
aws ec2 describe-volumes \
  --volume-ids $VOLUME_ID \
  --query "Volumes[0].Attachments[0].State"
```

**Expected output:**
```
"attached"
```

**Screenshot: Attachment state confirmation**

![Attachment state verification](https://github.com/user-attachments/assets/8e918200-00e1-41df-8b11-620764529996)

---

#### 5b: Verify Block Device Visibility at OS Level

Run `lsblk` to confirm that the newly attached volume appears as a block device on the instance. This step bridges the AWS API confirmation with actual OS-level visibility.

```bash
lsblk
```

**Example output (Nitro-based instance):**
```
NAME        MAJ:MIN RM    SIZE RO TYPE MOUNTPOINT
nvme1n1     259:0    0  476.9G  0 disk
nvme0n1     259:1    0  476.9G  0 disk
├─nvme0n1p1 259:6    0    256M  0 part
├─nvme0n1p2 259:7    0     31G  0 part
├─nvme0n1p3 259:8    0      1G  0 part
└─nvme0n1p4 259:9    0  444.7G  0 part
```

> **Nitro Instance Behavior:** On Nitro-based instances, EBS volumes attached as `/dev/sdb` appear in the OS as `/dev/nvme1n1` or similar NVMe paths. This is expected behavior. The `lsblk` output showing a new disk confirms the OS has recognized the attachment.

> **Next Step (Post-Lab):** If this volume will be used for data storage, the next steps would be to create a filesystem (`mkfs -t xfs /dev/nvme1n1`) and mount it (`mount /dev/nvme1n1 /mnt/data`). For persistent mounts, add the device to `/etc/fstab`.

**Screenshot: OS-level block device listing**


![Full attachment metadata](https://github.com/user-attachments/assets/c7cf1337-8ce1-426a-b77b-47bac3402406)

---

#### 5c: Confirm Full Attachment Details

Retrieve the full attachment metadata to verify all parameters are correct, including the device path, instance association, and termination behavior.

```bash
aws ec2 describe-volumes \
  --volume-ids vol-0601c88aa49288062 \
  --query "Volumes[0].Attachments"
```

**Expected output:**
```json
[
    {
        "DeleteOnTermination": false,
        "VolumeId": "vol-0601c88aa49288062",
        "InstanceId": "i-071b8164197111728",
        "Device": "/dev/sdb",
        "State": "attached",
        "AttachTime": "2026-02-13T06:51:59.000Z"
    }
]
```

> **Operational Note:** `DeleteOnTermination: false` confirms the volume will persist after the instance is terminated. This is the correct behavior for data volumes. If this were a temporary scratch volume, you may want to set `DeleteOnTermination: true` to avoid orphaned volume costs.

**Screenshot: Full attachment metadata**

![OS block device listing via lsblk](https://github.com/user-attachments/assets/7596bc69-0f0c-4039-b083-0fe2ee761fcd)

---

## Final Outcome

| Objective | Status |
|---|---|
| Region confirmed as `us-east-1` | Completed |
| EC2 instance `nautilus-ec2` identified | Completed |
| EBS volume `nautilus-volume` identified | Completed |
| Volume attached at `/dev/sdb` | Completed |
| Volume state confirmed as `attached` | Completed |
| Block device visible at OS level | Confirmed |

---

## Lessons Learned and Operational Considerations

- **Tag consistency is critical.** Tag-based resource discovery is only reliable when naming conventions are enforced consistently across the environment. Implement tagging policies using AWS Organizations or Service Control Policies (SCPs).
- **Availability Zone alignment is required.** EBS volumes can only be attached to instances in the same AZ. Always validate AZ alignment before provisioning volumes for a target instance.
- **Nitro device remapping is expected.** Do not confuse the AWS device name (`/dev/sdb`) with the OS-visible NVMe path. Use `lsblk` or `nvme list` inside the instance to identify the correct device for filesystem operations.
- **Attachment is asynchronous.** The API returns `attaching` immediately. Always poll the attachment state before proceeding with downstream operations.
- **`DeleteOnTermination` defaults to `false` for attached volumes.** Audit this setting for all data volumes to prevent accidental data loss or orphaned cost accumulation.

---

## Tags

`aws` `ec2` `ebs` `storage` `block-storage` `cloud` `devops` `infrastructure` `nautilus` `us-east-1`
