# Terraform S3 Bucket Backup and Deletion via Local-Exec Provisioners

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Environment Inspection](#environment-inspection)
- [Implementation](#implementation)
  - [Step 1: Verify the S3 Bucket Contents](#step-1-verify-the-s3-bucket-contents)
  - [Step 2: Confirm the Local Backup Directory](#step-2-confirm-the-local-backup-directory)
  - [Step 3: Create the Backup Directory](#step-3-create-the-backup-directory)
  - [Step 4: Author the Terraform Configuration](#step-4-author-the-terraform-configuration)
  - [Step 5: Review the Final Configuration](#step-5-review-the-final-configuration)
  - [Step 6: Validate and Apply](#step-6-validate-and-apply)
  - [Step 7: Verify Backup Integrity](#step-7-verify-backup-integrity)
  - [Step 8: Confirm Bucket Deletion](#step-8-confirm-bucket-deletion)
- [Terraform Configuration Reference](#terraform-configuration-reference)
- [Execution Output Walkthrough](#execution-output-walkthrough)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Errors and Resolutions](#errors-and-resolutions)

---

## Overview

This implementation automates a controlled S3 bucket decommission workflow using Terraform's `terraform_data` resource with `local-exec` provisioners. The workflow copies all bucket contents to a local backup directory before permanently deleting the bucket, ensuring zero data loss during cleanup operations.

The Terraform working directory is `/home/bob/terraform`. All changes are applied exclusively to the existing `main.tf` file without creating additional `.tf` files, in line with the task constraint.

---

## Problem Statement

The Nautilus DevOps team is executing a cloud environment cleanup initiative. Several AWS S3 buckets were provisioned for one-time use during a migration process and are no longer required. These buckets must be decommissioned to reduce resource sprawl and optimize the AWS environment.

The specific requirement for this implementation:

* Back up all contents of the S3 bucket `datacenter-bck-14899` to the local directory `/opt/s3-backup/` on the `terraform-client` host
* Permanently delete the S3 bucket `datacenter-bck-14899` after backup confirmation
* Accomplish both operations exclusively through Terraform using AWS CLI commands embedded in `local-exec` provisioners
* Modify only the existing `main.tf` file in `/home/bob/terraform`

---

## Solution Architecture

```
terraform_data.s3_cleanup
        |
        |-- provisioner "local-exec" [1]
        |       aws s3 cp s3://datacenter-bck-14899 /opt/s3-backup/ --recursive
        |
        |-- provisioner "local-exec" [2]
                aws s3 rb s3://datacenter-bck-14899 --force
```

The `terraform_data` resource is a lifecycle-management primitive introduced in Terraform 1.4. It carries no infrastructure state of its own and is ideal for running side-effect operations such as shell commands via `local-exec`. Provisioners execute sequentially in declaration order, which guarantees the copy completes before the delete begins.

The AWS CLI interacts with a LocalStack endpoint (`http://aws:4566`) simulating real AWS S3 behavior in the sandbox environment.

---

## Prerequisites

| Requirement | Version / Detail |
|---|---|
| Terraform | v1.11.0 (used in this implementation) |
| AWS Provider | hashicorp/aws v5.91.0 |
| AWS CLI | Installed and accessible in shell PATH |
| LocalStack | Running at `http://aws:4566` |
| Working Directory | `/home/bob/terraform` |
| Backup Target Directory | `/opt/s3-backup/` |

---

## Environment Inspection

Before authoring any configuration, the environment was inspected to understand the existing file layout and provider setup.

### Terraform Directory Listing

```bash
bob@iac-server ~/terraform via default  ls -la
```

```
total 36
drwxr-xr-x 1 bob bob 4096 May  1 01:50 .
drwxr-x--- 1 bob bob 4096 May  1 01:50 ..
drwxr-xr-x 3 bob bob 4096 May  1 01:50 .terraform
-rw-r--r-- 1 bob bob 1406 May  1 01:50 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob   21 May  1 01:50 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

> Screenshot: Directory listing showing the pre-existing Terraform workspace contents including `.terraform/`, `main.tf`, and `provider.tf`

<img width="1047" height="712" alt="image" src="https://github.com/user-attachments/assets/099193b0-3748-4ac3-9129-6222cd9af9a1" />

The `.terraform/` directory confirms the workspace was already initialized. The lock file and provider configuration are present, so `terraform init` is not required.

### Initial main.tf State

```bash
cat main.tf
```

```hcl
# Add your code below
```
> Screenshot:


<img width="1045" height="653" alt="image" src="https://github.com/user-attachments/assets/efcdf9a4-77ae-41c8-85f8-a926750609dc" />

The file exists but contains only a comment placeholder, confirming no infrastructure resources have been defined yet.

### Provider Configuration

```bash
cat provider.tf
```

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.91.0"
    }
  }
}

provider "aws" {
  region                      = "us-east-1"
  skip_credentials_validation = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true

  endpoints {
    ec2            = "http://aws:4566"
    apigateway     = "http://aws:4566"
    cloudformation = "http://aws:4566"
    cloudwatch     = "http://aws:4566"
    dynamodb       = "http://aws:4566"
    es             = "http://aws:4566"
    firehose       = "http://aws:4566"
    iam            = "http://aws:4566"
    kinesis        = "http://aws:4566"
    lambda         = "http://aws:4566"
    route53        = "http://aws:4566"
    redshift       = "http://aws:4566"
    s3             = "http://aws:4566"
    secretsmanager = "http://aws:4566"
    ses            = "http://aws:4566"
    sns            = "http://aws:4566"
    sqs            = "http://aws:4566"
    ssm            = "http://aws:4566"
    stepfunctions  = "http://aws:4566"
    sts            = "http://aws:4566"
    rds            = "http://aws:4566"
  }
}
```

> Screenshot: Full contents of `provider.tf` showing the LocalStack endpoint overrides for all AWS services

<img width="1040" height="765" alt="image" src="https://github.com/user-attachments/assets/9e31224f-abda-43d8-9a8c-1dc2f440e96b" />

The `skip_credentials_validation = true` and `skip_requesting_account_id = true` flags are required for LocalStack compatibility. The `s3_use_path_style = true` flag forces path-style S3 addressing, which is necessary when using a custom endpoint.

### Terraform Version Confirmation

```bash
terraform version
```

```
Terraform v1.11.0
on linux_amd64
+ provider registry.terraform.io/hashicorp/aws v5.91.0
```

> Screenshot: Terraform version output confirming v1.11.0 is active alongside AWS provider v5.91.0

<img width="1045" height="619" alt="image" src="https://github.com/user-attachments/assets/b649b93e-ec22-4caa-b0e0-63b9463722ed" />

---

## Implementation

### Step 1: Verify the S3 Bucket Contents

Before initiating any backup or deletion, the bucket contents were verified to understand what data would be transferred.

```bash
aws s3 ls s3://datacenter-bck-14899 --endpoint-url http://aws:4566 --recursive
```

```
2026-05-01 01:50:50         27 datacenter.txt
```

> Screenshot: AWS CLI output confirming `datacenter.txt` (27 bytes) exists within the `datacenter-bck-14899` bucket

<img width="1043" height="668" alt="image" src="https://github.com/user-attachments/assets/998b4c05-ca3a-40ff-83cc-2c1a578e08f7" />

The bucket contains a single object: `datacenter.txt` at 27 bytes. This serves as the baseline for validating backup fidelity after the Terraform apply.

---

### Step 2: Confirm the Local Backup Directory

The target backup directory was inspected to confirm its initial state before any files were written.

```bash
ls -la /opt/s3-backup/
```

```
total 12
drwxr-xr-x 2 bob  bob  4096 May  1 01:50 .
drwxr-xr-x 1 root root 4096 May  1 01:50 ..
```

> Screenshot: Empty `/opt/s3-backup/` directory confirming no pre-existing backup files are present

<img width="1042" height="769" alt="image" src="https://github.com/user-attachments/assets/54e8e2d8-66f7-44c9-b83e-b635f9429ee4" />

The directory already exists and is empty, confirming it was pre-provisioned as the intended backup landing zone.

---

### Step 3: Create the Backup Directory

Although `/opt/s3-backup/` already existed from the environment setup, the directory creation command was run with `sudo mkdir -p` to ensure idempotency. The `-p` flag prevents errors if the directory already exists and creates any missing parent directories.

```bash
sudo mkdir -p /opt/s3-backup/
```

> Screenshot: Terminal prompt after successful silent execution of `sudo mkdir -p /opt/s3-backup/` with no error output, confirming idempotent directory creation

<img width="1048" height="376" alt="image" src="https://github.com/user-attachments/assets/f2e18a9c-ce52-4077-bb5f-c22e57fe5a3b" />

---

### Step 4: Author the Terraform Configuration

The `main.tf` file was written using a heredoc redirect. This approach writes the complete configuration atomically in a single shell operation, avoiding partial writes.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
# Add your code below

resource "terraform_data" "s3_cleanup" {

  provisioner "local-exec" {
    command = "aws s3 cp s3://datacenter-bck-14899 /opt/s3-backup/ --recursive --endpoint-url http://aws:4566"
  }

  provisioner "local-exec" {
    command = "aws s3 rb s3://datacenter-bck-14899 --force --endpoint-url http://aws:4566"
  }
}
EOF
```

> Screenshot: Terminal showing the heredoc write command executed against `/home/bob/terraform/main.tf`

<img width="1045" height="732" alt="image" src="https://github.com/user-attachments/assets/d6e7b0f3-6054-4aa2-a870-ec7383952c41" />

**Configuration Design Rationale:**

* **`terraform_data`** is used instead of a `null_resource` because `terraform_data` is a first-class Terraform built-in available since v1.4, requiring no external provider. It is the current best practice for triggering side-effect operations.
* **Two sequential `local-exec` provisioners** enforce execution ordering: the copy must complete before the delete is issued. Terraform guarantees provisioners on the same resource execute in declaration order.
* **`--recursive`** on `aws s3 cp` ensures all objects in the bucket, including those within simulated folder prefixes, are transferred.
* **`--force`** on `aws s3 rb` deletes all remaining objects before removing the bucket, providing a safety fallback if the copy provisioner partially succeeded.
* **`--endpoint-url http://aws:4566`** routes all AWS CLI calls to the LocalStack instance.

---

### Step 5: Review the Final Configuration

After writing the file, its contents were confirmed to match the intended configuration exactly.

```bash
cat main.tf
```

```hcl
# Add your code below

resource "terraform_data" "s3_cleanup" {

  provisioner "local-exec" {
    command = "aws s3 cp s3://datacenter-bck-14899 /opt/s3-backup/ --recursive --endpoint-url http://aws:4566"
  }

  provisioner "local-exec" {
    command = "aws s3 rb s3://datacenter-bck-14899 --force --endpoint-url http://aws:4566"
  }
}
```

> Screenshot: `cat main.tf` output confirming the complete resource block is written correctly with both provisioners in the correct order

<img width="1045" height="732" alt="image" src="https://github.com/user-attachments/assets/d6e7b0f3-6054-4aa2-a870-ec7383952c41" />

---

### Step 6: Validate and Apply

The configuration was validated for syntax correctness and then applied non-interactively.

```bash
terraform validate && terraform apply -auto-approve
```

```
Success! The configuration is valid.


Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # terraform_data.s3_cleanup will be created
  + resource "terraform_data" "s3_cleanup" {
      + id = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
terraform_data.s3_cleanup: Creating...
terraform_data.s3_cleanup: Provisioning with 'local-exec'...
terraform_data.s3_cleanup (local-exec): Executing: ["/bin/sh" "-c" "aws s3 cp s3://datacenter-bck-14899 /opt/s3-backup/ --recursive --endpoint-url http://aws:4566"]
terraform_data.s3_cleanup (local-exec): Completed 27 Bytes/27 Bytes (5.0 KiB/s) with 1 file(s) remaining
terraform_data.s3_cleanup (local-exec): download: s3://datacenter-bck-14899/datacenter.txt to ../../../opt/s3-backup/datacenter.txt
terraform_data.s3_cleanup: Provisioning with 'local-exec'...
terraform_data.s3_cleanup (local-exec): Executing: ["/bin/sh" "-c" "aws s3 rb s3://datacenter-bck-14899 --force --endpoint-url http://aws:4566"]
terraform_data.s3_cleanup (local-exec): delete: s3://datacenter-bck-14899/datacenter.txt
terraform_data.s3_cleanup (local-exec): remove_bucket: datacenter-bck-14899
terraform_data.s3_cleanup: Creation complete after 0s [id=825d56e0-f309-5923-d1b7-de51284faf97]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> Screenshot: Full `terraform validate && terraform apply -auto-approve` output showing successful plan, sequential provisioner execution, file download confirmation, bucket deletion, and final `Apply complete!` summary

<img width="1045" height="775" alt="image" src="https://github.com/user-attachments/assets/9035cf06-5b6d-4c5e-95a8-23e9891502ff" />

**Apply Execution Breakdown:**

| Phase | Action | Result |
|---|---|---|
| Validate | Syntax check on `main.tf` | Success |
| Plan | Resource diff generation | 1 resource to add |
| Provisioner 1 | `aws s3 cp` recursive copy | 27 bytes transferred, `datacenter.txt` downloaded |
| Provisioner 2 | `aws s3 rb --force` | `datacenter.txt` deleted, bucket removed |
| State write | Resource ID assigned | `825d56e0-f309-5923-d1b7-de51284faf97` |

---

### Step 7: Verify Backup Integrity

After the apply completed, the backup directory was inspected to confirm `datacenter.txt` was successfully transferred.

```bash
ls -la /opt/s3-backup/
```

```
total 16
drwxr-xr-x 2 bob  bob  4096 May  1 02:15 .
drwxr-xr-x 1 root root 4096 May  1 01:50 ..
-rw-r--r-- 1 bob  bob    27 May  1 01:50 datacenter.txt
```

> Screenshot: `/opt/s3-backup/` directory listing confirming `datacenter.txt` at 27 bytes is present, owned by `bob`, with the original S3 modification timestamp preserved

<img width="1052" height="709" alt="image" src="https://github.com/user-attachments/assets/321c46c6-19f4-471b-a266-f902014edd61" />

The file size matches the original S3 object (27 bytes), confirming a complete and uncorrupted transfer.

---

### Step 8: Confirm Bucket Deletion

A final AWS CLI check was executed against the now-deleted bucket to confirm it no longer exists.

```bash
aws s3 ls s3://datacenter-bck-14899 --endpoint-url http://aws:4566
```

```
An error occurred (NoSuchBucket) when calling the ListObjectsV2 operation: The specified bucket does not exist
```

> Screenshot: AWS CLI error response confirming `NoSuchBucket` for `datacenter-bck-14899`, verifying permanent deletion

<img width="1048" height="766" alt="image" src="https://github.com/user-attachments/assets/dce33e54-15d7-4cda-9547-42e335d14229" />

The `NoSuchBucket` error is the expected and desired outcome, confirming the bucket has been fully decommissioned.

---

## Terraform Configuration Reference

**File:** `/home/bob/terraform/main.tf`

```hcl
# Add your code below

resource "terraform_data" "s3_cleanup" {

  provisioner "local-exec" {
    command = "aws s3 cp s3://datacenter-bck-14899 /opt/s3-backup/ --recursive --endpoint-url http://aws:4566"
  }

  provisioner "local-exec" {
    command = "aws s3 rb s3://datacenter-bck-14899 --force --endpoint-url http://aws:4566"
  }
}
```

**File:** `/home/bob/terraform/provider.tf` (unchanged, pre-existing)

The provider configuration was not modified. It establishes the AWS provider v5.91.0 targeting LocalStack at `http://aws:4566` for all service endpoints.

---

## Execution Output Walkthrough

The sequential provisioner execution visible in the Terraform apply output confirms the intended operational ordering:

1. **Provisioner 1 invocation:** Terraform shells out to `/bin/sh` with the `aws s3 cp` command
2. **Transfer progress:** `Completed 27 Bytes/27 Bytes (5.0 KiB/s) with 1 file(s) remaining` indicates real-time progress streaming from the AWS CLI
3. **Download confirmation:** `download: s3://datacenter-bck-14899/datacenter.txt to ../../../opt/s3-backup/datacenter.txt` confirms object-level transfer success
4. **Provisioner 2 invocation:** Terraform immediately proceeds to the second `local-exec` after Provisioner 1 exits with code 0
5. **Object deletion:** `delete: s3://datacenter-bck-14899/datacenter.txt` confirms the `--force` flag triggered per-object deletion
6. **Bucket removal:** `remove_bucket: datacenter-bck-14899` confirms the empty bucket was then removed
7. **Resource creation complete:** Terraform assigns a UUID to the `terraform_data` resource and writes it to state

---

## Best Practices Applied

* **Sequential provisioner ordering:** Declaring the copy provisioner before the delete provisioner in the same resource block guarantees data safety. Terraform executes provisioners in declaration order with no parallelism within a single resource.

* **`--force` flag on bucket deletion:** The `aws s3 rb --force` command handles any objects remaining in the bucket before attempting removal. This acts as a defensive fallback in case the copy provisioner succeeded only partially, preventing a bucket deletion failure from leaving the Terraform apply in an unresolvable error state.

* **Pre-apply bucket inspection:** Running `aws s3 ls` before writing any Terraform configuration established a verified baseline of bucket contents. This ensures post-apply verification can confirm 100% transfer fidelity rather than relying on assumptions.

* **Directory idempotency with `mkdir -p`:** Using `sudo mkdir -p /opt/s3-backup/` ensures the operation succeeds regardless of whether the directory pre-exists, making the setup step safe to repeat in automated or pipeline contexts.

* **Heredoc write for configuration files:** Writing `main.tf` via heredoc (`cat > file << 'EOF'`) provides atomic, complete file writes in a single shell operation. Single-quoting the `EOF` delimiter (`'EOF'`) prevents the shell from interpolating any dollar signs or backticks within the Terraform configuration, which is critical when the file contains string literals.

* **`terraform validate` before apply:** Running validation as a precondition gate prevents malformed configurations from reaching the apply phase. Chaining it with `&&` ensures the apply only executes if validation passes.

* **Endpoint URL consistency:** All AWS CLI commands in provisioners explicitly specify `--endpoint-url http://aws:4566`, matching the provider configuration. This prevents command failures if the system's default AWS CLI configuration points to a different region or endpoint.

* **`terraform_data` over `null_resource`:** Using the `terraform_data` built-in avoids a dependency on the `hashicorp/null` provider, reducing the provider footprint and using the current Terraform-recommended approach for side-effect-only resources.

---

## Lessons Learned

* **Provisioner exit code determines apply success.** If either `local-exec` provisioner exits with a non-zero code, Terraform marks the resource as tainted and the overall apply fails. This means the AWS CLI commands must succeed, making pre-flight checks (bucket existence, directory writability) important before running the apply in production environments.

* **`terraform_data` resources are not idempotent by default.** Unlike managed infrastructure resources, a `terraform_data` resource with only `local-exec` provisioners will not re-run its provisioners on subsequent applies unless the resource is explicitly replaced (via `terraform taint` or a `replace` trigger). In this workflow that is the correct behavior since the bucket no longer exists after the first apply.

* **The `--recursive` flag is required for directory-level S3 copies.** Running `aws s3 cp` against a bucket path without `--recursive` fails silently or errors, as the CLI treats the source as a single object reference rather than a prefix. Explicitly specifying `--recursive` ensures all objects under any key prefix are included.

* **LocalStack endpoint routing must be explicit in shell commands.** The Terraform provider configuration routes Terraform-managed API calls through `http://aws:4566`, but shell commands invoked by `local-exec` run as independent processes with their own AWS CLI configuration. Without `--endpoint-url http://aws:4566` in each command, those CLI calls would target real AWS endpoints and fail in the sandbox environment.

* **File modification timestamps from S3 are preserved on download.** The downloaded `datacenter.txt` retains its original S3 timestamp (`May 1 01:50`), which is visible in the post-apply `ls -la` output. This behavior is useful for auditing and confirms that the AWS CLI performs a true content transfer rather than a metadata-only operation.

---

## Errors and Resolutions

### `NoSuchBucket` on Post-Apply Verification

**Command executed:**
```bash
aws s3 ls s3://datacenter-bck-14899 --endpoint-url http://aws:4566
```

**Error message:**
```
An error occurred (NoSuchBucket) when calling the ListObjectsV2 operation: The specified bucket does not exist
```

**Root cause and context:** This error appears after a successful apply and is intentional. The second `local-exec` provisioner explicitly removes the bucket with `aws s3 rb --force`. The post-apply `aws s3 ls` command is a verification step to confirm the bucket no longer exists, and `NoSuchBucket` is the exact expected response from LocalStack (and real AWS) when a bucket has been deleted.

**Resolution:** No action required. This error confirms successful completion of the decommission workflow. The shell prompt shows a non-zero exit indicator (`✖`) for this command, which is normal behavior for a verification step that relies on a failure response as its success signal.

---

*Implementation executed on terraform-client host | Terraform v1.11.0 | AWS Provider v5.91.0 | LocalStack endpoint http://aws:4566*
