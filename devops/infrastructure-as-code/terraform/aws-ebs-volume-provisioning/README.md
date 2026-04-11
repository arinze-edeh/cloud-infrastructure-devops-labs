# AWS EBS Volume Provisioning with Terraform

> **Project:** Nautilus DevOps Infrastructure Migration to AWS
> **Phase:** Incremental Cloud Migration | Storage Layer
> **Tool:** Terraform (Infrastructure as Code)
> **Region:** `us-east-1`

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture and Design Intent](#2-architecture-and-design-intent)
3. [Task Requirements](#3-task-requirements)
4. [Prerequisites](#4-prerequisites)
5. [Directory Structure](#5-directory-structure)
6. [Implementation Guide](#6-implementation-guide)
   - [6.1 Navigate to the Terraform Working Directory](#61-navigate-to-the-terraform-working-directory)
   - [6.2 Inspect the Pre-existing Provider Configuration](#62-inspect-the-pre-existing-provider-configuration)
   - [6.3 Attempt 1: Write main.tf with Duplicate Provider Block (Error Encountered)](#63-attempt-1-write-maintf-with-duplicate-provider-block-error-encountered)
   - [6.4 Root Cause Analysis and Resolution](#64-root-cause-analysis-and-resolution)
   - [6.5 Attempt 2: Write Corrected main.tf Without Provider Block](#65-attempt-2-write-corrected-maintf-without-provider-block)
   - [6.6 Initialize Terraform](#66-initialize-terraform)
   - [6.7 Validate the Configuration](#67-validate-the-configuration)
   - [6.8 Plan the Deployment](#68-plan-the-deployment)
   - [6.9 Apply the Configuration](#69-apply-the-configuration)
7. [Verification](#7-verification)
8. [Errors Encountered and Resolutions](#8-errors-encountered-and-resolutions)
9. [Best Practices Applied](#9-best-practices-applied)
10. [Lessons Learned](#10-lessons-learned)

---

## 1. Project Overview

The Nautilus DevOps team is executing a phased migration of on-premises infrastructure to the AWS cloud. Rather than performing a single large-scale transition, the team has adopted an incremental migration strategy, decomposing the overall migration into smaller, independently deliverable units. This approach provides better risk control, minimizes disruption to ongoing operations, and allows systematic validation at each stage before advancing to the next.

This task represents one such unit: provisioning a persistent block storage volume on AWS using Terraform, following Infrastructure as Code (IaC) principles for repeatability, auditability, and team handoff.

---

## 2. Architecture and Design Intent

The storage layer is being established before dependent compute resources are provisioned. An AWS EBS (Elastic Block Store) volume of type `gp3` is created in the `us-east-1a` availability zone. The `gp3` volume type is the current-generation general-purpose SSD offering from AWS, providing a baseline of 3,000 IOPS and 125 MB/s throughput at no extra cost, making it suitable for most workloads.

The provider configuration (including LocalStack endpoint overrides and credential bypass settings) is maintained in a dedicated `provider.tf` file, cleanly separating infrastructure concerns from resource definitions.

---

## 3. Task Requirements

| Requirement | Value |
|---|---|
| Volume Name (Tag) | `xfusion-volume` |
| Volume Type | `gp3` |
| Volume Size | `2 GiB` |
| AWS Region | `us-east-1` |
| Availability Zone | `us-east-1a` |
| Working Directory | `/home/bob/terraform` |
| Configuration File | `main.tf` (only; no new `.tf` files) |

---

## 4. Prerequisites

* Terraform CLI installed and available in `PATH`
* A pre-configured `provider.tf` file already present in `/home/bob/terraform` with the `hashicorp/aws` provider pinned to version `5.91.0`
* LocalStack running and accessible at `http://aws:4566` for all AWS service endpoints
* Access to the terminal via VS Code: right-click under the **EXPLORER** section and select **Open in Integrated Terminal**

---

## 5. Directory Structure

```
/home/bob/terraform/
├── provider.tf       # Pre-existing: Terraform block, provider config, and LocalStack endpoints
└── main.tf           # Created during this task: EBS volume resource definition
```

---

## 6. Implementation Guide

### 6.1 Navigate to the Terraform Working Directory

Confirm the active working directory is the designated Terraform workspace.

```bash
pwd
```

**Expected Output:**

```
/home/bob/terraform
```

*Screenshot: Terminal showing `/home/bob/terraform` as the current directory*

---

### 6.2 Inspect the Pre-existing Provider Configuration

Before writing any Terraform configuration, inspect `provider.tf` to understand what is already declared. This step is critical to avoid duplicate provider blocks.

```bash
cat provider.tf
```

**Output observed:**

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

*Screenshot: Terminal output of `cat provider.tf` showing the full provider block*

**Key Observation:** A default (non-aliased) `provider "aws"` block is already declared in `provider.tf`. Any second `provider "aws"` block without an `alias` argument will cause a fatal initialization error.

---

### 6.3 Attempt 1: Write main.tf with Duplicate Provider Block (Error Encountered)

The initial `main.tf` was written including a `provider "aws"` block alongside the resource definition.

```bash
cat > main.tf << 'EOF'
provider "aws" {
  region = "us-east-1"
}

resource "aws_ebs_volume" "xfusion_volume" {
  availability_zone = "us-east-1a"
  size              = 2
  type              = "gp3"

  tags = {
    Name = "xfusion-volume"
  }
}
EOF
```

**Terraform init was then executed:**

```bash
terraform init
```

**Error output received:**

```
╷
│ Error: Duplicate provider configuration
│
│   on provider.tf line 10:
│   10: provider "aws" {
│
│ A default (non-aliased) provider configuration for "aws" was already given
│ at main.tf:1,1-15. If multiple configurations are required, set the "alias"
│ argument for alternative configurations.
╵
```

*Screenshot: Terminal showing `terraform init` failure with the Duplicate provider configuration error*

---

### 6.4 Root Cause Analysis and Resolution

**Root Cause:**

Terraform does not permit more than one default (non-aliased) provider configuration for the same provider within a single module. Since `provider.tf` already contained `provider "aws" { ... }`, adding another `provider "aws"` block in `main.tf` triggered a fatal conflict during the initialization phase, before any provider plugins were downloaded.

**Resolution:**

Remove the `provider "aws"` block entirely from `main.tf`. The provider configuration declared in `provider.tf` applies globally to all `.tf` files within the same module directory. The resource definition in `main.tf` will automatically inherit the provider configuration from `provider.tf`.

---

### 6.5 Attempt 2: Write Corrected main.tf Without Provider Block

Overwrite `main.tf` with only the resource definition, omitting any provider block.

```bash
cat > main.tf << 'EOF'
resource "aws_ebs_volume" "xfusion_volume" {
  availability_zone = "us-east-1a"
  size              = 2
  type              = "gp3"

  tags = {
    Name = "xfusion-volume"
  }
}
EOF
```

**Verify the file content:**

```bash
cat main.tf
```

**Expected output:**

```hcl
resource "aws_ebs_volume" "xfusion_volume" {
  availability_zone = "us-east-1a"
  size              = 2
  type              = "gp3"

  tags = {
    Name = "xfusion-volume"
  }
}
```

**Confirm both configuration files exist:**

```bash
ls -la *.tf
```

**Expected output:**

```
-rw-r--r-- 1 bob bob  178 Apr 11 02:24 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

*Screenshot: Terminal showing `ls -la *.tf` output with both `main.tf` and `provider.tf` listed*

---

### 6.6 Initialize Terraform

Initialize the Terraform working directory. This downloads the `hashicorp/aws` provider at the pinned version `5.91.0` and creates the `.terraform.lock.hcl` dependency lock file.

```bash
terraform init
```

**Expected output:**

```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "5.91.0"...
- Installing hashicorp/aws v5.91.0...
- Installed hashicorp/aws v5.91.0 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!
```

*Screenshot: Terminal showing successful `terraform init` output with provider installation confirmation*

---

### 6.7 Validate the Configuration

Run a static validation pass to confirm the configuration is syntactically and semantically correct before incurring any API calls.

```bash
terraform validate
```

**Expected output:**

```
Success! The configuration is valid.
```

*Screenshot: Terminal showing `terraform validate` returning `Success! The configuration is valid.`*

---

### 6.8 Plan the Deployment

Generate an execution plan to preview exactly what Terraform will create. This is a dry run and makes no changes to infrastructure.

```bash
terraform plan
```

**Expected output (key section):**

```
Terraform will perform the following actions:

  # aws_ebs_volume.xfusion_volume will be created
  + resource "aws_ebs_volume" "xfusion_volume" {
      + arn               = (known after apply)
      + availability_zone = "us-east-1a"
      + encrypted         = (known after apply)
      + final_snapshot    = false
      + id                = (known after apply)
      + iops              = (known after apply)
      + kms_key_id        = (known after apply)
      + size              = 2
      + snapshot_id       = (known after apply)
      + tags              = {
          + "Name" = "xfusion-volume"
        }
      + tags_all          = {
          + "Name" = "xfusion-volume"
        }
      + throughput        = (known after apply)
      + type              = "gp3"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

*Screenshot: Terminal showing `terraform plan` output with the single resource addition plan*

---

### 6.9 Apply the Configuration

Apply the configuration with auto-approval to provision the EBS volume without an interactive confirmation prompt.

```bash
terraform apply -auto-approve
```

**Expected output (key section):**

```
aws_ebs_volume.xfusion_volume: Creating...
aws_ebs_volume.xfusion_volume: Still creating... [10s elapsed]
aws_ebs_volume.xfusion_volume: Creation complete after 11s [id=vol-7846392957bc0d8d6]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

*Screenshot: Terminal showing `terraform apply -auto-approve` output with the EBS volume creation confirmation and volume ID*

---

## 7. Verification

Upon successful apply, Terraform confirms:

* **Resource created:** `aws_ebs_volume.xfusion_volume`
* **Volume ID assigned:** `vol-7846392957bc0d8d6`
* **Summary:** `1 added, 0 changed, 0 destroyed`

The volume is now available in `us-east-1a` with the following confirmed attributes:

| Attribute | Value |
|---|---|
| Name Tag | `xfusion-volume` |
| Type | `gp3` |
| Size | `2 GiB` |
| Availability Zone | `us-east-1a` |
| Volume ID | `vol-7846392957bc0d8d6` |

---

## 8. Errors Encountered and Resolutions

### Error: Duplicate Provider Configuration

| Field | Detail |
|---|---|
| **Error Message** | `Duplicate provider configuration` |
| **Triggered By** | `terraform init` |
| **Affected File** | `provider.tf` line 10 / `main.tf` line 1 |
| **Root Cause** | A non-aliased `provider "aws"` block was declared in both `main.tf` and the pre-existing `provider.tf`, violating Terraform's constraint of one default provider configuration per provider per module |
| **Resolution** | Removed the `provider "aws"` block from `main.tf`, retaining only the `aws_ebs_volume` resource definition. The provider configuration in `provider.tf` is inherited automatically by all resources in the module. |
| **Lesson** | Always inspect existing `.tf` files in the working directory before authoring new configuration files. In shared or pre-scaffolded environments, provider blocks, backend configurations, and variable declarations may already be present. |

---

## 9. Best Practices Applied

* **Separation of Concerns:** Provider configuration is isolated in `provider.tf`, keeping `main.tf` focused exclusively on resource definitions. This improves readability and reduces the risk of configuration conflicts in team environments.

* **Version Pinning:** The AWS provider is pinned to a specific version (`5.91.0`) in the `required_providers` block, ensuring reproducible builds across different environments and over time.

* **Lock File Committed:** The `.terraform.lock.hcl` file generated by `terraform init` records the exact provider version and hashes. This file should be committed to version control to guarantee consistent provider selection for all team members.

* **Plan Before Apply:** `terraform plan` was executed before `terraform apply` to review the proposed changes and confirm the expected diff (`1 to add, 0 to change, 0 to destroy`) prior to making any infrastructure changes.

* **Validate Before Plan:** `terraform validate` was run after correcting the configuration to catch any remaining syntax or schema errors without triggering provider API calls, saving time and avoiding unnecessary state interactions.

* **Tagging Strategy:** The EBS volume is tagged with `Name = "xfusion-volume"` at creation time, enabling immediate identification in the AWS console, cost allocation reports, and resource queries.

* **Modern Volume Type:** `gp3` is used instead of the legacy `gp2` type. `gp3` decouples IOPS and throughput from storage capacity, offering better price-performance and greater tuning flexibility.

---

## 10. Lessons Learned

* **Inspect the environment before writing configuration.** In scaffolded or shared Terraform workspaces, foundational files such as `provider.tf` may already declare providers, backends, or variable definitions. Writing a new provider block without first auditing existing files is a common source of initialization errors that are trivial to avoid.

* **Terraform's module-level provider scope is global.** Every `.tf` file within the same directory belongs to the same Terraform module and shares the same provider namespace. A provider block defined in any one file applies to all resources across all files in that module.

* **The `terraform init` failure was caught early.** Because Terraform validates configuration before downloading providers, the duplicate provider error surfaced during `init` rather than during `apply`. This is by design and reinforces the importance of running `init`, `validate`, and `plan` as distinct, observable stages rather than skipping directly to `apply`.

* **`-auto-approve` should be used deliberately.** In this controlled, single-resource task against a LocalStack environment, `-auto-approve` was appropriate. In production pipelines, interactive approval or pipeline-level gate controls should replace this flag to enforce human-in-the-loop validation for destructive or high-impact changes.





<img width="984" height="471" alt="image" src="https://github.com/user-attachments/assets/76d65e56-e091-42fb-a448-e32752b25d89" />
<img width="1000" height="600" alt="image" src="https://github.com/user-attachments/assets/34fef496-f228-47a2-8460-b4d077a17f69" />
<img width="983" height="694" alt="image" src="https://github.com/user-attachments/assets/097d4d14-f06f-4e4f-894c-9170f8b4923a" />
<img width="968" height="730" alt="image" src="https://github.com/user-attachments/assets/38879d69-f80d-480c-a467-51512dc6f882" />
<img width="1003" height="729" alt="image" src="https://github.com/user-attachments/assets/b37b49db-d119-45d2-9358-0adc5be4195a" />
<img width="1003" height="772" alt="image" src="https://github.com/user-attachments/assets/2550e41e-818a-4889-bd00-10528f00b115" />
<img width="981" height="740" alt="image" src="https://github.com/user-attachments/assets/3f1d86ce-e249-4ea6-a707-a49cb0f06cbb" />
<img width="988" height="514" alt="image" src="https://github.com/user-attachments/assets/812fe9a2-b1c2-4b76-91dc-e9c304588f4c" />
<img width="982" height="686" alt="image" src="https://github.com/user-attachments/assets/5e2ff42f-8405-4c5f-b1b5-4fc28574b007" />
<img width="972" height="543" alt="image" src="https://github.com/user-attachments/assets/1a105582-c390-4de4-8786-eb85bf2799f3" />
<img width="980" height="734" alt="image" src="https://github.com/user-attachments/assets/0eb30527-ec92-4ae6-8a95-c703662e50e3" />
<img width="979" height="733" alt="image" src="https://github.com/user-attachments/assets/966b779f-71c2-47f4-94c7-75e2cad42b19" />
