# AWS EBS Snapshot Creation: Point-in-Time Backup for Critical DevOps Volumes

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture and Backup Logic](#architecture-and-backup-logic)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1: Console Access and Region Validation](#step-1-console-access-and-region-validation)
  - [Step 2: Locate the Target EBS Volume](#step-2-locate-the-target-ebs-volume)
  - [Step 3: Create the EBS Snapshot](#step-3-create-the-ebs-snapshot)
  - [Step 4: Verify Snapshot Completion](#step-4-verify-snapshot-completion)
- [Final Outcome](#final-outcome)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Risks and Edge Cases](#risks-and-edge-cases)
- [Lessons Learned](#lessons-learned)
- [Tags](#tags)

---

## Overview

This document details the creation of a manual **Amazon EBS (Elastic Block Store) snapshot** for a critical DevOps volume (`devops-vol`) in the `us-east-1` region. EBS snapshots are incremental, point-in-time backups stored durably in Amazon S3. They serve as the foundation of disaster recovery strategies for EC2-backed workloads, enabling volume restoration, cross-region replication, and data lifecycle management.

This is part of an initial backup strategy rollout for the Nautilus DevOps team, establishing a verified baseline snapshot before automated lifecycle policies are applied.

---

## Problem Statement

The Nautilus DevOps team manages production workloads on EC2 instances backed by EBS volumes. Prior to implementing automated Data Lifecycle Manager (DLM) policies, a **manual snapshot** of the existing `devops-vol` EBS volume is required to:

- Establish an immediate, verifiable recovery point
- Validate snapshot creation permissions and naming conventions
- Confirm backup readiness before automated policies take over

**Volume in scope:** `devops-vol` (`vol-08b91bdb98c576455`), `5 GiB`, `gp2`, located in `us-east-1a`

---

## Architecture and Backup Logic

```
Target Region: us-east-1
                    |
         +----------v-----------+
         |  EBS Volume          |
         |  devops-vol (5 GiB)  |
         |  us-east-1a          |
         +----------+-----------+
                    |
           Create Snapshot
                    |
         +----------v-----------+
         |  EBS Snapshot        |
         |  devops-vol-ss       |
         |  Stored in Amazon S3 |
         |  (Managed by AWS)    |
         +----------------------+
```

**Decision logic:**

- IF target volume exists in `us-east-1` with correct name and state: CREATE snapshot with required name and description, then CONFIRM status reaches `completed`
- ELSE: Surface the error and halt -- do not proceed with incorrect volume

---

## Prerequisites

| Requirement | Details |
|---|---|
| AWS Account access | IAM user or role with `ec2:CreateSnapshot`, `ec2:DescribeVolumes`, `ec2:DescribeSnapshots` permissions |
| Target region | `us-east-1` (N. Virginia) |
| Target volume | `devops-vol` must exist and be in `available` or `in-use` state |
| Console access | AWS Management Console or AWS CLI |

---

## Implementation

### Step 1: Console Access and Region Validation

Log in to the AWS Management Console using your provided credentials. Before navigating to any service, confirm the active region is set to **US East (N. Virginia) / us-east-1**.

**Why this matters:** EBS volumes and snapshots are region-scoped resources. Operating in the wrong region will result in an empty volume list, causing confusion and potential errors.

> **Action:** Click the region selector in the top-right corner of the console and confirm or switch to `us-east-1`.

![AWS Console with region selector open, confirming us-east-1 (N. Virginia) is the active region](https://github.com/user-attachments/assets/fe0f1630-0b5b-4fef-a438-8c3499ca9dc2)
*Region selector confirms us-east-1 (N. Virginia) is the active region. Always validate this before any EBS operation.*

---

### Step 2: Locate the Target EBS Volume

Navigate to **EC2 Dashboard > Elastic Block Store > Volumes**.

Confirm the following volume is present and in the expected state:

| Field | Value |
|---|---|
| Name | `devops-vol` |
| Volume ID | `vol-08b91bdb98c576455` |
| Type | `gp2` |
| Size | `5 GiB` |
| IOPS | `100` |
| Availability Zone | `us-east-1a` |
| Created | `2026/02/16 01:11 GMT+1` |

> **Operational note:** The Snapshot Summary panel at the bottom of the Volumes page shows `0 / 1` backed-up volumes, confirming no snapshot currently exists for this volume. The "Failed to fetch default policy status" error is a known IAM restriction in this environment and does not block manual snapshot creation.

![EC2 Volumes list showing devops-vol (5 GiB, gp2) in us-east-1a with no existing snapshots](https://github.com/user-attachments/assets/2950c000-0254-4bfa-9a54-cb3a32df03b9)
*Volumes list confirms devops-vol exists in us-east-1a. The Snapshot Summary confirms no backup currently exists for this volume.*

---

### Step 3: Create the EBS Snapshot

With `devops-vol` identified, initiate snapshot creation.

**Steps:**

1. Select the checkbox next to `devops-vol`
2. Click **Actions > Create Snapshot**
3. On the **Create snapshot** form, configure the following:

| Field | Value |
|---|---|
| Description | `devops Snapshot` |
| Name tag (Key: `Name`) | `devops-vol-ss` |

4. Confirm **Encryption** shows `Not encrypted` (this matches the source volume configuration)
5. Click **Create snapshot**

> **Key Decision:** The `Name` tag value (`devops-vol-ss`) follows the team's volume snapshot naming convention: `{volume-name}-ss`. This ensures snapshots are easily traceable back to their source volume without querying Volume IDs.

![Create Snapshot form showing source volume vol-08b91bdb98c576455, description set to devops Snapshot, and Name tag set to devops-vol-ss](https://github.com/user-attachments/assets/7436774a-0f71-4175-be4f-04c178bac0c3)
*Create Snapshot form: description and Name tag configured per naming convention before submission.*

After clicking **Create snapshot**, the console displays a green success banner confirming the snapshot ID and source volume:

> **Successfully created snapshot `snap-08d6bc488f3759a94` from volume `vol-08b91bdb98c576455`.**

The banner also surfaces an option to enable **Fast Snapshot Restore (FSR)**, which is relevant for time-sensitive restore scenarios where full performance must be immediately available on restored volumes.

![Success banner confirming snap-08d6bc488f3759a94 was created from vol-08b91bdb98c576455, with option to enable Fast Snapshot Restore](https://github.com/user-attachments/assets/b003fca4-bf49-413c-af81-fb75a52b0e4f)
*Green success banner confirms snapshot creation. Fast Snapshot Restore is surfaced as an option for production environments requiring immediate restore performance.*

---

### Step 4: Verify Snapshot Completion

Navigate to **EC2 Dashboard > Elastic Block Store > Snapshots**.

Confirm the snapshot appears with the following attributes:

| Field | Expected Value | Observed Value |
|---|---|---|
| Name | `devops-vol-ss` | `devops-vol-ss` |
| Snapshot ID | auto-generated | `snap-08d6bc488f3759a94` |
| Volume size | `5 GiB` | `5 GiB` |
| Description | `devops Snapshot` | `devops Snapshot` |
| Storage tier | `Standard` | `Standard` |
| **Snapshot status** | **Completed** | **Completed** |
| Progress | `100%` | `100%` |
| Started | `2026/02/16 01:44 GMT+1` | confirmed |

> **Validation note:** A snapshot is only considered a valid recovery point when its status is `Completed` and progress shows `100%`. Snapshots in `pending` state cannot be used for volume restoration.

![Snapshots list showing devops-vol-ss with status Completed and 100% progress, confirming successful backup](https://github.com/user-attachments/assets/eeac0f39-5b70-4a8f-b265-7c167ec1a174)
*Snapshots list confirms devops-vol-ss reached Completed status at 100% progress -- a valid, restorable recovery point.*

---

## Final Outcome

| Objective | Status |
|---|---|
| Target volume identified in correct region | Confirmed |
| Snapshot created with correct name (`devops-vol-ss`) | Confirmed |
| Snapshot description applied (`devops Snapshot`) | Confirmed |
| Snapshot status reached `Completed` | Confirmed |
| Snapshot ID recorded (`snap-08d6bc488f3759a94`) | Confirmed |

---

## Key Decisions

**Manual snapshot vs. automated DLM policy:** A manual snapshot was chosen as the first backup to establish an immediate recovery point. DLM policies require additional IAM permissions not available in this environment (`dlm:GetLifecyclePolicy` returned access denied). The manual snapshot creates a baseline from which automated policies can take over.

**Naming convention (`devops-vol-ss`):** The `-ss` suffix (snapshot shorthand) applied to the source volume name creates a predictable, human-readable pattern. In environments with many volumes, consistent naming is critical for filtering, cost attribution, and operational triage.

**Standard storage tier:** The `Standard` tier was selected as the snapshot is intended for near-term recovery and reference, not long-term archival. The `Archive` tier offers up to 75% cost reduction but introduces a 24 to 72-hour restore delay -- not appropriate for a primary recovery point on an active volume.

**Fast Snapshot Restore (FSR) not enabled:** FSR was surfaced but not enabled, as this is a non-production volume in a restricted sandbox environment. FSR incurs per-AZ per-hour charges and is best reserved for volumes that require immediate full-performance availability upon restore in production.

---

## Best Practices and Operational Considerations

- **Always verify the region before creating snapshots.** EBS resources are region-scoped and a wrong-region operation will appear to succeed but create an unusable backup relative to the target EC2 instance.
- **Record snapshot IDs immediately after creation.** The success banner displays the ID transiently. Store it in your runbook, CMDB, or tagging system before navigating away.
- **Tag snapshots consistently.** At minimum, apply `Name`, `Environment`, `Owner`, and `CreatedBy` tags. This enables cost tracking, automated cleanup, and audit trails.
- **Do not rely solely on manual snapshots for production volumes.** Implement Data Lifecycle Manager (DLM) policies or AWS Backup to automate snapshot schedules, retention policies, and cross-region copy.
- **Validate restore paths periodically.** Creating snapshots without testing restore procedures provides a false sense of security. Schedule periodic test restores to a non-production environment.
- **Monitor snapshot costs.** EBS snapshots are incremental after the first full backup, but accumulated snapshots across many volumes can generate significant S3 storage costs. Use AWS Cost Explorer and set budget alerts.

---

## Risks and Edge Cases

**Volume in `in-use` state during snapshot:** AWS supports snapshotting volumes while they are attached and in use. However, for application consistency, snapshots of database volumes or volumes with active write I/O should be taken after flushing caches or using application-aware quiescing. AWS does not guarantee application consistency for in-use volumes without additional tooling.

**IAM permission gaps:** This environment surfaces multiple `Access denied` errors (Cost and Usage, DLM policy status). Before deploying snapshots in production, verify that the executing role holds `ec2:CreateSnapshot`, `ec2:DescribeSnapshots`, and `ec2:DescribeVolumes` at minimum. Missing permissions will silently fail in automation contexts.

**Snapshot limits:** AWS accounts have a default soft limit of 100,000 EBS snapshots per region. High-frequency automated snapshots without lifecycle policies will eventually exhaust this quota.

**Full snapshot size on first run:** The first snapshot of a volume captures all allocated data blocks, not just written data. A `5 GiB` volume with only `1 GiB` of actual data will still have an initial snapshot size reflective of all blocks written at least once. Subsequent snapshots are incremental.

---

## Lessons Learned

- **IAM restrictions in sandbox environments do not block all operations.** The `Failed to fetch default policy status` DLM error and Cost and Usage `Access denied` messages are scoped to specific IAM policies. `ec2:CreateSnapshot` was independently authorized, confirming the principle of least privilege applied selectively.
- **The green success banner is the primary in-console confirmation.** Always read it in full -- it contains the snapshot ID, source volume reference, and actionable follow-up options (like Fast Snapshot Restore) that are easy to miss.
- **Snapshot status polling is required before declaring success.** Navigation to the Snapshots section and explicit status confirmation (`Completed`, `100%`) is mandatory. A snapshot creation API call returning success only means the request was accepted -- not that the snapshot is usable.

---

## Tags

`aws` `ebs` `snapshots` `storage` `backup` `cloud-infrastructure` `devops` `ec2` `disaster-recovery` `us-east-1`
