# Terraform AWS S3 Public Bucket Provisioning

> Provisioning a publicly accessible AWS S3 bucket using Terraform and LocalStack, with ACL-based public read access enforced through infrastructure-as-code.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Solution Overview](#2-solution-overview)
3. [Architecture](#3-architecture)
4. [Prerequisites](#4-prerequisites)
5. [Implementation Guide](#5-implementation-guide)
   - [Step 1: Verify Working Directory and Existing Configuration](#step-1-verify-working-directory-and-existing-configuration)
   - [Step 2: Review the Provider Configuration](#step-2-review-the-provider-configuration)
   - [Step 3: Author the Main Terraform Configuration](#step-3-author-the-main-terraform-configuration)
   - [Step 4: Initialize the Terraform Working Directory](#step-4-initialize-the-terraform-working-directory)
   - [Step 5: Validate the Configuration](#step-5-validate-the-configuration)
   - [Step 6: Generate and Review the Execution Plan](#step-6-generate-and-review-the-execution-plan)
   - [Step 7: Apply the Configuration](#step-7-apply-the-configuration)
   - [Step 8: Verify Bucket Creation and ACL](#step-8-verify-bucket-creation-and-acl)
6. [Resource Reference](#6-resource-reference)
7. [Best Practices Applied](#7-best-practices-applied)
8. [Lessons Learned](#8-lessons-learned)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Problem Statement

The Nautilus DevOps team is actively migrating data storage to AWS S3 as part of a broader infrastructure consolidation initiative. During this migration, the team requires both private and public S3 buckets to store relevant data. As part of this effort, a publicly readable S3 bucket must be created and configured via Terraform to ensure reproducibility, auditability, and consistency across environments.

**Requirements:**

* Create a public S3 bucket named `devops-s3-9256` using Terraform
* Ensure the bucket is publicly accessible via ACL (`public-read`) once created
* All resources must be defined exclusively in `/home/bob/terraform/main.tf`
* The target AWS region is `us-east-1`
* The infrastructure is provisioned against a LocalStack endpoint (`http://aws:4566`)

---

## 2. Solution Overview

This implementation uses Terraform to provision four tightly coupled AWS S3 resources that together establish a fully public-read S3 bucket:

1. **`aws_s3_bucket`** - Creates the bucket with the specified name
2. **`aws_s3_bucket_ownership_controls`** - Sets object ownership to `BucketOwnerPreferred`, enabling ACL-based access control
3. **`aws_s3_bucket_public_access_block`** - Explicitly disables all public access block settings to allow public ACLs
4. **`aws_s3_bucket_acl`** - Applies the `public-read` ACL, granting read access to all users (`AllUsers` group)

The `aws_s3_bucket_acl` resource uses `depends_on` to explicitly enforce creation ordering, ensuring the ownership controls and public access block settings are applied before the ACL is set.

---

## 3. Architecture

```
LocalStack (http://aws:4566)
          |
          v
  +------------------+
  |  aws_s3_bucket   |
  |  devops-s3-9256  |
  +------------------+
          |
          +------------------------------+
          |                              |
          v                              v
+-------------------------+   +---------------------------+
| aws_s3_bucket_ownership |   | aws_s3_bucket_public_     |
| _controls               |   | access_block              |
| BucketOwnerPreferred    |   | All block flags = false   |
+-------------------------+   +---------------------------+
          |                              |
          +------------------------------+
                        |
                        v (depends_on both above)
              +---------------------+
              | aws_s3_bucket_acl   |
              |  ACL: public-read   |
              +---------------------+
```

**Resulting Grants:**

| Grantee | Permission |
|---|---|
| Bucket Owner (`webfile`) | `FULL_CONTROL` |
| `AllUsers` (public group) | `READ` |

---

## 4. Prerequisites

| Requirement | Details |
|---|---|
| Terraform | >= v1.x (provider lock file references `5.91.0`) |
| AWS Provider | `hashicorp/aws` version `5.91.0` |
| LocalStack | Running and accessible at `http://aws:4566` |
| AWS CLI | Configured and available (for post-apply verification) |
| OS User | `bob` with write access to `/home/bob/terraform/` |

---

## 5. Implementation Guide

### Step 1: Verify Working Directory and Existing Configuration

Begin by confirming the working directory and reviewing existing files.

```bash
pwd
# Expected: /home/bob/terraform

ls -la
# Expected output: README.MD and provider.tf present
```

*Screenshot: Terminal showing `/home/bob/terraform` directory listing with `README.MD` and `provider.tf`*

<img width="1036" height="463" alt="image" src="https://github.com/user-attachments/assets/df2fe68e-fead-4f15-a572-4dde5d0814c1" />

---

### Step 2: Review the Provider Configuration

Inspect the existing `provider.tf` to understand the AWS provider version and LocalStack endpoint overrides in use.

```bash
cat /home/bob/terraform/provider.tf
```

**Key observations from `provider.tf`:**

* AWS provider version is pinned to `5.91.0` via `required_providers`
* The `region` is set to `us-east-1`
* `skip_credentials_validation = true` and `skip_requesting_account_id = true` are set to accommodate LocalStack, which does not require real AWS credentials
* `s3_use_path_style = true` is required for LocalStack S3 compatibility
* All major AWS service endpoints are overridden to `http://aws:4566`, the LocalStack endpoint

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
    s3             = "http://aws:4566"
    iam            = "http://aws:4566"
    # ... (all services pointing to LocalStack)
  }
}
```

*Screenshot: Terminal output of `cat /home/bob/terraform/provider.tf` showing the full provider block*

<img width="1071" height="802" alt="image" src="https://github.com/user-attachments/assets/60c88f57-2d20-443f-a146-9a6d13420fc6" />

---

### Step 3: Author the Main Terraform Configuration

Create the `main.tf` file with all four required S3 resources. This file must be created at `/home/bob/terraform/main.tf` and must not be a separate `.tf` file.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_s3_bucket" "devops_bucket" {
  bucket = "devops-s3-9256"
}

resource "aws_s3_bucket_ownership_controls" "devops_bucket_ownership" {
  bucket = aws_s3_bucket.devops_bucket.id

  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_public_access_block" "devops_bucket_public_access" {
  bucket = aws_s3_bucket.devops_bucket.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_acl" "devops_bucket_acl" {
  depends_on = [
    aws_s3_bucket_ownership_controls.devops_bucket_ownership,
    aws_s3_bucket_public_access_block.devops_bucket_public_access,
  ]

  bucket = aws_s3_bucket.devops_bucket.id
  acl    = "public-read"
}
EOF
```

Confirm the file was written correctly:

```bash
cat /home/bob/terraform/main.tf
```

*Screenshots: Terminal output of `cat /home/bob/terraform/main.tf` showing all four resource blocks*

<img width="1011" height="461" alt="image" src="https://github.com/user-attachments/assets/176d9b08-cb65-4acb-8c41-b8e762c39ac5" />
<img width="1037" height="732" alt="image" src="https://github.com/user-attachments/assets/b93591e9-24f8-47f3-828a-f61b1b991222" />

**Resource breakdown:**

| Resource | Purpose |
|---|---|
| `aws_s3_bucket.devops_bucket` | Creates the S3 bucket named `devops-s3-9256` |
| `aws_s3_bucket_ownership_controls.devops_bucket_ownership` | Sets `BucketOwnerPreferred` to enable ACL support |
| `aws_s3_bucket_public_access_block.devops_bucket_public_access` | Disables all public access block restrictions |
| `aws_s3_bucket_acl.devops_bucket_acl` | Applies `public-read` ACL; depends on both above resources |

---

### Step 4: Initialize the Terraform Working Directory

Run `terraform init` to download the required provider plugin and create the `.terraform` lock file.

```bash
cd /home/bob/terraform && terraform init
```

**Expected output highlights:**

```
- Finding hashicorp/aws versions matching "5.91.0"...
- Installing hashicorp/aws v5.91.0...
- Installed hashicorp/aws v5.91.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

Terraform creates `.terraform.lock.hcl` to pin the provider version. This file should be committed to version control to guarantee consistent provider selections across team members and CI/CD pipelines.

*Screenshot: Terminal output of `terraform init` showing successful provider installation and initialization message*

<img width="1034" height="728" alt="image" src="https://github.com/user-attachments/assets/f8645e69-f64c-4eaf-a235-b56524107438" />

---

### Step 5: Validate the Configuration

Run `terraform validate` to check the configuration for syntax correctness and internal consistency before planning.

```bash
terraform validate
```

**Expected output:**

```
Success! The configuration is valid.
```

*Screenshot: Terminal output of `terraform validate` returning `Success! The configuration is valid.`*

---

### Step 6: Generate and Review the Execution Plan

Run `terraform plan` to preview the four resources Terraform will create. This is a dry-run and makes no changes to infrastructure.

```bash
terraform plan
```

**Expected plan summary:**

```
Plan: 4 to add, 0 to change, 0 to destroy.
```

The plan confirms the following resources will be created:

* `aws_s3_bucket.devops_bucket` with `bucket = "devops-s3-9256"`
* `aws_s3_bucket_acl.devops_bucket_acl` with `acl = "public-read"`
* `aws_s3_bucket_ownership_controls.devops_bucket_ownership` with `object_ownership = "BucketOwnerPreferred"`
* `aws_s3_bucket_public_access_block.devops_bucket_public_access` with all block flags set to `false`

*Screenshot: Terminal output of `terraform plan` showing all four resources scheduled for creation and the `Plan: 4 to add` summary line*

> **Note:** The plan output shows many attributes as `(known after apply)` because LocalStack assigns them dynamically at creation time.

---

### Step 7: Apply the Configuration

Apply the Terraform plan to provision all four resources against the LocalStack endpoint. The `-auto-approve` flag bypasses the interactive confirmation prompt.

```bash
terraform apply -auto-approve
```

**Expected output highlights:**

```
aws_s3_bucket.devops_bucket: Creating...
aws_s3_bucket.devops_bucket: Creation complete after 0s [id=devops-s3-9256]
aws_s3_bucket_ownership_controls.devops_bucket_ownership: Creating...
aws_s3_bucket_public_access_block.devops_bucket_public_access: Creating...
aws_s3_bucket_ownership_controls.devops_bucket_ownership: Creation complete after 0s [id=devops-s3-9256]
aws_s3_bucket_public_access_block.devops_bucket_public_access: Creation complete after 0s [id=devops-s3-9256]
aws_s3_bucket_acl.devops_bucket_acl: Creating...
aws_s3_bucket_acl.devops_bucket_acl: Creation complete after 0s [id=devops-s3-9256,public-read]

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

The creation order confirms Terraform's dependency graph was resolved correctly: the bucket is created first, then the ownership controls and public access block are created in parallel, and finally the ACL is applied last.

*Screenshot: Terminal output of `terraform apply -auto-approve` showing all four resources created successfully with the `Apply complete! Resources: 4 added` summary line*

---

### Step 8: Verify Bucket Creation and ACL

Use the AWS CLI with the LocalStack endpoint to confirm the bucket exists and has the correct ACL and public access block configuration.

**Verify the bucket exists:**

```bash
aws --endpoint-url=http://aws:4566 s3 ls | grep devops-s3-9256
```

**Expected output:**

```
2026-04-14 02:15:27 devops-s3-9256
```

*Screenshot: Terminal showing `aws s3 ls` output with `devops-s3-9256` present*

---

**Verify the bucket ACL:**

```bash
aws --endpoint-url=http://aws:4566 s3api get-bucket-acl --bucket devops-s3-9256
```

**Expected output:**

```json
{
    "Owner": {
        "DisplayName": "webfile",
        "ID": "75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a"
    },
    "Grants": [
        {
            "Grantee": {
                "DisplayName": "webfile",
                "ID": "75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a",
                "Type": "CanonicalUser"
            },
            "Permission": "FULL_CONTROL"
        },
        {
            "Grantee": {
                "Type": "Group",
                "URI": "http://acs.amazonaws.com/groups/global/AllUsers"
            },
            "Permission": "READ"
        }
    ]
}
```

The presence of the `AllUsers` grantee with `READ` permission confirms the bucket is publicly readable.

*Screenshot: Terminal output of `aws s3api get-bucket-acl` showing the `AllUsers` group grantee with `READ` permission*

---

**Verify the public access block configuration:**

```bash
aws --endpoint-url=http://aws:4566 s3api get-public-access-block --bucket devops-s3-9256
```

**Expected output:**

```json
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": false,
        "IgnorePublicAcls": false,
        "BlockPublicPolicy": false,
        "RestrictPublicBuckets": false
    }
}
```

All four public access block flags are `false`, confirming the bucket is not guarded by any AWS-level access restrictions.

*Screenshot: Terminal output of `aws s3api get-public-access-block` confirming all block flags are set to `false`*

---

## 6. Resource Reference

### `main.tf` (complete)

```hcl
resource "aws_s3_bucket" "devops_bucket" {
  bucket = "devops-s3-9256"
}

resource "aws_s3_bucket_ownership_controls" "devops_bucket_ownership" {
  bucket = aws_s3_bucket.devops_bucket.id

  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_public_access_block" "devops_bucket_public_access" {
  bucket = aws_s3_bucket.devops_bucket.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_acl" "devops_bucket_acl" {
  depends_on = [
    aws_s3_bucket_ownership_controls.devops_bucket_ownership,
    aws_s3_bucket_public_access_block.devops_bucket_public_access,
  ]

  bucket = aws_s3_bucket.devops_bucket.id
  acl    = "public-read"
}
```

---

## 7. Best Practices Applied

**Explicit `depends_on` for ACL ordering**
The `aws_s3_bucket_acl` resource uses an explicit `depends_on` to enforce that ownership controls and public access block settings are applied before the ACL. Without this, Terraform may attempt to set the ACL before the ownership model supports it, causing intermittent apply failures in real AWS environments.

**Separation of concerns across distinct resources**
Rather than using deprecated inline `acl` and `grant` arguments on the `aws_s3_bucket` resource, ownership, public access, and ACL settings are managed as separate resources. This aligns with the AWS provider's direction since version `4.x` and improves auditability.

**Provider version pinning**
The AWS provider is pinned to an exact version (`5.91.0`) using `required_providers`. Combined with the generated `.terraform.lock.hcl` file, this guarantees reproducible applies across environments and prevents unintentional provider upgrades from breaking infrastructure.

**LocalStack endpoint isolation**
By routing all AWS API calls to `http://aws:4566`, the lab environment is fully isolated from real AWS accounts. This pattern mirrors how teams use LocalStack in CI pipelines to test Terraform changes without incurring costs or risking production environments.

**Validation before apply**
Running `terraform validate` before `terraform plan` and `terraform apply` catches syntax and schema errors early, reducing feedback cycle time and preventing partial applies caused by configuration errors discovered mid-run.

**Use of `s3_use_path_style`**
Setting `s3_use_path_style = true` in the provider block is mandatory when targeting LocalStack or any S3-compatible endpoint that does not support virtual-hosted-style bucket URLs. This is a common misconfiguration that silently breaks S3 operations against non-AWS endpoints.

---

## 8. Lessons Learned

**ACL application requires explicit resource dependency sequencing**
In AWS provider versions `4.x` and above, setting a bucket ACL requires that `BucketOwnerPreferred` ownership is configured and that public access block settings permit public ACLs before the ACL resource is applied. Without `depends_on`, Terraform's parallel resource graph may create the ACL resource before these prerequisites are satisfied, resulting in `AccessControlListNotSupported` errors in real AWS environments.

**`block_public_acls = false` is a necessary prerequisite for `public-read`**
Even with the correct ownership setting, if `BlockPublicAcls` remains `true` (which is AWS's default for new buckets), attempting to set `public-read` will fail silently or be overridden. The `aws_s3_bucket_public_access_block` resource must explicitly set all four flags to `false` before the ACL can take effect.

**LocalStack behavior may differ from real AWS**
LocalStack generally mimics AWS API behavior but may not enforce all constraints identically. In real AWS environments, incorrect resource ordering or missing public access block configuration will result in hard errors. Always test the same Terraform code against a real AWS account or a dedicated staging environment before promoting to production.

**`terraform validate` does not catch runtime errors**
Validation checks HCL syntax and resource schema but cannot detect issues that only surface at apply time, such as API-level access control failures, resource quota limits, or ordering dependencies. Always review the plan output carefully before applying.

**Committing `.terraform.lock.hcl` ensures team-wide reproducibility**
The lock file generated by `terraform init` records the exact provider version and checksum. Omitting this file from version control means team members or CI pipelines may pull a different provider version, leading to plan diffs or unexpected behavior.

---

## 9. Troubleshooting

| Symptom | Root Cause | Resolution |
|---|---|---|
| `AccessControlListNotSupported` on apply | `BucketOwnerPreferred` not set before ACL | Ensure `aws_s3_bucket_ownership_controls` is created first via `depends_on` |
| `InvalidBucketAclWithBlockPublicAccess` | `block_public_acls = true` (AWS default) | Set all public access block flags to `false` in `aws_s3_bucket_public_access_block` |
| `connection refused` on `terraform apply` | LocalStack not running or DNS not resolving `aws` | Verify LocalStack container is up; confirm `/etc/hosts` or DNS resolves `aws` to `4566` |
| `Error: Invalid provider configuration` | Missing `skip_credentials_validation` | Add `skip_credentials_validation = true` and `skip_requesting_account_id = true` to the provider block |
| S3 path-style URL errors | `s3_use_path_style` not set | Add `s3_use_path_style = true` to the provider block |
| `terraform init` fails to download provider | Network access blocked or wrong `source` string | Confirm internet connectivity; verify `source = "hashicorp/aws"` spelling |

---





<img width="1063" height="587" alt="image" src="https://github.com/user-attachments/assets/073e7932-3a65-4c39-8e2e-1ed6dd2afb06" />

<img width="1043" height="628" alt="image" src="https://github.com/user-attachments/assets/b9838ab8-ba59-4dcf-b766-8b4ac4d3432b" />
<img width="1072" height="809" alt="image" src="https://github.com/user-attachments/assets/c39fa75e-dfec-4e30-b4e0-43ee25d168ab" />
<img width="1063" height="566" alt="image" src="https://github.com/user-attachments/assets/603e3417-23ed-44b8-a00f-288620655524" />
<img width="1069" height="821" alt="image" src="https://github.com/user-attachments/assets/0c4c774d-3f32-4ad6-a83a-85780f55ba3d" />
<img width="1076" height="635" alt="image" src="https://github.com/user-attachments/assets/713e4397-4af8-439a-9e25-0cb07428adfd" />
<img width="1073" height="579" alt="image" src="https://github.com/user-attachments/assets/1521bbf6-5187-4f57-9a6d-e7a8efceeb4e" />
