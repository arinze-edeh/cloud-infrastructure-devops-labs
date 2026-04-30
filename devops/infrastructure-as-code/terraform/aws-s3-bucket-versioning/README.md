# Enabling S3 Bucket Versioning via Terraform on AWS

> **Domain:** Infrastructure as Code | **Provider:** AWS | **Tool:** Terraform  
> **Bucket:** `nautilus-s3-6053` | **Working Directory:** `/home/bob/terraform`

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: Inspect the Existing Terraform Configuration](#phase-1-inspect-the-existing-terraform-configuration)
  - [Phase 2: Add the Versioning Resource to main.tf](#phase-2-add-the-versioning-resource-to-maintf)
  - [Phase 3: Validate the Configuration](#phase-3-validate-the-configuration)
  - [Phase 4: Plan the Infrastructure Changes](#phase-4-plan-the-infrastructure-changes)
  - [Phase 5: Apply the Configuration](#phase-5-apply-the-configuration)
  - [Phase 6: Verify Versioning Status via AWS CLI](#phase-6-verify-versioning-status-via-aws-cli)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This implementation demonstrates how to enable versioning on an existing AWS S3 bucket using Terraform's dedicated `aws_s3_bucket_versioning` resource. The configuration is appended to an existing `main.tf` file within an already-initialized Terraform working directory, following the principle of incremental infrastructure change with full state awareness.

---

## Problem Statement

The DevOps team received a data protection requirement to enable versioning on an S3 bucket under management. Data protection and recovery are fundamental aspects of production data management. Systems must be in place to ensure that data can be recovered in the event of accidental deletion or object corruption.

**Requirement:**

- S3 bucket name: `nautilus-s3-6053`
- Versioning must be enabled using Terraform
- The existing `main.tf` file must be updated directly; no new `.tf` files should be created
- Working directory: `/home/bob/terraform`

---

## Solution Architecture

The solution uses Terraform's `aws_s3_bucket_versioning` resource, which is the current AWS provider-recommended approach for managing S3 versioning as a standalone resource, separate from the bucket definition itself.

```
aws_s3_bucket.s3_ran_bucket
        |
        | (bucket ID reference)
        v
aws_s3_bucket_versioning.nautilus_versioning
        |
        | versioning_configuration { status = "Enabled" }
        v
S3 Bucket: nautilus-s3-6053 --> Versioning: ENABLED
```

**Key Design Decisions:**

- The versioning resource references the bucket via `aws_s3_bucket.s3_ran_bucket.id` rather than a hardcoded string, creating an explicit Terraform dependency graph and ensuring correct apply ordering.
- The `acl` argument on the parent bucket resource is retained as-is from the pre-existing configuration, with its deprecation warning acknowledged but intentionally not modified to stay within the defined task scope.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Terraform | Installed and initialized in `/home/bob/terraform` |
| AWS CLI | Configured with credentials and appropriate S3/IAM permissions |
| AWS Provider | Already initialized (`.terraform/` directory present) |
| Existing Bucket | `nautilus-s3-6053` already provisioned and tracked in `terraform.tfstate` |

---

## Project Structure

```
/home/bob/terraform/
├── .terraform/                  # Provider plugins (pre-initialized)
├── .terraform.lock.hcl          # Provider version lock file
├── main.tf                      # Primary resource definitions (modified in this implementation)
├── provider.tf                  # AWS provider configuration
├── terraform.tfstate            # Current state file reflecting live infrastructure
└── README.MD                    # Original task reference
```

---

## Implementation Guide

### Phase 1: Inspect the Existing Terraform Configuration

Before making any changes, the existing directory layout and current `main.tf` contents were reviewed to understand the pre-existing resource state and avoid unintended drift.

```bash
ls -la
```

**Output:**

```
total 40
drwxr-xr-x 1 bob bob 4096 Apr 30 00:58 .
drwxr-x--- 1 bob bob 4096 Apr 30 00:58 ..
drwxr-xr-x 3 bob bob 4096 Apr 30 00:58 .terraform
-rw-r--r-- 1 bob bob 1406 Apr 30 00:58 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  148 Apr 30 00:58 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-rw-r-- 1 bob bob 2732 Apr 30 00:58 terraform.tfstate
```

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_s3_bucket" "s3_ran_bucket" {
  bucket = "nautilus-s3-6053"
  acl    = "private"

  tags = {
    Name        = "nautilus-s3-6053"
  }
}
```

**Observation:** The `terraform.tfstate` file already exists and reflects the bucket as a managed resource. The Terraform working directory is fully initialized with the AWS provider. The `main.tf` contains only the bucket resource; no versioning configuration is present.

> Screenshot: Directory listing showing all Terraform project files including `.terraform/`, `main.tf`, `provider.tf`, and `terraform.tfstate`

<img width="1051" height="678" alt="image" src="https://github.com/user-attachments/assets/7d411d42-3694-465a-81dd-ee3648b1d344" />

> Screenshot: `cat main.tf` output displaying the existing `aws_s3_bucket` resource block for `nautilus-s3-6053`

<img width="1042" height="575" alt="image" src="https://github.com/user-attachments/assets/c3eb6c52-f1d8-441d-acc9-a4e1d8fc9ed7" />

---

### Phase 2: Add the Versioning Resource to main.tf

The `aws_s3_bucket_versioning` resource block was appended to `main.tf` using `vi`. No new files were created, in strict adherence to the task requirement.

```bash
vi main.tf
```

The following block was added immediately after the existing `aws_s3_bucket` resource:

```hcl
resource "aws_s3_bucket_versioning" "nautilus_versioning" {
  bucket = aws_s3_bucket.s3_ran_bucket.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

**Verification after editing:**

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_s3_bucket" "s3_ran_bucket" {
  bucket = "nautilus-s3-6053"
  acl    = "private"

  tags = {
    Name        = "nautilus-s3-6053"
  }
}
resource "aws_s3_bucket_versioning" "nautilus_versioning" {
  bucket = aws_s3_bucket.s3_ran_bucket.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

> Screenshot: Updated `main.tf` in terminal showing both the `aws_s3_bucket` and `aws_s3_bucket_versioning` resource blocks after editing with `vi`

<img width="1050" height="766" alt="image" src="https://github.com/user-attachments/assets/08aebacd-1536-4a95-b560-16a4919190da" />
<img width="1049" height="752" alt="image" src="https://github.com/user-attachments/assets/b4f290bc-efd6-4eef-b0aa-d23cdb422f0a" />

---

### Phase 3: Validate the Configuration

`terraform validate` was run to check for syntax errors and configuration correctness before planning.

```bash
terraform validate
```

**Output:**

```
╷
│ Warning: Argument is deprecated
│
│   with aws_s3_bucket.s3_ran_bucket,
│   on main.tf line 3, in resource "aws_s3_bucket" "s3_ran_bucket":
│    3:   acl    = "private"
│
│ acl is deprecated. Use the aws_s3_bucket_acl resource instead.
╵
Success! The configuration is valid, but there were some validation warnings as shown above.
```

**Analysis:** The deprecation warning on `acl = "private"` originates from the pre-existing bucket resource. This is a known AWS provider deprecation; the `acl` argument has been superseded by the `aws_s3_bucket_acl` standalone resource. The warning does not block execution. The newly added versioning resource validated cleanly with no errors.

> Screenshot: `terraform validate` output confirming configuration validity with the `acl` deprecation warning

<img width="1043" height="769" alt="image" src="https://github.com/user-attachments/assets/b1029454-0935-4ae6-bb02-31ff12362fa1" />

---

### Phase 4: Plan the Infrastructure Changes

`terraform plan` was executed to produce a detailed execution preview, confirming that only the versioning resource would be created and no existing resources would be modified or destroyed.

```bash
terraform plan
```

**Output:**

```
aws_s3_bucket.s3_ran_bucket: Refreshing state... [id=nautilus-s3-6053]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket_versioning.nautilus_versioning will be created
  + resource "aws_s3_bucket_versioning" "nautilus_versioning" {
      + bucket = "nautilus-s3-6053"
      + id     = (known after apply)

      + versioning_configuration {
          + mfa_delete = (known after apply)
          + status     = "Enabled"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**Analysis:**

- Terraform correctly refreshed the existing bucket state from the state file, confirming `nautilus-s3-6053` is already managed.
- The plan shows `1 to add, 0 to change, 0 to destroy`, confirming zero destructive impact on the existing bucket.
- `mfa_delete` is listed as `(known after apply)` because it is an AWS-managed attribute not explicitly set in the configuration, which defaults to `Disabled` on apply.

> Screenshot: `terraform plan` output showing the `aws_s3_bucket_versioning` resource marked for creation with `status = "Enabled"` and the `Plan: 1 to add, 0 to change, 0 to destroy` summary

---

### Phase 5: Apply the Configuration

`terraform apply` was executed and confirmed with `yes` to provision the versioning resource against the live AWS environment.

```bash
terraform apply
```

**Confirmation prompt:**

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

**Output after confirmation:**

```
aws_s3_bucket_versioning.nautilus_versioning: Creating...
aws_s3_bucket_versioning.nautilus_versioning: Creation complete after 1s [id=nautilus-s3-6053]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**Result:** The `aws_s3_bucket_versioning` resource was created successfully in 1 second. Terraform confirms 1 resource added with no changes or destructions to pre-existing infrastructure.

> Screenshot: `terraform apply` execution showing the creation progress and `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.` confirmation

---

### Phase 6: Verify Versioning Status via AWS CLI

Post-apply verification was performed using the AWS CLI to independently confirm that versioning is active on the bucket, outside of Terraform's state.

```bash
aws s3api get-bucket-versioning --bucket nautilus-s3-6053
```

**Output:**

```json
{
    "Status": "Enabled"
}
```

**Result:** The AWS API confirms versioning status as `Enabled` on `nautilus-s3-6053`. This independently validates the Terraform apply outcome without relying solely on Terraform state.

> Screenshot: `aws s3api get-bucket-versioning` command output returning `{ "Status": "Enabled" }` confirming successful versioning activation

---

## Errors and Resolutions

| Step | Warning / Issue | Root Cause | Resolution |
|---|---|---|---|
| `terraform validate` | `acl is deprecated. Use the aws_s3_bucket_acl resource instead.` | The `acl` argument on `aws_s3_bucket` was deprecated in AWS provider v4.x in favor of the standalone `aws_s3_bucket_acl` resource | Warning acknowledged. The `acl` argument was part of the pre-existing configuration and was not modified, as the task scope was limited to enabling versioning only. The configuration remains functional; the warning is non-blocking. |
| `terraform plan` | Same `acl` deprecation warning appears a second time | Terraform evaluates provider-level deprecations at both validate and plan stages | No action required. Behavior is expected and does not affect plan or apply outcomes. |

---

## Best Practices Applied

**Explicit Resource References Over Hardcoded Values**

The versioning resource references the bucket using `aws_s3_bucket.s3_ran_bucket.id` rather than the literal string `"nautilus-s3-6053"`. This creates an implicit dependency in Terraform's resource graph, ensuring the bucket is fully created before versioning is applied and reducing the risk of configuration drift if the bucket name changes.

**Separation of Concerns via Standalone Resources**

Using `aws_s3_bucket_versioning` as a dedicated resource, rather than embedding versioning inside the `aws_s3_bucket` block, aligns with current AWS provider design philosophy introduced in v4.x. This modular approach improves readability, enables independent lifecycle management, and is the production-recommended pattern going forward.

**Pre-Apply Validation and Planning**

The full `validate -> plan -> apply` workflow was followed without skipping steps. This is non-negotiable in production environments where unreviewed applies can cause destructive changes. The plan output was reviewed to confirm `0 to destroy` before proceeding.

**Post-Apply CLI Verification**

Versioning status was confirmed via `aws s3api get-bucket-versioning` independent of Terraform state. Relying solely on `terraform.tfstate` for verification can mask discrepancies between declared state and actual AWS resource configuration, particularly in environments with multiple operators or external changes.

**Targeted File Modification**

The versioning resource was added to the existing `main.tf` rather than creating a new `.tf` file, maintaining a clean, consolidated configuration structure appropriate for single-service Terraform directories.

---

## Lessons Learned

**The `acl` Deprecation Reflects a Broader AWS Provider Pattern**

Starting with AWS provider v4.0, Amazon and HashiCorp moved toward decomposing monolithic S3 bucket resources into purpose-specific sub-resources (`aws_s3_bucket_acl`, `aws_s3_bucket_versioning`, `aws_s3_bucket_lifecycle_configuration`, etc.). Teams still running configurations with `acl` inline on `aws_s3_bucket` should schedule migration to the standalone resource pattern to avoid issues in future provider major versions where deprecated arguments may be removed.

**State Refresh on `terraform plan` Protects Against Drift**

The `aws_s3_bucket.s3_ran_bucket: Refreshing state... [id=nautilus-s3-6053]` line in the plan output confirms Terraform queried the live AWS API before generating the diff. In environments where resources may be modified outside of Terraform (console, CLI, other automation), this refresh is the mechanism that catches drift. Disabling it with `-refresh=false` should only be done with deliberate intent.

**`mfa_delete` Defaults Require Explicit Configuration for Compliance Environments**

The plan output showed `mfa_delete = (known after apply)`, indicating AWS will default this to `Disabled`. In regulated environments (HIPAA, PCI-DSS, SOC 2) where MFA delete may be a compliance requirement, `mfa_delete` must be explicitly set to `Enabled` in the `versioning_configuration` block. This requires the bucket owner's root account or a specific IAM configuration and should be evaluated during architecture review.

**Independent AWS CLI Verification Is a Production Habit**

Running `aws s3api get-bucket-versioning` after apply treats the AWS API as the source of truth rather than Terraform state alone. In production, it is good practice to verify critical security and data protection configurations via the provider API directly, particularly after first-time enablement.

---

## References

- [Terraform AWS Provider: aws_s3_bucket_versioning](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_versioning)
- [Terraform AWS Provider: aws_s3_bucket](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket)
- [AWS CLI Reference: get-bucket-versioning](https://docs.aws.amazon.com/cli/latest/reference/s3api/get-bucket-versioning.html)
- [AWS S3 Versioning Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
- [Terraform v4.x S3 Resource Decomposition Guide](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/version-4-upgrade#s3-bucket-refactoring)









<img width="1049" height="767" alt="image" src="https://github.com/user-attachments/assets/322dda7b-299f-46f9-9a44-38e29295fa31" />
<img width="1073" height="699" alt="image" src="https://github.com/user-attachments/assets/49aa3293-c90c-4873-8970-5c1f765eed52" />
<img width="1033" height="321" alt="image" src="https://github.com/user-attachments/assets/7b89a6dc-b2be-41e3-b166-4503446caf63" />

