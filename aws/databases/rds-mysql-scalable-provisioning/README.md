# AWS RDS MySQL Private Instance Provisioning

> **Enterprise-grade, repeatable runbook for provisioning a private MySQL 8.4.x RDS instance on AWS using the CLI. Problem-focused, resolution-driven, production-ready.**

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Problem Statement](#problem-statement)
- [Prerequisites](#prerequisites)
- [Environment Reference](#environment-reference)
- [Known Errors and Resolutions](#known-errors-and-resolutions)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Phase 1: Identity Verification](#phase-1-identity-verification)
  - [Phase 2: Network Discovery](#phase-2-network-discovery)
  - [Phase 3: Engine Version Validation](#phase-3-engine-version-validation)
  - [Phase 4: Instance Provisioning](#phase-4-instance-provisioning)
  - [Phase 5: Availability Confirmation](#phase-5-availability-confirmation)
  - [Phase 6: Final Verification](#phase-6-final-verification)
- [Verification Checklist](#verification-checklist)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This runbook documents the end-to-end provisioning of a **private, storage-autoscaling-enabled MySQL 8.4.x RDS instance** (`datacenter-rds`) on AWS using the AWS CLI v1. It covers every discovery step, every error encountered, and the exact resolution applied -- making it fully reproducible in any sandbox or staging environment.

| Attribute | Value |
|-----------|-------|
| **Instance Identifier** | `datacenter-rds` |
| **Engine** | MySQL `8.4.8` |
| **Instance Class** | `db.t3.micro` |
| **Region** | `us-east-1` |
| **Publicly Accessible** | `false` |
| **Multi-AZ** | `false` |
| **Storage Autoscaling Threshold** | `50 GB` |
| **Initial Allocated Storage** | `20 GB` |
| **Storage Type** | `gp2` |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   AWS Region: us-east-1                 │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │         VPC: vpc-0c0636743c8122ad0 (Default)    │   │
│   │                                                 │   │
│   │   ┌──────────────────────────────────────────┐  │   │
│   │   │   Security Group: sg-08e6d60d4a9f62ee9   │  │   │
│   │   │                                          │  │   │
│   │   │   ┌───────────────────────────────────┐  │  │   │
│   │   │   │  RDS Instance: datacenter-rds     │  │  │   │
│   │   │   │  Engine:  MySQL 8.4.8             │  │  │   │
│   │   │   │  Class:   db.t3.micro             │  │  │   │
│   │   │   │  Storage: 20GB gp2 (max 50GB)     │  │  │   │
│   │   │   │  Access:  Private only            │  │  │   │
│   │   │   └───────────────────────────────────┘  │  │   │
│   │   └──────────────────────────────────────────┘  │   │
│   │                                                 │   │
│   │   Subnets: us-east-1a/b/c/d/e/f (default)       │   │
│   └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Problem Statement

The Nautilus Development Team required a new **private RDS instance** to store critical application data during the initial development phase. Requirements:

- Instance name: `datacenter-rds`
- Template: `sandbox` (single-AZ, free-tier eligible)
- Engine: MySQL, version `8.4.x`
- Instance class: `db.t3.micro`
- Storage autoscaling enabled with a `50 GB` threshold
- Instance must reach `available` state before task submission
- Region: `us-east-1` only

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AWS CLI | v1 installed and configured |
| IAM Permissions | `rds:CreateDBInstance`, `ec2:DescribeVpcs`, `ec2:DescribeSecurityGroups`, `rds:DescribeDBEngineVersions`, `rds:DescribeDBSubnetGroups` |
| Region | `us-east-1` |
| Shell | Bash (Linux/macOS) |

Verify CLI access before starting:

```bash
aws sts get-caller-identity
```

**Expected output:**

```json
{
    "UserId": "AIDAZY5UTYMPLHSJUXIWQ",
    "Account": "672004293406",
    "Arn": "arn:aws:iam::672004293406:user/kk_labs_user_998726"
}
```

***Screenshot: `Verified IAM identity confirming correct account and user before provisioning.`***
<img width="1030" height="401" alt="image" src="https://github.com/user-attachments/assets/9b8bcefb-8e0e-4cfc-9a7a-25e5ccb422fa" />

---

## Environment Reference

| Resource | Value |
|----------|-------|
| AWS Account ID | `672004293406` |
| IAM User | `kk_labs_user_998726` |
| Default VPC | `vpc-0c0636743c8122ad0` |
| Default Security Group | `sg-08e6d60d4a9f62ee9` |
| DB Subnet Group | `default` (auto-resolved, no custom group existed) |
| MySQL Version Used | `8.4.8` |
| Master Username | `admin` |

---

## Known Errors and Resolutions

This section documents every error encountered during provisioning and the exact fix applied. This is the most operationally valuable section for teams repeating this process.

---

### Error 1: `Unknown options: false`

**Command that triggered it:**

```bash
--publicly-accessible false
```

**Full error message:**

```
Unknown options: false
```

**Root Cause:**

AWS CLI v1 does not accept boolean values as arguments for boolean flags. The flag `--publicly-accessible` is a binary switch -- it does not take a `true/false` argument.

**Resolution:**

Replace `--publicly-accessible false` with the negation flag:

```bash
# WRONG (CLI v1)
--publicly-accessible false

# CORRECT (CLI v1)
--no-publicly-accessible
```

***Screenshot: CLI v1 rejecting the `false` value passed to `--publicly-accessible`.***
<img width="1038" height="607" alt="image" src="https://github.com/user-attachments/assets/47c94546-8bcc-4ec3-8e53-bf0c7b295b28" />

---

### Error 2: `InvalidParameterValue` -- MasterUserPassword

**Full error message:**

```
An error occurred (InvalidParameterValue) when calling the CreateDBInstance operation:
The parameter MasterUserPassword is not a valid password.
Only printable ASCII characters besides '/', '@', '"', ' ' may be used.
```

**Root Cause:**

The `@` character is explicitly prohibited in RDS master user passwords regardless of quoting in the shell. AWS validates this server-side before instance creation begins.

**Prohibited characters in RDS passwords:**

| Character | Symbol |
|-----------|--------|
| Forward slash | `/` |
| At sign | `@` |
| Double quote | `"` |
| Space | ` ` |

**Resolution:**

Remove all prohibited characters. Replace `Nautilus@12345` with a compliant password:

```bash
# WRONG
--master-user-password "Nautilus@12345"

# CORRECT
--master-user-password "Nautilus12345!"
```

***Screenshot: RDS rejecting the `@` character in the master user password.***

<img width="1036" height="433" alt="image" src="https://github.com/user-attachments/assets/24f28393-ffbf-4c74-96d8-bbbecd23e29f" />

---

## Step-by-Step Implementation

### Phase 1: Identity Verification

Confirm the correct IAM identity is active before any resource creation:

```bash
aws sts get-caller-identity
```

***Screenshot: Confirmed IAM user `kk_labs_user_998726` in account `672004293406`.***
<img width="1030" height="401" alt="image" src="https://github.com/user-attachments/assets/9b8bcefb-8e0e-4cfc-9a7a-25e5ccb422fa" />

---

### Phase 2: Network Discovery

#### Step 2a: Check for existing DB subnet groups

```bash
aws rds describe-db-subnet-groups \
  --region us-east-1 \
  --query "DBSubnetGroups[*].{Name:DBSubnetGroupName,VPC:VpcId,Status:SubnetGroupStatus}" \
  --output table
```

**Result:** Empty output -- no custom subnet groups existed. AWS will use the default VPC subnets automatically. No `--db-subnet-group-name` flag is required in the create command.

***Screenshot: `Empty subnet group list confirming reliance on the default VPC.`***
<img width="1040" height="480" alt="image" src="https://github.com/user-attachments/assets/bf686852-2117-47c6-b153-496a01cf5c47" />

#### Step 2b: Retrieve the default VPC ID

```bash
aws ec2 describe-vpcs \
  --region us-east-1 \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text
```

**Result:** `vpc-0c0636743c8122ad0`

#### Step 2c: Retrieve the default security group

```bash
aws ec2 describe-security-groups \
  --region us-east-1 \
  --filters "Name=vpc-id,Values=vpc-0c0636743c8122ad0" "Name=group-name,Values=default" \
  --query "SecurityGroups[0].GroupId" \
  --output text
```

**Result:** `sg-08e6d60d4a9f62ee9`

***Screenshot: `Default VPC ID and Security Group ID retrieved for use in instance creation.`***
<img width="1034" height="461" alt="image" src="https://github.com/user-attachments/assets/808bba61-6ed4-4b1e-a53d-383132992606" />

---

### Phase 3: Engine Version Validation

Confirm available MySQL 8.4.x engine versions in `us-east-1`:

```bash
aws rds describe-db-engine-versions \
  --engine mysql \
  --region us-east-1 \
  --query "DBEngineVersions[?contains(EngineVersion, '8.4')].{Version:EngineVersion,Status:Status}" \
  --output table
```

**Result:**

```
--------------------------
|DescribeDBEngineVersions|
+------------+-----------+
|   Status   |  Version  |
+------------+-----------+
|  available |  8.4.3    |
|  available |  8.4.4    |
|  available |  8.4.5    |
|  available |  8.4.6    |
|  available |  8.4.7    |
|  available |  8.4.8    |
+------------+-----------+
```

**Decision:** `8.4.8` selected -- latest patch, most stable, fully available.

***Screenshot: `Available MySQL 8.4.x engine versions; 8.4.8 selected as the target version.`***
<img width="1035" height="666" alt="image" src="https://github.com/user-attachments/assets/581b40ba-63eb-46a3-9fd3-409f86dca23f" />

---

### Phase 4: Instance Provisioning

With all values confirmed, run the final creation command:

```bash
aws rds create-db-instance \
  --db-instance-identifier datacenter-rds \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.4.8 \
  --master-username admin \
  --master-user-password "Nautilus12345!" \
  --allocated-storage 20 \
  --storage-type gp2 \
  --no-multi-az \
  --no-publicly-accessible \
  --no-deletion-protection \
  --backup-retention-period 1 \
  --max-allocated-storage 50 \
  --vpc-security-group-ids sg-08e6d60d4a9f62ee9 \
  --region us-east-1
```

**Parameter Reference:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `--db-instance-identifier` | `datacenter-rds` | Required instance name |
| `--db-instance-class` | `db.t3.micro` | Free-tier eligible, sandbox template |
| `--engine` | `mysql` | Required engine |
| `--engine-version` | `8.4.8` | Latest stable 8.4.x |
| `--master-username` | `admin` | Root DB user |
| `--master-user-password` | `Nautilus12345!` | No `/`, `@`, `"`, or spaces |
| `--allocated-storage` | `20` | Initial size in GB (minimum for MySQL) |
| `--storage-type` | `gp2` | General Purpose SSD |
| `--no-multi-az` | -- | Single-AZ (sandbox/free-tier) |
| `--no-publicly-accessible` | -- | Private instance (CLI v1 boolean flag) |
| `--no-deletion-protection` | -- | Allows cleanup post-task |
| `--backup-retention-period` | `1` | Required for autoscaling to be valid |
| `--max-allocated-storage` | `50` | Enables autoscaling; sets 50 GB threshold |
| `--vpc-security-group-ids` | `sg-08e6d60d4a9f62ee9` | Default VPC security group |

**Expected initial status in response:** `"DBInstanceStatus": "creating"`

***Screenshots: Successful `create-db-instance` response showing `"DBInstanceStatus": "creating"` and `"MaxAllocatedStorage": 50`.***
<img width="1033" height="843" alt="image" src="https://github.com/user-attachments/assets/43730ca1-0737-4620-9897-98a82fde6de2" />
<img width="1029" height="865" alt="image" src="https://github.com/user-attachments/assets/90ecf011-cdbc-4085-bc03-b2717bf9b556" />
<img width="1031" height="859" alt="image" src="https://github.com/user-attachments/assets/9754ac43-d043-49d3-8820-051071cea06a" />
<img width="1027" height="622" alt="image" src="https://github.com/user-attachments/assets/6c115965-9b51-42ca-96b2-f4c9c97059a8" />

---

### Phase 5: Availability Confirmation

Block until the instance reaches `available` state (polls every 15 seconds, times out after ~30 minutes):

```bash
aws rds wait db-instance-available \
  --db-instance-identifier datacenter-rds \
  --region us-east-1
```

The command returns silently to the prompt when the instance is `available`. Typical wait time: **10 to 15 minutes**.

Alternatively, poll manually:

```bash
aws rds describe-db-instances \
  --db-instance-identifier datacenter-rds \
  --region us-east-1 \
  --query "DBInstances[0].DBInstanceStatus" \
  --output text
```

Run until output changes from `creating` or `backing-up` to `available`.

***Screenshot: Waiter command returning to prompt, confirming instance reached `available` state.***
<img width="1032" height="694" alt="image" src="https://github.com/user-attachments/assets/50198282-f6fa-461e-8f12-cb781429d205" />

---

### Phase 6: Final Verification

Run the consolidated verification query to confirm all task requirements:

```bash
aws rds describe-db-instances \
  --db-instance-identifier datacenter-rds \
  --region us-east-1 \
  --query "DBInstances[0].{Status:DBInstanceStatus,Engine:Engine,Version:EngineVersion,Class:DBInstanceClass,Public:PubliclyAccessible,MultiAZ:MultiAZ,AutoscaleMaxGB:MaxAllocatedStorage}" \
  --output table
```

**Expected output:**

```
------------------------------------------------------------------------------------------
|                                   DescribeDBInstances                                  |
+----------------+--------------+---------+----------+---------+-------------+-----------+
| AutoscaleMaxGB |    Class     | Engine  | MultiAZ  | Public  |   Status    |  Version  |
+----------------+--------------+---------+----------+---------+-------------+-----------+
|  50            |  db.t3.micro |  mysql  |  False   |  False  |  available  |  8.4.8    |
+----------------+--------------+---------+----------+---------+-------------+-----------+
```

***Screenshot: Final verification table confirming all 7 required attributes are correct and the instance is `available`.***

<img width="1036" height="388" alt="image" src="https://github.com/user-attachments/assets/b489bdc6-6070-4d6f-8526-96feab8acdbb" />

---

## Verification Checklist

- [ ] `DBInstanceStatus` is `available`
- [ ] `Engine` is `mysql`
- [ ] `EngineVersion` starts with `8.4`
- [ ] `DBInstanceClass` is `db.t3.micro`
- [ ] `PubliclyAccessible` is `False`
- [ ] `MultiAZ` is `False`
- [ ] `MaxAllocatedStorage` is `50`
- [ ] Instance is in region `us-east-1`

---

## Lessons Learned

### 1. AWS CLI v1 vs v2 Boolean Flag Syntax

**Problem:** `--publicly-accessible false` is valid in CLI v2 but throws `Unknown options: false` in CLI v1.

**Rule:** In CLI v1, always use the negation prefix form for boolean flags:

```bash
# v1 pattern
--no-publicly-accessible
--no-multi-az
--no-deletion-protection
```

### 2. RDS Password Character Restrictions Are Server-Side

**Problem:** Shell quoting does not bypass RDS password validation. The characters `/`, `@`, `"`, and ` ` are rejected by the RDS API itself, not the CLI.

**Rule:** Always compose RDS passwords from alphanumeric characters plus `!`, `#`, `$`, `%`, `^`, `&`, `*`, `-`, `_`, `+`, `=` only.

### 3. Storage Autoscaling Is Enabled via a Single Parameter

**Problem:** Teams often look for a separate `--enable-storage-autoscaling` flag that does not exist.

**Rule:** Setting `--max-allocated-storage` to any value greater than `--allocated-storage` simultaneously enables autoscaling and sets the threshold. No additional flag is required.

### 4. Empty DB Subnet Groups Output Means Use the Default VPC

**Problem:** No custom subnet group found -- teams sometimes interpret this as a blocking error.

**Rule:** When `describe-db-subnet-groups` returns empty, omit `--db-subnet-group-name`. AWS automatically assigns the `default` subnet group tied to the account's default VPC.

### 5. Use the Waiter Before Submitting

**Problem:** Instance creation returns immediately with `creating` status. Submitting a graded task at this point will fail validation.

**Rule:** Always run `aws rds wait db-instance-available` or poll `DBInstanceStatus` manually until `available` is confirmed.

---

## References

- [AWS RDS CLI Reference -- create-db-instance](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-instance.html)
- [AWS RDS CLI Reference -- wait db-instance-available](https://docs.aws.amazon.com/cli/latest/reference/rds/wait/db-instance-available.html)
- [RDS Storage Autoscaling](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIOPS.StorageTypes.html#USER_PIOPS.Autoscaling)
- [AWS CLI v1 Boolean Parameters](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-parameters-types.html)
- [RDS MySQL Password Constraints](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Limits.html)

---


