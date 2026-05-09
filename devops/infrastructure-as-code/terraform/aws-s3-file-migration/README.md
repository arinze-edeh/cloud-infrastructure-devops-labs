# Terraform S3 Object Upload: File Migration to AWS S3 Using Infrastructure as Code

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture and Design Intent](#architecture-and-design-intent)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify the Source File](#step-1-verify-the-source-file)
  - [Step 2: Inspect the Working Directory](#step-2-inspect-the-working-directory)
  - [Step 3: Review the Provider Configuration](#step-3-review-the-provider-configuration)
  - [Step 4: Review the Existing main.tf](#step-4-review-the-existing-maintf)
  - [Step 5: Append the S3 Object Resource to main.tf](#step-5-append-the-s3-object-resource-to-maintf)
  - [Step 6: Verify the Updated main.tf](#step-6-verify-the-updated-maintf)
  - [Step 7: Run terraform plan](#step-7-run-terraform-plan)
  - [Step 8: Run terraform apply](#step-8-run-terraform-apply)
  - [Step 9: Verify the Uploaded Object via AWS CLI](#step-9-verify-the-uploaded-object-via-aws-cli)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Technologies Used](#technologies-used)

---

## Project Overview

The Nautilus DevOps team is executing an on-premises to cloud data migration, transferring files from local storage systems into AWS S3 buckets. In this implementation, a file located at `/tmp/devops.txt` on the infrastructure server is uploaded to an existing private S3 bucket named `devops-cp-3338` using Terraform as the provisioning tool.

The task mandates that the upload be declared exclusively within the existing `main.tf` file, without creating any additional `.tf` files, preserving the integrity of the existing IaC layout. The Terraform working directory is `/home/bob/terraform`, and the provider configuration is pre-established in `provider.tf` targeting a LocalStack-emulated AWS environment.

---

## Architecture and Design Intent

```
+-----------------------------+
|  IAC Server (Local)         |
|  /tmp/devops.txt            |
|  /home/bob/terraform/       |
|    provider.tf              |
|    main.tf                  |
+------------|----------------+
             |
             | Terraform (aws_s3_object)
             |
+------------|----------------+
|  LocalStack (http://aws:4566)|
|  S3 Bucket: devops-cp-3338  |
|    Key: devops.txt           |
+-----------------------------+
```

**Design Decisions:**

* The S3 bucket `devops-cp-3338` is pre-provisioned and its state is already tracked by Terraform. Only the object upload resource is added.
* The `etag` attribute is computed using the `filemd5()` Terraform built-in function, enabling idempotent drift detection. If the file changes on disk, Terraform will detect the MD5 mismatch and re-upload automatically.
* The `source` attribute points to the local file path on the machine executing `terraform apply`, which in this context is the IAC server.
* All resources target the LocalStack endpoint `http://aws:4566` to emulate AWS services in a sandboxed environment.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Terraform | v1.x installed and initialized in `/home/bob/terraform` |
| AWS Provider | hashicorp/aws version 5.91.0 (pinned in `.terraform.lock.hcl`) |
| LocalStack | Running and accessible at `http://aws:4566` |
| Source File | `/tmp/devops.txt` present on the IAC server (27 bytes) |
| S3 Bucket | `devops-cp-3338` already provisioned and tracked in `terraform.tfstate` |
| AWS CLI | Configured with LocalStack endpoint for post-apply verification |

---

## Repository Structure

```
/home/bob/terraform/
|-- .terraform/                  # Provider plugins and modules cache
|-- .terraform.lock.hcl          # Provider dependency lock file
|-- provider.tf                  # AWS provider configuration with LocalStack endpoints
|-- main.tf                      # Resource definitions (S3 bucket + S3 object)
|-- terraform.tfstate            # Current state tracking managed resources
|-- README.MD                    # Original task README
```

---

## Implementation Guide

### Step 1: Verify the Source File

Before modifying any Terraform configuration, confirm the source file exists on the IAC server and note its size for post-apply verification.

```bash
ls -lh /tmp/devops.txt
```

**Output:**

```
-rw-rw-r-- 1 bob bob 27 May  9 21:37 /tmp/devops.txt
```

The file exists, is 27 bytes, and has world-readable permissions. This confirms it is accessible to the Terraform process at runtime.

*Screenshot: Terminal output confirming /tmp/devops.txt exists with size 27 bytes*

<img width="518" height="320" alt="image" src="https://github.com/user-attachments/assets/6fddef98-fc56-4875-9826-62682a8bf006" />

---

### Step 2: Inspect the Working Directory

Confirm the state of the Terraform working directory before making any changes.

```bash
ls -la
```

**Output:**

```
total 40
drwxr-xr-x 1 bob bob 4096 May  9 21:37 .
drwxr-x--- 1 bob bob 4096 May  9 21:37 ..
drwxr-xr-x 3 bob bob 4096 May  9 21:37 .terraform
-rw-r--r-- 1 bob bob 1406 May  9 21:37 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  140 May  9 21:37 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-rw-r-- 1 bob bob 2714 May  9 21:37 terraform.tfstate
```

The presence of `.terraform/`, `.terraform.lock.hcl`, and `terraform.tfstate` confirms the workspace is already initialized and the S3 bucket is tracked in state.

*Screenshot: Working directory listing confirming initialized Terraform workspace*

<img width="524" height="239" alt="image" src="https://github.com/user-attachments/assets/15f36cf5-ecce-4973-8320-f9853a9809ce" />

---

### Step 3: Review the Provider Configuration

Inspect `provider.tf` to understand the provider setup and endpoint configuration before authoring any resource blocks.

```bash
cat provider.tf
```

**Output:**

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

Key observations:

* `skip_credentials_validation = true` and `skip_requesting_account_id = true` are required for LocalStack environments where no real AWS credentials are present.
* `s3_use_path_style = true` is mandatory for LocalStack S3 compatibility, as LocalStack does not support virtual-hosted style bucket addressing.
* All service endpoints, including `s3`, are redirected to `http://aws:4566`.

*Screenshot: provider.tf content confirming LocalStack endpoint configuration*

<img width="536" height="382" alt="image" src="https://github.com/user-attachments/assets/b39eb092-13e8-4dbc-a1fa-f59f80bad7e7" />

---

### Step 4: Review the Existing main.tf

Inspect the current `main.tf` to understand the existing resource declarations before appending the new resource.

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_s3_bucket" "my_bucket" {
  bucket = "devops-cp-3338"
  acl    = "private"

  tags = {
    Name = "devops-cp-3338"
  }
}
```

The bucket resource is already declared and its state is tracked. No new `provider {}` block is required. The new object resource will be appended directly to this file.

*Screenshot: Existing main.tf showing the pre-provisioned S3 bucket resource*

---

### Step 5: Append the S3 Object Resource to main.tf

Use a heredoc append to add the `aws_s3_object` resource to `main.tf` without creating a new file. This strictly satisfies the task requirement of modifying only `main.tf`.

```bash
cat >> main.tf << 'EOF'

resource "aws_s3_object" "devops_file" {
  bucket = "devops-cp-3338"
  key    = "devops.txt"
  source = "/tmp/devops.txt"
  etag   = filemd5("/tmp/devops.txt")
}
EOF
```

**Resource attribute breakdown:**

| Attribute | Value | Purpose |
|---|---|---|
| `bucket` | `devops-cp-3338` | Target S3 bucket name |
| `key` | `devops.txt` | Object key (path within the bucket) |
| `source` | `/tmp/devops.txt` | Local file path on the Terraform execution host |
| `etag` | `filemd5("/tmp/devops.txt")` | MD5 checksum for drift detection and idempotency |

*Screenshot: Terminal command appending the aws_s3_object resource block to main.tf*

---

### Step 6: Verify the Updated main.tf

Confirm the heredoc append was written correctly before executing the plan.

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_s3_bucket" "my_bucket" {
  bucket = "devops-cp-3338"
  acl    = "private"

  tags = {
    Name = "devops-cp-3338"
  }
}
resource "aws_s3_object" "devops_file" {
  bucket = "devops-cp-3338"
  key    = "devops.txt"
  source = "/tmp/devops.txt"
  etag   = filemd5("/tmp/devops.txt")
}
```

Both resource blocks are present and syntactically correct.

*Screenshot: Verified main.tf containing both the S3 bucket and S3 object resource blocks*

---

### Step 7: Run terraform plan

Execute a dry run to review the proposed changes before applying them.

```bash
terraform plan
```

**Output (summarized):**

```
aws_s3_bucket.my_bucket: Refreshing state... [id=devops-cp-3338]

Terraform used the selected providers to generate the following execution plan.

  # aws_s3_object.devops_file will be created
  + resource "aws_s3_object" "devops_file" {
      + bucket = "devops-cp-3338"
      + etag   = "628f77ec27c0e6eb1e0c6543cc3dd865"
      + key    = "devops.txt"
      + source = "/tmp/devops.txt"
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Warning: Argument is deprecated
  with aws_s3_bucket.my_bucket, on main.tf line 3
  acl is deprecated. Use the aws_s3_bucket_acl resource instead.
```

**Plan analysis:**

* Terraform correctly refreshed the existing bucket from state without attempting to recreate it.
* Only the new `aws_s3_object.devops_file` resource is planned for creation.
* The computed `etag` value `628f77ec27c0e6eb1e0c6543cc3dd865` confirms the file was read and its MD5 hash calculated at plan time.
* The deprecation warning for the `acl` argument is expected in AWS provider v5.x. The `acl` inline argument was deprecated in favor of the standalone `aws_s3_bucket_acl` resource. This is pre-existing in the bucket declaration and does not affect functionality in this LocalStack context.

*Screenshot: terraform plan output showing 1 resource to add with computed etag value*

---

### Step 8: Run terraform apply

Apply the plan to provision the S3 object in the LocalStack-emulated bucket.

```bash
terraform apply
```

When prompted, type `yes` to confirm.

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_s3_object.devops_file: Creating...
aws_s3_object.devops_file: Creation complete after 0s [id=devops.txt]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The apply completed immediately (0 seconds), which is consistent with LocalStack's in-memory S3 emulation behavior. The resource ID is set to the object key `devops.txt`.

*Screenshot: terraform apply confirming successful creation of aws_s3_object.devops_file*

---

### Step 9: Verify the Uploaded Object via AWS CLI

Confirm the object is accessible in the S3 bucket using the AWS CLI pointed at the LocalStack endpoint.

```bash
aws s3 ls s3://devops-cp-3338/ --endpoint-url http://aws:4566
```

**Output:**

```
2026-05-09 21:47:14         27 devops.txt
```

The object `devops.txt` is present in the bucket with a size of 27 bytes, matching the source file verified in Step 1. The timestamp confirms it was uploaded during the apply execution. The implementation is complete.

*Screenshot: AWS CLI listing confirming devops.txt (27 bytes) present in bucket devops-cp-3338*

---

## Errors and Resolutions

### Warning: `acl` Argument is Deprecated

**Observed:**
During both `terraform plan` and `terraform apply`, the following deprecation warning was emitted:

```
Warning: Argument is deprecated
  with aws_s3_bucket.my_bucket, on main.tf line 3, in resource "aws_s3_bucket" "my_bucket":
   3:   acl    = "private"
acl is deprecated. Use the aws_s3_bucket_acl resource instead.
```

**Root Cause:**
In AWS provider versions 4.x and above, the `acl` inline argument for `aws_s3_bucket` was deprecated. The recommended approach is to use a separate `aws_s3_bucket_acl` resource to manage bucket ACLs. The pre-existing `main.tf` uses the legacy inline `acl` syntax.

**Resolution:**
This warning does not cause plan or apply failure. Since the bucket resource was pre-provisioned and its `main.tf` declaration was provided as part of the task environment, no modification to the bucket block was made. The warning is acknowledged and does not affect the object upload workflow. In a production migration, the bucket declaration would be refactored to use `aws_s3_bucket_acl`.

---

## Best Practices Applied

* **Idempotency via etag:** The `etag = filemd5(...)` attribute ensures Terraform can detect file content changes on subsequent runs and re-upload only when the source file changes, avoiding unnecessary API calls.

* **Single-file modification discipline:** The task constraint of modifying only `main.tf` was honored precisely. No additional `.tf` files were created, preserving the workspace layout and reducing the risk of provider block duplication.

* **Plan before apply:** `terraform plan` was executed and reviewed before `terraform apply`, following the standard two-phase IaC workflow that prevents unintended resource modifications.

* **Post-apply verification:** An independent AWS CLI command was used to confirm the object presence in the bucket, providing out-of-band validation beyond Terraform's own state tracking.

* **Heredoc append pattern:** Using `cat >> main.tf << 'EOF' ... EOF` is a safe, atomic append pattern that avoids overwriting existing content and preserves the original resource block without requiring a text editor.

* **LocalStack path-style S3:** The `s3_use_path_style = true` provider setting is correctly configured, which is mandatory for LocalStack S3 compatibility and avoids DNS-based virtual-hosted routing failures.

---

## Lessons Learned

* **`filemd5()` is evaluated at plan time, not apply time.** The MD5 hash is computed when `terraform plan` runs, not when the apply executes. If the source file changes between plan and apply, the applied etag will reflect the state of the file at plan time, not at apply time. In high-frequency migration pipelines, always run plan and apply in immediate succession.

* **Terraform state and bucket pre-existence are separate concerns.** The bucket `devops-cp-3338` was already in `terraform.tfstate`. Had the bucket existed in LocalStack but not been tracked in state, Terraform would have attempted to create it and failed with a bucket-already-exists error. Verifying state alignment before appending dependent resources is a critical pre-step in any brownfield IaC task.

* **Deprecation warnings are noise until they are errors.** The `acl` deprecation warning in provider v5.x is present but non-blocking. However, in automated CI/CD pipelines where `terraform plan` output is parsed for success signals, unhandled deprecation warnings can be misread as errors. Teams should address deprecations proactively during provider upgrades rather than deferring them.

* **Object key is the resource ID in Terraform state.** After apply, the resource `aws_s3_object.devops_file` is tracked with `id=devops.txt` in `terraform.tfstate`. If the same key is uploaded with a different Terraform resource name, it will cause a conflict. Key naming should be treated as a stable identifier in S3-backed IaC.

* **LocalStack apply times are not production indicators.** The `Creation complete after 0s` timing reflects LocalStack's in-memory simulation, not real S3 API latency. Production uploads of large files will exhibit significantly different timing behavior and may require timeout configuration in Terraform.

---

## Technologies Used

| Technology | Version | Role |
|---|---|---|
| Terraform | 1.x | Infrastructure provisioning and state management |
| AWS Provider (hashicorp/aws) | 5.91.0 | AWS resource definitions |
| LocalStack | Latest | AWS service emulation (endpoint: http://aws:4566) |
| AWS CLI | v2 | Post-apply bucket object verification |
| Bash | 5.x | Heredoc append for main.tf modification |





<img width="524" height="316" alt="image" src="https://github.com/user-attachments/assets/d82abbf8-5c62-426b-813a-2e90c30932c4" />
<img width="524" height="366" alt="image" src="https://github.com/user-attachments/assets/42d82c43-66e0-4f68-b24c-0a809ff30b16" />
<img width="526" height="326" alt="image" src="https://github.com/user-attachments/assets/10fc0d83-b215-4f74-89b4-21465d0abec8" />
<img width="538" height="410" alt="image" src="https://github.com/user-attachments/assets/e73b66f6-3b00-4c82-9d86-5195da47e34a" />
<img width="538" height="410" alt="image" src="https://github.com/user-attachments/assets/0357f30c-a880-4360-85b2-07f936b3f480" />
<img width="524" height="255" alt="image" src="https://github.com/user-attachments/assets/315942fa-dc71-4ffb-a77f-a374a9c7da66" />
