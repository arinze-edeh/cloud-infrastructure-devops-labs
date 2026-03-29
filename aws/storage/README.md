# AWS Storage

![AWS](https://img.shields.io/badge/AWS-Storage-orange?style=flat-square&logo=amazonaws)
![CLI](https://img.shields.io/badge/Tool-AWS_CLI-blue?style=flat-square)
![Region](https://img.shields.io/badge/Region-us--east--1-green?style=flat-square)
![Projects](https://img.shields.io/badge/Projects-5-lightgrey?style=flat-square)

---

## Overview

This directory covers production-aligned AWS storage operations executed entirely via the AWS CLI. The work spans the full lifecycle of cloud storage management: provisioning and configuring S3 buckets for static hosting, migrating data between buckets, enabling versioning controls, expanding live EBS volumes without downtime, and creating point-in-time EBS snapshots.

Each project reflects a real operational scenario encountered in DevOps and platform engineering roles. Emphasis throughout is on CLI-driven repeatability, layered verification at each step, and zero-downtime execution where applicable.

---

## Directory Structure

```
aws/storage/
├── ebs-snapshots/
├── ec2-root-volume-expansion-and-filesystem-resize/
├── s3-bucket-migration-sync/
├── s3-static-website-hosting/
├── s3-versioning/
└── README.md
```

---

## Project Summaries

---

### [S3 Static Website Hosting](./s3-static-website-hosting/)

**Quick Summary:** Provisions and configures an S3 bucket to serve a public static website over HTTP using the AWS CLI, covering bucket creation through end-to-end verification.

**Purpose:** Deliver a publicly accessible information portal on AWS without managing compute infrastructure. The solution replaces a traditional web server with a fully serverless static hosting model.

**Approach:** Six sequential phases: bucket creation, public access unblocking, static website hosting enablement, bucket policy attachment, file upload with explicit content type, and HTTP response verification via `curl`. Each phase includes a verification command before proceeding, preventing silent misconfigurations.

**Key Decisions:**
- `--content-type "text/html"` set explicitly on upload to prevent browsers from triggering a file download instead of rendering
- Error document pointed to `index.html` as a safe fallback, avoiding broken 404 responses with no dedicated error page
- Noted that HTTPS requires CloudFront; the native S3 endpoint is HTTP-only

**Outcome:** Bucket `xfusion-web-27751` serving `Welcome to KKE labs!` with `HTTP/1.1 200 OK`, `ContentType: text/html`, and AES256 server-side encryption active.

---

### [EC2 Root Volume Expansion and Filesystem Resize](./ec2-root-volume-expansion-and-filesystem-resize/)

**Quick Summary:** Expands an EC2 instance root EBS volume from 8 GiB to 12 GiB with zero downtime, propagating the change through all three layers: AWS block device, partition table, and XFS filesystem.

**Purpose:** The `datacenter-ec2` instance was running critically low on storage. The volume needed to expand while the instance remained live and fully operational.

**Approach:** AWS-side expansion via `modify-volume`, followed by OS-level `growpart` to extend the partition boundary, then `xfs_growfs -d /` to expand the mounted XFS filesystem online. State was verified at every layer with `lsblk` and `df -hT` before and after each operation.

**Key Decisions:**
- Used `growpart` over manual `fdisk` to safely modify a live, mounted root partition
- Proceeded at `optimizing` state rather than waiting for `completed`, as storage is fully available at that stage
- GPT PMBR mismatch warnings from `fdisk` documented explicitly as expected, self-healing artifacts of the volume resize

**Outcome:** Root filesystem expanded from 8G to 12G (`11G` usable), usage dropped from 19% to 13%, with zero service interruption and no reboot required.

---

### [S3 Bucket Migration Sync](./s3-bucket-migration-sync/)

**Quick Summary:** Creates a new private S3 bucket and migrates all objects from an existing source bucket using `aws s3 sync`, with a dry-run pass to verify data consistency post-migration.

**Purpose:** Supports a data migration initiative requiring a clean destination bucket, full object transfer, and verified consistency before decommissioning the source.

**Approach:** Destination bucket provisioned with `s3 mb`, source enumerated with `--recursive --summarize`, sync executed with `aws s3 sync`, then a `--dryrun` re-run confirms no remaining delta between source and destination.

**Key Decisions:**
- `--dryrun` used as a consistency verification mechanism rather than a separate manual object count comparison
- Object counts and sizes validated before and after migration to detect any transfer gaps

**Outcome:** All objects from `xfusion-s3-16544` migrated to `xfusion-sync-10728` with no data loss, no sync errors, and consistency confirmed.

---

### [S3 Versioning](./s3-versioning/)

**Quick Summary:** Enables S3 bucket versioning on an existing bucket via the AWS CLI, adding object-level data protection against accidental deletions and overwrites.

**Purpose:** Improves storage resilience for a production-adjacent bucket by enabling historical version retention, a foundational requirement for compliance and recovery workflows.

**Approach:** Pre-change state captured with `get-bucket-versioning` (expected: empty output), versioning applied with `put-bucket-versioning Status=Enabled`, and post-change state confirmed to return `Status: Enabled`.

**Key Decisions:**
- Baseline state verified before applying changes to confirm versioning was not already enabled or in a suspended state
- Three-command workflow (check, apply, verify) kept deliberately minimal for auditability

**Outcome:** Bucket `nautilus-s3-9397` versioning enabled and confirmed. The bucket is now protected from single-operation data loss events.

---

### [EBS Snapshots](./ebs-snapshots/)

**Quick Summary:** Creates a named, described EBS snapshot for a production volume via the AWS Console, verifying successful completion before closing the task.

**Purpose:** Establishes an initial backup for a critical EBS volume (`devops-vol`) as part of the Nautilus DevOps team's backup strategy rollout.

**Approach:** Volume located via the EC2 Dashboard, snapshot initiated with name `devops-vol-ss` and description `devops Snapshot`, and snapshot status monitored until confirmed as `completed` in the Snapshots view.

**Key Decisions:**
- Named snapshot with a descriptive label to support audit trails and future restore identification
- Status explicitly verified as `completed` before closing, not assumed from initiation alone

**Outcome:** Snapshot `devops-vol-ss` created successfully with `completed` status, providing a verified point-in-time recovery baseline.

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Cloud Platform | AWS (S3, EC2, EBS) |
| CLI | AWS CLI v1.x / v2.x |
| OS Tools | `growpart`, `xfs_growfs`, `lsblk`, `df`, `fdisk`, `curl`, `chmod`, `ssh` |
| Authentication | AWS IAM, STS |
| Filesystem | XFS |
| Volume Type | EBS gp3 |
| Region | us-east-1 |

---

## Key Outcomes and Skills Demonstrated

- **Multi-layer systems thinking:** EBS volume expansion requires coordinated changes at the AWS API, partition table, and filesystem layers. Each is addressed independently and verified before the next step.
- **Zero-downtime operations:** Root volume expansion and filesystem resize completed on a live, running instance without reboot or service interruption.
- **CLI-first workflows:** All S3 provisioning, versioning, migration, and verification tasks executed entirely via AWS CLI, producing reproducible, scriptable runbooks.
- **Verification discipline:** Every phase includes an explicit verification command before and after the change. State is never assumed; it is confirmed.
- **Data protection controls:** Versioning, snapshots, and migration consistency checks reflect production-grade data durability thinking.
- **Troubleshooting documentation:** Known warnings (GPT PMBR mismatch, partition table ordering) are documented with root cause and resolution rather than left unexplained.

---

## How to Navigate

Each subdirectory contains a standalone `README.md` with full implementation detail, including pre-flight checks, phase-by-phase commands with expected outputs, screenshot placeholders, error sections, and a quick-reference cheatsheet where applicable.

Start with the project summary above for context, then open the individual project README for step-by-step reproduction.

For cross-project patterns (CLI verification loops, layered expansion, sync validation), compare the implementation sections across projects directly.

---
