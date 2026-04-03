# AWS S3 Data Protection: Enabling Bucket Versioning via CLI

**Domain:** AWS Storage | **Service:** Amazon S3 | **Interface:** AWS CLI | **Region:** us-east-1

---

## Table of Contents

- [Overview](#overview)
- [Business Context](#business-context)
- [Objectives](#objectives)
- [Tools and Services Used](#tools-and-services-used)
- [Environment Details](#environment-details)
- [Architecture Summary](#architecture-summary)
- [Implementation](#implementation)
  - [Step 1: Confirm Active AWS Region](#step-1-confirm-active-aws-region)
  - [Step 2: Verify Target S3 Bucket Exists](#step-2-verify-target-s3-bucket-exists)
  - [Step 3: Check Current Versioning Status](#step-3-check-current-versioning-status)
  - [Step 4: Enable S3 Bucket Versioning](#step-4-enable-s3-bucket-versioning)
  - [Step 5: Verify Versioning Is Enabled](#step-5-verify-versioning-is-enabled)
- [Final Result](#final-result)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Security and Best Practices](#security-and-best-practices)
- [Real-World Relevance](#real-world-relevance)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This project documents the end-to-end process of enabling **Amazon S3 bucket versioning** on a pre-existing bucket using the AWS CLI. Versioning is a foundational data protection control that preserves every version of every object stored in a bucket, enabling recovery from accidental deletions, overwrites, and data corruption events.

The implementation follows a **verify, configure, validate** pattern, which is standard practice in production cloud operations. All steps are executed via CLI to support repeatability, auditability, and integration into infrastructure-as-code pipelines.

---

## Business Context

In production cloud environments, unprotected S3 buckets represent a significant data durability risk. Without versioning:

- Accidental object deletions are **permanent and unrecoverable**
- Overwritten objects cannot be **rolled back**
- Compliance requirements for data retention and auditability cannot be met

The objective of this implementation is to harden an existing S3 bucket by enabling versioning, immediately improving its resilience against both operational errors and malicious actions without disrupting live object access.

---

## Objectives

- Confirm the correct AWS region is active before making configuration changes
- Verify the target bucket exists and is accessible
- Audit the current versioning state prior to any modifications
- Enable S3 bucket versioning using a single idempotent CLI command
- Validate the configuration change was applied successfully

---

## Tools and Services Used

| Tool / Service | Purpose |
|---|---|
| **Amazon S3** | Target object storage service |
| **AWS CLI** | Primary interface for all configuration operations |
| **IAM** | Preconfigured credentials scoped to required S3 permissions |
| **Linux Shell** | Execution environment |

---

## Environment Details

| Parameter | Value |
|---|---|
| **Cloud Provider** | AWS |
| **Service** | Amazon S3 |
| **Region** | us-east-1 |
| **Bucket Name** | nautilus-s3-9397 |
| **Access Method** | AWS CLI |
| **Credential Source** | Preconfigured IAM (lab environment) |

---

## Architecture Summary

```
IAM Credentials (preconfigured)
        |
        v
  AWS CLI (local shell)
        |
        v
  Amazon S3 Bucket: nautilus-s3-9397  (us-east-1)
        |
        v
  Versioning Configuration: Status=Enabled
        |
        v
  Every future PUT/DELETE preserves prior object versions
```

---

## Implementation

### Step 1: Confirm Active AWS Region

**Intent:** Before modifying any resource, confirm the CLI is operating against the correct region. This prevents accidental configuration changes being applied to unintended environments.

```bash
aws configure get region
```

**Expected Output:**

```
us-east-1
```

**Screenshot:** CLI confirms the active region is `us-east-1`.

<img width="1046" height="864" alt="AWS CLI region verification showing us-east-1 as the active region" src="https://github.com/user-attachments/assets/9b259c85-fe37-4c5f-8228-691da9ae2b4d" />

---

### Step 2: Verify Target S3 Bucket Exists

**Intent:** List all S3 buckets accessible to the current IAM principal and confirm `nautilus-s3-9397` is present before attempting to modify it. Operating on a non-existent bucket will produce an error and should be caught at this stage.

```bash
aws s3 ls
```

**Expected Output:**

```
2026-02-01 05:21:05 nautilus-s3-9397
```

**Screenshot:** Bucket listing confirms `nautilus-s3-9397` exists and was created on `2026-02-01`.

<img width="1035" height="869" alt="AWS CLI bucket listing confirming nautilus-s3-9397 exists in the account" src="https://github.com/user-attachments/assets/a9a975f2-dd3d-4918-8e64-7d56b26d567f" />

---

### Step 3: Check Current Versioning Status

**Intent:** Audit the existing versioning configuration before making changes. This establishes a pre-change baseline and confirms whether versioning is unset, suspended, or already enabled. An empty response indicates versioning has never been configured on the bucket.

```bash
aws s3api get-bucket-versioning \
  --bucket nautilus-s3-9397
```

**Expected Output (pre-change):**

```
(empty)
```

An empty response from `get-bucket-versioning` means versioning is in its **default unversioned state**. This is distinct from `Status: Suspended`, which indicates versioning was previously enabled and then paused.

**Screenshot:** No output is returned, confirming the bucket has no versioning configuration applied.

<img width="1031" height="850" alt="Pre-change versioning check returns empty output confirming versioning is not yet configured" src="https://github.com/user-attachments/assets/1626b4ac-67bd-4f62-90f6-1ae061c2cf5f" />

---

### Step 4: Enable S3 Bucket Versioning

**Intent:** Apply the versioning configuration to the target bucket using `put-bucket-versioning`. Setting `Status=Enabled` activates versioning for all subsequent object operations. This command is idempotent and safe to re-run.

```bash
aws s3api put-bucket-versioning \
  --bucket nautilus-s3-9397 \
  --versioning-configuration Status=Enabled
```

**Expected Output:**

```
(no output)
```

A silent exit with no output is the correct behavior for a successful `put-bucket-versioning` call. AWS CLI returns output only on errors. The absence of an error message is the confirmation of success at this stage.

**Screenshot:** Command executes without error, confirming the API call was accepted.

<img width="1027" height="742" alt="put-bucket-versioning command executes successfully with no error output" src="https://github.com/user-attachments/assets/443a7967-8f90-43a2-b15a-26c97c8b1c02" />

---

### Step 5: Verify Versioning Is Enabled

**Intent:** Re-query the versioning configuration to confirm the change was applied. This is the critical validation step. A response of `"Status": "Enabled"` closes the change loop and confirms the bucket is now protected.

```bash
aws s3api get-bucket-versioning \
  --bucket nautilus-s3-9397
```

**Expected Output:**

```json
{
    "Status": "Enabled"
}
```

**Screenshot:** JSON response confirms versioning status is `Enabled` on the target bucket.

<img width="1033" height="734" alt="Post-change versioning check returns Status Enabled confirming successful configuration" src="https://github.com/user-attachments/assets/526f05f2-8a9c-45a2-a7e6-76d71d91ef26" />

---

## Final Result

| Objective | Status |
|---|---|
| Region confirmed as us-east-1 | PASS |
| Target bucket `nautilus-s3-9397` verified | PASS |
| Pre-change versioning state audited | PASS |
| Versioning enabled via `put-bucket-versioning` | PASS |
| Post-change status confirmed as `Enabled` | PASS |

The bucket `nautilus-s3-9397` is now fully versioned. All future `PUT`, `DELETE`, and `COPY` operations will create new object versions rather than overwriting or permanently deleting existing ones.

---

## Key Decisions

**CLI over Console:** All operations were performed via the AWS CLI rather than the AWS Management Console. This approach supports repeatability, scripting, and integration into automation pipelines. Console-based changes are difficult to audit, version control, or replicate at scale.

**Pre and post-change auditing:** Querying the versioning state both before and after the change is a deliberate operational discipline. It establishes a baseline, confirms assumptions, and validates the outcome without relying on implicit trust in the API.

**Idempotency awareness:** `put-bucket-versioning` is idempotent. Running it against a bucket with versioning already enabled will not reset or disrupt existing object versions. This makes it safe to include in automated provisioning scripts without conditional guards.

**Versioning vs. replication:** Versioning was chosen as the appropriate control for this scenario. While cross-region replication provides geographic redundancy, versioning addresses the more immediate risk of accidental data mutation within the same bucket.

---

## Errors and Resolutions

No errors were encountered during this implementation. The following are common failure modes to be aware of in similar operations:

**`NoSuchBucket` error on `put-bucket-versioning`**
Cause: The bucket name is incorrect or does not exist in the configured region.
Resolution: Re-run `aws s3 ls` to confirm the exact bucket name, then verify the active region with `aws configure get region`.

**`AccessDenied` on `put-bucket-versioning`**
Cause: The IAM principal lacks `s3:PutBucketVersioning` permission.
Resolution: Attach a policy granting `s3:PutBucketVersioning` on the target bucket ARN, or use a role with the required permissions.

**Empty response treated as an error**
Cause: Engineers unfamiliar with AWS CLI behavior may interpret no output as a failure.
Resolution: A silent exit from `put-bucket-versioning` is the expected success response. Run `get-bucket-versioning` immediately after to confirm the state change.

---

## Security and Best Practices

- **Versioning is not a substitute for backups.** Versioning retains all object versions within the same bucket. A bucket-level deletion, accidental `--recursive` removal with version purge, or compromised credentials can still result in data loss. Combine versioning with cross-region replication and S3 Object Lock for comprehensive data protection.

- **Enable MFA Delete for production buckets.** MFA Delete adds a second layer of protection by requiring multi-factor authentication to permanently delete object versions or suspend versioning. This is strongly recommended for buckets storing sensitive or regulated data.

- **Lifecycle policies manage version accumulation.** Without lifecycle rules, every version of every object is retained indefinitely, which increases storage costs over time. Define policies to expire non-current versions after an appropriate retention window.

- **Versioning cannot be fully disabled once enabled.** Once versioning is enabled on a bucket, it can only be suspended, not reverted to the unversioned state. All existing versioned objects remain versioned. Plan versioning enablement accordingly for new and existing buckets.

- **Audit all versioning changes via CloudTrail.** `PutBucketVersioning` events are logged in AWS CloudTrail. Enable CloudTrail data events on sensitive buckets to maintain a full audit trail of versioning configuration changes.

---

## Real-World Relevance

This workflow reflects standard operational practice for cloud engineers responsible for storage governance. Enabling versioning via CLI is a frequent task in:

- **Incident response preparation:** Ensuring buckets containing application state, configuration files, or backups are protected before a deployment or migration.
- **Compliance enforcement:** Meeting requirements under frameworks such as SOC 2, ISO 27001, and HIPAA that mandate data retention and recoverability controls.
- **Infrastructure hardening pipelines:** Including `put-bucket-versioning` as a post-provisioning step in Terraform, CloudFormation, or Ansible playbooks to enforce versioning on all new buckets by default.

---

## Skills Demonstrated

- Amazon S3 administration and data protection configuration
- AWS CLI proficiency for resource auditing and configuration management
- Pre-change and post-change validation methodology
- Cloud storage security fundamentals including versioning, MFA Delete, and lifecycle policy awareness
- Production-grade operational discipline: verify before change, validate after change

























# AWS S3 Data Protection – Bucket Versioning (CLI)

## Overview
This lab demonstrates how to enable Amazon S3 bucket versioning using the AWS CLI.
Versioning is a critical data-protection mechanism that allows recovery from
accidental deletions, overwrites, and data corruption in production environments.

The task reflects a real-world DevOps scenario where data durability and
recoverability are required for managed cloud storage resources.

---

## Business Context
- Data protection and recovery are fundamental requirements for modern cloud systems.
- The DevOps team was tasked with improving resilience for an existing S3 bucket
  by enabling object versioning.
- This change ensures historical object versions can be restored when needed.

---

## Objectives
- Verify the target S3 bucket exists
- Check current versioning state
- Enable S3 bucket versioning using AWS CLI
- Confirm versioning is successfully enabled

---

## Tools & Services Used
- AWS S3
- AWS CLI
- Linux Shell
- IAM (preconfigured lab credentials)

---

## Environment Details
- Cloud Provider: AWS
- Service: Amazon S3
- Region: us-east-1
- Bucket Name: nautilus-s3-9397
- Access Method: AWS CLI

---

## Step 1: Confirm Active AWS Region

  verify aws region is set
  ensure region equals us-east-1

Command:
  aws configure get region

Expected Result:
  us-east-1

Screenshot:
 <img width="1046" height="864" alt="image" src="https://github.com/user-attachments/assets/9b259c85-fe37-4c5f-8228-691da9ae2b4d" />

---

## Step 2: Verify Target S3 Bucket Exists

  list all s3 buckets
  confirm target bucket is present

Command:
  aws s3 ls

Expected Result:
  nautilus-s3-9397 is listed

Screenshot:
 <img width="1035" height="869" alt="image" src="https://github.com/user-attachments/assets/a9a975f2-dd3d-4918-8e64-7d56b26d567f" />

---

## Step 3: Check Current Versioning Status

  query bucket versioning configuration
  determine if versioning is disabled or suspended

Command:
  aws s3api get-bucket-versioning \
    --bucket nautilus-s3-9397

Expected Result (before change):
  empty output

Screenshot:
  <img width="1031" height="850" alt="image" src="https://github.com/user-attachments/assets/1626b4ac-67bd-4f62-90f6-1ae061c2cf5f" />

---

## Step 4: Enable S3 Bucket Versioning

  apply versioning configuration
  set status to Enabled

Command:
  aws s3api put-bucket-versioning \
    --bucket nautilus-s3-9397 \
    --versioning-configuration Status=Enabled

Expected Result:
  command executes successfully with no output

Screenshot:
<img width="1027" height="742" alt="image" src="https://github.com/user-attachments/assets/443a7967-8f90-43a2-b15a-26c97c8b1c02" />

---

## Step 5: Verify Versioning Is Enabled

  re-check bucket versioning configuration
  confirm status is Enabled

Command:
  aws s3api get-bucket-versioning \
    --bucket nautilus-s3-9397

Expected Result:
  Status: Enabled

Screenshot:
  <img width="1033" height="734" alt="image" src="https://github.com/user-attachments/assets/526f05f2-8a9c-45a2-a7e6-76d71d91ef26" />

---

## Final Result
- S3 bucket versioning successfully enabled
- Bucket is now protected against accidental deletions and overwrites
- Data recovery capabilities improved without impacting existing objects

---

## Security & Best Practices
- Versioning supports rollback and recovery during incidents
- Critical for compliance, backups, and auditability
- Commonly combined with lifecycle policies and MFA delete in production

---

## Real-World Relevance
This workflow mirrors how DevOps and Cloud Engineers implement
data protection controls for production S3 buckets using CLI-based
configuration rather than manual console actions.

---

## Skills Demonstrated
- AWS S3 administration
- Data protection and recovery concepts
- AWS CLI proficiency
- Cloud operations best practices
