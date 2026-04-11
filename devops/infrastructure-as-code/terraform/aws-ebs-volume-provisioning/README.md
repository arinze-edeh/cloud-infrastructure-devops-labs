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
   - [6.1 Confirm the Working Directory](#61-confirm-the-working-directory)
   - [6.2 Write main.tf with Provider and Resource Block](#62-write-maintf-with-provider-and-resource-block)
   - [6.3 Verify main.tf Content](#63-verify-maintf-content)
   - [6.4 Run terraform init (First Attempt - Failed)](#64-run-terraform-init-first-attempt---failed)
   - [6.5 Inspect provider.tf to Diagnose the Error](#65-inspect-providertf-to-diagnose-the-error)
   - [6.6 Rewrite main.tf Without the Provider Block](#66-rewrite-maintf-without-the-provider-block)
   - [6.7 Verify the Corrected main.tf Content](#67-verify-the-corrected-maintf-content)
   - [6.8 Confirm Both Configuration Files Exist](#68-confirm-both-configuration-files-exist)
   - [6.9 Run terraform init (Second Attempt - Succeeded)](#69-run-terraform-init-second-attempt---succeeded)
   - [6.10 Validate the Configuration](#610-validate-the-configuration)
   - [6.11 Plan the Deployment](#611-plan-the-deployment)
   - [6.12 Apply the Configuration](#612-apply-the-configuration)
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

The storage layer is established before dependent compute resources are provisioned. An AWS EBS (Elastic Block Store) volume of type `gp3` is created in the `us-east-1a` availability zone. The `gp3` volume type is the current-generation general-purpose SSD offering from AWS, providing a baseline of 3,000 IOPS and 125 MB/s throughput at no added cost, making it suitable for most general workloads.

The provider configuration (including LocalStack endpoint overrides and credential bypass settings) is maintained in a pre-existing `provider.tf` file, cleanly separating infrastructure concerns from resource definitions. The task requires that only `main.tf` be created and no additional `.tf` files be introduced.

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
| Configuration File | `main.tf` only (no new `.tf` files permitted) |

---

## 4. Prerequisites

* Terraform CLI installed and available in `PATH`
* A pre-configured `provider.tf` file already present in `/home/bob/terraform` containing the `terraform` block, the `hashicorp/aws` provider pinned to version `5.91.0`, and LocalStack endpoint overrides
* LocalStack running and accessible at `http://aws:4566` for all AWS service endpoints
* Terminal access via VS Code: right-click under the **EXPLORER** section and select **Open in Integrated Terminal**

---

## 5. Directory Structure

```
/home/bob/terraform/
├── provider.tf       # Pre-existing: Terraform block, provider config, and LocalStack endpoints
└── main.tf           # Created during this task: EBS volume resource definition only
```

---

## 6. Implementation Guide

### 6.1 Confirm the Working Directory

Confirm the active working directory is the designated Terraform workspace before writing any configuration.

```bash
pwd
```

**Output:**

```
/home/bob/terraform
```

*Screenshot: Terminal showing `/home/bob/terraform` as the current working directory*

<img width="984" height="471" alt="image" src="https://github.com/user-attachments/assets/76d65e56-e091-42fb-a448-e32752b25d89" />

---

### 6.2 Write main.tf with Provider and Resource Block

Write `main.tf` using a heredoc. At this point, the contents of `provider.tf` had not yet been inspected, so the initial file included both a `provider "aws"` block and the `aws_ebs_volume` resource definition.

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

---

### 6.3 Verify main.tf Content

Confirm the file was written correctly before proceeding.

```bash
cat main.tf
```

**Output:**

```hcl
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
```

*Screenshot: Terminal showing `cat main.tf` output with both the provider block and resource block*

<img width="983" height="694" alt="image" src="https://github.com/user-attachments/assets/097d4d14-f06f-4e4f-894c-9170f8b4923a" />

---

### 6.4 Run terraform init (First Attempt - Failed)

Attempt to initialize the Terraform working directory.

```bash
terraform init
```

**Error output:**

```
Initializing the backend...
╷
│ Error: Terraform encountered problems during initialisation, including problems
│ with the configuration, described below.
│
│ The Terraform configuration must be valid before initialization so that
│ Terraform can determine which modules and providers need to be installed.
│
╵
╷
│ Error: Duplicate provider configuration
│
│   on provider.tf line 10:
│   10: provider "aws" {
│
│ A default (non-aliased) provider configuration for "aws" was already given at
│ main.tf:1,1-15. If multiple configurations are required, set the "alias"
│ argument for alternative configurations.
╵
╷
│ Error: Duplicate provider configuration
│
│   on provider.tf line 10:
│   10: provider "aws" {
│
│ A default (non-aliased) provider configuration for "aws" was already given at
│ main.tf:1,1-15. If multiple configurations are required, set the "alias"
│ argument for alternative configurations.
╵
```

*Screenshot: Terminal showing `terraform init` failure with the duplicate provider configuration error printed twice*

**Note:** Terraform emitted the same `Duplicate provider configuration` error twice during this failed initialization pass. This is expected CLI output behavior for this class of error and does not indicate two separate underlying problems.

---

### 6.5 Inspect provider.tf to Diagnose the Error

Following the initialization failure, `provider.tf` was inspected to understand what was already declared in the working directory.

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

*Screenshot: Terminal showing the full contents of `provider.tf` including the pre-existing `provider "aws"` block with LocalStack endpoint overrides*

**Diagnosis:** `provider.tf` already contained a fully configured, non-aliased `provider "aws"` block at line 10. The `provider "aws"` block written in `main.tf` created a second default provider configuration for the same provider, which Terraform does not allow within a single module. The resolution is to remove the `provider "aws"` block from `main.tf` entirely, retaining only the resource definition. The provider declared in `provider.tf` is automatically inherited by all resources across all `.tf` files in the same module directory.

---

### 6.6 Rewrite main.tf Without the Provider Block

Overwrite `main.tf` using a heredoc, this time containing only the `aws_ebs_volume` resource definition.

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

---

### 6.7 Verify the Corrected main.tf Content

Confirm the provider block has been removed and only the resource definition remains.

```bash
cat main.tf
```

**Output:**

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

*Screenshot: Terminal showing the corrected `main.tf` containing only the resource block with no provider block*

---

### 6.8 Confirm Both Configuration Files Exist

List all `.tf` files in the working directory to confirm both `main.tf` and `provider.tf` are present with the expected sizes and timestamps.

```bash
ls -la *.tf
```

**Output:**

```
-rw-r--r-- 1 bob bob  178 Apr 11 02:24 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

*Screenshot: Terminal showing `ls -la *.tf` output with both files listed, including their sizes, permissions, and timestamps*

---

### 6.9 Run terraform init (Second Attempt - Succeeded)

Re-initialize the Terraform working directory now that the duplicate provider conflict has been resolved.

```bash
terraform init
```

**Output:**

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

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration, rerun this command
to reinitialize your working directory. If you forget, other commands will
detect it and remind you to do so if necessary.
```

*Screenshot: Terminal showing successful `terraform init` with provider download confirmation and lock file creation*

---

### 6.10 Validate the Configuration

Run a static validation pass to confirm the configuration is syntactically and semantically correct before invoking any provider APIs.

```bash
terraform validate
```

**Output:**

```
Success! The configuration is valid.
```

*Screenshot: Terminal showing `terraform validate` returning `Success! The configuration is valid.`*

---

### 6.11 Plan the Deployment

Generate and review an execution plan. This is a read-only operation that makes no changes to infrastructure.

```bash
terraform plan
```

**Output:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

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

──────────────────────────────────────────────────────────────────────────────
Note: You didn't use the -out option to save this plan, so Terraform can't
guarantee to take exactly these actions if you run "terraform apply" now.
```

*Screenshot: Terminal showing `terraform plan` output confirming `Plan: 1 to add, 0 to change, 0 to destroy`*

---

### 6.12 Apply the Configuration

Apply the configuration with the `-auto-approve` flag to provision the EBS volume without an interactive confirmation prompt.

```bash
terraform apply -auto-approve
```

**Output:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

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
aws_ebs_volume.xfusion_volume: Creating...
aws_ebs_volume.xfusion_volume: Still creating... [10s elapsed]
aws_ebs_volume.xfusion_volume: Creation complete after 11s [id=vol-7846392957bc0d8d6]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

*Screenshot: Terminal showing `terraform apply -auto-approve` completing successfully with volume ID `vol-7846392957bc0d8d6` assigned*

---

## 7. Verification

Upon successful apply, Terraform confirmed the following:

* **Resource created:** `aws_ebs_volume.xfusion_volume`
* **Volume ID assigned:** `vol-7846392957bc0d8d6`
* **Apply summary:** `1 added, 0 changed, 0 destroyed`

The volume is now available with the following confirmed attributes:

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
| **Command that failed** | `terraform init` (first attempt) |
| **Error message** | `Duplicate provider configuration` |
| **Reported location** | `provider.tf` line 10 conflicting with `main.tf` line 1 |
| **Root cause** | A non-aliased `provider "aws"` block was written in `main.tf` without first inspecting the working directory. `provider.tf` already declared a fully configured default `provider "aws"` block at line 10. Terraform does not permit more than one default (non-aliased) provider configuration for the same provider within a single module. |
| **Discovery method** | The error surfaced during `terraform init`. `provider.tf` was then inspected with `cat provider.tf` to confirm the conflict and understand what was already declared. |
| **Resolution** | Removed the `provider "aws"` block from `main.tf` entirely and rewrote the file containing only the `aws_ebs_volume` resource definition. The provider declared in `provider.tf` is automatically inherited by all resources across all `.tf` files in the same module directory. |
| **Error printed twice** | Terraform emitted the same error message twice during the failed init run. This is expected CLI output behavior for this error class and does not represent two distinct problems. |

---

## 9. Best Practices Applied

* **Provider and resource separation:** Provider configuration is isolated in `provider.tf`, keeping `main.tf` focused exclusively on resource definitions. This is a standard convention in production Terraform codebases and reduces the risk of configuration conflicts in shared or scaffolded environments.

* **Version pinning:** The AWS provider is pinned to a specific version (`5.91.0`) in the `required_providers` block within `provider.tf`, ensuring reproducible builds across environments and over time.

* **Lock file inclusion:** The `.terraform.lock.hcl` file generated by `terraform init` records the exact provider version and checksums. This file should be committed to version control to guarantee consistent provider selection for all team members.

* **Validate before plan:** `terraform validate` was executed after correcting the configuration to confirm there were no remaining syntax or schema errors before invoking the provider API during `terraform plan`.

* **Plan before apply:** `terraform plan` was run before `terraform apply` to review and confirm the proposed diff (`1 to add, 0 to change, 0 to destroy`) prior to making any infrastructure changes.

* **Modern volume type:** `gp3` is used instead of the legacy `gp2` type. `gp3` decouples IOPS and throughput from storage capacity, offering better price-performance and greater flexibility for future tuning.

* **Tagging at creation:** The EBS volume is tagged with `Name = "xfusion-volume"` at provisioning time, enabling immediate identification in the AWS console, cost allocation reports, and automated resource queries.

---

## 10. Lessons Learned

* **Always inspect existing `.tf` files before writing new configuration.** In scaffolded or shared Terraform workspaces, foundational files such as `provider.tf` may already declare providers, backend configurations, or variable definitions. The error encountered in this task was a direct consequence of writing a `provider "aws"` block in `main.tf` without first auditing the pre-existing files in the working directory. A single `ls *.tf` followed by `cat provider.tf` before writing any code would have prevented the failed init attempt entirely.

* **Terraform validates configuration before downloading providers.** The `terraform init` failure occurred before any provider plugins were fetched. Terraform performs a configuration validity pass as part of initialization, which means structural errors like duplicate providers surface immediately at init time rather than later during plan or apply.

* **The same error appearing twice during init is not a separate issue.** Terraform emitted the `Duplicate provider configuration` error twice during the failed init run. This is a known behavior of the CLI output renderer for this class of error and does not indicate two distinct problems. The underlying cause was a single conflict between two files.

* **`cat >` with a heredoc overwrites, it does not append.** When correcting `main.tf`, the same `cat > main.tf << 'EOF'` pattern was used to overwrite the file in place. This is a reliable, atomic approach for replacing file content in a terminal environment without opening an editor, and is well suited to scripted or task-based workflows.

* **`-auto-approve` is appropriate in controlled, single-resource environments.** In this task, `-auto-approve` was used against a LocalStack environment with a known, single-resource plan. In production pipelines, interactive approval gates or CI-level plan approval workflows should replace this flag for any apply that touches stateful or shared infrastructure.





<img width="1000" height="600" alt="image" src="https://github.com/user-attachments/assets/34fef496-f228-47a2-8460-b4d077a17f69" />

<img width="968" height="730" alt="image" src="https://github.com/user-attachments/assets/38879d69-f80d-480c-a467-51512dc6f882" />
<img width="1003" height="729" alt="image" src="https://github.com/user-attachments/assets/b37b49db-d119-45d2-9358-0adc5be4195a" />
<img width="1003" height="772" alt="image" src="https://github.com/user-attachments/assets/2550e41e-818a-4889-bd00-10528f00b115" />
<img width="981" height="740" alt="image" src="https://github.com/user-attachments/assets/3f1d86ce-e249-4ea6-a707-a49cb0f06cbb" />
<img width="988" height="514" alt="image" src="https://github.com/user-attachments/assets/812fe9a2-b1c2-4b76-91dc-e9c304588f4c" />
<img width="982" height="686" alt="image" src="https://github.com/user-attachments/assets/5e2ff42f-8405-4c5f-b1b5-4fc28574b007" />
<img width="972" height="543" alt="image" src="https://github.com/user-attachments/assets/1a105582-c390-4de4-8786-eb85bf2799f3" />
<img width="980" height="734" alt="image" src="https://github.com/user-attachments/assets/0eb30527-ec92-4ae6-8a95-c703662e50e3" />
<img width="979" height="733" alt="image" src="https://github.com/user-attachments/assets/966b779f-71c2-47f4-94c7-75e2cad42b19" />
