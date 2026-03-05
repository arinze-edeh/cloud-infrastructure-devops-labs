# AWS RDS Snapshot and Restore Operations

### *Enterprise Database Backup and Recovery Workflow | AWS RDS | MySQL 8.4*

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Configuration](#environment-configuration)
- [Implementation](#implementation)
  - [Phase 1: Instance Validation](#phase-1-instance-validation)
  - [Phase 2: Snapshot Creation](#phase-2-snapshot-creation)
  - [Phase 3: Snapshot Restore](#phase-3-snapshot-restore)
  - [Phase 4: Final Verification](#phase-4-final-verification)
- [Results](#results)
- [Key Takeaways](#key-takeaways)
- [Technologies Used](#technologies-used)

---

## Overview

This document details the end-to-end operational process of snapshotting a live Amazon RDS MySQL instance and restoring it to an isolated new instance using the AWS CLI. The workflow validates that an organisation's RDS backup and recovery mechanism is functional before a major infrastructure update is pushed to production.

> **Context:** The Nautilus Development Team required a pre-update backup validation cycle. The DevOps team executed a full snapshot-to-restore pipeline to confirm data integrity and instance recoverability in the `us-east-1` region.

---

## Problem Statement

Before rolling out a major database infrastructure update, the team faced the following operational requirements:

- **Requirement 1:** Guarantee a recoverable snapshot of the current production RDS instance exists before any changes are applied.
- **Requirement 2:** Validate that the snapshot restoration process produces a fully functional, correctly configured RDS instance.
- **Requirement 3:** Confirm the restored instance reaches `available` state with the correct instance class (`db.t3.micro`), proving the backup pipeline is operationally sound.

**Failure to satisfy these requirements** would mean proceeding with a major infrastructure update without a verified recovery path, creating unacceptable risk to production data.

---

## Architecture

```
+-------------------+       Snapshot        +----------------------+
|                   |  ------------------>  |                      |
|   xfusion-rds     |                       |   xfusion-snapshot   |
|   (Source RDS)    |                       |   (Manual Snapshot)  |
|   MySQL 8.4.5     |                       |   Status: available  |
|   db.t3.micro     |                       |   100% Progress      |
|   Status:available|                       |                      |
+-------------------+                       +----------+-----------+
                                                       |
                                                  Restore
                                                       |
                                                       v
                                       +------------------------------+
                                       |  xfusion-snapshot-restore    |
                                       |  (Restored RDS Instance)     |
                                       |  MySQL 8.4.5                 |
                                       |  db.t3.micro                 |
                                       |  Status: available           |
                                       |  VPC: vpc-02545fc0b93b1446f  |
                                       +------------------------------+

Region: us-east-1
Account: 547914434401
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| AWS CLI | Installed and configured |
| IAM Permissions | `rds:CreateDBSnapshot`, `rds:RestoreDBInstanceFromDBSnapshot`, `rds:DescribeDBInstances`, `rds:DescribeDBSnapshots` |
| Region | `us-east-1` |
| Source RDS Instance | `xfusion-rds` in `available` state |
| Target Instance Class | `db.t3.micro` |

---

## Environment Configuration

**Verify identity and region before executing any operations:**

```bash
aws sts get-caller-identity
aws configure get region
```

***Screenshot: Identity and region verification output***

<img width="1028" height="472" alt="image" src="https://github.com/user-attachments/assets/752b4c79-04d2-4e09-bc65-51b41f700d85" />

**Expected output:**

```json
{
    "UserId": "AIDAX7ER6Y5QU3X2L3SPH",
    "Account": "547914434401",
    "Arn": "arn:aws:iam::547914434401:user/kk_labs_user_746364"
}
```

```
us-east-1
```

---

## Implementation

### Phase 1: Instance Validation

**Objective:** Confirm the source RDS instance `xfusion-rds` is in `available` state before initiating a snapshot. Taking a snapshot against an instance in a transitional state can result in an incomplete or failed snapshot.

```bash
aws rds describe-db-instances \
    --db-instance-identifier xfusion-rds \
    --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus]' \
    --output table
```

***Screenshot: Source instance availability confirmation***

<img width="1037" height="653" alt="image" src="https://github.com/user-attachments/assets/ca183227-9819-4741-af65-268e782cce01" />

**Result:**

```
------------------------------
|     DescribeDBInstances    |
+--------------+-------------+
|  xfusion-rds |  available  |
+--------------+-------------+
```

>Do not proceed until status is `available`.

---

### Phase 2: Snapshot Creation

**Objective:** Create a manual snapshot named `xfusion-snapshot` from the source instance. This snapshot serves as both the recovery artifact and the backup validation target.

#### 2a. Create the Snapshot

```bash
aws rds create-db-snapshot \
    --db-snapshot-identifier xfusion-snapshot \
    --db-instance-identifier xfusion-rds
```

***Screenshot: Snapshot creation API response***

<img width="1035" height="867" alt="image" src="https://github.com/user-attachments/assets/e28bedae-b84c-4d2e-929a-478729c0b20d" />

**Key fields from response:**

| Field | Value |
|---|---|
| `DBSnapshotIdentifier` | `xfusion-snapshot` |
| `DBInstanceIdentifier` | `xfusion-rds` |
| `Engine` | `mysql` |
| `EngineVersion` | `8.4.5` |
| `Status` | `creating` |
| `AllocatedStorage` | `5 GB` |
| `StorageType` | `gp2` |
| `SnapshotType` | `manual` |
| `AvailabilityZone` | `us-east-1c` |
| `DBSnapshotArn` | `arn:aws:rds:us-east-1:547914434401:snapshot:xfusion-snapshot` |

#### 2b. Confirm Snapshot Availability

```bash
aws rds describe-db-snapshots \
    --db-snapshot-identifier xfusion-snapshot \
    --query 'DBSnapshots[*].[DBSnapshotIdentifier,Status,PercentProgress]' \
    --output table
```

***Screenshot: Snapshot available at 100% progress***

<img width="1034" height="860" alt="image" src="https://github.com/user-attachments/assets/b9f63122-b445-4cb6-99ac-32526c73c3b8" />

**Result:**

```
------------------------------------------
|           DescribeDBSnapshots          |
+-------------------+-------------+------+
|  xfusion-snapshot |  available  |  100 |
+-------------------+-------------+------+
```

>Do not proceed to restore until `PercentProgress` is `100` and `Status` is `available`.

---

### Phase 3: Snapshot Restore

**Objective:** Restore `xfusion-snapshot` to a new RDS instance named `xfusion-snapshot-restore` with instance class `db.t3.micro`. This validates the snapshot is usable and produces an independently accessible database instance.

#### 3a. Execute the Restore

```bash
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier xfusion-snapshot-restore \
    --db-snapshot-identifier xfusion-snapshot \
    --db-instance-class db.t3.micro
```

***Screenshot: Restore command response with instance in creating state***

![Phase 3a - Restore Initiated](screenshots/05-restore-initiated.png)

**Key fields from restore response:**

| Field | Value |
|---|---|
| `DBInstanceIdentifier` | `xfusion-snapshot-restore` |
| `DBInstanceClass` | `db.t3.micro` |
| `DBInstanceStatus` | `creating` |
| `Engine` | `mysql` |
| `EngineVersion` | `8.4.5` |
| `MasterUsername` | `xfusion_admin` |
| `VpcId` | `vpc-02545fc0b93b1446f` |
| `MultiAZ` | `false` |
| `DBInstanceArn` | `arn:aws:rds:us-east-1:547914434401:db:xfusion-snapshot-restore` |

#### 3b. Monitor Restore Progress

The restore process transitions through expected intermediate states. Each state is normal and expected:

```bash
aws rds describe-db-instances \
    --db-instance-identifier xfusion-snapshot-restore \
    --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,DBInstanceClass]' \
    --output table
```

***Screenshot: Instance transitioning through intermediate states***

![Phase 3b - Restore Progress States](screenshots/06-restore-progress-states.png)

**State progression observed:**

```
creating  -->  configuring-enhanced-monitoring  -->  backing-up  -->  available
```

| State | Meaning |
|---|---|
| `creating` | Instance hardware and storage being provisioned |
| `configuring-enhanced-monitoring` | CloudWatch monitoring agent being configured |
| `backing-up` | Initial automated backup being taken per retention policy |
| `available` | Instance fully operational and accessible |

---

### Phase 4: Final Verification

**Objective:** Confirm the restored instance is `available` and running on the correct `db.t3.micro` instance class, satisfying all task requirements.

```bash
aws rds describe-db-instances \
    --db-instance-identifier xfusion-snapshot-restore \
    --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,DBInstanceClass]' \
    --output table
```

***Screenshot: Final state showing available status and correct instance class***

<img width="1037" height="243" alt="image" src="https://github.com/user-attachments/assets/efa38df6-1efa-4bb2-93f0-5a55f9db7231" />

**Final result:**

```
----------------------------------------------------------
|                   DescribeDBInstances                  |
+---------------------------+------------+---------------+
|  xfusion-snapshot-restore |  available |  db.t3.micro  |
+---------------------------+------------+---------------+
```

---

## Results

| Requirement | Expected | Actual | Status |
|---|---|---|---|
| Source instance state before snapshot | `available` | `available` | PASS |
| Snapshot identifier | `xfusion-snapshot` | `xfusion-snapshot` | PASS |
| Snapshot completion | `100%` progress | `100%` progress | PASS |
| Restored instance identifier | `xfusion-snapshot-restore` | `xfusion-snapshot-restore` | PASS |
| Restored instance class | `db.t3.micro` | `db.t3.micro` | PASS |
| Restored instance final state | `available` | `available` | PASS |
| Region | `us-east-1` | `us-east-1` | PASS |

**All 7 validation gates passed. Zero remediation steps required.**

---

## Key Takeaways

**Gate before snapshot.** Always confirm the source instance is `available` before calling `create-db-snapshot`. Snapshots initiated against transitional instances risk data inconsistency or failure.

**Specify instance class at restore time.** Passing `--db-instance-class` during `restore-db-instance-from-db-snapshot` eliminates a separate `modify-db-instance` call and a second wait cycle, reducing total operation time significantly.

**Intermediate states are not errors.** The `configuring-enhanced-monitoring` and `backing-up` states during restore are standard AWS provisioning steps. Interrupting or re-running the restore command during these states would create duplicate instances and additional cost.

**Manual snapshots do not expire.** Unlike automated snapshots which follow the retention policy, the `xfusion-snapshot` manual snapshot persists until explicitly deleted, making it a durable recovery artifact.

---

## Technologies Used

| Technology | Detail |
|---|---|
| **AWS RDS** | Managed relational database service |
| **MySQL** | Engine: `8.4.5` |
| **AWS CLI** | Command line interface for all operations |
| **IAM** | Identity and access management for scoped permissions |
| **VPC** | `vpc-02545fc0b93b1446f` with 6-AZ subnet coverage |
| **Region** | `us-east-1` (N. Virginia) |

---

<img width="1036" height="426" alt="image" src="https://github.com/user-attachments/assets/d6969664-31d2-4ff2-9d63-4af24df6f333" />




<img width="1033" height="864" alt="image" src="https://github.com/user-attachments/assets/0f988e60-1fd9-47f8-ae4f-ff99dd70f04d" />
<img width="1032" height="868" alt="image" src="https://github.com/user-attachments/assets/83e41f66-63a1-4554-a448-d3c6f98cec03" />
<img width="1029" height="862" alt="image" src="https://github.com/user-attachments/assets/10a80b83-6746-466f-a3d4-7f0038ba44fa" />
<img width="1031" height="619" alt="image" src="https://github.com/user-attachments/assets/1292c891-9006-4c29-9b5c-43e03550e723" />
<img width="1034" height="339" alt="image" src="https://github.com/user-attachments/assets/ca0347ce-ef3c-4e21-9e35-2ef16b0980d7" />
<img width="1035" height="430" alt="image" src="https://github.com/user-attachments/assets/559c7fbe-0d6e-42ac-8079-99f0dd7c420c" />
<img width="1034" height="613" alt="image" src="https://github.com/user-attachments/assets/33e8fa3c-25e5-450a-8e2c-72bd1f466422" />
<img width="1038" height="803" alt="image" src="https://github.com/user-attachments/assets/f9761fd9-d7c3-41e3-b6b6-e85706dec3b3" />



