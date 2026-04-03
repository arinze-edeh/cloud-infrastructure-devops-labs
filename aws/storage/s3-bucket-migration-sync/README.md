# [AWS S3 Cross-Bucket Data Migration and Sync](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)

> **Migrating 3,349 objects (76 MB) across S3 buckets using AWS CLI with zero data loss and full consistency verification**

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step 1: Verify AWS CLI Authentication](#step-1-verify-aws-cli-authentication)
- [Step 2: Create the Destination Bucket](#step-2-create-the-destination-bucket)
- [Step 3: Audit Source Bucket Object Count and Size](#step-3-audit-source-bucket-object-count-and-size)
- [Step 4: Inspect Source Bucket Contents](#step-4-inspect-source-bucket-contents)
- [Step 5: Confirm Destination Bucket is Empty](#step-5-confirm-destination-bucket-is-empty)
- [Step 6: Sync Data from Source to Destination](#step-6-sync-data-from-source-to-destination)
- [Step 7: Validate Destination Bucket Contents Post-Migration](#step-7-validate-destination-bucket-contents-post-migration)
- [Step 8: Verify Data Consistency with Dry-Run](#step-8-verify-data-consistency-with-dry-run)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices and Lessons Learned](#best-practices-and-lessons-learned)
- [Validation Checklist](#validation-checklist)
- [Key AWS Services Used](#key-aws-services-used)
- [Outcome](#outcome)

---

## Overview

This project documents a production-style S3 bucket data migration executed entirely via the AWS CLI. The task involved provisioning a new private S3 bucket and migrating all objects from an existing source bucket, then verifying consistency using both object listing and a dry-run sync. All operations were performed in the **us-east-1** region.

This pattern is directly applicable to:

- **Data consolidation** across environments (dev to staging, staging to prod)
- **Bucket renaming or restructuring** without downtime
- **Disaster recovery pre-positioning** by replicating critical data to a standby bucket
- **Cross-account or cross-region migrations** using the same `aws s3 sync` primitives

---

## Problem Statement

An existing S3 bucket (`xfusion-s3-16544`) contains production data across 3,349 objects totalling approximately 76 MB, including WordPress core files organized into structured prefixes (`wp-admin/`, `wp-content/`, `wp-includes/`). The requirement is to:

1. Provision a new private S3 bucket (`xfusion-sync-10728`) in the same region
2. Migrate all objects from the source to the destination with full fidelity
3. Verify data consistency to confirm zero data loss or corruption

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                      AWS us-east-1                       │
│                                                          │
│  ┌─────────────────────┐     aws s3 sync     ┌────────────────────────┐  │
│  │  xfusion-s3-16544   │ ──────────────────► │  xfusion-sync-10728    │  │
│  │  (Source Bucket)    │                     │  (Destination Bucket)  │  │
│  │  3,349 objects      │                     │  Private ACL           │  │
│  │  ~76 MB             │                     │  Standard_LRS storage  │  │
│  └─────────────────────┘                     └────────────────────────┘  │
│                                                          │
│                    IAM: kk_labs_user_831181              │
└──────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- AWS CLI v2 installed and configured
- IAM user with the following permissions:
  - `s3:CreateBucket`
  - `s3:ListBucket`
  - `s3:GetObject`
  - `s3:PutObject`
- AWS credentials configured via `~/.aws/credentials` or environment variables
- Target region: `us-east-1`

---

## Step 1: Verify AWS CLI Authentication

Before performing any operations, confirm the active IAM identity to ensure the correct credentials and permissions are in scope. This guards against accidental execution under an unintended account or role.

```bash
aws iam get-user
```

**Expected output confirms:**

- The active IAM username (`kk_labs_user_831181`)
- The account ARN, verifying the correct AWS account
- That credentials are valid and the CLI is properly configured

> **Operational note:** In production environments, prefer `aws sts get-caller-identity` for assumed roles or federated identities. `aws iam get-user` applies specifically to IAM users with long-term credentials.

📸 *Screenshot: IAM identity confirmed for `kk_labs_user_831181` in account `907301661335`*

<img width="1036" height="469" alt="aws iam get-user output confirming active IAM identity" src="https://github.com/user-attachments/assets/2ef857c9-3fdb-4f53-8984-e59dfb9d55b8" />

---

## Step 2: Create the Destination Bucket

Provision the new destination bucket in `us-east-1`. Specifying the region explicitly prevents the `--region` defaulting issue that can route bucket creation to an unintended endpoint.

```bash
aws s3 mb s3://xfusion-sync-10728 --region us-east-1
```

**Expected output:**

```
make_bucket: xfusion-sync-10728
```

> **Key considerations:**
> - S3 bucket names are globally unique. Naming collisions will return an error without creating the bucket.
> - The bucket is private by default. No public access is granted unless explicitly configured.
> - For production workloads, consider enabling versioning and server-side encryption (SSE-S3 or SSE-KMS) immediately after creation.

📸 *Screenshot: Destination bucket `xfusion-sync-10728` created successfully*

<img width="1032" height="673" alt="aws s3 mb output confirming new bucket creation" src="https://github.com/user-attachments/assets/dbc26f33-dad8-4508-ba1e-3fb8c9d4d34e" />

---

## Step 3: Audit Source Bucket Object Count and Size

Before migrating, audit the source bucket to establish a baseline. This object count and total size will serve as the ground-truth for post-migration validation.

```bash
aws s3 ls s3://xfusion-s3-16544 --recursive --summarize | tail -n 2
```

**Output:**

```
Total Objects: 3349
  Total Size: 76182921
```

This confirms the source bucket holds **3,349 objects** with a combined size of approximately **72.7 MB** (~76.2 MB raw bytes).

> **Why this matters:** Capturing pre-migration baselines is a fundamental data integrity practice. If the post-migration count or size differs, it signals an incomplete transfer or an error during the sync.

📸 *Screenshot: Source bucket audit showing 3,349 objects and total size of 76,182,921 bytes*

<img width="1028" height="648" alt="Source bucket summarize showing 3349 objects and 76182921 bytes" src="https://github.com/user-attachments/assets/49a14bd2-4b55-48d0-be54-9e4fb0d8a44b" />

---

## Step 4: Inspect Source Bucket Contents

List the top-level contents of the source bucket to understand the directory structure before executing the sync. This ensures no unexpected prefixes or large folders are missed.

```bash
aws s3 ls s3://xfusion-s3-16544
```

**Output confirms the following structure:**

- `PRE wp-admin/` -- WordPress admin panel files
- `PRE wp-content/` -- Themes, plugins, and uploaded media
- `PRE wp-includes/` -- WordPress core library files
- Root-level PHP files: `index.php`, `wp-activate.php`, `wp-config-sample.php`, `wp-login.php`, `wp-settings.php`, and others

> **Operational insight:** Reviewing the bucket structure before syncing helps identify whether folder-level `--exclude` or `--include` flags are needed, and confirms the data is organized as expected.

📸 *Screenshot: Source bucket listing showing WordPress directory structure and root-level PHP files*

<img width="1026" height="828" alt="aws s3 ls output showing wp-admin, wp-content, wp-includes prefixes and root PHP files" src="https://github.com/user-attachments/assets/99237cf2-97d8-4081-86d1-929c97df4280" />

---

## Step 5: Confirm Destination Bucket is Empty

Verify the destination bucket contains zero objects before initiating the sync. This prevents any assumptions about pre-existing state and avoids overwriting data unexpectedly.

```bash
aws s3 ls s3://xfusion-sync-10728 --recursive --summarize | tail -n 2
```

**Output:**

```
Total Objects: 0
  Total Size: 0
```

This confirms the destination bucket is clean and ready to receive all objects from the source.

> **Risk note:** Syncing into a non-empty destination can produce unexpected results if object keys overlap. Always confirm a clean state before executing a full migration.

📸 *Screenshot: Destination bucket confirmed empty before migration begins*

<img width="1023" height="865" alt="Destination bucket showing 0 objects and 0 size before migration" src="https://github.com/user-attachments/assets/eca421ab-64eb-4be7-9643-ac847db4ed84" />

---

## Step 6: Sync Data from Source to Destination

Execute the migration using `aws s3 sync`. This command performs an incremental, checksum-aware copy, transferring only objects that do not already exist in the destination or that differ from the source.

```bash
aws s3 sync s3://xfusion-s3-16544 s3://xfusion-sync-10728
```

The CLI outputs each object as it is copied, following the pattern:

```
copy: s3://xfusion-s3-16544/<key> to s3://xfusion-sync-10728/<key>
```

All 3,349 objects across `wp-admin/`, `wp-content/`, `wp-includes/`, and the root are transferred in sequence.

> **Why `aws s3 sync` over `aws s3 cp --recursive`:**
> - `sync` skips objects already present in the destination, making it idempotent and safe to re-run
> - `sync` computes ETags (MD5-based checksums) to detect content changes
> - `sync` supports `--delete` to mirror deletions from source (not used here to avoid accidental removal)

📸 *Screenshot: Live sync output showing object-by-object transfer from source to destination*

<img width="1032" height="499" alt="aws s3 sync output showing copy operations for WordPress files across all prefixes" src="https://github.com/user-attachments/assets/c1122a7d-f7dd-4f4d-bc5b-a6055f2dd381" />

---

## Step 7: Validate Destination Bucket Contents Post-Migration

After the sync completes, perform a full recursive listing of the destination bucket to confirm all objects were transferred with correct paths and timestamps.

```bash
aws s3 ls s3://xfusion-sync-10728 --recursive
```

The output lists all 3,349 objects with timestamps matching the sync operation (e.g., `2026-02-24 00:21:47`), covering the full WordPress directory structure including deeply nested widget files under `wp-includes/widgets/`.

> **Validation logic:** Compare the object count and key structure against the source listing from Step 4. A match at both the summary level and key-by-key level confirms a complete, uncorrupted transfer.

📸 *Screenshot: Destination bucket recursive listing showing all migrated objects including `wp-includes/widgets/` subtree*

<img width="1000" height="857" alt="Full recursive listing of destination bucket confirming successful object migration" src="https://github.com/user-attachments/assets/104fbab7-db51-4bcd-8601-ebc27ac917fb" />

---

## Step 8: Verify Data Consistency with Dry-Run

Execute a second sync with the `--dryrun` flag against the same source and destination. A clean dry-run -- producing no copy operations -- confirms the destination is fully consistent with the source and no objects were missed or corrupted.

```bash
aws s3 sync s3://xfusion-s3-16544 s3://xfusion-sync-10728 --dryrun
```

**Expected output:** No copy or delete operations reported.

This is the definitive consistency check. `aws s3 sync` uses ETag comparison to determine whether any objects differ between buckets. An empty dry-run output means every object in the source exists in the destination with a matching checksum.

> **Edge case:** If the dry-run reports pending copies, it indicates either objects that failed to transfer silently or objects added to the source after the initial sync. Re-running the sync (without `--dryrun`) resolves both cases.

📸 *Screenshot: `--dryrun` output confirms zero pending operations, validating full consistency*

<img width="1025" height="335" alt="aws s3 sync --dryrun producing no output, confirming source and destination are fully consistent" src="https://github.com/user-attachments/assets/fae5892a-d31c-4285-a7db-98263a3e8aef" />

---

## Key Decisions

| Decision | Rationale |
|---|---|
| Used `aws s3 sync` instead of `aws s3 cp --recursive` | `sync` is idempotent, checksum-aware, and safe to re-run without duplication |
| Captured pre-migration baseline with `--summarize` | Enables post-migration object count comparison as a primary integrity signal |
| Confirmed empty destination before sync | Eliminates risk of unintended overwrite or partial state before migration begins |
| Used `--dryrun` as final validation step | Provides checksum-level consistency verification beyond simple object count matching |
| Specified `--region us-east-1` on bucket creation | Prevents region defaulting errors and ensures both buckets co-locate for low-latency, free intra-region transfer |

---

## Errors and Resolutions

**No errors were encountered during this migration.** The steps executed cleanly in sequence. The following are common failure modes in equivalent scenarios:

**`BucketAlreadyExists` on `aws s3 mb`**
S3 bucket names are globally unique across all AWS accounts. If the chosen name is taken, select a different name. Appending a random suffix or account-specific identifier is a common convention.

**`AccessDenied` on `aws s3 sync`**
Typically caused by missing `s3:GetObject` on the source bucket or `s3:PutObject` on the destination. Verify the IAM policy attached to the active user or role covers both buckets.

**Dry-run reports pending copies after sync**
Indicates one or more objects failed to transfer silently, or new objects were written to the source during the sync window. Re-run `aws s3 sync` without `--dryrun` to resolve.

**Object count mismatch post-sync**
If the destination object count is lower than the source baseline, check for AWS CLI version issues, network interruptions, or throttling on the destination bucket. Re-run the sync to fill gaps.

---

## Best Practices and Lessons Learned

- **Always capture a pre-migration baseline.** Object count and total size from `--summarize` serve as the primary integrity gate before and after migration.
- **Prefer `sync` over `cp --recursive` for any migration workload.** `sync` is idempotent, checksum-aware, and handles retries gracefully.
- **Validate with a dry-run, not just a count.** A matching object count does not guarantee content integrity. The dry-run ETag comparison is the definitive consistency check.
- **Confirm bucket emptiness before initiating sync.** Syncing into a bucket with pre-existing objects can cause confusion when validating the outcome.
- **For large-scale migrations (multi-GB or millions of objects),** consider enabling S3 Transfer Acceleration on the destination, using `--no-progress` to reduce CLI overhead, or scripting parallel prefix-based syncs.
- **For production data,** enable versioning on the destination bucket before migration. This creates a recoverable state if any post-migration operations accidentally overwrite objects.

---

## Validation Checklist

- Destination bucket created successfully in `us-east-1`
- Destination bucket is private (no public access)
- Pre-migration baseline captured: 3,349 objects, 76,182,921 bytes
- Destination confirmed empty before sync
- All 3,349 objects synced successfully with no errors
- Post-migration recursive listing confirms correct key structure and timestamps
- Dry-run validation returns zero pending operations

---

## Key AWS Services Used

| Service | Role |
|---|---|
| **Amazon S3** | Source and destination object storage |
| **AWS CLI** | All bucket provisioning, data transfer, and validation operations |
| **AWS IAM** | Identity verification and access control for all S3 operations |

---

## Outcome

A complete, production-ready S3 bucket migration was executed with zero data loss. All 3,349 WordPress objects totalling approximately 76 MB were transferred from `xfusion-s3-16544` to `xfusion-sync-10728` using `aws s3 sync`, with integrity confirmed via both post-migration listing and a final dry-run consistency check. The migration is fully repeatable and idempotent due to the checksum-aware sync mechanism.





















# AWS S3 Bucket Data Migration & Sync

## Project Overview

As part of a data migration initiative, this project demonstrates how to:
- Create a **new private S3 bucket**
- Migrate all data from an existing S3 bucket
- Verify **data consistency and integrity**
- Perform all operations using **AWS CLI**

This task is executed entirely in the **us-east-1** region.

---

## Prerequisites

- AWS CLI installed
- IAM user with:
  - S3 full access
- Configured AWS credentials
- Correct AWS region set

---

## Step 1: Verify AWS CLI Authentication

- `aws sts get-caller-identity`

📸 Screenshot:
<img width="1036" height="469" alt="image" src="https://github.com/user-attachments/assets/2ef857c9-3fdb-4f53-8984-e59dfb9d55b8" />

## Step 2: Create the Destination Bucket
- `aws s3 mb s3://xfusion-sync-10728 --region us-east-1`

📸 Screenshot:
<img width="1032" height="673" alt="image" src="https://github.com/user-attachments/assets/dbc26f33-dad8-4508-ba1e-3fb8c9d4d34e" />


## Step 3: Check Source Bucket
- `aws s3 ls s3://xfusion-s3-16544 --recursive --summarize | tail -n 2`

📸 Screenshot:
<img width="1028" height="648" alt="image" src="https://github.com/user-attachments/assets/49a14bd2-4b55-48d0-be54-9e4fb0d8a44b" />

## Step 4: Verify Source Bucket Contents
- `aws s3 ls s3://xfusion-s3-16544`

📸 Screenshot:

<img width="1026" height="828" alt="image" src="https://github.com/user-attachments/assets/99237cf2-97d8-4081-86d1-929c97df4280" />


## Step 5: Check Destination Bucket
-`aws s3 ls s3://xfusion-sync-10728 --recursive --summarize | tail -n 2`

📸 Screenshot:
<img width="1023" height="865" alt="image" src="https://github.com/user-attachments/assets/eca421ab-64eb-4be7-9643-ac847db4ed84" />

## Step 5: Sync Data from Source to Destination
- `aws s3 sync s3://xfusion-s3-16544 s3://xfusion-sync-10728`

📸 Screenshot:

<img width="1032" height="499" alt="image" src="https://github.com/user-attachments/assets/c1122a7d-f7dd-4f4d-bc5b-a6055f2dd381" />

## Step 6: Validate Destination Bucket Data
- `aws s3 ls s3://xfusion-sync-10728 --recursive`

📸 Screenshot:

<img width="1000" height="857" alt="image" src="https://github.com/user-attachments/assets/104fbab7-db51-4bcd-8601-ebc27ac917fb" />

## Step 7: Verify Data Consistency (Optional)
- `aws s3 sync s3://xfusion-s3-16544 s3://xfusion-sync-10728 --dryrun`

📸 Screenshot:

<img width="1025" height="335" alt="image" src="https://github.com/user-attachments/assets/fae5892a-d31c-4285-a7db-98263a3e8aef" />

## Validation Checklist

- Destination bucket created successfully

- Bucket is private

- All objects copied successfully

- Object count matches source

- No sync errors reported

## Key AWS Services Used

- Amazon Web Services

- Amazon S3

- AWS CLI

- IAM

## Outcome

- Successful migration of all objects
-  No data loss or corruption
- Data consistency verified
