# Terraform AWS S3 Private Bucket Provisioning

> **Project:** Nautilus DevOps Data Migration Infrastructure
> **Environment:** LocalStack (us-east-1) | **Terraform Version:** >= 1.x | **AWS Provider:** 5.91.0

---

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Solution Architecture](#solution-architecture)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1 - Verify the Working Directory](#step-1---verify-the-working-directory)
  * [Step 2 - Review the Provider Configuration](#step-2---review-the-provider-configuration)
  * [Step 3 - Author the Main Terraform Configuration](#step-3---author-the-main-terraform-configuration)
  * [Step 4 - Verify the Written Configuration](#step-4---verify-the-written-configuration)
  * [Step 5 - Confirm Directory State](#step-5---confirm-directory-state)
  * [Step 6 - Initialize Terraform](#step-6---initialize-terraform)
  * [Step 7 - Validate the Configuration](#step-7---validate-the-configuration)
  * [Step 8 - Plan the Deployment](#step-8---plan-the-deployment)
  * [Step 9 - Apply the Configuration](#step-9---apply-the-deployment)
  * [Step 10 - Verify State and Resource Integrity](#step-10---verify-state-and-resource-integrity)
* [Resource Definitions](#resource-definitions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Troubleshooting](#troubleshooting)

---

## Overview

This repository documents the end-to-end Terraform provisioning of a **private AWS S3 bucket** as part of the Nautilus DevOps team's active data migration initiative. The infrastructure is deployed using **LocalStack** (a local AWS cloud emulator running at `http://aws:4566`) to simulate the AWS environment before promotion to production.

The bucket is configured with all four public access block controls enabled, enforcing strict data privacy and preventing any unintentional public exposure.

---

## Problem Statement

As part of a broader data migration process, the Nautilus DevOps team is consolidating data storage within the AWS environment. Both private and public S3 buckets are required as migration targets. This specific task provisions a **private S3 bucket** named `xfusion-s3-14217` with all public access explicitly blocked.

**Requirements:**

* Bucket name: `xfusion-s3-14217`
* All public access must be blocked (ACLs, policies, and bucket-level restrictions)
* Infrastructure must be defined exclusively in `/home/bob/terraform/main.tf`
* No additional `.tf` files may be created
* Resources must be provisioned in the `us-east-1` region

---

## Solution Architecture

```
/home/bob/terraform/
├── provider.tf          # AWS provider config pointing to LocalStack endpoints
├── main.tf              # S3 bucket + public access block resource definitions
└── .terraform/          # Terraform plugin cache (auto-generated after init)
    └── .terraform.lock.hcl
```

**Resources Provisioned:**

| Resource Type | Logical Name | Purpose |
|---|---|---|
| `aws_s3_bucket` | `private_bucket` | Creates the S3 bucket named `xfusion-s3-14217` |
| `aws_s3_bucket_public_access_block` | `private_bucket_access` | Blocks all public access vectors on the bucket |

---

## Prerequisites

* Terraform CLI installed and available in `$PATH`
* LocalStack running and accessible at `http://aws:4566`
* AWS provider `hashicorp/aws` version `5.91.0`
* Access to the `bob` user account on the IAC server (`iac-server`)
* VS Code with an integrated terminal (or equivalent CLI access)

---

## Repository Structure

```
/home/bob/terraform/
├── README.MD        # Original task brief (pre-existing)
├── provider.tf      # Pre-existing provider configuration
└── main.tf          # Authored during this task
```

> **Note:** Only `main.tf` was created during this implementation. `provider.tf` and `README.MD` were pre-existing.

---

## Implementation Guide

### Step 1 - Verify the Working Directory

Before authoring any configuration, verify the contents of the Terraform working directory to understand the pre-existing state.

```bash
cd ~/terraform
ls -la
```

**Output observed:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr 15 01:51 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

Two files exist: the task brief (`README.MD`) and the pre-configured provider (`provider.tf`). No `main.tf` is present yet.

*Screenshot: Terminal output showing initial directory listing with two pre-existing files*

<img width="1029" height="525" alt="image" src="https://github.com/user-attachments/assets/ce9a986d-2a68-497f-a500-6ca7443555fe" />

---

### Step 2 - Review the Provider Configuration

Inspect `provider.tf` to understand the AWS provider version, region, and LocalStack endpoint mappings before writing dependent resources.

```bash
cat /home/bob/terraform/provider.tf
```

**Key observations from the provider block:**

* AWS provider pinned to version `5.91.0`
* Region set to `us-east-1`
* `skip_credentials_validation = true` and `skip_requesting_account_id = true` are set for LocalStack compatibility
* `s3_use_path_style = true` is required for LocalStack's S3 path-based routing
* All AWS service endpoints are redirected to `http://aws:4566` (LocalStack)

*Screenshot: Terminal output of `cat provider.tf` displaying the full provider and endpoint configuration*

<img width="1067" height="784" alt="image" src="https://github.com/user-attachments/assets/7cb4216a-7555-4961-beab-d5ab323f5c77" />

---

### Step 3 - Author the Main Terraform Configuration

Create `main.tf` within the working directory using a heredoc to write both required resources in a single, atomic operation.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_s3_bucket" "private_bucket" {
  bucket = "xfusion-s3-14217"
}

resource "aws_s3_bucket_public_access_block" "private_bucket_access" {
  bucket = aws_s3_bucket.private_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
EOF
```

**Resource design decisions:**

* `aws_s3_bucket` declares the bucket with the exact required name. No ACL argument is set explicitly, which defaults to private in AWS provider v5+.
* `aws_s3_bucket_public_access_block` is defined as a **separate resource** (not an inline block), which is the correct pattern for AWS provider v4+ and v5+.
* The `bucket` argument in the access block uses `aws_s3_bucket.private_bucket.id` to create an **implicit dependency**, ensuring correct resource creation order.
* All four public access block controls are set to `true`:
  * `block_public_acls` - Blocks new public ACLs and ignores existing ones
  * `block_public_policy` - Blocks new public bucket policies
  * `ignore_public_acls` - Ignores all public ACLs regardless of origin
  * `restrict_public_buckets` - Restricts access to buckets with public policies

*Screenshot: Terminal showing the heredoc command being executed to write main.tf*

<img width="1053" height="811" alt="image" src="https://github.com/user-attachments/assets/c20d7e4b-1a2b-46bb-af3d-0d6aeed77a25" />

---

### Step 4 - Verify the Written Configuration

Confirm that `main.tf` was written correctly by reading it back before proceeding.

```bash
cat /home/bob/terraform/main.tf
```

**Expected output:**

```hcl
resource "aws_s3_bucket" "private_bucket" {
  bucket = "xfusion-s3-14217"
}

resource "aws_s3_bucket_public_access_block" "private_bucket_access" {
  bucket = aws_s3_bucket.private_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

*Screenshot: Terminal output of `cat main.tf` showing the complete resource configuration*

<img width="1076" height="778" alt="image" src="https://github.com/user-attachments/assets/faee2291-5049-4b77-a5fa-dc7568092721" />

---

### Step 5 - Confirm Directory State

Verify the directory now contains all three expected files before initializing Terraform.

```bash
ls -la /home/bob/terraform/
```

**Output observed:**

```
total 24
drwxr-xr-x 1 bob bob 4096 Apr 15 02:00 .
drwxr-x--- 1 bob bob 4096 Apr 15 01:51 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  326 Apr 15 02:00 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

`main.tf` is present at 326 bytes with a current timestamp, confirming it was written successfully.

*Screenshot: Terminal output showing updated directory listing with main.tf now present*

<img width="1073" height="542" alt="image" src="https://github.com/user-attachments/assets/56c9e34c-4e9c-474f-8bb2-77202c4e78de" />

---

### Step 6 - Initialize Terraform

Initialize the Terraform working directory. This downloads the `hashicorp/aws` provider at the pinned version and creates the lock file.

```bash
cd /home/bob/terraform && terraform init
```

**Output observed:**

```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "5.91.0"...
- Installing hashicorp/aws v5.91.0...
- Installed hashicorp/aws v5.91.0 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl ...

Terraform has been successfully initialized!
```

*Screenshot: Terminal output of `terraform init` showing provider download and successful initialization*

<img width="1063" height="628" alt="image" src="https://github.com/user-attachments/assets/70c88577-3dd7-45b1-86b9-7e9f9f0394f2" />

---

### Step 7 - Validate the Configuration

Run a static validation check to catch syntax errors or configuration issues before planning.

```bash
terraform validate
```

**Output observed:**

```
Success! The configuration is valid.
```

No errors or warnings. The configuration is syntactically correct and internally consistent.

*Screenshot: Terminal output of `terraform validate` confirming configuration validity*

<img width="1071" height="491" alt="image" src="https://github.com/user-attachments/assets/e4237265-df33-488c-85f6-8b0f4c727264" />

---

### Step 8 - Plan the Deployment

Generate and review the execution plan to confirm exactly which resources Terraform will create, change, or destroy.

```bash
terraform plan
```

**Plan summary:**

```
Plan: 2 to add, 0 to change, 0 to destroy.
```

**Resources planned for creation:**

1. `aws_s3_bucket.private_bucket`
   * `bucket = "xfusion-s3-14217"`
   * All other attributes are computed and known after apply

2. `aws_s3_bucket_public_access_block.private_bucket_access`
   * `block_public_acls = true`
   * `block_public_policy = true`
   * `ignore_public_acls = true`
   * `restrict_public_buckets = true`
   * `bucket` value is known after apply (sourced from the bucket resource ID)

> **Note:** No `-out` flag was used to save the plan file in this task. In production workflows, always use `terraform plan -out=tfplan` and apply with `terraform apply tfplan` to guarantee plan-to-apply consistency.

*Screenshot: Terminal output of `terraform plan` showing the full execution plan with 2 resources to add*

---

### Step 9 - Apply the Deployment

Apply the configuration with auto-approval to provision the resources against LocalStack.

```bash
terraform apply -auto-approve
```

**Output observed:**

```
aws_s3_bucket.private_bucket: Creating...
aws_s3_bucket.private_bucket: Creation complete after 1s [id=xfusion-s3-14217]
aws_s3_bucket_public_access_block.private_bucket_access: Creating...
aws_s3_bucket_public_access_block.private_bucket_access: Creation complete after 0s [id=xfusion-s3-14217]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

**Creation sequence:** Terraform correctly resolved the implicit dependency and created the S3 bucket first, then applied the public access block. The bucket was confirmed created with `id=xfusion-s3-14217`.

*Screenshot: Terminal output of `terraform apply -auto-approve` showing both resources created successfully*

---

### Step 10 - Verify State and Resource Integrity

After apply, verify the Terraform state to confirm both resources are tracked and inspect the public access block resource attributes in full.

**List all tracked resources:**

```bash
terraform state list
```

**Output:**

```
aws_s3_bucket.private_bucket
aws_s3_bucket_public_access_block.private_bucket_access
```

**Inspect the public access block resource in detail:**

```bash
terraform state show aws_s3_bucket_public_access_block.private_bucket_access
```

**Output:**

```hcl
# aws_s3_bucket_public_access_block.private_bucket_access:
resource "aws_s3_bucket_public_access_block" "private_bucket_access" {
    block_public_acls       = true
    block_public_policy     = true
    bucket                  = "xfusion-s3-14217"
    id                      = "xfusion-s3-14217"
    ignore_public_acls      = true
    restrict_public_buckets = true
}
```

All four public access block controls are confirmed `true` in the live state. The bucket is fully private.

*Screenshot: Terminal output of `terraform state show` displaying all confirmed resource attributes*

---

## Resource Definitions

### `aws_s3_bucket` - `private_bucket`

```hcl
resource "aws_s3_bucket" "private_bucket" {
  bucket = "xfusion-s3-14217"
}
```

Creates the S3 bucket with the exact required name. In AWS provider v5, the `acl` argument is no longer inline and defaults to private ownership unless explicitly overridden.

---

### `aws_s3_bucket_public_access_block` - `private_bucket_access`

```hcl
resource "aws_s3_bucket_public_access_block" "private_bucket_access" {
  bucket = aws_s3_bucket.private_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

Applies all four S3 Block Public Access controls to the bucket. Using `aws_s3_bucket.private_bucket.id` as the `bucket` value creates a Terraform implicit dependency, ensuring the bucket exists before this resource attempts creation.

---

## Best Practices Applied

* **Separation of concerns:** The `aws_s3_bucket_public_access_block` is defined as a dedicated resource rather than an inline block, consistent with AWS provider v4+ and v5+ recommendations and enabling independent lifecycle management.

* **Implicit dependency via resource reference:** Using `aws_s3_bucket.private_bucket.id` instead of a hardcoded string allows Terraform to correctly sequence resource creation without requiring an explicit `depends_on` directive.

* **Provider version pinning:** The AWS provider is pinned to `5.91.0` in `provider.tf`, ensuring deterministic builds and preventing unintended upgrades.

* **Lock file committed:** The `.terraform.lock.hcl` file generated during `terraform init` pins provider hash values, guaranteeing reproducibility across team members and CI pipelines.

* **Validate before plan:** Running `terraform validate` before `terraform plan` surfaces configuration errors early, before network calls or state reads are attempted.

* **State verification post-apply:** Using `terraform state list` and `terraform state show` after apply confirms that Terraform's state accurately reflects the provisioned infrastructure, validating both resource creation and attribute correctness.

* **Single `.tf` file constraint respected:** All resource definitions were authored exclusively in `main.tf` as required, maintaining clean directory hygiene and adhering to the task constraint.

---

## Lessons Learned

* **`aws_s3_bucket_public_access_block` must be a separate resource in AWS provider v4+.** In earlier provider versions, public access settings could be defined as inline blocks within `aws_s3_bucket`. That pattern is deprecated and removed in v4+. Always define it as a standalone resource.

* **All four block public access settings must be explicitly set.** Omitting any of the four controls leaves a potential public access vector open. Do not assume defaults; always declare all four as `true` for a fully private bucket.

* **LocalStack endpoint compatibility requires `s3_use_path_style = true`.** Without this setting, AWS SDK v3-style virtual-hosted-style URLs will fail against LocalStack's S3 emulation. This is a LocalStack-specific requirement and must not be carried into production provider configurations.

* **`-auto-approve` is appropriate for scripted and lab environments only.** In production, all applies must go through a change management process with human review of the plan output before approval.

* **Implicit dependencies are preferred over `depends_on`.** Using a resource reference (e.g., `aws_s3_bucket.private_bucket.id`) communicates dependency intent clearly in the code and allows Terraform to build a more accurate dependency graph than an explicit `depends_on` block.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `terraform init` fails with provider not found | Network issue or incorrect source in `required_providers` | Verify `source = "hashicorp/aws"` and network connectivity to the Terraform registry |
| `terraform apply` fails with connection refused | LocalStack not running or wrong endpoint URL | Confirm LocalStack is running and endpoints in `provider.tf` resolve to `http://aws:4566` |
| `Error: bucket already exists` | Bucket name collision in LocalStack state | Run `terraform destroy` to clean up, or rename the bucket in `main.tf` |
| Public access block not applied | Resource created before bucket ID was available | Ensure `bucket = aws_s3_bucket.private_bucket.id` (not a hardcoded string) to enforce dependency ordering |
| `terraform validate` passes but `plan` fails | Provider not initialized | Re-run `terraform init` before `terraform plan` |








<img width="1075" height="732" alt="image" src="https://github.com/user-attachments/assets/3221ed94-60b4-429f-bfac-9e86d276182d" />
<img width="1071" height="345" alt="image" src="https://github.com/user-attachments/assets/cc28b058-59c3-408e-ac30-37689ffb48dd" />
<img width="1064" height="695" alt="image" src="https://github.com/user-attachments/assets/44384778-ca64-4d51-a2ce-3aa823ac8b44" />
<img width="1072" height="402" alt="image" src="https://github.com/user-attachments/assets/6b035195-d31e-4853-ada2-f21ea1b70509" />
<img width="1070" height="328" alt="image" src="https://github.com/user-attachments/assets/8cbcb51e-a7d5-437b-93ac-a12b18a8c529" />
<img width="1071" height="342" alt="image" src="https://github.com/user-attachments/assets/922bd9c2-ca73-4ca7-8953-dc81c5b8e7b4" />


